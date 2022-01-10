 

## Kubeadm部署

### 1、组件

#### （1）master组件

 * #### apiserver

   集群统一入口，以restful方式，交给etcd存储

* #### scheduler

  节点调度，选择node节点应用部署

* #### controller-manager

  处理集群中常规后台任务，一个资源对应一个控制器

* #### etcd

  存储系统， 用于保存集群相关的数据

#### （2）node组件

* #### kubelet

  master派到node节点代表，管理本机容器

* #### kube-proxy

  提供网络代理，负载均衡等操作



### 2、核心概念

```shell
1、Pod
 * 最小部署单元
 * 一组容器的集合
 * 共享网络
 * 生命周期是短暂的
 
2、controller
 * 确保预期的pod副本数量
 * 无状态应用部署
 * 有状态应用部署 
 确保所有的node运行同一个pod
 一次性任务和定时任务
 
3、service
 * 定义一组pod的访问规则
```



### 3、集群部署规划

```shell
1、关闭防火墙
2、关闭selinux
3、关闭swap
   swapoff -a		#临时
   sed -ri 's/.*swap.*/#&/' /etc/fstab 	#永久
4、规划主机名
5、在master中添加hosts主机信息
6、将桥接的IPV4流量传递到iptables的链
cat >> /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system 生效
7、时间同步
```



### 4、安装docker、kubeadm，kubelet，kubectl

```shell
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install docker-ce docker-ce-cli containerd.io -y

cat >>/etc/docker/daemon.json <<-EOF
{
  "registry-mirrors": ["https://kzttrsz7.mirror.aliyuncs.com"]
}
EOF
================================================================================================================================
cat>> /etc/yum.repos.d/kubernets.repo <<-EOF
[kubernetes]
name=kubernetes
baseurl=https://mirrors.tuna.tsinghua.edu.cn/kubernetes/yum/repos/kubernetes-el7-$basearch
enabled=1
gpgcheck=0

[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
       http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
================================================================================================================================
yum -y install kebelet kubeadm kubectl
systemctl enable kubelet
```



### 5、部署Kubernetes Master

```shell
$ kubeadm init \
  --apiserver-advertise-address=192.168.1.1 \
  --image-repository registry.aliyuncs.com/google_containers \
  --kubernetes-version=v1.20.4 \
  --service-cidr=172.1.0.0/16 \
  --pod-network-cidr=10.244.0.0/16  #这个不能随意修改，必须跟kube-flannel.yml中的Network保持一致 
```

使用kubectl工具：

```shell
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
  export KUBECONFIG=/etc/kubernetes/admin.conf
```



master更换ip后需要重新初始化节点

```shell
master：需要删除一下文件才能执行kubeadm reset
[root@K8S-Master ~]# rm -rf /etc/kubernetes/*
[root@K8S-Master ~]# rm -rf /root/.kube/*
[root@K8S-Master ~]# rm -rf /var/lib/etcd/*

node：kubeadm reset
```



### 6、部署CNI网络插件

```shell
kubectl apply -f kube-flannel.yml
kubectl get pods -n kube-system
```



### 7、加入Kubertes Node

在node节点执行。

向集群添加新节点，执行kubeadm init输出的kubeadm join命令：

```shell
kubeadm join 192.168.1.1:6443 --token ukgzfa.m8m191o7hlsv0owt \
    --discovery-token-ca-cert-hash sha256:b370b4e8a784ad6cd04ed70bda0087dafe847e2225dc84f4f0b0ec9438681dcf
```

默认token有效期为24小时，过期之后，该token不可用，就需要重新创建token

```shell
kubeadm token create --print-join-command
```

### 8、测试集群

```shell
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get pod,svc
```

## Kubespray部署

### 1. 服务器说明

#### 1.1. 节点要求

###### 节点数 >=3台

###### CPU >=2

###### Memory >=2G

###### 安全组：关闭（允许节点之间任意端口访问，以及ipip隧道协议通讯）

#### 1.2. 演示环境说明

我们这里使用的是三台centos 7.5的虚拟机，具体信息如下表：  

|  系统类型  |    IP地址     | 节点角色 | CPU  | Memory | Hostname |
| :--------: | :-----------: | :------: | :--: | :----: | :------: |
| centos-7.5 | 10.155.19.223 |  master  | \>=2 | \>=2G  |  node-1  |
| centos-7.5 | 10.155.19.64  |  master  | \>=2 | \>=2G  |  node-2  |
| centos-7.5 | 10.155.19.147 |  worker  | \>=2 | \>=2G  |  node-3  |

### 2. 系统设置（所有节点）

> 注意：所有操作使用root用户执行

#### 2.1 主机名

主机名必须合法，并且每个节点都不一样（建议命名规范：数字+字母+中划线组合，不要包含其他特殊字符）。

```bash
# 查看主机名
$ hostname
# 修改主机名
$ hostnamectl set-hostname <your_hostname>
```

#### 2.2 关闭防火墙、selinux、swap，重置iptables

```bash
# 关闭selinux
$ setenforce 0
$ sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config
# 关闭防火墙
$ systemctl stop firewalld && systemctl disable firewalld

# 设置iptables规则
$ iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat && iptables -P FORWARD ACCEPT
# 关闭swap
$ swapoff -a && free –h

# 关闭dnsmasq(否则可能导致容器无法解析域名)
$ service dnsmasq stop && systemctl disable dnsmasq
```

#### 2.3 k8s参数设置

```bash
# 制作配置文件
$ cat > /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_nonlocal_bind = 1
net.ipv4.ip_forward = 1
vm.swappiness = 0
vm.overcommit_memory = 0
EOF
# 生效文件
$ sysctl -p /etc/sysctl.d/kubernetes.conf
```

#### 2.4 移除docker相关软件包（可选）

```bash
$ yum remove -y docker*
$ rm -f /etc/docker/daemon.json
```

### 3. 使用kubespray部署集群

这部分只需要在一个 **操作** 节点执行，可以是集群中的一个节点，也可以是集群之外的节点。甚至可以是你自己的笔记本电脑。我们这里使用更普遍的集群中的任意一个linux节点。

#### 3.1 配置免密

使 **操作** 节点可以免密登录到所有节点

```bash
# 1. 生成keygen（执行ssh-keygen，一路回车下去）
$ ssh-keygen
# 2. 查看并复制生成的pubkey
$ cat /root/.ssh/id_rsa.pub
# 3. 分别登陆到每个节点上，将pubkey写入/root/.ssh/authorized_keys
$ mkdir -p /root/.ssh
$ echo "<上一步骤复制的pubkey>" >> /root/.ssh/authorized_keys
```

#### 3.2 依赖软件下载、安装

```bash
# 安装基础软件
$ yum install -y epel-release python36 python36-pip git
# 下载kubespray源码
$ wget https://github.com/kubernetes-sigs/kubespray/archive/v2.15.0.tar.gz
# 解压缩
$ tar -xvf v2.15.0.tar.gz && cd kubespray-2.15.0
# 安装requirements
$ cat requirements.txt
$ pip3.6 install -r requirements.txt

## 如果install遇到问题可以先尝试升级pip
## $ pip3.6 install --upgrade pip
```

python更换国内源

```shell
1.编辑pip.conf

mkdir -p ~/.pip
vim ~/.pip/pip.conf

[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
[install]
trusted-host = https://pypi.tuna.tsinghua.edu.cn

```



#### 3.3 生成配置

项目中有一个目录是集群的基础配置，示例配置在目录inventory/sample中，我们复制一份出来作为自己集群的配置

```bash
# copy一份demo配置，准备自定义
$ cp -rpf inventory/sample inventory/mycluster
```

由于kubespray给我们准备了py脚本，可以直接根据环境变量自动生成配置文件，所以我们现在只需要设定好环境变量就可以啦

```bash
# 使用真实的hostname（否则会自动把你的hostname改成node1/node2...这种哦）
$ export USE_REAL_HOSTNAME=true
# 指定配置文件位置
$ export CONFIG_FILE=inventory/mycluster/hosts.yaml
# 定义ip列表（你的服务器内网ip地址列表，3台及以上，前两台默认为master节点）
$ declare -a IPS=(10.155.19.223 10.155.19.64 10.155.19.147)
# 生成配置文件
$ python3 contrib/inventory_builder/inventory.py ${IPS[@]}
```

#### 3.4 个性化配置

配置文件都生成好了，虽然可以直接用，但并不能完全满足大家的个性化需求，比如用docker还是containerd？docker的工作目录是否用默认的/var/lib/docker？等等。当然默认的情况kubespray还会到google的官方仓库下载镜像、二进制文件，这个就需要你的服务器可以上外面的网，想上外网也需要修改一些配置。

```bash
# 定制化配置文件
# 1. 节点组织配置（这里可以调整每个节点的角色）
$ vim inventory/mycluster/hosts.yaml
# 2. containerd配置（教程使用containerd作为容器引擎）
$ vim inventory/mycluster/group_vars/all/containerd.yml
# 3. 全局配置（可以在这配置http(s)代理实现外网访问）
$ vim inventory/mycluster/group_vars/all/all.yml
# 4. k8s集群配置（包括设置容器运行时、svc网段、pod网段等）
$ vim inventory/mycluster/group_vars/k8s-cluster/k8s-cluster.yml
# 5. 修改etcd部署类型为host（默认是docker）
$ vim ./inventory/mycluster/group_vars/etcd.yml
# 6. 附加组件（ingress、dashboard等）
$ vim ./inventory/mycluster/group_vars/k8s-cluster/addons.yml
```

#### 3.5 一键部署

配置文件都调整好了后，就可以开始一键部署啦，不过部署过程不出意外会非常慢。
如果您使用的是教程同一个版本建议下使用网盘下载好二进制文件和镜像

##### 网盘下载二进制（可选）

###### 链接: https://pan.baidu.com/s/11eDin8BDJVzGgXJW9e6cog 提取码: mrj9

下载好之后解压到每个节点的根目录即可，解压完成后的目录是/tmp/releases

##### 一键部署

```bash
# -vvvv会打印最详细的日志信息，建议开启
$ ansible-playbook -i inventory/mycluster/hosts.yaml  -b cluster.yml -vvvv
```

经过漫长的等待后，如果没有问题，整个集群都部署起来啦

###### 下载镜像（可选）

为了减少“一键部署”的等待时间，可以在部署的同时，预先下载一些镜像。

```bash
$ curl https://gitee.com/pa/pub-doc/raw/master/kubespray-v2.15.0-images.sh|bash -x
```

`重要：此操作需要确保上面的"一键部署"执行后，并成功安装了containerd后即可手动下载镜像）`

#### 3.6 清理代理设置

清理代理设置（运行时不再需要代理，删掉代理配置即可）

##### 删除docker的http代理（在每个节点执行）

```bash
$ rm -f /etc/systemd/system/containerd.service.d/http-proxy.conf
$ systemctl daemon-reload
$ systemctl restart containerd
```

##### 删除yum代理

```bash
# 把grep出来的代理配置手动删除即可
$ grep 8118 -r /etc/yum*
```



### 4、集群冒烟测试

#### 1. 创建nginx ds

```bash
 # 写入配置
$ cat > nginx-ds.yml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: nginx-ds
  labels:
    app: nginx-ds
spec:
  typF: NodePort
  selector:
    app: nginx-ds
  ports:
  - name: http
    port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-ds
spec:
  selector:
    matchLabels:
      app: nginx-ds
  templatF:
    metadata:
      labels:
        app: nginx-ds
    spec:
      containers:
      - name: my-nginx
        image: nginx:1.19
        ports:
        - containerPort: 80
EOF

# 创建ds
$ kubectl apply -f nginx-ds.yml


```

#### 2. 检查各种ip连通性

```bash
# 检查各 Node 上的 Pod IP 连通性
$ kubectl get pods  -o wide

# 在每个节点上ping pod ip
$ ping <pod-ip>

# 检查service可达性
$ kubectl get svc

# 在每个节点上访问服务
$ curl <service-ip>:<port>

# 在每个节点检查node-port可用性
$ curl <node-ip>:<port>
```

#### 3. 检查dns可用性

```bash
# 创建一个nginx pod
$ cat > pod-nginx.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: docker.io/library/nginx:1.19
    ports:
    - containerPort: 80
EOF

# 创建pod
$ kubectl apply -f pod-nginx.yaml

# 进入pod，查看dns
$ kubectl exec nginx -it -- /bin/bash

# 查看dns配置
root@nginx:/# cat /etc/resolv.conf

# 查看名字是否可以正确解析
root@nginx:/# ping nginx-ds
```

#### 4. 日志功能

测试使用kubectl查看pod的容器日志

```bash
$ kubectl get pods
$ kubectl logs <pod-name>
```

#### 5. Exec功能

测试kubectl的exec功能

```bash
$ kubectl get pods -l app=nginx-ds
$ kubectl exec -it <nginx-pod-name> -- nginx -v
```

### 5、访问dashboard

#### 1. 创建service

```bash
$ cat > dashboard-svc.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  namespace: kube-system
  name: dashboard
  labels:
    app: dashboard
spec:
  type: NodePort
  selector:
    k8s-app: kubernetes-dashboard
  ports:
  - name: https
    nodePort: 30000
    port: 443
    targetPort: 8443
EOF

$ kubectl apply -f dashboard-svc.yaml
```

