# 第 05 章：索引类型与全文搜索 (Indexes and Search)

> 本章对应 PostgreSQL 17 官方文档 **Part II. The SQL Language** 中第 11 章（Indexes）与第 12 章（Full Text Search）。
> 深入剖析 PostgreSQL 17 的 6 大内置索引访问方法、高级索引设计模式、全文检索与模糊搜索架构，并在各章节中深度对比 PostgreSQL 与 MySQL (InnoDB) 的底层机制与代码实现，辅以高并发生产实战。

---

## 1. 索引访问方法总览与选型指南

PostgreSQL 提供了关系型数据库中最丰富、扩展性最强的内置索引访问方法（Access Methods, AM）。与传统数据库仅提供 B+Tree 索引不同，PostgreSQL 针对不同数据维度和工作负载提供了 6 种开箱即用的原生索引机制。

```
                              ┌────────────────────────────────────────────────────────┐
                              │           PostgreSQL 原生索引体系选型决策树              │
                              └────────────────────────────────────────────────────────┘
                                                           │
                        ┌──────────────────────────────────┴──────────────────────────────────┐
                        ▼                                                                     ▼
                 【标量单一值数据】                                                    【复合/容器/空间/多值数据】
                        │                                                                     │
        ┌───────────────┴───────────────┐                             ┌───────────────────────┼───────────────────────┐
        ▼                               ▼                             ▼                       ▼                       ▼
  数据物理顺序高相关？              常规等值/范围/排序？            多值容器/JSONB/全文检索？    几何/范围重叠/多维空间？   非重叠空间/前缀/IP地址？
   (如时序日志追加)              (90%通用标量场景)                      (Inverted List)        (R-Tree / Lossy-Tree)      (Radix / Quad-Tree)
        │                               │                             │                       │                       │
        ▼                               ▼                             ▼                       ▼                       ▼
   ┌─────────┐                     ┌─────────┐                   ┌─────────┐             ┌─────────┐             ┌─────────┐
   │  BRIN   │                     │ B-Tree  │                   │   GIN   │             │  GiST   │             │ SP-GiST │
   └─────────┘                     └─────────┘                   └─────────┘             └─────────┘             └─────────┘
```

### 1.1 6 大内置索引访问方法深入剖析

| 索引类型 | 底层数据结构 / 核心原理 | 适用操作符 / 查询场景 | 典型应用字段 | 写入与空间开销 | PG17 特性增强 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **B-Tree** | Lehman & Yao 高并发 B-link 树，页面具有右向兄弟指针 | `=`, `<`, `<=`, `>`, `>=`, `BETWEEN`, `IN`, `IS NULL`, `ORDER BY` | 主键、数值、时间戳、普通字符串 | **空间/写入**：中等；点查与范围读均优 | 优化页面内去重（Deduplication）内存占用；提升 vacuum 清理效率 |
| **Hash** | 32位哈希分桶 + 溢出页链表 | 仅支持 `=` 等值精确匹配 | 高基数长字符串（如 UUID、URL、MD5/SHA 哈希） | **空间/写入**：小，不支持范围与排序 | WAL 日志全面支持崩溃安全；高并发点查吞吐极高 |
| **GiST** | Generalized Search Tree（广义平衡树），支持自定义损失性键 | `&&`（重叠）, `@>`（包含）, `<@`（被包含）, `<<`, `>>`, `<->`（距离） | 几何对象（PostGIS）、范围类型（`tsrange`, `daterange`）、全文检索 | **空间/写入**：较高；构建相对较慢 | 支持更多类型的数据压缩与排他约束加速 |
| **SP-GiST** | Space-Partitioned GiST（空间分区树，如四叉树、后缀树、基数树） | `=`, `^@`（前缀匹配）, `<<`, `>>`, `&&` | IP/网络地址（`inet`/`cidr`）、电话号码前缀、非均衡空间数据 | **空间/写入**：低~中等；分支非重叠，无平衡退化开销 | 优化非重叠前缀匹配查询路径 |
| **GIN** | Generalized Inverted Index（广义倒排索引），键 -> 行号数组/Bitmap | `@>`, `?`, `?|`, `?&`（JSONB/数组包含）, `@@`（全文检索匹配） | `jsonb`、`ARRAY`、`tsvector`、`pg_trgm` 相似度搜索 | **空间**：中~大；**写入**：开启 `fastupdate` 延迟合批优化 | 优化增量合批写入（Pending List）清空性能与内存占用 |
| **BRIN** | Block Range Index（块范围索引），仅记录每组连续物理块的 Min/Max 值 | `=`, `<`, `<=`, `>`, `>=`, `BETWEEN`（依赖物理顺序性） | 时序日志、自增序列、按时间追加的大数据量只读/归档表 | **空间**：极小（B-Tree 的 0.1%~1%）；**写入**：极快 | 支持多范围（Multi-Range）与 Bloom 过滤器组合索引 |

