# 10. 常用扩展生态、错误处理与 MySQL 迁移避坑指南

## 1. 章节定位与开发者关注点

PostgreSQL 被誉为“最强开源数据库”的核心原因之一在于其**无与伦比的扩展系统（Extension Ecosystem）**。
本章解读官方文档 **附录 F（提供的扩展模块）** 与 **附录 A（SQLSTATE 错误代码）**，并结合实际工程经验，重点解析：
1. 业务开发中**必装、必用**的高收益官方/主流扩展。
2. 应用层基于标准 SQLSTATE 错误码的**事务重试与异常捕获规范**。
3. 从 MySQL 迁移到 PostgreSQL 的**全方位避坑清单与代码对照**。

每个扩展与技术点在解读完 PostgreSQL 17 的机制后，**均紧跟与 MySQL 的替代方案、性能及实现对比**。

---

## 2. 开发者必备的超级扩展生态与 MySQL 对比

### 2.1 `pg_stat_statements`：SQL 性能全景监控与慢查询排查

#### 🐘 PostgreSQL 17 机制：
- **作用**：记录所有执行过的 SQL 的参数归一化模板、执行次数、总耗时、平均耗时、共享内存块读写命中率（Shared Buffer Hit）。
- **启用方法**（在 `postgresql.conf` 中配置 `shared_preload_libraries = 'pg_stat_statements'`）：

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- 查询系统中最耗 CPU/I/O 的 Top 10 慢 SQL 模板
SELECT 
    query,
    calls,
    round(total_exec_time::numeric, 2) AS total_time_ms,
    round(mean_exec_time::numeric, 2) AS avg_time_ms,
    rows,
    round((100.0 * shared_blks_hit / nullif(shared_blks_hit + shared_blks_read, 0))::numeric, 2) AS cache_hit_ratio
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

---

#### 🥊 PostgreSQL vs MySQL 深度对比：慢查询监控

| 维度 | PostgreSQL `pg_stat_statements` | MySQL `performance_schema` / Slow Log |
| :--- | :--- | :--- |
| **开销与影响** | 极其轻量（内核级内存计数器，开销 < 1%），**生产环境标配全开** | `performance_schema` 全开内存和 CPU 开销显著，很多生产环境默认关闭或精简 |
| **SQL 归一化** | 自动剥离字面量参数（如 `WHERE id = ?`），精准归类相同逻辑的 SQL 模板 | Slow Log 是原始文本追加，需借助外部 pt-query-digest 工具解析聚类 |
| **缓存命中统计** | 详细展示 `shared_blks_hit` 与 `shared_blks_read`，直观判断是否缺物理内存 | 缺乏针对单 SQL 模板的 Buffer Pool 命中率统计 |

---

### 2.2 `pg_trgm`：三元组模糊查询加速 `LIKE '%keyword%'`

#### 🐘 PostgreSQL 17 机制：
- **痛点**：在传统数据库中，前置通配符查询 `LIKE '%华为手机%'` 无法走 B-Tree 索引，导致几千万数据全表扫描。
- **解决方案**：`pg_trgm` 使用三元组（Trigram）将字符串切分，配合 GIN 索引实现前缀、后缀、模糊甚至文本相似度匹配（Fuzzy Match）的极速索引查询！

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- 给商品标题创建 GIN Trigram 索引
CREATE INDEX idx_products_title_trgm ON products USING gin (title gin_trgm_ops);

-- 以下 SQL 完全走 GIN 索引扫描，毫秒级响应！
SELECT id, title, similarity(title, '苹果手机') AS score
FROM products
WHERE title LIKE '%苹果%' OR title % '苹果手机'
ORDER BY score DESC;
```

---

#### 🥊 PostgreSQL vs MySQL 深度对比：前缀/模糊匹配

| 场景 | PostgreSQL (`pg_trgm` + GIN) | MySQL 8.x |
| :--- | :--- | :--- |
| `WHERE col LIKE '%keyword%'` | **走 GIN 索引扫描（毫秒级）** | **强制全表扫描（Seq Scan）** |
| `WHERE col LIKE '%abc'` (后置匹配) | **走 GIN 索引扫描** | **强制全表扫描** |
| 文本相似度拼写纠错 | 支持操作符 `%` 与 `similarity()` 排序 | 无原生支持，需引入 Elasticsearch 或外部算法 |

---

### 2.3 `ltree`：优雅高效的无限层级树形数据结构

#### 🐘 PostgreSQL 17 机制：
- **痛点**：电商类目树（`数码 > 手机 > 5G手机`）、组织架构（`总公司.华东分部.研发部.后端组`）传统设计需要递归 CTE 或多次自关联查询，层级深或数据量大时性能低下。
- **解决方案**：`ltree` 专门用于存储扁平化的树路径，支持类似正则表达式的层级匹配与 GiST 索引。

```sql
CREATE EXTENSION IF NOT EXISTS ltree;

CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50),
    path ltree  -- 存储路径如: 'Top.Electronics.Phones.Smartphones'
);

CREATE INDEX idx_categories_path_gist ON categories USING gist (path);

-- 查询 'Electronics' 节点下的所有后代子节点（无论多少层）
SELECT * FROM categories WHERE path <@ 'Top.Electronics';

-- 查询指定父节点下精确 2 层子节点
SELECT * FROM categories WHERE path ~ 'Top.Electronics.*{1,2}';
```

---

#### 🥊 PostgreSQL vs MySQL 深度对比：层级树形数据存储

| 方案 | PostgreSQL (`ltree`) | MySQL 8.x (传统方案) |
| :--- | :--- | :--- |
| **存储方式** | 原生 `ltree` 路径类型，紧凑压缩 | 字符串 `VARCHAR` 拼接路径或 `parent_id` 关联 |
| **查询子孙节点** | 单条操作符 `path <@ 'Top.Node'`，走 GiST 索引 | 必须使用递归 CTE 或全表扫描 `path LIKE 'Top.Node%'` |
| **层级正则匹配** | 支持类似 `Top.*.Backend.*{1,2}` 的层级通配符 | MySQL 无法对路径中的层级深度建立专用索引 |

---

### 2.4 `pgcrypto`：数据库内原生数据加密与哈希

#### 🐘 PostgreSQL 17 机制：
```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- 1. 密码哈希与校验（类似 BCrypt）
INSERT INTO users (username, password_hash)
VALUES ('alice', crypt('MySecurePassword123', gen_salt('bf', 8)));

-- 校验登录密码
SELECT (password_hash = crypt('MySecurePassword123', password_hash)) AS is_valid
FROM users WHERE username = 'alice';

-- 2. 对称加密敏感数据（身份证、银行卡）
INSERT INTO sensitive_data (id_card_encrypted)
VALUES (pgp_sym_encrypt('110101199001011234', 'AES_SECRET_KEY'));

