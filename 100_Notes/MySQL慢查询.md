---
tags:
    - 运维/MySQL
---
---
# 监控发现
- `long_query_time`：建议生产环境逐步下调至 **0.1s - 0.5s**（默认 10s 毫无意义）。
- `log_queries_not_using_indexes = 1`：**必须开启**。没有使用索引的 SQL 即使在测试环境因数据量小执行很快，上线后随数据增长也会暴雷。