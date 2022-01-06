## MongoDB

### 安装

```yaml
# 关闭防火墙和SELINUX
systemctl stop firewalld
systemctl disable firewalld
sed -ri 's/^SELINUX=permissive/SELINUX=disabled/g' /etc/selinux/config
setenforce 0

# 在/etc/rc.local下添加如下
if test -f /sys/kernel/mm/transparent_hugepage/enabled; then
  echo never > /sys/kernel/mm/transparent_hugepage/enabled
fi

if test -f /sys/kernel/mm/transparent_hugepage/defrag; then
  echo never > /sys/kernel/mm/transparent_hugepage/defrag
fi

# 下载二进制包
https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-rhel70-4.4.11.tgz

# 新建mongodb用户和目录结构
useradd mongodb
passwd mongodb
mkdir -p /mongodb/{log,data,conf}
chown mongodb.mongodb -R /mongodb

# 解压并配置用户环境变量
echo "export PATH=/usr/local/mongodb/bin:$PATH" >> /home/mongodb/.bash_profile
source /home/mongodb/.bash_profile

# 启动
su - mongodb
mongod --dbpath=/mongodb/data --logpath=/mongodb/logs/mongodb.log --port=27017 --logappend --fork

# 配置文件
---数据存储有关
storage:
  journal:
    enabled: true
  dbPath: "/mongodb/data"   # 数据路径的位置
   
---进程控制
processManagement:
  fork: true                 # 后台守护进程
  pidFilePath: <string>      # pid文件的位置，一般不用配置，可以去掉这行，自动生成到data中
  
---网络配置有关
net:
    bindIp: <ip>             # 监听地址
    port: <port>             # 端口号，默认不配置端口号，是27017
    
---安全验证有关配置
security:
  authorization: enabled     # 是否打开用户名密码验证
  
# 利用配置文件启动和关闭
mongod -f mongo.conf 
mongod -f mongo.conf --shutdown
```



### 基本操作

```yaml
mongodb 默认存在的库
test:登录时默认存在的库
管理MongoDB有关的系统库
admin库：系统预留库，MongoDB系统管理库
local库：本地预留库，存储关键日志
config库：MongoDB配置信息库

show databases/show dbs
show tables/show conllections
use admin
db/select database()
```

#### 命令种类

##### db 对象相关命令

```yaml
db.[TAB][TAB]
db.help()
db.test.[TAB][TAB]
db.test.help()
```

##### rs 复制集相关

(replication set)

```yaml
rs.[TAB][TAB]
rs.help()
```

##### sh 分片集群

（sharding cluster）

```yaml
sh.[TAB][TAB]
sh.help()
```

#### 对象操作

##### 库的操作

```yaml
> use test
switched to db test
> db.dropDatabase()
{ "ok" : 1 }
```

##### 集合的操作

```yaml
> db.createCollection('a')
{ "ok" : 1 }
```

```yaml
当插入一个文档的时候，一个集合就会自动创建
> use cai
switched to db cai
> db.stu.insert({id:101,name:"Cai",age:21,gender:"m"})
WriteResult({ "nInserted" : 1 })
> show tables;
stu
test
> 
```

##### 文档操作

###### 数据录入

```yaml
for(i=0;i<10000;i++){db.log.insert({"uid":1,"name":"test","age":1})}
```

###### 查询数据行数

```yaml
db.log.count()
```

###### 全表查询

```yaml
db.log.find()
```

###### 每页显示50条记录

```yaml
DBQuery.shellBatchSize=50;
```

###### 按照条件查询

```yaml
db.log.find({uid:1})
```

###### 以标准的json格式显示数据

```yaml
db.log.find({uid:1}).pretty()

{
	"_id" : ObjectId("61d537be4be04e421c519022"),
	"uid" : 1,
	"name" : "test",
	"age" : 1

```

###### 删除集合中的所有记录

```yaml
db.log.remove({})
```

###### 查看集合存储信息