---

## 2. 深入 6 大索引实现机制

### 2.1 B-Tree 索引（默认核心）
PostgreSQL 的 B-Tree 基于 Lehman & Yao 算法，其特点是在每个节点引入指向右侧同级节点的“High Key”与右指针，允许读操作在无须锁住父节点的情况下顺着兄弟指针并发向下和向右查找，极大降低了读写锁竞争。

- **B-Tree 去重机制（Deduplication）**：自 PG13 引入并在 PG17 进一步强化，当索引中出现大量重复键（如 `status` 列或低基数复合键）时，PostgreSQL 会在索引叶子页内将相同的 Key 合并，后挂 Posting List（TID 列表），将索引体积压缩 40%~70%，大幅提升缓存命中率。

```sql
-- 默认创建即为 B-Tree 索引
CREATE INDEX idx_users_created_at ON users (created_at);

-- 显式指定 deduplicate_items（默认为 ON）
CREATE INDEX idx_orders_status ON orders (status) WITH (deduplicate_items = ON);
```

### 2.2 Hash 索引
Hash 索引将索引列经过 32 位 Hash 函数计算后映射到固定桶（Buckets）中。在 PG 10 以前 Hash 索引不写 WAL 日志，但从 PG 10 起 Hash 索引已具备完整的 WAL 事务保证、可复制性与高并发锁优化。
- **选型建议**：当且仅当只需要 `=` 精确匹配，且索引键为极长的字符串（如长度 500 的 URL），且不希望占用过大 B-Tree 树高空间时使用。

```sql
CREATE INDEX idx_user_tokens_hash ON user_tokens USING hash (token);
```

### 2.3 GiST 索引（通用搜索树）
GiST 是一种高度可扩展的树状索引模板，由通用平衡树节点和用户自定义操作符集构成。
- **核心能力**：处理“是否重叠”、“是否相交”、“最近邻 KNN”等 B-Tree 无法处理的非线性多维数据。
- **排他约束（Exclusion Constraint）**：利用 GiST 索引可以在数据库引擎层原生防止时间区间或空间重叠。

```sql
-- 预订系统：同一会议室在同一时间段内严禁重复预订
CREATE TABLE room_reservations (
    reservation_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    room_id INT NOT NULL,
    during TSRANGE NOT NULL,
    EXCLUDE USING gist (room_id WITH =, during WITH &&)
);
```

### 2.4 SP-GiST 索引（空间分区树）
SP-GiST 允许开发非平衡、基于空间递归细分的索引结构（如 Radix Tree、Quad-Tree、k-d Tree）。
- 与 GiST 不同，SP-GiST 的子节点空间是**互不重叠**的，因此点查时不需要遍历多条可能重叠的分支，查询复杂度更稳定。

```sql
-- 对 IP 地址/网段进行高性能前缀与包含匹配
CREATE TABLE access_logs (
    id BIGINT GENERATED ALWAYS AS IDENTITY,
    client_ip INET NOT NULL,
    access_time TIMESTAMPTZ DEFAULT clock_timestamp()
);

CREATE INDEX idx_logs_ip_spgist ON access_logs USING spgist (client_ip);

-- 高效执行子网包含查询
SELECT * FROM access_logs WHERE client_ip <<= '192.168.1.0/24'::inet;
```

### 2.5 GIN 索引（广义倒排索引）
GIN 索引将复合容器内部的“元素（Elements）”提取出来，建立 `元素 -> [TID1, TID2, ...]` 的倒排列表。
- **Fastupdate 机制**：由于单条记录插入可能涉及数十个元素的倒排链更新，GIN 默认提供 `fastupdate = on`，新写入先追加到 `Pending List`（轻量内存/页），随后由后台 Worker、Vacuum 或累积达到 `gin_pending_list_limit`（默认 4MB）时合批重入倒排树，兼顾写入吞吐。

