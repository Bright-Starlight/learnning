# PostgreSQL 17 官方开发指南（四）：内置函数与操作符全景

---

## 1. 概述与核心导读

PostgreSQL 被誉为“世界上功能最强大的开源关系型数据库”，其在函数与操作符体系的完备性与正交性上独树一帜。除了 ANSI SQL 标准函数外，PostgreSQL 提供了丰富且高度灵活的原生操作符、高级数组/JSON 处理能力以及强大的序列生成器。特别是在 **PostgreSQL 17** 中，**SQL/JSON 标准函数（如 `JSON_TABLE`, `JSON_QUERY`, `JSON_VALUE`, `JSON_EXISTS` 等）** 得到了全面增强与正式落地，使半结构化数据处理能力跃升至全新高度。

本章深度解读 PostgreSQL 官方文档 Part II（SQL 语言）第 9 章，核心模块包含：
- **文本与正则操作符**：`ILIKE`、POSIX 正则符号 (`~`, `~*`, `!~`, `!~*`)、`split_part`、`string_agg`。
- **日期时间与时序计算**：事务时间与物理时间机制（`NOW()` vs `clock_timestamp()`）、`date_trunc` 强类型截断、`AGE()` 与 `INTERVAL` 运算。
- **逻辑控制与序列生成**：`COALESCE`、`NULLIF`、`GREATEST`、`LEAST` 及表值函数 `generate_series`。
- **JSON / JSONB 与 PG17 SQL/JSON 新标准**：核心提取/包含操作符、`jsonb_set`、`jsonb_insert`、JSONPath 与 `JSON_TABLE`。
- **数组与集合聚合**：`unnest`、`array_agg`、`string_agg`、`jsonb_agg` 与布尔聚合 (`bool_and`, `bool_or`)。
- **三大全流程业务实战** 与 **章节内嵌入式 PostgreSQL vs MySQL 8.x 深度技术对比**。

---

## 2. 字符串与高级正则处理

### 2.1 模式匹配与正则操作符

PostgreSQL 提供了远超标准 SQL 的文本匹配体系：

| 操作符 / 关键字 | 作用说明 | 区分大小写 | 索引支持 (GIN / pg_trgm) |
| :--- | :--- | :---: | :---: |
| `LIKE` | 标准通配符匹配 (`%`, `_`) | 是 | B-Tree (仅前缀), GIN (pg_trgm 任意模糊) |
| `ILIKE` | **PostgreSQL 特色**：忽略大小写通配符匹配 | 否 | GIN (pg_trgm 任意模糊) |
| `~` | POSIX 正则表达式匹配 | 是 | GIN (pg_trgm) |
| `~*` | POSIX 正则表达式匹配（忽略大小写） | 否 | GIN (pg_trgm) |
| `!~` | POSIX 正则不匹配 | 是 | - |
| `!~*` | POSIX 正则不匹配（忽略大小写） | 否 | - |
| `SIMILAR TO` | SQL99 标准正则模式匹配 | 是 | - |

```sql
-- 1. ILIKE 忽略大小写查询
SELECT * FROM users WHERE email ILIKE '%@GMAIL.COM';

-- 2. POSIX 正则表达式快速校验格式 (手机号 / 邮箱)
SELECT 
    '13812345678' ~ '^1[3-9]\d{9}$' AS is_valid_phone,
    'USER_Admin@Domain.io' ~* '^[a-z0-9._%+-]+@[a-z0-9.-]+\.[a-z]{2,}$' AS is_valid_email;
```

### 2.2 核心文本拆分与替换函数

```sql
-- 1. split_part(string, delimiter, field_index)
-- 极速提取特定下标分隔的子串（下标从 1 开始）
SELECT split_part('user_10086_vip_beijing', '_', 2); -- 返回 '10086'
SELECT split_part('192.168.1.105', '.', 4);          -- 返回 '105'

-- 2. regexp_replace(string, pattern, replacement [, flags])
-- 高级正则替换（'g' 表示全局替换）
SELECT regexp_replace('Order#2026-08-19 [URGENT]', '[^0-9]', '', 'g'); -- 提取纯数字: '20260819'

-- 3. regexp_matches(string, pattern [, flags])
-- 返回正则捕获组的文本数组
SELECT (regexp_matches('Server IP: 10.20.30.40 Port: 5432', 'IP:\s+([0-9.]+)\s+Port:\s+(\d+)'))[1] AS ip,
       (regexp_matches('Server IP: 10.20.30.40 Port: 5432', 'IP:\s+([0-9.]+)\s+Port:\s+(\d+)'))[2] AS port;
```

---

### 2.3 🥊 模糊匹配与正则对比：PostgreSQL (ILIKE, POSIX 操作符) vs MySQL (LOWER+LIKE, REGEXP)