```yaml
db.log.totalSize()  # 集合中索引+数据压缩之后的大小
```

### 用户及权限管理

```yaml
注意：
验证库：建立用户时use到的库，在使用用户时，要加上验证库才能登录

对于管理员用户，必须在admin下创建
1.建用户时，use到的库，就是此用户的验证库
2.登陆时，必须明确指定验证库才能登录
3.通常，管理员用的验证库是admin，普通用户的验证库一般是所管理的库设置为验证库
4.如果直接登录到数据库，不进行use，默认的验证库就是test，不是我们建立的
5.从3.6版本开始，不添加bindIp参数，默认不让远程登录，只能本地管理员登录
```

#### 用户建立语法

```yaml
use cai
db.createUser({
	user: "<name>",
	pwd: "<cleartext password>",
	roles: [
		{role: "<role>",
		db: "<database>"} | "<role>",
		...
	]
})

基本语法说明：
user：用户名
pwd：密码
roles：
	role：角色名
	db：作用对象
role：root ,readWrite,read
验证数据库：
mongo -u cai -p qwe 172.16.1.100/cai
```

```yaml
用户管理例子：创建超级管理员，管理所有数据库（必须use admin再去创建）
use admin
db.createUser({
	user: "root",
	pwd: "root123",
	roles: [{role: "root", db: "admin"}]
})

验证用户：
> db.auth('root','root123')
1
> db.auth('root','123')
Error: Authentication failed.
0

查看用户：
use admin
db.system.users.find().pretty()

删除用户：（root身份登录，use到验证库）
use cai
db.dropUser("cai")
```

### ReplicationSet

```yaml
# 基本原理
基本构成是1主2从的结构，自带互相监控投票机制（Raft(MongoDB)） Paxos（mysql MGR 用的是变种）
如果发生主库宕机，复制集内部会进行投票选举，选择一个新的主库替代原有主库对外提供服务。同时复制集
会自动通知客户端程序，主库已经发生切换。应用会连接到新的主库
```

#### 环境规划

```yaml
# 三个以上的mongodb节点（或多实例）

# 多个端口：28017、28018、28019、28020

# 多套目录：
mkdir -p /mongodb/{28017,28018,28019,28020}/{conf,data,log}

# 多套配置文件：
/mongodb/28017/conf/mongod.conf
/mongodb/28018/conf/mongod.conf
/mongodb/28019/conf/mongod.conf
/mongodb/28020/conf/mongod.conf
```

#### 集群架构

##### 一主二从

一个主库

两个从库组成，主库宕机时，这两个从库都可以被选为主库。

![mongodb一主二从](.\images\mongodb一主二从.png)

当主库宕机后,两个从库都会进行竞选，其中一个变为主库，当原主库恢复后，作为从库加入当前的复制集群即可。

![mongodb一主二从2](.\images\mongodb一主二从2.png)

##### 一主一从一aribiter

  一个主库

  一个从库，可以在选举中成为主库

  一个aribiter节点，在选举中，只进行投票，不能成为主库

![mongodb一主二从2](.\images\mongodb一主一从一arb.png)

说明：

> 　　由于arbiter节点没有复制数据，因此这个架构中仅提供一个完整的数据副本。arbiter节点只需要更少的资源，代价是更有限的冗余和容错。

  当主库宕机时，将会选择从库成为主，主库修复后，将其加入到现有的复制集群中即可。

![mongodb一主二从2](.\images\mongodb一主一从一arb2.png)

#### 配置文件

