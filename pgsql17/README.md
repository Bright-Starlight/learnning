# PostgreSQL 17 开发者核心指南与实战手册

> 基于官方文档（[PostgreSQL 17 Documentation](https://postgres.ac.cn/docs/17/index.html)）提炼，专为应用开发与架构设计定制。去除底层运维及内核杂音，深入剖析 SQL 语法、高级查询、数据类型、索引与全文检索、事务与并发控制、性能分析与优化、服务器端编程（PL/pgSQL 与触发器）、客户端接口及常用扩展生态，并全方位深度对比 MySQL (InnoDB)。

---

## 📑 章节目录导航

| 章节文件 | 对应官方文档 | 核心内容提要 |
| :--- | :--- | :--- |
| **[00. 总览与设计哲学](./00_PostgreSQL17_Developer_Guide_Overview.md)** | 全局概览 / 前言 | PG17 开发者升级亮点、PG vs MySQL 核心架构与哲学差异、业务场景选型指南 |
| **[01. SQL 语法与 DDL 表定义](./01_SQL_Syntax_and_DDL.md)** | Part II: Ch 4, 5 | 语法规范、表定义、约束体系（CHECK/EXCLUDE）、Generated 列、声明式分区表、**DDL 事务性** |
| **[02. 核心数据类型体系](./02_Data_Types.md)** | Part II: Ch 8, 10 | 数值、TIMESTAMPTZ、JSON/JSONB 深度对比、Array 数组、Range 范围类型、UUID 与强类型转换 |
| **[03. DML 与高级查询](./03_DML_and_Advanced_Queries.md)** | Part II: Ch 6, 7 | **RETURNING 子句**、ON CONFLICT (UPSERT)、**PG17 强化版 MERGE**、递归 CTE、窗口函数、LATERAL 连接 |
| **[04. 内置函数与高级操作符](./04_Functions_and_Operators.md)** | Part II: Ch 9 | ILIKE、POSIX 正则、date_trunc、generate_series、**SQL/JSON 构造与解析 (JSON_TABLE)**、数组操作符 |
| **[05. 索引原理、设计与全文搜索](./05_Indexes_and_Search.md)** | Part II: Ch 11, 12 | 6 大内置索引（B-Tree/Hash/GiST/SP-GiST/GIN/BRIN）、表达式索引、**部分索引 (Partial Index)**、覆盖索引、tsvector 全文检索 |
| **[06. 并发控制、MVCC 与锁机制](./06_Concurrency_Control_and_Transactions.md)** | Part II: Ch 13 | 隔离级别（**无幻读 RR / SSI**）、MVCC 隐藏列与可见性、FOR UPDATE / **SKIP LOCKED 任务队列**、**Advisory 咨询锁**、秒杀实战 |
| **[07. 性能优化与执行计划解读](./07_Performance_Optimization_and_Execution_Plans.md)** | Part II: Ch 14, 15 | **EXPLAIN (ANALYZE, BUFFERS)** 深度实战、Join 算法对比、**Extended Statistics 扩展统计信息**、并行查询调优 |
| **[08. 服务器编程、PL/pgSQL 与触发器](./08_Server_Programming_PLpgSQL_and_Triggers.md)** | Part V: Ch 36-45 | Function vs Procedure、PL/pgSQL 语法与异常捕获、行/语句触发器、**PG17 DDL 事件触发器（审计与拦截）** |
| **[09. 客户端接口与高级开发特性](./09_Client_Interfaces_and_Advanced_Features.md)** | Part IV & Part III | 多进程模型与连接池最佳实践（HikariCP / PgBouncer）、**原生异步通知 LISTEN/NOTIFY**、逻辑复制 CDC |
| **[10. 常用扩展生态与错误处理](./10_Popular_Extensions_and_Error_Handling.md)** | 附录 A & F | 必备扩展（pg_stat_statements, pg_trgm, ltree, pgcrypto 等）、SQLSTATE 错误码重试规范 |
| **[11. MySQL 与 PG 全维度对比与迁移全景指南](./11_PostgreSQL_vs_MySQL_Comprehensive_Comparison.md)** | 核心对比与避坑总结 | **十大维度技术对比矩阵**、架构进程模型差异、堆表 vs Undo Log、无间隙锁 vs Next-Key Lock、常用语法对照与十大避坑清单 |

---

## 🎯 核心特色

1. **精准去噪**：只保留对后端开发者（Java/Go/Python/Node.js 等）有实际价值的知识点。
2. **场景驱动**：每个章节配备 3~5 个真实业务场景 SQL 模板（电商、金融流水、时序日志、权限树、动态规格、排他预约、高并发秒杀等）。
3. **对比深刻**：每个章节均设置与 **MySQL (InnoDB)** 的深度对比，并附带专门的《[11. MySQL 与 PG 全维度对比与迁移全景指南](./11_PostgreSQL_vs_MySQL_Comprehensive_Comparison.md)》。
