#### 一：简单概述
k8s是什么？  
基于容器技术的分布式集群  
为应用提供部署运行，资源调度，服务发现，动态伸缩  
使应用安全，高可用，负载均衡，滚动升级，在线扩缩容  

etcd: 存储pods信息与服务发现  
flannel: 网络结构支持。后期用CNI，单机部署可以用容器的docker0网络  
kube-apiserver:  增删改查的唯一入口，如kubectl操作k8s。与etcd进行通信，持久化存储  
kube-controller-manager: 控制器，通过资源对象RC/RS/Deployment上定义的Label Selector来控制要监控的Pod的数量  
kube-scheduler: 调度器，调度Pod到node节点  
kubelet: 负责容器（docker）的创建，启停，销毁  
kube-proxy: 通信与负载均衡，通过Service的Label Selector来选择对应的Pod，建立起Service与Pod的请求转发路由表，从而实现Service的智能负载均衡  

k8s中的三种IP：  
Node IP：Node节点的IP地址  
Pod IP： Pod的IP地址  
Cluster IP：Service的IP地址  

#### 二：基本搭建
1.机器配置与内核升级(3.10 -> 4.4)  
#关闭 firewalld  
systemctl stop firewalld && systemctl disable firewalld  
#关闭 selinux  
setenforce 0  
#selinux设置为disabled    
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

#查看内核版本  
uname -r  
升级：  
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org  
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm  
yum --enablerepo=elrepo-kernel install kernel-lt -y  
cat /boot/grub2/grub.cfg |grep menuentry  
grub2-set-default 'CentOS Linux (4.4.177-1.el7.elrepo.x86_64) 7 (Core)'  
reboot  
uname -r # 内核确认  
yum remove -y kernel* # 移除无效内核 #此处增加*号  
yum remove -y kernel-tools-libs  
yum --enablerepo=elrepo-kernel install --skip-broken -y kernel-lt-headers   kernel-lt-tools kernel-lt-devel #此处增加了--skip-broken参数  
yum --enablerepo=elrepo-kernel install -y perf python-perf  
rpm -qa | grep kernel # 查看安装结果  
kernel-lt-tools-libs-4.4.177-1.el7.elrepo.x86_64  
kernel-lt-4.4.177-1.el7.elrepo.x86_64  
kernel-lt-tools-4.4.177-1.el7.elrepo.x86_64  
kernel-lt-devel-4.4.177-1.el7.elrepo.x86_64  
kernel-lt-headers-4.4.177-1.el7.elrepo.x86_64  

2.安装k8s单机集群  
yum -y update && yum -y install etcd kubernetes    #安装etcd跟k8s  
#修改/etc/kubernetes/kubelet  
KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=registry.cn-shanghai.aliyuncs.com/google-containers/pause-amd64:3.0"  
#修改/etc/kubernetes/apiserver  
KUBE_API_ADDRESS="--insecure-bind-address=0.0.0.0"  
KUBE_ADMISSION_CONTROL="--admission-control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ResourceQuota"  

启动：  
systemctl start etcd  
systemctl start docker  
systemctl start kube-apiserver  
systemctl start kube-controller-manager  
systemctl start kube-scheduler  
systemctl start kubelet  
systemctl start kube-proxy  

#### 三：基本操作
1.通过kubectl命令行创建一个应用  
kubectl run my-nginx --image=nginx --replicas=2 --port=80 #运行一个nginx应用  
kubectl get pods     #列出pods  
kubectl get deployments    #或者rs，列出deployment（部署）  
kubectl scale deployment/my-nginx --replicas=5    #扩展pods  
kubectl scale deployment/my-nginx --replicas=2    #缩展pods  
kubectl describe pods my-nginx-379829228-jqk4d     #列出pods的详细信息  
kubectl delete pods my-nginx-379829228-jqk4d    #删除pods,由于replicas机制，pod会生成一个新的  
kubectl delete deployments my-nginx    #删除部署，即彻底删除pods  
kubectl delete pods my-nginx-379829228-cwlbb --force --grace-period=0    #如果删除了部署，发现pods还存在  

