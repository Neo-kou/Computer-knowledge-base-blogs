
# 1. 什么是异步IO?

首先, 同步IO, 比如我们网络编程会调用的recv, send等函数, recv和send函数都是同步的, 即调用recv就会立马去内核执行copy操作, 然后把copy的值返回, 是一气呵成的, 中间不存在断点。

异步IO, 可以理解为, 调用recv时, 读请求和数据返回并没有一气呵成的, 他是分开的, 调用recv后, 他立马返回了, 并没有直接去拷贝数据返回, 而是发起一个请求, 真正做这个数据拷贝和数据返回可能是内核自己异步做的, 不影响主流程

![[fb5629ca-f45f-4d64-abed-8feb3919eb1f.png]]

基本流程可能是调recv, 把任务放入任务队列(Submission Queue)就返回, worker自己去处理任务, 然后把结果放到结果队列(Completion Queue)中, 但是这样的结果有两个问题:
1. 频繁的copy task和result 怎么解? --> mmp映射? 两个queue的内存通过mmap同时映射到用户态和内核态, 不需要频繁进行两态切换
2. 队列的线程安全问题 怎么解? --> 环形无锁队列?
---

# 2. io_uring的三个系统调用

19年的Linux内核新增了三个io_uring的系统调用: 
- io_uring_setup: 构建两个无锁队列
- io_uring_enter: 赛完事件后一起推送到worker进行处理
- io_uring_register: 往队列中塞事件

基本流程: 
- `io_uring_setup()` 创建 uring 实例，获取 fd；
- `mmap()` 将 SQ、CQ、SQE 数组映射到用户态共享内存；
- 用户循环：
    1）填充 SQE，把 IO 任务塞进提交队列；
    2）调用`io_uring_enter()` 一次提交批量请求；
    3）轮询 CQ 读取 CQE，处理 IO 完成结果；
- 销毁资源、关闭 fd。

两大io_uring的高性能特性:

1. 批量提交，极少系统调用
**传统模型**：1 次 IO = 1 次 syscall（read/write），百万并发下切换开销爆炸；
**io_uring**：攒几十上百个 SQE，一次 io_uring_enter 提交；

2. 共享内存无锁通信，零拷贝交互
**SQ/CQ 是内核 / 用户共享内存**：
用户填充 SQE 只操作本地内存，无需拷贝请求数据给内核；
IO 完成结果 CQE 直接写共享内存，不用内核拷贝返回；

> squeue和cqueue里面的entry, 其实本质上共用的同一块内存, 把IO任务塞进squeue时, 是开辟这块内存的时机, worker处理完操作后, 写回数据也是写到开辟的这一块内存中, 然后清理cqueue里面的entry, 本质上就是把sq和cq里面的entry节点给清除, 中间不存在copy操作

---

# 3. liburing库编码

