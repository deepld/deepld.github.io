# Block Cache 
## Class
    ShardedLRUCache：由 N 个 LRUCache 组成，分散锁的压力

    LRUCache：cache 加了一个大锁；传入了delter函数负责清理 item 的value
        保存了2个双向list，lru cache（无人使用、最先被替换的）、in use list（正在使用中）
        一个 item，通过 hash table 进行索引；并保存在 lru 或者 in use 的list之中

    LRUHandle：每个 cache item 的handler，记录了额外的信息；
        key 保存在 LRUHandle 结尾，key 是外部分配的，这里只是引用一下，因此需要复制；而value是直接存在cache中的

    Note：数据可以缓存子系统 page cache、env提供的 file类实现，这里存放的是 compress data；使用自身 LRU Cache 存放是uncompress data
    在 bulk load 的时候， 设置options.fill_cache = false; 防止污染缓存

# Log
    日志这么分段存储的好处？
    * 方便处理 corruptin；即使一个record 在两个log，也很容易拼接
    * 方便在 map reduce 中进行切分；方便处理大log，不需要全部buffer
    * 日志没有使用压缩

## 相关参数
    block_restart_interval：2^16, 64K
    max_file_size：2M
    ReadOptions：当前读取是否缓存到cache、是否在访问快照
    WriteOptions：是否sync
