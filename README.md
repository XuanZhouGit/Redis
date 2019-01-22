# Redis
Redis
## 1. ziplist
### 1.1 整体结构
ziplist是redis的基本数据结构,实际上是一个双向链表, redis的hash, list, set都基于ziplist实现,它的主要优点是省内存
 | zlbytes | zltail | zllen | entry | entry | ... | entry | zlend
------ | :----:  | :----: | :---: | :---: | :---: | :-: | :---: | :---:
size | uint32_t | uint32_t | uint16_t | | | | |uint8_t

其中:
zlbytes: 4字节,表示ziplist的大小(包括zlbytes的4个字节)
zltail: 4字节,表示最后一个节点在ziplist的偏移,能快速查找到尾部节点
zllen: 2字节,表示ziplist中的节点数目,也就是最多能表示2^16 - 1个节点,如果ziplist节点数超过2^16-1, zllen的值会被固定为2^16-1,只能通过遍历得到节点数目
entry: 存放数据的节点,长度不定
zlend: 1字节,固定为255

### 1.2 entry结构
