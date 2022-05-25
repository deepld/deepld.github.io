
## TODO
    ## Function
    1. // compression algorithm (see `ColumnFamilyOptions::sample_for_compression`).
    2. ChecksumType：enum ChecksumType，md5、hash
    3. // dictionary meta-blocks
    4. TEST_SYNC_POINT_CALLBACK

## Format
    https://github.com/facebook/rocksdb/wiki/Rocksdb-BlockBasedTable-Format
   
    ## Format
    <beginning_of_file>
    [data block 1]
    [data block 2]
    ...
    [data block N]
    [meta block 1: filter block]                  (see section: "filter" Meta Block)
    [meta block 2: index block]
    [meta block 3: compression dictionary block]  (see section: "compression dictionary" Meta Block)
    [meta block 4: range deletion block]          (see section: "range deletion" Meta Block)
    [meta block 5: stats block]                   (see section: "properties" Meta Block)
    ...
    [meta block K: future extended block]  (we may add more meta blocks in the future)
    [metaindex block]
    [Footer]                               (fixed size; starts at file_size - sizeof(Footer))
    <end_of_file>

    ## Detail
    Each data:  s-len，no-s-len，v-len | shared key | no shared key | value
    Each block：[share key prefix and data]* | restart offset array | (index type-1 + restart num)-4 | tailer checksum
    。。。。
    Footer：checksum type-1 | meta handle | index handle（optional） | version-4 | magic-4；

    ## Block
    1. block 内部使用 restart interval 进行二分查找；可以再额外添加一个 DataBlockHashIndex，使用 16 位的　offset，不超过64K，加快点差
    2. DataBlockHashIndex：存储每个key对应的restart interval index；查找时，如果检测到冲突，回退到 restart interval binary search

    ## TODO
    cuckoo table：https://rocksdb.org/blog/2014/09/12/cuckoo.html

## Build
    ## QA

    ## Function
    1. 自适应构建 compress dict，提升压缩率：设置 max_dict_bytes，开始时处于 buffer 状态，所有block都只缓存。当大小达到dict判断要求后，进入 unbuffer 状态，直接写入文件
    2. PropertiesCollector：构建过程中，定制自己的 collector 逻辑，产出需要的 meta，类似于回调；信息写入到 properties block 中
    3. Parallel Compression

    ## Flow
    1. 依次 add key
       1. one key 添加到 index、filter block，添加到 datablock
       2. block 写满：检查 FlushBlockPolicy，如果写入时达到一定size，或者已经超过了预留 deviation，就生成一个新的block
          1. Flush 到文件：WriteBlock -> WriteRawBlock，压缩data（tailer 之前的部分），tailer部分不压缩
          2. index block 中填一个一个 data block handle 的 entry
          3. 检查是否 prepopulat cache：一些场景中，刚刚写入到file的data是会被立即访问到的
          4. 检查是否需要 block align，是否需要加入到block cache
       3. 回调 Collector
    2. 直到 Finish：
       1. 将所有meta block写入，index、filter、properties
       2. 并加入到到 meta index block  

