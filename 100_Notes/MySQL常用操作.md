---
tags:
    - 运维/MySQL
---

---
# 命令行
---
## 启动/停止
- **后台标准启动**：
    
    ```sh
    sudo mysqld_safe &
    ```
    
    _加上 `&` 符号是为了让该进程在后台运行，不占用当前的终端窗口。_
- **指定配置文件启动**（如果你的配置文件不在默认路径）：
    
    ```sh
    sudo mysqld_safe --defaults-file=/etc/my.cnf &
    ```
    
- **安全停止服务**：  
    使用 `mysqld_safe` 启动后，不能直接去 `kill` 它（因为安全机制会误以为它崩溃了并再次重启）。应该通过客户端工具安全关闭主程序：
    
    ```sh
    mysqladmin -u root -p shutdown
    ```

---
## 连接/退出

- **连接本地数据库**：
    
    ```sh
    mysql -u root -p
    ```
    
    _(回车后输入密码。注意：`-p` 后面不要加空格直接写密码，或者回车后再密文输入)_
- **连接远程数据库**（指定 IP 和端口）：
    
    ```sh
    mysql -h 192.168.1.100 -P 3306 -u root -p
    ```
    
- **退出当前终端**：
    
    ```sh
    exit;  -- 或者 quit;
    ```
    

---

## 数据库备份/恢复（`mysqldump`）

- **备份单个数据库**（导出为 `.sql` 文件）：
    
    ```sh
    mysqldump -u root -p test_db > test_db_backup.sql
    ```
    
- **备份指定数据库中的某张表**：
    
    ```sh
    mysqldump -u root -p test_db users > users_table_backup.sql
    ```
    
- **备份全库（所有数据库）**：

    ```sh
    mysqldump -u root -p --all-databases > all_databases_backup.sql
    ```

    
- **导入/恢复数据库**：
    
    ```sh
    mysql -u root -p test_db < test_db_backup.sql
    ```
    
    _(注：导入前需确保 `test_db` 数据库在 MySQL 中已经存在)_

---
## 忘记root密码
> [!tip] 使用 `mysqld_safe` **跳过权限表启动数据库**以重置 `root` 密码

1. 停止当前正在运行的 MySQL 服务

```sh
sudo systemctl stop mysqld
# 或者
sudo service mysql stop
```

2. 使用安全模式（跳过权限验证）启动


```sh
sudo mysqld_safe --skip-grant-tables --skip-networking &
```

- `--skip-grant-tables`：核心参数，告诉 MySQL 启动时不加载权限表，任何人都可以免密登录。
- `--skip-networking`：**非常重要**！因为此时免密，必须加上这个参数关闭网络监听，只允许本地 socket 连接，防止外网黑客趁虚而入。

3. 免密登录并修改密码

直接输入 `mysql` 回车即可进入：

```sh
mysql
```

进入后执行修改逻辑：

```sql
FLUSH PRIVILEGES; -- 必须先刷新一次权限，否则无法修改密码

-- 如果是 MySQL 8.0+ 执行这句：
ALTER USER 'root'@'localhost' IDENTIFIED BY '你的新密码';

-- 如果是 MySQL 5.7 执行这句：
-- UPDATE mysql.user SET authentication_string=PASSWORD('你的新密码') WHERE User='root' AND Host='localhost';

FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' ACCOUNT UNLOCK; -- 确保账户未被锁定（8.0可选）
exit;
```

4. 重启正常服务

修改完成后，kill 掉刚刚的进程，并正常启动服务即可：

```sh
sudo mysqladmin -u root -p shutdown
sudo systemctl start mysqld
```