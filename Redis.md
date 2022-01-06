---

---

### 编译安装Redis

```shell
#1、下载Redis6.0源码包
wget https://download.redis.io/releases/redis-6.0.10.tar.gz?_ga=2.110525445.1230134410.1611887533-1504791385.1611887533

#2、安装依赖包
yum -y install gcc gcc-c++ tcl

#3、解压，开始安装
tar -xvf redis-6.0.10.tar.gz -C /opt
cd /opt/redis-6.0.10 
make && make install

#若是报错，检查依赖是否安装，gcc版本是否高于5.3

#4、升级gcc，安装scl源
yum install centos-release-scl scl-utils-build
yum list all --enablerepo='centos-sclo-rh' | grep gcc	#查找gcc有哪些版本可以安装
yum -y install devtoolset-9-gcc devtoolset-9-gcc-c++ devtoolset-9-binutils
scl enable devtoolset-9 bash
#检查gcc版本
gcc -v

#5、重新安装
make && make install
```

### 性能测试

```shell
redis-benchmark -h localhost -p 6379 -c 100 -n 100000
```

### 配置文件

```shell
#redis.conf
#以守护进程模式启动
daemonize	yes
#绑定的主机地址
bind 127.0.0.1
#监听端口
port 6379
#pid文件和log文件的路径
pidfile
logfile
#设置数据库的数量，默认数据库编号为0
databases 16
#指定本地持久化文件的文件名，默认是dump.rdb
save 900 1		#900秒内有一个key改变就保存
save 300 10
save 60 10000
dbfilename redis_6379.rdb
#本地数据库的目录
dir /data/redis/redis_6379
#是否打开aof日志功能
appendonly yes
#每一个命令，都立即同步到aof
appendfsync always
#每秒写一次
appendfsync everysec
#写入工作交给操作系统，由操作系统判断缓存冲区大小，统一写入到aof
appendfsync no
appendfilename "appendonly.aof"
```

```bash
localhost:6379> select 0		#选择数据库
OK
localhost:6379> DBSIZE			#查看数据库大小
(integer) 4
localhost:6379> 
```

### 基础命令

```bash
localhost:6379> keys *		#查看数据库所有的key，危险命令
1) "mylist"
2) "key:__rand_int__"
3) "counter:__rand_int__"
4) "myhash"
```

清空当前数据库 **flushdb**，清空全部数据库内容 **flushall**

```bash
localhost:6379> flushdb
OK
localhost:6379> keys *
(empty array)
```

### 数据类型

#### Redis-Key

```shell
#1、查看所有key
localhost:6379> keys *
1) "k1"
```

```shell
#2、设置key，获取key值，删除key
localhost:6379> set k1 v1					#设置单个key
OK	
localhost:6379> mset K1 V1 K2 V2 K3 V3		#设置多个key
OK
localhost:6379> get k1						#获取单个key
"v1"
localhost:6379> mget K1 K2 K3				#获取多个key
1) "V1"
2) "V2"
3) "V3"
localhost:6379> del k1						#删除key
(integer) 1
localhost:6379> get k1
(nil)
```

```shell
#3、查看key类型
localhost:6379> keys *
1) "k1"
localhost:6379> type k1
string
```

```shell
#4、判断key是否存在
localhost:6379> keys *
1) "k1"
localhost:6379> exists k1
(integer) 1
localhost:6379> exists k2
(integer) 0
```

```shell
#5、移除当前key
localhost:6379> move k1 1	#move key db编号
(integer) 0
```

```shell
#6、设置和查看 key 的过期时间，单位是秒（-1 永不过期；-2 没有这个key）
localhost:6379> EXPIRE k2 10	#设置过期时间
(integer) 1
localhost:6379> ttl k2			#查看过期时间
(integer) 8
localhost:6379> ttl k2
(integer) 7
localhost:6379> ttl k2
(integer) 4
localhost:6379> ttl k2
(integer) 1
localhost:6379> ttl k2
(integer) -2
```

