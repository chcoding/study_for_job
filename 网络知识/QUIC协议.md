# QUIC

## 什么是QUIC

QUIC（Quick UDP Internet Connections）是一种实验性传输层网络协议，提供与TLS/SSL相当的安全性，同时具有**更低的连接和传输延迟**。QUIC基于UDP，因此拥有极佳的弱网性能，在丢包和网络延迟严重的情况下仍可提供可用的服务。QUIC在**应用程序层面就能实现不同的拥塞控制算法**，不需要操作系统和内核支持，这相比于传统的TCP协议，拥有了更好的改造灵活性，非常适合在TCP协议优化遇到瓶颈的业务。

## QUIC优点

- 更短的连接延时，最快可以实现0RTT
- 基于UDP，没有队头阻塞的概念
- 应用层实现拥塞控制，对应用的适配性更好
- 连接迁移



### QUIC流控逻辑

QUIC 实现流量控制的方式：

- 通过 window_update 帧告诉对端自己可以接收的字节数，这样发送方就不会发送超过这个数量的数据。
- 通过 BlockFrame 告诉对端由于流量控制被阻塞了，无法发送数据。



**QUIC 的 每个 Stream 都有各自的滑动窗口，不同 Stream 互相独立，队头的 Stream A 被阻塞后，不妨碍 StreamB、C的读取**。

QUIC 实现了两种级别的流量控制，分别为 Stream 和 Connection 两种级别：

- **Stream 级别的流量控制**：Stream 可以认为就是一条 HTTP 请求，每个 Stream 都有独立的滑动窗口，所以每个 Stream 都可以做流量控制，防止单个 Stream 消耗连接（Connection）的全部接收缓冲。
- **Connection 流量控制**：限制连接中所有 Stream 相加起来的总字节数，防止发送方超过连接的缓冲容量。