#### 2. 访问dashboard

为了集群安全，从 1.7 开始，dashboard 只允许通过 https 访问，我们使用nodeport的方式暴露服务，可以使用 https://NodeIP:NodePort 地址访问 
关于自定义证书 
默认dashboard的证书是自动生成的，肯定是非安全的证书，如果大家有域名和对应的安全证书可以自己替换掉。使用安全的域名方式访问dashboard。 
在dashboard-all.yaml中增加dashboard启动参数，可以指定证书文件，其中证书文件是通过secret注进来的。

> \- –tls-cert-file  
> \- dashboard.cer  
> \- –tls-key-file  
> \- dashboard.key

#### 3. 登录dashboard

Dashboard 默认只支持 token 认证，所以如果使用 KubeConfig 文件，需要在该文件中指定 token，我们这里使用token的方式登录

```bash
# 创建service account
$ kubectl create sa dashboard-admin -n kube-system

# 创建角色绑定关系
$ kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin

# 查看dashboard-admin的secret名字
$ ADMIN_SECRET=$(kubectl get secrets -n kube-system | grep dashboard-admin | awk '{print $1}')

# 打印secret的token
$ kubectl describe secret -n kube-system ${ADMIN_SECRET} | grep -E '^token' | awk '{print $2}'
```

### 6、集群运维

#### 1. Master节点

**增加master节点**

```bash
# 1.编辑hosts.yaml，增加master节点配置
$ vi inventory/mycluster/hosts.yaml
# 2.执行cluster.yml（不要用scale.yml）
$ ansible-playbook -i inventory/mycluster/hosts.yaml cluster.yml -b -v
# 3.重启nginx-proxy - 在所有节点执行下面命令重启nginx-proxy
$ docker ps | grep k8s_nginx-proxy_nginx-proxy | awk '{print $1}' | xargs docker restart
```

**删除master节点**

`如果你要删除的是配置文件中第一个节点，需要先调整配置，将第一行配置下移，再重新运行cluster.yml，使其变成非第一行配置。举例如下：`

```bash
# 场景：下线node-1节点
$ vi inventory/mycluster/hosts.yaml
# 变更前的配置
  children:
    kube-master:
      hosts:
        node-1:
        node-2:
        node-3:
# 变更后的配置
  children:
    kube-master:
      hosts:
        node-2:
        node-1:
        node-3:
# 再执行一次cluster.yml
$ ansible-playbook -i inventory/mycluster/hosts.yaml -b cluster.yml
```

非第一行的master节点下线流程：

```bash
# 执行remove-node.yml(不要在hosts.yaml中删除要下线的节点)
$ ansible-playbook -i inventory/mycluster/hosts.yaml remove-node.yml -b -v -e "node=NODE-NAME"
# 同步hosts.yaml（编辑hosts.yaml将下线的节点删除，保持集群状态和配置文件的一致性）
$ vi inventory/mycluster/hosts.yaml
```

#### 2. Worker节点

**增加worker节点**

```bash
# 刷新缓存
$ ansible-playbook -i inventory/mycluster/hosts.yaml facts.yml -b -v
# 修改配置hosts.yaml，增加节点
$ vi inventory/mycluster/hosts.yaml
# 执行scale添加节点，--limit限制只在某个固定节点执行
$ ansible-playbook -i inventory/mycluster/hosts.yaml scale.yml --limit=NODE-NAME -b -v
```

**删除worker节点**

```bash
# 此命令可以下线节点，不影响其他正在运行中的节点，并清理节点上所有的容器以及kubelet，恢复初始状态，多个节点逗号分隔
$ ansible-playbook -i inventory/mycluster/hosts.yaml remove-node.yml -b -v -e "node=NODE-NAME-1,NODE-NAME-2,..."
# 同步hosts.yaml（编辑hosts.yaml将下线的节点删除，保持集群状态和配置文件的一致性）
$ vi inventory/mycluster/hosts.yaml
```

#### 3. ETCD节点

`如果要变更的etcd节点同时也是master或worker节点，需要先将master/worker节点按照前面的文档操作下线，保留纯粹的etcd节点`

##### **增加etcd节点**

```bash
# 编辑hosts.yaml（可以增加1个或2个etcd节点配置）
$ vi inventory/mycluster/hosts.yaml
# 更新etcd集群
$ ansible-playbook -i inventory/mycluster/hosts.yaml upgrade-cluster.yml --limit=etcd,kube-master -e ignore_assert_errors=yes -e etcd_retries=10
```

##### 删除**etcd**节点

```bash
# 执行remove-node.yml(不要在hosts.yaml中删除要下线的节点)
$ ansible-playbook -i inventory/mycluster/hosts.yaml remove-node.yml -b -v -e "node=NODE-NAME"
# 同步hosts.yaml（编辑hosts.yaml将下线的节点删除，保持集群状态和配置文件的一致性）
$ vi inventory/mycluster/hosts.yaml
# 运行cluster.yml给node节点重新生成etcd节点相关的配置
$ ansible-playbook -i inventory/mycluster/hosts.yaml -b cluster.yml
```

#### 4. 其他常用命令

##### 集群reset

```bash
# 运行reset.yml一键清理集群
$ ansible-playbook -i inventory/mycluster/hosts.yaml -b -v reset.yml
```

##### 自定义play起始点

当我们执行play的过程中如果有问题，需要重新的时候，如果重新执行指令会重新经历前面漫长的等待，这个时候“跳过”功能就显得非常有用

```bash
# 通过--start-at-task指定从哪个task处开始执行，会跳过前面的任务，举例如下
$ ansible-playbook --start-at-task="reset | gather mounted kubelet dirs"
```

##### 忽略错误

当有些错误是我们确认可以接受的或误报的，可以配置ignore_errors: true，避免task出现错误后影响整个流程的执行。

```bash
# 示例片段如下：
- name: "Remove physical volume from cluster disks."
  environment:
    PATH: "{{ ansible_env.PATH }}:/sbin"
  become: true
  command: "pvremove {{ disk_volume_device_1 }} --yes"
  ignore_errors: true
```

### 7、集群一键部署 - FAQ

#### 1. google相关的镜像、二进制文件下载失败

##### 现象

错误提示中可以看到诸如google、googleapi等url，提示连接超时

##### 分析

这种一般是没有设置代理（google属于墙外资源，需要代理访问）
编辑配置文件设置http_proxy和https_proxy

#### 2. TASK [container-engine/containerd : ensure containerd packages are installed]失败

##### 现象

执行到这个task会卡住很久，并且多次重试后最终退出

##### 分析

containerd下载文件较大，会耗时比较长，网络不好的情况，会下载超时，导致失败。
可以多尝试几次。如果依然不行可以在每个节点手动下载containerd

```bash
$ yum -d 2 -y --enablerepo=docker-ce install containerd.io-<containerd-version>
```



## 二进制部署

### 1. 环境准备

| 角色   | IP           | 组件                                                         |
| ------ | ------------ | ------------------------------------------------------------ |
| Master | 172.16.1.100 | kube-apiserver,  kubecontroller-manager, kube-scheduler, kubelet, kube-proxy, docker, etcd |
| Node1  | 172.16.1.101 | kubelet, kube-proxy, etcd, docker                            |
| Node2  | 172.16.1.102 | kubelet, kube-proxy, etcd, docker                            |
| Node3  | 172.16.1.103 | kubelet, kube-proxy, docker                                  |

![image-20220107162951576](.\Kubernetes\单节点master架构)

### 2. 系统初始话配置

```bash
# 关闭防火墙 
systemctl stop firewalld 
systemctl disable firewalld 
 
# 关闭selinux 
sed -i 's/SELINX=enforcing/SELINX=disabled/' /etc/selinux/config  # 永久 
setenforce 0  # 临时 
 
# 关闭swap 
swapoff -a  # 临时 
sed -ri 's/.*swap.*/#&/' /etc/fstab    # 永久 
 
# 根据规划设置主机名 
hostnamectl set-hostname Master
hostnamectl set-hostname Node1
hostnamectl set-hostname Node2
hostnamectl set-hostname Node3
 
# 在master添加hosts 
cat >> /etc/hosts << EOF 
172.16.1.100 Master
172.16.1.101 Node1
172.16.1.102 Node2
172.16.1.103 Node3
EOF 
 
# 将桥接的IPv4流量传递到iptables的链 
cat >>/etc/sysctl.d/k8s.conf<<-EOF 
net.bridge.bridge-nf-call-ip6tables = 1 
net.bridge.bridge-nf-call-iptables = 1 
EOF 
sysctl --system  # 生效 
 
# 时间同步 
yum install ntpdate -y 
ntpdate ntp.aliyun.com
```

### 3. 部署Etcd集群

| 节点名称 | IP           |
| -------- | ------------ |
| etcd-1   | 172.16.1.100 |
| etcd-2   | 172.16.1.101 |
| etcd-3   | 172.16.1.102 |

#### 3.1 准备cfssl证书生成工具

>**这里用Master机器统一操作**

```bash
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64

chmod +x cfssl_linux-amd64 cfssljson_linux-amd64 cfssl-certinfo_linux-amd64

mv cfssl_linux-amd64 /usr/local/bin/cfssl
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
mv cfssl-certinfo_linux-amd64 /usr/bin/cfssl-certinfo
```

#### 3.2 生成Etcd证书

创建工作目录

```bash
mkdir -p /root/TLS/{etcd,k8s}
cd /root/TLS/etcd
```

自签CA：

```json
cat > ca-config.json << EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "www": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
EOF

cat > ca-csr.json << EOF
{
    "CN": "etcd CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing"
        }
    ]
}
EOF
```

生成证书：

```bash
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
```

会生成ca.pem和ca-key.pem文件

##### 2.1 使用自签CA签发Etcd HTTPS证书

创建证书申请文件：

```json
cat > server-csr.json << EOF
{
    "CN": "etcd",
    "hosts": [
    "172.16.1.100",
    "172.16.1.101",
    "172.16.1.102"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing"
        }
    ]
}
EOF
```

注：上述文件hosts字段中IP为所有etcd节点的集群内部通信IP，一个都不能少！为了方便后期扩容可以多写几个预留的IP。

生成证书：

```bash
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www server-csr.json | cfssljson -bare server
```

会生成server.pem和server-key.pem文件。

 **从Github下载二进制文件**

下载地址：https://github.com/etcd-io/etcd/releases/download/v3.4.9/etcd-v3.4.9-linux-amd64.tar.gz

#### 3.3 部署Etcd集群

以下在节点1上操作，为简化操作，待会将节点1生成的所有文件拷贝到节点2和节点3.

##### 3.3.1 创建工作目录并解压二进制包

```bash
mkdir /opt/etcd/{bin,cfg,ssl} -p
tar zxvf etcd-v3.4.9-linux-amd64.tar.gz
mv etcd-v3.4.9-linux-amd64/{etcd,etcdctl} /opt/etcd/bin/
```

##### 3.3.2 创建etcd配置文件

```bash
cat > /opt/etcd/cfg/etcd.conf << EOF
#[Member]
ETCD_NAME="etcd-1"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://172.16.1.100:2380"
ETCD_LISTEN_CLIENT_URLS="https://172.16.1.100:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://172.16.1.100:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://172.16.1.100:2379"
ETCD_INITIAL_CLUSTER="etcd-1=https://172.16.1.100:2380,etcd-2=https://172.16.1.101:2380,etcd-3=https://172.16.1.102:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF
```

```yaml
# ETCD_NAME：节点名称，集群中唯一
# ETCD_DATA_DIR：数据目录
# ETCD_LISTEN_PEER_URLS：集群通信监听地址
# ETCD_LISTEN_CLIENT_URLS：客户端访问监听地址
# ETCD_INITIAL_ADVERTISE_PEERURLS：集群通告地址
# ETCD_ADVERTISE_CLIENT_URLS：客户端通告地址
# ETCD_INITIAL_CLUSTER：集群节点地址
# ETCD_INITIALCLUSTER_TOKEN：集群Token
# ETCD_INITIALCLUSTER_STATE：加入集群的当前状态，new是新集群，existing表示加入已有集群
```

##### 3.3.3 systemd管理etcd

