## Ansible

```shell
# 常用模块
# setup
ansible all -i hosts -m setup -a "filter=*address*"
172.16.1.93 | SUCCESS => {
    "ansible_facts": {
        "ansible_all_ipv4_addresses": [
            "172.16.1.93"
        ], 
        "ansible_all_ipv6_addresses": [], 
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false
}

# ansible批量管理服务介绍
# 1、ansible批量管理服务意义

01. 提高工作效率
02. 提高工作准确度
03. 减少维护的成本
04. 减少重复性工作
    
# 2、ansible批量管理服务功能

01. 可以实现批量系统操作配置
02. 可以实现批量软件服务部署
03. 可以实现批量文件数据分发
04. 可以实现批量系统信息收集

# 3、ansible批量管理服务部署

1）用脚本将控制端秘钥分发到被控端主机

将要分发的ip写入ip.txt中
    #!/bin/bash
    for ip in `cat ip.txt`
    do
        echo “=========host $ip pub-key is start==========”
        sshpass -pqweasd ssh-copy-id -i /root/.ssh/id_rsa.pub root@172.16.1.$ip “-o StrictHostKeyChecking=no ” &> /dev/null
        if [ $? -eq 0 ];then
            echo “host $ip is successed”
        fi
        echo “=========host $ip pub-key is end===========”
    done
2）安装部署软件

yum -y install ansible            –需要依赖epel的yum源
    
/etc/ansible/ansible.cfg          –ansible服务配置文件
/etc/ansible/hosts                –主机清单文件
/etc/ansible/roles                –角色目录
3）编写主机清单文件

vi /etc/ansible/hosts
4）测试是否可以管理多个主机

脚本 hostname
ansible all -a “command” 
[root@m01 scripts]# ansible all -a “hostname”
172.16.1.41 | CHANGED | rc=0 >>
backup01
172.16.1.8 | CHANGED | rc=0 >>
web02
172.16.1.9 | CHANGED | rc=0 >>
web03
172.16.1.31 | CHANGED | rc=0 >>
nfs01
172.16.1.7 | CHANGED | rc=0 >>
web01
# 4、ansible服务架构信息

1）主机清单配置
2）软件模块信息
3）基于秘钥连接主机
4）主机需要关闭selinux
5）软件剧本功能
5、ansible软件模块应用

ansible官网：https://docs.ansible.com/
```

## Host-Pattern

```yaml
逻辑与
	ansible "websrvs:&dbsrvs" -m ping  # 在websrvs组并且在dbsrvs组中的主机
	
逻辑非
	ansible	'websrvs:!dbsrvs' -m ping  # 在websrvs组，但不在dbsrvs族中的主机
    注意：此处为单引号
    
综合逻辑：
	ansible 'websrvs:dbsrvs:&appsrvs:!ftpsrvs' -m ping

正则表达式：
	ansible "websrvs:&dbsrvs" -m ping
	ansible "~(web|db).*\.magedu\.com" -m ping
```

## Ansible-Modules

### Command

```shell
# 模块的应用：
ansible 主机名称/主机组名称/主机地址信息/all -m(指定应用的模块信息) 模块名称 -a(指定动作信息) “执行的命令”

# command(默认) 注：此模块不支持 $VARNAME < > | ; & 等，用shell模块实现 
command — Executes a command on a remote node
            在一个远程主机上执行一个命令
# 简单用法：
[root@m01 scripts]# ansible 172.16.1.31 -m command -a “hostname”
172.16.1.31 | CHANGED | rc=0 >>
nfs01

# 扩展应用：
1）chdir – Change into this directory before running the command.
            在执行命令之前对目录进行切换
ansible 172.16.1.31 -m command -a “chdir=/tmp touch file1.txt”
2)creates – If it already exists,this step wom’t be run.
             如果文件已存在，不执行命令操作
[root@m01 scripts]# ansible 172.16.1.31 -m command -a “creates=/tmp/file1.txt touch file1.txt”
172.16.1.31 | SUCCESS | rc=0 >>
skipped, since /tmp/file1.txt exists
3)removes – If it already exists,this setp will be run.
             如果文件已存在，这个步骤将执行
[root@m01 scripts]# ansible 172.16.1.31 -m command -a “removes=/tmp/file1.txt touch file1.txt”
172.16.1.31 | CHANGED | rc=0 >>
4)free_form(required)
The command module takes a free form command to run.
There is no parameter actually nam ed ‘free form’. See the examples!
使用command模块的时候，-a参数后面必须写上一个合法的linux命令信息
```