```shell
#7、取消过期时间
localhost:6379> set k2 v2
OK
localhost:6379> expire k2 10		#设置过期时间
(integer) 1
localhost:6379> ttl k2				#查看过期时间
(integer) 8
localhost:6379> ttl k2
(integer) 6
localhost:6379> persist k2			#取消过期时间
(integer) 1
localhost:6379> ttl k2
(integer) -1 #-1 永不过期
```

```shell
#8、计算器
localhost:6379> set k3 1
OK
localhost:6379> incr k3				#每次执行一次就+1
(integer) 2
localhost:6379> incr k3
(integer) 3
localhost:6379> incr k3
(integer) 4
localhost:6379> incrby k3 100		#设置key加多少
(integer) 104
```

#### String（字符串）

```bash
localhost:6379> set key1 v1
OK
localhost:6379> get key1
"v1"
localhost:6379> append key1 "hello"		#追加字符串
(integer) 7
localhost:6379> get key1
"v1hello"
localhost:6379> strlen key1				#查看key值长度
(integer) 7
localhost:6379> append key1 ",world!"
(integer) 14
localhost:6379> strlen key1
(integer) 14
```

#### List（列表）

```bash
#LPUSH可向list的左边（头部）添加一个新元素
#RPUSH可向list的右边（头部）添加一个新元素
#LRANGE可从list中取出一定范围的元素
localhost:6379> lpush list1 A
(integer) 1
localhost:6379> lpush list1 B
(integer) 2
localhost:6379> rpush list1 0
(integer) 3
localhost:6379> rpush list1 1
(integer) 4
localhost:6379> llen list1
(integer) 4
localhost:6379> lrange list1 0 -1	#输入不存在的从而到达查看list中全部的值
1) "B"
2) "A"
3) "0"
4) "1"
localhost:6379> lrange list1 1 1	#查看1-1这个范围
1) "A"
localhost:6379> lrange list1 1 3	#查看1-3这个范围
1) "A"
2) "0"
3) "1"
localhost:6379> rpop list1			#从右边开始删除
"1"
localhost:6379> lrange list1 1 3
1) "A"
2) "0"
localhost:6379> lpop list1			#从左边开始删除
"B"
localhost:6379> lrange list1 0 -1
1) "A"
2) "0"
```

#### Hash（哈希）

```bash
localhost:6379> hmset user:1000 name cai age 20 job it
OK
localhost:6379> hmget user:1000 name	#查看这个哈希某个字段
1) "cai"
localhost:6379> hmget user:1000 age
1) "20"
localhost:6379> hmget user:1000 job
1) "it"
localhost:6379> hgetall user:1000		##查看这个哈希全部字段
1) "name"
2) "cai"
3) "age"
4) "20"
5) "job"
6) "it"
```

#### Set（集合）

```bash
localhost:6379> sadd set1 1 2 3 4 5		#创建集合
(integer) 5
localhost:6379> sadd set2 3 6 2 10
(integer) 4
localhost:6379> SMEMBERS set1			#查看集合
1) "1"
2) "2"
3) "3"
4) "4"
5) "5"
localhost:6379> SMEMBERS set2
1) "2"
2) "3"
3) "6"
4) "10"
localhost:6379> sadd set2 3 6 2 10 11 3 5 6	
(integer) 2
localhost:6379> SMEMBERS set2		#不允许出现重复
1) "2"
2) "3"
3) "5"
4) "6"
5) "10"
6) "11"
localhost:6379> sdiff set1 set2		#以第一个为基准，找二个没有的
1) "1"
2) "4"
localhost:6379> SMEMBERS set1
1) "1"
2) "2"
3) "3"
4) "4"
5) "5"
localhost:6379> SMEMBERS set2
1) "2"
2) "3"
3) "5"
4) "6"
5) "10"
6) "11"
localhost:6379> sdiff set2 set1		#以第二个为基准，找第一个没有的
1) "6"
2) "10"
3) "11"
localhost:6379> sinter set1 set2	#以第一个为准，求共有的
1) "2"
2) "3"
3) "5"
localhost:6379> sunion set1 set2	#去重后，按顺序排
1) "1"
2) "2"
3) "3"
4) "4"
5) "5"
6) "6"
7) "10"
8) "11"
```