## Index
    ## Function
    https://github.com/facebook/rocksdb/wiki/Index-Block-Format
    prefix seek：https://github.com/facebook/rocksdb/wiki/Prefix-Seek

    BlockBuilder::AddWithLastKey
    hot path：减少分支预测

    ## Note
    1. 相关配置
       1. block_restart_interval 设置为1，可以防止block 访问时候再做 linear scan
       2. kShortenSeparators：使用 shorten 模式，计算出小于 next key 大于 last block 的最小 successor，减少 index key 的 size
       3. include FirstKey: index 中记录了 datablock 的范围，减少读取 datablock；或者延 defer read 
       4. index_value_is_delta_encoded：基于index entry的相似性，可以使用 delta value，即下一个 entry 是基于上一个entry变化的，减少记录的内容量。而 data block 不会使用 value delta encode
    2. Hash index 用在block内部，以及 index block 中
    3. 当前实现中，index 和 filter 是保持一致的。如果filter block cut了，相应的 index block 也要 cut

    ## Note
    在添加每个key时，都回调了OnKeyAdded。很多场景不需要，目前是用于记录 first key in current block

    ## Build
    -- ShortenedIndexBuilder
        Successor last key -> block offset | [first key]，每个data block flush时候生成一个记录
        构建每个 entry 时，提供了 block first key（optional），block last key，next block first key（optional）

    --HashIndexBuilder
        https://github.com/facebook/rocksdb/wiki/Data-Block-Hash-Index
        快速找到 key 所在的 restart interval；空间占用：默认每个key占用4/3个字节；记录每个 key hash 后的，在 array 中记录其所在的 restart interval。array 为 int_8，因此只支持 < 255 个 restart interval。超过此范围的，不支持 hash index

        1. index_block_restart_interval = 1，当前只支持此模式；需要指定 prefix_extractor，只支持 prefix hash，以减少空间占用
        2. user_comparator()->CanKeysWithDifferentByteContentsBeEqual()，如果不相同的key会被comparator当做是相同的，那么就不能使用 hash index
        3. 自身包含了一个 ShortenedIndexBuilder，hash 只是补充；将 extractor 之后相同的 prefix 作为一个item存储
        4. 只使用 prefix index，否则冲突太大占用空间多；因为key是连续存储的，找到 prefix 就已经缩小了很多范围
        5. 按照 key 的 prefix 建立索引，指向对应的 Block；与 main index 不是一一对应的，会跨block（hash index 按前缀，main index按 size）
        6. 结构：prefix|restart index|count；prefix string、hash meta 是作为两个block分开存储的
           pending_block_num_：记录 prefix 相同的连续的key，在main index中跨越了几个 restart interval（每个RI对应于一个data block）

    --PartitionedIndexBuilder
        index 分级存储，类似于页表分级，减少内存占用空间：https://github.com/facebook/rocksdb/wiki/Partitioned-Index-Filters

    ## Read
    -- IndexReader：指向 index block，生成 index iterator
    -- BinarySearchIndexReader：始终使用 total_order_seek = true
    -- HashIndexReader：可以使用 hash index 或者 binary index
       -- BlockPrefixIndex: 读取 hash index
       1. 生成iterator读取时，根据 prefix 存储信息重构 hash index，会进行不少计算，对性能有一定影响；这些解析后的 hash index 必然与iterator的生命周期一致，hold在内存中
       2. 即使启动了cache index选项，只会 cache binary search index 的部分。hash index 的部分每次是从 file 中重新读取的。见 HashIndexReader::Create
       3. hash 计算方式：根据 prefix 快速定位到 start block
          1. 数组1中，不冲突的 hash pos直接存储 block start位置；发生冲突的 hash pos 指向另外一个数组的一段
          2. 数组2中，有足够的的位置。每段存储了所有冲突所有冲突prefix对应的所有block，并且这些block是从小到大存储的。这样先hash pos找到这个part，然后在这个小范围内二分查找。
    -- PartitionIndexReader
        1. table open时，调用 CacheDependencies，使用prefetch读出 block，并将所有的 index block 都存储在 block cache中；如果设置了pin，那么reader map 中保存所有 block handle
        2. 生成 iterator 时，如果之前 open 时设置了 pin_parttion，那么此时 reader 中已经包含了所有的 partition index block，直接生成通用的 TwoLevelIterator来访问
        3. 生成 iterator 时，如果没有pin过，生成 PartitionedIndexIterator 来从 cache 或者 file + prefetch 或取block，用index iterator + data iterator 来访问。

