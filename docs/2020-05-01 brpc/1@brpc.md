## Accept
    Acceptor::OnNewConnectionsUntilEAGAIN
    Socket::Create -> Socket::ResetFileDescriptor -> EventDispatcher::AddConsumer
## Read 
    ## 事件通知
    EventDispatcher::Run -> Socket::StartInputEvent -> bthead run Socket::ProcessEvent -> InputMessenger::OnNewMessages
    
    ## 切割
    InputMessenger::CutInputMessage -> ParseRpcMessage (loop handle.parse 直到成功) -> 生成 MostCommonMessage（未解析）, QueueMessage

    循环从socket中不断读出数据直到 recv buffer empty
    尝试用多种协议解析recv到的数据，直到成功找到匹配的 protocol，记录下这个index（一个connection通常是固定的protocol），根据协议meta切割出消息来（不是所有protocol都支持，比如redis）
    正常状态下，解析出的第一个消息先hold，其他解析的消息丢到 bthread 中去执行，第一个 message 抢占当前dispatch线程去执行
        DISP把所在的pthread让给了新建的bthread，使其有更好的cache locality，可以尽快地读取fd上的数据。而EDISP所在的bthread会被偷到另外一个pthread继续执行
    
    ## 处理
    ProcessInputMessage（提前注册好的 msg->_process）
## Send
    Clourse->Done -> SendRpcResponse -> Socket::Write -> Socket::StartWrite -> 
        IOBuf::pcut_into_file_descriptor -> IsWriteComplete -> Socket::KeepWrite -> Socket::DoWrite

    只有当前第一个在write过程的bthread可以写入socket，如果这个request的数据没有写入完成，或者有未完成的new reqeust，就转到新的 bthread 执行 KeepWrite 继续写, 等价 wait-free MPSC
    如果队列中有多个 request, Socket::IsWriteComplete 会把他们 reverse，按照插入的顺序进行后续send 
        note：request 的 next 初始为 WriteRequest::UNCONNECTED，Socket::StartWrite中交换 write_head 并设置 req->next 是两步非原子，IsWriteComplete中会 loop 等待其 next设为 UNCONNECTED之外的某个值（null或者另一个request）

    Socket::DoWrite 里边将多个request尽可能的放在一个writev中发送提升效率；无法写入时 Socket::WaitEpollOut 等待被唤醒（这不需要再被上层异步回调了，类似于同步代码）
        WaitEpollOut：使用 butex_wait 及一个过期时间，会立即释放掉 cpu，等待 bthread 的唤醒；唤醒后 RemoveEpollOut，减少 epoll 事件数量
        Socket::HandleEpollOut：事件到来后，bthread::butex_wake_except 唤醒等待

## Redis
    redis 因为没有直接写出 package 长度，无法只进行切割；因此切割时候就同时执行了 parse

## RPCZ Time
    1. InputMessenger::OnNewMessages：before read
    set_received_us -- Received request(57) from 127.0.0.1:51478 baidu

    2. ProcessRpcRequest head：after read、cut message、bthread switch or not
    set_start_parse_us -- Processing the request in a new bthread

    3. ProcessRpcRequest tail：after ParsePb、check、get method
    set_start_callback_us -- Enter example.EchoService.Echo

    4. SendRpcResponse：done->run
    set_start_send_us -- Leave example.EchoService.Echo

    5. SendRpcResponse：pb serialize
    sent_real_us -- Responded(32)
    -- 不是准确的时间：第一个send是准确的，后续排队的send，只要丢到了队列就算