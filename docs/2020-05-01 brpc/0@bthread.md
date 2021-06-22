---
layout: post
title:  "bthread 运行机制"
date:   2021-06-21 20:06:18 +0800
categories: brpc
---

[注释代码地址](https://github.com/deepld/brpc-annotation/tree/master/src/bthread)

文档只是简介，更多的还是看代码更能深刻理解

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
任务触发：唤醒等待者

## 任务分发
根据请求者当前所在的 pthread（或者说当前是否在 TaskGroup 对应的pthread线程中执行），有不同的处理

    bthread_start_urgent：
        如果在TaskGoup中执行的，即：一个bthread中启动一个新的bthread，立即抢占调用者 bthread 的堆栈，调度执行 TaskGroup::sched_to
        普通 pthread 线程 中执行 bthread 任务：，根据 pthread id 选择到一个 TaskGroup 组之后，在里边选择一个 TaskGroup，加入到其 local queue
    bthread_start_background：
        如果在TaskGoup中执行的，即：一个bthread中启动一个新的bthread，加入到 TaskGroup 的 remote queue
        普通 pthread 线程 中执行 bthread 任务：，根据 pthread id 选择到一个 TaskGroup 组之后，在里边选择一个 TaskGroup，加入到其 local queue


## 任务执行
bthread 的 worker 线程

    TaskGroup::run_main_task：TaskGroup的 main bthread，没有bthread执行时，运行在该 bthread 中
        TaskGroup::wait_task -> steal_task -> TaskControl::steal_task
            wait直到获取到了可执行的 Task。优先查找自己的队列，没有的话查询其他 TaskGroup的队列；
            此时，是 TaskGroup 之前执行的 bthread 已经完成，或者 主动分期了cpu（比如：sleep了）；开始执行 main bthread了
        TaskGroup::sched_to：开始执行获取的 bthread，对应于一个 TaskMeta 结构；确保生成了 bthread对应的ContextualStack
            jump_stack：开始执行新的bthread 堆栈，这里进行了执行环境的切换，main bthread 实际上 switch out了！

        TaskGroup::task_runner：这个是所哟 bthread 执行的第一入口，这里边包含了用户fn；其他部分代码是因为需要bthread抢占后恢复、bthread 管理的需求
            调用 TaskGroup 之前的 hook，完成 清理、预设工作（见代码注释，举例了2个场景）
            更新 TaskMeta version_butex++，表明 bthread 执行完成，这个 bthread 结构体即将释放了，TaskMeta信息不再有效
            设置 TaskGroup 回调，再次调用时，将释放当前 TaskMeta 即 bthread 对应的信息

            butex_wake_except：唤醒所有等待 该 bthread 的线程，比如：bthread_join
            TaskGroup::ending_sched：查看当前 TaskGroup、其他TaskGroup 中是否还有未完成的任务，有的话继续调度

        TaskGroup::sched_to：bthread 执行完成，切换到 main bthread 的堆栈继续执行
            调用 TaskGroup 之前的 hook，完成 清理、预设工作（注意：因为 TaskGroup 只有一个线程，因此 存储 hook 的 _last_context_remained不会担心会被覆盖掉）

   
# 其他内容
## 资源管理
使用 ReSourcePool 管理 bthread meta、butex；使用的句柄是Type中对一个的一个 in32_t 的 version，通过version值来判断，句柄是否已经无效了

## 堆栈管理
使用了轻量级的 boost context https://github.com/boostorg/context