```bash
cat > /usr/lib/systemd/system/etcd.service << EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=/opt/etcd/cfg/etcd.conf
ExecStart=/opt/etcd/bin/etcd \
--cert-file=/opt/etcd/ssl/server.pem \
--key-file=/opt/etcd/ssl/server-key.pem \
--peer-cert-file=/opt/etcd/ssl/server.pem \
--peer-key-file=/opt/etcd/ssl/server-key.pem \
--trusted-ca-file=/opt/etcd/ssl/ca.pem \
--peer-trusted-ca-file=/opt/etcd/ssl/ca.pem \
--logger=zap
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

##### 3.3.4 拷贝刚才生成的证书

把刚才生成的证书拷贝到配置文件中的路径：

```bash
cp ~/TLS/etcd/ca*pem ~/TLS/etcd/server*pem /opt/etcd/ssl/
```

##### 3.3.5 启动并设置开机启动

```bash
systemctl daemon-reload
systemctl start etcd
systemctl enable etcd
systemctl status etcd
```

##### 3.3.6 将上面节点1所有生成的文件拷贝到节点2 3

```bash
scp -r /opt/etcd/ root@172.16.1.101:/opt/
scp /usr/lib/systemd/system/etcd.service root@172.16.1.101:/usr/lib/systemd/system/
scp -r /opt/etcd/ root@172.16.1.102:/opt/
scp /usr/lib/systemd/system/etcd.service root@172.16.1.102:/usr/lib/systemd/system/
```

然后在节点2和节点3分别修改etcd.conf配置文件中的节点名称和当前服务器IP：

```bash
vi /opt/etcd/cfg/etcd.conf
#[Member]
ETCD_NAME="etcd-1"   # 修改此处，节点2改为etcd-2，节点3改为etcd-3
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://172.16.1.100:2380"   # 修改此处为当前服务器IP
ETCD_LISTEN_CLIENT_URLS="https://172.16.1.100:2379" # 修改此处为当前服务器IP

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://172.16.1.100:2380" # 修改此处为当前服务器IP
ETCD_ADVERTISE_CLIENT_URLS="https://172.16.1.100:2379" # 修改此处为当前服务器IP
ETCD_INITIAL_CLUSTER="etcd-1=https://172.16.1.100:2380,etcd-2=https://172.16.1.101:2380,etcd-3=https://172.16.1.102:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"


sed -ri '2s/etcd-1/etcd-2/g' /opt/etcd/cfg/etcd.conf
sed -ri '4,9s/172.16.1.100/172.16.1.101/g' /opt/etcd/cfg/etcd.conf
sed -ri '2s/etcd-1/etcd-3/g' /opt/etcd/cfg/etcd.conf
sed -ri '4,9s/172.16.1.100/172.16.1.102/g' /opt/etcd/cfg/etcd.conf
```

最后启动etcd并设置开机启动，同上。

##### 3.3.7 查看集群状态

```bash
ETCDCTL_API=3 /opt/etcd/bin/etcdctl --cacert=/opt/etcd/ssl/ca.pem --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --endpoints="https://172.16.1.100:2379,https://172.16.1.101:2379,https://172.16.1.102:2379" endpoint health --write-out=table

