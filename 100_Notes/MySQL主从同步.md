---
tags:
    - 运维/MySQL
---

> [!tip] 利用主库（Master）产生的变更日志，在从库（Slave/Replica）上进行完整重放，从而实现数据的最终一致性
---
# 核心架构
> [!tip] 主从同步的完整运行依赖于 **==3 个核心线程==**（主库 1 个，从库 2 个）以及 **==2 种关键日志==**（Binlog 和 Relay Log）

## ==主库：Binlog Dump 线程== (二进制日志转储线程)

- **触发时机**：当从库连接到主库时，主库会为该从库专门创建一个 Binlog Dump 线程。
- **职责**：负责读取主库的 **==Binlog（二进制日志）==**，并将其通过网络发送给从库。如果主库当前没有数据更新，该线程会进入休眠状态，直到有新的 Binlog 产生将其唤醒。

## ==从库：I/O 线程== (I/O Thread)
- **职责**：负责连接主库，向主库请求指定日志文件和位置（Position）之后的 Binlog 内容。收到内容后，**顺序写入**到从库本地的 **==Relay Log（中继日志）==** 中。

## ==从库：SQL 线程== (SQL Thread)
- **职责**：负责读取 I/O 线程写好的 **Relay Log**，解析其中的内容，并在从库本地的存储引擎中**重放（Replay）**这些操作。通过这种“照葫芦画瓢”的方式，把从库的数据变得和主库一模一样。

---

# 流程
当在主库执行一条写操作（如 `INSERT`），同步流程如下
## 1 主库写日志
客户端提交事务，主库完成数据修改，并将数据变更顺序记入 **Binlog**，然后向客户端返回成功。
## 2 从库发请求
从库的 **I/O 线程** 通过配置好的主节点信息（IP、端口、账号、日志文件名及偏移量）连接主库，请求增量同步。
## 3 主库发日志
主库的 **Binlog Dump 线程** 响应请求，读取本地 Binlog，通过网络打包发送给从库。
## 4 从库记中继
从库的 **I/O 线程** 接收到 Binlog 数据，将其写入本地的 **Relay Log**（中继日志）文件末尾，并更新本地记录的同步位置。
## 5 从库重放
从库的 **SQL 线程** 检测到 Relay Log 有新内容写入，立即读取并解析，在从库本地执行相同的 SQL 或行变更。
## 6 位置更新
执行成功后，SQL 线程更新自己的读取位置，等待下一次同步。


---
# 同步模式
根据主库在提交事务时，**==是否需要等待从库的回应==**，主从同步分为以下三种模式：

|同步模式|工作原理|优点|缺点 / 风险|
|---|---|---|---|
|**异步复制** _(Async)_  <br>**(MySQL默认)**|主库写完本地 Binlog 后，**立刻向客户端返回成功**，不管从库有没有收到。|**性能最高**，主库完全不被同步逻辑阻塞。|**存在丢失数据风险**。若主库宕机且 Binlog 还没传到从库，此时强行将从库提升为主库，会丢失这部分数据。|
|**半同步复制** _(Semi-Sync)_  <br>**(生产环境常用)**|主库写完 Binlog 后，必须等待**至少一个从库**收到并将数据写入 Relay Log 并返回 ACK 确认，主库才向客户端返回成功。如果超时未回应，会自动降级为异步复制。|**安全性高**。能确保主库崩溃时，至少有一个从库有完整的数据备份。|**性能有损耗**，网络延迟（RTT）会直接增加主库事务的响应时间。|
|**全同步复制** _(Sync)_|主库写完数据后，必须等待**所有的从库**全部执行完该事务并返回确认，主库才向客户端返回成功。|绝对的数据强一致性。|**性能极差**，任何一个从库网络波动或卡死，都会导致主库无法写入。通常由官方的 MGR（MySQL Group Replication）集群来实现类似的高一致性。|

---

# Binlog格式
## Statement 模式：
==基于 SQL 语句的复制==。主库怎么执行，从库就跟着执行一模一样的 SQL。
- _缺点_：如果 SQL 包含特定函数（如 `UUID()`, `NOW()`），在从库执行算出的值会和主库不同，**导致主从数据不一致**。
## Row 模式 (推荐)：
==基于行变更的复制==。不记录 SQL 语句，而是记录“哪一行数据的哪几个字段改成了什么”。
- _优点_：绝对安全精准，不会因为函数导致数据不一致。
- _缺点_：日志量巨大。比如一个 `UPDATE` 改了 100 万行，Row 模式会产生 100 万条日志，而 Statement 只有 1 条 SQL。
## Mixed 模式：
==混合模式==。MySQL 自动判断，常规 SQL 用 Statement 节省空间，遇到可能引发不一致的函数时自动切换为 Row 模式。

---

# 配置
> [!tip] 让主库（Master）开启二进制日志（Binlog）并创建同步账号，让从库（Slave）指定主库的坐标（IP、端口、日志位置或 GTID），然后开启同步线程

