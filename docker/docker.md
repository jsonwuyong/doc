Dokcer基础文档
====
creater:sunnywu 
createdate:2019-05-07  

[建议阅读文档](https://yeasy.gitbooks.io/docker_practice/kubernetes/concepts.html)

一：安装 
----

[其它系统安装](https://yeasy.gitbooks.io/docker_practice/install/windows.html)

1.安装（CentOS 7.6 64位）   
yum install -y yum-utils device-mapper-persistent-data lvm2  
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo  
yum install -y docker-ce docker-ce-cli containerd.io  

2.配置阿里云加速  
mkdir /etc/docker  
vim /etc/docker/daemon.json  
```
{
  "registry-mirrors": ["https://ws2l845l.mirror.aliyuncs.com"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "256m"
  }
}
{
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
```

#第一段{}是设置阿里云得镜像加速跟容器日志大小，加速地址在阿里云中可以找到，第二段是docker存储驱动设置  

3.启动docker  
systemctl restart docker  
systemctl enable docker  
systemctl status docker  
docker --version 

4.（可选）非 root 用户身份管理 Docker  
groupadd docker  
usermod -aG docker $USER  

二：docker基本命令 
---- 
### 镜像  
1.docker search    #搜索镜像，STARS流行率、点赞数，OFFICIAL官方，AUTOMATED个人，建议去hub.docker.com上搜索镜像  
2.docker pull    # 拉取镜像  
    eg：docker pull ubuntu:16.04  
3.docker images    #查看镜像  
4.docker inspect    #检查镜像或者容器的详细信息，默认返回 JSON 格式  
    eg：docker inspect -f  '{{.Id}}' ubuntu:16.04  
5.docker tag    #修改镜像tag  
    eg：docker tag ubuntu:16.04 ubuntu16.04:v1  
6.docker rmi    #删除镜像  
    eg：docker rmi ubuntu16.04:v1  
7.docker build     #通过Dockerfile创建镜像  
    eg：docker build -f Docker_apache2_file -t apache2:v2 .  

### 容器  
1.docker run    #运行容器，常用参数：  
-it：'交互'的'伪终端'，通常一起使用  
-d：后台运行  
-w：指定工作目录  
-h：为容器设置HOST主机名称  
-P：随机端口映射  
-p：具体端口映射  
-v：卷映射  
--link <name or id>:alias：容器链接，原理是修改了hosts文件  
--rm：容器运行完成后自动删除  
--env：环境变量  
--name：容器名称  
--restart always：容器退出后自动重启，no –默认值，如果容器挂掉不自动重启，on-failure –当容器以非 0 退出时重启,always –不管退出码是多少都要重启,unless-stopped –不管退出状态码是什么始终重启容器，但是当-d启动时，如果容器之前已经为停止状态，不重启。  
--entrypoint：指定启动程序  
    eg：docker run -itd --name ubuntu16.04 ubuntu:16.04   
2.docker ps    #查看容器,常用参数：-a 默认ps代表查看正在运行中的容器，ps -a 包含运行跟终止的容器  
3.docker exec 参数  容器ID|容器名称 命令行  #进入容器，常用参数: -it     docker attach 也是进入容器，区别是exit容器时会stop容器，建议用exec  
    eg：docker exec -it ubuntu16.04 /bin/bash  
4.docker commit 容器ID|容器名称 镜像名称    #保存对容器的修改成为一个镜像，常用参数-m、-a  
    eg：docker commit -m '安装apache2' ubuntu16.04 apache2:v1  
5.docker stop|start|restart|stats 容器ID|容器名称  #停止|启动|重启|列出容器资源消耗  
6.docker logs 容器ID|容器名称    #查看容器log  
7.docker cp    #容器跟本地cp文件  
    eg：docker cp bb6a112910a9:/var/log/apache2/error.log .  
8.docker rm 容器ID|容器名称   #删除容器,常用参数：-f 强制删除    删除所有容器：docker rm \`docker ps -a -q\` 或者 docker ps -a | awk 'NR>2{print $1}' |xargs docker rm   

### 仓库  
1.推送    
docker login --username=ci@vsi registry.cn-shanghai.aliyuncs.com    #(ci读写  bot只读)    
docker tag [ImageId] registry.cn-shanghai.aliyuncs.com/vsi-open/fangyuan:[镜像版本号]    
docker push registry.cn-shanghai.aliyuncs.com/vsi-open/fangyuan:[镜像版本号]    
2.拉取   
docker pull registry.cn-shanghai.aliyuncs.com/vsi-open/fangyuan:[镜像版本号]  
  
### 具体操作实例
通过docker命令行搭建简单的apache2服务器,并上传镜像仓库  
1.拉取基础镜像  
docker pull ubuntu:16.04  
2.运行镜像，进入镜像安装apache2  
docker run -itd --name ubuntu16.04 ubuntu:16.04  
docker exec -it ubuntu16.04 /bin/bash  
apt-get update && apt-get -y install apache2 && exit   
3.commit镜像  
docker commit -m '安装apache2' ubuntu16.04 apache2:v1  
4.新建启动文件  
mkdir -p /root/apache2/html && cd /root/apache2 && vim run.sh  
```  
#!/bin/bash  
service apache2 restart  
/bin/bash  
```  
chmod 755 run.sh  
5.运行测试  
```  
docker run -itd \  
-w /home \  
-h apache2 \  
-v /root/apache2/run.sh:/home/run.sh \  
-v /root/apache2/html:/var/www/html \  
-p 80:80 \  
--name apache2 \  
--restart always \  
--entrypoint "/home/run.sh" \  
apache2:v1  
```  
6.上传仓库（阿里云上创建镜像仓库）  
docker login --username=fy.li@izhaohu.com registry.cn-shanghai.aliyuncs.com  
docker tag apache2:v1 registry.cn-shanghai.aliyuncs.com/vsi-devops/fangyuan:v1  
docker push registry.cn-shanghai.aliyuncs.com/vsi-devops/fangyuan:v1  

三：Dockerfile
----  
1.FROM    #指定构建镜像的基础镜像  
2.MAINTAINER    #作者信息  
3.ARG    #指定本次构建文件中的变量  
4.ENV    #指定本次构建文件中的变量跟构建后镜像中的环境变量，建议使用key=value  
ENV key value      # 只能设置一个变量  
ENV key=value ...   # 允许一次设置多个变量   
5.RUN   #build时执行的命令，多条RUN建议用&&  
RUN command    #shell格式  
RUN ["executable", "param1", "param2"]    #exec格式  
6.CMD    #容器启动时默认执行的命令，可以被docker run 运行时候的参数--entrypoint覆盖  
CMD command   #shell格式  
CMD ["executable", "param1", "param2"]    #exec格式  
7.ENTRYPOINT    #容器启动时执行的命令，不会被覆盖  
ENTRYPOINT command   #shell格式  
ENTRYPOINT ["executable", "param1", "param2"]    #exec格式  
8.ADD    #增加文件或目录到容器，tar文件自动解压，可以增加url文件，建议使用ADD src...dest  
ADD src... dest>  
ADD ["src",... "dest"]  
9.COPY    #复制文件或目录到容器，建议使用COPY src... dest 
COPY src... dest  
COPY ["src",... "dest"]  
10.VOLUME    #目录挂载到容器  
VOLUME ["目录"]  
系统会自动生成一个主机文件挂载到容器中，主机文件地址为/var/lib/docker/volumes/......  
11.WORKDIR    #容器中的工作目录，相当于shell中的cd命令  
WORKDIR 目录地址  
12.ONBUILD    #为子镜像着想  
ONBUILD RUN ls -al -当下一个Dockerfile文件中from本镜像时，会执行此命令  
13.EXPOSE    #'申明'容器对外映射的端口，需要在 docker run 的时候使用-p或者-P选项生效  
注释：  
1.ARG、ENV区别：  
ARG只在本次构建文件中有效，ENV不仅在本次构建文件中有效，而且在容器中也有效  
2.RUN、CMD、ENTRYPOINT区别：  
RUN命令在build时执行，CMD跟ENTRYPOINT在build时不执行，只在容器启动时执行，RUN在Dockerfile文件中可以存在多条且都会执行，CMD跟ENTRYPOINT可以存在多条，但是只执行最后一条  
CMD为容器启动时'默认'执行的命令，可以被命令行参数覆盖，ENTRYPOINT则不会。      
3.shell格式跟exec格式区别：  
eg：java -jar icare.jar  
shell格式：CMD java -jar icare.jar  
exec格式：CMD ["java",'-jar','icare.jar']  
shell格式可以使用变量，exec格式不可以使用变量  
exec格式可以接收参数，shll格式不可以，原因是shell格式相当于CMD ['/bin/sh','-c','java -jar icare.jar'],可以通过inspect -f '{{.CMD}}'  
4.ADD跟COPY区别：  
功能跟用法一样，ADD更强大，tar文件自动解压，支持url  

具体操作例子:  
通过Dockerfile搭建简单的apache2服务器  
mkdir -p /root/apache2/html && cd /root/apache2 && vim run.sh  
```
#!/bin/bash
service apache2 restart
/bin/bash
```
vim Dockerfile  
```
FROM ubuntu:16.04
MAINTAINER fangyuan<20190508>
WORKDIR /home
RUN apt-get update \
  && apt-get -y install apache2
ADD run.sh /home/run.sh
RUN chmod 755 /home/run.sh
EXPOSE 80
ENTRYPOINT bash /home/run.sh
```
docker build -t apache2:v2 .  
docker run -itd -P --restart always 8e24ff5df512  

四.docker-compose  
----
多个容器组合成为一个完整得应用。项目理解成应用，服务理解成容器  
### 安装  
curl -L "https://github.com/docker/compose/releases/download/1.24.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose  
chmod +x /usr/local/bin/docker-compose  
ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose  
docker-compose -v  

### docker-compose基本命令  
1.docker-compose -f    #指定yml执行文件  
2.docker-compose -p    #指定项目名称，默认将使用所在目录名称作为项目名,通过yml生成得容器会自动加上项目名称作为前缀  
3.docker-compose config    #检查yml文件格式是否错误  
4.docker-compose up -d    #启动项目  
5.docker-compose down    #停止项目  
6.docker-compose images    #列出项目中镜像  
7.docker-compose ps    #列出项目中得服务  

### yml模板文件中命令  
1.build    #指定Dockerfile文件，非默认Dockerfile名时，搭配context、dockerfile使用，args可以覆盖Dockerfile中变量  
    eg：
```    
webapp:  
  build:  
    context: ./dir  
    dockerfile: Dockerfile-apache2  
    args:  
      buildno: 1  
```
2.capadd, capdrop    #指定容器的内核能力分配  
    eg:  
```
cap_add:  
  - ALL  
```
3.command    #覆盖容器启动后默认执行的命令  
4.container_name    #指定容器名称。默认将会使用 项目名称_服务名称_序号  
5.depends_on    #容器启动先后  
6.env_file    #指定环境变量文件  
7.environment    #指定环境变量  
8.expose    #暴露端口  
9.external_links    #链接本次yml文件之外得容器  
10.extra_hosts    #修改host解析关系  
11.healthcheck    #健康检查  
12.image #指定具体镜像  
13.labels    #标签信息  
14.links    #建议不用，因为services中定义得可以直接使用  
15.logging    #日志选项  
16.network_mode    #网络模式  
17.networks    #网络配置  
18.ports    #端口映射  
19.secrets    #敏感数据  
20.ulimits    #指定容器的 ulimits 限制值  
21.volumes    #数据卷所挂载路径设置  

具体操作例子：    
```
version: "3"  
services:  

   db:
     image: mysql:5.7
     volumes:
       - ./db_data:/var/lib/mysql
     restart: always
     environment:
       MYSQL_ROOT_PASSWORD: vsi666666
       MYSQL_DATABASE: wordpress

   wordpress:
     depends_on:
       - db
     image: wordpress
     ports:
       - "8080:80"
     restart: always
     environment:
       WORDPRESS_DB_HOST: db:3306
       WORDPRESS_DB_USER: root
       WORDPRESS_DB_PASSWORD: vsi666666
```

docker-compose up -d   
docker-compose ps  
访问ip:8080  
    
注释：当修改mysql密码，即重新运行yml时，记得删除映射的db_data目录，因为数据是持久化存储，不删除密码不会改变