```yaml
systemLog:
  destination: file
  path: /mongodb/28017/log/mongodb.log
  logAppend: true
storage:
  journal:
    enabled: true
  dbPath: /mongodb/28017/data
  directoryPerDB: true
  #engine: wiredTiger
  wiredTiger:
    engineConfig:
      cacheSizeGB: 1
      directoryForIndexes: true
    collectionConfig:
      blockCompressor: zlib
    indexConfig:
      prefixCompression: true
processManagement:
  fork: true
net:
  bindIp: 127.0.0.1,172.16.1.100
  port: 28017
replication:
  oplogSizeMB: 2048
  replSetName: my_repl
  
# 启动多个实例
mongod -f /mongodb/28017/conf/mongod.conf
mongod -f /mongodb/28018/conf/mongod.conf
mongod -f /mongodb/28019/conf/mongod.conf
mongod -f /mongodb/28020/conf/mongod.conf
```

#### 普通复制集

```yaml
# 1主2从
mongo --port 28017 admin
config = {_id: 'my_repl', members: [
	{_id: 0, host: '172.16.1.100:28017'},
	{_id: 1, host: '172.16.1.100:28018'},
	{_id: 2, host: '172.16.1.100:28019'}]
}
rs.initiate(config)

# 查询复制集状态
rs.status();
```

```yaml
# 1主1从1arbiter
mongo --port 28017 admin
config = {_id: 'my_repl', members: [
	{_id: 0, host: '172.16.1.100:28017'},
	{_id: 1, host: '172.16.1.100:28018'},
	{_id: 2, host: '172.16.1.100:28019', "arbiterOnly": true}]
}
rs.initiate(config)
```

##### 复制集管理

```yaml
# 查看复制集状态
rs.status();   # 查看整体复制集状态
rs.isMaster(); # 查看当前是否是主节点
rs.conf();     # 查看复制集配置信息

# 添加删除节点
rs.remove("ip:port")  # 删除一个节点
rs.add("ip:port")     # 新增从节点
rs.addArb("ip:port")  # 新增仲裁节点

# 连接到主节点
mongo --port 28017 admin
# 添加arbiter节点
my_repl:PRIMARY> rs.addArb("172.16.1.100:28020")
# 查看节点状态
my_repl:PRIMARY> rs.isMaster()
{
	"hosts" : [
		"172.16.1.100:28017",
		"172.16.1.100:28018"
	],
	"arbiters" : [
		"172.16.1.100:28019"
	]
}
```

##### 特殊从节点

```yaml
arbiter节点：主要负责选主过程中的投票，但是不存储任何数据，也不提供任何服务
hidden节点：隐藏节点，不参与选主，也不对外提供服务
delay节点：延时节点，数据落后主库一段时间，因为数据是延时的，也不应该提供服务或参与选主，所有通常会配合hidden（隐藏）
一般情况下会将delay+hidden一起配置使用
```

###### 延时节点

```yaml
cfg=rs.conf()
cfg.members[2].priority=0
cfg.members[2].hidden=true
cfg.members[2].slaveDelay=120
rs.reconfig(cfg)

# 取消以上配置
cfg=rs.conf()
cfg.members[2].priority=1
cfg.members[2].hidden=false
cfg.members[2].slaveDelay=0
rs.reconfig(cfg)

# 配置成功后，通过一下命令查询配置后的属性
rs.conf();
```

##### 其他操作

```yaml
# 查看副本集的配置信息
admin> rs.conf()

# 查看副本集各成员状态
admin> rs.status()

# 副本集角色切换（不要人为随便操作）
admin> rs.stepDown()
注：
admin> rs.freeze(300) # 锁定从，使其不会转变成主库
freeze()和stepDown单位都是秒

# 设置副本节点可读：在副本节点执行
admin> rs.slaveOk()
eg:
admin> use app
switched to db app
app> db.createCollection('a')
{ "ok" : 0, "errmsg" : "not master", "code" : 10107}

# 查看副本节点（监控主从延时）
admin> rs.printSlaveReplicationInfo()
source: 172.16.1.100:28018
	syncedTo: Wed Jan 05 2022 16:22:04 GMT+0800 (CST)
	0 secs (0 hrs) behind the primary 
source: 172.16.1.100:28019
	no replication info, yet.  State: (not reachable/healthy)


```

### ShardingCluster

#### 分片集群架构

**默认分片规则：**