#### 核心代码对照

##### 场景 A：忽略大小写的模糊匹配

```sql
-- ============================================================================
-- PostgreSQL 17：原生 ILIKE 关键字，配合 pg_trgm 索引可直接走 GIN 索引
-- ============================================================================
SELECT user_id, email FROM users WHERE email ILIKE '%Admin%';

-- ============================================================================
-- MySQL 8.x：LIKE 大小写敏感度取决于字符集 Collation（如 utf8mb4_bin 区分，utf8mb4_general_ci 不区分）
-- 若要强制忽略大小写，通常需套用 LOWER() 函数，导致普通 B-Tree 索引彻底失效
-- ============================================================================
SELECT user_id, email FROM users WHERE LOWER(email) LIKE '%admin%';
```

##### 场景 B：POSIX 正则表达式校验与匹配

```sql
-- ============================================================================
-- PostgreSQL 17：简洁原生操作符 (~, ~*, !~)，支持与 GIN 索引联动
-- ============================================================================
SELECT * FROM products WHERE sku ~ '^[A-Z]{3}-\d{4}$';

-- ============================================================================
-- MySQL 8.x：使用 REGEXP 或 RLIKE 关键字，无法使用常规索引加速
-- ============================================================================
SELECT * FROM products WHERE sku REGEXP '^[A-Z]{3}-[0-9]{4}$';
```

#### 架构与索引支持深度剖析

```
[PostgreSQL pg_trgm GIN 倒排索引]
查询: WHERE name ILIKE '%postgres%'
执行: Bitmap Index Scan on idx_trgm_name
      -> 提取三元组 (pos, ost, stg, tgr, gre, res)
      -> 毫秒级命中索引，直接返回元组指针！

[MySQL 模糊查询执行瓶颈]
查询: WHERE name LIKE '%postgres%'
执行: Table Scan (全表扫描)
      -> 包含前置通配符 '%' 导致 B-Tree 索引完全失效，随数据量增大查询耗时急剧上升。
```

| 对比维度 | PostgreSQL 17 | MySQL 8.x | 生产影响 |
| :--- | :--- | :--- | :--- |
| **模糊查询语法** | 原生 `ILIKE`、`LIKE`、`SIMILAR TO` | `LIKE`（行为受 Collation 约束），无 `ILIKE` | PG 语法意图显式，不受连接级 Collation 隐式行为干扰。 |
| **POSIX 正则操作符** | 提供 `~`, `~*`, `!~`, `!~*` 等一等公民操作符 | 仅有 `REGEXP` / `RLIKE` 关键字 | PG 语法紧凑，在复杂条件表达式中组合更灵活。 |
| **任意模糊匹配索引** | **强大支持**：`CREATE INDEX ... USING gin (col gin_trgm_ops)` 支持前缀/中缀/后缀模糊与正则索引加速 | **不支持**：`%keyword%` 无法走 B-Tree 索引；仅全文索引（FULLTEXT）支持整词搜索，无法支持任意子串匹配 | 在海量文本检索场景下，PG 的 `pg_trgm` 提供了超越 MySQL 的极速检索能力。 |

---

## 3. 日期时间函数与时序算术

### 3.1 时间获取机制（事务时间 vs 物理时间）

```sql
BEGIN;
SELECT NOW(), CURRENT_TIMESTAMP, transaction_timestamp(); -- 事务开启时的时间戳，在整个事务内恒定不变
SELECT statement_timestamp();                              -- 当前 SQL 语句开始执行的时间
SELECT clock_timestamp();                                  -- 物理时钟瞬时时间（随代码执行逐步递增）
SELECT pg_sleep(1);
SELECT NOW(), clock_timestamp();                           -- NOW 保持原值，clock_timestamp 增加了 1 秒
COMMIT;
```

### 3.2 `date_trunc` 极速时间截断与区间抽取

`date_trunc` 是时序统计、按周期聚合报表的利器，可直接将时间截断到年、季度、月、周、日、小时等粒度，并返回标准 `TIMESTAMPTZ`：

```sql
SELECT 
    date_trunc('year', TIMESTAMPTZ '2026-08-19 14:35:20+08')   AS trunc_year,   -- 2026-01-01 00:00:00+08
    date_trunc('month', TIMESTAMPTZ '2026-08-19 14:35:20+08')  AS trunc_month,  -- 2026-08-01 00:00:00+08
    date_trunc('day', TIMESTAMPTZ '2026-08-19 14:35:20+08')    AS trunc_day,    -- 2026-08-19 00:00:00+08
    date_trunc('hour', TIMESTAMPTZ '2026-08-19 14:35:20+08')   AS trunc_hour;   -- 2026-08-19 14:00:00+08

-- EXTRACT / date_part：提取时间分量
SELECT 
    EXTRACT(DOW FROM NOW()) AS day_of_week,     -- 星期几 (0 = 星期日, 1-6 = 周一至周六)
    EXTRACT(ISODOW FROM NOW()) AS iso_dow,      -- ISO 星期几 (1 = 周一, 7 = 周日)
    EXTRACT(EPOCH FROM NOW()) AS unix_timestamp; -- 秒级 Unix 时间戳
```

