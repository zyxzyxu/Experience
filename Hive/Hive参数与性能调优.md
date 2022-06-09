# Hive性能调优的方式
- SQL语句优化
	1. union all
		```sql
		insert into table test01 partition(tp)
		select age，max(birthday) birthday，'max' tp
		from user
		group by age
		
		union all
		
		insert into table test01 partition(tp)
		select age，min(birthday) birthday，'min' tp
		from user
		group by age；
		```
	- union all 前后的两个语句都是对同一张表按照age进行分组，然后分别取最大、最小值，对同一张表相同的字段进行两次分组，可以进行优化，使用语法：from ... insert into ... 使用一张表，可以进行多次插入操作：
		```sql
		--开启动态分区 
		set hive.exec.dynamic.partition=true； 
		set hive.exec.dynamic.partition.mode=nonstrict； 
		
		from user 
		
		insert into table test01 partition(tp) 
		select age，max(birthday) birthday，'max' tp 
		group by age
		
		insert into table stu partition(tp) 
		select age，min(birthday) birthday，'min' tp 
		group by age；
		```
	2. distinct
		```sql
		select count(1) from (select age from test01 group by age) a；
		
		select count(distinct age) from test01；
		```
	- 在数据量特别大的情况下使用第一种方式可以有效避免Reduce端的数据倾斜
	- **就当前的业务和环境下使用distinct一定会比上面那种子查询的方式效率高**，原因如下：
		1. 在当前案例下，去重字段为年龄，年龄的枚举值非常有限，age的最大枚举值才是100，如果转化成MapReduce来解释的话，在Map阶段，每个Map会对age去重。由于age枚举值有限，因而每个Map得到的age也有限，最终得到reduce的数据量也就是map数量* age枚举值的个数
		2. 最新的Hive 3.0中新增了 count(distinct)优化，通过配置hive.optimize.countdistinct，即使真的出现数据倾斜也可以自动优化，自动改变SQL执行的逻辑
		3. 第二种方式比第一种简洁明了，如果没有特殊问题，简洁为优

	- **在一些情况下不要过度优化，调优讲究适时调优，过早调优可能做的是无用功甚至负效应**
- 数据格式优化
	- Hive提供多种数据存储组织格式，不同格式对程序的运行效率也会有极大的影响
	- Hive提供的格式有TEXT、SequenceFile、RCFile、ORC和Parquet等
	- SequenceFile是一个二进制key/value对结构的平面文件，在早期的Hadoop平台上被广泛用于MapReduce输出/输出格式，以及作为数据存储格式
	- *Parquet*是一种列式数据存储格式，可以兼容多种计算引擎，如MapRedcue和Spark等，对多层嵌套的数据结构提供了良好的性能支持，是目前Hive生产环境中数据存储的主流选择之一
	- ORC优化是对RCFile的一种优化，它提供了一种高效的方式来存储Hive数据，同时也能够提高Hive的读取、写入和处理数据的性能，能够兼容多种计算引擎。事实上，在实际的生产环境中，ORC已经成为了Hive在数据存储上的主流选择之一

| 数据格式 | CPU时间 | 用户等待耗时 |
| ：---： | ：---： | ：---： |
| TextFile | 33分 | 171秒 |
| SequenceFile | 38分 | 162秒 |
| Parquet | 2分22秒 | 50秒 |
| ORC | 1分52秒 | 56秒 |

>**CPU时间**：表示运行程序所占用服务器CPU资源的时间
**用户等待耗时**：记录的是用户从提交作业到返回结果期间用户等待的所有时间

