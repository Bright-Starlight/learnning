# PostgreSQL 17 官方开发指南（三）：数据操作（DML）与高级查询

---

## 1. 概述与核心导读

在现代企业级应用开发中，数据库早已不仅仅是持久化存储的容器，更是承担高并发数据加工、复杂报表分析与原子流水线处理的核心计算引擎。PostgreSQL 17 在标准 SQL 兼容性、DML 执行效率和高级查询功能上保持了业界领先地位。

本章深度解读 PostgreSQL 官方文档 Part II（SQL 语言）第 6 章与第 7 章，重点剖析以下核心内容：
- **高级 DML 与 RETURNING**：通过单条语句完成数据变更与结果反馈，彻底消除网络往返（RTT）与二次查询开销。
- **UPSERT (`ON CONFLICT`)**：基于唯一索引的高性能原子插入与精准冲突合并。
- **PostgreSQL 17 MERGE 重大增强**：全面支持 `MERGE ... RETURNING`（含 `merge_action()` / `$action` 伪列），实现标准企业级 ETL 与数据同步。
- **CTE（公用表表达式）与物化控制**：掌握 `MATERIALIZED` / `NOT MATERIALIZED` 优化器提示与数据修改 CTE（DML in CTE）。
- **递归 CTE（WITH RECURSIVE）**：树形层次、图遍历与 ANSI SQL `CYCLE` 防环控制。
- **窗口函数（Window Functions）与 LATERAL 连接**：复杂分析、动态滑动窗口与 Top-N 查询最优解法。
- **多维聚合与条件聚合**：`GROUPING SETS`、`CUBE`、`ROLLUP` 与 `FILTER (WHERE ...)` 语法。
- **四大全流程业务实战** 与 **章节内嵌入式 PostgreSQL vs MySQL 8.x 深度对比**。

---

## 2. 基础 DML 强化与 RETURNING 子句

### 2.1 INSERT / UPDATE / DELETE 核心语法与高级用法

PostgreSQL 支持 ANSI SQL 标准 DML，并提供高度灵活的扩展语法：

```sql
-- 1. 批量插入 + 默认值填充 + 表达式计算
INSERT INTO user_accounts (username, email, balance, created_at)
VALUES 
  ('alice', 'alice@example.com', 100.00, DEFAULT),
  ('bob',   'bob@example.com',   250.50, clock_timestamp());

-- 2. 基于查询的插入 (INSERT ... SELECT)
INSERT INTO order_archive (order_id, customer_id, total_amount, archived_at)
SELECT id, customer_id, total_amount, NOW()
FROM orders
WHERE status = 'COMPLETED' AND created_at < NOW() - INTERVAL '1 year';

-- 3. 多表关联更新 (UPDATE ... FROM)
UPDATE inventory i
SET stock_count = i.stock_count - oi.quantity,
    updated_at = NOW()
FROM order_items oi
WHERE i.product_id = oi.product_id
  AND oi.order_id = 1001;

-- 4. USING 子句关联删除 (DELETE ... USING)
DELETE FROM cart_items ci
USING user_blacklist ub
WHERE ci.user_id = ub.user_id
  AND ub.status = 'BANNED';
```

### 2.2 RETURNING 子句深度解析

在传统应用开发中，执行 `INSERT`、`UPDATE`、`DELETE` 后若需获取自增 ID、默认计算列（如 `DEFAULT gen_random_uuid()`）或被删除的历史数据，常规做法往往是再次发起 `SELECT`。这不仅额外增加了一次网络 I/O 往返（RTT），在高并发场景下还极易发生并发数据漂移。

PostgreSQL 的 `RETURNING` 子句允许在 DML 语句末尾直接返回受影响行的字段、运算表达式或整行记录（`RETURNING *`）：

```sql
-- 1. INSERT RETURNING：获取自增主键与默认生成的 UUID / 时间戳
INSERT INTO users (username, email)
VALUES ('charlie', 'charlie@example.com')
RETURNING id, gen_random_uuid() AS user_token, created_at;

-- 2. UPDATE RETURNING：原子扣减并获取更新前后的计算值
UPDATE accounts
SET balance = balance - 50.00,
    updated_at = NOW()
WHERE id = 101 AND balance >= 50.00
RETURNING id, balance AS new_balance, (balance + 50.00) AS old_balance;

-- 3. DELETE RETURNING：安全获取并归档被删除数据
DELETE FROM audit_logs
WHERE log_time < NOW() - INTERVAL '90 days'
RETURNING log_id, user_id, action, log_time;
```

> [!TIP]
> `RETURNING` 子句支持任意标量表达式、类型转换、函数调用以及聚合别名。在客户端驱动（如 JDBC、pgx、psycopg3）中，携带 `RETURNING` 的 DML 会以标准结果集（ResultSet / Cursor）形式返回，开发体验与 `SELECT` 完全一致。

---

### 2.3 🥊 RETURNING 深度对比：PostgreSQL 原生直接返回 vs MySQL 完全不支持

#### 核心代码对照

##### 场景 A：插入数据并获取自动生成的字段（自增 ID、UUID、默认时间戳）

```sql
-- ============================================================================
-- PostgreSQL 17：单条 SQL 原子完成插入并获取所有生成列（网络交互：1 次）
-- ============================================================================
INSERT INTO orders (user_id, order_no, total_amount)
VALUES (1001, 'ORD-2026-001', 299.00)
RETURNING id, order_no, created_at, status;

-- ============================================================================
-- MySQL 8.x：必须依赖连接级别的 LAST_INSERT_ID() 或二次 SELECT（网络交互：2 次）
-- ============================================================================
-- 步骤 1：插入记录
INSERT INTO orders (user_id, order_no, total_amount)
VALUES (1001, 'ORD-2026-001', 299.00);

-- 步骤 2：额外发起查询获取自增 ID 与其他默认列
SELECT LAST_INSERT_ID() AS id, order_no, created_at, status 
FROM orders 
WHERE id = LAST_INSERT_ID();
```

##### 场景 B：批量插入场景下获取所有新生成的 ID

```sql
-- ============================================================================
-- PostgreSQL 17：批量插入多行，RETURNING 返回全部对应的真实 ID 列表
-- ============================================================================
INSERT INTO order_items (order_id, product_id, quantity)
VALUES (100, 1, 2), (100, 2, 1), (100, 3, 5)
RETURNING item_id, product_id;
-- 结果集精确返回 3 行，每行对应独立的 item_id！

-- ============================================================================
-- MySQL 8.x：LAST_INSERT_ID() 仅返回批量插入的第一行 ID！
-- ============================================================================
INSERT INTO order_items (order_id, product_id, quantity)
VALUES (100, 1, 2), (100, 2, 1), (100, 3, 5);
-- SELECT LAST_INSERT_ID(); 仅返回第一条记录的 ID！
-- 后续记录的 ID 只能靠应用层假设“ID 连续自增”去推算（在并发或分布式场景极易出错）。
```

