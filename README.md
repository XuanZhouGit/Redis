# Redis
Redis是一个开源的高性能key-value存储系统,有这些特点:

1. 高性能:https://redis.io/topics/benchmarks

2. 支持丰富的数据类型(string, hash, list, set, sorted set等)

3. 所有操作都是原子性的,支持事务

4. 可以通过AOF/RDB方式将内存数据保存到磁盘上保证持久存储

5. 支持主从同步

6. 支持publish/subscribe, notify等特性

## 1. ziplist
### 1.1 整体结构
ziplist是redis的基本数据结构,实际上是一个双向链表, redis的hash, list, set都基于ziplist实现,它的主要优点是省内存

struct | zlbytes | zltail | zllen | entry | entry | ... | entry | zlend
------ | :-----: | :----: | :---: | :---: | :---: | :-: | :---: | :---:
size   | uint32_t|uint32_t|uint16_t|      |       |     |       |uint8_t

其中:
zlbytes: 4字节,表示ziplist的大小(包括zlbytes的4个字节)

zltail: 4字节,表示最后一个节点在ziplist的偏移,能快速查找到尾部节点

zllen: 2字节,表示ziplist中的节点数目,也就是最多能表示2^16 - 1个节点,如果ziplist节点数超过2^16-1, zllen的值会被固定为2^16-1,只能通过遍历得到节点数目

entry: 存放数据的节点,长度不定

zlend: 1字节,固定为255

### 1.2 entry结构

<prevlen> <encoding> <entry-data>
  
prevlen: 1字节或5字节,表示前一个节点所占字节数, 方便ziplist向前遍历找到前项节点:

1. 如果前一个节点所占字节数小于254, prevlen就占1字节

2. 如果前一个节点所占字节数大于254, prevlen就需要5字节,第一个字节为254, 后面4字节用于表示前一节点大小

encoding字段很复杂:

1. 1字节, |00pppppp| : 前两bit为00, 后6bit为entry-data的长度len(< 64), entry-data为len字节的字符数组

   1字节, |11000000| : entry-data为int16_t的整数
   
   1字节, |11010000| : entry-data为int32_t的整数
   
   1字节, |11100000| : entry-data为int64_t的整数
   
   1字节, |11110000| : entry-data为24bit有符号整数
   
   1字节, |11111110| : entry-data为8bit有符号整数
   
   1字节, |1111xxxx| : entry-data为xxxx - 1, xxxx只有1-13是可用的,所以需要减一来表示0-12
   
   1字节, |11111111| : zlend
   
2. 2字节, |01pppppp|qqqqqqqq| : 前2bit为01, 后14bit为entry-data的长度len(< 2^14), entry-data为len字节的字符数组

3. 5字节, |10000000|qqqqqqqq|rrrrrrrr|ssssssss|tttttttt| : 第一个字节为10000000, 后4字节为entry-data的长度len(< 2^32), entry-data为len字节的字符数组

### 1.3 创建ziplist


struct | zlbytes | zltail | zllen | zlend
------ | :-----: | :----: | :---: | :---:
size   |  4Bytes | 4Bytes | 2Bytes|1Byte
value  | 1011    | 1010   | 0     |11111111

### 1.4 插入元素(复杂度O(N) ~ O(N^2))

struct | zlbytes | zltail | zllen | ... | prev | next | ... | zlend
------ | :-----: | :----: | :---: | :-: | :---:|:---: | :-: | :---:
size   |  4Bytes | 4Bytes | 2Bytes| ... | 9Bytes| ... | ... |1Byte
value  | 1011    | 1010   |1000000| ... | ... |  prev_entry_length: 1001<br>... | ... | 11111111

在prev,next之间插入"test"

struct | zlbytes | zltail | zllen | ... | prev | entry | next | ... | zlend
------ | :-----: | :----: | :---: | :-: | :---:|:-----:|:---: | :-: | :---:
size   |  4Bytes | 4Bytes | 2Bytes| ... | 9Bytes|6Bytes|  ... | ... |1Byte
value  | 1011    | 1010   |1000000| ... | ... | prev_entry_length: 1001<br> encoding: 00000100<br> entry-data: "test" | prev_entry_length: 1000<br>... | ...  | 11111111

插入元素(entry)时,需要将entry之后的节点移位,所以一般情况时,插入entry需要O(N)复杂度,由于插入元素时需要更新next的prev_entry_length的值,如果prev_entry_length所占大小由1字节变成5字节,那么next的长度发生变化,引起next->next的prev_entry_length, 最坏情况可能变成O(N^2)的连锁更新
### 1.5 删除元素(复杂度O(N) ~ O(N^2))
删除操作可以看成插入操作的逆操作,与插入类似,可能引起连锁更新

### 1.6 遍历

header | e1 | e2 | e3 | e4 | ... | zlend
:----: |----|----|----|----|-----|------

向后遍历: 比如指向e1的p开始,计算e1的长度(len1), (p+len1)即指向e2
向前遍历: 比如指向e3的p开始,读取e3中的prev_entry_length(len2), (p-len2)即指向e2

## 2 持久存储
Redis之所以性能好,读写速度快,是因为它的所有操作都基于内存,但内存的数据如果进程崩溃或系统重启就会丢失,所以数据持久化对于内存数据库很重要,它保证了数据库的可靠性, Redis提供了两种持久化方案,AOF及RDB(4.0开始支持AOF-RDB混合)
### 2.1 AOF(Append-only file)
AOF实际上是一份执行日志,所有redis修改相关的命令追加到AOF文件中,通过回放这些命令就能恢复数据库
Redis现在支持3种刷新策略:

1. AOF_FSYNC_NO :Write由主线程完成,不做fsync,只在redis close的时候进行fsync,写性能高但可靠性低,可能丢失上次fsync之后的数据

2. AOF_FSYNC_ALWAYS: Write和fsync都由主线程程完成, 每次都进行阻塞write和fsync,写性能低但可靠性高,最多丢失一条数据

3. AOF_FSYNC_EVERYSEC: Write在主线程完成,fsync在子线程非阻塞进行,2秒钟最多进行一次,最多丢失2秒的数据,写性能高且可靠性高

### 2.2 RDB