### Shell

```yaml
# shell(万能模块)
shell    – Execute commands in nodes
            在节点上执行操作
# 简单用法：
[root@m01 ~]# ansible 172.16.1.31 -m shell -a “hostname”
172.16.1.31 | CHANGED | rc=0 >>
nfs01
```

### Script

```yaml
# script(脚本模块)
1、编写一个脚本 
2、运行ansible命令执行脚本  
PS：script模块参数功能和commadn模块类似
```

### Copy

```yaml
# copy
copy    – Copies files to remote locations
           将数据信息进行批量分发
# 基本用法：
ansible 172.16.1.31 -m copy -a “src=/etc/hosts dest=/etc/hosts_bak”
# 扩展用法：
1、传输文件时修改属主和属组的信息
ansible 172.16.1.31 -m copy -a “src=/etc/hosts dest=/etc/hosts_bak owner=test group=test”
2、在传输文件时修改文件权限信息
ansible 172.16.1.31 -m copy -a “src=/etc/hosts dest=/etc/hosts_bak mode=777”
3、在传输文件时对文件先进行备份
ansible 172.16.1.31 -m copy -a “src=/etc/hosts dest=/etc/hosts_bak backup=yes"
4、传输一个文件并编辑文件的信息
ansible 172.16.1.31 -m copy -a “content=’test123′ dest=/etc/rsync.password”
5、remote_src
If no, it will search for src at originating/master machine.
      src参数指定文件信息，会在本地管理端服务进行查找
If yes, it will go to the remote/target machine for the src. Default is no.
      src参数指定文件信息，会从远程主机上进行查找
PS: ansible软件copy模块复制目录信息
ansible 172.16.1.31 -m copy -a “src=/test /dest/=/test”
src后面目录没有/: 将目录本身以及目录下面的内容都进行远程传输复制
ansible 172.16.1.31 -m copy -a “src=/test/ /dest/=/test”
src后面目录有/:   只将目录下面的内容都进行远程传输复制
```

### File

```yaml
# file
file – Sets attributes of files
        设置文件属性信息 
# 基本用法：
ansible 172.16.1.31 -m file -a “dest=/etc/hosts owner=test group=test mode=666”
# 扩展用法:
1、可以利用模块创建数据信息( 文件 目录 连接文件 )
state 参数
=absent        —缺席/删除数据信息
 ansible 172.16.1.31 -m file -a “dest=/test/test.txt state=absent”
 ansible 172.16.1.31 -m file -a “dest=/test/ state=absent”
=directory    —创建一个目录信息
 ansible 172.16.1.31 -m file -a “dest=/test state=directory”
 ansible 172.16.1.31 -m file -a “dest=/test/test01/test02 state=directory”
=file        —检测创建的数据信息是否存在
=hard        —创建一个硬链接文件
 ansible 172.16.1.31 -m file -a “src=/test/test.txt dest=/test_hard.txt state=hard”
=link        —创建一个软连接文件
 ansible 172.16.1.31 -m file -a “src=/test/test.txt dest=/test_hard.txt state=link”
=touch        —创建一个文件信息
 ansible 172.16.1.31 -m file -a “dest=/test/test.txt state=touch”
=recurse    —递归修改目录下的属组和属主信息
 ansible 172.16.1.31 -m file -a “dest=/test  owner=test recurse=yes”
```

### Fetch

```yaml

# fetch
fetch – 将数据从被控端拉取过来控制端 
# 基本用法：
ansible 172.16.1.31 -m fetch -a “src=/etc/hosts dest=/tmp”
```

### Yum

```yaml
# yum
name     —指定安装软件名称
state    —是否安装软件
            installed        —安装软件
            present            
            latest
            absent            —卸载软件
            removed
            
# 基本用法：
ansible 172.16.1.31 -m yum -a “name=iotop state=installed”
```

### Service

```yaml
# service
service模块：管理服务器的运行状态    停止    开启    重启
name:          —指定管理的服务名称
state：        —指定服务状态
                 started        启动
                 restarted      重启
                 stopped        停止
                 enabled        —指定服务是否开机自启动       
# 基本用法：                 
ansible 172.16.1.31 -m service -a “name=nfs state=started enabled=yes”


```