初始1个chunk
缺省chunk大小：64MB
MongoDB：自动拆分并迁移chunk

默认是在存储数据时先在第一个节点开辟一个chunk开始存数据，当存满了以后会分裂出一个新的chunk继续存储。

在MongoDB空闲时，均衡器（balancer）会自动把第一个节点的数据进行对所有节点平均分配，也就是迁移chunk。

![mongodb分片集群](.\images\mongodb分片集群.png)

#### 环境规划

```yaml
# 十个实例
（1）configserver:38018-38020
3台构成的复制集（1主两从，不支持arbiter）38018-38020（复制集名字configsvr）
（2）shard节点：
sh1：38021-23（1主2从，其中一个节点为arbiter，复制集名字sh1）
sh2：38024-26（1主2从，其中一个节点为arbiter，复制集名字sh2）
（3）mongos：
38017
```

#### 分片集群搭建

##### shard节点

```yaml
# 目录规划
mkdir -p /mongodb/{38021..38026}/{conf,log,data}

# 第一组复制集节点：21-23（1主1从1aribiter）
systemLog:
  destination: file
  path: /mongodb/38021/log/mongodb.log
  logAppend: true
storage:
  journal:
    enabled: true
  dbPath: /mongodb/38021/data
  directoryPerDB: true
  #engine: wiredTiger
  wiredTiger:
    engineConfig:
      cacheSizeGB: 1
      directoryForIndexes: true
    collectionConfig:
      blockCompressor: zlib
    indexConfig:
      prefixCompression: true
net:
  bindIp: 127.0.0.1,172.19.232.224
  port: 38021
replication:
  oplogSizeMB: 2048
  replSetName: sh1
sharding:
  clusterRole: shardsvr
processManagement:
  fork: true
  
\cp /mongodb/38021/conf/mongodb.conf /mongodb/38022/conf
\cp /mongodb/38021/conf/mongodb.conf /mongodb/38023/conf

sed -ri 's/38021/38022/g' /mongodb/38022/conf/mongodb.conf
sed -ri 's/38021/38023/g' /mongodb/38022/conf/mongodb.conf

# 第二组复制集节点：24-26（1主1从1aribiter）
systemLog:
  destination: file
  path: /mongodb/38024/log/mongodb.log
  logAppend: true
storage:
  journal:
    enabled: true
  dbPath: /mongodb/38024/data
  directoryPerDB: true
  #engine: wiredTiger
  wiredTiger:
    engineConfig:
      cacheSizeGB: 1
      directoryForIndexes: true
    collectionConfig:
      blockCompressor: zlib
    indexConfig:
      prefixCompression: true
net:
  bindIp: 127.0.0.1,172.19.232.224
  port: 38024
replication:
  oplogSizeMB: 2048
  replSetName: sh2
sharding:
  clusterRole: shardsvr
processManagement:
  fork: true
  
\cp /mongodb/38024/conf/mongodb.conf /mongodb/38025/conf
\cp /mongodb/38024/conf/mongodb.conf /mongodb/38026/conf

sed -ri 's/38024/38025/g' /mongodb/38025/conf/mongodb.conf
sed -ri 's/38024/38026/g' /mongodb/38026/conf/mongodb.conf

# 启动所有节点，搭建复制集
su - mongodb
mongod -f /mongodb/38021/conf/mongodb.conf 
mongod -f /mongodb/38022/conf/mongodb.conf 
mongod -f /mongodb/38023/conf/mongodb.conf 
mongod -f /mongodb/38024/conf/mongodb.conf 
mongod -f /mongodb/38025/conf/mongodb.conf 
mongod -f /mongodb/38026/conf/mongodb.conf

mongo --port 38021
use admin
config = {_id: 'sh1', members: [
	{_id: 0, host: '172.19.232.224:38021'},
	{_id: 1, host: '172.19.232.224:38022'},
	{_id: 2, host: '172.19.232.224:38023', "arbiterOnly": true}]
}
rs.initiate(config)

mongo --port 38024
use admin
config = {_id: 'sh2', members: [
	{_id: 0, host: '172.19.232.224:38024'},
	{_id: 1, host: '172.19.232.224:38025'},
	{_id: 2, host: '172.19.232.224:38026', "arbiterOnly": true}]
}
rs.initiate(config)
```

