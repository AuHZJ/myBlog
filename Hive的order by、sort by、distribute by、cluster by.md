# Hive 的 sort by 与 order by、distribute by 与 cluster by

[toc]

---

## sort by 与 order by

我们知道，在`MapReduce`中，每个分区的数据是`key`值有序的，有几个`reduce`任务就有几个分区，当只有一个分区时，数据就是全局有序的了。`sort by`的功能就是保证每个分区有序，而`order by`就相当于全局有序，即这几个分区连起来也是有序的。

为了能够看出他们的区别，我们需要提前设置`reduce`任务的个数大于`1`：

```hive
hive > set mapred.reduce.tasks=2;
```

创建测试表`sortandorder`：

```hive
create table sortandorder(
    id int
)
row format delimited
stored as textfile;
```

导入测试数据：

```hive
hive > load data local inpath '/home/au/sortandorder' into table sortandorder;

sortandorder文件数据如下：
1
3
2
9
10
8
4
7
6
5
```

用order by查询数据：

```hive
hive > select * from sortandorder order by id;
结果如下：
1
2
3
4
5
6
7
8
9
10
```

用sort by查询数据：

```hive
hive > select * from sortandorder sort by id;
结果如下：
1
2
5
7
9
10
3
4
6
8
```

由于分了两个区，可以看出`order by`是全局排序，`sort by`是区内排序，分区个数由`reduce`任务个数决定。


## distribute by 与 cluster by

`distribute by`：按照指定的字段或表达式对数据进行划分，输出到对应的`reduce`或者文件中。

`cluster by`：除了兼具`distribute by`的功能，还具有`sort by`的排序功能。

利用上面的`sortandorder`的表进行`distribute by`分区，存入本地文件`/home/au/distributeandcluster/`：

```hive
insert overwrite local directory '/home/au/distributeandcluster/'
select id from sortandorder distribute by id; // 还是用上面sortandorder的表
```

运行完后可在`/home/au/distributeandcluster/`目录下看到有两个文件`(00000_0，000001_0)`，并且两个文件内都是**无序的**。

同样，用`cluster by`进行分区，存入本地文件`/home/au/distributeandcluster/`：

```hive
insert overwrite local directory '/home/au/distributeandcluster/'
select id from sortandorder cluster by id; // 还是用上面sortandorder的表
```

运行完后可在`/home/au/distributeandcluster/`目录下看到有两个文件`(00000_0，000001_0)`，并且两个文件内都是**有序的**。