#### 架构与机制深度解析

| 对比维度 | PostgreSQL 17 | MySQL 8.x | 架构与生产影响 |
| :--- | :--- | :--- | :--- |
| **语言规范支持** | `INSERT / UPDATE / DELETE / MERGE ... RETURNING` 完整支持 | **完全不支持** `RETURNING` 子句 | PG 具备统一的正交语法；MySQL 需根据不同操作编写额外代码。 |
| **网络往返（RTT）** | **1 次 RTT** 即可完成写操作与数据拉取 | 至少 **2 次 RTT**（先写后查） | 在跨可用区或云数据库网络延迟 1~2ms 场景下，PG 的吞吐与响应时间优势极其明显。 |
| **批量插入 ID 捕获** | 精确返回每一行的生成主键与计算列 | `LAST_INSERT_ID()` 仅返回首行 ID | MySQL 应用层若推算自增 ID，遇到自增步长变化或插入冲突时极易发生数据错位。 |
| **并发安全性** | **强一致原子性**：返回的数据即为变更时的瞬时视图 | 二次 `SELECT` 存在并发窗口期 | MySQL 在高并发下二次查询可能查到被其他事务修改后的脏数据（幻读/更新漂移）。 |

---

## 3. UPSERT 语法：`ON CONFLICT` 深度剖析

`INSERT ... ON CONFLICT` 是 PostgreSQL 实现“存在则更新，不存在则插入”（UPSERT）的原生机制，完全保证 ACID 原子性与并发安全。

### 3.1 核心语法结构

```sql
INSERT INTO target_table (column_list)
VALUES (value_list)
ON CONFLICT [ conflict_target ] conflict_action;
```

其中 `conflict_target` 可以是：
- `(column_name [COLLATE collation]) [WHERE index_predicate]`：冲突字段列表（必须与唯一的 B-Tree 索引或排他性约束完全匹配）。
- `ON CONSTRAINT constraint_name`：显式指定唯一约束名。

`conflict_action` 可以是：
- `DO NOTHING`：发生冲突时静默忽略，不抛出异常，返回 0 行影响。
- `DO UPDATE SET column = expression [WHERE condition]`：发生冲突时执行更新。

### 3.2 `EXCLUDED` 虚拟伪表与条件更新

在 `DO UPDATE` 子句中，PostgreSQL 提供一个特殊的虚拟表 `EXCLUDED`，代表**原本试图插入但发生冲突的新数据行**。

```sql
-- 创建测试表与唯一索引
CREATE TABLE product_statistics (
    product_id BIGINT PRIMARY KEY,
    view_count BIGINT DEFAULT 1,
    cart_count BIGINT DEFAULT 0,
    last_viewed_at TIMESTAMPTZ DEFAULT NOW()
);

-- 业务插入/累计统计数据
INSERT INTO product_statistics (product_id, view_count, cart_count, last_viewed_at)
VALUES (1001, 1, 0, NOW())
ON CONFLICT (product_id) 
DO UPDATE SET
    view_count = product_statistics.view_count + EXCLUDED.view_count,
    cart_count = product_statistics.cart_count + EXCLUDED.cart_count,
    last_viewed_at = GREATEST(product_statistics.last_viewed_at, EXCLUDED.last_viewed_at)
WHERE product_statistics.last_viewed_at < EXCLUDED.last_viewed_at -- 仅当新时间戳更新时才触发写操作
RETURNING product_id, view_count, cart_count;
```

### 3.3 部分索引（Partial Index）与冲突目标匹配

如果表上的唯一索引带有 `WHERE` 谓词（部分索引），`ON CONFLICT` 的冲突目标必须显式声明相同的 `WHERE` 条件：

```sql
-- 创建针对活跃用户的唯一索引
CREATE UNIQUE INDEX uk_active_user_phone 
ON users (phone) 
WHERE is_deleted = FALSE;

-- 必须在 ON CONFLICT 中带上相同的 WHERE 条件
INSERT INTO users (phone, username, is_deleted)
VALUES ('13800000000', 'david', FALSE)
ON CONFLICT (phone) WHERE is_deleted = FALSE
DO UPDATE SET username = EXCLUDED.username;
```

---

### 3.4 🥊 UPSERT 深度对比：PostgreSQL (ON CONFLICT) vs MySQL (ON DUPLICATE KEY UPDATE)

#### 核心代码对照

```sql
-- ============================================================================
-- PostgreSQL 17：语义清晰，精准指定冲突索引，支持 EXCLUDED 伪表与 WHERE 过滤
-- ============================================================================
INSERT INTO user_points (user_id, points, updated_at)
VALUES (1001, 50, NOW())
ON CONFLICT (user_id) 
DO UPDATE SET 
    points = user_points.points + EXCLUDED.points,
    updated_at = EXCLUDED.updated_at
WHERE user_points.points + EXCLUDED.points >= 0 -- 条件更新：积分扣减后不能为负
RETURNING user_id, points;

-- ============================================================================
-- MySQL 8.x：语法较松散，无法指定冲突目标，无法直接在 UPDATE 加 WHERE 谓词
-- ============================================================================
INSERT INTO user_points (user_id, points, updated_at)
VALUES (1001, 50, NOW())
AS new_row -- MySQL 8.0.19+ 别名语法（替代已弃用的 VALUES(points)）
ON DUPLICATE KEY UPDATE 
    points = user_points.points + new_row.points,
    updated_at = new_row.updated_at;
```

#### 核心机制与架构差异

```
               [PostgreSQL ON CONFLICT]                           [MySQL ON DUPLICATE KEY UPDATE]
                          |                                                      |
            +-------------+-------------+                          +-------------+-------------+
            | 显式声明唯一键 target (A) |                          | 隐式检测表中所有唯一索引   |
            +-------------+-------------+                          +-------------+-------------+
                          |                                                      |
       +------------------+------------------+                    +--------------+--------------+
       |                                     |                    |                             |
[仅当 A 冲突时触发]                  [其他唯一索引 B 冲突]       [主键冲突触发]            [唯一索引 B 冲突触发]
       |                                     |                    |                             |
[安全执行 DO UPDATE]                 [抛出 UniqueViolation]       +--------------+--------------+
                                                                                 |
                                                                   [均触发同一个 UPDATE！极易误改]
```