+---------------------------+--------+-------------+-------+
|         ENDPOINT          | HEALTH |    TOOK     | ERROR |
+---------------------------+--------+-------------+-------+
| https://172.16.1.100:2379 |   true |    8.2838ms |       |
| https://172.16.1.102:2379 |   true |  9.110157ms |       |
| https://172.16.1.101:2379 |   true | 13.874302ms |       |
+---------------------------+--------+-------------+-------+
```

如果输出上面信息，就说明集群部署成功。

### 4. 安装Docker

#### 4.1 解压二进制包

```bash
tar zxvf docker-19.03.9.tgz
mv docker/* /usr/bin
```

#### 4.2 systemd管理docker

```bash
cat > /usr/lib/systemd/system/docker.service << EOF
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target

[Service]
Type=notify
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target
EOF
```

#### 4.3 创建配置文件

```json
mkdir /etc/docker
cat > /etc/docker/daemon.json << EOF
{
  "registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"]
}
EOF
```

#### 4.4 启动并设置开机启动

```bash
systemctl daemon-reload
systemctl start docker
systemctl enable docker
systemctl status docker
```

### 5. 部署Master Node

#### 5.1 生成kube-apiserver证书

##### 5.1.1 自签证书颁发机构（CA）

```json
cd ~/TLS/k8s

cat > ca-config.json << EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
EOF

cat > ca-csr.json << EOF
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```

生成证书：

```bash
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
```

会生成ca.pem和ca-key.pem文件。

##### 5.1.2 使用自签CA签发kube-apiserver HTTPS证书

创建证书申请文件：

```json
cat > server-csr.json << EOF
{
    "CN": "kubernetes",
    "hosts": [
      "10.0.0.1",
      "127.0.0.1",
      "172.16.1.100",
      "172.16.1.101",
      "172.16.1.102",
      "172.16.1.103",
      "kubernetes",
      "kubernetes.default",
      "kubernetes.default.svc",
      "kubernetes.default.svc.cluster",
      "kubernetes.default.svc.cluster.local"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```

注：上述文件hosts字段中IP为所有Master/LB/VIP IP，一个都不能少！为了方便后期扩容可以多写几个预留的IP。

生成证书：

```bash
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes server-csr.json | cfssljson -bare server
```

会生成server.pem和server-key.pem文件。

#### 5.2 从Github下载二进制文件

下载地址： https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.20.md

注：打开链接你会发现里面有很多包，下载一个server包就够了，包含了Master和Worker Node二进制文件。

##### 5.2.1 解压二进制包

```bash
mkdir -p /opt/kubernetes/{bin,cfg,ssl,logs} 
tar zxvf kubernetes-server-linux-amd64.tar.gz
cd kubernetes/server/bin
cp kube-apiserver kube-scheduler kube-controller-manager /opt/kubernetes/bin
cp kubectl /usr/bin/
```

#### 5.4 部署kube-apiserver

##### 5.4.1 创建配置文件

```bash
cat > /opt/kubernetes/cfg/kube-apiserver.conf << EOF
KUBE_APISERVER_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--etcd-servers=https://172.16.1.100:2379,https://172.16.1.101:2379,https://172.16.1.102:2379 \\
--bind-address=172.16.1.100 \\
--secure-port=6443 \\
--advertise-address=172.16.1.100 \\
--allow-privileged=true \\
--service-cluster-ip-range=10.0.0.0/24 \\
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction \\
--authorization-mode=RBAC,Node \\
--enable-bootstrap-token-auth=true \\
--token-auth-file=/opt/kubernetes/cfg/token.csv \\
--service-node-port-range=30000-32767 \\
--kubelet-client-certificate=/opt/kubernetes/ssl/server.pem \\
--kubelet-client-key=/opt/kubernetes/ssl/server-key.pem \\
--tls-cert-file=/opt/kubernetes/ssl/server.pem  \\
--tls-private-key-file=/opt/kubernetes/ssl/server-key.pem \\
--client-ca-file=/opt/kubernetes/ssl/ca.pem \\
--service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--service-account-issuer=api \\
--service-account-signing-key-file=/opt/kubernetes/ssl/server-key.pem \\
--etcd-cafile=/opt/etcd/ssl/ca.pem \\
--etcd-certfile=/opt/etcd/ssl/server.pem \\
--etcd-keyfile=/opt/etcd/ssl/server-key.pem \\
--requestheader-client-ca-file=/opt/kubernetes/ssl/ca.pem \\
--proxy-client-cert-file=/opt/kubernetes/ssl/server.pem \\
--proxy-client-key-file=/opt/kubernetes/ssl/server-key.pem \\
--requestheader-allowed-names=kubernetes \\
--requestheader-extra-headers-prefix=X-Remote-Extra- \\
--requestheader-group-headers=X-Remote-Group \\
--requestheader-username-headers=X-Remote-User \\
--enable-aggregator-routing=true \\
--audit-log-maxage=30 \\
--audit-log-maxbackup=3 \\
--audit-log-maxsize=100 \\
--audit-log-path=/opt/kubernetes/logs/k8s-audit.log"
EOF
```

```yaml
注：上面两个\ \ 第一个是转义符，第二个是换行符，使用转义符是为了使用EOF保留换行符。
# --logtostderr：启用日志
# ---v：日志等级
# --log-dir：日志目录
# --etcd-servers：etcd集群地址
# --bind-address：监听地址
# --secure-port：https安全端口
# --advertise-address：集群通告地址
# --allow-privileged：启用授权
#  --service-cluster-ip-range：Service虚拟IP地址段
# --enable-admission-plugins：准入控制模块
# --authorization-mode：认证授权，启用RBAC授权和节点自管理
# --enable-bootstrap-token-auth：启用TLS bootstrap机制
# --token-auth-file：bootstrap token文件
# --service-node-port-range：Service nodeport类型默认分配端口范围
# --kubelet-client-xxx：apiserver访问kubelet客户端证书
# --tls-xxx-file：apiserver https证书

# 1.20版本必须加的参数：--service-account-issuer，--service-account-signing-key-file
# --etcd-xxxfile：连接Etcd集群证书
# --audit-log-xxx：审计日志
# 启动聚合层相关配置：--requestheader-client-ca-file，--proxy-client-cert-file，--proxy-client-key-file，--requestheader-allowed-names，--requestheader-extra-headers-prefix，--requestheader-group-headers，--requestheader-username-headers，--enable-aggregator-routing
```

##### 5.4.2 拷贝刚才生成的证书

把刚才生成的证书拷贝到配置文件中的路径：

```bash
cp ~/TLS/k8s/ca*pem ~/TLS/k8s/server*pem /opt/kubernetes/ssl/
```

##### 5.4.3 启用 TLS Bootstrapping 机制

创建配置文件中token文件：

```bash
cat > /opt/kubernetes/cfg/token.csv << EOF
c47ffb939f5ca36231d9e3121a252940,kubelet-bootstrap,10001,"system:node-bootstrapper"
EOF
```

格式：token，用户名，UID，用户组

token也可自行生成替换：

```bash
head -c 16 /dev/urandom | od -An -t x | tr -d ' '
```

##### 5.4.4 systemd管理apiserver

```bash
cat > /usr/lib/systemd/system/kube-apiserver.service << EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-apiserver.conf
ExecStart=/opt/kubernetes/bin/kube-apiserver \$KUBE_APISERVER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

##### 5.4.5 启动并设置开机启动

```bash
systemctl daemon-reload
systemctl start kube-apiserver 
systemctl enable kube-apiserver
systemctl status kube-apiserver
```

#### 5.5 部署kube-controller-manager

##### 5.5.1 创建配置文件

```bash
cat > /opt/kubernetes/cfg/kube-controller-manager.conf << EOF
KUBE_CONTROLLER_MANAGER_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--leader-elect=true \\
--kubeconfig=/opt/kubernetes/cfg/kube-controller-manager.kubeconfig \\
--bind-address=127.0.0.1 \\
--allocate-node-cidrs=true \\
--cluster-cidr=10.244.0.0/16 \\
--service-cluster-ip-range=10.0.0.0/24 \\
--cluster-signing-cert-file=/opt/kubernetes/ssl/ca.pem \\
--cluster-signing-key-file=/opt/kubernetes/ssl/ca-key.pem  \\
--root-ca-file=/opt/kubernetes/ssl/ca.pem \\
--service-account-private-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--cluster-signing-duration=87600h0m0s"
EOF
```

```yaml
# --kubeconfig：连接apiserver配置文件
# --leader-elect：当该组件启动多个时，自动选举（HA）
# --cluster-signing-cert-file/--cluster-signing-key-file：自动为kubelet颁发证书的CA，与apiserver保持一致
```

##### 5.5.2 生成kubeconfig文件

生成kube-controller-manager证书：

```bash
# 切换工作目录
cd ~/TLS/k8s

# 创建证书请求文件
cat > kube-controller-manager-csr.json << EOF
{
  "CN": "system:kube-controller-manager",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing", 
      "ST": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF

# 生成证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
```

生成kubeconfig文件（以下是shell命令，直接在终端执行）：

```bash
KUBE_CONFIG="/opt/kubernetes/cfg/kube-controller-manager.kubeconfig"
KUBE_APISERVER="https://172.16.1.100:6443"

kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}
kubectl config set-credentials kube-controller-manager \
  --client-certificate=./kube-controller-manager.pem \
  --client-key=./kube-controller-manager-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-controller-manager \
  --kubeconfig=${KUBE_CONFIG}
kubectl config use-context default --kubeconfig=${KUBE_CONFIG}
```

##### 5.5.3 systemd管理controller-manager

```bash
cat > /usr/lib/systemd/system/kube-controller-manager.service << EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-controller-manager.conf
ExecStart=/opt/kubernetes/bin/kube-controller-manager \$KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

##### 5.5.4 启动并设置开机启动

```bash
systemctl daemon-reload
systemctl start kube-controller-manager
systemctl enable kube-controller-manager
systemctl status kube-controller-manager
```

#### 5.6 部署kube-scheduler

##### 5.6.1 创建配置文件

```bash
cat > /opt/kubernetes/cfg/kube-scheduler.conf << EOF
KUBE_SCHEDULER_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--leader-elect \\
--kubeconfig=/opt/kubernetes/cfg/kube-scheduler.kubeconfig \\
--bind-address=127.0.0.1"
EOF
```

```yaml
# --kubeconfig：连接apiserver配置文件
# --leader-elect：当该组件启动多个时，自动选举（HA）
```

##### 5.6.2 生成kubeconfig文件

生成kube-scheduler证书：

```bash
# 切换工作目录
cd ~/TLS/k8s

# 创建证书请求文件
cat > kube-scheduler-csr.json << EOF
{
  "CN": "system:kube-scheduler",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing",
      "ST": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF

# 生成证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-scheduler-csr.json | cfssljson -bare kube-scheduler
```

生成kubeconfig文件：

```bash
KUBE_CONFIG="/opt/kubernetes/cfg/kube-scheduler.kubeconfig"
KUBE_APISERVER="https://172.16.1.100:6443"

kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}
kubectl config set-credentials kube-scheduler \
  --client-certificate=./kube-scheduler.pem \
  --client-key=./kube-scheduler-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-scheduler \
  --kubeconfig=${KUBE_CONFIG}
kubectl config use-context default --kubeconfig=${KUBE_CONFIG}
```

##### 5.6.3 systemd管理scheduler

```bash
cat > /usr/lib/systemd/system/kube-scheduler.service << EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-scheduler.conf
ExecStart=/opt/kubernetes/bin/kube-scheduler \$KUBE_SCHEDULER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

##### 5.6.4 启动并设置开机启动

```bash
systemctl daemon-reload
systemctl start kube-scheduler
systemctl enable kube-scheduler
systemctl status kube-scheduler
```

##### 5.6.5 查看集群状态

生成kubectl连接集群的证书：

```bash
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing",
      "ST": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF

# 生成证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
```

生成kubeconfig文件：

```bash
mkdir /root/.kube
KUBE_CONFIG="/root/.kube/config"
KUBE_APISERVER="https://172.16.1.100:6443"

kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}
kubectl config set-credentials cluster-admin \
  --client-certificate=./admin.pem \
  --client-key=./admin-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}
kubectl config set-context default \
  --cluster=kubernetes \
  --user=cluster-admin \
  --kubeconfig=${KUBE_CONFIG}
kubectl config use-context default --kubeconfig=${KUBE_CONFIG}
```

通过kubectl工具查看当前集群组件状态：

```bash
kubectl get cs
NAME                STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-2               Healthy   {"health":"true"}   
etcd-1               Healthy   {"health":"true"}   
etcd-0               Healthy   {"health":"true"}  
```

如上输出说明Master节点组件运行正常。

##### 5.6.6 授权kubelet-bootstrap用户允许请求证书

```bash
kubectl create clusterrolebinding kubelet-bootstrap \
--clusterrole=system:node-bootstrapper \
--user=kubelet-bootstrap
```

### 6. 部署Worker Node

**下面还是在Master Node上操作，即同时作为Worker Node**

#### 6.1 创建工作目录并拷贝二进制文件

在所有worker node创建工作目录：

```bash
mkdir -p /opt/kubernetes/{bin,cfg,ssl,logs} 
```

从master节点拷贝：

```bash
cd kubernetes/server/bin
cp kubelet kube-proxy /opt/kubernetes/bin   # 本地拷贝
```

#### 6.2 部署kubelet

##### 6.2.1 创建配置文件

```bash
cat > /opt/kubernetes/cfg/kubelet.conf << EOF
KUBELET_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--hostname-override=k8s-master \\
--network-plugin=cni \\
--kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig \\
--bootstrap-kubeconfig=/opt/kubernetes/cfg/bootstrap.kubeconfig \\
--config=/opt/kubernetes/cfg/kubelet-config.yml \\
--cert-dir=/opt/kubernetes/ssl \\
--pod-infra-container-image=lizhenliang/pause-amd64:3.0"
EOF
```

```yaml
# --hostname-override：显示名称，集群中唯一
# --network-plugin：启用CNI
# --kubeconfig：空路径，会自动生成，后面用于连接apiserver
# --bootstrap-kubeconfig：首次启动向apiserver申请证书
# --config：配置参数文件
# --cert-dir：kubelet证书生成目录
# --pod-infra-container-image：管理Pod网络容器的镜像
```

##### 6.2.2 配置参数文件

```yaml
cat > /opt/kubernetes/cfg/kubelet-config.yml << EOF
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: 0.0.0.0
port: 10250
readOnlyPort: 10255
cgroupDriver: cgroupfs
clusterDNS:
- 10.0.0.2
clusterDomain: cluster.local 
failSwapOn: false
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /opt/kubernetes/ssl/ca.pem 
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
maxOpenFiles: 1000000
maxPods: 110
EOF
```

##### 6.2.3 生成kubelet初次加入集群引导kubeconfig文件

```bash
KUBE_CONFIG="/opt/kubernetes/cfg/bootstrap.kubeconfig"
KUBE_APISERVER="https://172.16.1.100:6443" # apiserver IP:PORT
TOKEN="c47ffb939f5ca36231d9e3121a252940" # 与token.csv里保持一致

# 生成 kubelet bootstrap kubeconfig 配置文件
kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}
kubectl config set-credentials "kubelet-bootstrap" \
  --token=${TOKEN} \
  --kubeconfig=${KUBE_CONFIG}
kubectl config set-context default \
  --cluster=kubernetes \
  --user="kubelet-bootstrap" \
  --kubeconfig=${KUBE_CONFIG}
kubectl config use-context default --kubeconfig=${KUBE_CONFIG}
```

##### 6.2.4 systemd管理kubelet

```bash
cat > /usr/lib/systemd/system/kubelet.service << EOF
[Unit]
Description=Kubernetes Kubelet
After=docker.service

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kubelet.conf
ExecStart=/opt/kubernetes/bin/kubelet \$KUBELET_OPTS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

##### 6.2.5 启动并设置开机启动

```bash
systemctl daemon-reload
systemctl start kubelet
systemctl enable kubelet
systemctl status kubelet
```

##### 6.2.6 批准kubelet证书申请并加入集群

```bash
# 查看kubelet证书请求
kubectl get csr
NAME                                                   AGE    SIGNERNAME                                    REQUESTOR           CONDITION
node-csr-BLd7-i6xZWyQeTrlb-VRPEbDPblhvYOjj4QyY7VlQdI   5s    kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending

# 批准申请
kubectl certificate approve node-csr-BLd7-i6xZWyQeTrlb-VRPEbDPblhvYOjj4QyY7VlQdI

# 查看节点
kubectl get node
NAME         STATUS     ROLES    AGE   VERSION
k8s-master1   NotReady   <none>   7s   v1.20.13
```

注：由于网络插件还没有部署，节点会没有准备就绪 NotReady

#### 6.3 部署kube-proxy

##### 6.3.1 创建配置文件

```bash
cat > /opt/kubernetes/cfg/kube-proxy.conf << EOF
KUBE_PROXY_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--config=/opt/kubernetes/cfg/kube-proxy-config.yml"
EOF
```

##### 6.3.2 配置参数文件

```yaml
cat > /opt/kubernetes/cfg/kube-proxy-config.yml << EOF
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
metricsBindAddress: 0.0.0.0:10249
clientConnection:
  kubeconfig: /opt/kubernetes/cfg/kube-proxy.kubeconfig
hostnameOverride: k8s-master1
clusterCIDR: 10.244.0.0/16
EOF
```

##### 6.3.3 生成kube-proxy.kubeconfig文件

```bash
# 切换工作目录
cd ~/TLS/k8s

# 创建证书请求文件
cat > kube-proxy-csr.json << EOF
{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing",
      "ST": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF

# 生成证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy

# 生成kubeconfig文件：
KUBE_CONFIG="/opt/kubernetes/cfg/kube-proxy.kubeconfig"
KUBE_APISERVER="https://172.16.1.100:6443"

kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}
kubectl config set-credentials kube-proxy \
  --client-certificate=./kube-proxy.pem \
  --client-key=./kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=${KUBE_CONFIG}
kubectl config use-context default --kubeconfig=${KUBE_CONFIG}
```

##### 6.3.4 systemd管理kube-proxy

```bash
cat > /usr/lib/systemd/system/kube-proxy.service << EOF
[Unit]
Description=Kubernetes Proxy
After=network.target

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-proxy.conf
ExecStart=/opt/kubernetes/bin/kube-proxy \$KUBE_PROXY_OPTS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

##### 6.3.5 启动并设置开机启动

```bash
systemctl daemon-reload
systemctl start kube-proxy
systemctl enable kube-proxy
systemctl status kube-proxy
```

#### 6.4 部署网络组件CNI

Calico是一个纯三层的数据中心网络方案，是目前Kubernetes主流的网络方案。

部署Calico：

```bash
kubectl apply -f calico.yaml
kubectl get pods -n kube-system
```

等Calico Pod都Running，节点也会准备就绪：

```bash
kubectl get node
NAME         STATUS   ROLES    AGE   VERSION
k8s-master   Ready    <none>   37m   v1.20.13
```

> 部署calico之后，status是running，但是ready始终不行，错误如下

![calico-error](.\Kubernetes\calico-error.png)

>解决：通过ip link查看有多余的一张无用的tunl0网卡，将这个网卡移除
>
>lsmod | grep ipip  //找到是否存在ipip模块
>
>modprobe -r ipip  //移除ipip模块，tunl0就消失了
>
>重新applycalico.yaml文件，即可初始化成功

![calico-error2](E:\linux笔记\Linux\Kubernetes\calico-error2.png)

![calico-error3](E:\linux笔记\Linux\Kubernetes\calico-error3.png)

#### 6.5 授权apiserver访问kubelet

```yaml
cat > apiserver-to-kubelet-rbac.yaml << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
      - pods/log
    verbs:
      - "*"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF

kubectl apply -f apiserver-to-kubelet-rbac.yaml
```

#### 6.6 新增加Worker Node

##### 6.6.1 拷贝已部署好的Node相关文件到新节点

在Master节点将Worker Node涉及文件拷贝到新节点172.16.1.101/102/103

```bash
scp -r /opt/kubernetes root@172.16.1.101:/opt/
scp -r /usr/lib/systemd/system/{kubelet,kube-proxy}.service root@172.16.1.101:/usr/lib/systemd/system
scp /opt/kubernetes/ssl/ca.pem root@172.16.1.101:/opt/kubernetes/ssl

scp -r /opt/kubernetes root@172.16.1.102:/opt/
scp -r /usr/lib/systemd/system/{kubelet,kube-proxy}.service root@172.16.1.102:/usr/lib/systemd/system
scp /opt/kubernetes/ssl/ca.pem root@172.16.1.102:/opt/kubernetes/ssl

scp -r /opt/kubernetes root@172.16.1.103:/opt/
scp -r /usr/lib/systemd/system/{kubelet,kube-proxy}.service root@172.16.1.103:/usr/lib/systemd/system
scp /opt/kubernetes/ssl/ca.pem root@172.16.1.103:/opt/kubernetes/ssl
```

##### 6.6.2 删除kubelet证书和kubeconfig文件

```bash
rm -f /opt/kubernetes/cfg/kubelet.kubeconfig 
rm -f /opt/kubernetes/ssl/kubelet*
```

注：这几个文件是证书申请审批后自动生成的，每个Node不同，必须删除

##### 6.6.3 修改主机名

```bash
vi /opt/kubernetes/cfg/kubelet.conf
--hostname-override=k8s-node1

vi /opt/kubernetes/cfg/kube-proxy-config.yml
hostnameOverride: k8s-node1
```

##### 6.6.4 启动并设置开机启动

```bash
systemctl daemon-reload
systemctl start kubelet kube-proxy
systemctl enable kubelet kube-proxy
systemctl status kubelet kube-proxy
```

##### 6.6.5 在Master上批准新Node kubelet证书申请

```bash
# 查看证书请求
kubectl get csr
NAME           AGE   SIGNERNAME                    REQUESTOR           CONDITION
node-csr-9A3euvYXMLK42ruK_zsMiZ49GxtvjmIgBPqtLqoe4Y8   19s    kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
node-csr-BLd7-i6xZWyQeTrlb-VRPEbDPblhvYOjj4QyY7VlQdI   6m1s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Approved,Issued
node-csr-IFUpUHGiq82EbotrSpTmNfYSI8K85Kd49uTXlLcStAc   19s    kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
node-csr-SiCATjcGKZq6KwV9RBFumM_Pg-drOIFMKq63OJ5D22Q   19s    kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending


# 授权请求
kubectl certificate approve node-csr-4zTjsaVSrhuyhIGqsefxzVoZDCNKei-aE2jyTP81Uro
```

##### 6.6.6 查看Node状态

```bash
kubectl get node
NAME       STATUS   ROLES    AGE     VERSION
k8s-master1   Ready    <none>   47m    v1.20.13
k8s-node1    Ready    <none>   6m49s   v1.20.13
```

Node2（172.16.1.102）节点同上。记得修改主机名！

Node3（172.16.1.103 ）节点同上。记得修改主机名！

```bash
NAME          STATUS   ROLES    AGE    VERSION
k8s-master1   Ready    <none>   3h2m   v1.20.13
k8s-node1     Ready    <none>   170m   v1.20.13
k8s-node2     Ready    <none>   170m   v1.20.13
k8s-node3     Ready    <none>   170m   v1.20.13
```

### 7. 部署CoreDNS

**CoreDNS.yaml**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: coredns
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:coredns
rules:
  - apiGroups:
    - ""
    resources:
    - endpoints
    - services
    - pods
    - namespaces
    verbs:
    - list
    - watch
  - apiGroups:
    - discovery.k8s.io
    resources:
    - endpointslices
    verbs:
    - list
    - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:coredns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:coredns
subjects:
- kind: ServiceAccount
  name: coredns
  namespace: kube-system
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
          lameduck 5s
        }
        ready
        kubernetes CLUSTER_DOMAIN REVERSE_CIDRS {
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . UPSTREAMNAMESERVER {
          max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }STUBDOMAINS
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/name: "CoreDNS"
spec:
  # replicas: not specified here:
  # 1. Default is 1.
  # 2. Will be tuned in real time if DNS horizontal auto-scaling is turned on.
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  selector:
    matchLabels:
      k8s-app: kube-dns
  template:
    metadata:
      labels:
        k8s-app: kube-dns
    spec:
      priorityClassName: system-cluster-critical
      serviceAccountName: coredns
      tolerations:
        - key: "CriticalAddonsOnly"
          operator: "Exists"
      nodeSelector:
        kubernetes.io/os: linux
      affinity:
         podAntiAffinity:
           preferredDuringSchedulingIgnoredDuringExecution:
           - weight: 100
             podAffinityTerm:
               labelSelector:
                 matchExpressions:
                   - key: k8s-app
                     operator: In
                     values: ["kube-dns"]
               topologyKey: kubernetes.io/hostname
      containers:
      - name: coredns
        image: coredns/coredns:1.8.6
        imagePullPolicy: IfNotPresent
        resources:
          limits:
            memory: 170Mi
          requests:
            cpu: 100m
            memory: 70Mi
        args: [ "-conf", "/etc/coredns/Corefile" ]
        volumeMounts:
        - name: config-volume
          mountPath: /etc/coredns
          readOnly: true
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        - containerPort: 9153
          name: metrics
          protocol: TCP
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            add:
            - NET_BIND_SERVICE
            drop:
            - all
          readOnlyRootFilesystem: true
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
        readinessProbe:
          httpGet:
            path: /ready
            port: 8181
            scheme: HTTP
      dnsPolicy: Default
      volumes:
        - name: config-volume
          configMap:
            name: coredns
            items:
            - key: Corefile
              path: Corefile
---
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  annotations:
    prometheus.io/port: "9153"
    prometheus.io/scrape: "true"
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "CoreDNS"
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: 10.0.0.2   # 填写集群svc的ip段中的ip
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
  - name: metrics
    port: 9153
    protocol: TCP
```



### 8. 部署管理平台（KubeSphere）

#### 8.1 准备工作

```bash
# 安装kubesphere需要事先准备好默认存储storageclass
1、先安装nfs软件
yum -y install nfs-utils
```

通过yaml文件建立sc存储

**nfs-rabc.yaml**

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-provisioner
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
   name: nfs-provisioner-runner
rules:
   -  apiGroups: [""]
      resources: ["persistentvolumes"]
      verbs: ["get", "list", "watch", "create", "delete"]
   -  apiGroups: [""]
      resources: ["persistentvolumeclaims"]
      verbs: ["get", "list", "watch", "update"]
   -  apiGroups: ["storage.k8s.io"]
      resources: ["storageclasses"]
      verbs: ["get", "list", "watch"]
   -  apiGroups: [""]
      resources: ["events"]
      verbs: ["watch", "create", "update", "patch"]
   -  apiGroups: [""]
      resources: ["services", "endpoints"]
      verbs: ["get","create","list", "watch","update"]
   -  apiGroups: ["extensions"]
      resources: ["podsecuritypolicies"]
      resourceNames: ["nfs-provisioner"]
      verbs: ["use"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-provisioner
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
#vi nfs-deployment.yaml；创建nfs-client的授权
kind: Deployment
apiVersion: apps/v1
metadata:
   name: nfs-client-provisioner
spec:
   replicas: 1
   strategy:
     type: Recreate
   selector:
     matchLabels:
        app: nfs-client-provisioner
   template:
      metadata:
         labels:
            app: nfs-client-provisioner
      spec:
         serviceAccount: nfs-provisioner
         containers:
           - name: nfs-client-provisioner
             image: quay.io/external_storage/nfs-client-provisioner:latest
             volumeMounts:
               - name: nfs-client-root
                 mountPath:  /persistentvolumes
             env:
               - name: PROVISIONER_NAME #供应者的名字
                 value: storage.pri/nfs #名字虽然可以随便起，以后引用要一致
               - name: NFS_SERVER
                 value: 172.16.1.100 # 填写提供存储的服务器ip
               - name: NFS_PATH
                 value: /nfs/data
         volumes:
           - name: nfs-client-root
             nfs:
               server: 172.16.1.100  # 填写提供存储的服务器ip
               path: /nfs/data
```

**storageclass-nfs.yaml**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: storage-nfs
provisioner: storage.pri/nfs
reclaimPolicy: Delete
```

#### 8.2 建立storageclass存储

```bash
# 执行命令创建sc
kubectl apply -f nfs-rabc.yaml
kubectl apply -f storageclass-nfs.yaml
```

![kubesphere-storage](.\Kubernetes\kubesphere-storage.png)

#### 8.3 修改kubesphere配置文件

**cluster-configuration.yaml**

```yaml
---
apiVersion: installer.kubesphere.io/v1alpha1
kind: ClusterConfiguration
metadata:
  name: ks-installer
  namespace: kubesphere-system
  labels:
    version: v3.2.1
spec:
  persistence:
    storageClass: "storage-nfs"  # 将建立的sc的名字填入
  authentication:
    jwtSecret: ""           # Keep the jwtSecret consistent with the Host Cluster. Retrieve the jwtSecret by executing "kubectl -n kubesphere-system get cm kubesphere-config -o yaml | grep -v "apiVersion" | grep jwtSecret" on the Host Cluster.
  local_registry: ""        # Add your private registry address if it is needed.
  # dev_tag: ""               # Add your kubesphere image tag you want to install, by default it's same as ks-install release version.
  etcd:
    monitoring: false       # Enable or disable etcd monitoring dashboard installation. You have to create a Secret for etcd before you enable it.
    endpointIps: localhost  # etcd cluster EndpointIps. It can be a bunch of IPs here.
    port: 2379              # etcd port.
    tlsEnable: true
  common:
    core:
      console:
        enableMultiLogin: true  # Enable or disable simultaneous logins. It allows different users to log in with the same account at the same time.
        port: 30880
        type: NodePort
    # apiserver:            # Enlarge the apiserver and controller manager's resource requests and limits for the large cluster
    #  resources: {}
    # controllerManager:
    #  resources: {}
    redis:
      enabled: false
      volumeSize: 2Gi # Redis PVC size.
    openldap:
      enabled: false
      volumeSize: 2Gi   # openldap PVC size.
    minio:
      volumeSize: 20Gi # Minio PVC size.
    monitoring:
      # type: external   # Whether to specify the external prometheus stack, and need to modify the endpoint at the next line.
      endpoint: http://prometheus-operated.kubesphere-monitoring-system.svc:9090 # Prometheus endpoint to get metrics data.
      GPUMonitoring:     # Enable or disable the GPU-related metrics. If you enable this switch but have no GPU resources, Kubesphere will set it to zero. 
        enabled: false
    gpu:                 # Install GPUKinds. The default GPU kind is nvidia.com/gpu. Other GPU kinds can be added here according to your needs. 
      kinds:         
      - resourceName: "nvidia.com/gpu"
        resourceType: "GPU"
        default: true
    es:   # Storage backend for logging, events and auditing.
      # master:
      #   volumeSize: 4Gi  # The volume size of Elasticsearch master nodes.
      #   replicas: 1      # The total number of master nodes. Even numbers are not allowed.
      #   resources: {}
      # data:
      #   volumeSize: 20Gi  # The volume size of Elasticsearch data nodes.
      #   replicas: 1       # The total number of data nodes.
      #   resources: {}
      logMaxAge: 7             # Log retention time in built-in Elasticsearch. It is 7 days by default.
      elkPrefix: logstash      # The string making up index names. The index name will be formatted as ks-<elk_prefix>-log.
      basicAuth:
        enabled: false
        username: ""
        password: ""
      externalElasticsearchUrl: ""
      externalElasticsearchPort: ""
  alerting:                # (CPU: 0.1 Core, Memory: 100 MiB) It enables users to customize alerting policies to send messages to receivers in time with different time intervals and alerting levels to choose from.
    enabled: false         # Enable or disable the KubeSphere Alerting System.
    # thanosruler:
    #   replicas: 1
    #   resources: {}
  auditing:                # Provide a security-relevant chronological set of records，recording the sequence of activities happening on the platform, initiated by different tenants.
    enabled: false         # Enable or disable the KubeSphere Auditing Log System.
    # operator:
    #   resources: {}
    # webhook:
    #   resources: {}
  devops:                  # (CPU: 0.47 Core, Memory: 8.6 G) Provide an out-of-the-box CI/CD system based on Jenkins, and automated workflow tools including Source-to-Image & Binary-to-Image.
    enabled: false             # Enable or disable the KubeSphere DevOps System.
    # resources: {}
    jenkinsMemoryLim: 2Gi      # Jenkins memory limit.
    jenkinsMemoryReq: 1500Mi   # Jenkins memory request.
    jenkinsVolumeSize: 8Gi     # Jenkins volume size.
    jenkinsJavaOpts_Xms: 512m  # The following three fields are JVM parameters.
    jenkinsJavaOpts_Xmx: 512m
    jenkinsJavaOpts_MaxRAM: 2g
  events:                  # Provide a graphical web console for Kubernetes Events exporting, filtering and alerting in multi-tenant Kubernetes clusters.
    enabled: false         # Enable or disable the KubeSphere Events System.
    # operator:
    #   resources: {}
    # exporter:
    #   resources: {}
    # ruler:
    #   enabled: true
    #   replicas: 2
    #   resources: {}
  logging:                 # (CPU: 57 m, Memory: 2.76 G) Flexible logging functions are provided for log query, collection and management in a unified console. Additional log collectors can be added, such as Elasticsearch, Kafka and Fluentd.
    enabled: false         # Enable or disable the KubeSphere Logging System.
    containerruntime: docker
    logsidecar:
      enabled: true
      replicas: 2
      # resources: {}
  metrics_server:                    # (CPU: 56 m, Memory: 44.35 MiB) It enables HPA (Horizontal Pod Autoscaler).
    enabled: false                   # Enable or disable metrics-server.
  monitoring:
    storageClass: ""                 # If there is an independent StorageClass you need for Prometheus, you can specify it here. The default StorageClass is used by default.
    # kube_rbac_proxy:
    #   resources: {}
    # kube_state_metrics:
    #   resources: {}
    # prometheus:
    #   replicas: 1  # Prometheus replicas are responsible for monitoring different segments of data source and providing high availability.
    #   volumeSize: 20Gi  # Prometheus PVC size.
    #   resources: {}
    #   operator:
    #     resources: {}
    #   adapter:
    #     resources: {}
    # node_exporter:
    #   resources: {}
    # alertmanager:
    #   replicas: 1          # AlertManager Replicas.
    #   resources: {}
    # notification_manager:
    #   resources: {}
    #   operator:
    #     resources: {}
    #   proxy:
    #     resources: {}
    gpu:                           # GPU monitoring-related plug-in installation. 
      nvidia_dcgm_exporter:        # Ensure that gpu resources on your hosts can be used normally, otherwise this plug-in will not work properly.
        enabled: false             # Check whether the labels on the GPU hosts contain "nvidia.com/gpu.present=true" to ensure that the DCGM pod is scheduled to these nodes.
        # resources: {}
  multicluster:
    clusterRole: none  # host | member | none  # You can install a solo cluster, or specify it as the Host or Member Cluster.
  network:
    networkpolicy: # Network policies allow network isolation within the same cluster, which means firewalls can be set up between certain instances (Pods).
      # Make sure that the CNI network plugin used by the cluster supports NetworkPolicy. There are a number of CNI network plugins that support NetworkPolicy, including Calico, Cilium, Kube-router, Romana and Weave Net.
      enabled: false # Enable or disable network policies.
    ippool: # Use Pod IP Pools to manage the Pod network address space. Pods to be created can be assigned IP addresses from a Pod IP Pool.
      type: none # Specify "calico" for this field if Calico is used as your CNI plugin. "none" means that Pod IP Pools are disabled.
    topology: # Use Service Topology to view Service-to-Service communication based on Weave Scope.
      type: none # Specify "weave-scope" for this field to enable Service Topology. "none" means that Service Topology is disabled.
  openpitrix: # An App Store that is accessible to all platform tenants. You can use it to manage apps across their entire lifecycle.
    store:
      enabled: false # Enable or disable the KubeSphere App Store.
  servicemesh:         # (0.3 Core, 300 MiB) Provide fine-grained traffic management, observability and tracing, and visualized traffic topology.
    enabled: false     # Base component (pilot). Enable or disable KubeSphere Service Mesh (Istio-based).
  kubeedge:          # Add edge nodes to your cluster and deploy workloads on edge nodes.
    enabled: false   # Enable or disable KubeEdge.
    cloudCore:
      nodeSelector: {"node-role.kubernetes.io/worker": ""}
      tolerations: []
      cloudhubPort: "10000"
      cloudhubQuicPort: "10001"
      cloudhubHttpsPort: "10002"
      cloudstreamPort: "10003"
      tunnelPort: "10004"
      cloudHub:
        advertiseAddress: # At least a public IP address or an IP address which can be accessed by edge nodes must be provided.
          - ""            # Note that once KubeEdge is enabled, CloudCore will malfunction if the address is not provided.
        nodeLimit: "100"
      service:
        cloudhubNodePort: "30000"
        cloudhubQuicNodePort: "30001"
        cloudhubHttpsNodePort: "30002"
        cloudstreamNodePort: "30003"
        tunnelNodePort: "30004"
    edgeWatcher:
      nodeSelector: {"node-role.kubernetes.io/worker": ""}
      tolerations: []
      edgeWatcherAgent:
        nodeSelector: {"node-role.kubernetes.io/worker": ""}
        tolerations: []
```

**kubesphere-installer.yaml**

```yaml
---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: clusterconfigurations.installer.kubesphere.io
spec:
  group: installer.kubesphere.io
  versions:
    - name: v1alpha1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              x-kubernetes-preserve-unknown-fields: true
            status:
              type: object
              x-kubernetes-preserve-unknown-fields: true
  scope: Namespaced
  names:
    plural: clusterconfigurations
    singular: clusterconfiguration
    kind: ClusterConfiguration
    shortNames:
      - cc

---
apiVersion: v1
kind: Namespace
metadata:
  name: kubesphere-system

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ks-installer
  namespace: kubesphere-system

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: ks-installer
rules:
- apiGroups:
  - ""
  resources:
  - '*'
  verbs:
  - '*'
- apiGroups:
  - apps
  resources:
  - '*'
  verbs:
  - '*'
- apiGroups:
  - extensions
  resources:
  - '*'
  verbs:
  - '*'
- apiGroups:
  - batch
  resources:
  - '*'
  verbs:
  - '*'
- apiGroups:
  - rbac.authorization.k8s.io
  resources:
  - '*'
  verbs:
  - '*'
- apiGroups:
  - apiregistration.k8s.io
  resources:
  - '*'
  verbs:
  - '*'
- apiGroups:
  - apiextensions.k8s.io
  resources:
  - '*'
  verbs:
  - '*'
- apiGroups:
  - tenant.kubesphere.io
  resources:
  - '*'
  verbs:
  - '*'
- apiGroups:
  - certificates.k8s.io
  resources:
  - '*'
  verbs:
  - '*'
- apiGroups:
  - devops.kubesphere.io
  resources:
  - '*'
  verbs:
  - '*'
- apiGroups:
  - monitoring.coreos.com
  resources:
  - '*'
  verbs:
  - '*'
- apiGroups:
  - logging.kubesphere.io
  resources:
  - '*'
  verbs:
  - '*'
- apiGroups:
  - jaegertracing.io
  resources:
  - '*'
  verbs:
  - '*'
- apiGroups:
  - storage.k8s.io
  resources:
  - '*'
  verbs:
  - '*'
- apiGroups:
  - admissionregistration.k8s.io
  resources:
  - '*'
  verbs:
  - '*'
- apiGroups:
  - policy
  resources:
  - '*'
  verbs:
  - '*'
- apiGroups:
  - autoscaling
  resources:
  - '*'
  verbs:
  - '*'
- apiGroups:
  - networking.istio.io
  resources:
  - '*'
  verbs:
  - '*'
- apiGroups:
  - config.istio.io
  resources:
  - '*'
  verbs:
  - '*'
- apiGroups:
  - iam.kubesphere.io
  resources:
  - '*'
  verbs:
  - '*'
- apiGroups:
  - notification.kubesphere.io
  resources:
  - '*'
  verbs:
  - '*'
- apiGroups:
  - auditing.kubesphere.io
  resources:
  - '*'
  verbs:
  - '*'
- apiGroups:
  - events.kubesphere.io
  resources:
  - '*'
  verbs:
  - '*'
- apiGroups:
  - core.kubefed.io
  resources:
  - '*'
  verbs:
  - '*'
- apiGroups:
  - installer.kubesphere.io
  resources:
  - '*'
  verbs:
  - '*'
- apiGroups:
  - storage.kubesphere.io
  resources:
  - '*'
  verbs:
  - '*'
- apiGroups:
  - security.istio.io
  resources:
  - '*'
  verbs:
  - '*'
- apiGroups:
  - monitoring.kiali.io
  resources:
  - '*'
  verbs:
  - '*'
- apiGroups:
  - kiali.io
  resources:
  - '*'
  verbs:
  - '*'
- apiGroups:
  - networking.k8s.io
  resources:
  - '*'
  verbs:
  - '*'
- apiGroups:
  - kubeedge.kubesphere.io
  resources:
  - '*'
  verbs:
  - '*'
- apiGroups:
  - types.kubefed.io
  resources:
  - '*'
  verbs:
  - '*'
- apiGroups:
  - monitoring.kubesphere.io
  resources:
  - '*'
  verbs:
  - '*'
- apiGroups:
  - application.kubesphere.io
  resources:
  - '*'
  verbs:
  - '*'


---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: ks-installer
subjects:
- kind: ServiceAccount
  name: ks-installer
  namespace: kubesphere-system
roleRef:
  kind: ClusterRole
  name: ks-installer
  apiGroup: rbac.authorization.k8s.io

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ks-installer
  namespace: kubesphere-system
  labels:
    app: ks-install
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ks-install
  template:
    metadata:
      labels:
        app: ks-install
    spec:
      serviceAccountName: ks-installer
      containers:
      - name: installer
        image: kubesphere/ks-installer:v3.2.1
        imagePullPolicy: "Always"
        resources:
          limits:
            cpu: "1"
            memory: 1Gi
          requests:
            cpu: 20m
            memory: 100Mi
        volumeMounts:
        - mountPath: /etc/localtime
          name: host-time
          readOnly: true
      volumes:
      - hostPath:
          path: /etc/localtime
          type: ""
        name: host-time
```

#### 8.4 开始安装KubeSphere

```bash
kubectl apply -f kubesphere-installer.yaml
kubectl apply -f cluster-configuration.yaml

# 通过如下命令查看安装进度
kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l app=ks-install -o jsonpath='{.items[0].metadata.name}') -f
```

![kubesphere-install](E:\linux笔记\Linux\Kubernetes\kubesphere-install.png)

出现如上图所示即安装成功，在浏览器输入对应地址即可打开界面。

![kubesphere-install-2](E:\linux笔记\Linux\Kubernetes\kubesphere-install-2.png)





## 核心组件概念

```bash
#kubectl语法格式
kubectl [command] [TYPE] [NAME] [flages]

kubectl create deployment nginx --image=nginx	#创建nginx容器
kubectl expose deployment nginx --port=80 --type=NodePort	#暴露nginx的端口
kubectl get pod,svc			#查看所有的容器
kubectl apply -f *.yaml		#通过yaml文件创建Pod
```

```bash
#方法一：
#使用 kubectl create 命令生成yaml文件
kubectl create deployment web --image=nginx -o yaml --dry-run > nginx1.yaml

#方法二：
#使用 kubectl get 命令导出yaml文件,新版中--export已经去除
kubectl get deploy nginx -o yaml --export > nginx2.yaml
kubectl get deploy nginx -o yaml  > nginx2.yaml
```

#### Pod

```yaml
#基本概念
1、最小部署单元
2、包含多个容器（一组容器的集合）
3、一个pod中容器共享网络命名空间
4、pod是短暂的

#存在意义
1、创建容器使用docker，一个docker对应一个容器，一个容器运行一个进程
2、pod是多线程，运行多个应用程序(一个pod中有多个docker，一个docker运行一个应用程序)
3、pod存在亲密性
	* 两个应用之间进行交互
	* 网络之间调用
	* 两个应用需要频繁调用
	
#Pod实现机制
1、共享网络：通过Pause容器，把其他业务容器加入到Pause容器中，让所有业务容器在同一个名称空间中，实现网络共享
2、共享存储：映入数据卷概念Vloume，使用数据卷进行持久化存储

#Pod镜像拉取策略
imagePullPolicy: Always
 * IfNotPresent: 默认值，镜像在宿主机上不存在时才拉取
 * Always: 每次创建 Pod 都会重新拉取一次镜像
 * Never: Pod 永远不会主动拉取镜像
 
#Pod资源限制
resource:
	requests: #调度
		memory:	"64Mi"
		cpu: "250m"
	
	limits: #最大
		memory: "128Mi"
		cpu: "500m"
		
#Pod重启机制
restartPolicy: Never
	* Always: 当容器终止退出后，总是重启容器，默认策略
	* OnFailurF: 当容器异常退出(退出状态码非0)时，才重启容器
	* Never：当容器终止退出，从不重启容器
	
#Pod健康检查 
 * livenessProbe（存活检查）
 	如果检查失败，将杀死容器，根据Pod的restartPolicy来操作
 * readinessProbe（就绪检查）
 	如果检查失败，Kubernetes会把Pod从service endpoints中提出
 
 livenessProbe支持以下三种检查方法：
 	* httpGet	发送HTTP请求，返回200-400范围状态码为成功
 	* exec		执行shell命令，返回状态码是0为成功
 	* tcpSocket 发起TCP Socket建立成功
```

##### Pod调度

![pod创建过程](E:\linux笔记\Linux\Kubernetes\pod创建过程.png)

```yaml
#过程
master节点：
1、用户使用create yaml创建pod，请求给apiserver，apiserver将yaml中的属性信息(metadata)写入etcd中；
2、apiserver触发watch机制准备创建pod，信息转发给scheduler调度器，scheduler使用调度算法选择node，调度将node信息给apiserver，apiserver将绑定的node信息写入etcd中；

node节点：
1、apisserver又通过watch监听机制，调用kubelet，指定pod信息，触发docker run命令创建容器；
2、node节点创建好pod之后反馈给kubelet，kubelet又将pod的状态信息给apiserver，apiserver又将pod的状态信息吸入etcd中；
```

##### 影响调度的属性

```yaml
#1、资源限制
	根据request找到足够资源的node节点进行调度

#2、节点选择器标签
 	spec:
 		nodeSelector:
 		  env_rolF: dev
首先对节点创建标签，节点选择器通过标签进行节点调度
```

##### **Pod回滚策略**

###### 重建

```yaml
#重建过程中，服务会中断
spec:
  replicas: 2
  selector:
    matchLabels:
      app: wethink-web
  strategy:
    type: Recreate
```

![重建](E:\linux笔记\Linux\Kubernetes\重建.png)



###### 滚动

```yaml
maxSurge: 25% #比如有 4 个实例，每次最多只能重启 1 个实例，保证服务可用
maxUnavailable: 25% #比如有 4 个实例，最多只能容忍 1 个实例不可用

spec:
  replicas: 2
  selector:
    matchLabels:
      app: wethink-web
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
```

![滚动](E:\linux笔记\Linux\Kubernetes\滚动.png)

```bash
kubectl rollout pause deployment wethink-web -n wethink 
#执行回滚时，此条命令会将滚动暂停，这样新服务和旧服务都会访问到
```

![滚动-pause](E:\linux笔记\Linux\Kubernetes\滚动-pause.png)

```bash
kubectl rollout resume deployment wethink-web -n wethink
#将暂停的回滚继续执行回滚操作
```

![滚动-resume](E:\linux笔记\Linux\Kubernetes\滚动-resume.png)

```bash
kubectl rollout undo deployment -n wethink
#回滚到上一个版本
```



###### 蓝绿

```shell
#在蓝绿发布过程中，有两套生产环境：蓝环境和绿环境。蓝色是当前版本并拥有实时流量，绿色是包含更新代码的环境。无论任何时候，只有一套环境有实时流量。要发布一个新版本，需要先将代码部署到没有流量的环境中，这是执行最终测试的地方。当IT人员确认应用程序已经准备就绪，就会将所有流量都将路由到绿色环境。那么绿色环境就已经生效，并且执行发布。

#给新老版本服务的yaml加一个labels，并且使两个版本的服务都在运行
```

![蓝绿-1](E:\linux笔记\Linux\Kubernetes\蓝绿-1.png)

**在service中指向访问的版本**

![蓝绿-2](E:\linux笔记\Linux\Kubernetes\蓝绿-2.png)



**进行切换时，服务不会断开**

![蓝绿-3](E:\linux笔记\Linux\Kubernetes\蓝绿-3.png)



###### 金丝雀

```shell
#与蓝绿部署类似，金丝雀发布也是始于两套环境：有实时流量的环境以及没有实时流量但包含了更新的代码的环境。与蓝绿部署不同的是，流量是逐渐迁移到更新的代码。一开始是1%，然后10%、25%，以此类推，直至100%。通过自动化发布，当确认代码能够正确运行时，它就可以逐步推广到更大、更关键的环境中。如果在任何时候发生了问题，所有流量都会被回滚到之前的版本。这在很大程度上降低了风险，因为仅有一小部分用户会使用到新的代码。
```

#### Controller

```yaml
#1、什么是controller
 * 在集群上管理和运行容器的对象
	
#2、Pod和Controller的关系
 * Pod是通过Controller实现应用的运维（比如伸缩，滚动升级等等）
 * Pod和Controller是通过label标签建立关系
 
#3、Deployment控制器应用场景
 * 部署无状态应用
 * 管理Pod和ReplicasSet副本
 * 部署，滚动升级等功能
 ** 应用场景：web服务，微服务
 
#4、使用deployment部署应用（yaml）
```

![controller-1](E:\linux笔记\Linux\Kubernetes\controller-1.png)

```bash
* 创建一个web资源
[root@K8S-Master ~]# kubectl create deployment web --image=nginx --dry-run -o yaml > web.yaml
[root@K8S-Master ~]# kubectl apply -f web.yaml
[root@K8S-Master ~]# kubectl get pods -o wide
[root@K8S-Master ~]# kubectl get pods -o wide
NAME                  READY   STATUS    RESTARTS   AGE     IP            NODE        NOMINATED NODE   READINESS GATES
web-96d5df5c8-twxxd   1/1     Running   0          4m30s   10.244.2.12   k8s-node2   <none>           <none>

* 将资源端口暴露出来
[root@K8S-Master ~]# kubectl expose  deployment web --port=80 --type=NodePort --target-port=80 --name=web1 -o yaml >web1.yaml
[root@K8S-Master ~]# kubectl apply -f web1.yaml
[root@K8S-Master ~]# kubectl get pods,svc
NAME                      READY   STATUS    RESTARTS   AGE
pod/web-96d5df5c8-h697j   1/1     Running   0          3m34s

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/kubernetes   ClusterIP   172.15.0.1      <none>        443/TCP        2d21h
service/web          NodePort    172.15.26.240   <none>        80:31628/TCP   2m47s
service/web1         NodePort    172.15.14.41    <none>        80:32533/TCP   95s

#浏览器输入任一节点ip加对应端口就能访问到测试页面
```

![controller-2](E:\linux笔记\Linux\Kubernetes\controller-2.png)

##### 无状态

###### 应用升级

```bash
#5、应用升级回滚和弹性伸缩
 * 将web.yaml文件中的image加上版本号进行资源创建
[root@K8S-Master ~]# kubectl apply -f web.yaml
```

![controller-3](E:\linux笔记\Linux\Kubernetes\controller-3.png)

```shell
 * 资源创建好后可以在节点上使用docker查看1.14的nginx镜像
[root@K8S-Node1 ~]# docker images
```

![controller-4](E:\linux笔记\Linux\Kubernetes\controller-4.png)

```shell
 * 将nginx 1.14 升级到 1.15，docker会将1.15的镜像下载到本地
[root@K8S-Master ~]# kubectl set image deployment web nginx=nginx:1.15
deployment.apps/web image updated

 * 查看应用升级状态
[root@K8S-Master ~]# kubectl rollout status deployment web
deployment "web" successfully rolled out
```

![controller-5](E:\linux笔记\Linux\Kubernetes\controller-5.png)

![controller-6](E:\linux笔记\Linux\Kubernetes\controller-6.png)![controller-7](E:\linux笔记\Linux\Kubernetes\controller-7.png)



###### 应用回滚

```shell
 * 查看历史版本
[root@K8S-Master ~]# kubectl rollout history deployment web
deployment.apps/web 
REVISION  CHANGE-CAUSE
1         <none>
2         <none>

 * 回退历史版本 1.14
[root@K8S-Master ~]# kubectl rollout undo deployment web
deployment.apps/web rolled back

[root@K8S-Master ~]# kubectl rollout status deployment web
Waiting for deployment "web" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "web" rollout to finish: 1 old replicas are pending termination...
deployment "web" successfully rolled out
```

![controller-8](E:\linux笔记\Linux\Kubernetes\controller-8.png)

```shell
 * 通过历史版本回滚到指定的版本
[root@K8S-Master ~]# kubectl rollout undo deployment web --to-revision=5
```

![controller-9](E:\linux笔记\Linux\Kubernetes\controller-9.png)



###### 弹性伸缩

```shell
 * 将已存在的资源进行伸缩
[root@K8S-Master ~]# kubectl scale deployment web --replicas=10
deployment.apps/web scaled

[root@K8S-Master ~]# kubectl get pods
NAME                   READY   STATUS    RESTARTS   AGE
web-5bb6fd4c98-58nmx   1/1     Running   0          3m8s
web-5bb6fd4c98-9ptnj   1/1     Running   0          9s
web-5bb6fd4c98-fdx4z   1/1     Running   0          9s
web-5bb6fd4c98-ft9q8   1/1     Running   0          9s
web-5bb6fd4c98-gj48h   1/1     Running   0          9s
web-5bb6fd4c98-jmqc9   1/1     Running   0          9s
web-5bb6fd4c98-qr7xt   1/1     Running   0          9s
web-5bb6fd4c98-v6dp5   1/1     Running   0          9s
web-5bb6fd4c98-wjvsm   1/1     Running   0          9s
web-5bb6fd4c98-zjmsx   1/1     Running   0          9s

[root@K8S-Master ~]# kubectl scale deployment web --replicas=5
deployment.apps/web scaled

[root@K8S-Master ~]# kubectl get pods
NAME                   READY   STATUS    RESTARTS   AGE
web-5bb6fd4c98-58nmx   1/1     Running   0          5m16s
web-5bb6fd4c98-fdx4z   1/1     Running   0          2m17s
web-5bb6fd4c98-ft9q8   1/1     Running   0          2m17s
web-5bb6fd4c98-gj48h   1/1     Running   0          2m17s
web-5bb6fd4c98-v6dp5   1/1     Running   0          2m17s
```



##### 有状态

###### StatefulSet

```shell
#1、有状态和无状态
1)、无状态
 * 认为Pod都是一样的
 * 没有顺序要求
 * 不用考虑在那个node运行
 * 随意进行伸缩和扩展
 
2)、有状态
 * 以上因素都需要考虑
 * 每个Pod都是独立的
 * 唯一的网络标识符，持久存储
 * 有序启动，例如mysql主从
 
#2、部署有状态应用
 * 无头service
 ** ClusterIP: none
```

![service-1](E:\linux笔记\Linux\Kubernetes\service-1.png)

```shell
 1)StatefulSet部署有状态应用
##**sts.yaml** 
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx

---

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx-statefulset
  namespace: default
spec:
  servicename: nginx
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  templatF:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

![service-2](E:\linux笔记\Linux\Kubernetes\service-2.png)

```shell
 * 查看Pod（三个Pod，每个都是唯一的名称）和service（无头ClusterIp）
```

![service-3](E:\linux笔记\Linux\Kubernetes\service-3.png)

```shell
 * deployment额statefulset区别： 有身份（唯一标识）
 * 根据主机名 + 按照一定规则生成域名
 *** 每个Pod有唯一主机名
 *** 唯一域名：
     格式： 主机名称.service名称.名称空间.svc.cluster.local
           nginx-statefulset-0.nginx.default.svc.cluster.local
```



###### DaemonSet

```shell
#3、部署守护进程DaemonSet
 * 在每个node上运行一个Pod，新加入的node也同样运行一个pod，当旧节点被删除后，它上面的pod也会被回收掉
 ** 例如：在每个node节点安装数据采集工具
```

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ds-test
  labels:
    app: filebeat
spec:
  selector:
    matchLabels:
      app: filebeat
  templatF:
    metadata:
      labels:
        app: filebeat
    spec:
      containers:
      - name: logs
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - name: varlog
          mountPath: /tmp/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```

![service-4](E:\linux笔记\Linux\Kubernetes\service-4.png)



###### Job

```shell
#4、job（一次性任务）
##job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  templatF:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl", "-Mbignum=bpi",  "-wle", "print bpi(2000)" ]
      restartPolicy: Never
  backoffLimit: 4
```

![service-5](E:\linux笔记\Linux\Kubernetes\service-5.png)



###### CronJob

```shell
#5、cronjob（定时任务）
##cronjob.yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedulF: "*/1 * * * *"
  jobTemplatF:
    spec:
      templatF:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