##### config节点

```yaml
# 目录规划
mkdir -p /mongodb/{38018..38020}/{conf,log,data}

# 配置文件
systemLog:
  destination: file
  path: /mongodb/38018/log/mongodb.log
  logAppend: true
storage:
  journal:
    enabled: true
  dbPath: /mongodb/38018/data
  directoryPerDB: true
  #engine: wiredTiger
  wiredTiger:
    engineConfig:
      cacheSizeGB: 1
      directoryForIndexes: true
    collectionConfig:
      blockCompressor: zlib
    indexConfig:
      prefixCompression: true
net:
  bindIp: 127.0.0.1,172.19.232.224
  port: 38018
replication:
  oplogSizeMB: 2048
  replSetName: configReplSet
sharding:
  clusterRole: configsvr
processManagement:
  fork: true
  
\cp /mongodb/38018/conf/mongodb.conf /mongodb/38019/conf
\cp /mongodb/38018/conf/mongodb.conf /mongodb/38020/conf

sed -ri 's/38018/38019/g' /mongodb/38019/conf/mongodb.conf
sed -ri 's/38018/38020/g' /mongodb/38020/conf/mongodb.conf

# 启动节点，配置复制集
su - mongodb
mongod -f /mongodb/38018/conf/mongodb.conf 
mongod -f /mongodb/38019/conf/mongodb.conf 
mongod -f /mongodb/38020/conf/mongodb.conf

mongo --port 38018
use admin
config = {_id: 'configReplSet', members: [
	{_id: 0, host: '172.19.232.224:38018'},
	{_id: 1, host: '172.19.232.224:38019'},
	{_id: 2, host: '172.19.232.224:38020'} ]
}
rs.initiate(config)

注：configserve可以是一个节点，官方建议复制集。configserver不能有aribiter。
新版本中，要求必须是复制集
注：mongodb 3.4之后，虽然要求config server为relica set，但是不支持aribiter
```

##### mongos节点

```yaml
# 目录规划
mkdir -p /mongodb/38017/{conf,log}

# 配置文件
systemLog:
  destination: file
  path: /mongodb/38017/log/mongos.log
  logAppend: true
net:
  bindIp: 127.0.0.1,172.19.232.224
  port: 38017
sharding:
  configDB: configReplSet/172.19.232.224:38018,172.19.232.224:38019,172.19.232.224:38020
processManagement:
  fork: true
  
# 启动mongos
su - mongodb
mongos -f /mongodb/38017/conf/mongos.conf

# 分片集群添加节点
连接到其中一个mongos（172.19.232.224），做以下配置
1.连接到mongs的admin数据库
su - mongodb
mongo 172.19.232.224:38017/admin
2.添加分片
db.runCommand({ addshard : "sh1/172.19.232.224:38021,172.19.232.224:38022,172.19.232.224:38023",name:"shard1"})
db.runCommand({ addshard : "sh2/172.19.232.224:38024,172.19.232.224:38025,172.19.232.224:38026",name:"shard2"})
3.列出分片
db.runCommand( { listshards : 1 } )
4.整体状态查看
sh.status()
```

#### 使用分片集群

##### RANGE自动分片

```yaml
# 1.激活数据库分片功能
mongo --port 38017 admin
mongos> ( { enablesharding : "test1" } )
mongos> db.runCommand( { enablesharding : "test1" } )

# 2.指定分片键对集合分片
# 创建索引
mongos> use test1
mongos> db.tab1.ensureIndex( { id: 1 } )

# 开启分片
mongos> use admin
mongos> db.runCommand( { shardcollection : "test1.tab1",key : {id: 1} } )

# 3.集合分片验证
mongos> use test1
mongos> for(i=1;i<10000;i++){ db.tab1.insert({"id":1,"name":"cai","age":21,"date":new Date()});}
mongos> db.tab1.stats()
```