### 3.3 `INTERVAL` 算术运算与 `AGE()` 计算

```sql
-- 1. INTERVAL 运算
SELECT 
    NOW() + INTERVAL '7 days'           AS next_week,
    NOW() - INTERVAL '3 months 2 hours' AS past_time,
    date_trunc('month', NOW()) + INTERVAL '1 month - 1 day' AS month_end_date;

-- 2. AGE(timestamp, timestamp)：计算人类可读的年龄/时间跨度
SELECT AGE(TIMESTAMP '2026-08-19', TIMESTAMP '1995-03-15'); 
-- 返回: '31 years 5 mons 4 days'
```

---

### 3.4 🥊 日期截断与时钟对比：PostgreSQL (`date_trunc`) vs MySQL (繁琐的 `DATE_FORMAT`)

#### 核心代码对照

##### 场景 A：按月/按周维度聚合销售报表

```sql
-- ============================================================================
-- PostgreSQL 17：date_trunc 原生强类型截断，返回完整 TIMESTAMPTZ，支持时区
-- ============================================================================
SELECT 
    date_trunc('month', created_at) AS order_month,
    COUNT(*) AS total_orders,
    SUM(amount) AS total_amount
FROM orders
GROUP BY date_trunc('month', created_at)
ORDER BY order_month;

-- ============================================================================
-- MySQL 8.x：无 date_trunc 函数！必须通过 DATE_FORMAT 转成字符串中转
-- ============================================================================
SELECT 
    DATE_FORMAT(created_at, '%Y-%m-01 00:00:00') AS order_month,
    COUNT(*) AS total_orders,
    SUM(amount) AS total_amount
FROM orders
GROUP BY DATE_FORMAT(created_at, '%Y-%m-01 00:00:00')
ORDER BY order_month;
```

##### 场景 B：事务时间 vs 瞬时物理时钟

```sql
-- ============================================================================
-- PostgreSQL 17：语义划分极其清晰严格
-- ============================================================================
SELECT NOW();               -- 事务开始时间（整个事务中完全恒定）
SELECT clock_timestamp();   -- 物理时钟瞬时时间（代码逐行执行时实时递增）

-- ============================================================================
-- MySQL 8.x：NOW() 是语句级别开始时间，SYSDATE() 是物理瞬时时间
-- ============================================================================
SELECT NOW();               -- 语句执行开始时刻
SELECT SYSDATE();           -- 物理动态时钟（注意：在基于语句的主从复制中可能导致不一致）
```

#### 关键技术差异分析

1. **类型安全性与时区保留**：PostgreSQL 的 `date_trunc` 输出依然是高精度的 `TIMESTAMPTZ` 类型，可直接参与后续的 `+ INTERVAL` 运算，时区转换（`AT TIME ZONE`）完备；MySQL 通过 `DATE_FORMAT` 将时间降级为 `VARCHAR` 字符串，后续若需参与计算必须再次调用 `STR_TO_DATE`，既容易丢失时区信息，又带来性能损耗。
2. **性能与函数索引**：在 PostgreSQL 中可以直接对 `date_trunc('day', created_at)` 建立表达式 B-Tree 索引；而在 MySQL 中虽然可以建虚拟生成列，但字符串索引的存储开销与比较成本远高于 PG 的 8 字节整数时间戳。

---

## 4. 数学、逻辑控制与 `generate_series`

### 4.1 核心逻辑与标量函数

- `COALESCE(val1, val2, ...)`：返回参数列表中第一个非 NULL 值。
- `NULLIF(val1, val2)`：若 `val1 = val2` 返回 NULL，否则返回 `val1`（常用于除零保护：`x / NULLIF(y, 0)`）。
- `GREATEST(v1, v2, ...)` / `LEAST(v1, v2, ...)`：求一组值中的最大值 / 最小值（跳过 NULL）。

```sql
-- 除零安全保护与默认值保底
SELECT 
    total_revenue,
    total_clicks,
    total_revenue / NULLIF(total_clicks, 0) AS revenue_per_click,
    COALESCE(discount_rate, 1.00) AS effective_discount
FROM campaign_stats;
```

### 4.2 `generate_series`：强大的序列与连续数据集生成器

`generate_series` 是 PostgreSQL 最具特色的集合返回函数（Set-Returning Function, SRF）之一，支持生成整数、数值和日期时间区间的连续集合。