![service-6](E:\linux笔记\Linux\Kubernetes\service-6.png)



#### Service

```shell
#1、存在的意义
 * 防止Pod失联（服务发现）
 * 定义一组Pod访问策略（负载均衡）
 
#2、Pod和Service的关系
 * 根据label和selector标签建立关联的
 
 Service：					Pod：
 	selector：					labels:
 		app: nginx					app:nginx
 		
#3、常用Service类型
 * ClusterIP：集群内部使用，默认使用
 * NodePort：在所有安装了kube-proxy的节点上打开一个端口，此端口可以代理至后端Pod，
             然后集群外部可以使用节点的IP地址和NodePort的端口号访问到集群Pod的服务。
             NodePort默认端口访问：30000-32767
 * LoadBalancer：会用云提供商的负载均衡器公开服务。	
 * ExternalName: 通过返回定义的CNAME别名
```



#### Secret

```shell
作用：加密数据存在etcd中，让Pod容器以挂载Volume方式进行访问
场景：凭证

加密：echo -n wethink|base64

#1、创建secret加密数据
##secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
```

![secret-1](E:\linux笔记\Linux\Kubernetes\secret-1.png)

```shell
#2、以变量的形式挂载到pod容器中
##secret-var.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: nginx
    image: nginx
    env:
      - name: USERNAME
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: username
      - name: PASSWORD
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: password
```

