
***[阅读后的代码注释版本github地址，这里对关键流程都加了解析](https://github.com/deepld/leveldb/tree/annotation) ，默认分支：annotation*** 

# 阅读目标
>很早以前看过 hyptertable 的实现，是HBase的 C++版本，跟level这些基本都差不多；此次再回炉一遍，目标定位在设计思路、各种处理细节，不讨论 LSM tree 的原理这类东西
>这里记录的是一些简要的point，更详细的跟着代码流程跑一遍，看相关的注释即可；后续要需要再补充更多的内容

# 功能限制
    功能：不提供 column family、column 的机制，需要在 user_key 自己进行拼装；一个底层 key 只有一个value
    其他：对性能的考虑不多，适合hdd硬件环境下的使用（ssd环境下使用RocksDB更好）

# 主要结构说明
    1. DBImpl：所有的DB逻辑都放在这里
    2. Cache：
       1. TableCache：用于缓存所有 sst 文件的 meta 信息， file number -> Table
       2. 默认建立 8M 的 block_cache

    3. Memory
       1. Table：对应于一个sst文件在内存中的meta缓存，还包含了 index block + meta block

    4. Version：当前有效的文件集合、以及协议
       1. VersionEdit

    5. Iterator：提供接口，以及一个 cleanup 的 func list，*用于关闭时进行清理*
       1. DBIter：
       2. TwoLevelIterator：遍历多个block的 iterator；切换 block 时会调用Table::BlockReader，设置回调；如果这个block访问完成了，就可以调用回调删除相应的block了
       3. IteratorWrapper：缓存 key 、Valid 的内容，减少访问是对virtual的使用？更好的 locality，提升部分场景的性能
    
    6. Key：不同的key有不同的 
       1. user key：
       2. LookupKey
       3. memtable_key

    7. Comparator：不同的key对应不同的 Comparator，比较顺序： user key的顺序、seqno

    8. Filter：
       1. InternalFilterPolicy    

# 文件格式
<https://github.com/google/leveldb/blob/master/doc/table_format.md> 记录的都是相对偏移，因此 offset 用 32位即可

## Block
    BlockBuilder：用于生成 block；格式见：cc文件头部；
        使用连续内存记录整个block；使用前缀压缩，达到 N（block_restart_interval）个key时，重置 restart point；|range-1|range-2|。。。|restart point offset array|restart count

    Block：仅用于block的读取，提供了相应的 Iterator；数据被 restart point 切分成了多个区域
        成员：是否 own data；格式：data1：share + no_share + value|data2|。。。|restart point offset|restart point count
        反向查找效率很低：需要从上一个restart point开始遍历，一直找到当前 current KV 的前边
        block内部的seek：在多个 restart point 中进行二分查找；每次解析出 restart point 对应的第一个key，进行比较

    BlockContents：记录读出的block的结果，相当于上下文 context

    FilterBlockBuilder：根据数据大小，每 kFilterBaseLg-2k 生成 1 个filter组；这些组构成了一个 filter block；一个filter block里边是多个filter
        作用：收集 key 连续存放在 string 中，最后统一交给 FilterPolicy 生成 filter block（GenerateFilter）
            需要一个临时的 tmp_keys_，需要整理成 Slice 的数组，而不是连续 key 的数组
        格式：|filter 1| filter 2|....|offset of filters|param(bits per key)
        当要计算的 data block 的offset在 kFilterBase（2k）之内时，放在相同的 filter block（见单测）

    FilterBlockReader：根据param（base_lg_），计算出 key 所在的 filter index，传入 filter的数据，让 FilterPolicy 去检查

    BloomFilterPolicy：只计算了一次hash，然后自身进行转换得到新的hash值，共转换K次；用 K 个hash值置 K 个位，用于判断
        Note：filter出错时（或者位数不足），默认返回是 key match，即：认为key大概率存在，后续 read 需要启动真正的查询

 ## Table
    TableBuilder：生成一个完整的 File；key必须是已经排序好的。
     1. 依次按顺序添加 key、value，添加到 filter block、data block 中；达到 options.block_size-4K 后就生成一个新的block
     2. 达到一个block限制后，将block数据进行压缩、计算crc，写入到文件中：data|compress type|crc；重置对应的 filter block，filter与data block一一对应（此刻先不写入，最后一次性写入到文件末尾）；更新 index_block（key -> data block 的 BlockHandle）；meta、filter也是block的一种，因此写入方式与 data block 一致，都是 key + value
     3. 结束时：写入全部 filter block（不压缩）；写入 meta block（filter name -> filter block handler）；写入index block
     4. 写入 footer（meta index offset、index block offset、magcic number）

     Note：在发生 block切换时，pending_index_entry 为 true；因为 index block 中的key记录上一个block的最小值，将上一个 block的 last key 和下一个block的first key作比较。FindShortestSeparator选择一个存储容量最小的 index key节省空间, 并且 >= prev block last key, < next block first key 

    Table：打开并访问一个文件；缓存了 cache_id、filter、index、当前正在iterator访问的一个block
     1. 读取footer、index block的全部内容；Table 中将缓存整个 index block
     2. 读取filter：读出 meta handle -> meta block -> 每个filter的handler；FilterBlockReader 解析名字、filter的param，生成多个filter
     3. 遍历table：返回 two level iterator（level-1 index block 的iterator；level-2 每个 block内部的iterator）
        Table::BlockReader：iterator 遍历时切换 block 用到，读取整个block，并生成新的 level-2 iterator；在访问该block完成后，将delete 为block分配的内存。
     4. 访问key：Table::InternalGet：找到key 对应的 block，访问对应的 filter block；查询filter、查询 key
 
     BASE：ReadBlock，每次将一个block并解压出来；有可能 从file读出来时，file 传回了自己own的一块儿缓存给使用者，那么使用者不要释放

# Other
    Todo: 线程关系，锁处理方式

# QA
    1. 何时使用
    if (snapshots_.empty()) {
        compact->smallest_snapshot = versions_->LastSequence();
    } else {
        compact->smallest_snapshot = snapshots_.oldest()->sequence_number();
    }

    table cache 满了之后，sst被踢出去？

    2. bool Version::RecordReadSample(Slice internal_key) {
    3. 快照存在时，进行compact，之后该如何处理
