# Git

### 配置用户信息：

```shell
git config --global user.name name			#名称
git config --global user.email qq@qq.com	#邮箱
git config --global color.ui true			#语法高亮
```

### .git目录：

```shell
branches	#分支目录
config		#定义项目特有的配置选项
desription	#仅供git web程序使用
HEAD		#指示当前的分支
hooks		#包含git钩子文件
info		#包含一个全局排除文件（exclude文件）
objects		#存放所有数据内容，有info和pack两个子文件夹
refs		#存放指向数据(分支)的提交对象的指针
index		#保存暂存区信息，在执行git init的时候，这个文件还没有
```

### 总结：

```shell
#如何真正意思上通过版本控制系统 管理文件
1. 工作目录必须有个代码文件
2. 通过 git add file 添加到暂存区域
3. 通过 git commit -m "你自己输入的信息" 添加到本地仓库
```

### 常用命令：

```shell
1、git init				#初始化仓库，把一个目录初始化为版本仓库(可以是空目录，也可以是带内容的目录)
2、git status			#查看仓库当前状态
3、git add file			#添加文件到暂存区
4、git add  .	 or git add *	#添加所有文件到暂存区
5、git rm --cache file			#将文件从暂存区撤回到工作区，然后直接删除文件
6、git rm -f file				#直接从暂存区同工作区一同删除文件命令
7、git commit -m					#从暂存区提交到本地仓库
8、git mv old_name new_name		#直接更改文件名称，更改完直接commit提交即可
9、git diff						#默认比对工作区和暂存区有什么不同
10、git diff --cached			#比对暂存区和本地仓库
11、git commit -am “add newfile”	#如果某个文件已经被仓库管理，再次更改此文件，直接使用一条命令提交即可	
12、git log						#查看历史提交过的信息
		-p		#查看具体的改动
		-l 		#查看最近一次
13、git reflog				#查看所有分支的所有操作记录(包括已经被删除的commit记录和reset的操作)
14、git reset --hard	哈希值	  			  #回滚数据到某一个提交
15、git log --oneline --decorate			#查看当前指针的指向
16、git branch 							#查看所有分支
17、git branch testing					#创建一个testing分支		
18、git checkout testing					#切换到测试分支
19、git checkout -b testing				#创建并切换到testing分支
20、git branch -d testing				#删除testing分支		
21、git merge testing					#将testing分支代码合并到master分支，先切换到master分支
22、git tag 打标签
		git tag -a v1.0(标签) -m “描述信息”		 #将代码打上版本标签并添加描述信息
		git show v1.0							 #查看v1.0的信息
		git reset --hard v2.0					 #直接还原数据到v2.0
		git tag -d v2.0							 #删除标签
23、git remote add origin git@...git				#添加远程仓库，名称为origin
24、git remote									#查看远程仓库的名称
24、git push -u origin master					#将本地仓库代码推送到远程仓库
25、git clone git@gitee.com:cloud_chl/linux.git	#将远程仓库代码下载到本地
```