| 核心维度 | PostgreSQL 17 (`ON CONFLICT`) | MySQL 8.x (`ON DUPLICATE KEY UPDATE`) | 生产场景影响 |
| :--- | :--- | :--- | :--- |
| **冲突目标精准度** | **强制指定**特定列、复合列、唯一约束名或部分索引条件 | **无法指定目标**，表内任意 PRIMARY KEY 或 UNIQUE KEY 冲突均触发 | MySQL 在多唯一索引表上极其危险，可能因次要唯一索引冲突误更新主键行。 |
| **部分索引（Partial Index）** | 原生支持（如 `WHERE is_deleted = false` 的唯一索引） | **不支持**部分索引 | PG 支持软删除场景下的高效局部唯一性冲突处理。 |
| **更新条件过滤 (`WHERE`)** | 支持 `DO UPDATE ... WHERE predicate`，条件不满足则跳过更新 | **不支持** `WHERE` 过滤，只能通过 `IF()` 表达式赋原值模拟 | PG 可避免无意义的行级写锁与 WAL 日志膨胀；MySQL 即使值未变也会产生锁开销。 |
| **伪表引用语法** | 标准 `EXCLUDED.col_name`，语义清晰直观 | 旧版 `VALUES(col)`（已弃用），8.0.19+ 改为 `AS new_row` | PG 语法在版本演进中长期稳定一致。 |

---

## 4. PostgreSQL 17 MERGE 语句重大增强

PostgreSQL 15 首次引入了标准 ANSI SQL 的 `MERGE` 语句，而在 **PostgreSQL 17** 中，`MERGE` 迎来里程碑级强化：**正式支持 `RETURNING` 子句与 `$action` / `merge_action()` 伪列**，并且在性能和条件分支处理上得到了大幅优化。

### 4.1 核心语法（PG 17 标准）

```sql
MERGE INTO target_table AS t
USING source_table_or_query AS s
ON join_condition
WHEN MATCHED [AND condition] THEN
    UPDATE SET column = expr, ... | DELETE | DO NOTHING
WHEN NOT MATCHED [BY TARGET] [AND condition] THEN
    INSERT (column_list) VALUES (values_list) | DO NOTHING
WHEN NOT MATCHED BY SOURCE [AND condition] THEN -- PG17 扩展分支
    UPDATE SET ... | DELETE | DO NOTHING
[RETURNING merge_returning_list];
```

### 4.2 PG 17 `MERGE ... RETURNING` 实战案例

PostgreSQL 17 允许在 `MERGE` 尾部添加 `RETURNING`，并可通过 `merge_action()` 获取每一行实际发生的动作类型（`INSERT` / `UPDATE` / `DELETE`）：

```sql
-- 目标表：用户会员积分总表
CREATE TABLE member_points (
    member_id BIGINT PRIMARY KEY,
    total_points INT NOT NULL,
    status VARCHAR(20) NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- 源临时表：每日积分变动流水汇总
CREATE TEMP TABLE staging_points_delta (
    member_id BIGINT,
    delta_points INT,
    is_cancelled BOOLEAN
);

INSERT INTO staging_points_delta VALUES 
  (1, 50, false),   -- 既有用户增分
  (2, -100, false), -- 既有用户扣分
  (3, 200, false),  -- 新用户注册加分
  (4, 0, true);     -- 注销用户

-- PG 17 批量合并并直接返回变更审计日志
MERGE INTO member_points AS t
USING staging_points_delta AS s
ON t.member_id = s.member_id
WHEN MATCHED AND s.is_cancelled = TRUE THEN
    DELETE
WHEN MATCHED AND (t.total_points + s.delta_points) >= 0 THEN
    UPDATE SET 
        total_points = t.total_points + s.delta_points,
        updated_at = NOW()
WHEN NOT MATCHED AND s.is_cancelled = FALSE THEN
    INSERT (member_id, total_points, status, updated_at)
    VALUES (s.member_id, GREATEST(0, s.delta_points), 'ACTIVE', NOW())
RETURNING 
    merge_action() AS action_taken, -- 返回 'INSERT', 'UPDATE', 或 'DELETE'
    t.member_id,
    t.total_points AS current_points,
    NOW() AS audit_time;
```

---

### 4.3 🥊 MERGE 深度对比：PostgreSQL 17 标准 MERGE + RETURNING vs MySQL 完全不支持 MERGE

#### 核心代码对照

##### 业务需求：将 staging 表的数据同步至 production 表，匹配到的执行更新，未匹配到的执行插入，标记删除的执行删除。

```sql
-- ============================================================================
-- PostgreSQL 17：单条 ANSI MERGE 语句原子覆盖全部分支（增/改/删/审计）
-- ============================================================================
MERGE INTO prod_items p
USING stage_items s ON p.sku = s.sku
WHEN MATCHED AND s.op = 'D' THEN DELETE
WHEN MATCHED THEN UPDATE SET price = s.price, stock = s.stock
WHEN NOT MATCHED AND s.op != 'D' THEN INSERT (sku, price, stock) VALUES (s.sku, s.price, s.stock)
RETURNING merge_action(), p.sku;

-- ============================================================================
-- MySQL 8.x：不支持 MERGE 语句！必须拆分为多条 SQL 并包裹在显式事务中
-- ============================================================================
START TRANSACTION;

-- 1. 手动执行条件删除
DELETE p FROM prod_items p
JOIN stage_items s ON p.sku = s.sku
WHERE s.op = 'D';

-- 2. 执行插入或更新 (UPSERT)
INSERT INTO prod_items (sku, price, stock)
SELECT s.sku, s.price, s.stock
FROM stage_items s
WHERE s.op != 'D'
AS new_s
ON DUPLICATE KEY UPDATE 
    price = new_s.price,
    stock = new_s.stock;

-- 3. 无法获取实际哪些行执行了 INSERT、哪些执行了 UPDATE，需应用层额外比对
COMMIT;
```

#### 机制差异对比表

| 特性维度 | PostgreSQL 17 (`MERGE`) | MySQL 8.x (无 MERGE) |
| :--- | :--- | :--- |
| **标准兼容性** | **完全兼容 ANSI SQL:2016 / SQL:2023** | **完全不支持** 标准 MERGE 语法 |
| **条件删除分支 (`WHEN MATCHED THEN DELETE`)** | 原生支持在单次扫描中直接删除满足条件的行 | 必须通过前置独立的 `DELETE ... JOIN` 完成 |
| **审计返回 (`RETURNING merge_action()`)** | 原生支持返回每行实际发生的动作（INSERT/UPDATE/DELETE） | 无法感知，只能通过 `ROW_COUNT()` 获取模糊影响行数 |
| **源表反向匹配 (`NOT MATCHED BY SOURCE`)** | **PG 17 原生支持**（处理源端已下架数据的清理） | 不支持，必须编写复杂的 `LEFT JOIN ... WHERE s.id IS NULL` |
| **执行开销与锁持有** | 单次全表 JOIN 扫描，行锁在单条语句执行周期内持有 | 多条语句导致多次表扫描，事务时间拉长，死锁风险显著增加 |

