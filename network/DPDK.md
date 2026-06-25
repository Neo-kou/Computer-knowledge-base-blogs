# 1. DPDK是什么

**DPDK（Data Plane Development Kit，数据平面开发套件）**：一套开源高性能用户态数据包处理开发库 + 网卡驱动，**绕开 Linux 内核协议栈**，实现百万 PPS~ 千万 PPS 线速收发包、微秒级低延迟，多用于网关、防火墙、OVS 加速、流量采集、负载均衡等高性能网络场景。

> **网卡绑定固定 CPU 线程**正是 DPDK 最标志性的设计之一，但细节是：**网卡硬件队列绑定 CPU 逻辑核 (lcore)，不是整张网卡绑一个线程**。

![[5d961a320364c3884fb3cc7a2378ad99~tplv-be4g95zd3a-image.jpeg]]

---
# 2. 为什么需要DPDK

==普通 Linux 内核收包流程：==
- 网卡收到数据包→**硬件中断 CPU**→切内核态→内核拷贝数据包到内核缓冲区→内核协议栈解析→再拷贝到用户态应用
- 三大损耗：**硬件中断开销、内核 <-> 用户态两次内存拷贝、CPU 上下文切换**，千兆网卡跑满尚可，10G/25G 网卡很难跑满线速，大包小包混杂 CPU 极易被中断打满。

==DPDK 核心四大黑科技:==
- **用户态 PMD 轮询驱动:** PMD=Poll Mode Driver（轮询驱动），**抛弃硬件中断，CPU 线程主动循环轮询网卡硬件队列收包**；网卡被 DPDK 绑定后脱离内核驱动接管，数据包直接放到**用户态大页内存**，**零系统调用、零内核拷贝**
- **CPU 亲和绑核:**  a. DPDK 把 CPU 物理核叫`lcore`（逻辑核）；b. 一个 CPU 核固定绑定网卡 1 个 RX 接收队列 + 1 个 TX 发送队列，这个核只跑收发包轮询线程，不被操作系统调度、不跑其他进程；c. 现代网卡有多硬件队列（比如 8 队列 / 16 队列），配合 RSS 分流，多 CPU 核并行处理同一张网卡流量。
- **大页 HugePage 内存:** 放弃 4K 小页，使用 2M/1G 巨型页，减少 CPU 页表缺页、提升 Cache 命中率，DPDK 所有数据包缓冲区（mbuf）都在大页上分配

> 纠正误区：**不是整张网卡绑单个线程**：单网卡多队列→多 CPU 核分工，一个核只管一个队列的数据。

---
# 3. 网络适配器模式

![[3583d45b-4638-4909-b1f2-944ae2987a5c.png]]
桥接模式代表, 你的虚拟机的和你的Windows主机的网卡是在同一层的, 是平级的, 有可能你的Windows主机地址是192.168.0.123, 你的虚拟机地址可能是192.168.0.125, 是在同一个局域网
![[ea0b9a84-f208-46fb-9578-a258b884a4bd.png]]

NAT模式: 是在Windows主机上面, 开辟了一个子网, 这个虚拟机是跑在Windows主机给的虚拟的路由器的子网上面的
![[9b419b7d-a55a-4635-9284-5f5cc687ec63.png]]

only host模式: 虚拟机只能和你的Windows主机连接, 不能被外部连接, 也不能上网访问百度

---
# 4. 网卡和sk_buff

![[522293d7-89b7-4e0e-a739-5f5ba20539f0.png]]
**网卡核心作用是实现设备和网络之间的数据收发**，光电 / 电信号与二进制比特（0/1）的转换是网卡的核心工作之一, 普通场景：网卡 → 内核协议栈 → 应用, DPDK 场景：DPDK 接管网卡驱动，**绕开内核**，网卡数据直接送入用户态内存，配合绑核、轮询收包，做到高性能


**`sk_buff`（socket buffer，套接字缓冲区）是 Linux 内核网络栈的核心数据结构**，用来**管理一个完整的网络数据包**，不是单纯标记网卡地址，而是贯穿**网卡接收 → 内核协议栈解析 → 应用层**全流程的数据包载体, Linux 内核 TCP/IP 协议栈**全程基于 sk_buff 处理报文**, 基本流程如下:

>1. 网卡收到电 / 光信号，转成二进制以太网帧，存入内核内存
1. **网卡驱动** 申请 `sk_buff`，让 sk_buff 指向这片报文内存
2. 网卡触发中断，CPU 进入内核，开始处理这个 sk_buff
3. 二层（以太网）：校验 MAC、丢弃非法帧，剥掉二层头
4. 三层（IP）：解析 IP 头、路由、分片重组
5. 四层（TCP/UDP）：解析端口、序号、重传、拥塞控制
6. 协议栈处理完毕 → 数据通过 Socket 交给应用程序

当网络中有大量请求时, 内核中也有大量的sk_buff, 此时内核用链表来组织这些sk_buff
```c
struct sk_buff { 
	// ... 报文数据、协议头、长度等字段 ... 
	struct sk_buff *next;  // 下一个 sk_buff 
	struct sk_buff *prev;  // 上一个 sk_buff // ... };
}
```

- `recv()`：把 **sk_buff 里的报文数据** 拷贝到**应用层缓冲区**；
- `send()`：把 **应用层缓冲区数据** 拷贝到 **sk_buff**，再交由内核网络栈发送

![[Pasted image 20260607102954.png]]

DPDK本质上就是使用uio或vfio从网卡截断数据, 然后再使用kni把数据传给tcp/ip层协议栈, 再加上内部的巨页设置

---
# 5. 关键问题

1. dpdk能不能提升应用层redis的qps -- 不明显
2. dpdk能不能减少应用层nginx的延迟 -- 不明显
3. dpdk能不能提升网卡的吞吐量 -- 比较明显

==DPDK使用场景:==

- 数据传输
- 网关防火墙

DPDK 主要用于**超过 10Gbps 吞吐量**或**微秒级延迟要求**的高性能网络场景，是电信、金融、云计算、网络安全等领域的核心技术

> DPDK中有一个m_buff的定义, 这个mbuff可以理解为sk_buff, 网卡里面来了数据之后, 会直接放入这个mbuff, 把这个mbuff比作一个水槽, 网络数据比作洪水, 洪水来了后是直接放进水槽的, 然后使用DPDK进行rte_eth_rx_burst函数时, 是去接收网卡中的数据的, 这个函数是不涉及到拷贝操作的, 可以理解为你要用水, 就直接操作水槽里面的水了, 而不是把水槽里面的水拿一份出来放在你家的水缸, 然后你再去操作水缸, 这里是直接操作mbuff中的数据了


DPDK没有走内核, 是直接截断的数据包, 所以说只要编码没有限制端口, 那么所有端口的数据都可以接受到, 整个网络协议包是一个套娃的过程, 首先是以太网协议, 里面包含IP协议, 里面再包含UDP协议, 下面是DPDK回包时要进行的以太网协议数据填充
![[4e5f9ed6-43e5-458b-86a3-6aa77c15c605.png]]

接着往下是IP协议的数据填充:
![[5e583c06-88dc-4712-853f-5756f07697e6.png]]![[Pasted image 20260612222855.png]]
![[c3e08f10-80ce-44b9-813e-2942e4d59930.png]]
下面是UDP的数据填充:
![[fb7f6af1-c8bc-40a7-87fd-e9f257035c56.png]]

---

# 6. DPDK应用层使用原理
先看一下实现原理:

![[e265cad7-ed04-43f1-bc38-e3985e8534d5.png]]