```sql
-- 1. 生成整数序列（常用于批量造数、笛卡尔拆分）
SELECT generate_series(1, 5) AS id; -- 输出 1, 2, 3, 4, 5

-- 2. 生成连续日期间隔（时序补零核心）
SELECT generate_series(
    '2026-08-01'::DATE, 
    '2026-08-07'::DATE, 
    INTERVAL '1 day'
)::DATE AS calendar_date;
```

---

### 4.3 🥊 连续序列生成对比：PostgreSQL 原生 `generate_series` vs MySQL 需自建数字表/递归 CTE

#### 核心代码对照

##### 需求：生成 2026-08-01 到 2026-08-07 连续 7 天的日期序列，用于报表左连接补零。

```sql
-- ============================================================================
-- PostgreSQL 17：单行表值函数直接搞定，优雅高效
-- ============================================================================
SELECT generate_series('2026-08-01'::DATE, '2026-08-07'::DATE, INTERVAL '1 day')::DATE AS dt;

-- ============================================================================
-- MySQL 8.x：完全无原生 generate_series 函数！必须依赖递归 CTE 或物理数字表
-- ============================================================================
WITH RECURSIVE date_seq AS (
    SELECT CAST('2026-08-01' AS DATE) AS dt
    UNION ALL
    SELECT dt + INTERVAL 1 DAY
    FROM date_seq
    WHERE dt < '2026-08-07'
)
SELECT dt FROM date_seq;
```

#### 架构与实现成本对比

| 对比维度 | PostgreSQL 17 (`generate_series`) | MySQL 8.x (递归 CTE / 辅助表) |
| :--- | :--- | :--- |
| **语法优雅度** | 原生单行表值函数，支持步长（Step）、多类型重载（Int/BigInt/Numeric/Timestamp） | 必须书写 8 行以上的递归 CTE，或提前维护一张存有 0~100000 的物理 `t_digits` 辅助表 |
| **内存与执行开销** | 引擎内核流式生成 Tuple（On-the-fly），内存开销恒定为 $O(1)$ | 递归 CTE 需要在内存建立临时表存储工作集，受到 `cte_max_recursion_depth`（默认 1000）深度限制 |
| **多序列交叉笛卡尔积** | 支持在 `FROM` 子句中轻松 `CROSS JOIN generate_series(...)` | 编写极其冗长繁琐 |

---

## 5. JSON / JSONB 与 PG17 SQL/JSON 标准

PostgreSQL 拥有两类 JSON 类型：
- `json`：纯文本存储，保留原始格式与空格，写入快但每次解析慢。
- `jsonb`：二进制分解存储，自动去除无意义空格和重复 Key，支持 **GIN 倒排索引**，查询与修改性能极高（生产实践推荐统一使用 `jsonb`）。

### 5.1 JSONB 核心操作符速查

| 操作符 | 返回类型 | 说明 | 示例 |
| :--- | :--- | :--- | :--- |
| `->` | `jsonb` | 按键名或数组下标提取对象（返回 JSONB 类型） | `'{"a": {"b": 1}}'::jsonb -> 'a'` |
| `->>` | `text` | 按键名或数组下标提取对象（返回纯文本类型） | `'{"name": "Alice"}'::jsonb ->> 'name'` |
| `#>` | `jsonb` | 按路径数组提取深层嵌套对象 | `'{"a": {"b": [10, 20]}}'::jsonb #> '{a,b,1}'` (返回 `20`) |
| `#>>` | `text` | 按路径数组提取深层嵌套文本 | `'{"a": {"b": [10, 20]}}'::jsonb #>> '{a,b,1}'` (返回 `'20'`) |
| `@>` | `boolean` | **包含判断**（左侧是否包含右侧 JSON 结构，支持 GIN 索引） | `'{"tags": ["dev", "db"]}'::jsonb @> '{"tags": ["db"]}'` |
| `<@` | `boolean` | **被包含判断**（左侧是否被包含于右侧中） | `'{"a": 1}'::jsonb <@ '{"a": 1, "b": 2}'::jsonb` |
| `?` | `boolean` | 顶层字符串键名是否存在 | `'{"a": 1, "b": 2}'::jsonb ? 'a'` |
| `?\|` | `boolean` | 给定文本数组中的任一键名是否存在 | `'{"a": 1}'::jsonb ?\| array['a', 'c']` |
| `?&` | `boolean` | 给定文本数组中的所有键名是否均存在 | `'{"a": 1, "b": 2}'::jsonb ?& array['a', 'b']` |
| `-` / `#-` | `jsonb` | 剔除指定键名或指定路径节点 | `'{"a": 1, "b": 2}'::jsonb - 'a'` |