---

## 5. CTE 与 MATERIALIZED / NOT MATERIALIZED 提示控制

### 5.1 CTE（公用表表达式）基础与数据流水线

CTE（`WITH` 语句）不仅极大提升了复杂 SQL 的可读性与模块化程度，还支持将 `INSERT / UPDATE / DELETE` 放入 CTE 中，形成原子数据处理管道（DML in CTE）。

```sql
-- 数据管道：注销超时订单，并同步记录归档日志
WITH expired_orders AS (
    UPDATE orders
    SET status = 'CANCELLED', updated_at = NOW()
    WHERE status = 'PENDING' AND created_at < NOW() - INTERVAL '30 minutes'
    RETURNING id AS order_id, user_id, total_amount
),
inserted_logs AS (
    INSERT INTO order_audit_logs (order_id, action, detail, operator)
    SELECT order_id, 'AUTO_CANCEL', 'Payment timeout expired', 'SYSTEM'
    FROM expired_orders
    RETURNING log_id, order_id
)
SELECT eo.order_id, eo.user_id, eo.total_amount, il.log_id
FROM expired_orders eo
JOIN inserted_logs il ON eo.order_id = il.order_id;
```

### 5.2 `MATERIALIZED` 与 `NOT MATERIALIZED` 优化器控制

PostgreSQL 在版本演进中对 CTE 的执行策略进行了重要优化：
- **PG 11 及更早版本**：CTE 永远被**强制物化（Materialized）**，即作为优化屏障（Optimization Fence），外层查询无法将谓词下推（Predicate Pushdown）到 CTE 内部。
- **PG 12+ 及 PG 17**：如果 CTE 是只读的、无副作用且仅被引用一次，优化器默认会将其**内联展开（Inlined / NOT MATERIALIZED）**，允许谓词下推与索引联动。
- **显式控制**：开发者可以使用 `AS MATERIALIZED` 或 `AS NOT MATERIALIZED` 强制干预执行计划。

```sql
-- 场景 A：强制物化 (AS MATERIALIZED)
-- 当 CTE 计算非常昂贵且在外层查询中被多次 JOIN/扫描时，避免重复计算
WITH expensive_stats AS MATERIALIZED (
    SELECT category_id, AVG(price) AS avg_price, STDDEV(price) AS price_std
    FROM products
    GROUP BY category_id
)
SELECT p.id, p.name, p.price, s.avg_price
FROM products p
JOIN expensive_stats s ON p.category_id = s.category_id
WHERE p.price > s.avg_price + s.price_std;

-- 场景 B：强制内联 (AS NOT MATERIALIZED)
-- 确保外部过滤条件可以下推到 CTE 内部，命中索引
WITH filtered_users AS NOT MATERIALIZED (
    SELECT id, username, org_id, created_at
    FROM users
    WHERE is_deleted = FALSE
)
SELECT * FROM filtered_users WHERE org_id = 999; -- org_id 索引条件将被直接下推至底表扫描
```

---

### 5.3 🥊 CTE 深度对比：PostgreSQL (DML in CTE + 物化控制) vs MySQL 8.0 CTE

#### 核心代码对照

```sql
-- ============================================================================
-- PostgreSQL 17：DML in CTE 流水线，单条语句完成“出队 + 消费 + 写入审计”
-- ============================================================================
WITH popped_task AS (
    DELETE FROM task_queue
    WHERE task_id = (
        SELECT task_id FROM task_queue 
        WHERE status = 'PENDING' 
        ORDER BY priority DESC, created_at ASC 
        FOR UPDATE SKIP LOCKED 
        LIMIT 1
    )
    RETURNING task_id, payload
)
INSERT INTO task_history (task_id, payload, executed_at)
SELECT task_id, payload, NOW()
FROM popped_task
RETURNING task_id, executed_at;

-- ============================================================================
-- MySQL 8.x：CTE 仅支持纯 SELECT 查询，完全不支持在 CTE 中包含 DML！
-- 必须改写为多条命令的存储过程或应用层事务控制
-- ============================================================================
-- 必须分步：
-- 1. SELECT ... FOR UPDATE SKIP LOCKED 查出 task_id
-- 2. INSERT INTO task_history
-- 3. DELETE FROM task_queue WHERE task_id = ...
```

#### 机制差异全景对比

| 对比维度 | PostgreSQL 17 | MySQL 8.0+ |
| :--- | :--- | :--- |
| **DML in CTE (可写 CTE)** | **原生支持**，`WITH` 内部可自由使用 `INSERT/UPDATE/DELETE/MERGE` 并配合 `RETURNING` | **完全不支持**，CTE 内部只能是 `SELECT` |
| **物化控制提示** | 原生支持 `AS MATERIALIZED` 与 `AS NOT MATERIALIZED` 显式控制优化器 | 不支持物化提示，优化器自行决定是否合并或派生临时表 |
| **优化屏障（Optimization Fence）** | 可通过 `MATERIALIZED` 阻断谓词下推，规避特定计算异常（如除零或无效类型转换） | 无法精准人工干预 CTE 的内联与物化决策 |
| **事务原子管道能力** | 一条 SQL 贯穿“删除旧数据 -> 插入新归档 -> 输出结果”，无需应用层维护多步事务 | 必须依赖显式多语句事务，增加了事务持有时间与连接占用 |

---

## 6. 递归 CTE（WITH RECURSIVE）与防环控制

递归 CTE 是解决层次遍历（组织架构、商品分类、物料清单 BOM）、图路径搜索和时序递推的标准武器。

### 6.1 递归执行原理图解

```
                        +----------------------------+
                        |  Non-recursive Term (锚点)  |
                        +----------------------------+
                                       |
                                       v
                        +----------------------------+
                        | Working Table (工作表 T0)  | <----------------+
                        +----------------------------+                  |
                                       |                                 |
                                       v                                 |
                        +----------------------------+                  |
                        |   Recursive Term (递归项)   |                  |
                        |     (引用工作表计算下一层)   |                  |
                        +----------------------------+                  | (循环直至工作表为空)
                                       |                                 |
                          [生成中间结果集 Intermediate]                   |
                                       |                                 |
                  +--------------------+--------------------+            |
                  |                                         |            |
                  v                                         v            |
     [Append 到 Final Result]                    [更新为新的 Working Table] -+
```

### 6.2 环路检测：`CYCLE` 子句 (ANSI SQL 标准)

