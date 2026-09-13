---
tags:
    - 运维/Redis
---

> [!note]
> - **主节点（master）负责写，从节点（slave）通过复制主节点的数据变更来保持同步**。从节点默认只读（`replica-read-only yes`），可以分担读请求。
> - **首次全量同步（RDB）+ 后续增量命令传播**，断线时通过 **backlog + offset** 尽量增量同步
> - **异步**，不保证强一致，只保证最终一致

# 建立复制关系
1. **配置文件**：`replicaof <master-ip> <master-port>`
2. **启动命令**：`redis-server --replicaof <master-ip> <master-port>`
3. **运行时命令**：`REPLICAOF <master-ip> <master-port>`（Redis 5.0 前是 `SLAVEOF`）

从节点执行后会连接主节点，发送 `PSYNC` 命令请求同步。

# 全量同步（第一次连接或无法增量时）
## 1. 从节点发送 PSYNC
```text
PSYNC ? -1
```

`?` 表示不知道主节点的 runid，`-1` 表示没有偏移量。这是首次连接的标志。

## 2. 主节点回复 FULLRESYNC

```text
+FULLRESYNC <runid> <offset>
```

告诉从节点自己的 runid 和当前复制偏移量。

## 3. 主节点执行 BGSAVE 生成 RDB  
主节点 fork 子进程生成 RDB 快照文件。**在生成期间的新写命令**会被存入 `replication buffer`（每个从节点一个）。

## 4. 发送 RDB 给从节点  
主节点把 RDB 文件通过网络发给从节点。从节点先清空自己的数据，再加载 RDB。

## 5. 发送缓冲区中的增量命令  
RDB 发送完后，主节点把 `replication buffer` 中积累的写命令发给从节点。从节点执行这些命令，追上主节点。

## 6. 进入命令传播阶段  
之后主节点每执行一条写命令，就异步发给所有从节点，保持持续同步。

# 增量同步（断线重连时）