```c
// rx
struct rte_mbuf *rx[BURST_SIZE];

unsigned num_recvd = rte_eth_rx_burst(gDpdkPortId, 0, rx, BURST_SIZE);

if (num_recvd > BURST_SIZE) {
    rte_exit(EXIT_FAILURE, "Error receiving from eth\n");
} else if (num_recvd > 0) {
    unsigned i = 0;
    for (i = 0;i < num_recvd;i ++) {
        ddos_detect(rx[i]);
    }
	rte_ring_sp_enqueue_burst(ring->in, (void**)rx, num_recvd, NULL);
}
// tx
struct rte_mbuf *tx[BURST_SIZE];
unsigned nb_tx = rte_ring_sc_dequeue_burst(ring->out, (void**)tx, BURST_SIZE, NULL);
if (nb_tx > 0) {
    rte_eth_tx_burst(gDpdkPortId, 0, tx, nb_tx);
    unsigned i = 0;
    for (i = 0;i < nb_tx;i ++) {
        rte_pktmbuf_free(tx[i]);
    }
}
```

一共四个线程, main线程不断循环(while1), 从网卡中读取数据放入in queue, 然后从out queue中取出数据再写到网卡, pkg_process这个线程, 用于做UDP/TCP/IP协议解析, 解析出来的数据放入recv_buffer, 然后tcp_entry线程读recv_buffer, tcp_entry线程处理完后写到send_buffer, 然后pkg_process线程再从send_buffer中拿数据进行解析, 解析后又把数据放回ring_out_queue中等待main函数去拿数据写到网卡
![[b55dcefb-93b5-4948-adc9-50a4d194051f.png]]
```c
static int pkt_process(void *arg) {
    struct rte_mempool *mbuf_pool = (struct rte_mempool *)arg;
    struct inout_ring *ring = ringInstance();
    while (1) {
        struct rte_mbuf *mbufs[BURST_SIZE];
        unsigned num_recvd = rte_ring_mc_dequeue_burst(ring->in, (void**)mbufs, BURST_SIZE, NULL);
        unsigned i = 0;
        for (i = 0;i < num_recvd;i ++) {
            struct rte_ether_hdr *ehdr = rte_pktmbuf_mtod(mbufs[i], struct rte_ether_hdr*);
            if (ehdr->ether_type == rte_cpu_to_be_16(RTE_ETHER_TYPE_IPV4)) {
                struct rte_ipv4_hdr *iphdr =  rte_pktmbuf_mtod_offset(mbufs[i], struct rte_ipv4_hdr *,
                sizeof(struct rte_ether_hdr));  
                ng_arp_entry_insert(iphdr->src_addr, ehdr->s_addr.addr_bytes);  
                if (iphdr->next_proto_id == IPPROTO_UDP) {
                    udp_process(mbufs[i]);
                } else if (iphdr->next_proto_id == IPPROTO_TCP) {
                    ng_tcp_process(mbufs[i]);
                } else {
                    rte_kni_tx_burst(global_kni, mbufs, num_recvd);
                }
            } else {
                rte_kni_tx_burst(global_kni, mbufs, num_recvd);
            }
        }
        rte_kni_handle_request(global_kni);
#if ENABLE_UDP_APP
        udp_out(mbuf_pool);
#endif
#if ENABLE_TCP_APP
        ng_tcp_out(mbuf_pool);
#endif
    }
    return 0;
}
```

> 这里的recv_buffer和send_buffer都是指的一个TCP连接的buffer, 而不是所有TCP连接共用的buffer
![[ec169de4-2547-48a9-9f08-8423cc67c2ed.png]]

---

# 7. DPDK实现并发

1. 一请求一线程: 效率低
2. 使用epoll: 可以, 但是不能用内核自带的epoll, 需要自己来写

epoll的原理:
![[12679933-87eb-4384-b9ec-39c725537442.png]]

整个epoll的数据结构有一颗红黑树和一个就绪队列, 最后是长这样的:
![[80d2968c-b76a-4ba4-b369-f3843723f797.png]]
即是红黑树的节点, 又是就绪队列的节点, 红黑树是整集, 就绪队列是就绪集

epoll一共有四个关键函数, 三个对外: epoll_create epoll_ctl epoll_wiat, 一个对内: epoll_event_callback
![[3d3c929e-4eb0-4494-9080-0dc0350f32fd.png]]

三次握手的最后一次, 会调epoll_event_callback
接受到网络数据时, 会调epoll_event_callback
epoll_event_callback会把对应的fd添加到就绪队列