在图数据结构或异常父子数据中，极易出现循环引用（如 `A -> B -> C -> A`），导致递归陷入死循环。PostgreSQL 支持原生的 `CYCLE` 子句自动检测环路：

```sql
CREATE TABLE graph_nodes (
    node_id INT PRIMARY KEY,
    next_node_id INT
);

INSERT INTO graph_nodes VALUES (1, 2), (2, 3), (3, 1); -- 存在环路: 1 -> 2 -> 3 -> 1

-- 使用 CYCLE 子句防止死循环
WITH RECURSIVE traverse AS (
    SELECT node_id, next_node_id, 1 AS depth
    FROM graph_nodes
    WHERE node_id = 1
    
    UNION ALL
    
    SELECT g.node_id, g.next_node_id, t.depth + 1
    FROM graph_nodes g
    JOIN traverse t ON g.node_id = t.next_node_id
)
CYCLE node_id SET is_cycle USING path_array
SELECT node_id, next_node_id, depth, is_cycle, path_array
FROM traverse;
```

---

### 6.3 🥊 递归 CTE 深度对比：PostgreSQL (标准 CYCLE 子句) vs MySQL 8.0 手动防环

#### 核心代码对照

```sql
-- ============================================================================
-- PostgreSQL 17：原生 CYCLE 子句，自动生成环路标记与路径追踪数组
-- ============================================================================
WITH RECURSIVE dept_tree AS (
    SELECT id, parent_id, name, 1 AS lvl
    FROM departments WHERE id = 100
    UNION ALL
    SELECT d.id, d.parent_id, d.name, dt.lvl + 1
    FROM departments d JOIN dept_tree dt ON d.parent_id = dt.id
)
CYCLE id SET is_cycle USING path -- 标准防环，自动在检测到重复 ID 时停止下钻
SELECT id, name, lvl, is_cycle, path FROM dept_tree;

-- ============================================================================
-- MySQL 8.0：不支持 CYCLE 子句！必须手动拼接路径字符串并使用 FIND_IN_SET 判断
-- ============================================================================
WITH RECURSIVE dept_tree AS (
    -- 锚点：初始化路径字符串
    SELECT id, parent_id, name, 1 AS lvl, CAST(id AS CHAR(200)) AS path_str, 0 AS is_cycle
    FROM departments WHERE id = 100
    UNION ALL
    -- 递归：手动检查 ID 是否已存在于路径字符串中
    SELECT 
        d.id, d.parent_id, d.name, dt.lvl + 1,
        CONCAT(dt.path_str, ',', d.id),
        CASE WHEN FIND_IN_SET(d.id, dt.path_str) > 0 THEN 1 ELSE 0 END
    FROM departments d 
    JOIN dept_tree dt ON d.parent_id = dt.id
    WHERE FIND_IN_SET(d.id, dt.path_str) = 0 -- 手动终止条件
)
SELECT id, name, lvl, is_cycle, path_str FROM dept_tree;
```

#### 关键技术差异

1. **防环机制**：PostgreSQL 使用原生底层高效算法追踪访问路径，支持 `CYCLE col SET flag USING path`；MySQL 必须依靠 `cte_max_recursion_depth` 参数兜底或在 SQL 中进行昂贵的字符串拼接与 `FIND_IN_SET` 查找。
2. **数据结构**：PostgreSQL 原生提供强类型数组 `path_array`，支持任意维度与类型的元素；MySQL 只能使用有限长度的 `VARCHAR` 字符串模拟，深度较大时存在截断溢出风险。

---

## 7. 窗口函数（Window Functions）全景

窗口函数在不破坏行粒度（不压缩分组行数）的前提下，跨与当前行相关的行集合执行多维聚合与排序计算。

### 7.1 核心语法与窗口帧（Window Frame）规范

```sql
function_name([args]) OVER (
    [PARTITION BY partition_expr, ...]
    [ORDER BY sort_expr [ASC | DESC] [NULLS FIRST | NULLS LAST], ...]
    [
        { ROWS | RANGE | GROUPS } 
        BETWEEN frame_start AND frame_end
        [ EXCLUDE { CURRENT ROW | GROUP | TIES | NO OTHERS } ]
    ]
)
```

#### 窗口帧模式对比：
- `ROWS`：按物理行数偏移（如 `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW`）。
- `RANGE`：按值范围偏移（基于 `ORDER BY` 的列值差值区间计算）。
- `GROUPS`：按同行值组（Peer Groups）偏移。
- `EXCLUDE`：支持在窗口计算中精确剔除当前行、当前同值组或同行关联项。

### 7.2 核心窗口函数分类及特性

| 函数分类 | 函数名称 | 说明与特点 |
| :--- | :--- | :--- |
| **排名函数** | `ROW_NUMBER()` | 绝对连续递增序列（1, 2, 3, 4），无并列 |
| | `RANK()` | 并列跳跃排序（1, 2, 2, 4） |
| | `DENSE_RANK()` | 并列连续排序（1, 2, 2, 3） |
| | `NTILE(n)` | 将数据集等频切分为 n 个分箱（桶号 1 ~ n） |
| **值偏移函数** | `LAG(col, offset, default)` | 获取当前行之前第 offset 行的值（环比、同比必备） |
| | `LEAD(col, offset, default)` | 获取当前行之后第 offset 行的值 |
| **边界值函数** | `FIRST_VALUE(col)` | 窗口帧内第一行的值 |
| | `LAST_VALUE(col)` | 窗口帧内最后一行的值（注意默认帧范围对结果的影响） |
| | `NTH_VALUE(col, n)` | 窗口帧内第 n 行的值 |

> [!WARNING]
> 当指定了 `ORDER BY` 但省略窗口帧时，SQL 标准默认帧为 `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`。若使用 `LAST_VALUE(col)`，其默认只计算到当前行，无法取到整个分区的最后一个值。要取分区最终值，必须显式指定 `ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING` 或使用 `FIRST_VALUE(col) OVER (ORDER BY ... DESC)`。

```sql
-- 窗口函数全功能演示
SELECT 
    dept_id,
    emp_name,
    salary,
    -- 部门内薪资严格排序与去重排序
    ROW_NUMBER() OVER w AS row_num,
    DENSE_RANK() OVER w AS dense_rk,
    -- 部门内前一位同事与后一位同事的薪资（无则填充 0）
    LAG(salary, 1, 0.00) OVER w AS prev_salary,
    LEAD(salary, 1, 0.00) OVER w AS next_salary,
    -- 部门内累计薪资（Running Total）
    SUM(salary) OVER (
        PARTITION BY dept_id 
        ORDER BY salary DESC 
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_dept_salary,
    -- 部门平均薪资对比
    AVG(salary) OVER (PARTITION BY dept_id) AS dept_avg_salary
FROM employees
WINDOW w AS (PARTITION BY dept_id ORDER BY salary DESC);
```