### 5.2 动态更新函数：`jsonb_set` 与 `jsonb_insert`

```sql
-- 1. jsonb_set(target, path, new_value [, create_missing])
-- 修改深层字段：将 config.settings.theme 修改为 "dark"
UPDATE user_profiles
SET config = jsonb_set(config, '{settings,theme}', '"dark"'::jsonb, true)
WHERE user_id = 1001;

-- 2. jsonb_insert(target, path, new_value [, insert_after])
-- 向 JSON 数组指定位置插入元素
SELECT jsonb_insert('{"tags": ["pg", "sql"]}'::jsonb, '{tags, 1}', '"cloud"'::jsonb, false);
-- 结果: {"tags": ["pg", "cloud", "sql"]}
```

### 5.3 PostgreSQL 17 SQL/JSON 标准特性全景

PostgreSQL 17 实现了 ANSI SQL/JSON 标准的完整支持，包括 `JSON_TABLE`, `JSON_QUERY`, `JSON_VALUE`, `JSON_EXISTS` 以及高级 JSONPath 语法：

```sql
-- 1. JSONPath 谓词匹配 (jsonb_path_exists, jsonb_path_query)
SELECT jsonb_path_query(
    '{"orders": [{"id": 1, "amount": 150}, {"id": 2, "amount": 50}]}'::jsonb,
    '$.orders[*] ? (@.amount > 100).id'
); -- 输出: 1

-- 2. PG 17 新增：JSON_TABLE（将 JSON 文档直接映射为关系型二维表）
SELECT jt.*
FROM json_test_table t,
     JSON_TABLE(
         t.raw_payload,
         '$.items[*]' COLUMNS (
             item_id INT PATH '$.id',
             item_name VARCHAR(50) PATH '$.name',
             price NUMERIC(10,2) PATH '$.price',
             is_in_stock BOOLEAN PATH '$.in_stock'
         )
     ) AS jt;
```

---

### 5.4 🥊 JSON 操作符与 SQL/JSON 对比：PostgreSQL (丰富操作符, JSONB + GIN) vs MySQL (JSON_EXTRACT, `->>`)

#### 核心代码对照

##### 场景 A：嵌套字段提取与包含查询

```sql
-- ============================================================================
-- PostgreSQL 17：操作符直观简洁，支持通过 GIN 索引加速包含查询 (@>)
-- ============================================================================
-- 1. 提取嵌套文本
SELECT doc #>> '{user, profile, name}' FROM app_logs;

-- 2. 包含检索（命中 GIN 倒排索引）
SELECT * FROM app_logs WHERE doc @> '{"tags": ["urgent"], "status": "FAIL"}';

-- ============================================================================
-- MySQL 8.x：依赖函数或有限操作符，包含查询需使用 JSON_CONTAINS
-- ============================================================================
-- 1. 提取嵌套文本
SELECT JSON_UNQUOTE(JSON_EXTRACT(doc, '$.user.profile.name')) FROM app_logs;
-- 或简写为: SELECT doc->>'$.user.profile.name' FROM app_logs;

-- 2. 包含检索（无法直接通过常规 B-Tree 索引对整个 JSON 包含加速）
SELECT * FROM app_logs 
WHERE JSON_CONTAINS(doc, '["urgent"]', '$.tags') AND JSON_EXTRACT(doc, '$.status') = 'FAIL';
```

##### 场景 B：深层局部字段动态更新

```sql
-- ============================================================================
-- PostgreSQL 17：jsonb_set 路径更新
-- ============================================================================
UPDATE user_configs 
SET config = jsonb_set(config, '{ui, theme}', '"dracula"'::jsonb) 
WHERE id = 10;

-- ============================================================================
-- MySQL 8.x：JSON_SET 函数更新
-- ============================================================================
UPDATE user_configs 
SET config = JSON_SET(config, '$.ui.theme', 'dracula') 
WHERE id = 10;
```

#### 内核存储与架构对比

| 对比维度 | PostgreSQL 17 | MySQL 8.x | 架构差异与影响 |
| :--- | :--- | :--- | :--- |
| **数据类型与存储** | 提供 `json`（纯文本）与 `jsonb`（分解二进制格式），自动去重去空 | 单一 `JSON` 类型（内部也是二进制编码存储） | PG 的 `jsonb` 是真正的一等公民，存储紧凑且支持海量专用函数。 |
| **索引能力** | **完美支持 GIN 倒排索引**（`CREATE INDEX ... USING gin (col)`），对 JSON 内部任意字段检索均可走索引 | 只能通过创建“虚拟生成列 + B-Tree 索引”或 Multi-Valued Index（8.0.17+ 针对数组） | PG 一个 GIN 索引即可覆盖 JSON 内所有字段的检索；MySQL 必须为每个待查询 key 显式建生成列索引。 |
| **操作符丰富度** | 提供 `->`, `->>`, `#>`, `#>>`, `@>`, `<@`, `?`, `?|`, `?&`, `-`, `#-` 等 10+ 种原生操作符 | 仅提供 `->` 和 `->>`，其余全依赖 `JSON_XXX` 函数调用 | PG 代码表达紧凑高效，与 SQL 查询管道高度融合。 |
| **SQL/JSON 标准 (`JSON_TABLE`)** | **PG 17 完整落地 ANSI SQL/JSON 标准**（`JSON_TABLE`, `JSON_QUERY` 等） | MySQL 8.0 早先引入了 `JSON_TABLE` | 两者在最新版本中均支持了将 JSON 文档关系化拍平为二维表的标准语法。 |