### Redis持久化

```shell
#若是没有配置数据持久化，redis重启之后缓存数据直接丢失，找不回。
#配置持久化后，redis重启后回去找rdb文件，将数据重新导入
save 900 1		#900秒内有一个key改变就保存
save 300 10
save 60 10000
dbfilename redis_6379.rdb

#rdb适用于全量备份，加载速度快
#rdb和aof同时存在时，优先读取aof
```

### 主从复制

```shell
#从装好redis后，修改配置文件，启动reids-server，进入redis-cli -h localhost
SLAVEOF 172.16.1.109 6379		#直接同步主库数据

#1、从库发起同步请求
#2、主库收到请求后执行bgsave保存当前内存中的数据到磁盘
#3、主库将持久化的数据发送给从库的数据目录
#4、从库收到主库的持久化数据后，先清空自己当前内存中的数据
#5、从库将主库发送过来的持久化文件加载到自己的内存中

#局限性
#1、执行主从复制之前，先将数据备份一份
#2、建议将主从复制写入到配置文件中
#3、在业务低峰期做主从复制
#4、拷贝数据时候会占用带宽
#5、不能自动完成主从切换，需要人工介入
```

### 安全认证

```shell
requirepass 123456
127.0.0.1:6379> AUTH 123456
```

### 哨兵

```shell
#基于主从复制，需要先将从服务器配置好slaveof 主redisip 6379
bind 172.16.1.109
port 26379
daemonize yes
logfile /opt/redis-6.0.10/redis_26379/logs/redis_26379.log
dir /data/redis/redis_26379
sentinel monitor mymaster 172.16.1.109 6379 2
sentinel down-after-milliseconds mymaster 3000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 18000
```

```bash
[root@Redis redis-6.0.10]# redis-cli -h 172.16.1.111 -p 26379
172.16.1.111:26379> info sentinel
# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
master0:name=mymaster,status=ok,address=172.16.1.111:6379,slaves=2,sentinels=3
```

```shell
172.16.1.113:6379> CONFIG GET slave-priority
1) "slave-priority"	
2) "100"
172.16.1.113:6379> CONFIG set slave-priority 0	#修改redis从的优先级
OK

[root@Redis redis-6.0.10]# redis-cli -h 172.16.1.111 -p 26379
172.16.1.111:26379> sentinel failover mymaster	#强制切换redis主
OK
172.16.1.111:26379> info sentinel
# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
master0:name=mymaster,status=ok,address=172.16.1.111:6379,slaves=2,sentinels=3
```

### 集群

```shell
bind 172.16.1.111
port 6380
daemonize yes
pidfile /opt/redis-6.0.10/redis_6380/pid/redis_6380.pid
logfile /opt/redis-6.0.10/redis_6380/logs/redis_6380.log
dbfilename redis_6380.rdb
dir /data/redis/redis_6380/
cluster-enabled yes						#启动集群模式cluster
cluster-config-file nodes_6380.conf		#集群的配置文件
cluster-node-timeout 15000				#集群的超时时间
```

```shell
#查看集群节点
[root@Redis redis-6.0.10]# redis-cli -h 172.16.1.111 -p 6380
172.16.1.111:6380> CLUSTER nodes
b36eaca3ff7cc62074e71244a897cc6cded5e2a6 :6380@16380 myself,master - 0 0 0 connected
172.16.1.111:6380> cluster meet 172.16.1.112 6380
OK
172.16.1.111:6380> CLUSTER nodes
b36eaca3ff7cc62074e71244a897cc6cded5e2a6 172.16.1.111:6380@16380 myself,master - 0 0 0 connected
a01f5d491825ed7e74af4b4ef6ced3bb1884d59e 172.16.1.112:6380@16380 master - 0 1612279323393 1 connect
#集群配置文件不要手动修改
[root@Redis redis-6.0.10]# cat /data/redis/redis_6380/nodes_6380.conf 
b36eaca3ff7cc62074e71244a897cc6cded5e2a6 172.16.1.111:6380@16380 myself,master - 0 0 0 connected
a01f5d491825ed7e74af4b4ef6ced3bb1884d59e 172.16.1.112:6380@16380 master - 0 1612279321347 1 connected
vars currentEpoch 1 lastVoteEpoch 0
```

