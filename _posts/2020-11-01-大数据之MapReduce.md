# MapReduce

## 简介

* 一个分布式运算的编程框架，是用户开发基于Hadoop的数据分析应用的核心框架
* 核心功能是将用户编写的业务逻辑代码和自带默认组件整合成一个完整的分布式运算程序，并发运行在一个Hadoop集群上
* 优点：易于编程、良好的扩展性、高容错性、适合PB级以上海量数据的离线处理
* 缺点：不擅长实时计算、不擅长流式计算（输入数据是动态的）、不擅长DAG（有向图）计算（多个应用程序存在依赖关系，后一个应用输入为前一个的输出，MapReduce不是不能完成，每个操作都会写入磁盘会造成大量的磁盘IO，导致性能低下）

## 核心思想

![1602427614167](https://img-blog.csdnimg.cn/20201019084725992.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyODI5ODM1,size_16,color_FFFFFF,t_70#pic_center)

1. 分布式的运算程序往往需要分成至少2个阶段  
   1. 第一个阶段的MapTask并发实例，完全并行运行，互不相干。
   2. 第二个阶段的ReduceTask并发实例互不相干，但是他们的数据依赖于上一个阶段的所有MapTask并发实例的输出。
   3. MapReduce编程模型只能包含一个Map阶段和一个Reduce阶段，如果用户的业务逻辑非常复杂，那就只能多个MapReduce程序，串行运行。
   4. 总结：分析WordCount数据流走向深入理解MapReduce核心思想。

2. 进程
   1. MrAppMaster：负责整个程序的过程调度及状态协调
   2. MapTask：         负责Map阶段的整个数据处理流程
   3. ReduceTask：    负责Reduce阶段的整个数据处理流程

## 编程规范

* Mapper阶段

  * 用户自定义的Mapper要继承自己的父类Mapper
  * Mapper的输入数据是KV对的形式（KV类型可自定义）
  * Mapper中的业务逻辑写在map（）方法中
  * Mapper的输出数据是KV对的形式（KV类型可自定义）
  * map（）方法（MapTask进程）对每一个<K，V>调用一次
* Reduce阶段

  * 用户自定义的Reduce要继承自己的父类Reduce
  * Reduce的输入数据类型对于Mapper的输出数据类型（KV）
  * Reduce的业务逻辑写在reduce（）方法中
  * ReduceTask进程对每一个相同K的<K，V>组调用一次reduce（）方法
* Driver阶段

  * 创建job对象
  * 配置jar路径
  * 配置mapper、reduce路径
  * 配置map输出的KV类型
  * 配置最终输出的KV类型
  * 配置输入、输出
  * 启动job
  * 相当于YARN集群的客户端，用于提交我们整个程序到YARN集群，提交的是封装了MapReduce程序相关运行参数的job对象


## Hadoop序列化

概述

* 序列化就是把内存中的对象，转换成字节序列（其他数据传输协议）以便于存储到磁盘持久化和网络传输
* 反序列化将收到字节序列（其他数据传输协议）或者是磁盘的持久化数据，转换成内存中的对象

特点：**紧凑、快速、可扩展性、互操作**

自定义Bean对象

* 实现Writable接口
* 构造无参构造
* 重写序列化方法（write）
* 重写反序列化方法（readFields）
* 反序列化的顺序和序列化的顺序完全一致
* 重写toString，用\t分开
* 还需实现Comparable接口（compareTo方法实现排序）

## MapReduce原理

### InputFormat数据输入  

* 数据块：Block是HDFS物理上把数据分成一块一块
* 数据切片：逻辑上对输入进行分片，并不会在磁盘将其分成片进行存储

![1603162739457](https://img-blog.csdnimg.cn/20201026085856581.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyODI5ODM1,size_16,color_FFFFFF,t_70#pic_center)

##### job提交流程源码和切片源码

```java
waitForCompletion()

submit();

// 1建立连接
	connect();	
		// 1）创建提交Job的代理
		new Cluster(getConfiguration());
			// （1）判断是本地yarn还是远程
			initialize(jobTrackAddr, conf); 

// 2 提交job
submitter.submitJobInternal(Job.this, cluster)
	// 1）创建给集群提交数据的Stag路径
	Path jobStagingArea = JobSubmissionFiles.getStagingDir(cluster, conf);

	// 2）获取jobid ，并创建Job路径
	JobID jobId = submitClient.getNewJobID();

	// 3）拷贝jar包到集群
copyAndConfigureFiles(job, submitJobDir);	
	rUploader.uploadFiles(job, jobSubmitDir);

// 4）计算切片，生成切片规划文件
writeSplits(job, submitJobDir);
		maps = writeNewSplits(job, jobSubmitDir);
		input.getSplits(job);

// 5）向Stag路径写XML配置文件
writeConf(conf, submitJobFile);
	conf.writeXml(out);

// 6）提交Job,返回提交状态
status = submitClient.submitJob(jobId, submitJobDir.toString(), job.getCredentials());

```

![1603163026191](https://img-blog.csdnimg.cn/202010260859447.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyODI5ODM1,size_16,color_FFFFFF,t_70#pic_center)

##### FileInputFormat切片源码解析  

* 先找到数据存储目录
* 开始遍历处理（规划目录下的每一个文件）
* 遍历第一个文件ss.txt
  * 获取文件大小
  * 计算切片大小
    * computeSplitSize（Math.max(minSize, Math.min(maxSize, blockSize))）
    * 本地运行为32M、集群1.x为64M、2.x为128M
  * 每次切片，都要判断切完剩下部分是否大于1.1倍不大于切划一块切片
  * 将切片消息写到一个切片规划文件中
  * 整个切片的核心过程在getSplit方法中
  * InputSplit只记录切片的元数据消息
* 提交切片规划文件到YARN上，YRAN上的MrAPPmaster就可以根据切片规划文件计算开启MapTask个数

##### FileInputFormat切片机制  

切片机制

* 简单地按照文件的内容长度进行切片
* 切片大小，默认等于Block大小
* 切片是不考虑数据集整体，而是逐个针对每一个文件单独切片

##### CombineTextInputFormat切片机制  

*  将多个小文件从逻辑上规划到一个MapTask处理  
*  虚拟存储切片最大值设置  （设置最好根据实际的小文件大小情况）

![1603166127423](https://img-blog.csdnimg.cn/20201026090025391.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyODI5ODM1,size_16,color_FFFFFF,t_70#pic_center)

​         当剩余数据大小超过设置的最大值且不大于最大值2倍，此时将文件均分成2个虚拟存储块（防止出现太小切片）  

​         判断虚拟存储的文件大小是否大于setMaxInputSplitSize值，大于等于则单独形成一个切片  

​         如果不大于则跟下一个虚拟存储文件进行合并，共同形成一个切片  

##### TextInputFormat切片机制

TextInputFormat是默认的FileInputFormat实现类，按行读取每一条记录，键是存储该行在整个文件中起始字节偏移量，

##### KeyValueTextInputFormat切片机制

每一行均为一条记录，分割为key，value。可以通过在驱动类中设置conf.set(KeyValueLineRecordReader.KEY_VALUE_SEPERATOR, " ");来设定分隔符，默认分隔符是tab（\t）

```
key1 rarawradad
key2 ewewewwe
```

NLineInputFormat切片机制

使用NLineInputFormat，代表每个map进程处理的inputSplit不再按Block块去划分，而是按NLineInputFormat指定行数划分  总行数/n 不整除=商+1

##### 自定义InputFormat

* 自定义一个类继承FileInputFormat
  * 重写isSplitable()方法，返回false不可切割
  * 重写createRecordReader（），创建自定义RecordReader对象，并初始化
* 改写RecordReader，实现一次读取一个完整文件封装未KV
  * 采用IO流一次读取一个文件输出到value中因为设置不可切片，最终把所有文件的封装到value中
  * 获取文件路径信息+名称，并设置key
* 设置Deiver
  * job.setInputFormatClass(CustomFileInputFormat.class);
  * job.setOutputFormatClass(SequenceFileOutputFormat.class)

### MapReduce工作流程

1）MapTask收集我们的map()方法输出的kv对，放到内存缓冲区中

2）从内存缓冲区不断溢出本地磁盘文件，可能会溢出多个文件

3）多个溢出文件会被合并成大的溢出文件

4）在溢出过程及合并的过程中，都要调用Partitioner进行分区和针对key进行排序

5）ReduceTask根据自己的分区号，去各个MapTask机器上取相应的结果分区数据

6）ReduceTask会取到同一个分区的来自不同MapTask的结果文件，ReduceTask会将这些文件再进行合并（归并排序）

7）合并成大文件后，Shuffle的过程也就结束了，后面进入ReduceTask的逻辑运算过程（从文件中取出一个一个的键值对Group，调用用户自定义的reduce()方法）

3．注意

Shuffle中的缓冲区大小会影响到MapReduce程序的执行效率，原则上说，缓冲区越大，磁盘io的次数越少，执行速度就越快。

缓冲区的大小可以通过参数调整，参数：io.sort.mb默认100M。

4．源码解析流程

~~~
context.write(k, NullWritable.get());

output.write(key, value);

collector.collect(key, value,partitioner.getPartition(key, value, partitions));

​    HashPartitioner();

collect()

​    close()

​    collect.flush()

sortAndSpill()

  sort()  QuickSort

mergeParts();  

collector.close();
~~~

### Shuffle机制

概念：主要是**Map阶段**之后，**Reduce阶段**之前对数据的**分区**、**排序**、**合并**、**分组**过程

![1604546405410](https://img-blog.csdnimg.cn/2020110514113862.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyODI5ODM1,size_16,color_FFFFFF,t_70#pic_center)

#### 分区（Partition）

1. 概述：为了将不同类型的内容输出到不同文件中，进行分类存储。

2. 默认分区：
   1. **HashPartitioner**根据key的hashCode对**ReduceTasks个数**取模得到的进行分区，用户不能进行设置。
   2. 底层原理 ： **(key.hashCode() & Integer.MAX_VALUE) % numReduceTasks**  
3. 自定义Partitioer步骤：
   1. 自定义类继承Partitioner重写getPartition方法（主要是控制分区代码逻辑）
   2. 在job驱动中，设置自定义Partitioner，
      * job.setPartitionerClass(CustomPartitioner.class);
   3. ​     自定义Partition后，要根据自定义Partitioner的逻辑设置相应数量的ReduceTask  
      * job.setNumReduceTasks(5);
4. 小知识点：
   1. ReduceTask数量> getPartition的结果数，则会多产生几个空的输出文件part-r-000xx；  
   2. 1<ReduceTask数量<getPartition的结果数，则有一部分分区数据无处安放，会抛出Exception；  
   3. ReduceTask的数量=1，则不管MapTask端输出多少个分区文件，最终结果都交给这一个ReduceTask，最终也就只会产生一个结果文件 part-r-00000；  
   4. 分区号必须从零开始，逐一累加。  



#### 排序（WritableComparable）

1. 概述：MapTask与ReduceTask均默认会对数据按照key进行排序（**字典排序，实现的方法是快速排序**）。任何数据均会被排序，而不管逻辑上是否需要。
   1. **MapTask**会将数据放在**环形缓冲区**中，当环形缓冲区使用率达到一定阈值时，再对缓冲区中的数据进行一次**快速排序**并**溢写**到磁盘上，当数据处理完毕后，会对磁盘上所有文件进行**归并排序**。
   2. **ReduceTask**从每个MapTask上**远程拷贝**数据文件到内存中，如果文件大小超过一定阈值，就**溢写**到磁盘上。如果磁盘文件数目到达一定能阈值，就进行一次归并排序**汇总**成一个文件，当数据拷贝完毕后，统一对**内存磁盘**上的所有数据进行一次**归并排序**
2. 排序分类
   1. 部分排序
      * MapReduce根据输入记录的键对数据排序，保证输出的每一个文件内部有序
   2. 全排序
      * 最终输出结果只有一个文件，且文件内部有序。实现方式是只设置一个ReduceTask。但是失去了MapReduce所提供的并行架构
   3. 辅助排序
      * Reduce阶段对key分组。在接受key为bean对象是，想让一个或几个字段相同的key进入到同一个reduce中。
   4. 二次排序
      * 在自定义排序过程中，如果自定义bean实现了compareTo逻辑即为二次排序
3. 自定义排序
   * bean对象做为key传输，需要实现WritableComparable接口重写compareTo方法，就可以实现排序。

#### 合并（Combiner）

1. 概述
   1. MR程序中Mapper和Reduce之外的一种组件
   2. 父类是的Reduce
   3. Combiner在每一个MapTask所在的节点运行，Reduce是接收全局所有Mapper的输出架构
   4. 对每一个MapTask输出进行局部汇总减少网络传输量
   5. 使用的前提不能影响最终业务逻辑，要跟Reduce的输入KV对应
2. 自定义Combiner实现步骤
   1. 自定义一个Combiner继承Reducer，重写Reduce方法  
   2. 在Job驱动类中设置  
      *   job.setCombinerClass(WordcountCombiner.class);  

#### 分组（GroupingComparator）

1. 概述
   * 对Reduce阶段的数据根据某一个或几个字段进行分组。
2. 自定义分组排序
   1. 自定义类继承WritableComparator  
   2. 重写compare()方法  
   3. 创建一个构造将比较对象的类传给父类  

### MapTask工作机制

（1）Read阶段：MapTask通过用户编写的RecordReader，从输入InputSplit中解析出一个个key/value。

​    （2）Map阶段：该节点主要是将解析出的key/value交给用户编写map()函数处理，并产生一系列新的key/value。

​    （3）Collect收集阶段：在用户编写map()函数中，当数据处理完成后，一般会调用OutputCollector.collect()输出结果。在该函数内部，它会将生成的key/value分区（调用Partitioner），并写入一个环形内存缓冲区中。

​    （4）Spill阶段：即“溢写”，当环形缓冲区满后，MapReduce会将数据写到本地磁盘上，生成一个临时文件。需要注意的是，将数据写入本地磁盘之前，先要对数据进行一次本地排序，并在必要时对数据进行合并、压缩等操作。

​    溢写阶段详情：

​    步骤1：利用快速排序算法对缓存区内的数据进行排序，排序方式是，先按照分区编号Partition进行排序，然后按照key进行排序。这样，经过排序后，数据以分区为单位聚集在一起，且同一分区内所有数据按照key有序。

​    步骤2：按照分区编号由小到大依次将每个分区中的数据写入任务工作目录下的临时文件output/spillN.out（N表示当前溢写次数）中。如果用户设置了Combiner，则写入文件之前，对每个分区中的数据进行一次聚集操作。

​    步骤3：将分区数据的元信息写到内存索引数据结构SpillRecord中，其中每个分区的元信息包括在临时文件中的偏移量、压缩前数据大小和压缩后数据大小。如果当前内存索引大小超过1MB，则将内存索引写到文件output/spillN.out.index中。

​    （5）Combine阶段：当所有数据处理完成后，MapTask对所有临时文件进行一次合并，以确保最终只会生成一个数据文件。

​    当所有数据处理完后，MapTask会将所有临时文件合并成一个大文件，并保存到文件output/file.out中，同时生成相应的索引文件output/file.out.index。

​    在进行文件合并过程中，MapTask以分区为单位进行合并。对于某个分区，它将采用多轮递归合并的方式。每轮合并io.sort.factor（默认10）个文件，并将产生的文件重新加入待合并列表中，对文件排序后，重复以上过程，直到最终得到一个大文件。

​    让每个MapTask最终只生成一个数据文件，可避免同时打开大量文件和同时读取大量小文件产生的随机读取带来的开销。

### OutputFormat数据输出  

![1608555459693](img\hadoop\1608555459693.png)

### 计数器

![1608555602493](img\hadoop\1608555602493.png)

### Map Join

Map Join适用于一张表十分小、一张表很大的场景。

采用DistributedCache

​    （1）在Mapper的setup阶段，将文件读取到缓存集合中。

​    （2）在驱动函数中加载缓存。

// 缓存普通文件到Task运行节点。

job.addCacheFile(new URI("file://e:/cache/pd.txt"));

![1608555686469](img\hadoop\1608555686469.png)

~~~java
// 6 加载缓存数据
		job.addCacheFile(new URI("file:///e:/input/inputcache/pd.txt"));
		
		// 7 Map端Join的逻辑不需要Reduce阶段，设置reduceTask数量为0
		job.setNumReduceTasks(0);

~~~

~~~java
	@Override
	protected void setup(Mapper<LongWritable, Text, Text, NullWritable>.Context context) throws IOException, InterruptedException {

		// 1 获取缓存的文件
		URI[] cacheFiles = context.getCacheFiles();
		String path = cacheFiles[0].getPath().toString();
		
		BufferedReader reader = new BufferedReader(new InputStreamReader(new FileInputStream(path), "UTF-8"));
		
		String line;
		while(StringUtils.isNotEmpty(line = reader.readLine())){

			// 2 切割
			String[] fields = line.split("\t");
			
			// 3 缓存数据到集合
			pdMap.put(fields[0], fields[1]);
		}
		
		// 4 关流
		reader.close();
	}

~~~