2.通过yaml或json文件创建一个应用  
#创建一个nginx-rc.yaml文件
```  
apiVersion: v1 
kind: ReplicationController 
metadata: 
    name: my-nginx 
spec: 
    replicas: 3
    template: 
        metadata: 
            labels: 
                app: nginx 
        spec: 
            containers: 
                - name: nginx 
                  image: nginx
                  ports: 
                      - containerPort: 80 
```
#通过yaml文件创建应用  
kubectl create -f nginx-rc.yaml  

#根据yaml文件删除应用  
#kubectl delete -f nginx-rc.yaml  

3.创建Service供集群外访问  
kubectl expose rc/my-nginx --type="NodePort" --port 80 #创建services  
kubectl get services    #查看services  
kubectl describe services my-nginx    #显示services详细  
kubectl delete services my-nginx    #删除services  


4.通过yaml文件创建service  
#创建一个nginx-svc.yaml文件  
```
apiVersion: v1
kind: Service
metadata:
    name: my-nginx
spec:
    type: NodePort
    ports:
        - port: 80
          nodePort: 30080
    selector:
        app: nginx
```
kubectl create -f nginx-svc.yaml    #创建服务  
kubectl get services    #查看services  
kubectl describe services my-nginx    #显示services详细  

注释：  
1.type 默认为ClusterIP，不能分配外部端口，一般外部访问有NodePort、LoadBalancer 和 Ingress，各有优缺点，Nodeport端口范围为30000-32767  
2.ReplicaSet（RS）是Replication Controller（RC）的升级版本，RS是由RC（已淘汰） 演化而来，RS支持基于集合的标签，RC仅支持基于等式的标签。deployment 是带版本控制的 rs，可以回滚  

附录  
1.etcdctl常用命令:(pods存在etcd中)   
数据库操作  
etcdctl set /testdir/testkey "Hello world"    指定键值  
etcdctl get /testdir/testkey    获取  
etcdctl update /testdir/testkey "Hello"    更新  
etcdctl rm /testdir/testkey    删除  
etcdctl mk /testdir/testkey "Hello world"    键不存在，则创建一个新的键值  
etcdctl mkdir testdir2    键目录不存在，则创建一个新的键目录  
etcdctl setdir testdir3    目录不存在就创建，如果目录存在更新目录  
etcdctl updatedir testdir2    更新目录  
etcdctl rmdir dir1  
etcdctl ls    显示查看，经常会使用此命令，etcdctl ls /registry/../../..  
非数据库操作  
etcdctl backup --data-dir /var/lib/etcd  --backup-dir /home/etcd_backup    备份  
etcdctl watch testdir/testkey    监测一个键值的变化，一旦键值发生更新，就会输出最新的值并退出  
etcdctl exec-watch testdir/testkey -- sh -c 'ls'    监测一个键值的变化，一旦键值发生更新，就执行给定命令。  
etcdctl member list    通过list、add、remove命令列出、添加、删除etcd实例到etcd集群中。  
etcdctl member remove 8e9e05c52164694d  
etcdctl member add node3 http://ip:2380  

2.kubectl get 类别：（带#号注释为常用）  
```
componentstatuses (缩写 cs) #验证集群状态
configmaps (缩写 cm)
daemonsets (缩写 ds)
deployments (缩写 deploy) #获取deplyment状态
endpoints (缩写 ep) #获取endpoints
events (缩写 ev)
horizontalpodautoscalers (缩写 hpa)
ingresses (缩写 ing)
jobs
limitranges (缩写 limits)
namespaces (缩写 ns) #获取namespaces
networkpolicies
nodes (缩写 no) #获取nodes
persistentvolumeclaims (缩写 pvc)
persistentvolumes (缩写 pv)
pods (缩写 po) #获取pods
podsecuritypolicies (缩写 psp)
podtemplates
replicasets (缩写 rs) #获取rs
replicationcontrollers (缩写 rc) #获取rc
resourcequotas (缩写 quota)
secrets
serviceaccounts (缩写 sa)
services (缩写 svc) #获取服务
statefulsets
storageclasses
thirdpartyresources
```

