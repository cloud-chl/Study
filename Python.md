### Python

### 1、猜数字

```python
#!/usr/bin/python3
print('-----第一个python程序-----')
temp = input("猜一下我现在想的数字：")
guess = int(temp)
if guess == 8:
        print("猜对了！")
else:
        print("猜错了")
print("游戏结束")
```

```python
dir(__builtins__)		#查看所有内置函数
help(input)				#帮助
```

### 2、添加循环

```python
#!/usr/bin/python3
print('-----第一个python程序-----')
temp = input("猜一下我现在想的数字：")
guess = int(temp)
while guess != 8:
        temp = input("错了，重新输：")
        guess = int(temp)
        if guess == 8:
                print("猜对了！")
        else:
                if guess > 8:
                        print("大了大了")
                else:
                        print("小了小了")
print("游戏结束")
```

### 3、列表

```python
#1、增加
列表.insert(索引，数据)	#在指定位置插入数据
列表.append(数据)		  #末尾追加数据
列表.extend(列表2)		  #列表2的数据追加到列表中

#2、修改
列表[索引] = 数据			 #修改指定索引的数据

#3、删除
del 列表[索引]		      #删除指定索引的数据
列表.remove[数据]		  #删除第一个出现的指定数据
列表.pop				   #删除末尾数据
列表.pop(索引)			  #删除指定索引数据
列表.clear			   #清空列表

#4、统计
len(列表)				   #列表长度
列表.count(数据)		  #数据在列表中出现的次数

#5、排序		
列表.sort()			   #升序排序
列表.sort(reverse=True)  #降序排序
列表.reverse()		   #逆序、反转
```

### 4、元组

```python
info_tuple=("Cai",21,1.65,"Cai")

#1、取值和取索引
print(info_tuple[0])
#知道内容，取索引
print(info_tuple.index("Cai"))

#2、统计计数
print(info_tuple.count("Cai"))
# 统计元组包含元素的个数
print(len(info_tuple))

#3、元组遍历
for my_info in info_tuple:
    print(my_info)
```

### 5、字典

```python
xiaoming = {"name": "小明"}
print(xiaoming)

#1.取值
print(xiaoming["name"])

#2.增加/删除
# key不存在，就新增
# key存在，就修改
xiaoming["age"] = 18
xiaoming["name"] = "大明"
print(xiaoming)

#3.删除
xiaoming.pop("name")
print(xiaoming)

#4.统计键值对数量
print(len(xiaoming))

#5.合并字典
# 若两个字典中有重复的key，新加的字典会覆盖原有的
temp_dict = {"height": 1.75}
xiaoming.update(temp_dict)
print(xiaoming)

#6.清空字典
#xiaoming.clear()
print(xiaoming)
print("-"*30)

#7.循环遍历字典
for key in xiaoming:
    print("%s - %s" % (key,xiaoming[key]))
print("-"*30)
card_list = [
    {"name": "Cai",
     "QQ": "2450",
     "Phone": "186"},
    {"name": "Fang",
     "QQ": "1234",
     "Phone": "7890"}
]

for card_info in card_list:
    print(card_info)
```

### 6、字符串

```python
name.capitalize()		#首字母大写
name1.casefold()		#全部小写

```