## Filter
    ## Format
    https://github.com/facebook/rocksdb/wiki/Partitioned-Index-Filters
    https://github.com/facebook/rocksdb/wiki/RocksDB-Bloom-Filter

    ## Todo
    read FilterBitsBuilder、FilterBitsReader

    ## Note
    1. table build 时指定 FullFilter 还是 Partition filter；传入 option 的 FilterPolicy 生成底层 filter 类型
       1. BlockBasedFilterBlockBuilder 已经废弃。按2k建立bloom filter，没有考虑cache line对齐
    2. 传递了 extractor 之后，就会建立 prefix key filter；另外可以选择，是否再建立 whole key filter；他们在同一个filter中建立
    3. 使用partition方式时，index 和 filter 的切分策略一致，任何一个写满了，两个都要生成一个新的 partition
    4. 配置
       1. optimize_filters_for_memory：使用 Ribbon Filter

    ## Build
    --FullFilterBlockBuilder
        1. whole key 和 prefix key可能相同。确保不添加重复的 whole key、prefix key；先各缓存一份，直到找到一个不一致的，才add进去。
        2. 存储在block cache 中的，是已经解析完成的 ParsedFullFilterBlock，不是 raw block

    --PartitionedFilterBlock
        1. 复用 index builder来构建两极 filter，在 FullFilter的基础上，每 N 个key建一个分区
        2. prefix filter模式：不同key的有相同的prefix，这些key刚好在不同的filter partition下，需要将prefix加到两个partition中。see：PrefixInWrongPartitionBug
        3. 需要关联到 partitioned index，用于协调 index、filter 之间的 cut 操作；他们之间是一一对应得，即划分 partition 的key一致。

    ## Read
    --FullFilterBlockReader
    --PartitionedFilterBlockReader
        1. 先用level1的 index block找到对应的partition block，再用 FullFilterBlockReader 方式读取 ParsedFullFilterBlock
        2. 对 partition block，always use block cache，强制设置。见：GetFilterPartitionBlock

## Table Reader
    ## TODO
    1. prefetch 和 preload 的区别？

    ## Open
    1. 读取footer、meta index block，并通过 meta index 访问到 properties block
    2. 检查 read 时传入的prefix extractor与 meta中记录的是否一致并协调，否则将无法使用 prefix hash indexe

## Iterator 
    https://github.com/facebook/rocksdb/wiki/Iterator-Implementation
    
    ## QA
    1. block_contents_pinned_：如何起作用？

    ## Todo
    PinnedIteratorsManager
    Iterator 中的 Global SequenceNumber，作用是什么

    ## Class
    MetaBlockIter：访问 properties
    BlockBasedTableIterator：聚合index、data block 对外访问
        IndexBlockIter：访问index block内部，binary index、hash index，找到value存储的data block handle
          -- delay read：index中默认只保存 data block的 last key。如果也配置了first key，那么 data block 就可以延迟解析，一些情况下可能就不需要解析了。is_at_first_key_from_index_，但是如果访问到了 value，需要 PrepareValue，确保 data block 启动实际解析
        DataBlockIter：访问每个data block；内部可以有 hash index
    MergingIterator：使用 heap，return merge iterator value
        
    ## Option
    1. total_order_seek：seek时是否使用filter先进行check
        1. 所有data block iterator 都设置为 true
        2. false：有prefix exaxtor，使用filter先检查对应的key是否存在。
    2. whole_key_filtering：按whole key也建立filter
    3. iterate_upper_bound 影响 auto_prefix_mode 不能使用
    

    ## Note
    1. 即使定位到了restart interval之后，Block必须从 restart interval 的 start开始读取，因为使用了 delta key encoding

    ## DataBlockIter
    -- PrevImpl：
        找到当前 restart interval中，排在search key之前的所有的 item，缓存起来到 CachedPrevEntry，用于后续 Prev 遍历
        如果key 是delta encode的，后续不能直接从buffer中读取出来，而且会丢失，那么需要copy 到临时的buffer中去
    -- SeekForGetImpl：
        点查优化：优先使用 hash index 来定位 key 所在的 restart interval
            减少了 restart interval 的二分查找，这个查找过程，需要 decode 对应restart interval 的 start key出来才能比较
        如果hash中没找到，仍然需要继续搜索最后一个 interval；如果当前block的所有的 key 都小于 target，说明需要继续往后边的 block 去搜索，因此返回true，见注释，