---

### 7.3 🥊 窗口函数深度对比：PostgreSQL vs MySQL 8.0 支持度与性能差异

#### 规范支持与高级功能对比

| 窗口特性 | PostgreSQL 17 | MySQL 8.0+ | 架构分析与实际影响 |
| :--- | :--- | :--- | :--- |
| **基础窗口函数** | 完整支持（`ROW_NUMBER`, `RANK`, `LAG`, `LEAD` 等） | 完整支持 | 两者在基本函数命名和行为上符合 ANSI SQL 标准。 |
| **`GROUPS` 帧规范** | **完全支持**（按同值组 Peer Groups 偏移） | **完全不支持**（仅支持 `ROWS` 和 `RANGE`） | 复杂金融或科学计算中处理同值多行聚合时，PG 表达更精确。 |
| **`EXCLUDE` 帧剔除子句** | **完全支持** (`EXCLUDE GROUP / TIES / CURRENT ROW`) | **完全不支持** | PG 能够轻松实现“计算组内除当前行外的其他行平均值”等复杂统计。 |
| **结合 `FILTER` 子句** | 原生支持：`COUNT(*) FILTER (WHERE status = 'A') OVER (...)` | 不支持，必须嵌套 `CASE WHEN` | PG 代码清晰且执行阶段直接跳过无效行。 |
| **命名窗口 (`WINDOW w AS`)** | 完整支持并在多个函数中复用 | 8.0+ 支持命名窗口 | 均能有效减少重复的 `OVER (...)` 代码冗余。 |
| **并行计算支持** | **支持 Parallel WindowAgg**，多工作进程并行加速大表窗口计算 | 窗口函数计算仅单线程执行 | 在千万级大表上做全量窗口分析时，PostgreSQL 性能领先数倍。 |

---

## 8. LATERAL 连接（横向派生表）

`LATERAL` 是 PostgreSQL 极其强大但常被低估的高级特性。它允许 `FROM` 子句中的右侧子查询或表函数**直接引用左侧表表达式中的字段**，突破了标准 SQL 子查询无法向外引用的作用域限制。

### 8.1 LATERAL 工作机制

```sql
SELECT c.id, c.name, latest_order.order_id, latest_order.total_amount
FROM customers c
CROSS JOIN LATERAL (
    -- 子查询内部可以直接使用外部表 c 的字段 c.id
    SELECT o.id AS order_id, o.total_amount
    FROM orders o
    WHERE o.customer_id = c.id
    ORDER BY o.created_at DESC
    LIMIT 1
) latest_order;
```

### 8.2 LATERAL 的核心优势
1. **Top-N 极致优化**：对每个外层行取关联的 Top-K 记录，配合关联字段复合索引可走极其高效的 `Index Scan`，性能远超大表全量窗口函数排序。
2. **多列拆分与函数解包**：调用返回多列或多行的集合函数（如 `jsonb_to_recordset`、`unnest`）并与主表平铺关联。

---

### 8.3 🥊 LATERAL 深度对比：PostgreSQL 原生 LATERAL 解决 Top-N vs MySQL 8.0.14+ 限制

#### 核心代码对照（每位客户获取最近 3 笔订单）

```sql
-- ============================================================================
-- PostgreSQL 17 与 MySQL 8.0.14+ 语法层面均支持 LATERAL
-- ============================================================================
SELECT c.id AS customer_id, c.name, o.order_id, o.total_amount, o.order_date
FROM customers c
CROSS JOIN LATERAL (
    SELECT id AS order_id, total_amount, order_date
    FROM orders
    WHERE customer_id = c.id
    ORDER BY order_date DESC
    LIMIT 3
) o;
```

#### 执行计划与内核优化深度剖析

```
[PostgreSQL LATERAL 执行计划]
-> Nested Loop
   -> Seq Scan on customers c
   -> Index Scan using idx_orders_cust_date on orders (customer_id = c.id, ORDER BY DESC LIMIT 3)
   (极速：每个客户仅通过索引精确跳读 3 条记录，全过程无临时表、无全表排序！)

[MySQL LATERAL 常见执行瓶颈]
-> Nested Loop
   -> Table scan on customers c
   -> Table scan on <derived2>
      -> Limit: 3 row(s)
         -> Sort: orders.order_date DESC  <-- 容易退化为 filesort，或创建内存派生临时表
```

| 对比维度 | PostgreSQL 17 | MySQL 8.0.14+ | 架构影响 |
| :--- | :--- | :--- | :--- |
| **索引下推与跳跃扫描** | 优化器完美将 `LIMIT N` 与复合索引 `(foreign_key, sort_col DESC)` 结合，实现纯 `Index Scan` | 虽支持语法，但在复杂关联或子查询嵌套时优化器容易退化为 `Using temporary` 与 `filesort` | PG 在千万级订单表按用户查 Top-N 时耗时稳定在毫秒级，MySQL 容易出现 CPU 飙升。 |
| **表值函数（SRF）结合** | 完美支持 `LATERAL unnest(arr)`、`LATERAL jsonb_to_recordset(...)` | 不支持表值函数（无原生数组和丰富 SRF） | PG 将 LATERAL 广泛应用于非结构化数据解包与行列转换。 |

---

## 9. 高级多维聚合与条件聚合

### 9.1 GROUPING SETS, CUBE 与 ROLLUP

在一次扫描中完成多维度交叉分析报表，免去多次 `UNION ALL` 的全表重复扫描。

```sql
SELECT 
    region,
    category,
    EXTRACT(YEAR FROM sale_date) AS sale_year,
    SUM(amount) AS total_sales,
    GROUPING(region, category, EXTRACT(YEAR FROM sale_date)) AS grp_bitmap
FROM sales_records
GROUP BY ROLLUP (region, category, EXTRACT(YEAR FROM sale_date));
```
- `ROLLUP (a, b, c)`：产生递进层级汇总 `(a, b, c), (a, b), (a), ()`。
- `CUBE (a, b, c)`：产生全组合笛卡尔积汇总（共 $2^3 = 8$ 种组合）。
- `GROUPING SETS ((a), (b, c))`：精确指定所需的特定分组维度集合。

### 9.2 `FILTER (WHERE ...)` 条件聚合

PostgreSQL 原生支持标准 SQL 的 `FILTER` 语法，用于在聚合函数内部指定过滤谓词：

