## 常用命令：

```shell
attach		# 当前 shell 下 attach 连接执行运行镜像
build		# 通过 Dockerfile 定制镜像
commit		# 提交当前容器为新的镜像
cp			# 从容器中拷贝指定文件或目录到宿主机中
create		# 创建一个新的容器，同 run ，但不启动容器
container	# 列出容器 = ps
diff		# 查看 docker 容器变化
events		# 从 docker 服务获取容器实时时间
exec		# 在已存在的容器上运行命令
export		# 到处容器的内容作为一个 tar 归档文件【对应 import】
history		# 展示一个镜像形成历史
images		# 列出系统当前镜像
import		# 从tar包中的内容创建一个新的文件系统映像【对应 export】
info		# 显示系统相关信息
inspect		# 查看容器详细信息（元数据）
kill		# kill指定 docker 容器
load		# 从一个 tar 包中加载一个镜像【对应 save】
login		# 注册或登录一个 docker 源服务器
logout		# 从当前 Docker registry 退出
logs		# 输出当前容器日志信息
port		# 查看映射端口对应的容器内源端口
pause		# 暂停容器
ps			# 列出容器列表 = container
pull		# 从docker镜像源服务器拉去指定镜像或库镜像
push		# 推送指定镜像或者库镜像至docker源服务器
restart		# 重启运行的容器
rm			# 移除一个或多个容器
rmi			# 移除一个或多个容器【无容器使用该镜像才可删除，否则需要删除相关容器才可继续或 -f 强制删除】
run			# 创建一个新的容器并运行一个命令
save		# 保存一个镜像为一个 tar 包【对应 load】
search		# 在docker hub 中搜索镜像
start		# 启动容器
stop		# 停止容器
stats		# 查看容器资源状态
tag			# 给源中镜像打标签
top			# 查看容器内运行的进程信息
unpause		# 取消暂停容器
version		# 查看 docker	版本号
wait		# 截取容器停止时的退出状态值
```

### 容器命令

```shell
#运行容器
docker run [options] image
	options:
		--name "image_name"		#容器名字		tomcat1	tomcat2
		-d		#后台方式运行
		-it		#使用交互的方式运行，进入容器查看内容
		-p		#制定容器的端口	-p	8080:8080
			-p	ip:主机端口:容器端口
			-p	主机端口:容器端口	（常用）
			-p	容器端口			#宿主机随机端口绑定容器端口
		-P		#随机指定端口
		-v		#卷挂载
			-v	本地目录：容器内目录	 #指定挂载
			-v	容器内路径			#匿名挂载
			-v 	卷名：容器内路径	  #具名挂载
			-v	容器内路径：ro,rw	   #设置容器权限，设置权限后，只能从外部操作，容器内无法操作
		-e		#环境配置：docker run -d -p 3306:3306 -e MYSQL_ROOT_PASSWORD=mysql a70d36bc331a
		--restart=always	#重启docker时容器跟随重启，不会停止
	
#后台启动容器
	docker run -d centos	#启动容器,容器前台必须要有一个进程在运行，docker发现没有进程后会自动停止
	
#退出容器
	exit			#直接停止容器退出
	Ctrl+P+Q		#不停止容器退出
	
#删除容器
	docker rmi  镜像id				   #删除指定镜像，无法删除正在运行的容器，强制删除 rm -f
	docker rmi -f $(docker images -q)	#删除全部容器
	
#列出所有的运行中的容器
	docker ps "command"
		command：
			-a	#列出当前正在运行的容器+历史运行过的容器
			-n	#显示最近创建的容器
			-q	#只显示容器的编号
		
#查看容器中的进程信息
	docker top 容器id
	
#启动和停止容器
	docker start	容器id
	docker restart 容器id
	docker stop 容器id
	docker kill 容器id
	
#查看日志
	docker logs	-tf --tailf 10 容器id
	docker logs -fn 10 41fcd397422a
	
#进入正在运行的容器	(容器通常是在后台运行，需要进入容器，修改配置)
	docker exec -it 11d49dbfafee /bin/bash		#进入容器后开启一个子shell
	docker attach 11d49dbfafee					#进入容器当前正在执行的shell
	
#从主机上拷贝文件到容器内
	docker cp 容器id 
	docker cp 11d49dbfafee:/root/index.html .	#将容器内root目录下的文件拷贝到当前目录下
	
#测试环境部署(一般部署都是停止容器后，还是可以查到容器id，这种方法就是用完就删除，查找不到容器的id)
	docker run -it --rm tomcat:9.0
```

