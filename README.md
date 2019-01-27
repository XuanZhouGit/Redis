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
AOF实际上是一份执行日志,所有redis修改相关的命令追加到AOF文件中,通过回放这些命令就能恢复数据库,更新AOF文件的流程如图:

![Alt text](https://github.com/XuanZhouGit/Redis/blob/master/redis_aof.PNG)


Redis现在支持3种刷新策略:

1. AOF_FSYNC_NO :Write由主线程完成,不做fsync,只在redis被关闭或是AOF被关闭的时候进行fsync,写性能高但可靠性低,可能丢失上次fsync之后的数据

2. AOF_FSYNC_ALWAYS: Write和fsync都由主线程程完成, 每次都进行阻塞write和fsync,写性能低但可靠性高,最多丢失一条数据

3. AOF_FSYNC_EVERYSEC: Write在主线程完成,fsync在子线程非阻塞进行,2秒钟最多进行一次,最多丢失2秒的数据,写性能高且可靠性高

```
void flushAppendOnlyFile(int force) {
    ssize_t nwritten;
    int sync_in_progress = 0;
    mstime_t latency;

    if (sdslen(server.aof_buf) == 0) return;

    if (server.aof_fsync == AOF_FSYNC_EVERYSEC)
        sync_in_progress = bioPendingJobsOfType(BIO_AOF_FSYNC) != 0;

    if (server.aof_fsync == AOF_FSYNC_EVERYSEC && !force) {
        /* With this append fsync policy we do background fsyncing.
         * If the fsync is still in progress we can try to delay
         * the write for a couple of seconds. */
        if (sync_in_progress) {
            if (server.aof_flush_postponed_start == 0) {
                /* No previous write postponing, remember that we are
                 * postponing the flush and return. */
                server.aof_flush_postponed_start = server.unixtime;
                return;
            } else if (server.unixtime - server.aof_flush_postponed_start < 2) {
                /* We were already waiting for fsync to finish, but for less
                 * than two seconds this is still ok. Postpone again. */
                return;
            }
            /* Otherwise fall trough, and go write since we can't wait
             * over two seconds. */
            server.aof_delayed_fsync++;
            serverLog(LL_NOTICE,"Asynchronous AOF fsync is taking too long (disk is busy?). Writing the AOF buffer without waiting for fsync to complete, this may slow down Redis.");
        }
    }
    /* We want to perform a single write. This should be guaranteed atomic
     * at least if the filesystem we are writing is a real physical one.
     * While this will save us against the server being killed I don't think
     * there is much to do about the whole server stopping for power problems
     * or alike */

    latencyStartMonitor(latency);
    nwritten = aofWrite(server.aof_fd,server.aof_buf,sdslen(server.aof_buf));
    latencyEndMonitor(latency);
    /* We want to capture different events for delayed writes:
     * when the delay happens with a pending fsync, or with a saving child
     * active, and when the above two conditions are missing.
     * We also use an additional event name to save all samples which is
     * useful for graphing / monitoring purposes. */
    if (sync_in_progress) {
        latencyAddSampleIfNeeded("aof-write-pending-fsync",latency);
    } else if (server.aof_child_pid != -1 || server.rdb_child_pid != -1) {
        latencyAddSampleIfNeeded("aof-write-active-child",latency);
    } else {
        latencyAddSampleIfNeeded("aof-write-alone",latency);
    }
    latencyAddSampleIfNeeded("aof-write",latency);

    /* We performed the write so reset the postponed flush sentinel to zero. */
    server.aof_flush_postponed_start = 0;

    if (nwritten != (ssize_t)sdslen(server.aof_buf)) {
        static time_t last_write_error_log = 0;
        int can_log = 0;

        /* Limit logging rate to 1 line per AOF_WRITE_LOG_ERROR_RATE seconds. */
        if ((server.unixtime - last_write_error_log) > AOF_WRITE_LOG_ERROR_RATE) {
            can_log = 1;
            last_write_error_log = server.unixtime;
        }

        /* Log the AOF write error and record the error code. */
        if (nwritten == -1) {
            if (can_log) {
                serverLog(LL_WARNING,"Error writing to the AOF file: %s",
                    strerror(errno));
                server.aof_last_write_errno = errno;
            }
        } else {
            if (can_log) {
                serverLog(LL_WARNING,"Short write while writing to "
                                       "the AOF file: (nwritten=%lld, "
                                       "expected=%lld)",
                                       (long long)nwritten,
                                       (long long)sdslen(server.aof_buf));
            }

            if (ftruncate(server.aof_fd, server.aof_current_size) == -1) {
                if (can_log) {
                    serverLog(LL_WARNING, "Could not remove short write "
                             "from the append-only file.  Redis may refuse "
                             "to load the AOF the next time it starts.  "
                             "ftruncate: %s", strerror(errno));
                }
            } else {
                /* If the ftruncate() succeeded we can set nwritten to
                 * -1 since there is no longer partial data into the AOF. */
                nwritten = -1;
            }
            server.aof_last_write_errno = ENOSPC;
        }

        /* Handle the AOF write error. */
        if (server.aof_fsync == AOF_FSYNC_ALWAYS) {
            /* We can't recover when the fsync policy is ALWAYS since the
             * reply for the client is already in the output buffers, and we
             * have the contract with the user that on acknowledged write data
             * is synced on disk. */
            serverLog(LL_WARNING,"Can't recover from AOF write error when the AOF fsync policy is 'always'. Exiting...");
            exit(1);
        } else {
            /* Recover from failed write leaving data into the buffer. However
             * set an error to stop accepting writes as long as the error
             * condition is not cleared. */
            server.aof_last_write_status = C_ERR;

            /* Trim the sds buffer if there was a partial write, and there
             * was no way to undo it with ftruncate(2). */
            if (nwritten > 0) {
                server.aof_current_size += nwritten;
                sdsrange(server.aof_buf,nwritten,-1);
            }
            return; /* We'll try again on the next call... */
        }
    } else {
        /* Successful write(2). If AOF was in error state, restore the
         * OK state and log the event. */
        if (server.aof_last_write_status == C_ERR) {
            serverLog(LL_WARNING,
                "AOF write error looks solved, Redis can write again.");
            server.aof_last_write_status = C_OK;
        }
    }
    server.aof_current_size += nwritten;

    /* Re-use AOF buffer when it is small enough. The maximum comes from the
     * arena size of 4k minus some overhead (but is otherwise arbitrary). */
    if ((sdslen(server.aof_buf)+sdsavail(server.aof_buf)) < 4000) {
        sdsclear(server.aof_buf);
    } else {
        sdsfree(server.aof_buf);
        server.aof_buf = sdsempty();
    }

    /* Don't fsync if no-appendfsync-on-rewrite is set to yes and there are
     * children doing I/O in the background. */
    if (server.aof_no_fsync_on_rewrite &&
        (server.aof_child_pid != -1 || server.rdb_child_pid != -1))
            return;

    /* Perform the fsync if needed. */
    if (server.aof_fsync == AOF_FSYNC_ALWAYS) {
        /* aof_fsync is defined as fdatasync() for Linux in order to avoid
         * flushing metadata. */
        latencyStartMonitor(latency);
        aof_fsync(server.aof_fd); /* Let's try to get this data on the disk */
        latencyEndMonitor(latency);
        latencyAddSampleIfNeeded("aof-fsync-always",latency);
        server.aof_last_fsync = server.unixtime;
    } else if ((server.aof_fsync == AOF_FSYNC_EVERYSEC &&
                server.unixtime > server.aof_last_fsync)) {
        if (!sync_in_progress) aof_background_fsync(server.aof_fd);
        server.aof_last_fsync = server.unixtime;
    }
}
```

### 2.2 RDB




