---
layout: post
title:  "bthread 运行机制"
date:   2021-06-21 20:06:18 +0800
categories: brpc
---

[注释代码地址](https://github.com/deepld/brpc-annotation/tree/master/src/bthread)

文档只是简介，更多的还是看代码更能深刻理解；代码中所有注释搜索前缀：NOTE(deepld)

# 类结构
## TaskControl
* 全局访问入口，负责TaskGroup的管理，bthread的分发等
* 负载均衡：线程访问按照 pthread id 进行 hash 划分，将 TaskGroup 再分成 PARKING_LOT_NUM = 4 个组。分发任务时，每个分组进行轮询分发
* 动态线程：如果设置了FLAGS_bthread_min_concurrency，一开始只生成 FLAGS_bthread_min_concurrency 个，不够时再动态生成 pthread


## TaskGroup
* 对应于一个pthread，管理运行在一个 pthread 上的所有 bthread 对象；
* main bthread：TaskGroup 默认运行的代码堆栈，在没有其他 bthread 时，运行在 main bthread 上
* 任务队列，包括了两个队列 local（优先级高）、remote（优先级低）
* 这是bthread实现的主要逻辑的位置，

## bthread
### TaskMeta
* 记录了 bthread 对应的全部结构，bthread 运行堆栈、tid、相关lock、是否 interrupted等

### butex
* 实现了类似 mutex 的功能，记录了 waiter list，但是区分为 pthread waiter 和 bthread waiter； 
* 在 bthread wait期间，将会释放自己的 cpu，开始执行 TaskGroup 中的其他 bthread 任务；timer到期和 bmutx的 wakup，会进行唤醒
* 一个工作是保持 pthread 和 bthread 模式下的行为一致

## Timer
* 包含一个全局的 timer，TimerThread

# 任务管理

## Note
    1. 一共4个ParkingLot，每个 ParkingLot 包含 N个 TaskGroup；一个 ParkingLot 的 TaskGroup 共享 cond
    2. bthread 对应的 TaskMeta 结构存储在 ReSourcePool 中。tid 中存储的 version 与task meta中 verison不一致时，表明 bthread 已经被释放了

## 初始化
    get_or_new_task_control -> TaskControl::init：启动 timer 线程，TimerThread::start、创建各个 TaskGroup；
    TaskControl::worker_thread，一个pthread worker的入口
        TaskControl::create_group，TaskGroup::TaskGroup：加入到 task control的 ParkingLot 中，一个slot的多个pthread
            TaskGroup::init：group 内部的队列初始化：local、remote；分配main堆栈；main task 的fn为null，堆栈入口也为空（实际是 TaskGroup::run_main_task）
            TaskControl::_add_group：加入到 task control的 group 数组中；这个结构用于后续的 task steal 选取
        TaskGroup::run_main_task：在此函数中，执行 main task 的任务

## 任务分发
    根据请求者当前所在的 pthread（或者说当前是否在 TaskGroup 对应的pthread线程中执行），有不同的处理

    bthread_start_urgent：
        如果在TaskGoup中执行的（TaskGoup对应的pthread设置了 tls 变量），即：在一个bthread中启动一个新的bthread，立即抢占调用者 bthread 的堆栈，调度执行 TaskGroup::sched_to
        普通 pthread 线程 中执行 bthread 任务：根据 pthread id 选择到一个 TaskGroup 组之后，在里边选择一个 TaskGroup，加入到其 remote queue

    bthread_start_background：
        如果在TaskGoup中执行的，即：在一个bthread中启动一个新的bthread，加入到 TaskGroup 的 local queue
        普通 pthread 线程 中执行 bthread 任务：根据 pthread id 选择到一个 TaskGroup 组之后，在里边选择一个 TaskGroup，加入到其 remote queue

## 任务执行
    TaskGroup::run_main_task：TaskGroup的 main bthread，没有bthread执行时，运行在该 bthread 中
        TaskGroup::wait_task -> steal_task
            wait直到获取到了可执行的 Task：此时，是 TaskGroup 之前执行的 bthread 已经完成，或者 主动放弃了cpu（比如：sleep了）；回到 main bthread 中了
            优先查找自己的 remote 队列，没有的话查询其他 TaskGroup 的 local、remote队列；
            TaskControl::steal_task：为了尽可能均衡的，steal时记录了一个断点，从断点的 park 中选择一个 TaskGroup 开始执行，然后切换到奥下一个 park

        TaskGroup::sched_to：开始执行获取的 bthread，对应于一个 TaskMeta 结构；确保生成了 bthread 对应的ContextualStack，堆栈入口：task_runner（是生成堆栈时设置的）
            TaskGroup::task_runner：只是 bthread 最开始的执行入口，当 bthread 进入后执行到 user m->fn 中，如果被抢占，恢复堆栈后通常在 m->fn 中
            jump_stack：开始执行新的bthread 堆栈，这里进行了执行环境的切换，main bthread 实际上 switch out了！

        TaskGroup::task_runner：新bthread开始执行，这个是所有 bthread 执行的第一入口。这里边包含了用户fn；其他部分代码是因为需要bthread抢占后恢复、bthread 管理的需求
            调用 TaskGroup 之前的 hook，完成 清理、预设工作（见代码注释，举例了2个场景）
            更新 TaskMeta version_butex++，表明 bthread 执行完成，这个 bthread 结构体即将释放了，TaskMeta信息不再有效
            设置 TaskGroup 回调（set_remained），在切换到新bthread之后被其回调，将释放当前 TaskMeta 即 bthread 对应的信息

            butex_wake_except：唤醒所有等待 该 bthread 的线程，比如：bthread_join
            TaskGroup::ending_sched：查看当前 TaskGroup 的 local queue、remote queue，以及其他TaskGroup 中是否还有未完成的任务，有的话继续调度

        TaskGroup::sched_to：bthread 执行完成，继续查找队列中是否还有新的 bthread；如果没有，就切换到 main bthread 的堆栈继续执行
            调用 TaskGroup 之前存储的 hook，完成 清理、预设工作（注意：因为 TaskGroup 只有一个线程，因此 存储 hook 的 _last_context_remained不会担心会被覆盖掉）

## 线程抢占
### 主动释放
bthread_start_urgent、butex_wait、TaskGroup::usleep、TaskGroup::yield 会主动释放 cpu，让渡给本 group 的其他 bthread去执行
    
    TaskGroup::usleep
        set_remain 为 TaskGroup::_add_sleep_event，在新bthread实际执行前执行；通过TimerThread::schedule，设置 ready_to_run_from_timer_thread 的timer
        TaskGroup::sched：调度当前TaskGroup的一个Task，依次查找 local queue、steal_task  
            TaskGroup::sched_to：真正开始执行
        ready_to_run_from_timer_thread：超时后，执行恢复操作。将 bthread 重新加入调度中，TaskControl::choose_one_group、TaskGroup::ready_to_run_remote，加入 remote 队列

    TaskGroup::yield
        set_remain 为 ready_to_run_in_worker，在新 bhhread执行之前，将 last bthread 直接加入到调度中，TaskGroup::ready_to_run，加入到当前 TaskGroup 的 local 队列
        TaskGroup::sched

    bthread_start_urgent
        set_remain 为 ready_to_run_in_worker，在新 bhhread执行之前，将 last bthread 直接加入到调度中，TaskGroup::ready_to_run，加入到当前 TaskGroup 的 local 队列

    TaskControl::signal_task：先给自己所在 ParkingLot 分配一个task，然后遍历其他的 ParkingLot，依次唤醒 n 个worker
        一个ParkingLot的TaskGroup共享 cond，signal_task时 ParkingLot 会notify n 个 TaskGroup（当前n=1）

### last bthread 回调
TaskGroup::set_remained：设置下个抢占自己的任务完成之后的回调，以便将来恢复自己 bthread 的调度执行 _last_context_remained

    bthread 正常结束，下一个bthread执行之前，先清理 last bthread 的 TaskMeta -> _release_last_context
        设置：TaskGroup::task_runner tail，bhread执行完成、ending_sched之前
        回调：TaskGroup::sched_to 尾部执行

    bthread 被抢占，在执行抢占bthread的代码（TaskGroup::sched_to）之前，设置回调 ready_to_run_in_worker：丢到 TaskGroup 的 local queue 中，然后 signal
        回调：TaskGroup::task_runner head, 这样当新的 bthread 执行前，会将被抢占的 bthread 重新丢回到 TaskGroup 的local队列中，等待被调度

    butex_wait：当前bthread sleep，下一个bthread执行之前，将last bthread 加入到 butex 的 wait list -> wait_for_butex
        回调：TaskGroup::task_runner head, 见后续 butex

# 其他内容
## 资源管理
使用 ReSourcePool 管理 bthread meta、butex；使用的句柄是Type中对一个的一个 in32_t 的 version，通过version值来判断，句柄是否已经无效了
    
    global保存该type的 RP_MAX_BLOCK_NGROUP 个BlockGroup，每个BlockGroup包含 N 个block，每个block有 BLOCK_NITEM 个type

    分配：每个线程一个thead local的pool，get时先检查 local free list、global free list、local 分配的 block
        1. 第一次上述分配途径都是空，将从 global 生成一个 block，并记录其 index，从block拿出一个 type 返回，其中 block index + block offset 组合 instance id
        2. 后续可以通过 id 取出这个offset，算出 type 所在的 BlockGroup、Block、block 内 offset

    释放：首先还给local 的 free list，满了之后还给 global free list，这样多线程可以共享 global 的free list；

    访问有两种方式：
    1. 使用指针：直接返回的是对象的 ptr，或者是 ptr->int 的一个对象地址；int中还可以存储其他信息，比如 butex
    2. 使用索引：返回的是 int64，根据这个index可以找到对应偏移的 对象地址，见：address_resource

## mutex
### use in pthread
    futex_wait_private、futex_wake_private：用于 ParkingLot，一组 TaskGroup 共用一个 mutex
        ParkingLot 中存储的是一个 int，这个int的 &int add 对应于一个全局 map 的key，保存了这个 mutex 的真实对象 SimuFutex，包含 ref 等

    wakeup_pthread：ButexPthreadWaiter->sig 这个int的值存储了 cond 的状态。如果已经 wakeup 过，对应的值是 PTHREAD_SIGNALLED
    wait_pthread：期望ButexPthreadWaiter->sig 值是 PTHREAD_NOT_SIGNALLED，否则认为不需要 wait sleep

### use in bthread
butex 存储在 ReSourcePool 中，但返回的是 &butex->value, 这里存储了其引用；使用时，bthread 的 TaskeMeta 存储了waiter，ButexWaiter -> butex 能访问到 butex

    butex_wait：每次wait，就增加一个 ButexBthreadWaiter 结构挂在 butex
        如果设置了超时，在timer到期后，会执行： erase_from_butex_and_wakeup -> erase_from_butex
            从 ButexWaiter 保存的成员中获取 Butex；更新 ButexWaiter的stat为 WAITER_STATE_TIMEDOUT(用于唤醒后读取)，从 butex waiter list中删除；
            将 ButexWaiter 保存的 bthread 加入到调度中去，ready_to_run_general；

        调度到其他 bthread 之前，set_remain为 wait_for_butex
            检查 last bthread wait的条件是否已经达到了，那么立即唤醒（加入调度中）
            否则加入到 butex 的 wait list 中去，等待 wakup

        butex_wait 被调度再次执行时，检查 waiter 的 status，得出错误码

    butex_wake：从waiter list中取出第一个 waiter，将其对应的 bthread 加入到调度队列中

    TaskGroup::interrupt：只是唤醒一个 bthread，而不是wait在一个butex上的所有bthread
        interrupt_and_consume_waiters：根据 bthread 保存的waiter信息，找到对应的 butex、以及 bthread是否加入了timer；比较 waiter 中保留的 butex version，与 bthread 的versinon是否一致
        bthread 处于 wait 状态，必然是下边两种情况之一：

        1. butex 而陷入的wait：
            erase_from_butex: 从 ButexWaiter 保存的成员中获取 Butex；更新 ButexWaiter的stat为 WAITER_STATE_INTERRUPTED(用于唤醒后读取)，从 butex waiter list中删除；
            set_butex_waiter：再讲 waiter 设置到 bthread 中保存，唤醒 bthread 之后，访问 waiter 的 state 获取错误码
            butex wait 可能也设置了超时，包含一个 timer id，但这个 timer id保存在 bthread记录的waiter中；当 bthread 唤醒之后，尾部代码会删除timer

        2. 因为sleep而陷入的wait：
            bthread 的 TaskMeta中记录有 timer id，根据这个删除 timer

## 堆栈管理
使用了轻量级的 boost context https://github.com/boostorg/context

# 其他工具
1. ExecutionQueue：相当于一个 MPSC queue，提供 wait-free 语义，实现了异步串行化的执行；用 message queue 的方式来解决竞争在某些场景下替换 mutex 可以取得较好的性能
   支持高优先级的抢占





