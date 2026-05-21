# 1. select/poll/epoll的再理解
select比起epoll, 虽然说感官上哪哪儿都不如epoll, 但是在少量连接, 并且这些连接大多数处于活跃状态时, select的效率是会比epoll高的, 比如说一个分布式微服务的各个微服务内部通信时

![[select和epoll的选择.drawio]]

![[为什么select在这种场景比epoll快.png]]

---

# 2. epoll的两种模式再理解
之前写的博客, 以及我自己的认知是, ET模式是一定比LT模式高效的, 因为用户态和内核态切换变少了, 并且宏观上网络IO变快了。

但是在游戏, redis等场景下, LT模式是要比ET模式好的, 因为这些场景是只有有数据就赶紧拿出来处理后先给客户端发了再说, 如果用ET模式全部读取完, 在这种场景下反而是不对的
![[为什么redis要用lt模式.png]]
而在nginx下, 用et又是很好的选择, 因为nginx是做纯转发、纯 IO 密集、无复杂业务逻辑, 也不需要读取网络中发的数据是啥, 并且网络连接暴多, 这种场景用et, 一次性拿完数据做转发, 系统触发次数少, 效率更高, 并且给真正的服务发送的数据也是比较完整的
![[为什么nginx要用et模式.png]]

> 总结为两句话就是:

- **业务重、实时性强、单条处理、低并发**（Redis、游戏服、IM 逻辑服）-> LT

- **IO 密集、纯转发代理、可批量处理、海量长连接**（Nginx、网关、LB）-> ET
---

# 3. 多路IO的再理解
多路复用IO本质上就是可以在一个线程内, 同时等待多个文件描述符(也可以叫做fd事件)是否就绪

IO = 就绪 + 处理

对于accept函数来说, 当客户端三次握手后, 全连接队列里面有数据时, accept函数即就绪

对于recv函数来说, 当接受缓冲区有数据时, 即就绪

对于send函数来说, 只要发送缓冲区没有满, 那么就是就绪的

多路IO就是可以在单线程, 同时监听多个fd的就绪事件

---

# 4. 对P2P的理解
P2P本质上就是, 两个端直接通信, 没有客户端和服务器之分, A端直接连接B端
这里有一篇文章我感觉讲的不错([P2P通信原理与实现（C语言） - 知乎](https://zhuanlan.zhihu.com/p/455475899))
一般情况下, 进行P2P通信时, 需要一个中间人服务器, 让AB双端拿到对方的地址等信息, 然后AB端再进行P2P通信, 但是如果是直接在你自己的机器测验, 地址啥的都知道, 就不需要这么麻烦了

下面是P2P的代码, AB端用同一份代码([2.1.1-network-io/tcpclient.c · 杭电码农-NEO/Linux cloud_server code_repository - 码云 - 开源中国](https://gitee.com/NEO_kou/linux-cloud_server-code_repository/blob/master/2.1.1-network-io/tcpclient.c))

这份代码很有意思, 没有listen函数, 也没有accept函数, 这份代码并不是走的普通的TCP三次握手, 可以注意到这里是用while(1)一直在进行connect
![[connect_p2p.png]]
当我们A端运行时, 会一直卡在这里, 因为B端是没有启动进程的, 对应的端口是没有进行bind的, 收益A端会一直connect, 一旦B端启动, 也会connectA端, 此时AB端是TCP 同时打开（Simultaneous Open）, 这是 TCP 协议中一个真实存在的特性（RFC 793 定义）。当两端**同时**主动向对方发起 connect 时，不需要 listen/accept 也能建立连接
![[TCP同时打开流程.png]]