3.yaml文件内容说明：  
```
apiVersion: v1             #指定api版本，此值必须在kubectl apiversion中，通过kubectl api-versions进行查看
kind: Pod                  #指定创建资源的角色/类型  
metadata:                  #资源的元数据/属性  
  name: web04-pod          #资源的名字，在同一个namespace中必须唯一  
  labels:                  #设定资源的标签
    k8s-app: apache  
    version: v1  
    kubernetes.io/cluster-service: "true"  
  annotations:             #自定义注解列表  
    - name: String         #自定义注解名字  
spec:                      #specification of the resource content 指定该资源的内容  
  restartPolicy: Always    #表明该容器一直运行，默认k8s的策略，在此容器退出后，会立即创建一个相同的容器  
  nodeSelector:            #节点选择，先给主机打标签kubectl label nodes kube-node1 zone=node1  
    zone: node1  
  containers:  
  - name: web04-pod        #容器的名字  
    image: web:apache      #容器使用的镜像地址  
    imagePullPolicy: Never #三个选择Always、Never、IfNotPresent，每次启动时检查和更新（从registery）images的策略，
                           # Always，每次都检查
                           # Never，每次都不检查（不管本地是否有）
                           # IfNotPresent，如果本地有就不检查，如果没有就拉取
    command: ['sh']        #启动容器的运行命令，将覆盖容器中的Entrypoint,对应Dockefile中的ENTRYPOINT  
    args: ["$(str)"]       #启动容器的命令参数，对应Dockerfile中CMD参数  
    env:                   #指定容器中的环境变量  
    - name: str            #变量的名字  
      value: "/etc/run.sh" #变量的值  
    resources:             #资源管理
      requests:            #容器运行时，最低资源需求，也就是说最少需要多少资源容器才能正常运行  
        cpu: 0.1           #CPU资源（核数），两种方式，浮点数或者是整数+m，0.1=100m，最少值为0.001核（1m）
        memory: 32Mi       #内存使用量  
      limits:              #资源限制  
        cpu: 0.5  
        memory: 32Mi  
    ports:  
    - containerPort: 80    #容器开发对外的端口
      name: httpd          #名称
      protocol: TCP  
    livenessProbe:         #pod内容器健康检查的设置
      httpGet:             #通过httpget检查健康，返回200-399之间，则认为容器正常  
        path: /            #URI地址  
        port: 80  
        #host: 127.0.0.1   #主机地址  
        scheme: HTTP  
      initialDelaySeconds: 180 #表明第一次检测在容器启动后多长时间后开始  
      timeoutSeconds: 5    #检测的超时时间  
      periodSeconds: 15    #检查间隔时间  
      #也可以用这种方法  
      #exec: 执行命令的方法进行监测，如果其退出码不为0，则认为容器正常  
      #  command:  
      #    - cat  
      #    - /tmp/health  
      #也可以用这种方法  
      #tcpSocket: //通过tcpSocket检查健康   
      #  port: number   
    lifecycle:             #生命周期管理  
      postStart:           #容器运行之前运行的任务  
        exec:  
          command:  
            - 'sh'  
            - 'yum upgrade -y'  
      preStop:             #容器关闭之前运行的任务  
        exec:  
          command: ['service httpd stop']  
    volumeMounts:         
    - name: volume         #挂载设备的名字，与volumes[*].name 需要对应    
      mountPath: /data     #挂载到容器的某个路径下  
      readOnly: True  
  volumes:                 #定义一组挂载设备  
  - name: volume           #定义一个挂载设备的名字  
    #meptyDir: {}  
    hostPath:  
      path: /opt           #挂载设备类型为hostPath，路径为宿主机下的/opt,这里设备类型支持很多种
```
