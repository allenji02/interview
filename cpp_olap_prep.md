# 可能问题的收集(基于C++后端/OLAP引擎方向)

## 网络和Socket模型

- OSI Layering

	- 设计哲学? 功能解耦, 专注每一层的问题

	- 物理层: 无线/线缆

		- 蓝牙和WLAN有什么区别?

		- 光缆和网线呢?

		- 分时复用还是分频复用?

	- 数据链路层: 局域网传输

		- MAC是什么? Media-Access-Control 以什么单位区分? 网卡 有什么功能? 区分局域网内的硬件

	- 网络层: 广域网传输

		- ARP协议什么工作? IP转MAC, 完成实际分发 反向呢? RARP

		- 路由协议? BGP/OSPF/EIGRP, 区别?(概念上, 收敛速度上, 管理难度上)

		- ICMP: 提供错误信息用于debug(什么错误? 比如TTL过期), ping和traceroute使用ICMP

	- 传输层: TCP/UDP

		- TCP

			- 三次握手的过程: 发起方SYN: 进入SYN_SEND状态, 服务端收到, 进入SYN_RCV状态, 回复SYN/ACK, 发起方收到, 进入Established状态, 再次回复ACK, 服务端进入Established状态, 结束

			- 为什么不能两次? 服务端无法确认客户端接收能力/自己能不能成功发包, 

			- 第2次握手传回了ACK，为什么还要传回SYN？服务端传回发送端所发送的 ACK 是为了告诉客户端：“我接收到的信息确实就是你所发送的信号了”，这表明从客户端到服务端的通信是正常的。回传 SYN 则是为了建立并确认从服务端到客户端的通信。

			- 第三次能不能客户端带数据? 一般不带, 但源码没有阻止这种可能: 对于非阻塞socket, connect发起后立即写缓存, 导致connect成功时, write_buffer内有数据, 可能会发生ACK带数据

			- 四次挥手: 发起方FIN, 进入FIN-WAIT-1, 等待服务端ACK, 进入FIN-WAIT-2, 服务端进入CLOSE-WAIT. 后续服务端可以继续发消息, 客户端不再新发. 直到服务端结束, 发起FIN, 进入LAST-ACK. 客户端收到后进入TIME-WAIT, 等待两个MSL后关闭, 服务端则是收到最后一个ACK后关闭.

			- 为什么四次? 半关闭特性

			- 为什么等两个MSL? 一旦ACK失败, 对方会在1MSL后重传FIN, 重传的FIN到达客户端又需要一个MSL.

			- 重传: 一旦在一个RTO后依然没有收到ACK, 就重传. 如何确定RTO? 基于RTT的加权平均和方差, 太大不容易发生重传, 等待时间过长, 太短频繁重传, 容易阻塞.

			- 快速重传: ACK携带到目前位置最大的连续有序Seq No + 1. 也有问题: 对于多个丢包, 只知道最小的一次丢包.

			- 选择重传: 基于快速重传, 但分段. 一次丢包直接重传所属区间, 同时SACK报告已收到的其他分段, 可以快速缩小区间.

			- D-SACK: 标记重复包

			- 流控: 滑动窗口. 起源: 一个一个传太慢, 一次传太多会拥塞, 沟通一个合适的宽度. 两个窗口: 发送窗口和接收窗口, 收到ACK时发送窗口前移, 维护两个游标, 指向未确认的第一个包和要发送的第一个包, 窗口内没有要发送的包就暂停, 设置定时器和等待ACK. 收到数据包时接收窗口前移. 那么如何确定窗口大小? 主要由接受方根据自身缓存大小在ACK包中回复

			- Congestion control: 为了最大化利用网络负载(不浪费但也不拥堵)

				- 慢启动:

				- 拥塞避免

				- 拥塞发生

				- 快速恢复

			- Nagle优化

			- 粘包和拆包: 上层可以自行约定数据delimeter, 在发生粘包时可以分割, 拆包时可以等待其他包. 固定包大小(不flexible). 自行标记数据body长度, 接收时收齐位置.

		- UDP

			- 和TCP什么区别? Header更小, 不可靠(少很多可靠机制) -> 所以也更快, 适用与实时游戏和音视频会议, 对丢包比较宽容, 但对延迟不宽容；UDP不是点对点, 适合大量分发.

			- 用UDP设计可靠协议? 加入缺少的那些机制 (乱序, 丢包: SeqNo. 重传/冗余)
	- 应用层:

		- HTTP/HTTPS

			- 状态码: 1xx提示, 2xx成功, 3xx重定向, 4xx客户端问题, 5xx服务端问题.

				- 常见代码? 200 OK, 301 永久重定向, 400 Bad Request, 403 禁止访问, 404 Not Found, 500 内部错误, 502 网关错误

				- HTTP不同分段: 有`\r\n`隔开, 解决了上述TCP粘包拆包问题

				- 常见字段: Host, Content-length, Connection Keep-Alive(保持TCP连接)

				- 请求方法: 

				- 缓存技术: 强制缓存: 由服务器在Response头中标记Cache-Control或Expires实现, 客户端会保持缓存至这个时间, Cache-control 选项更多一些，设置更加精细，所以建议使用 Cache-Control 来实现强缓存；协商缓存: 第二次请求起带资源tag, 交给服务器比较, 如果服务器认为不需要重传数据, 就响应304. 具体实现方式?

				- 版本特性: HTTP1.0没有长连接, HTTP1.1引入, 2.0, 3.0改用UDP

				- HTTPS: 在HTTP下加入TLS层, 非对称+对称混合加密 (RSA太慢, 直传AES不安全)

					- 中间人攻击: 证书和数字签名, 验证过程, 

		- DNS

			- 请求过程: Local DNS Server -> Root Server -> gTLD Server -> NameServer

			- 请求策略: 迭代 or 递归?

			- 常用端口? UDP53

			- linux命令: nslookup, dig, nscd restart

		- RPC

			- 和HTTP区别? 其实RPC也可以基于HTTP协议实现, 更多的是开发, 迭代和治理思维的变化: 传输效率: Thirft/Protobuf等都是二进制传输, 体积小效率高, 而非JSON; RPC框架一般自带服务注册/发现等治理模块, 和负载均衡等. 缺点: 需要预定义scheme等, 不适合对外暴露

