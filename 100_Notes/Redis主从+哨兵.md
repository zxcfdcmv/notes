---
tags:
    - 运维/Redis
---

> [!note]
> - **主节点（master）负责写，从节点（slave）通过复制主节点的数据变更来保持同步**。从节点默认只读（`replica-read-only yes`），可以分担读请求。
> - **首次全量同步（RDB）+ 后续增量命令传播**，断线时通过 **backlog + offset** 尽量增量同步
> - **异步**，不保证强一致，只保证最终一致
---
# 建立复制关系
1. **配置文件**：`replicaof <master-ip> <master-port>`
2. **启动命令**：`redis-server --replicaof <master-ip> <master-port>`
3. **运行时命令**：`REPLICAOF <master-ip> <master-port>`（Redis 5.0 前是 `SLAVEOF`）

从节点执行后会连接主节点，发送 `PSYNC` 命令请求同步。

---
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

---
# 增量同步（断线重连时）
如果从节点只是短暂断线，重新连接时不想再全量同步，就用增量同步。

> [!tip] 前提条件
> - 主节点的 **replication backlog**（复制积压缓冲区）中还保留着从节点缺失的那部分数据
> - 从节点提供的 runid 和 offset 匹配


**流程**：

```text
从节点: PSYNC <runid> <offset>
主节点: +CONTINUE
```

主节点从 backlog 中找到 offset 之后的数据，发给从节点。

**replication backlog 是什么？**
- 一个**固定大小的环形缓冲区**（默认 1MB，`repl-backlog-size` 可配）
- 主节点在命令传播阶段，除了发给从节点，还会写入 backlog
- 它是**所有从节点共享**的
- 如果从节点断线太久，缺失的数据已被覆盖，就只能全量同步

**backlog 大小怎么估算？**

```text
backlog = 平均写入速率 × 最长断线时间
```

比如每秒写 1MB，允许断线 60 秒，就该配 60MB。

---
# 命令传播阶段
进入稳定状态后，主节点每执行一条写命令，会：

1. 写入自己的内存
2. 追加到 AOF（如果开启）
3. 发送给所有从节点
4. 写入 replication backlog

从节点收到命令后执行，并**异步**向主节点回复偏移量（`REPLCONF ACK <offset>`）。主节点通过这个 ACK 知道从节点的同步进度。

**心跳机制**：

- 主节点每秒向从节点发 `PING`
- 从节点每秒向主节点发 `REPLCONF ACK <offset>`

**为什么从节点是异步复制？**  
主节点不会等从节点确认就返回给客户端，所以主从之间是**异步**的。这意味着主节点宕机时，可能有一部分数据还没同步到从节点，导致**数据丢失**。这是 Redis 主从复制的固有特性。

---
# 复制偏移量与 runid

## 复制偏移量（offset）

- 主节点和从节点各自维护一个 offset
- 主节点每发送 N 字节数据，offset 增加 N
- 从节点每接收 N 字节，offset 增加 N
- 通过比较 offset 判断是否同步

## runid

- 每个 Redis 实例启动时生成的 40 位随机字符串
- 从节点保存主节点的 runid
- 如果主节点重启，runid 会变（除非用 `debug reload`），从节点发现 runid 不匹配就全量同步

---
# 全量同步 vs 增量同步对比

|维度|全量同步|增量同步|
|---|---|---|
|触发时机|首次连接、runid 不匹配、backlog 覆盖|短暂断线重连|
|数据传输|RDB 全量文件|backlog 中的增量命令|
|开销|大（fork + 网络 + 从节点清空重载）|小|
|命令|`PSYNC ? -1`|`PSYNC <runid> <offset>`|
|主节点回复|`+FULLRESYNC`|`+CONTINUE`|