##### zone方式进行range手工定制分片

```yaml
mongo --port 38017 admin
mongos> use zonedb
mongos> db.vast.ensureIndex( {order_id: 1})

# 对于zonedb开启分片功能
mongos> use admin
mongos> db.runCommand( { enablesharding : "zonedb" } )
mongos> sh.shardCollection("zonedb.vast", {order_id: 1});

mongos> sh.addShardTag("shard1", "shard00")
mongos> sh.addShardTag("shard2", "shard01")

mongos> sh.addTagRange(
"zonedb.vast",
{ "order_id" : MinKey },
{ "order_id" : 500 },"shard00" )

mongos> sh.addTagRange(
"zonedb.vast",
{ "order_id" : 501 },
{ "order_id" : MaxKey },"shard01" )

use zonedb
for(i=1;i<1000;i++){ db.vast.insert({"id":1,"name":"ceshi","age":110,"date":new Date()});}
db.vast.getShardDistribution()
```

##### Hash分片

```yaml
对hashdb库下的vast大表进行hash
创建哈希索引
(1)对于hashdb开启分片功能
mongo --port 38017 admin
use admin
admin> db.runCommand( { enablesharding : "hashdb" } )

(2)对于hashdb库下的vast表建立hash索引
use hashdb
hashdb> db.vast.ensureIndex( { id: "hashed" } )

(3)开启分片
use admin
admin> sh.shardCollection( "hashdb.vast", { id: "hashed" } )

(4)录入10w行数据测试
use hashdb
for(i=1;i<100000;i++){ db.vast.insert({"id":1,"name":"guangzhou","age":100,"date":new Date()}); }

(5)hash分片结果测试
mongo --port 38021
use hashdb
db.vast.count()

mongo --port 38024
use hashdb
db.vast.count()
```

#### 分片集群查询及管理

```yaml
# 1.判断是否Shard集群
admin> db.runCommand({ isdbgrid : 1})

# 2.列出所有分片信息
admin> db.runCommand({ listshards : 1})

# 3.列出开启分片的数据库
admin> use config
admin> db.databases.find( { "partitioned": true } )
或者
admin> db.databases.find() //列出所有数据库分片情况

# 4.查看分片的片键
config> db.collections.find().pretty()

# 5.查看分片的详细信息
admin> sh.status()

# 6.删除分片节点
(1) 确认blance是否再工作
sh.getBalancerState()
(2) 删除shard2节点
mongos> db.runCommand( { removeShard: "shard2" } )
注意：删除操作一定会立即出发blancer
```

#### Balancer

```yaml
mongos的一个重要功能，自动巡查所有shard节点上chunk的情况，自动做chunk迁移。
什么时候工作？
1、自动运行，会检测系统不繁忙的时候做迁移
2、在做节点删除的时候，立即开始迁移工作
3、balancer只能再预设定的时间窗口内运行

开启和关闭balancer
mongos> sh.stopBalancer()
mongos> sh.startBalancer()
```

##### 自定义 自动平衡进行的时间段

```yaml
use config
sh.setBalancerState( true )
db.settings.update({ _id : "balancer"}, { $set : { activeWindow : { start : "3:00", stop, "5:00" } } }, true)

sh.getBalancerWindow()
sh.status()
```



### 备份恢复

