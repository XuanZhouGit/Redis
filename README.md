# Redis
Redis
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
