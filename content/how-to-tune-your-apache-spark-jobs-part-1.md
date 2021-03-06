Title: Apache Spark性能调优（一）
Slug: how-to-tune-your-apache-spark-jobs-part-1
Date: 2015-06-16
Category: Spark
Author: 杨文华
Tags: Spark, YARN, Tuning
Type: 翻译
OriginAuthor: Sandy Ryza
OriginTime: 2015-03-09
OriginUrl: http://blog.cloudera.com/blog/2015/03/how-to-tune-your-apache-spark-jobs-part-1/


原编者注：[Estimating Financial Risk with Spark](http://spark-summit.org/east/2015/talk/estimating-financial-risk-with-spark)是原文作者Sandy在Spark Summit East 2015的一个演讲，内附视频和演讲的内容。

当你写Spark代码时，需要理解transformation、action和RDD，它们是编写Spark程序的基础；当你发现任务失败，或者打算分析任务运行慢的原因时，你还需要理解job、stage和task，它们对于写出“好”的、“快”的Spark应用程序至关重要。理解Spark的底层运行机制，对于写出高效的应用程序是非常有帮助的。

通过这篇文章，你可以学到Spark程序是怎样在集群上执行的基础知识，同时还可以获得怎样写出高效的Spark应用程序的一些建议。


#####你的Spark程序是怎样执行的#####


一个Spark应用程序由一个driver进程和一组分散在集群节点上的executor进程组成。

Driver进程负责控制任务的总体流程，executor进程负责以task的形式来执行任务，同时存储用户需要cache的数据。Driver进程和executor进程一般都是常驻在应用程序的整个运行周期中的（[dynamic resource allocation](http://spark.apache.org/docs/1.2.0/job-scheduling.html#dynamic-resource-allocation)实现了动态分配executor）。每个executor都有一定数量的slots（槽位）来执行task，在executor的整个生命周期中多个task是可以并行执行的。虽然集群上的具体处理过程会跟所使用的集群管理系统有关（YARN、Mesos、Spark Standalone），但每个Spark应用程序都有driver和executor。

![pic1](/images/how-to-tune-your-apache-spark-jobs-part-1-f1.png)

Job是应用程序执行层级中的第一层，在应用程序里面执行一个action操作即会触发启动一个对应的Spark job。Spark通过检查action所依赖的RDDs来制定执行计划，这个执行计划从“最远”的RDDs（即没有其他依赖的RDDs或者是已经cache的数据）开始，到最后一个需要生成action结果的RDD结束。

执行计划会将job的transformation操作合并成stages。一个stage对应于一组执行同一段代码的task的集合，每个task处理数据的不同子集。每个stage都包含了一系列不需要shuffle所有数据就可以完成的transformations。

Spark如何确定数据是否需要shuffle呢？一个RDD是由固定数量的分区组成了，每个分区又包含了一定数量的数据记录。对于窄依赖（比如map、filter），计算一个分区的数据只来源于其父RDD的某一个分区，每一条数据记录也只对应于其父RDD中的唯一一条数据记录（特殊的，coalesce这样的transformation虽然可以依赖父RDD的多个分区，但是子RDD的某一个分区只依赖父RDD的有限分区－不需要shuffle所有数据，我们仍然认为coalesce是一种窄依赖）。

Spark还有另一种transformation称作宽依赖（比如groupByKey、reduceByKey），在宽依赖中，计算一个分区所需的数据可能来源于父RDD的多个分区。所有key相同的数据都需要被放到同一个分区中，且由同一个task来处理，为了实现这样的操作，Spark就需要执行shuffle，即让数据在集群中传输并最终落到一个新的stage和一些新的分区中。

看下面的这段代码：

<pre>
sc.textFile("someFile.txt").
  map(mapFunc).
  flatMap(flatMapFunc).
  filter(filterFunc).
  count()
</pre>

这段代码执行了一个action操作，这个action依赖一系列针对源于一个文本文件的RDD的transformation操作。这段代码会在一个stage执行，因为所有这三个transformation操作（map、flatMap、filter）的输出，都只依赖于各自的输入所对应的分区。

相反的，看看下面这段代码，目标是在文本文件中出现次数大等于1000的单词中，找出每个字符出现的次数。

<pre>
val tokenized = sc.textFile(args(0)).flatMap(_.split(' '))
val wordCounts = tokenized.map((_, 1)).reduceByKey(_ + _)
val filtered = wordCounts.filter(_._2 >= 1000)
val charCounts = filtered.flatMap(_._1.toCharArray).map((_, 1)).
  reduceByKey(_ + _)
charCounts.collect()
</pre>

整个过程可以划分为三个stages。reduceByKey是stage的边界，因为计算reduceByKey的结果需要按照key对数据进程重新分区。

下图是一个更为复杂的转换图，包含了一个join操作。

![pic2](/images/how-to-tune-your-apache-spark-jobs-part-1-f2.png)

见下图，粉色方框的部分是上图的stage划分。

![pic3](/images/how-to-tune-your-apache-spark-jobs-part-1-f3.png)

在每一个stage边界上，数据会被父stages的任务写到磁盘（译者注：本地磁盘），然后子stage的task再通过网络拉取这些数据。由于stage边界会产生大量的磁盘和网络I/O，stage边界是昂贵的，应当尽量避免。父stage的数据分区数量跟子stage的数据分区数量可能是不一样的。触发stage边界的transformation操作（比如reduceByKey）一般都接受numPartitions这个参数，以让开发人员可以决定给子stage的数据划分多少个分区。

在MapReduce的任务调优中，reducer的个数是一个重要参数，类似的，在Spark中，stage边界的分区数量调优通常也能对程序的性能起到关键性作用。在[Apache Spark性能调优（二）](/how-to-tune-your-apache-spark-jobs-part-2.html)中我们会深入探讨该数量的调优。


#####选择正确的操作#####


当一个开发人员尝试用Spark解决一个问题时，可以选择不同的actions和transformations的组合来得到想要的结果。然而并非所有的实现都具有相同的性能：避开一些常规的陷阱，选择一种正确的实现，往往可以让程序的执行效率完全不同。对于怎样选择合适的actions和transformations，一些规则可以给你指明方向。

近期[SPARK-5097](https://issues.apache.org/jira/browse/SPARK-5097)的一些工作让SchemaRDD逐渐稳定，这将向开发人员开放Spark的Catalyst优化程序，Spark可以自动选择使用那些更优的操作。待SchemaRDD成为Spark的一个稳定组件之后，对于开发者来说，某些操作的选择将是透明的。

选择合适的操作组合，主要的目的就是减少shuffle的次数和数据量，这是因为shuffle往往是昂贵的，所有的shuffle数据都必须写磁盘和走网络传输。repartition、join、cogroup，以及任何\*by、\*ByKey的transformation都会产生shuffle。并非所有的操作都具有相同的性能，新手Spark开发者常常遇到的性能陷阱是选择了错误的操作。选择合适的操作通常有以下几个规则：

+ 在做组合归约时，避免使用groupByKey。例如，rdd.groupByKey().mapValues(\_.sum)跟rdd.reduceByKey(\_ + \_)可以得到一样的结果，但是前者会让所有数据走网络传输，而后者会在每个分区中，对每个key先算出本地sum，然后在shuffle之后，使用这些本地sum计算全局sum。

+ 当输入和输出的数据类型不同时，避免使用reduceByKey。例如，想要找出每个key对应的不重复字符串，一种方法是用map操作将每条数据转换成一个Set，然后用reduceByKey来合并这些Sets：
<pre>
rdd.map(kv => (kv.\_1, new Set\[String\]() + kv.\_2))
    .reduceByKey(\_ ++ \_)
</pre>
<p>这段代码会产生大量的不必要的对象，因为每一条记录都会产生新的set对象。更好的方式是使用aggregateByKey，它可以更高效的处理map-side的累计：</p>
<pre>
val zero = new collection.mutable.Set\[String\]()
rdd.aggregateByKey(zero)(
    (set, v) => set += v,
    (set1, set2) => set1 ++= set2)
</pre>

+ 避免使用flatMap-join-groupBy这种形式的组合。当两个数据集已经按照key group了，如果这时候你想join两个数据集，然后再group，你可以改而使用cogroup，后者可以避免unpacking和repacking的消耗。


#####什么时候不产生shuffle#####

知道什么情况下上述的那些transformations不产生shuffle同样很重要。当前一个transformation已经按照同样的partitioner对数据进行了分区，Spark就可以避免产生shuffle。考虑下面这段代码：

<pre>
rdd1 = someRdd.reduceByKey(...)
rdd2 = someOtherRdd.reduceByKey(...)
rdd3 = rdd1.join(rdd2)
</pre>

因为没有给reduceByKey传递partitioner，这里会使用默认的partitioner，rdd1和rdd2都会采用hash-partitioned的方式来分区，于是这两个reduceByKey会产生两次shuffle。如果RDDs（rdd1和rdd2）有同样数量的分区，那么join操作不会产生额外的shuffle。因为上述的RDDs是使用相同的分区方式，rdd1里面同一个分区中的key一定是分布在rdd2的某一个分区里面的。所以，rdd3里面任何一个分区的数据只会依赖rdd1的某一个分区以及rdd2的某一个分区，不会出现第三次shuffle。

假设someRdd有四个分区，而someOtherRdd有两个分区，两个reduceByKey都是三个分区，task的执行是如下图所示的这样：

![pic4](/images/how-to-tune-your-apache-spark-jobs-part-1-f4.png)

但是如果rdd1和rdd2使用不同的partitioner，或者使用默认partitioner（hash）但是具有不同的分区数，结果有什么不同呢？这种情况下，只有其中一个rdd（分区数较少的那一个）需要在做join时重新shuffle。

相同的transformation操作，相同的输入，不同的分区数量（译者注：rdd1的reduceByKey用两个分区，rdd2的reduceByKey用三个分区）：

![pic5](/images/how-to-tune-your-apache-spark-jobs-part-1-f5.png)

对两个数据集做join操作时，一种避免shuffle的方法是利用[广播变量](http://spark.apache.org/docs/latest/programming-guide.html#broadcast-variables)。当其中一个数据集足够小，小到可以放进executor的内存里面时，就可以在driver上把它加载进一个hash table，然后广播到每个executor，然后map操作可以引用这个hash table来做查找（lookups）。


#####什么情况下shuffle多反而更好#####

通常我们都需要最小化shuffle的次数。当并行度增加时，额外的shuffle反而可以提升性能。例如，如果需要处理的数据由一些较大的不可分割的文件组成，那么由InputFormat指定的分区方式可能会让每个分区都分配了大量的数据，产生的分区过少，不能充分利用可用CPU核数。在这种情况下，加载完数据之后对数据做一次repartition，给一个更大的分区数（会触发shuffle），就可以让后续的操作利用更多的集群CPU。

另外一种例外的场景是使用reduce或者aggregate操作来聚合数据。由于driver负责合并计算所有executor的结果，如果在一个具有很大分区的数据上做聚合，driver的计算资源很快就会成为瓶颈。为了缓解driver的负载，可以首先使用reduceByKey或者aggregateByKey来做一次分布式的聚合，以将整个数据集分解成为更小的分区。各个分区内的数据在发送给driver做最后的聚合之前，会先在各个分区内并行的合并。可以参见[treeReduce](https://github.com/apache/spark/blob/f90ad5d426cb726079c490a9bb4b1100e2b4e602/mllib/src/main/scala/org/apache/spark/mllib/rdd/RDDFunctions.scala#L58)和[treeAggregate](https://github.com/apache/spark/blob/f90ad5d426cb726079c490a9bb4b1100e2b4e602/mllib/src/main/scala/org/apache/spark/mllib/rdd/RDDFunctions.scala#L90)的例子看它们是怎么工作的（注意：在Spark1.2中，目前最近的版本都是把他们标记为developer APIs的，但是[SPARK-5430](https://issues.apache.org/jira/browse/SPARK-5430)寻求将他们添加到Spark core的稳定版本中）。

这个技巧在需要聚合的数据已经按照key分组的情况下特别有用。假设有这样一个场景，一个Spark应用程序需要统计一个语料库中每个单词出现的次数，然后把结果汇总到driver上，生成一个map。一种方法是直接使用aggregate操作，计算每个分区上的本地map，然后合并这些map到driver上。另外一种方法是先使用aggregateByKey，在各个分区上并行的计算单词数量，然后简单的通过collectAsMap操作在driver上得到结果。


#####二次排序#####

另外一个重要需要注意的性能陷阱是repartitionAndSortWithinPartitions操作，他看起来是一个神秘的操作，但是似乎总是出现各种奇怪的现象。这个操作让排序与shuffle机制相结合，排序可以跟其他的操作合并到一起，这样就可以更高效的处理大量的数据。

Hive on Spark里join的内部实现都使用了这个操作。它同时还是[二次排序（secondary sort）](http://www.quora.com/What-is-secondary-sort-in-Hadoop-and-how-does-it-work)的重要组成部分，即当你想要对一组数据记录按照key分组，同时在遍历key对应的值的时候，让这些值能够按照特定的顺序被遍历。实际应用中，某些算法想要按照用户对事件进行分组，然后按照用户出现（译者注：如注册、访问）的先后顺序对每个用户的事件进行分析。目前利用repartitionAndSortWithinPartitions来做次要排序还需要一点额外的准备工作，不过[SPARK-3655](https://issues.apache.org/jira/browse/SPARK-3655)可以让事情变得简单很多。


#####结语#####

你现在应该对影响Spark程序运行效率的基本因素有了较好的认识。在[下一节](/how-to-tune-your-apache-spark-jobs-part-2.html)，我们会继续从资源请求、并行和数据结构等方面介绍Spark的调优。

原文作者Sandy Ryza是Cloudera的数据科学家，他同时还为Apache Hadoop和Apache Spark项目贡献代码。他是O’Reilly出版的[Advanced Analytics with Spark](http://shop.oreilly.com/product/0636920035091.do)一书的作者之一。

<div class="meta_info">
<p><span>[译文信息]</span></p>
<p>原文作者: Sandy Ryza</p>
<p>原作时间: 2015-03-09</p>
<p>原作链接: http://blog.cloudera.com/blog/2015/03/how-to-tune-your-apache-spark-jobs-part-1/</p>
</div>