- 小文件过多优化
	>小文件如果过多，对 hive 来说，在进行查询时，每个小文件都会当成一个块，启动一个Map任务来完成，而一个Map任务启动和初始化的时间远远大于逻辑处理的时间，就会造成很大的资源浪费。而且，同时可执行的Map数量是受限的
	- 小文件产生原因
		1. 直接向表中插入数据
		```sql
		insert into table test02 values (1，'a')，(2，'b')；
		```
		多次插入少量数据就会出现多个小文件，生产环境基本没有使用
		2. 通过load方式加载数据
		```sql
		load data local inpath '/export/list.csv' overwrite into table test02 -- 导入文件
		
		load data local inpath '/export/list' overwrite into table test02 -- 导入文件夹
		```
		使用load data 方式可以导入文件或文件夹，当导入一个文件时，hive表就有一个文件，当导入文件夹时，hive表的文件数量为文件夹下所有文件的数量
		3. 通过查询方式加载数据
		```sql
		insert overwrite table test02 select id，name from test03；
		```
		这种方式是生产环境中常用的，也是最容易产生小文件的方式

		insert 导入数据时会启动 MR 任务，MR 中 reduce 有多少个就输出多少个文件，所以，文件数量 = ReduceTask数量* 分区数

		也有很多简单任务没有reduce，只有map阶段，则文件数量 = MapTask数量* 分区数
		每执行一次 insert 时hive中至少产生一个文件，因为 insert 导入时至少会有一个MapTask。

		像有的业务需要每10分钟就要把数据同步到 hive 中，这样产生的文件就会很多。

	- 小文件过多产生的影响
		1. 首先对底层存储HDFS来说，HDFS本身就不适合存储大量小文件，小文件过多会导致namenode元数据特别大，占用太多内存，严重影响HDFS的性能
		2. 对 hive 来说，在进行查询时，每个小文件都会当成一个块，启动一个Map任务来完成，而一个Map任务启动和初始化的时间远远大于逻辑处理的时间，就会造成很大的资源浪费。而且，同时可执行的Map数量是受限的。
	- 解决小文件过多
		1. 使用 hive 自带的 concatenate 命令，自动合并小文件
		使用方法：
		```sql
		#对于非分区表
		alter table A concatenate;
		
		#对于分区表
		alter table B partition(day=20220101) concatenate;
		```
		举例：
		```sql
		#向 A 表中插入数据
		hive (default)> insert into table A values (1,'aa',67),(2,'bb',87);
		hive (default)> insert into table A values (3,'cc',67),(4,'dd',87);
		hive (default)> insert into table A values (5,'ee',67),(6,'ff',87);
		
		#执行以上三条语句，则A表下就会有三个小文件,在hive命令行执行如下语句
		#查看A表下文件数量
		hive (default)> dfs -ls /user/hive/warehouse/A;
		Found 3 items
		-rwxr-xr-x   3 root supergroup        378 2022-01-01 14：46 /user/hive/warehouse/A/000000_0
		-rwxr-xr-x   3 root supergroup        378 2022-01-01 14：47 /user/hive/warehouse/A/000000_0_copy_1
		-rwxr-xr-x   3 root supergroup        378 2022-01-01 14：48 /user/hive/warehouse/A/000000_0_copy_2
		
		#可以看到有三个小文件，然后使用 concatenate 进行合并
		hive (default)> alter table A concatenate;
		
		#再次查看A表下文件数量
		hive (default)> dfs -ls /user/hive/warehouse/A;
		Found 1 items
		-rwxr-xr-x   3 root supergroup        778 2020-12-24 14：59 /user/hive/warehouse/A/000000_0
		
		#已合并成一个文件
		```
		> 注意： 
		> 1、concatenate 命令只支持 RCFILE 和 ORC 文件类型。 
		> 2、使用concatenate命令合并小文件时不能指定合并后的文件数量，但可以多次执行该命令。 
		> 3、当多次使用concatenate后文件数量不在变化，这个跟参数 mapreduce.input.fileinputformat.split.minsize=256mb 的设置有关，可设定每个文件的最小size。
		2. 调整参数减少Map数量
		- 设置map输入合并小文件的相关参数：
		```sql
		#执行Map前进行小文件合并
		#CombineHiveInputFormat底层是 Hadoop的 CombineFileInputFormat 方法
		#此方法是在mapper中将多个文件合成一个split作为输入
		set hive.input.format=org.apache.hadoop.hive.ql.io.CombineHiveInputFormat; -- 默认
		
		#每个Map最大输入大小(这个值决定了合并后文件的数量)
		set mapred.max.split.size=256000000;   -- 256M
		
		#一个节点上split的至少的大小(这个值决定了多个DataNode上的文件是否需要合并)
		set mapred.min.split.size.per.node=100000000;  -- 100M
		
		#一个交换机下split的至少的大小(这个值决定了多个交换机上的文件是否需要合并)
		set mapred.min.split.size.per.rack=100000000;  -- 100M
		```
		- 设置map输出和reduce输出进行合并的相关参数：
		```sql
		#设置map端输出进行合并，默认为true
		set hive.merge.mapfiles = true;
		
		#设置reduce端输出进行合并，默认为false
		set hive.merge.mapredfiles = true;
		
		#设置合并文件的大小
		set hive.merge.size.per.task = 256*1000*1000;   -- 256M
		
		#当输出文件的平均大小小于该值时，启动一个独立的MapReduce任务进行文件merge
		set hive.merge.smallfiles.avgsize=16000000;   -- 16M 
		```
		- 启用压缩
		```sql
		# hive的查询结果输出是否进行压缩
		set hive.exec.compress.output=true;
		
		# MapReduce Job的结果输出是否使用压缩
		set mapreduce.output.fileoutputformat.compress=true;
		```
		3. 减少Reduce的数量
		```sql
		#reduce 的个数决定了输出的文件的个数，所以可以调整reduce的个数控制hive表的文件数量，
		#hive中的分区函数 distribute by 正好是控制MR中partition分区的，
		#然后通过设置reduce的数量，结合分区函数让数据均衡的进入每个reduce即可。
		
		#设置reduce的数量有两种方式，第一种是直接设置reduce个数
		set mapreduce.job.reduces=10;
		
		#第二种是设置每个reduce的大小，Hive会根据数据总大小猜测确定一个reduce个数
		set hive.exec.reducers.bytes.per.reducer=5120000000; -- 默认是1G，设置为5G
		
		#执行以下语句，将数据均衡的分配到reduce中
		set mapreduce.job.reduces=10;
		insert overwrite table A partition(dt)
		select * from B
		distribute by rand();
		
		#解释：如设置reduce数量为10，则使用 rand()， 随机生成一个数 x % 10 ，
		#这样数据就会随机进入 reduce 中，防止出现有的文件过大或过小
		```
		4. 使用hadoop的archive将小文件归档
		Hadoop Archive简称HAR，是一个高效地将小文件放入HDFS块中的文件存档工具，它能够将多个小文件打包成一个HAR文件，这样在减少namenode内存使用的同时，仍然允许对文件进行透明的访问
		```sql
		#用来控制归档是否可用
		set hive.archive.enabled=true;
		#通知Hive在创建归档时是否可以设置父目录
		set hive.archive.har.parentdir.settable=true;
		#控制需要归档文件的大小
		set har.partfile.size=1099511627776;
		
		#使用以下命令进行归档
		ALTER TABLE A ARCHIVE PARTITION(dt='2022-01-01', hr='12');
		
		#对已归档的分区恢复为原文件
		ALTER TABLE A UNARCHIVE PARTITION(dt='2022-01-01', hr='12');
		```
		>注意：归档的分区可以查看不能 insert overwrite，必须先 unarchive

		>如果是新集群，没有历史遗留问题的话，建议hive使用 orc 文件格式，以及启用 lzo 压缩。这样小文件过多可以使用hive自带命令 concatenate 快速合并。