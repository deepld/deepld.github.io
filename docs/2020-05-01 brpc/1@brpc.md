
# Server

## Accept
    Acceptor::OnNewConnectionsUntilEAGAIN
    Socket::Create -> Socket::ResetFileDescriptor -> EventDispatcher::AddConsumer

## Request 
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
    
    ## 回应
    Clourse->Done -> SendRpcResponse -> Socket::Write -> Socket::StartWrite -> 
        IOBuf::pcut_into_file_descriptor -> IsWriteComplete -> Socket::KeepWrite -> Socket::DoWrite

    只有当前第一个在write过程的bthread可以写入socket，如果这个request的数据没有写入完成，或者有未完成的new reqeust，就转到新的 bthread 执行 KeepWrite 继续写
    如果队列中有多个 request, Socket::IsWriteComplete 会把他们 reverse，按照插入的顺序进行后续send 
        note：request 的 next 初始为 WriteRequest::UNCONNECTED，Socket::StartWrite中交换 write_head 并设置 req->next 是两步非原子，IsWriteComplete中会 loop 等待其 next设为 UNCONNECTED之外的某个值（null或者另一个request）

    Socket::DoWrite 里边将多个request尽可能的放在一个writev中发送提升效率；无法写入时 Socket::WaitEpollOut 等待被唤醒（这不需要再被上层异步回调了，类似于同步代码）
        WaitEpollOut：使用 butex_wait 及一个过期时间，会立即释放掉 cpu，等待 bthread 的唤醒；唤醒后 RemoveEpollOut，减少 epoll 事件数量
        Socket::HandleEpollOut：事件到来后，bthread::butex_wake_except 唤醒等待

## Redis
    redis 因为没有直接写出 package 长度，无法只进行切割；因此切割时候就同时执行了 parse

# Client
    ## Send Request
        Channel::Init -> hannel::InitSingle
            Channel::InitChannelOptions：设置protocol和相关处理函数、链接类型、协议index        
            SocketMap::Insert：从全局map中，找到或者创建 server address 对应的 socket，方便后续继续使用

        Channel::CallMethod
            更新 Controller 的参数、生成 bthread id，并锁定一个range，为后续 retry 做准备；
            SerializeRequestDefault：_serialize_request 对body部分，serialize、compressedData
            设置超时 HandleTimeout(bthread_id_t)，或者设置 backup request timer (执行HandleBackupRequest)
        Controller::IssueRPC
            根据socketid取出socket，并根据pool还是single找出 sending sock        
            PackRpcRequest：打包request，authentication、添加元数据信息到 meta，并Serialize meta；再meta、body写入到 io buf
                correlation_id 存储在pack的meta中，以便server回应能找到bthread、controller
            Socket::Write：同server send部分，注意：WriteRequest中包含了 bthread id
        对同步请求，在这里等待 bthread_id_join

    ## Recv Response
        Socket::ProcessEvent -> InputMessenger::OnNewMessages -> InputMessenger::CutInputMessage -> ParseRpcMessage -> ProcessInputMessage -> 
        ProcessRpcResponse -> ParsePbFromIOBuf：protobuf meta解析、ParseFromCompressedData：protbuf data 解析（没有压缩时，还是调用 ParsePbFromIOBuf）
            根据 bthread id 通过 bthread_id_lock，得到之前存储的 controller
            ControllerPrivateAccessor -> Controller::OnVersionedRPCReturned

        Controller::EndRPC
            删除定时器 bthread_timer_del、Controller::Call::OnComplete：清理 send sock
            清理线程：如果是异步，执行_done->Run()；同步的话，直接 bthread_about_to_quit、bthread_id_unlock_and_destroy，将唤醒 join bthread 的等待线程
                
    ## 对 bthread_id_t 的使用
        Channel::CallMethod：
            生成 bthread_id_t 保存 controller；id失败时，会调用 Controller::HandleSocketFailed
            bthread_id_lock_and_reset_range：为本次发送和重试，预留lock version；发送成功之前，将 id lock住
            bthread_id_unlock：request交给 socket 之后，释放lock

        ProcessRpcResponse：收到回应之后，通过 id 取回 controller

        Controller::OnVersionedRPCReturned：bthread_id_about_to_destroy：提前唤醒，告知准备 destroy 了

        Controller::EndRPC：bthread_id_unlock_and_destroy

        如果期间发生 error：
            HandleTimeout：设置在timer中，bthread_id_error(correlation_id, ERPCTIMEDOUT);
            HandleBackupRequest：backup request 失败
        
## RPCZ Time
    ## Server
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

    ## Client
    Channel::CallMethod
    set_start_send_us -- Requesting 

    Controller::IssueRPC: after cntl initialize，pack serialize、write（maybe just add to write queue）
    set_sent_us -- Requested(30) 

    ProcessRpcResponse：server response, get read event, but before read socket
    set_received_us -- Received response(17) of request[1]

    ProcessRpcResponse：after read socket、cut messsage、parse message；before deserialize message
    start_parse_real_us -- Processing the response in a new bthread

    Controller::EndRPC：before done->run
    start_callback_real_us -- Back to user's callsite