```sql
-- 为 JSONB 字段与标签数组建立 GIN 倒排索引
CREATE TABLE product_catalog (
    id BIGINT PRIMARY KEY,
    tags TEXT[],
    attributes JSONB
);

-- jsonb_ops 支持 ?、?&、?|、@> 操作符
CREATE INDEX idx_catalog_tags ON product_catalog USING gin (tags);
CREATE INDEX idx_catalog_attr_jsonb ON product_catalog USING gin (attributes jsonb_ops);
-- jsonb_path_ops 仅支持 @>，但索引体积更小、查询更快
CREATE INDEX idx_catalog_attr_path ON product_catalog USING gin (attributes jsonb_path_ops);
```

### 2.6 BRIN 索引（块范围索引）
BRIN 是专门为海量有序数据（如时序数据、自增流水数据）设计的索引。它不存储每行数据的索引条目，而是以若干个连续数据页（默认 `pages_per_range = 128`，即 1MB 物理块）为一个组，仅记录该组内该列的 `[Min, Max]`。
- **空间消耗**：1 亿条数据的 B-Tree 索引可能需要 2~3 GB，而 BRIN 索引仅需数十 KB 至数 MB。

```sql
CREATE TABLE metric_timeseries (
    metric_id BIGINT,
    recorded_at TIMESTAMPTZ NOT NULL,
    value DOUBLE PRECISION
);

-- 创建 BRIN 索引，指定每 64 页统计一次极值
CREATE INDEX idx_metrics_brin ON metric_timeseries USING brin (recorded_at) WITH (pages_per_range = 64);
```

---

#### 🥊 索引类型体系对比：PG (6 大丰富索引 + 开放可扩展 AM) vs MySQL (单一 B+Tree 体系)

| 对比维度 | PostgreSQL 17 | MySQL 8.x (InnoDB) | 深度原理解析与架构差异 |
| :--- | :--- | :--- | :--- |
| **原生索引类型** | **6 种原生内置索引**（B-Tree, Hash, GiST, SP-GiST, GIN, BRIN） | **几乎单一 B+Tree**（另有局限的 FULLTEXT 和 SPATIAL R-Tree） | PG 将 Access Method 抽象为标准可插拔接口；MySQL InnoDB 存储引擎内核与 B+Tree 强绑定。 |
| **JSON/半结构化索引** | **GIN 倒排索引**：一条索引覆盖 JSONB 内部所有键值对与层级，支持动态任意 Key 检索 | **B+Tree 虚拟生成列**：必须预先为具体的 JSON Path 创建虚拟列再建立 B+Tree 索引 | PG 支持任意未知属性查询；MySQL 对无固定 Schema 的 JSON 检索极其痛苦且维护成本高。 |
| **多维空间与区间重叠** | **GiST / SP-GiST**：原生支持时间范围重叠、KNN 距离排序、排他约束 | **仅 SPATIAL 索引 (R-Tree)**，仅支持 GIS 几何对象，不支持时间/数值区间重叠 | PG 的 GiST 允许在数据库层拦截重叠预订等业务冲突；MySQL 必须依赖应用层事务锁或行级排他。 |
| **时序与自增海量数据** | **BRIN 索引**：仅记录块范围极值，空间节省 99.5%，写入零开销 | **无**（只能建立全量 B+Tree 索引，占用数 GB 内存与磁盘） | PG BRIN 在 IoT、时序监控、自增归档场景下呈现压倒性优势。 |
| **开放扩展能力** | **完全开放**：第三方插件可注册全新索引（如 AI 向量索引 `pgvector` HNSW/IVFFlat、`RUM`、`BM25`） | **封闭**：无法在 InnoDB 内部自定义底层索引访问方法 | PG 能够直接化身为高性能向量数据库与多模数据库。 |

##### 典型代码对照：多属性动态 JSON 检索

```sql
-- 🐘 PostgreSQL 17：一条 GIN 倒排索引覆盖所有动态属性查询
CREATE TABLE pg_products (
    id BIGINT PRIMARY KEY,
    specs JSONB
);

-- 创建 GIN 倒排索引（包含内部所有 Key-Value）
CREATE INDEX idx_pg_specs ON pg_products USING GIN (specs);

-- 查询：屏幕为 OLED 且 电池 >= 5000 的任意属性组组合（全走索引！）
SELECT * FROM pg_products 
WHERE specs @> '{"screen": "OLED", "battery": 5000}';
```

