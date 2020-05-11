# innodb
## 1、缓冲池
- 为了提高检索速度，会按页将查询到的数据放入缓冲池
- 缓冲池不仅缓存索引页和数据页，会也缓存undolog页，插入缓冲，锁信息等(页默认大小为16k)
- 缓冲池可以配置多个，查询的页信息按取模的方式找到对应的缓冲池
- 查询出的数据会被放入LRU链表

### 1.1 innodb中的LRU
- 加入了midpoint位置，新查出的数据从这个位置向后放(默认为列表长度的5/8),可以通过 show variables like 'innodb_old_blocks_pct' 查看
- 可以通过 innodb_old_blocks_times 设置读取到mid位置后等待多久才会被加入LRU的热端(midpoint前边的为热端)
- 通过 show engine innodb status 命名查看LRU以及Free列表的是用情况。也可以查看缓冲池的命中率，一般要高于95%

### 1.2 checkpoint
#### 1.2.1 背景
- 为了协调cpu速度和磁盘速度差距太大的问题，有变更时只会修改缓冲池(修改后变为脏页)
- 为了保证持久性采用了[write ahead log]策略，即当事务提交时先写重做日志，在修改页

#### 1.2.2 checkpoint机制
- 虽然有不同的实现机制，但其作用无非就是将缓冲池的脏页刷到磁盘，只是从哪获取脏页、刷多少、同步还是异步，频率等这些不同而已
- sharp checkpoint:数据库正产关闭是将所有的脏页刷新回磁盘
- fuzzy checkpoint:只刷新一部分脏页
 - master thread checkpoint:每秒或每10秒从脏页列表刷新一定比例到磁盘
 - flush_lru_list checkpoint:
 - Async/Sync flush checkpoint
 - dirty page too much checkpoint

### 1.3 插入缓冲
#### 1.3.1 背景
 - 数据都是按照主键顺序存放在数据页的,所以插入聚簇索引时不需要随机读取另一个页中的记录,但是对于非聚簇索引叶子节点的插入不在是顺序的,这时需要离散的读取非聚簇索引也
 由于随机读取的存在而导致插入性能下降

#### 1.3.2 insert buffer
- 对于非聚簇索引的插入或修改不是每一次直接插入到索引页中,而是先判断插入的非聚簇索引页是否在缓冲池中,若不在则先放入InsertBuffer中,然后在以一定的频率合并到索引叶子节点
- insert buffer是一个物理页,其内部也是由b+tree实现，非叶子节点存放(space:表的id,marker:兼容老版本,offset:页所在的偏移量)search key，叶子节点会多存储metadata和插入记录的各个字段
- merge insert buffer发生在一下几种情况
 - 辅助索引页被读取到缓冲池
 - insert buffer bitmap页追踪到该辅助索引页以无可用空间
 - Master Thread


### 1.4 dubbo write
#### 1.4.1 当对页进行数据恢复时如果页本身已经发生损坏,这时是不能通过redoLog对其进行恢复的
#### 1.4.2
- 缓冲池中有两个dubbo write buffer每个1M,物理文件也有两个dubbo write页,每个1M
- 当写入数据时先写dubbo write buffer,在写dubbo write页,在同步到数据页
- 这样会防止页被修改一部分时由于宕机引起的页损坏,因为这个页要插入的完整信息都在dubbo write页做了备份

### 1.5 hash index

### 1.6 异步io

### 1.7 redolog buffer
innodb存储引擎会先将重做日志信息放入缓冲池中，刷新到磁盘的方式如下

- master thread每秒会选择部分重做日志缓冲刷到文件
- 每个事物提交时会将重做日志缓冲刷到文件
- 重做日志缓冲池小于1/2时，会将重做日志缓冲刷到文件

## 2 日志文件
### 2.1 慢查询日志
- 查看定义为慢查询的时间：show variables like 'long_query_time'
- 查看是否启用慢日志：show variables like 'slow_query_log'
- 全局开启慢日志查询：set global slow_query_log=ON
