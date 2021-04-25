
# 结构说明
    DBImpl::Writer：带管理功能，临时整合一次batch写入的上下文（增加了写入状态的管理、唤醒方式等），以便能和其他batch write 并列进行处理
    WriteBatch：代表一次batch写入；batch内的所有key、value都encode到一起；格式：seq-8|count-4|[type|K1|V1,type|K2|V2......]

# 主要功能
    1. 并发处理能力，对同时执行的写入进行聚合，减少磁盘写入次数，聚合小IO；另外有一种实现方式（批量等待方式），设置一个wait时间和总的size limit，当前超过 wait时间或者已经到达的数据超过 size limit时就进行flush；level db的处理方式不会给写入请求带来较大延迟，对iops友好；批量等待方式会多产生一些延迟，但是对大吞吐友好。

# 执行流程：db_impl.cc 
    1. 针对多线程并发写入的场景，维护一个 deque（writers_），所有当前未执行完成的 writer 都在这里等待。
    2. 获取写入许可：MakeRoomForWrite，几种情况下，需要等待或者退出：
        1. 发生了后台故障（比如后边遇到的 log sync失败），是不可修复错误，不再接受写入
        2. 达到了write限制（写入太快，memlayer flush太快，L0层的文件个数达到了Slow down限制），等待1ms
        3. MemLayer还有足够的空间，那么允许立即写入
        4. 有正在 flush 的 immutable 的 memlayer，那么需要一直等到其 flush 完成（不是应该用双buffer机制？）
        5. 达到了write限制（写入太快，memlayer flush太快，L0层的文件个数达到了 Stop的硬限制），一直等待相应的compact完成
        6. MemLayer没有足够空间了，但是各方都没有达到限制，说明只是本e次的MemLayer满了，切换到immutable，并生成新的log和MemLayer；等待compact调度

    3. 选择队列中一些 writer 合并在一起 BuildBatchGroup，他们有相同的 seq
        1. 执行选择的依据：
            1. 小write不能与太大的聚合在一起，否则带来不好的写入体验，所以要设置一个上限（默认1M，top writer size < 128K时，设为 size + 128K ）
            2. 选项不同的不能放在一起，SLA不一致，如：以 top writer为标准，如果后续有write指定了sync，但是与top writer不一致就不能执行
            3. 如果只有一个writer，就使用原来的 WriteBatch；否则新建一个WriteBatch，将 小于 max Size的所有Batch整合成顺序的string（以便后续顺序写入到日志，加入到 memlayer）
    4. 写入日志和 MemLayer 
    5. 唤醒所有写入完成的 Writer（last_writer之前的）；唤醒下次需要写入的batch 头部的 writer，在其他线程开始新的循环

# 错误处理能力
    写入日志失败：设置失败标志，所有等待后台线程的线程全部 wakeup，并且都将返回错误；系统也不再接收新的写入了。

## Memtable
    ## format
    memory key：  key size | key | seqno + type | value size | value
    Internal key：user key + seqno

    ## Comparator
    InternalKeyComparator: 先正向比较 user key，然后比较 seqno

    skip list：相当于一个 set

# Improve
    无锁队列的方式进一步优化，但是会造成 cpu 忙等待；有时候用 mutex 方式反而性能更好

    并且由当前位于第一位的线程执行 后续操作，执行完成后再唤醒其他请求线程。
    锁的保护范围

# QA
    1. 可能的bug：启动时进行数据恢复，写入seq之后立即读取出来，WriteBatchInternal::SetSequence(write_batch, last_sequence + 1);？
    而实际write时，一个batch中的所有log，应该是共享 seqno？----