```shell
172.16.1.111:6381> CLUSTER info		#查看集群状态,一共16384个槽
cluster_state:fail
cluster_slots_assigned:0
cluster_slots_ok:0
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:0
cluster_current_epoch:5
cluster_my_epoch:3
cluster_stats_messages_ping_sent:605
cluster_stats_messages_pong_sent:636
cluster_stats_messages_meet_sent:1
cluster_stats_messages_sent:1242
cluster_stats_messages_ping_received:636
cluster_stats_messages_pong_received:606
cluster_stats_messages_received:1242
```

```shell
#手动给每个主节点分配槽位
[root@Redis redis-6.0.10]# redis-cli -h 172.16.1.111 -p 6380 cluster addslots {0..5461}
OK
[root@Redis redis-6.0.10]# redis-cli -h 172.16.1.112 -p 6380 cluster addslots {5462..10922}
OK
[root@Redis redis-6.0.10]# redis-cli -h 172.16.1.113 -p 6380 cluster addslots {10923..16383}
OK
[root@Redis redis-6.0.10]# redis-cli -h 172.16.1.111 -p 6380 -c	#启动集群模式
```

### 集群工具

#### 参数

```shell

redis-cli --cluster help
Cluster Manager Commands:
  create         host1:port1 ... hostN:portN      #创建集群
                 --cluster-replicas <arg>         #从节点个数
  check          host:port                        #检查集群
                 --cluster-search-multiple-owners #检查是否有槽同时被分配给了多个节点
  info           host:port                     	  #查看集群状态
  fix            host:port                     	  #修复集群
                 --cluster-search-multiple-owners #修复槽的重复分配问题
  reshard        host:port                        #指定集群的任意一节点进行迁移slot，重新分slots
                 --cluster-from <arg>             #需要从哪些源节点上迁移slot，可从多个源节点完成迁移，以逗号隔开，传递的是节点的node id，还可以直接传递--from all，这样源节点就是集群的所有节点，不传递该参数的话，则会在迁移过程中提示用户输入
                 --cluster-to <arg>         #slot需要迁移的目的节点的node id，目的节点只能填写一个，不传递该参数的话，则会在迁移过程中提示用户输入
                 --cluster-slots <arg>      #需要迁移的slot数量，不传递该参数的话，则会在迁移过程中提示用户输入。
                 --cluster-yes              #指定迁移时的确认输入
                 --cluster-timeout <arg>    #设置migrate命令的超时时间
                 --cluster-pipeline <arg>   #定义cluster getkeysinslot命令一次取出的key数量，不传的话使用默认值为10
                 --cluster-replace          #是否直接replace到目标节点
  rebalance      host:port                                      #指定集群的任意一节点进行平衡集群节点slot数量 
                 --cluster-weight <node1=w1...nodeN=wN>         #指定集群节点的权重
                 --cluster-use-empty-masters                    #设置可以让没有分配slot的主节点参与，默认不允许
                 --cluster-timeout <arg>                        #设置migrate命令的超时时间
                 --cluster-simulate                             #模拟rebalance操作，不会真正执行迁移操作
                 --cluster-pipeline <arg>                       #定义cluster getkeysinslot命令一次取出的key数量，默认值为10
                 --cluster-threshold <arg>                      #迁移的slot阈值超过threshold，执行rebalance操作
                 --cluster-replace                              #是否直接replace到目标节点
  add-node       new_host:new_port existing_host:existing_port  #添加节点，把新节点加入到指定的集群，默认添加主节点
                 --cluster-slave                                #新节点作为从节点，默认随机一个主节点
                 --cluster-master-id <arg>                      #给新节点指定主节点
  del-node       host:port node_id                              #删除给定的一个节点，成功后关闭该节点服务
  call           host:port command arg arg .. arg               #在集群的所有节点执行相关命令
  set-timeout    host:port milliseconds                         #设置cluster-node-timeout
  import         host:port                                      #将外部redis数据导入集群
                 --cluster-from <arg>                           #将指定实例的数据导入到集群
                 --cluster-copy                                 #migrate时指定copy
                 --cluster-replace                              #migrate时指定replace

```