```sql
SELECT 
    dept_id,
    COUNT(*) AS total_employees,
    COUNT(*) FILTER (WHERE gender = 'FEMALE') AS female_count,
    AVG(salary) FILTER (WHERE age < 30) AS avg_salary_young,
    SUM(salary) FILTER (WHERE performance_rating = 'A') AS top_performer_bonus_pool
FROM employee_profiles
GROUP BY dept_id;
```

---

### 9.3 🥊 条件聚合深度对比：PostgreSQL FILTER (WHERE ...) vs MySQL CASE WHEN

#### 核心代码对照

```sql
-- ============================================================================
-- PostgreSQL 17：原生 ANSI SQL FILTER 语法（意图清晰，易读且性能优越）
-- ============================================================================
SELECT 
    dept_id,
    COUNT(*) AS total_staff,
    COUNT(*) FILTER (WHERE status = 'ACTIVE')               AS active_staff,
    AVG(salary) FILTER (WHERE performance = 'S')           AS s_tier_avg_salary,
    COUNT(DISTINCT role_id) FILTER (WHERE is_manager = TRUE) AS manager_roles
FROM employees
GROUP BY dept_id;

-- ============================================================================
-- MySQL 8.x：不支持 FILTER！必须全部使用 CASE WHEN 表达式转换
-- ============================================================================
SELECT 
    dept_id,
    COUNT(*) AS total_staff,
    COUNT(CASE WHEN status = 'ACTIVE' THEN 1 END)          AS active_staff,
    AVG(CASE WHEN performance = 'S' THEN salary END)       AS s_tier_avg_salary,
    COUNT(DISTINCT CASE WHEN is_manager = TRUE THEN role_id END) AS manager_roles
FROM employees
GROUP BY dept_id;
```

#### 关键技术差异分析

1. **SQL 表达直观性**：`COUNT(*) FILTER (WHERE ...)` 直接体现“过滤后计数”的意图；而 MySQL 的 `COUNT(CASE WHEN ... THEN 1 END)` 依赖 `COUNT` 忽略 NULL 的副作用，若误写为 `SUM(CASE WHEN ... THEN 1 ELSE 0 END)` 则语义容易混乱。
2. **执行引擎开销**：PostgreSQL 引擎在计算带 `FILTER` 的聚合函数时，底层直接对不满足谓词的行跳过聚合累加状态机的调用；MySQL 必须为每一行计算 `CASE WHEN` 标量表达式并返回 NULL 后再传递给聚合函数，存在额外的表达式求值开销。

---

## 10. 四大业务实战场景

### 场景 1：多级电商类目/组织架构树查询与完整路径生成

**需求**：已知品类表包含父子层级关系，要求查询指定根类目（或全量）的完整树形结构、层级深度、全路径面包屑（如 `数码 > 电脑 > 笔记本`）并检测潜在死循环。

```sql
-- 1. 表结构与数据初始化
CREATE TABLE categories (
    category_id INT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    parent_id INT REFERENCES categories(category_id)
);

INSERT INTO categories VALUES
  (1, '电子数码', NULL),
  (2, '电脑办公', 1),
  (3, '手机通讯', 1),
  (4, '笔记本电脑', 2),
  (5, '游戏本', 4),
  (6, '智能手机', 3);

-- 2. 递归 CTE 构建路径与层级树
WITH RECURSIVE category_tree AS (
    -- 锚点：顶层根节点
    SELECT 
        category_id,
        name,
        parent_id,
        1 AS level,
        name::TEXT AS full_path,
        ARRAY[category_id] AS path_nodes
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    -- 递归迭代：子节点关联
    SELECT 
        c.category_id,
        c.name,
        c.parent_id,
        ct.level + 1,
        ct.full_path || ' > ' || c.name,
        ct.path_nodes || c.category_id
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.category_id
)
CYCLE category_id SET is_cycle USING cycle_path
SELECT 
    category_id,
    REPEAT('  ', level - 1) || '└── ' || name AS visual_tree,
    level,
    full_path,
    is_cycle
FROM category_tree
ORDER BY path_nodes;
```

---

### 场景 2：电商各品类 Top 3 热销商品实时统计

**需求**：查询全平台 100+ 品类中，每个品类销量最高的 3 款商品详情。

#### 方案 A：窗口函数 `ROW_NUMBER()`
```sql
WITH ranked_products AS (
    SELECT 
        category_id,
        product_id,
        product_name,
        sales_volume,
        ROW_NUMBER() OVER (
            PARTITION BY category_id 
            ORDER BY sales_volume DESC
        ) AS rk
    FROM products
)
SELECT category_id, product_id, product_name, sales_volume
FROM ranked_products
WHERE rk <= 3
ORDER BY category_id, rk;
```

#### 方案 B：`LATERAL` 连接（针对有索引的海量数据极速查询）
```sql
-- 前提条件：存在复合索引 CREATE INDEX idx_prod_cat_sales ON products(category_id, sales_volume DESC);
SELECT 
    c.category_id,
    c.name AS category_name,
    top_p.product_id,
    top_p.product_name,
    top_p.sales_volume
FROM categories c
CROSS JOIN LATERAL (
    SELECT p.product_id, p.product_name, p.sales_volume
    FROM products p
    WHERE p.category_id = c.category_id
    ORDER BY p.sales_volume DESC
    LIMIT 3
) top_p
ORDER BY c.category_id, top_p.sales_volume DESC;
```

> [!TIP]
> **性能对比决胜点**：
> - 当品类数量较少（如数十个），而商品总数巨大（如千万级）时，`LATERAL` 方案通过索引扫描仅需读取 `品类数 * 3` 次索引页，执行时间通常在毫秒级；
> - 而 `ROW_NUMBER()` 需要全表扫描或全索引扫描整个千万级表并进行 WindowAgg 运算，开销随数据量线性增长。

---

### 场景 3：账户资金流水与期末余额计算

**需求**：用户出入金流水并发扣款、记录每笔交易后瞬时余额，并利用 `RETURNING` 实现无锁原子响应。

```sql
-- 1. 表结构
CREATE TABLE account_balance (
    account_id BIGINT PRIMARY KEY,
    current_balance NUMERIC(18, 2) NOT NULL CHECK (current_balance >= 0)
);

CREATE TABLE balance_transactions (
    txn_id BIGSERIAL PRIMARY KEY,
    account_id BIGINT NOT NULL,
    amount NUMERIC(18, 2) NOT NULL, -- 正为入金，负为扣款
    balance_after NUMERIC(18, 2) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 2. 资金扣减与流水生成原子管道
WITH deduction AS (
    UPDATE account_balance
    SET current_balance = current_balance - 150.00
    WHERE account_id = 8888 AND current_balance >= 150.00
    RETURNING account_id, current_balance AS new_balance
)
INSERT INTO balance_transactions (account_id, amount, balance_after)
SELECT account_id, -150.00, new_balance
FROM deduction
RETURNING txn_id, account_id, amount, balance_after, created_at;

-- 3. 历史对账与运行累计余额重算（对账核验）
SELECT 
    txn_id,
    account_id,
    amount,
    balance_after AS recorded_balance,
    -- 利用窗口函数滑动重构流水余额
    SUM(amount) OVER (
        PARTITION BY account_id 
        ORDER BY txn_id ASC
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS calculated_running_balance
FROM balance_transactions
WHERE account_id = 8888;
```

