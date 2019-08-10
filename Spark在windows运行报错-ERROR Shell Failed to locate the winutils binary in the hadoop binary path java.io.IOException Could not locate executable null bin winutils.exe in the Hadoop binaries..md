# `Spark`在`windows`运行报错-`ERROR Shell Failed to locate the winutils binary in the hadoop binary path java.io.IOException Could not locate executable null\bin\winutils.exe in the Hadoop binaries.`

在`windows`的`idea`运行`spark`程序，报了如下错：

```java
Using Spark's default log4j profile: org/apache/spark/log4j-defaults.properties
19/08/06 18:53:56 INFO SparkContext: Running Spark version 2.4.3
19/08/06 18:53:56 WARN NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
19/08/06 18:53:56 ERROR Shell: Failed to locate the winutils binary in the hadoop binary path
java.io.IOException: Could not locate executable null\bin\winutils.exe in the Hadoop binaries.
	at org.apache.hadoop.util.Shell.getQualifiedBinPath(Shell.java:378)
	at org.apache.hadoop.util.Shell.getWinUtilsPath(Shell.java:393)
	at org.apache.hadoop.util.Shell.<clinit>(Shell.java:386)
	at org.apache.hadoop.util.StringUtils.<clinit>(StringUtils.java:79)
	at org.apache.hadoop.security.Groups.parseStaticMapping(Groups.java:116)
	at org.apache.hadoop.security.Groups.<init>(Groups.java:93)
	at org.apache.hadoop.security.Groups.<init>(Groups.java:73)
	at org.apache.hadoop.security.Groups.getUserToGroupsMappingService(Groups.java:293)
	at org.apache.hadoop.security.UserGroupInformation.initialize(UserGroupInformation.java:283)
	at org.apache.hadoop.security.UserGroupInformation.ensureInitialized(UserGroupInformation.java:260)
	at org.apache.hadoop.security.UserGroupInformation.loginUserFromSubject(UserGroupInformation.java:789)
	at org.apache.hadoop.security.UserGroupInformation.getLoginUser(UserGroupInformation.java:774)
	at org.apache.hadoop.security.UserGroupInformation.getCurrentUser(UserGroupInformation.java:647)
	at org.apache.spark.util.Utils$$anonfun$getCurrentUserName$1.apply(Utils.scala:2422)
	at org.apache.spark.util.Utils$$anonfun$getCurrentUserName$1.apply(Utils.scala:2422)
	at scala.Option.getOrElse(Option.scala:121)
	at org.apache.spark.util.Utils$.getCurrentUserName(Utils.scala:2422)
	at org.apache.spark.SparkContext.<init>(SparkContext.scala:293)
	at org.apache.spark.SparkContext$.getOrCreate(SparkContext.scala:2520)
	at com.au.common.Base$.main(Base.scala:17)
	at com.au.common.Base.main(Base.scala)
```

解决方案：指定一个`winutils.exe`文件即可。
1. 下载我给的`bin`目录文件。
	百度云链接：https://pan.baidu.com/s/1422rEurIxnMr6wPJr5G8UA 
	提取码：ymm2 
	复制这段内容后打开百度网盘手机App，操作更方便哦。
2. 将`bin`放到任意目录，我这里是`D:/test`目录下（注意一定要在`winutils.exe`外套上`bin`，不然还是会报错）。
3. 在代码中指定该`bin`目录的上一级，即`test`目录（实际上就是模拟`hadoop`），代码为：`System.setProperty("hadoop.home.dir", "D:/test")`。

测试代码，亲测有效（词频统计）：

```scala
import org.apache.spark.{SparkConf, SparkContext}

object Base {
  def main(args: Array[String]): Unit = {

    System.setProperty("hadoop.home.dir", "D:/test") // 加入这句代码，将下载的bin目录放到任意目录，我这里是新建的test目录

    val conf = new SparkConf().setMaster("local[*]").setAppName("wordcount")
    val sc = SparkContext.getOrCreate(conf)

    val rdd = sc.textFile("D:/markdown note/Flume学习笔记.md")
    rdd.flatMap(_.split(" ")).groupBy(x => x).mapValues(x => x.size).foreach(println)
  }
}
```