![secret-2](E:\linux笔记\Linux\Kubernetes\secret-2.png)

```shell
#3、以Volume形式挂载到Pod容器中
##secret-vol.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretname: mysecret
```

![secret-3](E:\linux笔记\Linux\Kubernetes\secret-3.png)



#### ConfigMap

```shell
作用：存储不加密数据到etcd，让Pod以变量或者Volume挂载到容器中
场景：配置文件

1）、创建配置文件
[root@K8S-Master ~]# cat redis.properties 
redis.host=127.0.0.1
redis.port=6379
redis.password=123456

2)、创建ConfigMap
[root@K8S-Master ~]# kubectl create configmap redis-config --from-file=redis.properties
[root@K8S-Master ~]# kubectl get configmap
NAME               DATA   AGE
kube-root-ca.crt   1      118m
redis-config       1      7s
```

![secret-4](E:\linux笔记\Linux\Kubernetes\secret-4.png)

```shell
3)、以Volume形式挂载到Pod容器中
##cm.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: busybox
      image: busybox
      command: [ "/bin/sh", "-c","cat /etc/config/redis.properties" ]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: redis-config
  restartPolicy: Never
```

![cm-1](E:\linux笔记\Linux\Kubernetes\cm-1.png)

```shell
4)、以变量形式挂载到Pod容器中
 （1）、创建yaml文件，声明变量信息，configmap创建
 （2）、以变量形式挂载
 
 ##myconfig.yaml
 apiVersion: v1
kind: ConfigMap
metadata:
  name: myconfig
  namespace: default
data:
  special.level: info
  special.typF: hello
  
##config-var.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: busybox
      image: busybox
      command: [ "/bin/sh", "-c" , "echo $(LEVEL) $(TYPE)" ]
      env:
        - name: LEVEL
          valueFrom:
            configMapKeyRef:
              name: myconfig
              key: special.level
        - name: TYPE
          valueFrom:
            configMapKeyRef:
              name: myconfig
              key: special.type
  restartPolicy: Never
```