-- 解密读取
SELECT pgp_sym_decrypt(id_card_encrypted, 'AES_SECRET_KEY') AS id_card
FROM sensitive_data;
```

---

#### 🥊 PostgreSQL vs MySQL 深度对比：数据库内加解密

| 维度 | PostgreSQL (`pgcrypto`) | MySQL 8.x |
| :--- | :--- | :--- |
| **安全密码哈希** | 支持 `gen_salt('bf')` (Blowfish/BCrypt)，带自适应 Work Factor | 内置仅提供 MD5/SHA2，缺乏带 Salt 的原生 BCrypt 函数 |
| **PGP 标准加解密** | 支持标准 OpenPGP 对称与非对称加解密 | 仅提供基础 `AES_ENCRYPT()` / `AES_DECRYPT()` |

---

## 3. SQLSTATE 错误码与应用层健壮重试设计

PostgreSQL 返回标准的 5 字符 SQLSTATE 错误代码（附录 A）。在应用开发（Go/Java/Node/Python）中，**严禁通过字符串匹配错误信息，必须基于 SQLSTATE 错误码进行精准捕获与重试**。

### 核心错误码分类与应用层响应策略：

| 错误码 (SQLSTATE) | 含义 (Condition Name) | 典型触发场景 | 应用层处理策略 |
| :--- | :--- | :--- | :--- |
| **`40001`** | `serialization_failure` | 串行化隔离级别冲突（SSI） | **立即重试整个事务**（采用指数退避加抖动） |
| **`40P01`** | `deadlock_detected` | 死锁发生 | **立即重试整个事务** |
| **`23505`** | `unique_violation` | 唯一键冲突（如并发注册） | 捕获并向前端抛出“记录已存在”友好提示 |
| **`23503`** | `foreign_key_violation` | 外键约束失效 | 检查关联数据有效性，提示“关联数据不存在” |
| **`23514`** | `check_violation` | 违反 CHECK 约束 | 业务参数非法，拒绝写入 |
| **`57014`** | `query_canceled` | 语句超时（`statement_timeout`） | 记录慢查询告警，优化 SQL 或提升超时限制 |
| **`55P03`** | `lock_not_available` | 获取锁超时（`NOWAIT` 冲突） | 提示“系统繁忙/资源被占用，请稍后再试” |

---

#### 🥊 PostgreSQL vs MySQL 深度对比：错误码体系与驱动捕获

| 维度 | PostgreSQL | MySQL |
| :--- | :--- | :--- |
| **错误码标准** | **严格遵循 ANSI SQL-92 SQLSTATE**（如 `40001`, `23505`） | 混合使用 MySQL 专有错误码（如 `1062` Duplicate entry）与 SQLSTATE |
| **跨语言一致性** | 无论 Java (JDBC)、Go (pgx)、Node (pg)、Python (psycopg3)，均统一返回 5 字符 SQLSTATE | 不同语言驱动有时返回 Vendor Error Code，有时返回 SQLSTATE，兼容处理较繁琐 |

---

## 4. 从 MySQL 迁移到 PostgreSQL 的十大避坑对照清单

```
+----+-----------------------+----------------------------------+----------------------------------+---------------------------------------------+
| 序 | 避坑点                | MySQL (InnoDB) 表现              | PostgreSQL 17 表现               | 开发者规范与应对方案                        |
+----+-----------------------+----------------------------------+----------------------------------+---------------------------------------------+
| 1  | 标识符大小写          | 允许小写/驼峰，用反引号 `col`    | 默认全部转小写，双引号区分       | 一律使用全小写 + 下划线 (snake_case)        |
| 2  | 字符串与双引号        | 允许用双引号 `"str"` 表示字符串  | 双引号代表列名/表名，单引号代表字面量 | 字符串一律使用标准单引号 'str'              |
| 3  | 隐式类型转换          | 字符串与数字混比自动强转 (全表扫)| 严格强类型报错，拒绝自动强转     | 参数类型必须与字段严格一致，或使用 ::cast   |
| 4  | NULL 排序行为         | ORDER BY ASC 时 NULL 排在最前面  | ORDER BY ASC 时 NULL 默认在最后面| 显式使用 NULLS FIRST 或 NULLS LAST          |
| 5  | 时间时区存储          | DATETIME 无时区，TIMESTAMP 仅到2038| 推荐 TIMESTAMPTZ (存UTC/带时区)   | 业务时间字段一律选用 TIMESTAMPTZ            |
| 6  | 批量更新关联          | UPDATE t1 JOIN t2 ON ... SET ... | 不支持直接 JOIN，使用 UPDATE...FROM | 改写为 UPDATE t1 SET ... FROM t2 WHERE ...  |
| 7  | GROUP BY 聚合         | 宽松模式下允许 SELECT 非聚合列   | 严格遵循 SQL 标准，非法列直接报错| SELECT 列必须在 GROUP BY 中或在主键函数依赖中|
| 8  | 自增主键机制          | AUTO_INCREMENT (表级元数据)      | GENERATED ALWAYS AS IDENTITY     | 优先采用 SQL 标准 IDENTITY 语法代替 SERIAL  |
| 9  | JSON 类型选型         | 仅 JSON 类型                     | 提供 JSON 与 JSONB 两种          | 99.9% 业务场景无脑选择 JSONB (支持 GIN 索引)|
| 10 | 事务与 DDL 迁移       | DDL 会自动隐式提交当前事务       | DDL 支持完整的事务与原子回滚     | 在 Flyway / Liquibase 中享受安全原子迁移     |
+----+-----------------------+----------------------------------+----------------------------------+---------------------------------------------+
```