---

### 场景 4：多渠道库存同步（PG17 MERGE + RETURNING 批量合并实战）

**需求**：电商系统每分钟接收来自外部 ERP 的全量/增量库存变动包。若 SKU 存在则更新可用库存并刷新时间；若不存在则新增 SKU；若 ERP 标记该 SKU 已下架则软删除或移出，且实时返回处理动作。

```sql
-- 目标库存表
CREATE TABLE warehouse_inventory (
    sku_code VARCHAR(64) PRIMARY KEY,
    stock_qty INT NOT NULL DEFAULT 0,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    last_sync_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ERP 同步批次临时表
CREATE TEMP TABLE erp_sync_batch (
    sku_code VARCHAR(64),
    stock_qty INT,
    action_flag VARCHAR(16) -- 'SYNC', 'DELETE'
);

INSERT INTO erp_sync_batch VALUES
  ('SKU-1001', 80, 'SYNC'),
  ('SKU-1002', 0,  'DELETE'),
  ('SKU-2001', 50, 'SYNC');

-- PostgreSQL 17 原生 MERGE 同步
MERGE INTO warehouse_inventory AS target
USING erp_sync_batch AS source
ON target.sku_code = source.sku_code
-- 分支 1：存在且下架 -> 更新下架状态
WHEN MATCHED AND source.action_flag = 'DELETE' THEN
    UPDATE SET is_active = FALSE, last_sync_at = NOW()
-- 分支 2：存在且正常同步 -> 覆盖库存
WHEN MATCHED AND source.action_flag = 'SYNC' THEN
    UPDATE SET stock_qty = source.stock_qty, is_active = TRUE, last_sync_at = NOW()
-- 分支 3：不存在且正常同步 -> 新增记录
WHEN NOT MATCHED AND source.action_flag = 'SYNC' THEN
    INSERT (sku_code, stock_qty, is_active, last_sync_at)
    VALUES (source.sku_code, source.stock_qty, TRUE, NOW())
RETURNING 
    merge_action() AS executed_action,
    target.sku_code,
    target.stock_qty,
    target.is_active;
```

---

## 11. 与 MySQL 深度对比技术总结矩阵

| 维度 / 特性 | PostgreSQL 17 | MySQL 8.x | 架构影响与工程落地差异 |
| :--- | :--- | :--- | :--- |
| **RETURNING 机制** | 原生完整支持 `INSERT / UPDATE / DELETE / MERGE ... RETURNING` | **完全不支持** | **开发效率与性能巨大差异**：<br>PG 单次交互获取计算结果/生成 ID，网络 RTT 减半；MySQL 必须使用 `LAST_INSERT_ID()`（且仅支持自增主键）或在应用层发起二次 `SELECT`。 |
| **UPSERT 机制** | `ON CONFLICT (cols) DO UPDATE SET ... WHERE ...` | `INSERT ... ON DUPLICATE KEY UPDATE col = expr` | **PG 优势显著**：<br>1. PG 支持指定特定唯一索引或部分索引；<br>2. 支持在 `DO UPDATE` 后加 `WHERE` 谓词实现条件更新；<br>3. 提供 `EXCLUDED` 伪表统一语义。<br>MySQL 容易因多个唯一索引发生意外更新，且无法指定冲突触发的目标索引。 |
| **MERGE 语句** | **完全支持 ANSI SQL MERGE**，并在 PG 17 中新增 `RETURNING` 与 `$action` | **完全不支持 MERGE** | MySQL 只能通过多条 `INSERT ... ON DUPLICATE KEY UPDATE` 模拟部分插入更新逻辑，无法处理条件删除分支。 |
| **CTE 与物化控制** | 支持 `AS MATERIALIZED` / `AS NOT MATERIALIZED`，支持 CTE 内嵌写操作（DML in CTE） | 支持基本只读 CTE，**不支持 DML in CTE**，不支持显式物化控制 | PG 可用一条复杂 CTE 实现流水线跨表操作并保证原子事务；MySQL 必须拆分成存储过程或多个独立事务语句。 |
| **递归防环** | 原生 ANSI SQL `CYCLE` 子句自动防死循环 | 需依靠参数限制深度或手动字符串查找防环 | PG 代码健壮性高，无需担心意外环路导致服务端连接打满。 |
| **窗口函数高级特性** | 支持 `GROUPS` 帧、`EXCLUDE` 子句、并行 WindowAgg | 仅支持基础 `ROWS`/`RANGE`，无 `EXCLUDE`，单线程计算 | PG 在大数据量报表与复杂窗口计算上性能和表达力全面领先。 |
| **LATERAL 连接** | 原生深度优化，完美配合复合索引实现极速 Top-N 跳读 | 8.0.14+ 支持，但优化器在复杂查询下容易退化为临时表排序 | PG 执行计划更智能，Top-N 场景吞吐量优势巨大。 |
| **条件聚合** | 原生支持 `AGG_FUNC() FILTER (WHERE cond)` | 仅支持 `AGG_FUNC(CASE WHEN cond THEN val END)` | PG 的 `FILTER` 表达清晰规范，与窗口函数结合更直观，优化器开销更低。 |

---

## 12. 总结与开发最佳实践

1. **善用 `RETURNING` 替代二次查询**：在所有生成主键、状态流转、扣款更新场景中，强制使用 `RETURNING` 减少应用与数据库之间的交互轮次。
2. **区分 UPSERT 与 MERGE 的适用场景**：
   - 高频单表、高并发单行写入优先使用 `INSERT ... ON CONFLICT`，轻量且锁冲突最小。
   - 跨表批量同步、全量对账、包含删除/多条件分支的复杂 ETL 任务优先使用 PostgreSQL 17 的 `MERGE`。
3. **精准把控 CTE 物化提示**：复杂统计复用结果集时标记 `AS MATERIALIZED` 避免重复执行；需要下推索引条件时标记 `AS NOT MATERIALIZED`。
4. **Top-N 场景优先评估 `LATERAL`**：在主子表 1:N 且子表有合适联合索引时，使用 `LATERAL + LIMIT` 代替全局 `ROW_NUMBER()`。