### Cron

```yaml
# cron
cron模块：批量设置多个主机的定时任务信息
  minute：    Minute when the job should run ( 0-59, * ,*/2 ,etc)
              设置分钟信息
  hour:       Hour when the job should run ( 0-23, *, */2, etc )
              设置小时信息
  day:        Day of the month the job should run ( 1-31, *, */2, etc )
              设置天信息
  month:      Month of the year the job should run ( 1-12, *, */2, etc )
              设置月信息
  weekday:    Day of the week that the job should run ( 0-6 for Sunday-Saturday, *, etc )
              设置周信息
 job:        用于定义定时任务需要做的事情  
# 基本用法：
ansible 172.16.1.31 -m cron -a “minute=0 hour=2 job=’/usr/sbin/ntpdate ntp1.aliyun.com >/dev/null 2>&1′”

# 扩展用法：
01.给定时任务设置注释信息
 ansible 172.16.1.31 -m cron -a “name=’time sync’ minute=0 hour=2 job=’/usr/sbin/ntpdate ntp1.aliyun.com >/dev/null 2>&1′”
 
02.删除指定定时任务
ansible 172.16.1.31 -m cron -a “name=’time sync’ state=absent”
PS：ansible可以删除的定时任务，只能是ansible设置好的定时任务

03.批量注释定时任务
ansible 172.16.1.31 -m cron -a “name=’time sync’ job=’/usr/sbin/ntpdate ntp1.aliyun.com >/dev/null 2>&1′ disabled=yes”
```

### Mount

```yaml
# mount
mount模块:批量进行挂载操作
   src：   需要挂载的存储设备或文件信息
   path：  指定目标挂载点目录
   fstype：指定挂载时的文件系统类型
   state：
       present/mounted        —进行挂载
       present：不会实现立即挂载，实现开机自动挂载
      *mounted：会实现立即挂载，并且会修改fstab文件，实际开机自动挂载            
       absent/unmounted    —进行卸载
       absent：会实现立即卸载，并且会删除fstab文件，禁止开机自动挂载
      *unmounted：会实现立即卸载，但是不会删除fstab文件  
```

### User

```yaml
# user
user模块：实现批量创建用户
# 基本用法：
    ansible 172.16.1.31 -m user -a “name=test01”
# 扩展用法：
01.指定用户UID
    ansible 172.16.1.31 -m user -a “name=test01 uid=666” 
02.指定用户组信息
    ansible 172.16.1.31 -m user -a “name=test03 group=test3”
    ansible 172.16.1.31 -m user -a “name=test04 groups=test2”
03.创建虚拟用户
    ansible 172.16.1.31 -m user -a “name=rsync create_home=no shell=/sbin/nologin”
04.给指定用户创建密码
PS:利用ansible程序user模块设置用户密码信息，需要将密码明文转换文密文进行设置
1)生成密文密码信息：
ansible all -i localhost, -m debug -a “msg={{ ‘密码qweasd’ | password_hash(‘sha512′,’加密校验信息123’) }}”
2)创建用户并给予密码
ansible 172.16.1.31 -m user -a ‘name=test08 password=$6$123$90hLUJ33QF7tP7CByH96s4RzRgV6f2KIUQYpKE.Ku9x0uhJV/CTt7isTYS9rR79Or4L3PJulhz9I3adDQCIkk0’
```



## Ansible-Playbook

```shell
#剧本执行
ansible-playbook hello.yml -i hosts
	-C --check		#只检测可能会发生的变化，但不真正执行操作
	--list-hosts	#列出运行任务的主机
	--limit			#主机列表，只针对主机列表中的主机执行
	-v				#显示过程 -vv -vvv更详细

#剧本加密
[root@Ansible ~]# ansible-vault encrypt yaml/hello.yml 
New Vault password: 
Confirm New Vault password: 
Encryption successfu
[root@Ansible ~]# cat yaml/hello.yml 
$ANSIBLE_VAULT;1.1;AES256
61653830373565373736323230353636316435343538303065396432663530386463623966613739
3565646466653761303566613762333238353765363435380a316138666464346462636331613338
61343738663239316361323566333233373966653961343133623264663538653263386661393532
3232386561366330350a633534626162383430653636366266616334366263643237323035393934
66323230333866343763316135383131343761626135323065353362336434346530613363393364
63343264356364303838633632303337373935653630346161623262316561613366366535666165
62656631383130643461363561613461626566383836613439303737646532333639343439366161
38663965316162376337386631623439653264616237346136656662643535643131336365373639
6433

#剧本加密查看
[root@Ansible ~]# ansible-vault view yaml/hello.yml 

#剧本修改加密
[root@Ansible ~]# ansible-vault rekey yaml/hello.yml 
Vault password: 
New Vault password: 
Confirm New Vault password: 
Rekey successful

#剧本解密
[root@Ansible ~]# ansible-vault decrypt yaml/hello.yml 
Vault password: 
Decryption successful
```

