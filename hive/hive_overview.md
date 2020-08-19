## Hive详解
[国外hive知识网站](https://www.educative.io/collection/page/6089575797620736/6248215343005696/6394690049933312#bhive-viewsb"%3Ehttps://www.educative.io/collection/page/6089575797620736/6248215343005696/6394690049933312#bhive-viewsb%3C/a%3E)

大纲：

* 简述
    * 架构

* 元数据/表
    * 表类型
        * 分区/分桶
        * 内表/外表
    * 数据类型
    * 视图/索引

* 计算
    * join 类型
    * UDF/UDTF/UDAF
    * HQL执行原理
    * MR执行引擎
    * MR  Shuffle
    * 数据倾斜
    * join 原理类型

* 数据存储
    * 文件格式
    * 压缩



### 简述 
Hive是一个基于Hadoop 的结构化语言计算引擎，它本质上不是一个数据库，他只是将用户数据的SQL语言进行解析处理成MR/Spark的计算程序执行。

* 元数据： hive会将表的元数据信息，比如表名，存放位置，等等放在关系型数据库中，如mysql derby
* 计算引擎: hive 可以将SQL翻译成 一个个的MR/Spark 程序，分步骤的再Hadoop集群上执行并返回最终结果
* 数据存储：表数据，中间结果都是存放在Hdfs中的，而且基于hdfs的副本机制可以确保数据得可用性。

#### 架构
![72cdc9cf03e1c0f021f238761b684b59.png](en-resource://database/7172:0)



### 元数据/表
#### 表类型
再hive 中有几种表得选择：分区，分桶，内表，外表
##### 分区/分桶
###### 分区
分区是通过分区字段的值对数据进行文件夹的划分，经典的比如每日的数据分区，每日的数据为一个文件夹。
语法：partitioned by ( xxx String) .
###### 分桶

分桶表是 通过对分桶id 进行hash 然后决定记录的存放文件，使用文件划分为桶，桶其实就是在入表的时候对分桶KEy做了一层hash，然后分到不同的文件中去，可以对后续倍数或者因子数量表Join的时候简化 shuffle 。分桶内数据可以选择是否排序,如下:
语法： clustered by (字段) [sorted by (字段)] into xx buckets 

分桶表的特点：

分桶是基于对指定列值 使用 hash function 后取模存储的概念。

* i.根据列的不同类型会采用不同的hash函数。bucket 列值相同的列会存放到相同的bucket中。
* 每一个桶仅用一个文件存储
* 分桶表会创建均匀大小的桶文件。
  优点：

* 与非桶表，桶表可以提供更高效的抽样（<u>tablesample（bucket x out of y) </u>）
* 在做mapjoin时，桶表由于均匀的文件大小所以会有更高效的性能。
* 桶表可以避免产生shuffle 所以会提供更高效的join
  缺点：

* 需要自己处理插入表，使用load data 不起效果。同时写表需要一定的shuffle消耗。


<u>注意：set hive.enforce.bucketing = true;开启这个配置，hive会根据表得桶数量适配对应的reduce，否则需要设置reduce task 数量 然后select。。fromcluster by xxx sort by xxx</u>


##### 内表外表
对比：

1. 删除表之后，文件目录下的数据以及元数据都会被删除，而删除外表的时候不会删除文件，只会删除元数据。
2. 外表一般是目录有数据之后才创建的表，而内表一般是load 数据进去。

#### 数据类型

**基础数据类型**：
* tinyint  一个byte有符号整数
* smallint  两个byte有符号整数
* int  4byte有符号整数（一个字节8位，就是32位，首位作符号，那就是31位2进制的最大数字）
* bigint 8byte 有符号整数
* boolean true/false
* float  单精度浮点数
* double  双精度浮点数
* string  字符串
* timesteamp 时间戳
* binary 字节数组
  两个基础类型作比较时，会隐式转成大的那个

**集合数据类型：**

* struct 和C语言中的struct/对象类似，都可以通过 “点”符号进行访问 如 字段.first
* map map是一个组键值对，使用数组访问法，例如：字段["key"]
* array 数组是具有相同类型名称的变量集合，这些变量成为数组的元素，每个数据元素都有一个编号，使用 [1] 访问。


##### 
#### 视图/索引
**视图：** 由于目前使用的hive 不支持物化视图，所以视图再目前的hive里面只是保存了一个查询的 逻辑结构。当一个查询引用一个视图的时候，这个视图锁定义的查询语句将和用户的查询语句组合在一起，然后提供hive指定查询计划（所以其实实际上只是美化SQL）

**索引：** 
**机制原理**：
hive索引其实是一张索引表，在表里面存储索引列的值，该值对应的HDFS的文件路劲， 该值在数据文件中的偏移量。
当Hive通过索引列查询时首先 通过一个mr job 去查询索引表，根据索引列的过滤条件，查询出该索引列值对应的HDFS 文件目录以及偏移量，并且把这些数据输出到HDFS的一个文件中然后再根据这个文件中去筛选源文件，所谓查询job的输入。

**创建索引**：
```
create index test_index on table test(id) as 
'org.apache.hadoop.hive.ql.index.compact.CompactIndexHandler'
with deferred rebuild
in table test;
```
**生成索引数据**：
刚刚建完hive 索引表之后是没有数据的，需要生成索引数据
```
alter index test_index on test rebuild;
```

**缺点：**

* 使用过程繁杂
* 需用额外的job扫描索引表
* 不会自动刷新，如果表数据变动 ，索引需要手动刷新


### 计算
#### HQL执行原理
#### MR shuffle过程
#### MR shuffle
#### 数据倾斜
#### UDF/UDAF/UDTF


#### join类型

* map side join
* reduce side join
* bucket map side join
* Sort-Merge-Bucket join
* left semi join






### 数据存储
#### 文件格式
#### 文件压缩





### [实际问题]()：
#### 取top n 问题：
排序函数：

* rank()
* dense_rank()
* row_number()