![cm-2](E:\linux笔记\Linux\Kubernetes\cm-2.png)



#### Volume

##### NFS

```shell
1、安装nfs，设置挂载路径
yum -y install nfs-utils

cat >>/etc/exports<<-EOF
/data/nfs *(rw,sync,no_root_squash)
EOF

2、在k8s集群部署引用nfs持久网络存储
```

![nfs-1](E:\linux笔记\Linux\Kubernetes\nfs-1.png)

##### hostPath

```shell
# hostPath挂载宿主机路径
hostPath卷常用的type：
* type为空白字符串: 默认选项，意味着挂载hostPath卷之前不会执行任何检查。
* DirectroyOrCreate： 如果给定的path不存在任何东西，那么将根据需要创建一个权限为0755的空白目录，和kublet具有相同的组和权限。
* Directory：目录必须存在于给定的路径下。
* FileOrCreate： 如果给定的路径不存储任何东西，则会根据需要创建一个空白文件，权限设置为0644，和kubelet具有相同的组合权限。
* File：文件，必须存在于给定的路径下。
* Socket：UNIX套接字，必须存在于给定的路径下。
* CharDevice：字符设备，必须存在于给定的路径下。
* BlockDevice：块设备，必须存在于给定的路径下。
```

##### emptyDir

```shell
volumes:
- name: share-volume
  emptyDir: {}
```

![Kubernetes-Volume-1](E:\linux笔记\Linux\Kubernetes\Kubernetes-Volume-1.png)

```shell
# 当一个pod中运行两个容器，将/opt目录共享，在任一容器的opt目录下创建文件，另一个容器的opt目录下也能看见
```

![Kubernetes-Volume-2](E:\linux笔记\Linux\Kubernetes\Kubernetes-Volume-2.png)

#### PV、PVC

**nfs类型**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: slow
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /tmp
    server: 172.17.0.2
```

```shell
persistentVolumeReclaimPolicy     #回收策略
  Recycle：回收，rm -rf  Deployment -> PVC -> PV 
  Retain：保留
  Delete：删除pvc后pv也会被删掉，这一类的pv，需要支持删除的功能，动态存储默认方式

