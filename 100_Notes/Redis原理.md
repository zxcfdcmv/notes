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
---
# 数据结构
|类型|底层实现|特点|
|---|---|---|
|String|SDS（简单动态字符串）|记录长度、预分配、二进制安全|
|List|quicklist（ziplist + 双向链表）|兼顾内存与性能|
|Hash|ziplist / hashtable|小数据用 ziplist 省内存|
|Set|intset / hashtable|整数集合用 intset|
|ZSet|ziplist / skiplist + hashtable|跳表实现范围查询 O(logN)|

**关键设计**：
- **SDS**：O(1) 获取长度，避免缓冲区溢出，支持二进制数据
    
- **跳表（skiplist）**：多层索引，实现有序集合的高效范围查询
    
- **渐进式 rehash**：字典扩容时分散到多次操作，避免单次阻塞
---
# 内存管理
## **过期删除策略**（两种结合）
1. **惰性删除**：访问 key 时才检查是否过期
2. **定期删除**：每隔一段时间随机抽查部分 key 删除

## **内存淘汰策略**（8 种）
- `noeviction`：不淘汰，写入报错
- `allkeys-lru`：所有 key 中淘汰最近最少使用的
- `volatile-lru`：设置了过期时间的 key 中淘汰 LRU
- `allkeys-lfu` / `volatile-lfu`：淘汰最不经常使用的
---
# 持久化
## RDB（快照）
- 某一时刻把内存数据全量写入二进制文件
- `bgsave` 通过 fork 子进程完成，利用 **写时复制（COW）**
- 优点：文件小、恢复快；缺点：可能丢数据
## AOF（追加日志）
- 记录每条写命令，重启时重放
- 刷盘策略：`always` / `everysec` / `no`
- AOF 重写：压缩日志体积（也是 fork 子进程）
- 4.0 后支持 **混合持久化**：RDB 全量 + AOF 增量