```sql
-- 🐬 MySQL 8.0：必须预先知道查询的 Key，创建生成列再建 B+Tree 索引
CREATE TABLE mysql_products (
    id BIGINT PRIMARY KEY,
    specs JSON,
    -- 必须手动提取虚拟生成列
    screen_val VARCHAR(64) AS (specs->>'$.screen') STORED,
    battery_val INT AS (CAST(specs->>'$.battery' AS UNSIGNED)) STORED,
    INDEX idx_screen (screen_val),
    INDEX idx_battery (battery_val)
);

-- 如果业务要新增查询 specs->>'$.cpu'，MySQL 必须执行 ALTER TABLE DDL 增加列和索引！
SELECT * FROM mysql_products 
WHERE screen_val = 'OLED' AND battery_val = 5000;
```

---

## 3. 特殊索引设计模式

### 3.1 表达式索引（Expression / Functional Index）
当查询条件对列进行了函数运算、类型转换或表达式求值时，常规索引无法命中，必须建立表达式索引。PostgreSQL 原生要求表达式内的函数必须被标记为 **`IMMUTABLE`（不可变函数）**。

```sql
-- 针对大小写不敏感的邮箱查询
CREATE TABLE app_users (
    id BIGINT PRIMARY KEY,
    email VARCHAR(255) NOT NULL
);

-- 创建小写表达式索引
CREATE INDEX idx_users_lower_email ON app_users (lower(email));

-- 准确命中索引
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM app_users WHERE lower(email) = 'developer@postgres.org';
```

---

#### 🥊 表达式/函数索引对比：PostgreSQL 原生表达式索引 vs MySQL 8.0 函数索引

| 维度 | PostgreSQL 17 | MySQL 8.0 (InnoDB) |
| :--- | :--- | :--- |
| **底层实现机制** | **原生首等公民**：直接在索引元数据中记录表达式与运算结果，无需依附于物理列 | **隐藏虚拟列封装**：MySQL 8.0 在底层自动生成隐藏的 `Generated Column` 并挂载 B+Tree 索引 |
| **语法书写规则** | 直接写表达式：`CREATE INDEX idx ON tbl (lower(col));` | **必须加双括号**：`CREATE INDEX idx ON tbl ((lower(col)));` |
| **函数稳定性要求** | 严格要求函数为 `IMMUTABLE`（确定性输入输出） | 要求表达式为确定性函数（Deterministic），不支持局部非确定性操作 |
| **多字段复合表达式** | 支持复杂复合表达式如 `(first_name || ' ' || last_name)` | 支持，但底层虚拟列名自动生成（类似 `!hidden!idx_xxx`） |

##### 语法对照与避坑：

```sql
-- 🐘 PostgreSQL 17
CREATE INDEX idx_pg_lower_email ON app_users (lower(email));

-- 🐬 MySQL 8.0+（注意必须写双层括号，且需注意字符集 Collation 匹配）
CREATE INDEX idx_mysql_lower_email ON app_users ((lower(email)));
-- MySQL 5.7 则必须先显式 ADD COLUMN lower_email VARCHAR(255) AS (lower(email)) VIRTUAL 然后再建索引。
```

---

### 3.2 部分索引（Partial / Conditional Index）
部分索引只为满足 `WHERE` 谓词条件的子集行建立索引条目。
- **核心价值**：
  1. 减少 90%~99% 的索引空间占用与缓存浪费；
  2. 极大降低对冷数据或非目标数据更新时的索引维护开销；
  3. 实现条件唯一约束（例如：每个用户只能存在一条 `is_default = true` 的支付方式）。

```sql
-- 场景 1：千万级工单表，99% 的工单已关闭 (status = 'CLOSED')，仅 1% 为待处理 ('PENDING')
CREATE INDEX idx_tickets_pending ON tickets (created_at, priority)
WHERE status = 'PENDING';

-- 场景 2：条件唯一约束
CREATE TABLE user_payment_methods (
    id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    card_number TEXT NOT NULL,
    is_default BOOLEAN NOT NULL DEFAULT FALSE
);

-- 每个用户仅允许拥有一个默认支付方式（is_default=false 时允许多条，is_default=true 时严格唯一）
CREATE UNIQUE INDEX uq_user_default_payment ON user_payment_methods (user_id)
WHERE is_default = TRUE;
```

---

