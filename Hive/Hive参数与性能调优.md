# Hive性能调优的方式
- SQL语句优化
	1. union all
		```sql
		insert into table test01 partition(tp)
		select age,max(birthday) birthday,'max' tp
		from user
		group by age
		
		union all
		
		insert into table test01 partition(tp)
		select age,min(birthday) birthday,'min' tp
		from user
		group by age;
		```
	- union all 前后的两个语句都是对同一张表按照age进行分组，然后分别取最大、最小值，对同一张表相同的字段进行两次分组，可以进行优化，使用语法：from ... insert into ... 使用一张表，可以进行多次插入操作：
		```sql
		--开启动态分区 
		set hive.exec.dynamic.partition=true; 
		set hive.exec.dynamic.partition.mode=nonstrict; 

		from user 

		insert into table test01 partition(tp) 
		select age,max(birthday) birthday,'max' tp 
		group by age

		insert into table stu partition(tp) 
		select age,min(birthday) birthday,'min' tp 
		group by age;
		```
	2. distinct
		```sql
		select count(1) from (select age from test01 group by age) a;

		select count(distinct age) from test01;
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
	| --- | --- | --- |
	| TextFile | 33分 | 171秒 |
	| SequenceFile | 38分 | 162秒 |
	| Parquet | 2分22秒 | 50秒 |
	| ORC | 1分52秒 | 56秒 |

	>**CPU时间**：表示运行程序所占用服务器CPU资源的时间
	>**用户等待耗时**：记录的是用户从提交作业到返回结果期间用户等待的所有时间
- 小文件过多优化
	>小文件如果过多，对 hive 来说，在进行查询时，每个小文件都会当成一个块，启动一个Map任务来完成，而一个Map任务启动和初始化的时间远远大于逻辑处理的时间，就会造成很大的资源浪费。而且，同时可执行的Map数量是受限的