---
tags:
    - 运维/Redis
---

> [!note]
> 内存键值数据库
---
# 架构
> [!tip]
> 命令单线程 + IO多路复用

- **单线程处理命令**：避免多线程锁竞争、上下文切换开销，保证命令的原子性
    
- **I/O 多路复用**：用 epoll（Linux）/ kqueue 等同时监听大量连接，单线程也能支撑高并发
    
- **6.0 之后引入多线程 I/O**：只把网络读写（socket read/write）并行化，命令执行仍是单线程
```text
客户端 → 多路复用器(epoll) → 事件分发 → 单线程命令处理 → 内存数据结构
```
# 数据结构
|类型|底层实现|特点|
|---|---|---|
|String|SDS（简单动态字符串）|记录长度、预分配、二进制安全|
|List|quicklist（ziplist + 双向链表）|兼顾内存与性能|
|Hash|ziplist / hashtable|小数据用 ziplist 省内存|
|Set|intset / hashtable|整数集合用 intset|
|ZSet|ziplist / skiplist + hashtable|跳表实现范围查询 O(logN)|