---
## 1 实验环境假设

- **主库 (Master)**：IP 为 `192.168.1.100`，端口 `3306`
- **从库 (Slave)**：IP 为 `192.168.1.200`，端口 `3306`
- **MySQL 版本**：MySQL 8.0+（5.7 同样适用）

---

## 2 配置主库 (Master)

1. 修改主库配置文件

修改主库的 `my.cnf`（Linux 通常在 `/etc/my.cnf` 或 `/etc/mysql/my.cnf`），在 `[mysqld]` 标签下添加以下核心配置：

```ini
[mysqld]
# 1. 基础同步设置
server-id = 100                 # 主库的唯一标识ID，主从不能重复
log-bin = mysql-bin             # 开启 Binlog 并指定日志文件前缀
binlog_format = ROW             # 推荐使用 ROW 格式复制，最安全

# 2. 开启 GTID 模式（核心推荐）
gtid_mode = ON                  # 开启 GTID
enforce_gtid_consistency = ON   # 强制保证 GTID 一致性

# 3. 选填：指定需要同步的数据库（不填默认同步全库）
# binlog-do-db = test_db
```

2. 重启主库服务

```bash
sudo systemctl restart mysqld
```

3. 创建用于同步的专有账号

登录主库，创建一个专供从库 I/O 线程连接的账号，并赋予 `REPLICATION SLAVE` 权限：


```sql
-- 登录主库
mysql -u root -p

-- 创建用户（密码尽量复杂）
CREATE USER 'repl_user'@'192.168.1.200' IDENTIFIED WITH mysql_native_password BY 'ReplPassword@123';

-- 授权主从复制权限
GRANT REPLICATION SLAVE ON *.* TO 'repl_user'@'192.168.1.200';

-- 刷新权限
FLUSH PRIVILEGES;
```

---

## 3 配置从库 (Slave)

1. 修改从库配置文件

修改从库的 `my.cnf`，同样在 `[mysqld]` 标签下配置。**注意 `server-id` 不能与主库冲突。**

```ini
[mysqld]
# 1. 基础同步设置
server-id = 200                 # 从库的唯一标识ID，必须与主库不同
relay-log = slave-relay-log     # 开启中继日志

# 2. 开启 GTID 模式（必须与主库一致）
gtid_mode = ON
enforce_gtid_consistency = ON

# 3. 建议配置：从库只读（防止误写导致主从数据不一致）
read_only = ON
super_read_only = ON
```

2. 重启从库服务

```bash
sudo systemctl restart mysqld
```

---

## 4 确保初始数据一致（极其关键）

在建立同步连接前，**必须确保主库和从库的初始数据是完全一致的**，否则从库重放日志时会因为找不到数据或主键冲突而报错卡死。

- **如果都是新库**：直接跳过此步。
- **如果主库已有数据**：
    1. 在主库执行备份：`mysqldump -u root -p --all-databases --single-transaction --master-data=2 > backup.sql`
    2. 将 `backup.sql` 传输到从库。
    3. 在从库导入数据：`mysql -u root -p < backup.sql`

---

## 5 在从库开启同步

登录从库，执行 `CHANGE REPLICATION SOURCE TO` 指令（MySQL 8.0 之前为 `CHANGE MASTER TO`），指向主库：

```sql
-- 登录从库
mysql -u root -p

-- 配置主库信息（基于 GTID 模式）
CHANGE REPLICATION SOURCE TO
    SOURCE_HOST='192.168.1.100',
    SOURCE_PORT=3306,
    SOURCE_USER='repl_user',
    SOURCE_PASSWORD='ReplPassword@123',
    SOURCE_AUTO_POSITION=1; -- 1 表示使用 GTID 自动定位更新位置

-- 启动从库复制线程（I/O 线程和 SQL 线程）
START REPLICA; -- MySQL 8.0 语法（5.7 用 START SLAVE;）
```

---

## 6 验证同步状态

在从库上执行以下命令，观察主从状态：

```sql
SHOW REPLICA STATUS \G   -- MySQL 8.0 语法（5.7 用 SHOW SLAVE STATUS \G）
```

在打印出的长列表信息中，重点观察以下 **三个核心指标**：

1. **`Replica_IO_Running: Yes`**：表示从库的 I/O 线程已成功连接主库，正在接收 Binlog。
2. **`Replica_SQL_Running: Yes`**：表示从库的 SQL 线程正常，正在重放中继日志。
3. **`Seconds_Behind_Master: 0`**：表示从库落后主库的秒数。`0` 代表当前完全同步，没有延迟。
> [!warning] 如果前两个指标有任何一个是 `No`，说明同步失败。可以在输出中查看 `Last_IO_Error` 或 `Last_SQL_Error` 获取具体的报错原因（通常是网络不通、密码打错或数据冲突）