#### 🥊 部分索引对比：PG 原生支持条件索引省 95% 空间 vs MySQL 完全不支持 (只能建全量索引)

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
│ 业务痛点：5000 万行订单表中，只有 50 万行（1%）处于 'UNPAID' 待支付状态，后台调度系统每秒高频轮询 │
└─────────────────────────────────────────────────────────────────────────────────────────────────┘
```

| 维度 | PostgreSQL 17 (部分索引) | MySQL 8.x (无部分索引) |
| :--- | :--- | :--- |
| **索引创建方式** | `CREATE INDEX ... WHERE status = 'UNPAID'` | 只能全量创建 `CREATE INDEX ... (status, created_at)` |
| **索引条目数量** | 仅包含 50 万行有效数据（1%） | 必须强行包含 5000 万行全部数据（100%） |
| **索引体积** | **约 15 MB**（全部常驻 Buffer Cache） | **约 1.6 GB**（严重挤占 InnoDB Buffer Pool） |
| **DML 写入性能影响** | 更新 99% 的已完成订单时，**完全不触发该索引维护** | 任何订单状态修改均要更新该 B+Tree 索引，造成大量索引页写放大 |
| **条件唯一约束** | 原生支持：`CREATE UNIQUE INDEX ... WHERE ...` | **无法实现**（唯一索引会对所有行生效，无法排除 `status='DELETED'`） |

##### 代码与业务实现对照：

```sql
-- 🐘 PostgreSQL 17：极简优雅，兼顾极致性能与条件唯一性
-- 1. 业务只索引活跃用户
CREATE INDEX idx_pg_active_users ON users (last_login_time) WHERE is_deleted = FALSE;

-- 2. 软删除场景下的唯一约束（已删除账号允许同名，未删除账号用户名全局唯一）
CREATE UNIQUE INDEX uq_pg_user_name ON users (username) WHERE is_deleted = FALSE;
```

```sql
-- 🐬 MySQL 8.0：完全不支持部分索引！
-- 1. 必须索引全部 5000 万行，包含已删除数据
CREATE INDEX idx_mysql_users ON users (is_deleted, last_login_time);

-- 2. 软删除唯一约束在 MySQL 中极难解决：
-- 传统方案只能在删除时将 username 改为 'admin_deleted_1688001122'，侵入业务且破坏数据可溯源性。
```

---

### 3.3 覆盖索引与 `INCLUDE` 子句（Covering Index）
自 PG 11 引入 `INCLUDE` 语法，允许在 B-Tree 索引中将非键列（Payload 列）附加到叶子节点，而不参与 B-Tree 树形排序。
- **Index-Only Scan 机制**：查询只需读取索引叶子页即可获取所有所需列，无需回表读取数据堆（Heap Scan）。
- **可见性映射表（Visibility Map, VM）**：PostgreSQL 的 MVCC 版本信息存放在数据行（Heap Tuple）中，而非索引页中。为了安全执行 Index-Only Scan，PG 依赖 VM 文件记录每个数据页是否对所有当前事务“全可见（All-Visible）”。若对应数据页是 All-Visible，则真正实现 0 回表读取。

```sql
-- 需求：经常根据 user_id 查询用户的 nickname 和 avatar_url
CREATE INDEX idx_users_lookup ON users (user_id) INCLUDE (nickname, avatar_url);

-- 执行查询，执行计划显示 Index Only Scan
EXPLAIN (ANALYZE, BUFFERS)
SELECT user_id, nickname, avatar_url FROM users WHERE user_id = 10086;
```

---

#### 🥊 覆盖索引机制对比：PG (Index Only Scan + VM 表) vs MySQL (二级索引隐式包含主键)

| 对比维度 | PostgreSQL 17 (`INCLUDE` 覆盖索引) | MySQL 8.x (InnoDB 覆盖索引) |
| :--- | :--- | :--- |
| **底层实现机制** | `CREATE INDEX ... INCLUDE (col1, col2)`：非键列仅存放在叶子页 Payload 中，不参与 B-Tree 排序 | 联合索引 `(col1, col2, col3)` 参与整棵 B+Tree 排序，或借助二级索引叶子节点隐式包含的主键列 |
| **MVCC 可见性判断与回表** | **依赖 Visibility Map (VM)**：若 VM 标记全可见（All-Visible），则 0 回表；否则仍需访问堆页读取 `xmin`/`xmax` | **天然支持**：InnoDB 二级索引包含主键，只要查询字段都在二级索引中，直接在内存完成覆盖扫描 |
| **索引树深度与维护** | `INCLUDE` 列不参与排序，索引树体积比多列联合索引更紧凑，B-Tree 深度更浅 | 联合索引所有列参与排序和比较，树节点体积较大 |
| **主键依赖度** | 索引与物理堆表解耦，索引只存 TID 行指针 `(page_no, offset)` | 二级索引强绑定聚簇主键；若主键为长 UUID，所有二级索引体积均会膨胀 |

```sql
-- 🐘 PostgreSQL 17 覆盖索引
CREATE INDEX idx_pg_cov ON orders (user_id) INCLUDE (status, total_amount);
-- EXPLAIN 显示：Index Only Scan using idx_pg_cov on orders (Heap Fetches: 0)

