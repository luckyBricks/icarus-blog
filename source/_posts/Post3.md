---
title: Kettle运行转换/工作的几种优化方式
date: 2020-07-06 15:36:10
toc: true
categories:
	- BI
---

在PDI中定义好一个转换/工作的Schema后，定期执行ETL任务可以利用**.bat文件的计划任务执行、Carte服务的常驻后台控制、以及Spoon自身提供的简单的Job计划执行任务**的方式。当转换的数据量很大，或者是更新数据的时间变短，对Job本身的优化就变得很重要了。
<!-- more -->
先记录一下.bat文件的运行逻辑，做个简单备忘。

```powershell
D:
cd D:\data-integration  #跳转到PDI的根目录下
kitchen.bat -rep=PentahoLocal -user=Admin -pass=password -dir=/home/admin -job=testJob1 -level=basic>D:\JOB.log		#携带参数启动kitchen,存储库的位置在Pentaho Server中

<#
参数含义如下
-rep 在Spoon开发ETL过程中的存储库名称
-user -pass 进入存储库的账户名和密码
-dir 存储.kjb .ktl文件的存储库绝对路径
-job 运行的工作名，不带后缀
-level 导出日志的等级
#>
```



首先展示的是一个未经优化的Job在运行后的日志记录，耗时87s

