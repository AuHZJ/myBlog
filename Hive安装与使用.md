# Hive安装与使用

[toc]

---

## Hive安装

1. 到官网下载 [apache-hive-2.3.5-bin.tar.gz](http://www.apache.org/dyn/closer.cgi/hive/) 文件到`/home/au/software/hive`（目录随意）。
2. 解压该文件：`tar -zxvf apache-hive-2.3.5-bin.tar.gz -C 解压后存放的位置`。
3. 修改文件名：`mv apache-hive-2.3.5-bin hive`，或者创建软连接：`ln -s apache-hive-2.3.5-bin hive`。
4. 配置环境变量：`vi ~/.bashrc`，添加如下内容：

```shell
export HIVE_HOME=/home/au/software/hive
export PATH=$PATH:$HIVE_HOME/bin
```

5. 让环境变量生效：`source ~/.bashrc`。
6. 修改`hive`配置文件，在`hive/conf`目录下创建文件`hive-site.xml`：`touch hive-site.xml`。
7. 修改`hive-site.xml`，添加如下内容：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <property>
        <name>javax.jdo.option.ConnectionUserName</name><!--mysql用户名-->
        <value>hive</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionPassword</name><!--mysql密码-->
        <value>hive</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionURL</name><!--mysql链接地址-->
        <value>jdbc:mysql://localhost:3306/hive?createDatabaseIfNotExist=true</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionDriverName</name><!--mysql驱动包-->
        <value>com.mysql.jdbc.Driver</value>
    </property>
</configuration>
```

8. 在`mysql`中创建`hive`用户，密码也为`hive`（用户名密码随意，取名`hive`方便记忆），并允许远程登录：

```mysql
grant all on hive.* to 'hive'@'%' identified by 'hive';
flush privileges;
```

9. 上传`mysql`驱动包`mysql-connector-java-5.1.10.jar`到`hive/lib`目录下。
10. 初始化`hive`数据库：`schematool -dbType mysql -initSchema`，最后一行出现`schemaTool completed`代表初始化成功。


## 设置Hive执行环境

Hive的运行模式即任务的执行环境，分为本地与集群两种，可以通过mapred.job.tracker来指明。
设置方式：
`hive > set mapred.job.tracker.local`。

由于`hive`打印表默认没有表头，通过如下设置来显示表头：
`hvie > set hive.cli.print.header=true`。

设置`reduce`任务的个数，在比较`sort by`和`order by`的区别时需要设置，否则看不出效果：
`hive > set mapred.reduce.tasks=2`。


## Hive支持的数据类型

在`Java`中支持的数据类型在`Hive`中都支持，具体如下：

```hive
tinyint(y)    smallint(s)    int    bigint(l)
float    double    decimal
boolean
string    varchar    char
timestamp    date    interval
binary
Array数组    Map键值对    Struct对象
```




## 表的创建

展示所有的库：`show databases`。
使用某个库：`use database-name`。
展示所有的表：`show tables`。

数据导入表的方式：

> * `insert`：使用`insert`语句插入数据会转化成`MapReduce`任务。
> * `load`：使用`load`方式插入就是将文件上传至`hdfs`，即`hdfs dfs -put`。
> * 子查询。

表的类型有：

> * 内部表
> * 外部表
> * 分区表
> * 桶表

### 内部表

内部表：就是一般的表，未被`external`修饰的是内部表，与数据库中的`Table`在概念上类似，每一个`Table`在`Hive`中都有一个相应的目录存数据，存储位置在`hive.metastore.warehouse.dir`设置，默认位置是`hdfs`上的`/user/hive/warehouse`，所有的内部表数据都是保存在这个目录中，当表定义被删除的时候，表中的数据和元数据随之一并被删除。

**建立内部表：**

```hive
create table people(
    id int,
    name string
)
row format delimited       // 表中一行对应与hdfs文件的一行
fields terminated by ','   // 一行中每列用逗号分隔，如果不写这句话，那么在hdfs存储的一行数据就会紧挨在一起
stored as textfile;        // 默认值为textfile，以文本格式存放在hdfs
```

**插入数据：**

```hive
insert into people values(1, '张三');
```

**本地文件导入：**

```hive
load data local inpath '/home/au/people_data' into table people;

people_data文件内容：
1,jack
2,rose
3,merry
4,lisa
```


**`hdfs`文件导入：**

```hive
load data inpath '/people_data' into table people;
```

也可以直接将文件上传至`hdfs`上该表存放的路径，我这里是`/user/hive/warehouse/au.db/people/`，因为导入文件的时候实际上就是将文件上传至该目录：

```shell
hdfs dfs -put /home/au/people_data /user/hive/warehouse/au.db/people/
```

**子查询插入数据：**

建表时利用子查询：

```hive
create table people_back
as
select * from people;
```

插入时利用子查询：

```hive
insert into/overwrite table people
select * from people_back;
```


### 外部表

外部表：被`external`修饰的是外部表，数据存在与否和表的定义互不约束，仅仅只是表对`hdfs`上相应文件的一个引用，即把`hdfs`中某个目录下的数据当做表来使用，当删除表定义的时候，表中的数据依然存在，数据存放路径不在`/user/hive/warehouse`。

创建外部表：

1. 先在hdfs根目录创建一个`ext_table`目录，并将本地文件`student_data`（事先创建好）上传至该目录：

```shell
hdfs dfs -mkdir /ext_table
hdfs dfs -put /home/au/student_data /ext_table/

student_data文件内容：
1,jack
2,rose
3,merry
4,lisa
```

2. 创建`student`表

```hive
create table student(
    id int,
    name string
)
row format delimited       // 表中一行对应与hdfs文件的一行
fields terminated by ','   // 一行中每列用逗号分隔，如果不写这句话，那么在hdfs存储的一行数据就会紧挨在一起
stored as textfile         // 默认值为textfile，以文本格式存放在hdfs
location '/ext_table';     // 指定student_data数据文件所在位置
```

当删除`student`表时，只会删除`hive`中`student`表的定义，`ext_table`目录中的数据并不会被删除。



### 分区表

分区底层实现逻辑为：与`Mapreduce`类似，在一个表对应的目录下，一个分区对应一个目录。
分区表使用场景：单表数据量巨大，而且查询又经常限定某一个类别。

**分区的指标只能是伪列，即不能用表中字段作为分区指标。**

创建分区表：

```hive
create table teacher(
    id int,
    name string
)
partitioned by (month string)  // 不能用表中字段作为分区指标
row format delimited
fields terminated by ','
stored as textfile;
```

导入数据：

```hive
load data local inpath '/home/au/teacher_data' into table teacher partition(month='1');
load data local inpath '/home/au/teacher_data' into table teacher partition(month='2');
load data local inpath '/home/au/teacher_data' into table teacher partition(month='3');

teacher_data文件内容：
1,jack
2,rose
3,merry
4,lisa
```

导入完后查询数据：

```hive
hive > select * from teacher;

id	name	month
1	jack	1
2	rose	1
3	merry	1
4	lisa	1
1	jack	2
2	rose	2
3	merry	2
4	lisa	2
1	jack	3
2	rose	3
3	merry	3
4	lisa	3


hive > show partitions teacher;

partition
month=1
month=2
month=3
```

在`hdfs`上查看`/user/hive/warehouse/au.db/teacher`目录可查看到有三个目录：`month=1，month=2，month=3`。


###　桶表

桶表：将大表进行哈希散列抽样存储，方便做数据和代码验证，**桶表中的数据只能从其它表中用子查询进行插入**。比如将表分成`10`份，每次只拿其中的十分之一来使用，可以快速的得到结果。
分桶底层实现逻辑：在表对应的目录下，将源文件拆分成`N`个小文件。
使用场景：对于一个庞大的数据集我们经常需要拿出来一小部分作为样例，然后在样例上验证我们的查询，优化我们的程序，利用分桶可以实现数据的抽样。

开启分桶功能：

```hive
hive > set hive.enforce.bucketing=true;
```

创建桶表：

```hive
create table teacher_bucket(
    id int,
    name string
)
clustered by(id) into 2 buckets // 根据id分成2桶
row format delimited
fields terminated by ','
stored as textfile;
```

插入数据：

```hive
insert into table teacher_bucket
select * from student; // 上面创建的student表作为输入
```

查出某一桶：

```hive
select * from teacher_bucket
tablesample(bucket 1 out of 2 on id); // 查第一桶

在`hdfs`上查看`/user/hive/warehouse/au.db/teacher_bucket`目录可查看到有两个个目录：`000000_0，000001_0`。
```



## Array、Map、Struct的使用

`Hive`是支持复合类型的，有`Array`、`Map`、`Struct`三种，并且他们可以嵌套使用，`Array`是通过下标取值，`Map`是通过`key`取值，`Struct`是通过。

### Array

创建带有`Array`类型的列的表：

```hive
create table tab_array(
    a array<int>,
    b array<string>
)
row format delimited
fields terminated by '\t' // 每列以制表符\t分隔
collection items terminated by ',' // 每个array中的数据以逗号,分隔
stored as textfile;
```

导入数据：

```hive
load data local inpath '/home/au/tab_array' into table tab_array;

tab_array文件中数据如下（建议不要直接复制，中间的\t可能会变成四个空格，这样就失效了）：
1,2,3,4 hello,world
5,6,7   my,name
8,9,0   is,java
```

查询数据（`array`通过下标访问，如`array[0]`）：

```hive
查询全部数据：
hive > select * from tab_array;
结果：
[1,2,3,4]	["hello","world"]
[5,6,7]	["my","name"]
[8,9,0]	["is","java"]

通过数组下标访问：
hive > select a[0],b[1] from tab_array;
结果：
1	world
5	name
8	java


```

### Map

创建带有`Map`类型的列的表：

```hive
create table tab_map(
    a map<int,string>,
    b map<string,int>
)
row format delimited
fields terminated by '\t' // 每列以制表符\t分隔
collection items terminated by ',' // 每个map中的数据以逗号,分隔
map keys terminated by ':' // map中key，value以冒号:分隔
stored as textfile;
```

导入数据：

```hive
load data local inpath '/home/au/tab_map' into table tab_map;

tab_map文件中数据如下：
1:jack,2:rose   merry:3,lisa:4
5:maria mark:6,space:7,deep:8
```

查询数据（`map`通过`key`访问，如`map[1]`，`map["merry"]`）：

```hive
hive > select * from tab_map;
结果：
{1:"jack",2:"rose"}	{"merry":3,"lisa":4}
{5:"maria"}	{"mark":6,"space":7,"deep":8}


查询某个map值：
hive > select a[1], b["merry"] from tab_map;
结果：
jack	3
NULL	NULL
```

### Struct

`Struct`和`C`语言中的结构体类似，也可以看成`Java`中的一个类，每一行数据中的`Struct`代表一个对象。

创建带有`struct`类型的列的表：

```hive
create table tab_struct(
    id int,
    info struct<age:int,tel:string>
)
row format delimited
fields terminated by '\t' // 每列以制表符\t分隔
collection items terminated by ',' // 每个struct中的数据以逗号,分隔
stored as textfile;

create table tab_struct(
    id int,
    info struct<age:int,tel:string>
)
row format delimited
fields terminated by '\t'
collection items terminated by ','
stored as textfile;
```

导入数据：

```hive
load data local inpath '/home/au/tab_struct' into table tab_struct;

tab_struct文件中数据如下：
1    19,18788881111
2   20,13898761827
```

查询数据（`struct`通过点来访问属性，如`info.age`，`info.tel`）：

```hive
hive > select * from tab_struct;
结果：
1	{"age":19,"tel":"18788881111"}
2	{"age":20,"tel":"13898761827"}

hive > select id, info.age, info.tel from tab_struct;
结果：
1	19	18788881111
2	20	13898761827
```