---

## 6. 数组与聚合函数

### 6.1 数组核心操作与 `unnest`

```sql
-- 1. 数组追加与拼接
SELECT 
    ARRAY[1, 2] || 3                 AS arr_concat,      -- {1,2,3}
    array_append(ARRAY[1, 2], 3)     AS arr_append,      -- {1,2,3}
    array_cat(ARRAY[1, 2], ARRAY[3, 4]) AS arr_merge;   -- {1,2,3,4}

-- 2. unnest：将数组元素展开为独立行（扁平化）
SELECT unnest(ARRAY['PostgreSQL', 'Rust', 'Go']) AS skill;
-- 输出 3 行

-- 3. unnest WITH ORDINALITY：展开并保留行序号
SELECT * FROM unnest(ARRAY['High', 'Medium', 'Low']) WITH ORDINALITY AS t(priority, priority_level);
```

### 6.2 常用高级聚合函数

```sql
SELECT 
    dept_id,
    -- 字符串拼接聚合
    string_agg(emp_name, ', ' ORDER BY salary DESC) AS top_earners,
    -- 数组聚合
    array_agg(emp_id ORDER BY hire_date ASC) AS employee_id_list,
    -- JSONB 数组聚合
    jsonb_agg(jsonb_build_object('id', emp_id, 'name', emp_name)) AS employee_json_list,
    -- 布尔聚合（全真 / 任一真）
    bool_and(is_active) AS all_active,
    bool_or(has_admin_role) AS any_admin
FROM employees
GROUP BY dept_id;
```

---

### 6.3 🥊 数组与高级聚合对比：PostgreSQL (一等公民数组 + 丰富聚合) vs MySQL (缺失数组与繁琐替代)

#### 核心特性代码对照

##### 场景 A：字符串聚合与排序（PostgreSQL `string_agg` vs MySQL `GROUP_CONCAT`）

```sql
-- ============================================================================
-- PostgreSQL 17：string_agg 语法清晰，无隐式截断限制
-- ============================================================================
SELECT dept_id, string_agg(emp_name, ', ' ORDER BY salary DESC) AS staff_list
FROM employees
GROUP BY dept_id;

-- ============================================================================
-- MySQL 8.x：使用 GROUP_CONCAT，存在著名的默认 1024 字节截断大坑！
-- ============================================================================
-- 注意：若拼接字符串超过 group_concat_max_len（默认 1024 字节），数据会被静默截断！
SET SESSION group_concat_max_len = 1048576; -- 必须显式调大参数保底
SELECT dept_id, GROUP_CONCAT(emp_name ORDER BY salary DESC SEPARATOR ', ') AS staff_list
FROM employees
GROUP BY dept_id;
```

##### 场景 B：将分组明细聚合为结构化数组 / 列表

```sql
-- ============================================================================
-- PostgreSQL 17：array_agg 原生收集为强类型数组，支持直接下标与数组操作符
-- ============================================================================
SELECT dept_id, array_agg(emp_id) AS id_list
FROM employees
GROUP BY dept_id;

-- ============================================================================
-- MySQL 8.x：无原生数组！必须聚合为 JSON 数组或逗号字符串
-- ============================================================================
SELECT dept_id, JSON_ARRAYAGG(emp_id) AS id_list
FROM employees
GROUP BY dept_id;
```

##### 场景 C：布尔条件聚合（PostgreSQL `bool_and` / `bool_or` vs MySQL `MIN`/`MAX` 替代）

```sql
-- ============================================================================
-- PostgreSQL 17：原生布尔聚合，意图极其明确
-- ============================================================================
SELECT 
    project_id, 
    bool_and(task_completed) AS is_all_finished, -- 全真则真
    bool_or(is_blocked)       AS has_any_blocker  -- 任一真则真
FROM project_tasks
GROUP BY project_id;

-- ============================================================================
-- MySQL 8.x：无 bool 聚合函数！必须借助数值转型与 MIN / MAX 模拟
-- ============================================================================
SELECT 
    project_id,
    MIN(task_completed) = 1 AS is_all_finished,
    MAX(is_blocked) = 1     AS has_any_blocker
FROM project_tasks
GROUP BY project_id;
```

