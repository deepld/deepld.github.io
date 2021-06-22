# ENV
    ## QA
    Random::Skewed 有何好处？

 ## Class
    EnvWrapper：用于封装一个已有的env，并能方便的覆盖其中部分行为
    Limiter：posix env使用 两个limiter，限制 mmap、open file的个数，发放令牌。
        文件打开默认都加了 kOpenBaseFlags = O_CLOEXEC，fork 时不会

    SequentialFile：单线程访问
    RandomAccessFile：多线程访问；
        PosixMmapReadableFile：mmap 令牌足够时，使用mmap进行映射后访问，比较适合 slice的访问模式，不需要额外分配内存保存 read 的结果
        PosixRandomAccessFile：mmap 不足时，直接裸访问；超过 open handle 限制时，每次 read 先open 再close；

    PosixWritableFile：自带 buffer 方式是write
        针对menifest文件，其sync之前，要先确保 dir 的修改已经sync完成了

    PosixLockTable：自己实现的一个std::set来维护，仅仅用 F_SETLK 无法隔离进程内的访问。

## 线程
    生成没有关闭句柄的后台线程

## logger
    PosixLogger：构建output时，先使用stack buffer，不够时再new出来
    Logging：是为了进行 string 和 int 的相互转换，打印出 human readable的信息

## other
    Random：比较简单的
    Status：使用一个连续空间，记录 code、message

# Helper：
    ErrorEnv：用于test中的错误注入

## coding
    varint 用于存储整形数据，通常是 length
    定长存储：对 int 类型进行 encode，确保存入的是小端的字节顺序（如果本身就是小字端，实际可以不进行转换）
    变长存储：使用 128 进制存储数据；小于128时，只需占用1字节；当数字较大时，占用的空间比直接存储更大
        每个字节使用7位，最高位表示是否需要更多字节

# 开发技巧
## QA
    NoDestructor、SingletonEnv 用于 static 函数 中的 singleton，应该使用 once 来初始化？
    NoDestructor：先申请内存，再new出来对象，这样的对象就不会被调用 析构了
    SingletonEnv：类似于 NoDestructor

## List
    HandleTable：比gcc默认的hashtable要快；这里用了指针的指针来操作list，写起来比较清晰；
        slot 的个数和当前 item的个数 相等时，说明 hash 表比较拥挤了，需要 2 倍增长
        ptr是指针的指针类型，指向的是 node 的 next_hash 位置的地址；
            遍历：当 *ptr 为 nullptr时，此时ptr指向的是最后一个 node 的 next_hash 的地址；否则指向的是 对应 node 的 prev
            删除：ptr 设置为对应 node 的 prev 的 next_hash，直接更改 *ptr 即可删除 node
        
    Cache接口中，使用了 struct Handle {} 来避免使用 void* 作为 Cache handle的类型

## 编译器检查
printf参数检查
    
    编译时检查：__attribute__((__format__(__printf__, 2, 3)))

使用clang的lock标志，辅助检查 lock问题
    
    clang 提供的编译期能够进行一定 race conditions and deadlock 的检查机制；已经是标配了，提供了较好的静态检查机制。
    
    额外定义一个类（通常封装mutex），标记其为某种 CAPABILITY ”能力”。在其成员函数上，定义出对能力的各种操作；之后，这个”能力”**就能用来做编译检查判断了**
    比如：GUARDED_BY、REQUIRES 防止竞态；ACQUIRED_BEFORE、EXCLUDES and REQUIRES(!__); 用来防止死锁
    详见: https://clang.llvm.org/docs/ThreadSafetyAnalysis.html#mutexheader
    
## 内存底层库
    使用了 tcmalloc，之前用 jemalloc，应该差不多

# Test
故障注入：fault_injection_test.cc
    
    FaultInjectionTestEnv：中记录了两次sync间隔之间信息：新创建的文件、每个文件已经sync到的位置；
        可以将尚未 sync 的文件那部分数据，进行truncate清理，或者删除执行任意sync之前新创建的文件，来模拟故障；详见注释

故障注入：db_test.cc

    SpecialEnv：模拟文件系统故障，磁盘满、sync失败、写入失败等；生成定制的 WritableFile

Fake实现：memenv.cc

    InMemoryEnv：