### 镜像命令

```shell
#提交定制的镜像
	docker commit 
	docker commit -m="描述信息" -a="作者" 容器id 目标镜像名:[TAG]
#镜像导入导出
	docker save nginx > /tmp/nginx.tar.gz
	docker load -i /tmp/nginx.tar.gz
```

## Docker Volume

```shell
#将容器内的目录与宿主机目录建立关联
	docker run -it -v 主机目录：容器目录
	docker run -it -v /home/test:/home centos /bin/bash
	docker inspect 容器id	可以查看到容器元数据中mounts有挂载的目录

#匿名挂载
	-v	容器内路径
	docker run -d -P --name nginx01 -v /etc/nginx nginx
	docker volume ls	#查看所有的卷(volume)，-v只写了容器内路径，没有写容器外路径，这就是匿名挂载
	local  6b76844db030de22f7b7eca860742abe09067d3ae387b423b67889380bf80c0f
	
#具名挂载
	docker run -d -P --name nginx02 -v juming-nginx:/etc/nginx nginx
	docker volume ls	#通过-v查看卷，显示 ”卷名：容器内路径" 就是具名挂载
	local  juming-nginx
```

### 数据卷容器

两个docker容器数据同步

```shell
#建一个父docker，子docker通过--volumes-from继承父docker所有的数据,可以删除任一docker容器，只要有一个docker容器还存在，数据就不会丢失
docker run -it --name centos01 centos:7.4
docker run -it --name centos02 --volumes-from centos01 centos:7.4 /bin/bash
```

## DockerFile

dockerfile就是用来构建docker镜像的构建文件

```shell
#创建一个dockerfile文件
FROM centos

VOLUME ["volume01","volume02"]	#构建镜像时自动匿名挂载这两个卷在根目录

CMD echo "----end----"
CMD /bin/bash

#查看构建的镜像元数据
docker inspect id
        "Mounts": [
            {
                "Type": "volume",
                "Name": "9c037f18c4d02ced52396425133f2200be2cc0cec8a2468690525ec724f0ffc1",
                "Source": "/var/lib/docker/volumes/9c037f18c4d02ced52396425133f2200be2cc0cec8a2468690525ec724f0ffc1/_data",
                "Destination": "volume01",	#匿名挂载的卷
                "Driver": "local",
                "Mode": "",
                "RW": true,
                "Propagation": ""
            },
            {
                "Type": "volume",
                "Name": "086141e7cafac32ebc5bdbb489a6957e985b6028b0368944042c8faa09075420",
                "Source": "/var/lib/docker/volumes/086141e7cafac32ebc5bdbb489a6957e985b6028b0368944042c8faa09075420/_data",
                "Destination": "volume02",	#匿名挂载的卷
                "Driver": "local",
                "Mode": "",
                "RW": true,
                "Propagation": ""
            }
#若构建镜像时没有挂载卷，则需要通过 “-v 卷名:容器内路径” 手动进行挂载

#构建镜像
docker build -f /home/docker-test-colume/dockerfile1 -t home/centos7:1.0 .
```

### DockerFile构建过程

1、每个保留关键字（指令）都必须是大写字母

2、执行从上到下的顺序

3、#表示注释

4、每一个指令都会创建提交一个新的镜像层，并提交

### DockerFile指令

```shell
FROM			# 基础镜像，一切从这里开始构建
MAINTAINER		# 镜像构建作者，姓名+邮箱
RUN				# 镜像构建是需要运行的命令
ADD				# 步骤，tomcat镜像，添加内容，tomcat压缩包
WORKDIR			# 镜像工作目录
VOLUME			# 挂载的目录
EXPOSE			# 保留端口配置
CMD				# 指定这个容器启动时需要运行的命令,只有最后一个会生效，可被替代
ENTRYPOINT		# 指定这个容器启动时需要运行的命令，可以追加命令
ONBUILD			# 当构建一个被继承 DockerFile 这个时候就会运行ONBUILD的指令。触发指令
COPY			# 类似ADD，将我们文件拷贝到镜像中
ENV				# 构建的时候设置环境变量
```