-- 🐬 MySQL 8.0 覆盖索引
CREATE INDEX idx_mysql_cov ON orders (user_id, status, total_amount);
-- EXPLAIN 显示：Using index
```

---

### 3.4 复合索引顺序与最左前缀
- 复合 B-Tree 索引 `(a, b, c)` 可以支持 `WHERE a = ?`、`WHERE a = ? AND b = ?`，以及 `WHERE a = ? ORDER BY b`。
- **PG 索引选择与设计核心准则**：
  1. **等值筛选列放在最左侧**（高基数列优先）；
  2. **范围条件列（`>`, `<`, `BETWEEN`）放在等值列之后**；
  3. **排序列应与 `ORDER BY` 方向保持一致或完全反向**（PG 支持 `ASC NULLS FIRST` / `DESC NULLS LAST` 索引构建）。

---

## 4. PostgreSQL 全文检索与模糊搜索架构

PostgreSQL 原生内置了成熟的全文检索（FTS）与三元组（Trigram）模糊搜索体系，无需额外搭建 Elasticsearch 集群即可满足绝大部分站内中英文全文搜索与模糊查询需求。

### 4.1 核心概念：`tsvector` 与 `tsquery`
1. **`tsvector`（分词文档向量）**：将文本解析、词干提取（Stemming）、去除停用词（Stopwords）并记录词位置（Positions）与权重（Weight: A, B, C, D）。
2. **`tsquery`（检索条件表达式）**：支持布尔操作符 `&` (AND)、`|` (OR)、`!` (NOT)、`<->` (短语/邻近距离匹配)。
3. **匹配操作符 `@@`**：判断 `tsvector` 是否匹配 `tsquery`。

```sql
-- 生成 tsvector 与权重标记
SELECT to_tsvector('english', 'PostgreSQL is the world''s most advanced open source database');
-- 输出: 'advanc':6 'databas':9 'most':5 'open':7 'postgresql':1 'sourc':8 'world':3

-- 使用 @@ 操作符与短语距离匹配
SELECT to_tsvector('english', 'PostgreSQL 17 high performance search') 
    @@ to_tsquery('english', 'postgresql & (performance | optimization)'); -- 返回 true

-- 邻近匹配: 'open' 紧跟 'source' (距离为 1)
SELECT to_tsvector('english', 'open source software') @@ to_tsquery('english', 'open <-> source'); -- 返回 true
```

### 4.2 全文检索实战：生成列 + GIN 索引 + 相关度评分

```sql
-- 1. 创建文章表，利用 GENERATED ALWAYS AS 生成预分词 tsvector 列
CREATE TABLE articles (
    article_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title TEXT NOT NULL,
    summary TEXT,
    content TEXT NOT NULL,
    -- 结合不同权重：A 权重（最高），B 权重，C 权重
    tsv_search TSVECTOR GENERATED ALWAYS AS (
        setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(summary, '')), 'B') ||
        setweight(to_tsvector('english', coalesce(content, '')), 'C')
    ) STORED
);

-- 2. 为 tsvector 列建立 GIN 倒排索引
CREATE INDEX idx_articles_tsv ON articles USING gin (tsv_search);

-- 3. 全文检索查询：结合 ts_rank_cd 与 ts_headline 高亮摘要
SELECT 
    article_id,
    title,
    ts_rank_cd(tsv_search, query) AS rank_score,
    ts_headline('english', content, query, 'StartSel=<mark>, StopSel=</mark>, MaxWords=35, MinWords=15') AS snippet