```yaml
# mongoexport
$ mongoexport --help  
参数说明：
-h:指明数据库宿主机的IP
-u:指明数据库的用户名
-p:指明数据库的密码
-d:指明数据库的名字
-c:指明collection的名字
-f:指明要导出那些列
-o:指明到要导出的文件名
-q:指明导出数据的过滤条件
--authenticationDatabase admin

1.单表备份至json格式
mongoexport -uroot -proot123 --port 27017 --authenticationDatabase admin -d cai -c log -o /mongodb/log.json

注：备份文件的名字可以自定义，默认导出了JSON格式的数据。

2. 单表备份至csv格式
如果我们需要导出CSV格式的数据，则需要使用----type=csv参数：

 mongoexport -uroot -proot123 --port 27017 --authenticationDatabase admin -d test -c log --type=csv -f uid,name,age,date  -o /mongodb/log.csv
```

```yaml
# mongoimport
$ mongoimport --help
参数说明：
-h:指明数据库宿主机的IP
-u:指明数据库的用户名
-p:指明数据库的密码
-d:指明数据库的名字
-c:指明collection的名字
-f:指明要导入那些列
-j, --numInsertionWorkers=<number>  number of insert operations to run concurrently                                                  (defaults to 1)
//并行
数据恢复:
1.恢复json格式表数据到log1
mongoimport -uroot -proot123 --port 27017 --authenticationDatabase admin -d cai -c log1 /mongodb/log.json
2.恢复csv格式的文件到log2
上面演示的是导入JSON格式的文件中的内容，如果要导入CSV格式文件中的内容，则需要通过--type参数指定导入格式，具体如下所示：
错误的恢复

注意：
（1）csv格式的文件头行，有列名字
mongoimport   -uroot -proot123 --port 27017 --authenticationDatabase admin   -d cai -c log2 --type=csv --headerline --file  /mongodb/log.csv

（2）csv格式的文件头行，没有列名字
mongoimport   -uroot -proot123 --port 27017 --authenticationDatabase admin   -d cai -c log3 --type=csv -f id,name,age,date --file  /mongodb/log.csv
--headerline:指明第一行是列名，不需要导入。
```

MySQL ---> MongoDB

```yaml
mysql   -----> mongodb  
world数据库下city表进行导出，导入到mongodb

（1）mysql开启安全路径
vim /etc/my.cnf   --->添加以下配置
secure-file-priv=/tmp

--重启数据库生效
/etc/init.d/mysqld restart

（2）导出mysql的city表数据
source /root/world.sql

select * from world.city into outfile '/tmp/city1.csv' fields terminated by ',';

（3）处理备份文件
desc world.city
  ID          | int(11)  | NO   | PRI | NULL    | auto_increment |
| Name        | char(35) | NO   |     |         |                |
| CountryCode | char(3)  | NO   | MUL |         |                |
| District    | char(20) | NO   |     |         |                |
| Population

vim /tmp/city.csv   ----> 添加第一行列名信息

ID,Name,CountryCode,District,Population

(4)在mongodb中导入备份
mongoimport -uroot -proot123 --port 27017 --authenticationDatabase admin -d world  -c city --type=csv -f ID,Name,CountryCode,District,Population --file  /tmp/city1.csv

use world
db.city.find({CountryCode:"CHN"});

-------------
world共100张表，全部迁移到mongodb

select table_name ,group_concat(column_name) from columns where table_schema='world' group by table_name;

select * from world.city into outfile '/tmp/world_city.csv' fields terminated by ',';

select concat("select * from ",table_schema,".",table_name ," into outfile '/tmp/",table_schema,"_",table_name,".csv' fields terminated by ',';")
from information_schema.tables where table_schema ='world';

导入：
提示，使用infomation_schema.columns + information_schema.tables

mysql导出csv：
select * from test_info   
into outfile '/tmp/test.csv'   
fields terminated by ','　　　 ------字段间以,号分隔
optionally enclosed by '"'　　 ------字段用"号括起
escaped by '"'   　　　　　　  ------字段中使用的转义符为"
lines terminated by '\r\n';　　------行以\r\n结束

mysql导入csv：
load data infile '/tmp/test.csv'   
into table test_info    
fields terminated by ','  
optionally enclosed by '"' 
escaped by '"'   
lines terminated by '\r\n'; 
```