```c
#define EVENT_ACCEPT    0
#define EVENT_READ      1
#define EVENT_WRITE     2
// io_uring_sqe结构体中有一个字段是user_data, 这个数据是可以塞入到squeue后, 设置进去, 然后从cqueue拿出结果后重新拿出来的
// 用来作为从cqueue中拿出数据后, 判断是读事件还是写事件还是accept事件
struct conn_info {
    int fd;
    int event;
};
int init_server(unsigned short port) {  
    int sockfd = socket(AF_INET, SOCK_STREAM, 0);  
    struct sockaddr_in serveraddr;  
    memset(&serveraddr, 0, sizeof(struct sockaddr_in));
    serveraddr.sin_family = AF_INET;    
    serveraddr.sin_addr.s_addr = htonl(INADDR_ANY);
    serveraddr.sin_port = htons(port);  
    if (-1 == bind(sockfd, (struct sockaddr*)&serveraddr, sizeof(struct sockaddr))) {      
        perror("bind");    
        return -1;  
    }  
    listen(sockfd, 10);
    return sockfd;
}
#define ENTRIES_LENGTH      1024
#define BUFFER_LENGTH       1024
int set_event_recv(struct io_uring *ring, int sockfd,
                      void *buf, size_t len, int flags) {
    struct io_uring_sqe *sqe = io_uring_get_sqe(ring);
    struct conn_info accept_info = {
        .fd = sockfd,
        .event = EVENT_READ,
    };
    io_uring_prep_recv(sqe, sockfd, buf, len, flags);
    memcpy(&sqe->user_data, &accept_info, sizeof(struct conn_info));
}

int set_event_send(struct io_uring *ring, int sockfd,
                      void *buf, size_t len, int flags) {
    struct io_uring_sqe *sqe = io_uring_get_sqe(ring);
    struct conn_info accept_info = {
        .fd = sockfd,
        .event = EVENT_WRITE,
    };
    io_uring_prep_send(sqe, sockfd, buf, len, flags);
    memcpy(&sqe->user_data, &accept_info, sizeof(struct conn_info));
}

int set_event_accept(struct io_uring *ring, int sockfd, struct sockaddr *addr,
                    socklen_t *addrlen, int flags) {
    // 作用：从共享内存的 SQE 数组里拿一个未使用的 SQE 条目，用来填充 IO 指令（read/write/connect 等参数）
    // 此时只是拿到了一个空白的内存, 内存里确实多了一块你即将填写的空间，但队列逻辑上没有新增任务, 因为队列的tail没变
    struct io_uring_sqe *sqe = io_uring_get_sqe(ring);
    struct conn_info accept_info = {
        .fd = sockfd,
        .event = EVENT_ACCEPT,
    };
    // 作用只是把 accept 需要的参数：fd、addr、addrlen、opcode 写到你拿到的那块 SQE 内存里, 本质上是结构体赋值
    io_uring_prep_accept(sqe, sockfd, (struct sockaddr*)addr, addrlen, flags);
    memcpy(&sqe->user_data, &accept_info, sizeof(struct conn_info));
}

int main(int argc, char *argv[]) {
    unsigned short port = 9999;
    int sockfd = init_server(port);
    struct io_uring_params params;
    memset(&params, 0, sizeof(params));
    struct io_uring ring;
    // 构建两个队列, io_uring_setup在这里面
    io_uring_queue_init_params(ENTRIES_LENGTH, &ring, &params);
#if 0
    struct sockaddr_in clientaddr;  
    socklen_t len = sizeof(clientaddr);
    accept(sockfd, (struct sockaddr*)&clientaddr, &len);
#else
    struct sockaddr_in clientaddr;  
    socklen_t len = sizeof(clientaddr);
    set_event_accept(&ring, sockfd, (struct sockaddr*)&clientaddr, &len, 0);
#endif
    char buffer[BUFFER_LENGTH] = {0};
    while (1) {
        // 告诉内核「本次有多少个新 SQE 等待处理」, io_uring_enter在这里面, 会移动tail指针
        io_uring_submit(&ring);
        // 代码会阻塞在这里, 等待cqueue里面有结果后才会返回, 等待并收割完成事件
        struct io_uring_cqe *cqe;
        io_uring_wait_cqe(&ring, &cqe);
        struct io_uring_cqe *cqes[128];
        int nready = io_uring_peek_batch_cqe(&ring, cqes, 128);  // epoll_wait
        int i = 0;
        for (i = 0;i < nready;i ++) {
            struct io_uring_cqe *entries = cqes[i];
            struct conn_info result;
            memcpy(&result, &entries->user_data, sizeof(struct conn_info));
            if (result.event == EVENT_ACCEPT) {
                set_event_accept(&ring, sockfd, (struct sockaddr*)&clientaddr, &len, 0);
                //printf("set_event_accept\n"); //
                int connfd = entries->res;
                set_event_recv(&ring, connfd, buffer, BUFFER_LENGTH, 0)
            } else if (result.event == EVENT_READ) {  //
                int ret = entries->res;
                //printf("set_event_recv ret: %d, %s\n", ret, buffer); //
                if (ret == 0) {
                    close(result.fd);
                } else if (ret > 0) {
                    set_event_send(&ring, result.fd, buffer, ret, 0);
                }
            }  else if (result.event == EVENT_WRITE) {
                int ret = entries->res;
                //printf("set_event_send ret: %d, %s\n", ret, buffer);
                set_event_recv(&ring, result.fd, buffer, BUFFER_LENGTH, 0);
            }
        }
        // 这里需要把cqueue中已经处理过的数据清空
        io_uring_cq_advance(&ring, nready);
    }
}
```

io_uring中和epoll类似, 也有三种事件状态, 分别是: EVENT_ACCEPT, EVENT_RECV, EVENT_WRITE, 下面来讲一下epoll和io_uring的区别:

- epoll事件就绪时, 代表某个fd可读可写, 需要程序员手动的去recv/send, 每一次就绪都是一次system call
- io_uring事件就绪时, 代表某个fd的recv或send操作内核已经帮你已经做完了, system call的次数可以忽略不计
- epoll就是告诉内核你要关心哪些fd的哪些事件, 事件就绪后由程序员自己去处理
- io_uring就是把要做的事情告诉内核, 内核在后台做完IO后给你结果

其实在宏观上来看, epoll和io_uring的区别, 就类似于reactor和proactor之间的区别

epoll和io_uring的性能测试:
1. qps
2. 并发连接数量
3. 并发连接数量到底的时间, 建连

---
# 4. 网络常见面试题解析

1. <font color = purple>UDP如何做到并发?</font>
TCP是面向连接的，一个连接对应一个Socket文件描述符（FD）；而UDP是无连接的，一个UDP Socket可以同时接收来自任何客户端的数据包。因此，UDP的“并发”其实是指如何同时处理多个客户端发来的数据包

<font color = purple>方案一: 多端口并发</font>
![[38d67328-972b-4314-bd1d-523f905a79e9.png]]
**原理**：服务端开启一组端口（例如 8000 - 8100），客户端连接时，先连接一个主端口（如8000）进行握手，服务端分配一个空闲的工作端口（如8005）给客户端，客户端后续都往8005发数据, 一个端口可以处理多个连接, 假设开通1000个端口, 一个端口处理2000个连接, 并发量还是很高的, 并且UDP不会存在粘包的问题, 假如一个UDP的包1000字节, 我读了500字节, 那么剩余的500字节会被直接丢弃

<font color = purple>方案二: 单端口 + epoll</font>
**原理**: 服务端只监听一个端口, 上层使用epoll监听udpfd, 一旦udpfd有事件到来, 直接调用recv把整个udp的包拿出来, 然后丢到线程池做处理, 处理完直接send。这里需要注意的是, 使用epoll时要用非阻塞的recv读取, 每一次epoll_wait后, 要用while(1)把所有数据都拿出来, 因为一次udpfd就绪后, 不一定只有一个udp的包到来, 用while(1)把所有的UDP包都拿出来, 并且这里压根不需要关心UDP的包的完整性和粘包问题, 因为一次recv就会拿到UDP一个包的所有数据(UDP的数据不是流式的)


<font color = purple>方案三: 单端口 + `SO_REUSEPORT`</font>
**原理**：多个进程或线程**绑定同一个UDP端口**。内核层面会自动将到达该端口的数据包负载均衡（哈希分发）给这些进程/线程
**处理流程**：启动多个Worker进程，每个进程都创建Socket绑定到同一个UDP端口，并在自己的事件循环中调用 `recvfrom`。内核会保证一个数据包只会被一个进程唤醒并读取，避免了“惊群效应”

---
2. <font color = purple>TCP和UDP的区别</font>:
- **TCP是基于连接的, UDP是基于数据包的**
TCP的包是有顺序的, 先发的先收, UDP的包是没有顺序的

TCP会有粘包问题, UDP的包每一次recv时, 就是拿的一个数据包, 不会存在粘包问题(并且一次recv没拿完的数据会被丢弃), TCP的粘包问题有两种方法可以解决: 

一: 应用层的数据头部, 前两个字节定义TCP包的长度, recv时先读前两个字节, 把本次TCP包的长度读出来, 然后再recv读取特定的长度(基于顺序的, 第一次recv一定是读到的长度)

二: 应用层做分隔符, 假如在一个TCP包发完后, 在最后加上/r/n/r/n, 然后在recv时通过这个分割符做数据包的切分

UDP的包是无序的, 要想应用层做到有序的解析UDP的包, 可以在应用层为每一个UDP包定义序号, 然后再把TCP的可靠传输机制拿过来即可

- **TCP和UDP做网络并发的方案是不一样的**
TCP做并发的思路是做资源隔离, 因为一个客户端对应一个clientfd
UDP做并发的思路就很多了, 一种是做多端口并发, 一种是reuseport让多个线程绑定同一个UDP端口

- TCP和UDP的使用场景不一样
直播, 下载, moba实时战斗类游戏都会用UDP, UDP的实时性比较强, 丢包后会直接丢弃旧帧, TCP的话有一系列机制, 会阻塞后面的所有数据, 等待重传。

---