FROM articles, to_tsquery('english', 'database & performance') query
WHERE tsv_search @@ query
ORDER BY rank_score DESC
LIMIT 10;
```

### 4.3 全文检索 vs `pg_trgm` 三元组模糊匹配
- **Full Text Search (FTS)**：基于语义分词与词根（如 `search` 匹配 `searching`、`searched`），适合自然语言文档检索。
- **`pg_trgm` (Trigram)**：将字符串拆分为 3 字符滑动窗口。专门用于解决 `LIKE '%keyword%'` 前缀/中间通配模糊搜索、拼写纠错（Fuzzy Search）与相似度匹配。

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- 为商品标题创建基于三元组的 GIN 索引
CREATE INDEX idx_goods_name_trgm ON goods USING gin (name gin_trgm_ops);

-- 高效加速任意位置模糊匹配（彻底告别全表扫描！）
SELECT * FROM goods WHERE name LIKE '%机械键盘%';

-- 计算文本相似度排序
SELECT name, similarity(name, '罗技无线鼠标') AS sm
FROM goods
WHERE name % '罗技无线鼠标'
ORDER BY sm DESC LIMIT 5;
```

---

#### 🥊 全文检索与模糊匹配对比：PostgreSQL (tsvector + pg_trgm) vs MySQL (FULLTEXT 索引)

| 维度 | PostgreSQL 17 (`tsvector` + `pg_trgm`) | MySQL 8.x (`FULLTEXT` 索引) |
| :--- | :--- | :--- |
| **任意位置模糊搜索 (`LIKE '%key%'`)** | **`pg_trgm` + GIN 索引毫秒级命中**（三元组滑动切分索引） | **无法使用任何索引**，强制全表扫描（Full Table Scan），大表必死 |
| **分词与倒排索引** | 原生 `tsvector` + GIN 倒排索引，支持丰富词根解析（如 `zhparser` 中文插件） | 内置 `FULLTEXT` 索引，中文支持依赖 `ngram` 解析器 |
| **短语与位置检索** | 支持 `<->` 精确词距操作符（例如 `apple <2> phone`） | 仅支持简单的 `IN BOOLEAN MODE` 短语引用 `"apple phone"` |
| **加权评分体系** | 原生支持 A/B/C/D 四级权重 (`setweight`) 与 `ts_rank_cd` 密集度评分算法 | 仅提供基础的相关度权重计算，无法自定义字段分级加权 |
| **拼写容错与相似度** | `pg_trgm` 原生支持编辑距离、三元组相似度计算与 `%` 阈值匹配 | 不支持拼写容错，输入错字直接返回 0 条结果 |

##### 典型代码对照：中缀模糊匹配 `LIKE '%phone%'`

```sql
-- 🐘 PostgreSQL 17：使用 pg_trgm 瞬间走 GIN 索引
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_pg_trgm ON products USING gin (title gin_trgm_ops);

-- 执行查询：走 Bitmap Index Scan，0 毫秒返回
EXPLAIN ANALYZE SELECT * FROM products WHERE title LIKE '%华为Mate60%';
```

```sql
-- 🐬 MySQL 8.0：FULLTEXT 与 B+Tree 对中缀 LIKE 均束手无策
SELECT * FROM products WHERE title LIKE '%华为Mate60%';
-- 执行计划：type=ALL (全表扫描！千万数据直接引发慢查询告警和 CPU 100%)
```

---

## 5. 生产级业务实战场景

### 场景 1：千万级订单表按状态过滤（部分索引实现百倍空间与性能优化）

**业务背景**：订单中心表 `orders` 数据量 5000 万，历史已完成（`COMPLETED`）和已取消（`CANCELLED`）订单占 98.5%，唯独待支付（`UNPAID`）和待出库（`PROCESSING`）订单占 1.5%（约 75 万条），但后台调度系统每秒高频轮询待处理订单。

```sql
-- 方案对比：

-- [方案 A: 全量 B-Tree 索引]
CREATE INDEX idx_orders_status_full ON orders (status, created_at);
-- 索引体积: ~1.8 GB，写操作频繁维护无用历史数据索引

-- [方案 B: 部分索引 (推荐)]
CREATE INDEX idx_orders_active_processing ON orders (created_at, order_id)
WHERE status IN ('UNPAID', 'PROCESSING');
-- 索引体积: ~24 MB (仅原索引的 1.3%！)，内存可全量常驻 Buffer Pool

-- 调度轮询 SQL：完美命中极小的部分索引
SELECT order_id, user_id, amount, created_at
FROM orders
WHERE status = 'UNPAID' AND created_at <= NOW() - INTERVAL '15 minute'
ORDER BY created_at ASC
LIMIT 100;
```

---

### 场景 2：高并发范围重叠校验与地理位置检索（GiST 索引应用）

**业务背景**：网约车/外卖系统中，需要高频查询骑手当前地理坐标（经纬度 Point）附近 3 公里内的所有商家，并支持按直线距离升序排序（KNN 检索）。