Capacity:     # pv的容量
volumeMode:   # 挂载的类型，Filesystem，block
accessModes： # 这个pv的访问模式：
                   # ReadWriteOnce：RWO，可以被单节点以读写的模式挂载。
 				   # ReadWriteMany：RWX，可以被多节点以读写的模式挂载。
 				   # ReadOnlyMany：ROX，可以被多节点以只读的模式挂载。
 				   
storageClassName： # PV的类，可以说是一个类名，PVC和PV的这个name必须一样，才能绑定。
```

```shell
PV的状态：
  Available:  #空闲的PV，没有被任何PVC绑定
  Bound：     #已经被PVC绑定
  Released：  #PVC被删除，但是资源未被重新使用
  Failed：    #自动回收失败
```

```yaml
# pv
apiVersion: v1
kind: PersistentVolume
metadata:
  name: core-logs-pv
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 10Gi
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: wethink-core-logs-pvc
    namespace: wethink
  nfs:
    path: /app/core-logs
    server: 192.168.3.133
  persistentVolumeReclaimPolicy: Retain
  volumeMode: Filesystem
status:
  phase: Bound
```

```yaml
# pvc
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wethink-core-logs-pvc
  namespace: wethink
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: ""
  volumeMode: Filesystem
  volumeName: core-logs
status:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 10Gi
  phase: Bound
```

```yaml
# yaml中使用pvc
- name: core-logs
  persistentVolumeClaim:
    claimName: wethink-core-logs-pvc    #pvc的名称
```

#### CronJob

```yaml
在k8s中运行周期性计划任务，crontab。
```

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            imagePullPolicy: IfNotPresent
            command:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

参数：

```yaml
concurrencyPolicy: Allow      #并发调度策略：Allow 允许同时运行多个任务。
                   Forbid     #不运行并发执行
                   Replace    #替换之前的任务               
failedJobHistoryLimit: 1      #保留失败的任务次数
successfulJobsHistoryLimit: 3 #成功的Job保留的次数
suspend: false                #挂起，true：cronjob不会被执行
```

![cronjob-1](E:\linux笔记\Linux\Kubernetes\cronjob-1.png)

#### Taint

```yaml
#污点和污点容忍

Taint污点： 节点不做普通分配调度，是节点属性。让不能容忍这个污点的Pod不能部署在这个打了污点的服务器上

2）场景
	*	专用节点	配置特点硬件节点
	*	给予Taint驱逐

3）具体演示
（1） 查看节点污点情况
* 污点值有三个
  NoSchedule:	     # 一定不被调度
  PreferNoSchedule:  # 尽量不被调度
  NoExecute:         # 不会调度，并且驱逐Node已有的Pod
```

![节点污点查看](E:\linux笔记\Linux\Kubernetes\节点污点查看.png)

```shell
（2） 为节点添加污点
kubectl taint node [node] key=value:污点三个值
```

```shell
(3) 移除污点
kubectl taint nodes k8s-node1 key:NoExecute-
```

![节点污点查看2](E:\linux笔记\Linux\Kubernetes\节点污点查看2.png)

#### Tolerations

```yaml
#节点上有多个Taint，每个Taint都需要容忍Pod才能部署上去
tolerations:
- effect: NoSchedule
  key: test1
  operator: Exists    #匹配到key为test1的污点就能容忍
    
tolerations:
- operator: Exists    #容忍所有的污点

tolerations:
- operator: Exists
  key: test1          #只要污点存在这个key，就能容忍
  
tolerationSeconds: 60 # 需要和NoExecute一直配置，Pod只能在这个节点停留60秒，之后还是会被清除。当node节点不正常时，master节点会将node节点打上NoSchdule和NoExecute两个污点，导致Pod全部被删除，这个参数就避免Pod因为网络波动导致Pod被清除
```

#### Affinity

```yaml
#亲和性
nodeSelector和nodeAffinity:	Pod调度到某些节点上，Pod属性，调度时候实现

NodeAffinity: 节点亲和力
  * RequireDuringSchedulinglgnoredDuringExecution:   #硬亲和力，即支持必须部署在指定的节点上，也支持必须不部署在指定的节点上
  * PreferredDuringSchedulinglgnoredDuringExecution: #软亲和力，尽量部署在满足条件的节点上，或者是尽量不要部署在被匹配的节点


spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: beta.kubernetes.io/os
                    operator: In
                    values:
                      - linux
                  - key: beta.kubernetes.io/arch
                    operator: In
                    values:
                      - amd64
                      
nodeAffinity和nodeSelector基本一样，根据节点上的标签约束来绝对pod调度到哪些节点
	* 硬亲和性
		约束条件必须满足
	* 软亲和性
		尝试满足，不保证
		
operator: In
支持常用操作符：
	In: 部署在满足多个条件的节点上
	NotIn: 	不要部署在满足这些条件的节点上
	Exists:  部署在具有某个存在key为指定值的node节点
	DoesNotExists: 与Exists相反
	Gt: 大于指定的条件
	Lt: 小于指定的条件
```

```yaml
PodAffinity: Pod亲和力
  A应用B应用C应用，将A应用根据某种策略尽量部署在一起

  * RequireDuringSchedulinglgnoredDuringExecution: 
    将A应用和B应用部署在一起
  * PreferredDuringSchedulinglgnoredDuringExecution:
    尽量将A应用和B应用部署在一起
```

#### Antiffinity

```yaml
PodAntiffinity: Pod反亲和力
  A应用B应用C应用，将A应用根据某种策略尽量或不部署在一起      Label
  * RequireDuringSchedulinglgnoredDuringExecution: 
    不要将A与与之匹配的应用部署在一起                      Label
  * PreferredDuringSchedulinglgnoredDuringExecution:
	尽量不要将A与与之匹配的应用部署在一起					  Label
```

#### RBAC

```shell
RBAC：基于角色的访问控制，Role-Based Access Control。他是一种基于企业内个人角色来管理一些资源的访问方法。

RBAC：4个顶级资源，Role、ClusterRole、RoleBinding、ClusterRoleBinding

Role：角色，宝行一组权限的规则。没有拒绝规则，只是附加允许。Namespace隔离，只作用于命名空间内。

ClusterRole：和Role的区别，Role只作用于命名空间内。ClusterRole作用于整个集群。

RoleBinding：作用于命名空间内，将ClusterRole或者Role绑定到User、Group、ServiceAccount。

ClusterRoleBinding：作用于整个集群。

ServiceAccount、User、Group
```

```yaml

1、概述
1）访问k8s集群时，需要经过三个步骤完成具体操作
  第一步：认证（传输安全）
    * 传输安全：对外不暴露8080端口，只能内部访问，对外使用端口6443
    * 认证：客户端身份认证常用方式
      ** https证书认证，基于ca证书
      ** http token认证，通过token识别用户
      ** http基本认证，用户名+密码认证
  	
  第二步：鉴权（授权）
    * 基于RBAC进行鉴权操作
    * 基于角色访问控制
    
  第三步：准入控制
    * 就是在创建资源经过身份验证之后，apiserver在数据写入之前做一次拦截，然后对资源进行更改，判断正确性操作。
  
2）进行访问时，过程中都需要经过apiserver，apiserver做统一协调，比如门卫。
   访问过程中，需要证书、token、或者用户名+密码。
   如果需要访问Pod，需要serviceAccount。
```

```shell
2、RBAC
基于角色的访问控制
 * 角色
  ** role：特定命名空间访问权限
  ** ClusterRole：所有命名空间访问权限
  
 * 角色绑定
  ** roleBinding：角色绑定到主体
  ** ClusterRoleBinding：集群角色绑定到主体
  
 * 主体
  ** user：用户
  ** group：用户组
  ** serviceAccount：服务账号
```

#### Rook

```yaml
一个自我管理的分布式存储编排系统，它本身并不是存储系统，在存储和k8s之间搭建了一个桥梁，存储系统的搭建和维护变得特别简单，Rook支持CSI，CSI做一些PVC的快照、扩容等操作。

Operator：主要用有状态的服务，或者用于比较复杂应用的管理。
Helm：主要用于无状态的服务，配置分离。

Rook:
    Agent：在每个存储节点上运行，用于配置一个FlexVolume插件，和k8s的存储卷进行集成。挂载网络存储、加载存储卷、格式化文件系统。
    Discover：主要用于检测连接到存储节点上的存储设备上。
  
Ceph：
    OSD：直接连接每一个集群节点的物理磁盘或者是目录。集群的副本数、高可用性和容错性。
    Mon：集群监控，所有集群节点都会想Mon汇报。他记录了集群的拓扑以及数据存储位置的信息。
    MDS：元数据服务器，负责跟踪文件层次结构并存储ceph元数据。
    RGW：restful API接口。
    MGR：提供额外的监控和界面。
```

#### Helm

```shell
1、helm引入
	1）、之前方式部署应用基本过程
		* 编写yaml文件
		** deployment
		** service
		** ingress
		
		* 如果使用之前的方式部署一应用，少数服务的应用，比较合适
		* 比如部署微服务项目，可能有十几个服务，每个服务都有一套
		  yaml文件，需要维护大量的yaml文件，版本管理特别不方便
		  
2、使用helm可以解决那些问题？
	1）使用helm可以把这些yaml作为一个整体管理
	2）实现yaml高效复用
	3）使用heml应用级别的版本管理
	
3、heml介绍
	helm是一个kubernetes的包管理工具，就像Linux下的包管理器，如yum/apt等，
	可以很方便的将之前打包好的yaml文件部署到kubernetes上

4、helm三个重要概念：
	1）helm    是一个命令行客户端工具
	2）Chart   把yaml打包，是yaml集合
	3）Release 基于chart部署实体，应用级别的版本管理
	
5、helm在2019年发布v3版本，与之前版本相比有变化
	1）v3版本删除Tiller
	2）release可以在不同命名空间重用
	3）将chart推送到docker仓库中
```

##### helm安装

```shell
1、下载helm压缩文件，上传只至服务器
2、解压helm压缩文件，将helm可执行文件复制到/usr/bin下面
```

##### 配置helm仓库

```shell
1、添加仓库
# helm repo add 仓库名称 仓库地址
helm repo add stable http://mirror.azure.cn/kubernetes/charts
helm repo add aliyun https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
helm repo update       	 #更新仓库
helm repo remove stable  #删除仓库

#[root@master linux-amd64]# helm repo list
#NAME  	URL                                                   
#stable	http://mirror.azure.cn/kubernetes/charts              
#aliyun	https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
```

##### helm快速部署应用

```shell
一、使用命令搜索应用
helm search repo 名称 (weave)

二、根据搜索内容选择安装
helm install 安装之后的名称 搜索之后应用名称

 * 查看安装之后状态
   helm list
   helm status 安装之后的名称
```



```shell
#weave的svc并未对外暴露端口，需要手动修改weave的svc,将ClusterIp修改为NodePort
kubectl edti svc ui-weave-scope
```



```shell
浏览器输入节点ip:32765即可打开weave界面
```

##### 自定义Chart

```shell
1、使用命令创建chart
# helm create mychart

```

![chart-1](E:\linux笔记\Linux\Kubernetes\chart-1.png)

```yaml
.
├── charts                 # 依赖文件
├── Chart.yaml             # 这个chart的版本信息
├── templates              # 模板
│   ├── deployment.yaml
│   ├── _helpers.tpl       # 自定义的模板或者函数
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── NOTES.txt          # 这个chart的信息
│   ├── serviceaccount.yaml
│   ├── service.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml            # 配置全局变量或者一些参数
```



```shell
2、准备好yaml文件
kubectl expose deployment web1 --port=80 --target-port=80 --type=NodePort --dry-run -o yaml > service.yaml
kubectl expose deployment web1 --port=80 --target-port=80 --type=NodePort --dry-run -o yaml > service.yaml
```

```shell
3、安装mychart
helm install web1 mychart/
```

![chart-2](E:\linux笔记\Linux\Kubernetes\chart-2.png)

```shell
4、应用升级
helm upgrade chart名称
```

![chart-3](E:\linux笔记\Linux\Kubernetes\chart-3.png)



#### Ingress

```shell
概念：ingress和Service、Deployment也是一个资源类型，ingress用于实现用域名的方式访问k8s内部应用

1、把端口号对外暴露，通过ip+端口号进行访问
 * 使用Service里面的NodePort实现
 
2、NodePort缺陷
 * 在每个节点上都会启动端口，在访问时通过任何节点，通过任何节点ip+端口号都可以实现访问
 * 意味着每个服务器端口只能使用一次，一个端口对应一个应用
 * 实际访问中都是用域名，根据不同域名跳转到不同端口服务中
 
3、Ingress和Pod关系
 * pod和ingress通过service关联的
 ** ingress作为统一入口，由service关联一组pod
 
4、ingress工作流程
 * ingress作为统一入口，通过访问域名来访问service关联的pod
 
5、使用ingress
 * 第一步 部署ingress Controller
 * 第二步 创建ingress规则
#这里选择官方维护的nginx控制器，实现部署  
 
6、使用ingress对外暴露应用
 1）创建nginx应用，对外暴露端口使用NodePort
```



 
