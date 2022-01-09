## MySQL

### 基本操作

#### Insert

```yaml
# 语法：
INSERT INTO tbname(字段1，字段2,...)
VALUE(值1,值2,...);

# 方言语法
INSERT INTO taname SET 字段1=值1 ， 字段2=值2, ....;
```

##### INGORE

```yaml
# 会让INSERT直插入数据库不存在的记录
INSERT [INGORE] INTO tbname ...;
```

#### Update

```yaml
UPDATE [IGNORE] tbname
SET 字段1=值1，字段2=值2,...
[WHERE 条件1 ...]
[ORDER BY ...]
[LIMIT ...];
```

#### Delete

```yaml
DELETE [IGNORE] FROM tbname
[WHERE 条件1，条件2，....]
[ORDER BY .....]
[LIMIT ...];
```



### 基本查询

```yaml
SELECT 字段 FROM 表名;
 
# 查询表中所有数据
select * from t_emp ;

# 查询某个字段，设置别名显示
select empno, sal*12 as "income" from t_emp;
```

#### 数据分页

```yaml
SELECT ...... FROM 表名 LIMIT 0,10;

# 查询前五条数据
select empno,ename from t_emp limit 0,5;

# 查询后五条数据
select empno,ename from t_emp limit 10,5;
```

#### 结果集排序

```yaml
SELECT ...... FROM ...... ORDER BY 列名 [ASC | DESC];（默认ASC升序）

# 按照sal字段升序
select empno,ename,sal,deptno from t_emp order by sal;

# 按照sal字段降序
select empno,ename,sal,deptno from t_emp order by sal desc;
```

#### 多个排序字段

```yaml
# 按照sal字段降序，数据相同，按照hiredate升序   
select empno,ename,sal,hiredate from t_emp order by sal desc,hiredate ASC;
```

#### 去除重复记录

```yaml
SELECT DISTINCT 字段 FROM ......;

# 去除job字段中的重复数据
select distinct job from t_emp te ;
```

#### 条件查询

```yaml
SELECT ...... FROM ...... WHERE 条件 [AND | OR] 条件 ......;

# 查询满足两个条件的数据
select empno,ename,sal from t_emp te where (deptno=10 or deptno=20) and sal>=2000;
```

#### 聚合函数

##### AVG 

```yaml
# 求工资的平均值
select avg(sal+ifnull(comm,0)) from t_emp te ;
```

##### SUM

```yaml
# 求和所有工资的
select sum(sal) from t_emp te where deptno in(10,20);
```

##### MAX

```yaml
# 求最大值
select max(comm) from t_emp te ;
```

##### MIN

```yaml
# 求最小值
select min(comm) from t_emp te ;
```

##### COUNT

```yaml
# 统计数量
select count(*) from t_emp te ;
select count(ifnull(comm,0)) from t_emp te ;
```

#### 分组查询

```yaml
# 通过部门编号进行分组查询
select deptno, round(avg(sal)) 
from t_emp te  group by deptno ;
```

##### WITH ROLLUP

```yaml
# 
select deptno,avg(sal),sum(sal),max(sal),min(sal),count(*) 
from t_emp te 
group by deptno with rollup ;
```

##### GROUP_CONCAT

```yaml
select deptno,group_concat(ename),count(*)
from t_emp te 
where sal >= 2000
group by deptno ;
```

#### Having子句

```yam
select deptno
from t_emp te 
group by deptno having avg(sal)>=2000;
```

### 常用函数

#### 数字函数

```yaml
ABS       # 绝对值               ABS(-100)
ROUND     # 四舍五入             ROUND(4.62)
FLOOR     # 强制舍位到最近的整数   FLOOR(9.9)
CEIL      # 强制舍位到最近的整数   CEIL(3.2)
POWER     # 幂函数               POWER(2,3)
LOG       # 对数函数             LOG(7,3)
LN        # 对数函数             LN(10)
SQRT      # 开平方               SQRT(9)
PI        # 圆周率               PI()
SIN       # 三角函数             SIN(1)
COS       # 三角函数             COS(1)
TAN       # 三角函数             TAN(1)
COT       # 三角函数             COT(1)
RADIANS   # 角度转换函数          RADIANS(30)
DEGREES   # 弧度转换函数          DEGREES(1)
```

#### 字符函数

```yaml 
LOWER     # 转换小写字符          LOWER(ename)
UPPER     # 转换大写字符          UPPER(ename)
LENGTH    # 字符数量              LENGTH(ename)
CONCAT    # 连接字符串            CONCAT(sal,"$")
INSTR     # 字符出现的位置        INSTR(ename,"A")
INSERT    # 插入/替换字符         INSERT("你好",1,0,"先生")
REPLACE   # 替换字符             REPLACE("你好先生","先生","女士")
SUBSTR    # 截取字符串           SUBSTR("你好世界",3,4)
SUBSTRING # 截取字符串           SUBSTRING("你好世界",3,2)
LPAD      # 左侧填充字符          LPAD("Hello",10,"*")
RPAD      # 右侧填充字符          RPAD("Hello",10,"*")
TRIM      # 去除首位空格          TRIM(" 你好先生 ")
```

#### 日期函数 

```yaml
NOW()          # 获取系统日期和时间，格式yyyy-MM-dd hh:mm:ss
CURDATE()      # 获取当前系统时间，格式yyyy-MM-dd
CURTIME()      # 获取当前系统时间，格式hh:mm:ss
DATE_FORMAT()  # 用于格式化日期，返回用户想要的日期格式  date_format(日期，格式)
格式：%Y   # 年份
     %d   # 日期
     %W   # 星期(名称)
     %U   # 本年第几周
     %h   # 小时(12)
     %s   # 秒
     %T   # 时间(24)
     %m   # 月份
     %w   # 星期(数字)
     %j   # 本年第几天
     %H   # 小时(24)
     %i   # 分钟
     %r   # 时间(12)
DATE_ADD()     # 实现日期的偏移计算，而且时间单位很灵活  date_add(日期，INTERVAL 偏移量 时间单位)
DATEDIFF()     # 计算两个日期之间相差的天数  DATEDITT(日期，日期)
```

#### 条件函数

```yaml
IFNULL    # 判断 ifnull(表达式,值)
IF        # 判断 if(表达式,值1,值2)

CASE
  WHEN 表达式 THEN 值1
  WHEN 表达式 THEN 值2
  ....
  ELSE 值N
END AS ...
```



### 表连接

#### 内连接

语法：

```yaml
select ... from t1 join t2 on 连接条件；
select ... from t1 join t2 where 连接条件；
select ... from t1, t2 where 连接条件；
```

```yaml
select te.empno,te.ename,td.dname
from t_emp te join t_dept td on te.deptno = td.deptno ;
```

#### 外连接

```yaml
```