```sql
CREATE EXTENSION IF NOT EXISTS earthdistance CASCADE;

CREATE TABLE merchants (
    merchant_id BIGINT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    location POINT NOT NULL -- (经度, 纬度)
);

-- 为 POINT 列创建 GiST 索引
CREATE INDEX idx_merchants_location_gist ON merchants USING gist (location);

-- KNN 距离排序：利用 <-> 操作符直接在 GiST 索引内部进行最近邻堆排序
SELECT merchant_id, name, location,
       location <-> point(116.4074, 39.9042) AS distance_degrees
FROM merchants
ORDER BY location <-> point(116.4074, 39.9042)
LIMIT 20;
```

---

### 场景 3：亿级时序日志表超低成本索引（BRIN 索引实战）

**业务背景**：IOT 设备上报流水表，每天产生 2000 万条日志，历史保留 180 天（总计约 36 亿行）。按时间段进行范围统计分析。

```sql
CREATE TABLE device_telemetry (
    log_id BIGINT GENERATED ALWAYS AS IDENTITY,
    device_id INT NOT NULL,
    recorded_time TIMESTAMPTZ NOT NULL,
    payload JSONB
) PARTITION BY RANGE (recorded_time);

-- 在日分区或总表上创建 BRIN 索引
CREATE INDEX idx_telemetry_time_brin ON device_telemetry 
USING brin (recorded_time) 
WITH (pages_per_range = 128, autosummarize = ON);

-- 空间与效果对比：
-- 36 亿行数据下：
-- B-Tree 索引体积: ~80 GB
-- BRIN 索引体积:   ~120 MB (压缩比 > 600:1)
-- 查询执行：BRIN 直接跳过 99.9% 无关数据块，带来与 B-Tree 极其接近的点范围扫描性能
```

---

### 场景 4：电商商品标题多关键词检索与相关度排序（tsvector + GIN）

**业务背景**：数码商城商品检索，用户输入“苹果 蓝牙 耳机 防水”，需要快速匹配包含关键词的商品，并根据标题命中率和商品权重综合计算得分输出。

```sql
CREATE TABLE products (
    product_id BIGINT PRIMARY KEY,
    title TEXT NOT NULL,
    brand VARCHAR(50),
    sales_volume INT DEFAULT 0,
    search_vector TSVECTOR
);

-- 自动更新触发器 / 函数
CREATE OR REPLACE FUNCTION trg_sync_product_search() RETURNS trigger AS $$
BEGIN
    NEW.search_vector := 
        setweight(to_tsvector('simple', coalesce(NEW.title, '')), 'A') ||
        setweight(to_tsvector('simple', coalesce(NEW.brand, '')), 'B');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_products_tsv_update BEFORE INSERT OR UPDATE ON products
FOR EACH ROW EXECUTE FUNCTION trg_sync_product_search();

-- 创建 GIN 倒排索引
CREATE INDEX idx_products_search_gin ON products USING gin (search_vector);

-- 检索查询：包含多词与相关性加权排序
WITH search_param AS (
    SELECT plainto_tsquery('simple', '苹果 蓝牙 耳机') AS q
)
SELECT 
    p.product_id,
    p.title,
    p.brand,
    ts_rank_cd(p.search_vector, sp.q, 32 /* 规范化文档长度 */) * (1 + ln(p.sales_volume + 1) * 0.1) AS final_score
FROM products p, search_param sp
WHERE p.search_vector @@ sp.q
ORDER BY final_score DESC
LIMIT 50;
```

---

## 6. 最佳开发实践与运维排查

1. **生产环境无阻塞创建索引（CONCURRENTLY）**：
   - 生产环境严禁直接运行 `CREATE INDEX`（会持有 `SHARE` 锁阻塞所有 `INSERT`/`UPDATE`/`DELETE` 写操作）。
   - 必须使用 `CREATE INDEX CONCURRENTLY`，以非阻塞方式经历两阶段扫描创建索引。
2. **避免无效与冗余索引**：
   - 联合索引 `(a, b)` 已涵盖 `(a)`，无需重复创建单列索引 `(a)`。
   - 利用系统视图 `pg_stat_user_indexes` 监控索引扫描次数（`idx_scan`），及时清理 `idx_scan = 0` 的废弃冗余索引。
3. **维护与索引膨胀治理**：
   - 高频 UPDATE/DELETE 会导致 B-Tree 索引页出现空洞和膨胀，推荐定期使用 `REINDEX TABLE CONCURRENTLY` 进行无锁在线重建。