#### 关键技术差异分析

1. **原生数组支持**：PostgreSQL 将数组设计为一等公民数据类型，支持任意维度的数值、字符串与复合类型数组，配合 `unnest()` 和 `array_agg()` 可轻松完成集合与行集的相互转换；MySQL 没有原生数组类型，处理多选标签等场景时被迫在“创建多对多中间表”与“存储 JSON/逗号串”之间妥协。
2. **GROUP_CONCAT 截断陷阱**：MySQL 的 `GROUP_CONCAT` 存在系统变量 `group_concat_max_len` 限制，超长数据会被**静默截断且不报错**，在生产报表和导出中是臭名昭著的隐患；PostgreSQL 的 `string_agg` 基于内存动态扩展缓冲，无此类人为硬编码截断限制。

---

## 7. 三大业务实战场景

### 场景 1：业务报表按天统计并用 `generate_series` 自动补全缺失日期（0 销售额填充）

**需求**：统计最近 7 天每天的订单数与销售总额。若某天无销售，报表仍需展示该日期且销售额记为 `0.00`。

```sql
-- 1. 测试数据表
CREATE TABLE sales_orders (
    order_id SERIAL PRIMARY KEY,
    amount NUMERIC(10,2) NOT NULL,
    order_date DATE NOT NULL
);

INSERT INTO sales_orders (amount, order_date) VALUES 
  (100.00, '2026-08-15'),
  (250.50, '2026-08-15'),
  (80.00,  '2026-08-17'),
  (300.00, '2026-08-19'); -- 16日与18日无数据

-- 2. 利用 generate_series 与 LEFT JOIN 补零查询
WITH date_dimension AS (
    SELECT generate_series(
        '2026-08-14'::DATE, 
        '2026-08-20'::DATE, 
        INTERVAL '1 day'
    )::DATE AS report_date
)
SELECT 
    d.report_date,
    COUNT(s.order_id) AS total_orders,
    COALESCE(SUM(s.amount), 0.00) AS total_sales_amount
FROM date_dimension d
LEFT JOIN sales_orders s ON d.report_date = s.order_date
GROUP BY d.report_date
ORDER BY d.report_date ASC;
```

---

### 场景 2：复杂嵌套 JSON 数据解析、提取与动态字段修改（jsonb_set / JSONPath）

**需求**：电商平台订单扩展信息 `metadata` 为非结构化 JSONB，包含收件人、支付渠道及多层级物流包裹信息。要求：
1. 提取首个包裹的快递单号；
2. 动态更新物流状态为 `DELIVERED` 并写入当前送达时间。

```sql
-- 1. 表结构与数据
CREATE TABLE customer_orders (
    order_id INT PRIMARY KEY,
    metadata JSONB NOT NULL
);

INSERT INTO customer_orders VALUES (
    9001,
    '{
        "customer": {"name": "Bob", "level": "VIP"},
        "packages": [
            {"tracking_no": "SF1001", "status": "IN_TRANSIT", "carrier": "SF-Express"},
            {"tracking_no": "SF1002", "status": "PENDING", "carrier": "SF-Express"}
        ],
        "payment": {"method": "ALIPAY", "installment": 3}
    }'::jsonb
);

-- 2. 解析首个包裹的快递单号与承运商
SELECT 
    order_id,
    metadata #>> '{customer,name}' AS customer_name,
    metadata #>> '{packages,0,tracking_no}' AS first_tracking_no,
    jsonb_path_query(metadata, '$.packages[*] ? (@.status == "IN_TRANSIT").tracking_no') AS in_transit_pack
FROM customer_orders
WHERE order_id = 9001;

-- 3. 动态更新首个包裹状态为 DELIVERED 并新增 delivery_time
UPDATE customer_orders
SET metadata = jsonb_set(
    jsonb_set(metadata, '{packages,0,status}', '"DELIVERED"'::jsonb),
    '{packages,0,delivered_at}', 
    to_jsonb(to_char(NOW(), 'YYYY-MM-DD"T"HH24:MI:SS"Z"'))
)
WHERE order_id = 9001
RETURNING order_id, metadata #> '{packages,0}' AS updated_package;
```

---

### 场景 3：用户多选标签通过 `unnest` 和 `string_agg` 转换与聚合

**需求**：用户画像表存储用户标签数组 `VARCHAR[]`。
1. 统计全平台最热门的 Top 3 标签及对应用户量；
2. 为指定用户群将其标签转换为竖线分隔的格式化大写字符串。