### 错误忽略

```shell
# 错误忽略
 tasks:
    - name: create new file
      file: name=/tmp/newfile state=touch
      igonre_errors: True
```

### 触发器

```yaml
# 触发器
 tasks:
    - name: install httpd
      yum: name=httpd
    - name: copy conf
      copy: src=/etc/httpd/conf/httpd.conf dest=/etc/httpd/conf/ backup=yes
      notify: restart service		# 文件修改，触发handlers
    - name: start service
      service: name=httpd state=started enabled=yes
    
  handlers:							# 定义触发器
    - name: restart service
      service: name=httpd state=restartd
```

### 标签

```yaml
# 标签,只执行某个标签，其他不执行
  tasks:
    - name: install httpd
      yum: name=httpd
      tags: inshttpd
[root@Ansible ~]# ansible-playbook yaml/setup_httpd.yaml -i hosts -t inshttpd
```

### 变量

```yaml
# 定义变量
  tasks:
    - name: install {{ pkname }}
      yum: name={{ pkname }}
    - name: start service
      service: name={{ pkname }} state=started enabled=yes
[root@Ansible ~]# ansible-playbook -i hosts -e 'pkname=vsftpd' yaml/1.yaml

#定义多个变量
vars:
 - var1: value1
 - var2: value2
 #example:
  vars:
    - pkname1: httpd
    - pkname2: vsftpd

  tasks:
    - name: install {{ pkname1 }}
      yum: name={{ pkname1 }}
    - name: install {{ pkname2 }}
      yum: name={{ pknam2 }}
```

### 迭代

```yaml
# 迭代
  tasks:
    - name: create some files
      file: name=/tmp/{{ item }} state=touch
      with_items:
         - file1
         - file2
         - file3
    - name: install some package
      yum: name={{ item }}
      with_items:
         - htop
         - sl
         - hping3
```

```yaml
# 迭代嵌套子变量
---
- hosts: node
  remote_user: root

  tasks:
    - name: add some group
      group: name={{ item }} state=present
      with_items:
        - group1
        - group2
        - group3
    - name: add some users
      user: name={{ item.name }} group={{ item.group }} state=present
      with_items:
        - { name: 'user1', group: 'group1' }
        - { name: 'user2', group: 'group2' }
        - { name: 'user3', group: 'group3' }
```



### 判断

```yaml
# 判断
  tasks:
    - name: create some files
      file: name=/tmp/{{ item }} state=touch
      when: ansible_distribution_major_version == "7"
```



## Ansible-Template

```yaml
# 文本文件，嵌套有脚本（使用模板编程语言编写）
# Jinja2语言，使用字面量，有下面形式
# 字符串：使用单引号或双引号
# 数字：整数，浮点数
# 列表：[item1， item2，...]
# 元组：(item1， item2，...)
# 字典：{key1:value1, key2:value2, ...}
# 布尔型：true/false
# 算术运算：+, -, *, /, //, %, **
# 比较操作：==, !=, >, >=, <, <=
# 逻辑运算：and, or, not
# 流表达式：For If When

templates功能：根据模块文件动态生成对应的配置文件
  templates文件必须存放于templates目录下，且命名为.j2结尾
  yaml/yml文件需和templates目录平级，目录结构如下：
  ./
  |--tempnginx.yaml
  |--templates
     |--nginx.conf.j2
```

**for**

```yaml
---
- hosts: node
  remote_user: root
  vars:
    ports:
      - 81
      - 82
  tasks:
    - name: copy conf
      template: src=/root/templates/for1.conf.j2 dest=/root/for1.conf
####################################################################
{% for port in ports %}
server {
  listen {{ port }}
}
{% endfor %}
```

