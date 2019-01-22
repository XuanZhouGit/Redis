# Redis
Redis
## 1. ziplist
layout | zlbytes | zltail | zllen | entry | entry | ... | entry | zlend
size | uint32_t | uint32_t | uint16_t | | | | |uint8_t
