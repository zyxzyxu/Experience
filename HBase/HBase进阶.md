# HBase进阶
- Master架构
	Master
	主要进程，具体实现类为HMaster，通常部署在namenode上
	通过zookeeper管理分布在各个服务器上的 regionServer
	Master服务端
	1. 负载均衡器
		通过读取meta表了解Regin的分配，通过连接zk了解RS的启动情况。5分钟调控一次分配平衡
	2. 元数据表管理器 (hdfs文件系统：元数据表->/hbase/data/hbase/meta)
		管理Master自己的预写日志，如果宕机，让backUpMaster读取日志数据
	3. MasterProcWAL预写日志管理器 (hdfs文件系统：MasterWAL数据->/hbase/MasterData/WALs)
		本质写数据到hdfs，32M文件或者1H滚动当操作执行到meta表之后删除WAL

- Region Server架构
	Region Server
	主要进程，具体实现类为HRegionServer，通常部署在dataNode上。
	Region Server服务端
	1. WAL
		hdfs文件系统的WAL预写日志
	2. Block cache 读缓存（数量1）
		当Client调用get请求时，会从hdfs文件系统存储的Table表中读取每个RS管理的Region信息，存一份到内存中方便下次读取
	3. Mem Store 写缓存（数量 = store个数）
		当Client调用put请求时，会先写入内存中的Mem Store中，攒的过程中不断按照Row_key进行排序，攒够一批写入hdfs管理的table表中
	除了主要的组件，还会启动多个线程监控一些必要的服务：
	Region拆分
	Region合并
	MemStore刷写
	WAL预写日志滚动

- 写流程
	HBase写流程
	1. Client先问zookeeper请求创建连接,获取meta表位于哪个Region Server。
	2. 访问对应的Region Server读取meta表，查询出目标数据位于哪个Region Server中的哪个Region中。并将该table的region信息以及meta表的位置信息缓存在客户端的MetaCache，方便下次访问。
	3. 与目标RegionServer进行通讯；
	4. 将数据顺序写入（追加）到WAL；
	5. 将数据写入对应的Mem Store，数据会在Mem Story进行排序；
	6. 向客户端发送ack；
	7. 等达到Mem Store的刷写时机后，将数据刷写到HFile。

- MemStore Flush
	MemStore刷写时机：
	1. 当某个memstroe的大小达到了 hbase.hregion.memstore.flush.size（默认值128M），其所在region的所有memstore都会刷写。
	当memstore的大小达到了hbase.hregion.memstore.flush.size（默认值128M）* hbase.hregion.memstore.block.multiplier（默认值4）时，会阻止继续往该memstore写数据。
	2. 当region server中memstore的总大小达到java_heapsize* hbase.regionserver.global.memstore.size（默认值0.4）* hbase.regionserver.global.memstore.size.lower.limit（默认值0.95），region会按照其所有memstore的大小顺序（由大到小）依次进行刷写。直到region server中所有memstore的总大小减小到上述值以下。当region server中memstore的总大小达到java_heapsize* hbase.regionserver.global.memstore.size（默认值0.4）时，会阻止继续往所有的memstore写数据。
	3. 到达自动刷写的时间，也会触发memstore flush。自动刷新的时间间隔由该属性进行配置hbase.regionserver.optionalcacheflushinterval（默认1小时）。
	4. 当WAL文件的数量超过hbase.regionserver.max.logs，region会按照时间顺序依次进行刷写，直到WAL文件数量减小到hbase.regionserver.max.log以下（该属性名已经废弃，现无需手动设置，最大值为32）。

- 读流程
	HBase读流程
	1. Client先访问zookeeper，获取hbase:meta表位于哪个Region Server。
	2. 访问对应的Region Server，获取hbase:meta表,查询出目标数据位于哪个Region Server中的哪个Region中。并将该table的region信息以及meta表的位置信息缓存在客户端的meta cache，方便下次访问。
	3. 与目标Region Server进行通讯；
	4. 分别在BlockCache和MemStore和Store File（HFile）中查询目标数据，并将查到的所有数据进行合并。此处所有数据是指同一条数据的不同版本（time stamp）或者不同的类型（Put/Delete）。
	5. 将查询到的新的数据块（Block，HFile数据存储单元，默认大小为64KB）缓存到Block Cache。
	6. 将合并后的最终结果返回给客户端。

- HFile 合并（默认是7天）
	Compaction分为两种，分别是Minor Compaction和Major Compaction
	1. Minor Compaction会将临近的若干个较小的HFile合并成一个较大的HFile，并清理掉部分过期和删除的数据。
	2. Major Compaction会将一个Store下的所有的HFile合并成一个大HFile，并且会清理掉所有过期和删除的数据。

- HBase优化
	预分区
	1. 手动设定预分区
	```
	create 'staff1','info', SPLITS => ['1000','2000','3000','4000']
	```
	2. 生成16进制序列预分区
	```
	create 'staff2','info',{NUMREGIONS => 15, SPLITALGO => 'HexStringSplit'}
	```
	3. 按照文件中设置的规则预分区
	```
	create 'staff3', 'info',SPLITS_FILE => 'splits.txt'
	```
	4. 使用JavaAPI创建预分区

- RowKey设计
	1. 生成随机数、hash、散列值
	2. 字符串反转
	3. 字符串拼接

- 基础优化
	1. Zookeeper会话超时时间
	```
	hbase-site.xml
	属性：zookeeper.session.timeout
	解释：默认值为90000毫秒（90s）。当某个RegionServer挂掉，90s之后Master才能察觉到。可适当减小此值，以加快Master响应，可调整至60000毫秒。
	```
	2. 设置RPC监听数量
	```
	hbase-site.xml
	属性：hbase.regionserver.handler.count
	解释：默认值为30，用于指定RPC监听的数量，可以根据客户端的请求数进行调整，读写请求较多时，增加此值。
	```
	3. 手动控制Major Compaction
	```
	hbase-site.xml
	属性：hbase.hregion.majorcompaction
	解释：默认值：604800000秒（7天）， Major Compaction的周期，若关闭自动Major Compaction，可将其设为0
	```
	4. 优化HStore文件大小
	```
	hbase-site.xml
	属性：hbase.hregion.max.filesize
	解释：默认值10737418240（10GB），如果需要运行HBase的MR任务，可以减小此值，因为一个region对应一个map任务，如果单个region过大，会导致map任务执行时间过长。该值的意思就是，如果HFile的大小达到这个数值，则这个region会被切分为两个Hfile。
	```
	5. 优化HBase客户端缓存
	```
	hbase-site.xml
	属性：hbase.client.write.buffer
	解释：默认值2097152bytes（2M）用于指定HBase客户端缓存，增大该值可以减少RPC调用次数，但是会消耗更多内存，反之则反之。一般我们需要设定一定的缓存大小，	以达到减少RPC次数的目的。
	```
	6. 指定scan.next扫描HBase所获取的行数
	```
	hbase-site.xml
	属性：hbase.client.scanner.caching
	解释：用于指定scan.next方法获取的默认行数，值越大，消耗内存越大。
	```
	7. BlockCache占用RegionServer堆内存的比例
	```
	hbase-site.xml
	属性：hfile.block.cache.size
	解释：默认0.4，读请求比较多的情况下，可适当调大
	```
	8. MemStore占用RegionServer堆内存的比例
	```
	hbase-site.xml
	属性：hbase.regionserver.global.memstore.size
	解释：默认0.4，写请求较多的情况下，可适当调大
	```