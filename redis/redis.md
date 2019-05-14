## redis键值对象 redisObject
主要包括五个属性
1. type (String、list、hash、set、sortset)
2. encoding  编码
3. ptr  具体的值
4. refcount 引用计数
5. lru 最后一次访问时间
encoding是对type 的扩展，也是ptr指针指向对应底层数据结构的依据
类型	编码	对象
REDIS_STRING	REDIS_ENCODING_INT	使用整数值实现的字符串对象。
REDIS_STRING	REDIS_ENCODING_EMBSTR	使用 embstr 编码的简单动态字符串实现的字符串对象。
REDIS_STRING	REDIS_ENCODING_RAW	使用简单动态字符串实现的字符串对象。
REDIS_LIST	REDIS_ENCODING_ZIPLIST	使用压缩列表实现的列表对象。
REDIS_LIST	REDIS_ENCODING_LINKEDLIST	使用双端链表实现的列表对象。
REDIS_HASH	REDIS_ENCODING_ZIPLIST	使用压缩列表实现的哈希对象。
REDIS_HASH	REDIS_ENCODING_HT	使用字典实现的哈希对象。
REDIS_SET	REDIS_ENCODING_INTSET	使用整数集合实现的集合对象。
REDIS_SET	REDIS_ENCODING_HT	使用字典实现的集合对象。
REDIS_ZSET	REDIS_ENCODING_ZIPLIST	使用压缩列表实现的有序集合对象。
REDIS_ZSET	REDIS_ENCODING_SKIPLIST	使用跳跃表和字典实现的有序集合对象。
## redis数据库结构 redisDB
主要包含两个属性
1. dict 字典类型，保存当前数据库所有键值对
2. expires 字典类型，保存所有键以及键的过期时间
增、删、改、查操作都是基于dict字典的操作，但是可能还有些附加操作，比如维护引用计数、维护lru,维护过期时间等。
### 过期键处理(两种配合使用)
1. 惰性删除，当获取时判断是否已经过期，如果过期则删除
2. 定时删除，任务触发时从数据库随机取出一部分数据，对过期的进行清除。