```shell
#1、编写Dockerfile文件
FROM cenots
MAINTAINER Cai<cloud_chl@163.com>

ENV MYPATH /usr/local
WORKDIR $MYPATH

RUN yum -y install vim-enhanced
RUN yum -y install net-tools

EXPOSE 80

CMD echo "$MYPATH"
CMD echo "----build end ---"
CMD /bin/bash

#2、构建镜像
#docker build -f dockerfile文件路径 -t 自定义镜像名(小写):[tag] .
Successfully built 452d5ccf8515
Successfully tagged caicentos:1.0	#表示构建成功

#3、编写java项目环境dockerfile
FROM centos:7
MAINTAINER cai<cloud_chl@163.com>

COPY readme.txt /usr/local/readme.txt

ADD apache-tomcat-8.5.61.tar.gz /usr/local/
ADD jdk-8u281-linux-x64.tar.gz /usr/local

RUN yum -y install vim

ENV MYPATH /usr/local
WORKDIR $MYPATH

ENV JAVA_HOME /usr/local/jdk1.8.0_281
ENV CLASSPATH $JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
ENV CATALINA_HOME /usr/local/apache-tomcat-8.5.61
ENV CATALINA_BASE /usr/local/apache-tomcat-8.5.61
ENV PATH $PATH:$JAVA_HOME/bin:$CATALINA_HOME/lib:$CATALINA_HOME/bin

EXPOSE 8080

CMD $CATALINA_HOME/bin/startup.sh && tailf $CATALINA_HOME/logs/catalina.out

#4、构建镜像
docker run -d -p 9090:8080 -v /home/tomcat/webapps:/usr/local/apache-tomcat-8.5.61/webapps/web -v /home/tomcat/logs:/usr/local/apache-tomcat-8.5.61/logs tomcat:1.0

```

## Docker Registry

```shell
#构建私有镜像仓库
{
  "registry-mirrors": ["https://kzttrsz7.mirror.aliyuncs.com"],
  "insecure-registries": ["172.16.1.108:5000"]
}

#构建本地registry仓库
docker container run -d -p 5000:5000 --restart=always --name registry -v /opt/registry:/var/lib/registry registry

#将本地的镜像修改tag并上传到本地仓库
docker tag nginx:latest 172.16.1.108:5000/nginx:v1
docker push 172.16.1.108:5000/nginx:v1

#从本地仓库下载镜像
docker pull 172.16.1.108:5000/nginx:v1

#启动带秘钥功能的registry容器
yum -y install httpd-tools
mkdir /opt/registry-auth
htpasswd -Bbn Cai qwe > /opt/registry-auth/hpasswd		#生成账户信息
cat /opt/registry-auth/htpasswd
docker container run -d -p 5000:5000 -v /opt/registry-auth/:/auth -v /opt/registry:/var/lib/registry --name registry-auth -e "REGISTRY_AUTH=htpasswd" -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" -e "REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd" registry

#开启秘钥验证后，push都需要先登录才能进行操作
docker login 172.16.1.108:5000
```



## Docker Network

### Docker本地网络类型

```shell
docker network ls
nonoe	#无网络模式
bridge	#默认模式，相当于NAT
host	#公用宿主机Network NameSpace
container	#与其他容器公用Network NameSpace
```



### 自定义网络

```shell
#自定义一个网络
#--driver bridge
#--subnet 192.168.0.0/15
#--gateway 192.168.0.1
docker network create --driver bridge --subnet 192.168.0.0/16 --gateway 192.168.0.1 mynetwork		#定义了一个叫mynetwork的网络
docker network inspect mynetwork	#查看mynetwork的元数据
```

### 网络连通

```shell
#将其他网段的docker容器连接到另外一个网段，使两个不同网段的容器能通信
#一个容器两个不同网段的ip地址
#将另一个网段的容器加入mynetwork
docker netwrok connect mynetwork tomcat0-1
```

### Macvlan

```shell
docker network create --driver macvlan --subnet=172.16.1.0/24 --gateway=172.16.1.254 -o parent=eth0 macvlan_1
```

### Overlay

```shell
#docker之间跨主机访问
#1、启动consul服务，实现网络的统一配置管理
docker run -d -p 8500:8500 -h consul --name consul progrium/consul -server -bootstrap
vim /etc/docker/daemon.json
{
  "hosts":["tcp://0.0.0.0:2376","unix:///var/run/docker.sock"],
  "cluster-store": "consul://172.16.1.110:8500",
  "cluster-advertise": "172.16.1.110:2376"
}
#需要修改/usr/lib/systemd/system/docker.service
#然后重启docker

#2、创建overlay网络
docker network create -d overlay --subnet 192.168.0.0/24 --gateway 192.168.0.254 oll
#两边主机都需要修改配置文件，一端创建之后，两边都能看到创建的oll网络

#3、启动容器测试
docker run -it --network oll --name oll-1 busybox /bin/bash
#每个容器有两块网卡，eth0实现容器间通讯，eth1实现外网访问
```


