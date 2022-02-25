## Recv

    消息解析读取:
    DISP把所在的pthread让给了新建的bthread，使其有更好的cache locality，可以尽快地读取fd上的数据。而EDISP所在的bthread会被偷到另外一个pthread继续执行

## Send
    wait-free MPSC

## Request
    

    Socket::StartInputEvent

    InputMessenger