## Storage Layer
    分层存储：block cache -> persisit cache  -> file
    https://github.com/facebook/rocksdb/wiki/Prefetching-in-RocksDB（empty）
    
    ## Persistent cache：分层存储的一级，cache 的一个补充
    https://github.com/facebook/rocksdb/wiki/Persistent-Read-Cache
    在多个文件中存储 block cache，使用 LRU index；配置为存储 compress 或者 uncompress 的数据
    
    ## BlockFetcher：数据访问代理，局部
    在block cache miss之后，block fetcher 提供数据访问代理和解压；从文件中获取数据后，加到 persistent_cache 中去（fill_cache）
    内存管理：提供一个5k的 stack buf 临时使用，最终根据数据类型，从不同的 CacheAllocationPtr 分配 heep 内存作为 final result
    访问顺序：-> uncompress persist cache 
         -> prefetch buffer (check already cached part -> 如果需要 readahead，此处直接进行 file io，就不需要后续步骤了）
         -> compress persist cache -> file io (并会尝试存储到compress persist cache；但不会存储到 FilePrefetchBuffer）

    ## BlockPrefetcher：判断是否 prefetch
    1. 用于 table iterator，预测 data block 是否发生了顺序读取，如果是就生成一个 FilePrefetchBuffer。
    2. 在 BlockBasedTableIterator、PartitionedIndexIterator 中用到了

    ## FilePrefetchBuffer
    代理已经判断为 readahead 的 file read，也负责进行 decompress（因为这些都涉及到 pool 或者 heep 内存分配）
    提供一个访问缓存（compressed raw data）；每次多读取一些数据并缓存下来：但后续的读取必然要先 copy 到 buffer，再 copy 到result
    buffer的数据只能是顺序读取，使用双buffer切换（实际3 buffer）
    
    ## 访问说明            
    1. table open
       1. 先到 filesystem prefetch，然后通过 FilePrefetchBuffer 读取 head 附近的 512K，确保meta都提前 prefetch 到 buffer
       2. 此时是第一次访问index，不会立即加入到cache中去（但实际上index block已经解析出来了）？？？
    2. 访问顺序 
       1. block cache -> BloclFetcher 进行后续访问(见BloclFetcher)
       2. 没有命中的cache，通过BlockPrefetcher获取data后，更新到cache中
    
    ## Prefetch
    1. table open 的时候创建一个临时 FilePrefetchBuffer
    2. table Iterator：每次 InitDataBlock 会触发 prefetch 检查
       策略：filesystem prefetch -> use FilePrefetchBuffer
    3. 即使是从 block cache中访问到了data，也会更新 prefetch_buffer 中的 parten，确保后续如果触发了IO 能够继续做 readahead

## Cache 
    https://github.com/facebook/rocksdb/wiki/Memory-usage-in-RocksDB
    https://github.com/facebook/rocksdb/wiki/Block-Cache
    https://github.com/facebook/rocksdb/wiki/Simulation-Cache

    ## QA
    1. prefetch：这样读取的data，不会进入block cache；用于 disable index/filter/footer into block cache？

    ## Function
    1. 提供 compress cache、uncompress cache；使用dio时，可以用 compress cache 替代 page cache
    2. 使用cache之后，可以更好的进行 memory 的追踪和限制(如：CacheReservationManager)；默认，partitioned indexes/filters都使用 cache 

    ## Class
    CacheReservationManager：
        对堆栈中分配的内存，委托 Block Cache 进行统一的大小管理，似乎是防止 heep + block cache 总和超过限制？（TODO）
        对新增的 heap 内存，就分配一个unique CacheKey with null value 存储到 cache中，固定大小为 kSizeDummyEntry
        只有当 used < 2/3 Reserve 时才进行decrease，减少频繁的 shrink 和 extend 操作（涉及到 LRU cache的 lock）
    
    ## CacheKey
    https://github.com/pdillinger/unique_id，类似于UUID的效果，生成存储在 block cache 中的key
    CacheKey：用两个int64表示，转为Slice后当做unique key；对不同类型的使用，划分在不同的range范围
    OffsetableCacheKey：universally unique，有很高的unique可能性（如果冲突了，后果是什么？）

    ## table build，每次生成一个新的 block时
    1. uncompress cache：存储后续可直接使用的类型，index、data直接存储为 Block；filter 可存储为 parsed filter，方便后续直接使用
    2. compressed cache：如果设置了，将block加入进去，并 invalid os page cache

    ## index and filter
    1. prefetch：即使尚未需要访问（尚未进行get，但是），也先都把filter block读出来
    2. cache_index_and_filter_blocks
       1. pin：是否pin在cache中不swap out；例如，open 时 prefetch index block 之后，立即将其移动到 LRU 中等待swap out
       2. open时 index reader 的block,指向cache中缓存的value；可能被cache swap out，后续访问时需要再load进来插入cache
   
## Compress
    ## TODO
    CacheReservationManagerImpl
    CompressionContext
    ParallelCompressionRep
    StartParallelCompression
    compression_dict_buffer_cache_res_mgr