```sql
-- 1. 表结构与数据
CREATE TABLE user_tags (
    user_id INT PRIMARY KEY,
    tags VARCHAR(50)[] NOT NULL
);

INSERT INTO user_tags VALUES 
  (1, ARRAY['developer', 'database', 'postgres']),
  (2, ARRAY['designer', 'ui', 'frontend']),
  (3, ARRAY['developer', 'golang', 'postgres']),
  (4, ARRAY['manager', 'developer']);

-- 2. 统计全平台标签热度排行 (unnest 扁平化统计)
SELECT 
    tag,
    COUNT(*) AS user_count
FROM (
    SELECT unnest(tags) AS tag FROM user_tags
) sub
GROUP BY tag
ORDER BY user_count DESC, tag ASC
LIMIT 3;

-- 3. 标签重构与格式化合并 (string_agg)
SELECT 
    user_id,
    string_agg(UPPER(t.tag), ' | ' ORDER BY t.tag) AS formatted_tag_bar
FROM user_tags,
     LATERAL unnest(tags) AS t(tag)
GROUP BY user_id
ORDER BY user_id;
```

---

## 8. 与 MySQL 深度对比技术总结矩阵

| 维度 / 特性 | PostgreSQL 17 | MySQL 8.x | 架构影响与工程落地差异 |
| :--- | :--- | :--- | :--- |
| **模糊匹配与正则** | 原生 `ILIKE`、POSIX 正则操作符 (`~`, `~*`, `!~`)，结合 `pg_trgm` GIN 索引可实现超高速模糊匹配 | 仅支持 `LIKE`（区分大小写取决于 Collation）、`REGEXP`，无直接操作符；全文索引或前缀模糊匹配扩展难度大 | **PG 开发体验与索引极具优势**：<br>PG 可以为 `ILIKE '%abc%'` 创建 GIN 倒排三元组索引，避免全表扫描。 |
| **日期时间截断** | `date_trunc('day', col)`、`date_trunc('month', col)` 统一简洁且返回标准 `TIMESTAMPTZ` 类型 | 需依赖 `DATE_FORMAT(col, '%Y-%m-01')` 字符串拼接或 `STR_TO_DATE` 转换 | PG 保持强类型计算，性能更高且天然保留时区信息；MySQL 字符串中转存在隐式转换与精度丢失风险。 |
| **时钟与事务时间** | 明确区分 `NOW()`（事务开始时间）与 `clock_timestamp()`（物理瞬时时钟） | `NOW()` / `CURRENT_TIMESTAMP()` 均为语句开始时间，`SYSDATE()` 为物理时间 | PG 的语义划分更加严格，方便进行高精度代码耗时打点或长时间事务中的真实时间追踪。 |
| **序列生成器** | 原生表函数 `generate_series(start, stop, step)`，支持整数、数值与日期 | **无原生函数**。必须依赖辅助数字物理表、递归 CTE 生成或应用层循环生成 | 时序报表补零场景中，PG 单条 SQL 即可完成优雅补齐，MySQL 代码冗长繁琐。 |
| **JSON 支持深度** | 拥有 `json` 与 `jsonb`，丰富的操作符（`->`, `->>`, `#>`, `@>`, `?` 等），支持 GIN 索引与 **PG 17 SQL/JSON 标准 (`JSON_TABLE`)** | 仅有单一 `JSON` 类型，操作符仅支持 `->` 和 `->>`，主要依赖 `JSON_EXTRACT`、`JSON_SET` 等函数 | PG 的 JSONB 是真正的一等公民，可作为高性能 NoSQL 替代方案，具备深度索引与丰富操作符。 |
| **数组与聚合操作** | 原生支持一维及多维数组类型、`unnest`、`array_agg`、`string_agg`、`bool_and` | **不支持原生数组类型**，字符串聚合为 `GROUP_CONCAT`（有最大长度截断风险） | PG 可直接使用数组字段存储标签、权限 ID 集合，简化多对多关系设计，聚合无长度硬截断。 |

---

## 9. 总结与开发最佳实践

1. **时序报表统计优先采用 `date_trunc` + `generate_series`**：构建时间连续维度并左关联业务表，消除应用层在内存中回填 0 值的复杂逻辑。
2. **JSON 数据存储一律选 `jsonb`**：享受高效二进制编码、去重与 GIN 索引加速；需要多层级解析或行列转换时，优先评估 PostgreSQL 17 的 `JSON_TABLE`。
3. **模糊搜索与正则场景配合 `pg_trgm` 插件**：利用 GIN 索引加速 `ILIKE` 与正则匹配，避免随数据膨胀导致的全表扫描。
4. **多对多轻量标签采用 Array + unnest**：对于小规模多值属性（如用户标签、角色列表），使用数组字段配合 `unnest` / `array_agg` 往往比维护中间关联表更轻量高效。