- IO多路复用

		- select, poll(select + larger fd set), epoll(event poll: not scanning fd but register fd with call back)

		- 实际问题: 

- 网络统一是大端的: 最重要的bit最先出现. 转换: htonx, ntohx.


1. TCP全流程

- 3-way handshake

why not 2? For server to ensure no re-connection happens

- 4-way farewell

why not 3? Half-close, after client stops transmission, it can still accept msg from server, if server also wants to stop, 3-way is OK.

- 状态转换

- Congestion Control

Slow start

- Flow Control

Sliding Window

- Reliable Transportation

Sequence No., Retransmission


2. Socket建立链接流程

Server: create, bind, listen, accept

Client: create, bind, connect

3. Multiplexing方法



有什么实际问题?

4. UDP vs. TCP

依然是: TCP的可靠性由什么实现: 有序重传

这会带来很大的冗余metadata和握手确认过程, 造成性能损失

5. Web全链路

DNS解析, HTTP请求, Socket发起TCP连接, IP路由.

	- DNS相关: 服务器层级, 查询方式对比, 协议细节(头, 端口号等等)

	- HTTP和HTTPS区别, TLS双向建立流程, 类比SSH, 加密原理

	- 路由协议对比, (E,I-)BGP/OSPF, 收敛方式

	- ARP/ICMP细节

6. 高并发处理

服务侧缓存(Redis), 用户侧缓存(CDN), 短期:削峰(MQ), 长期:扩容(负载均衡+resharding+分库分表+定期落盘+服务注册/发现)


7. 问题收集

大小端

protobuf vs json

rpc vs http

RDMA


## 进程和线程: 原理,通信,同步,异常处理

1. 原理和概念:

Process: 操作系统管理大部分资源的最小单元: 地址空间
Thread: 轻量级的调度单元, 通过Share地址空间实现concurrency, 有局部栈, 但也有内核中的thread实现
Coroutine: Work on threads, 完全由协程库(用户态处理) (Pros&Cons: 减少内核切换, 但一旦Process不在运行所有coroutine都不能运行)

2. 通信:

进程: pipeline(有亲缘关系)和queue(无亲缘关系), 共享内存/变量, Signal, MQ, 网络(rpc/socket)

速度, 容量和expressiveness呢?

3. 同步:

锁(mutex, spin), 信号量(semaphore), condition variable, 原子操作

	- Deadlock and avoidance

	- S-X Lock: Also 缓存一致性

	- Optimistic and pessimistic lock

	- 一些规范

	- 生产/消费者

有什么性能问题? 安全问题?

4. 异常处理, 中断和信号

5. 线程池


## 分布式

- 时钟同步

- 下面的一致性问题

- 一致性快照

- 2阶段提交, 出口提交



## 内存管理,分区

1. 虚拟内存, 分区, MMU



2. 分页, 分段, 页表, 多级页表 TLB重填

性能问题? 

3. 内存对齐, 栈溢出


## 调度

进程调度算法/LRU: 优缺点? 进化过程?

上下文切换过程: 保存什么? 保存在哪? 如何回复? 有异常怎么办

## OS其他

特权机制: 优缺点?

	系统调用

文件系统

函数调用过程: Caller and callee-save, 栈帧, ABI

## 数据库: MySQL

1. 引擎

MyISAM, InnoDB和InMemory对比

2. 并发控制/一致性问题

	- 可能出现的问题: R-R, W-R, R-W, W-W

	- 隔离级别

	- 事务性是什么, 实现方式, MVCC

3. 索引

	- 分类

	- 最左前缀

	- Hash索引

4. 范式

    - 1,2,3,3.5,BC-NF

5. 优化

	- 索引优化



## 数据库: Redis


1. SQL和NoSQL, 或者说: RDB和NRDB

2. Hash问题

3. 缓存穿透

4. 跳表



## 一致性级别:

Causal, Linearizable, Sequential

## 设计模式

单例模式


Java的一些: 依赖注入, 控制反转, 切面编程

## Linux

命令相关

shell算术

grep

find

kill

ss

ip

ps

traceroute

cp, rn, mv


## C++:系统相关

`volatile`关键字

环境感知

内存分区

断点调试(gdb问题)

## C++:多线程

C++11线程库:

原子

异常处理中的线程等待

全局变量进行异常捕获

## C++:面向对象

- `final`关键字

- `explicit`构造

- `delete`构造删除

- `default`构造

- `virtual`方法

- `abstract`类

- `friend`

继承方式

泛型编程

内存布局(类的), 对齐, 大小

多态原理: 虚表和虚指针


## C++:语言特性

`noexcept`关键字

Iterator

Dangling指针和wild指针

vector原理, 扩容, 均摊复杂度

运算符重载

## C++:新特性

- L-Value vs. R-Value:

可以被取地址的是左值，表达式、不可取地址的是右值

- R-Value Reference

由`&&`标记，可以引用表达式的值，延长其在内存中的生命周期，避免重复拷贝。
由std::move创建, 原对象的值失效.

- 智能指针：

传统指针: 手动申请, 手动释放

`auto_ptr`: 使用copy constructor实现move, 随着move语义的引入而失效

`shared_ptr`和`unique_ptr`, `weak_ptr`

```cpp
shared_from_this()
```

- 类型转换

- lambda函数

- auto遍历器

- 函数类型推断

## 数值计算

## 编译原理

## 体系结构

## 数据结构和算法

Bitmap

快选

KMP

AC自动机

DP

## 分布式追踪:Dapper论文

采样类型

如何解决罕见因素

基于query

## 数据库优化: LSM树, B/B+树, 红黑树对比和原理

结构? 性能? 复杂度?

一切的根源都是性能:

快速范围查询, 总体查询复杂度, 同等复杂度的常数, 文件和内存读写次数...


## 分布式共识:Raft论文和其他原则

- 运作过程


- 容错能力


- 实现细节



## 大数据相关

1. Docker

	- 命令

	- 原理

	- 问题

2. Istio, k8s

	- 集群调度


	- 容错


	- 扩展


	- 问题

3. Hadoop生态

### OLAP相关


### 性能优化:机制和实现

### 容错:机制和实现

## 问的多的(SOTA的): RocksDB

## 问的多的(SOTA的): ClickHouse

## 问的多的(SOTA的): HBase/Hive