![未经优化的运行时间](https://656e-env-iybewaod-1257393063.tcb.qcloud.la/_posts/post3/1.png)

这个Job共有5个Trans，每个Trans中包含1次查询及Lookup匹配，并在事实表中插入/更新3-4列的数据，每次操作插入/更新的数据行在700行上下。对这个Job以及附属的相应Trans可以优化的常见操作有如下几种。

## 尽可能提升Trans中每个批次提交的数据量，减少与数据库交互的次数

![提交记录的数量调整](https://656e-env-iybewaod-1257393063.tcb.qcloud.la/_posts/post3/2.png)

提交记录的数量越大，每个Batch向数据表中写入的量就越大，就不会浪费过多时间在最耗费资源的建立数据库链接的步骤上。但是按照kettle中Trans设计的逻辑，一个Trans中的数据库操作（表插入/更新）是并行的，如果一个Batch中提交的行过多，在一个操作没有结束的时候同一个Trans的另一个操作就启动，就会将数据表锁死，导致并行的事务失效。因此应尽量避免在同一个Trans中设计关于同一个数据表或外键依赖的插入/更新操作，可以在流程中规定一次执行。也可以让并行的操作分别引用不同的索引列（SQL中写入FORCE INDEX列），避免插入数据时全表锁死。

## 调整JVM大小进行性能优化，修改Kettle定时任务中的Kitchen/Pan/Spoon脚本

在服务器允许的情况下可以尽可能提升运行ETL服务的JVM内存容量，考虑到尽可能节约内存，可以先在运行ETL时利用JMX监控ETL服务的内存占用情况，探明使用的内存峰值容量，以此为依据设置JVM的内存大小。在JVM启动时加入如下参数即可利用JMX监控其运行情况。

``` powershell
-Dcom.sun.management.jmxremote
-Djava.rmi.server.hostname=100.0.66.1 # JVM运行的本地地址
-Dcom.sun.management.jmxremote.port=9999 #JVM运行的本地端口
-Dcom.sun.management.jmxremote.ssl=false
-Dcom.sun.management.jmxremote.authenticate=false
```

 需要注意的是，这几个参数在使用时要连在一起，实际使用过程中发现，如果这几个参数中间有夹杂其他的JVM参数，则可能无法开启JMX的远程访问。 

| 修改启动脚本的代码片段           |
| :------------------------------- |
| 样例：OPT=-Xmx1024m **-Xms512m** |

``` powershell
 set OPT=-Xmx512m 
 -cp %CLASSPATH% 
 -Djava.library.path=libswt\win32\ 
 -DKETTLE_HOME="%KETTLE_HOME%" 
 -DKETTLE_REPOSITORY="%KETTLE_REPOSITORY%" 
 -DKETTLE_USER="%KETTLE_USER%" 
 -DKETTLE_PASSWORD="%KETTLE_PASSWORD%" 
 -DKETTLE_PLUGIN_PACKAGES="%KETTLE_PLUGIN_PACKAGES%" 
 -DKETTLE_LOG_SIZE_LIMIT="%KETTLE_LOG_SIZE_LIMIT%"
```

参数参考：

**-Xmx1024m**：设置JVM最大可用内存为1024M。 

 **-Xms512m**：设置JVM促使内存为512m。此值可以设置与-Xmx相同，以避免每次垃圾回收完成后JVM重新分配内存。  

**-Xmn2g**：设置年轻代大小为2G。**整个JVM内存大小=年轻代大小 + 年老代大小 + 持久代大小**。持久代一般固定大小为64m，所以增大年轻代后，将会减小年老代大小。此值对系统性能影响较大，Sun官方推荐配置为整个堆的3/8。  

**-Xss128k**：设置每个线程的堆栈大小。JDK5.0以后每个线程堆栈大小为1M，以前每个线程堆栈大小为256K。更具应用的线程所需内存大小进行调整。在相同物理内存下，减小这个值能生成更多的线程。但是操作系统对一个进程内的线程数还是有限制的，不能无限生成，经验值在3000~5000左右。

## 调整提升转换的属性

![转换属性的调整](https://656e-env-iybewaod-1257393063.tcb.qcloud.la/_posts/post3/3.png)

提升每个Trans中的存储容量，也能提升表插入/更新的Batch容量。同理，使用线程池链接数据库也可以达到优化的效果。

## Kettle自身优化的其他常用注意点

1. 尽量使用缓存，缓存尽量大一些（主要是文本文件和数据流）；

2. 可以使用sql来做的一些操作尽量用sql，Group , merge , stream lookup,split field这些操作都是比较慢的，想办法避免他们.，能用sql就用sql；

3. 插入大量数据的时候尽量把索引删掉；

4. 尽量避免使用update , delete操作，尤其是update,如果可以把update变成先delete,  后insert；

5. 能使用truncate table的时候，就不要使用deleteall row这种类似sql合理的分区，如果删除操作是基于某一个分区的，就不要使用delete row这种方式（不管是deletesql还是delete步骤）,直接把分区drop掉，再重新创建；

6. 尽量缩小输入的数据集的大小（增量更新也是为了这个目的）；

7. 尽量使用数据库原生的方式装载文本文件(Oracle的sqlloader, mysql的bulk loader步骤)；

8. 尽量不要用kettle的calculate计算步骤，能用数据库本身的sql就用sql ,不能用sql就尽量想办法用procedure,实在不行才是calculate步骤；

9. 要知道你的性能瓶颈在哪，可能有时候你使用了不恰当的方式，导致整个操作都变慢，观察kettle log生成的方式来了解你的ETL操作最慢的地方；

10. 远程数据库用文件+FTP的方式来传数据，文件要压缩。（只要不是局域网都可以认为是远程连接）

## ETL过程中的索引使用

**在ETL过程中的索引需要遵循以下使用原则：**

1. 当插入的数据为数据表中的记录数量10%以上时，首先需要删除该表的索引来提高数据的插入效率，当数据全部插入后再建立索引。

2. 避免在索引列上使用函数或计算，在where子句中，如果索引列是函数的一部分，优化器将不使用索引而使用全表扫描。

3. 避免在索引列上使用 NOT和 “!=”，索引只能告诉什么存在于表中，而不能告诉什么不存在于表中，当数据库遇到NOT和 “!=”时，就会停止使用索引转而执行全表扫描。

4. 索引列上用 >=替代 >

   **高效：select * from temp where deptno>=4**

   **低效：select * from temp where deptno>3**

   两者的区别在于，前者DBMS将直接跳到第一个DEPT等于4的记录而后者将首先定位到DEPTNO=3的记录并且向前扫描到第一个DEPT大于3的记录。

## ETL数据抽取的SQL优化

1. Where子句中的连接顺序
2. 删除全表是用TRUNCATE替代DELETE
3. 尽量多使用COMMIT
4. 用EXISTS替代IN
5. 用NOT EXISTS替代NOT IN
6. 优化GROUP BY
7. 有条件的使用UNION-ALL替换UNION
8. 分离表和索引

## 总结

如上的一些角度和方法应根据实际的情况选用优化的方式，在对开篇的那个Job进行了一定的优化后，大幅地降低了每次ETL过程的时间，效果显著。

![优化效果](https://656e-env-iybewaod-1257393063.tcb.qcloud.la/_posts/post3/4.png)

从Log中可以发现，执行这个ETL花了5s左右，而大部分的时间用来启动Kitchen服务了，如果要求的ETL间隔时间更短的话，就无法在时效内完成ETL的流程。因此我也在考虑使用Carte，由PDI集成的一个远程Kettle管理服务器，保持Kitchen进程在后台的始终活跃，大幅降低加载耗时。