#### 环境搭建

```shell
#安装ruby环境
yum makecache fast
yum -y install rubygems
gem sources --remove https://rubygems.org
gem sources -a https://mirrors.aliyun.com/rubygems/
gem update - system
gem install redis -v 3.3.5
history
```

#### 创建集群

```shell
#创建集群
redis-cli --cluster create 172.16.1.111:6380 172.16.1.112:6380 172.16.1.113:6380 172.16.1.111:6381 172.16.1.112:6381 172.16.1.113:6381 --cluster-replicas 1

#添加节点进集群
redis-cli --cluster add-node 172.16.1.111:6391 172.16.1.111:6380

#将节点从集群移除
redis-cli --cluster del-node  172.16.1.111:6391 7196afc4bbe72fb872b196ef363fa6c9849b5d9
```

#### 扩容，收缩集群

```shell
#扩容，添加新的节点后，需要重新分配槽位
redis-cli --cluster reshard 172.16.1.111:6380
How many slots do you want to move (from 1 to 16384)? 4096			#每个节点分配多少个槽位
What is the receiving node ID? 5c491a9e1b96f49b2ac342c6e7839fb0e457e9ce		#接收的节点，新加入的主节点
Source node #1:all		从哪些节点进行分配

#收缩，将节点移除，进行槽位收缩
redis-cli --cluster reshard 172.16.1.111:6380
How many slots do you want to move (from 1 to 16384)? 1365	#要移除多少槽位，因为只能一个一个填写接收的节点，所以需要将总的熟练/接收节点数量
What is the receiving node ID? 5c491a9e1b96f49b2ac342c6e7839fb0e457e9ce
Please enter all the source node IDs.
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
Source node #1: 2206b5d4c30b34c21982c74a64d3f5c01bc84147
Source node #2: done 
```

#### 查看集群

```shell
#检查集群节点槽位
redis-cli --cluster  check 172.16.1.111:6381

#检查集群误差
[root@Redis redis-6.0.10]# redis-cli --cluster rebalance 172.16.1.111:6380
>>> Performing Cluster Check (using node 172.16.1.111:6380)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
*** No rebalancing needed! All nodes are within the 2.00% threshold.
```

#### 数据导入导出

```shell
#不加copy参数相当于mv，老数据迁移成功就删掉了
redis-cli --cluster import 172.16.1.111:6380 --cluster-from 172.16.1.111:6379
#添加copy参数相当于cp,老数据迁移成功后会保留
redis-cli --cluster import 172.16.1.111:6380 --cluster-copy --cluster-from 172.16.1.111:6379 
#添加replace参数会覆盖掉同名的数据，对新集群新增加的数据不受影响
redis-cli --cluster import 172.16.1.111:6380 --cluster-copy --cluster-replace --cluster-from  172.16.1.111:6379 
```

### 限制使用内存大小

```shell
vim redis.conf
maxmemory 	#一般为操作系统的一半
```

### 动态调整内存故障

```shell
config get maxmemory				#获取当前最大内存的大小
config set maxmemory 32212254720	#修改内存大小
config set maxmemory 0				#设置为0，不限制内存使用大小
```

```shell
vm.overcommit_memory = 1			#直接放行
vm.overcommit_memory = 0 			#则比较，此次请求分配的虚拟内存大小和系统当前空闲的物理内存加上swap，觉定是否放行
vm.overcommit_memory = 2			#则会比较 进程所有已分配的虚拟内存大小加上此次请求分配的虚拟内存和系统当前的空闲物理内存加上swap，决定是否放行
```

