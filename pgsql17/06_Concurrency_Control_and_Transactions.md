# 第 06 章：并发控制与事务机制 (Concurrency Control and Transactions)

> 本章对应 PostgreSQL 17 官方文档 **Part II. The SQL Language** 中第 13 章（Concurrency Control）。
> 深入剖析 PostgreSQL 多版本并发控制（MVCC）底层原理、事务隔离级别行为边界、行级/表级/咨询锁机制，并在各章节中深度对比 PostgreSQL 与 MySQL (InnoDB) 的底层机制与代码实现，辅以高并发秒杀、任务派发与分布式锁生产实战。

---

## 1. 事务隔离级别与 PostgreSQL 实际行为

SQL 标准定义了 4 个事务隔离级别，但在 PostgreSQL 的工程实现中，**Read Uncommitted 在底层直接等同于 Read Committed**（PostgreSQL 永远不会读到未提交的脏数据）。

```
               ┌────────────────────────────────────────────────────────┐
               │           PostgreSQL 事务隔离级别与异常现象对照矩阵       │
               └────────────────────────────────────────────────────────┘

    隔离级别 (Isolation Level)      脏读 (Dirty Read)     不可重复读 (Non-Repeatable)   幻读 (Phantom Read)     序列化异常 (Serialization)
   ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
    Read Uncommitted (底层同RC)         ❌ 不可能                ⚠️ 可能                   ⚠️ 可能                    ⚠️ 可能
    Read Committed (默认级别)           ❌ 不可能                ⚠️ 可能                   ⚠️ 可能                    ⚠️ 可能
    Repeatable Read (快照读)           ❌ 不可能                ❌ 不可能                 ❌ 不可能 (PG原生特性)     ⚠️ 可能 (Write Skew)
    Serializable (SSI 快照隔离)        ❌ 不可能                ❌ 不可能                 ❌ 不可能                  ❌ 自动拦截并报错 40001
```

### 1.1 Read Committed（语句级快照，默认）
- **行为机制**：事务中的**每条 SQL 语句执行开始时**，都会获取一个新的活跃事务快照（Snapshot）。
- **并发表现**：如果事务 T1 读完数据后，事务 T2 提交了修改，T1 再次执行相同 SELECT 时将读取到 T2 提交后的新数据。
- **并发写冲突（EvalPlanQual 机制）**：当执行 `UPDATE`/`DELETE` 遇到被其他事务锁定的行时，当前事务会等待；一旦对方提交，当前事务会在最新的行版本上重新评估 `WHERE` 条件，条件依然满足则执行更新。

### 1.2 Repeatable Read（事务级快照）
- **行为机制**：在**事务内第一条非事务控制语句执行时**获取一个事务级快照，并在整个事务生命周期内一直复用该快照。
- **原生无幻读（No Phantom Read）**：在 PostgreSQL 中，Repeatable Read 通过一致性数据快照**天然解决了幻读**问题，完全不需要依赖类似 MySQL InnoDB 的间隙锁（Gap Lock / Next-Key Lock）。
- **写冲突报错（First-Committer-Wins）**：若 T1 试图更新一行已被 T2（后于 T1 快照开始但先于 T1 提交）更新过的记录，T1 将直接报错：
  `ERROR: could not serialize access due to concurrent update`（应用层需捕获重试）。

### 1.3 Serializable（可串行化快照隔离 - SSI）
PostgreSQL 实现了学术界前沿的 **Serializable Snapshot Isolation (SSI)** 算法（基于 Michael Cahill 论文）：
- **零加锁读开销**：不需要像传统数据库（如 MySQL 2PL）那样加昂贵的表级/行级读锁（S-Lock）或谓词锁。
- **SIREAD 锁与依赖图检测**：在内存中为读取的对象维护轻量的 `SIREAD` 标记，动态追踪事务间的读写反向依赖（rw-antidependency）。一旦发现有向图中可能形成环（危险结构），立即中止其中一个事务并抛出错误码 **`40001` (`serialization_failure`)**。
- **写偏序（Write Skew）防护**：彻底杜绝经典的医生值班问题、余额透支问题。

```sql
-- 开启 Serializable 事务的标准范式
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- 业务逻辑查询与更新...
SELECT sum(balance) FROM user_accounts WHERE family_id = 100;
UPDATE user_accounts SET balance = balance - 100 WHERE account_id = 1;

COMMIT;
-- 若捕获 SQLSTATE 40001，客户端自动发起指数退避重试
```

---

#### 🥊 隔离级别行为对比：PG (RR 级别无幻读、SSI 串行化) vs MySQL (RR 级别依赖 Next-Key 锁防幻读)

| 对比维度 | PostgreSQL 17 | MySQL 8.x (InnoDB) | 深度原理解析与架构差异 |
| :--- | :--- | :--- | :--- |
| **默认事务隔离级别** | **Read Committed (读已提交)** | **Repeatable Read (可重复读)** | PG 偏向高并发吞吐与写冲突快速放行；MySQL 偏向传统主从复制 (Binlog Statement 格式历史包袱) 的一致性保证。 |
| **RR 级别下对幻读的处理** | **快照读与所有读原生无幻读**，纯 MVCC 快照控制，**无间隙锁开销** | **快照读无幻读，但“当前读 (Current Read)”必须依赖 Next-Key 锁（行锁 + 间隙锁）** | MySQL 在 `SELECT ... FOR UPDATE` 或 `UPDATE` 时锁住范围间隙，极易引发难以排查的死锁；PG 无间隙锁，并发写入性能显著更高。 |
| **RR 级别写冲突策略** | **First-Committer-Wins**：后提交者直接报错 `could not serialize access` 快速失败 | **锁等待（Lock Wait）**：后提交者阻塞等待前事务释放锁，可能出现覆盖更新或死锁 | PG 保证绝对一致性，强制应用层重试；MySQL 采用等待与覆盖策略。 |
| **Serializable 串行化实现** | **SSI（可串行化快照隔离）**：全并发无锁读取，内存图算法检测冲突，开销极低 | **2PL（两阶段锁）**：将所有普通 `SELECT` 隐式强制升级为 `SELECT ... FOR SHARE` 读锁 | PG Serializable 依然拥有极高的读并发吞吐；MySQL 读写严重互相阻塞。 |

##### 典型时序对照：RR 级别下的幻读与锁行为

```
【场景】：两个并发事务，T1 查询 id BETWEEN 10 AND 20 的用户，T2 插入 id=15 的新用户。

🐘 PostgreSQL 17 (RR 隔离级别):
  T1: BEGIN ISOLATION LEVEL REPEATABLE READ;
  T1: SELECT * FROM users WHERE id BETWEEN 10 AND 20; -- [返回 2 行]
  T2: BEGIN;
  T2: INSERT INTO users (id, name) VALUES (15, 'Tom'); -- ✅ 瞬间插入成功！完全不被 T1 阻塞！
  T2: COMMIT;
  T1: SELECT * FROM users WHERE id BETWEEN 10 AND 20; -- [依然返回 2 行，完美快照隔离，无幻读]
  T1: COMMIT;
  -- 结果：并发写入吞吐极高，读写完全互不阻塞。

🐬 MySQL 8.0 (InnoDB RR 隔离级别当前读):
  T1: BEGIN;
  T1: SELECT * FROM users WHERE id BETWEEN 10 AND 20 FOR UPDATE; -- 加 (10, 20) Next-Key 间隙锁
  T2: BEGIN;
  T2: INSERT INTO users (id, name) VALUES (15, 'Tom'); -- ❌ 阻塞等待！被 T1 的 Gap Lock 卡死！
  -- 若 T1 耗时较长，T2 将直接报 ERROR 1205 (HY000): Lock wait timeout exceeded!
```

---

## 2. MVCC 多版本并发控制底层实现

PostgreSQL 的 MVCC 与 MySQL/Oracle 有根本性的设计差异：**版本链存放在数据堆表（Heap Table）本身中，而非回滚段（Undo Log）中**。

```
 数据页 (Heap Page) 内部 Tuple 物理结构
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ Page Header                                                                            │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ Line Pointer 1 ──┐                                                                     │
│ Line Pointer 2 ─┐│                                                                     │
├─────────────────┼┼─────────────────────────────────────────────────────────────────────┤
│                 │▼ [Tuple Version 1 (旧版本)]                                           │
│                 │  xmin: 100 (创建TX) | xmax: 105 (删除/更新TX) | t_ctid: (0, 2) ───┐   │
│                 │  Data: { id: 1, name: 'Alice', balance: 100 }                     │   │
│                 │                                                                   │   │
│                 ▼ [Tuple Version 2 (新版本, HOT链更新)] <────────────────────────────┘   │
│                   xmin: 105 (创建TX) | xmax: 0   (当前有效)   | t_ctid: (0, 2)          │
│                   Data: { id: 1, name: 'Alice', balance: 200 }                          │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

### 2.1 隐藏系统列深度解析

每个 Heap Tuple 的 Header 中都包含关键元数据系统列：

| 隐藏列 | 类型 | 核心作用与可见性判定逻辑 |
| :--- | :--- | :--- |
| **`xmin`** | `xid` (32位/64位) | 插入该 Tuple 的事务 ID。若 `xmin` 尚未提交或晚于当前快照，该行不可见。 |
| **`xmax`** | `xid` | 更新或删除该 Tuple 的事务 ID（若该行未被更新/删除，`xmax` 为 0）。若 `xmax` 已提交且早于当前快照，该行已过期不可见。 |
| **`cmin` / `cmax`** | `cid` (32位) | 命令标识符（Command Identifier），表示该行是由同一事务内的第几条 SQL 语句创建/删除的，支持事务内部的自见性判断。 |
| **`ctid`** | `tid` (页号, 槽位号) | 当前 Tuple 的物理行指针（Physical Location）。若行被 UPDATE，旧版本的 `ctid` 会指向新版本的 `ctid`。 |

```sql
-- 查询系统隐藏列
SELECT ctid, xmin, xmax, cmin, cmax, id, name, balance 
FROM user_accounts 
WHERE id = 1;
```

### 2.2 VACUUM 与 HOT (Heap-Only Tuples) 优化机制
由于 UPDATE 会在堆表中产生新 Tuple，旧 Tuple 变为死元组（Dead Tuples），因此 PostgreSQL 设计了自动清理与就地优化机制：

1. **AutoVacuum 机制**：后台进程定期扫描包含大量 Dead Tuples 的页，回收存储空间，更新 Visibility Map（VM）和 Free Space Map（FSM），并防范 32 位事务 ID 回卷（Transaction ID Wraparound）。
2. **HOT (Heap-Only Tuples) 技术**：
   - 当执行 `UPDATE` 时，**若被更新的字段没有被任何索引引用**，且当前数据页内还有空闲空间（受 `fillfactor` 控制），新 Tuple 直接存放在同一数据页中。
   - 索引指针直接指向 Root Tuple，无需为新版本插入新的索引项，通过页内链表指针（Tuple Chain）完成跳转，**极大减少索引膨胀与写入 IO**。

```sql
-- 为高频 UPDATE 表预留 20% 页内空闲空间以激发 HOT 优化
ALTER TABLE user_accounts SET (fillfactor = 80);
```

---

#### 🥊 MVCC 底层原理对比：PG (堆表元组版本 + VACUUM 回收) vs MySQL (Undo Log 回滚段 + 最新行原地更新)

```
PostgreSQL MVCC (In-Place Heap Multi-Version):
[Heap Page] ──> [Tuple v1 (xmin:100, xmax:105)] ──ctid──> [Tuple v2 (xmin:105, xmax:0)]
* 旧版本依然存放在数据表中，依赖 VACUUM 清理死元组。

MySQL InnoDB MVCC (Rollback Segment in Undo Log):
[Clustered Index Page] ──> [Current Row (DB_TRX_ID:105)] ──roll_ptr──> [Undo Log: v1 (DB_TRX_ID:100)]
* 聚簇索引页永远只存最新行版本，历史旧版本存放在 Undo Log 回滚段链表中。
```

| 维度 | PostgreSQL 17 | MySQL 8.x (InnoDB) |
| :--- | :--- | :--- |
| **版本存储位置** | **堆表本身（Heap Pages）**：新旧版本元组均作为数据行追加在物理表中 | **Undo Log 回滚段**：数据页中只保留最新行，旧版本通过 `roll_ptr` 追溯 Undo 链表 |
| **回滚与中止成本 (ABORT / ROLLBACK)** | **极快（毫秒级）**：仅需在事务状态位图（CLOG/Commit Log）中将状态标记为 `ABORTED`，无需实际擦除数据 | **较慢**：需遍历并解析 Undo Log 逆向执行反向补偿操作回滚聚簇索引 |
| **垃圾回收机制** | **AutoVacuum 进程**：扫描物理页清理 Dead Tuples，维护 VM/FSM 空间映射 | **Purge 线程**：后台定期清理不再被任何读视图引用的 Undo 页 |
| **长事务的破坏力** | 阻止 Vacuum 回收后续产生的死元组，导致**堆表物理文件膨胀（Table Bloat）** | 阻止 Undo Log 清理，导致 **Undo 空间急剧膨胀（ibdata 暴涨）**且全库查询变慢 |
| **索引写入开销** | 普通更新需要更新所有索引；通过 **HOT (Heap-Only Tuples)** 优化可实现 0 索引修改 | 更新非主键列无需修改聚簇索引位置，但需更新涉及的二级索引 |

> [!WARNING]
> **PostgreSQL 运维避坑核心**：
> 在 PostgreSQL 中，绝对要避免长事务未提交（如应用连接池忘记关闭连接导致 `idle in transaction`）。长事务会“锚定”最老活动 XID，导致整个集群的 AutoVacuum 无法清理后续产生的垃圾元组，引起磁盘和内存暴涨。必须在服务端设置参数：
> `SET idle_in_transaction_session_timeout = '60s';`

---

## 3. 锁机制深度剖析 (Locking Mechanisms)

### 3.1 表级锁（8 种模式与冲突矩阵）

PostgreSQL 表级锁涵盖 8 种粒度，保证 DDL 与 DML、并发维护之间的安全性：

```
                         PostgreSQL 8 种表级锁冲突矩阵 (❌ 表示冲突阻塞)
                    ┌────────────────────────────────────────────────────────────────────────┐
                    │ Requested Lock Mode                                                    │
                    │ 1: ACCESS SHARE        5: SHARE                                        │
                    │ 2: ROW SHARE           6: SHARE ROW EXCLUSIVE                          │
                    │ 3: ROW EXCLUSIVE       7: EXCLUSIVE                                    │
                    │ 4: SHARE UPDATE EXCL   8: ACCESS EXCLUSIVE                             │
                    └────────────────────────────────────────────────────────────────────────┘

 Current Lock Mode       1.AS   2.RS   3.RE   4.SUE   5.S   6.SRE   7.E   8.AE
 ─────────────────────────────────────────────────────────────────────────────
 1. ACCESS SHARE (SELECT)                                                 ❌
 2. ROW SHARE (FOR SHARE)                                           ❌    ❌
 3. ROW EXCLUSIVE (INSERT/UPDATE/DELETE)                     ❌     ❌    ❌
 4. SHARE UPDATE EXCLUSIVE (VACUUM/ANALYZE/CREATE INDEX CONC) ❌     ❌     ❌    ❌
 5. SHARE (CREATE INDEX 标准模式)              ❌             ❌     ❌    ❌    ❌
 6. SHARE ROW EXCLUSIVE                 ❌    ❌     ❌     ❌     ❌    ❌    ❌
 7. EXCLUSIVE                    ❌    ❌     ❌     ❌     ❌     ❌    ❌    ❌
 8. ACCESS EXCLUSIVE (ALTER TABLE/DROP) ❌ ❌ ❌ ❌ ❌ ❌ ❌ ❌
```

### 3.2 行级锁模式与并发控制选项

PostgreSQL 提供了 4 种精细的行级显式锁：

| 行级锁模式 | 对应 SQL 语法 | 是否锁定非键更新 | 与外键检查冲突 | 典型使用场景 |
| :--- | :--- | :--- | :--- | :--- |
| **`FOR UPDATE`** | `SELECT ... FOR UPDATE` | 是（排他行锁） | 是（最高冲突） | 严苛扣减库存、转账资金等关键字段修改 |
| **`FOR NO KEY UPDATE`** | `SELECT ... FOR NO KEY UPDATE` | 否（仅锁非键列） | **否（不阻塞引用该行的外键 INSERT/UPDATE）** | 修改非主键/唯一键字段（如修改用户昵称、更新最后登录时间） |
| **`FOR SHARE`** | `SELECT ... FOR SHARE` | 共享行锁 | 否 | 防止该行被并发修改或删除，允许多方并发读 |
| **`FOR KEY SHARE`** | `SELECT ... FOR KEY SHARE` | 弱共享锁 | 否 | 外键参照完整性检查时的底层默认加锁模式 |

---

#### 🥊 行级锁粒度对比：PG (FOR NO KEY UPDATE 不阻碍外键) vs MySQL (仅独占 X 锁)

```
+----------------------------------------------------------------------------------------------------+
| 业务场景：在电商系统中，父表 users (user_id, balance, nickname)，子表 orders 拥有外键关联 user_id。 |
| 并发动作：事务 T1 更新用户的 nickname；事务 T2 为该用户创建新订单并在 orders 表中插入外键引用记录。  |
+----------------------------------------------------------------------------------------------------+
```

| 维度 | PostgreSQL 17 (`FOR NO KEY UPDATE`) | MySQL 8.x (InnoDB 行锁体系) |
| :--- | :--- | :--- |
| **行锁细分粒度** | **4 种细粒度行锁**（FOR UPDATE, FOR NO KEY UPDATE, FOR SHARE, FOR KEY SHARE） | **仅 2 种行锁**（独占 X 锁、共享 S 锁） |
| **非键更新对子表的影响** | **完全不阻塞**！T1 执行 `UPDATE users SET nickname = '...'` 底层加 `NO KEY UPDATE` 锁，与外键检查的 `KEY SHARE` 兼容 | **严重阻塞/死锁**！T1 更新 nickname 会对 users 行加独占 X 锁，T2 插入 orders 时外键检查需获取 S 锁，导致阻塞等待 |
| **外键高并发吞吐** | 父表常态化更新非关键字段时，子表高频并发插入**零性能损耗** | 父表更新与子表插入产生锁竞争，导致 TPS 急剧下降 |

##### 典型代码与时序对照：

```sql
-- 🐘 PostgreSQL 17：极高并发外键写入
-- 事务 T1（修改昵称）
BEGIN;
SELECT * FROM users WHERE user_id = 1001 FOR NO KEY UPDATE;
UPDATE users SET nickname = 'AwesomePG' WHERE user_id = 1001;
-- 此时事务 T2（并发创建订单）
BEGIN;
INSERT INTO orders (order_id, user_id, amount) VALUES (9901, 1001, 299.0); -- ✅ 瞬间成功，零等待！
COMMIT;
-- T1 随后 COMMIT;
```

```sql
-- 🐬 MySQL 8.0：外键死锁与阻塞高发区
-- 事务 T1
BEGIN;
SELECT * FROM users WHERE user_id = 1001 FOR UPDATE; -- 加上排他 X 锁
UPDATE users SET nickname = 'AwesomeMySQL' WHERE user_id = 1001;
-- 此时事务 T2
BEGIN;
INSERT INTO orders (order_id, user_id, amount) VALUES (9901, 1001, 299.0); -- ❌ 被阻塞！等待 T1 释放 X 锁！
```

---

### 3.3 锁等待控制子句：NOWAIT 与 SKIP LOCKED

- **`NOWAIT`**：如果目标行已被其他事务加锁，立即抛出错误（`ERROR: 55P03: could not obtain lock on row in relation...`），绝不阻塞等待。
- **`SKIP LOCKED`**：跳过所有已被并发事务锁定的记录，仅对未加锁的行加锁并返回。**这是构建超高吞吐分布式消息队列和工作派发流的核心利器！**

```sql
-- 立即获取未被锁定的前 10 条任务，其他已被锁定的任务自动跳过
SELECT task_id, payload 
FROM async_tasks 
WHERE status = 'READY' 
ORDER BY priority DESC, created_at ASC 
LIMIT 10 
FOR UPDATE SKIP LOCKED;
```

---

#### 🥊 任务调度对比：PG (成熟 SKIP LOCKED 零冲突队列) vs MySQL (8.0 引入)

| 对比维度 | PostgreSQL 17 | MySQL 8.0+ | MySQL 5.7 及更早 |
| :--- | :--- | :--- | :--- |
| **`SKIP LOCKED` 原生支持** | **自 PG 9.5 原生支持**（极其成熟，经受数十年金融级消息队列验证） | 自 MySQL 8.0 引入 | **完全不支持** |
| **队列实现复杂度** | 极简：一条 `FOR UPDATE SKIP LOCKED` + CTE + `RETURNING` 原子完成领取与状态更新 | 需分步执行 `SELECT ... FOR UPDATE SKIP LOCKED`，再执行 `UPDATE` 更新状态 | 只能加悲观行锁排队（串行化）或依赖 Redis 等外部队列 |
| **开源队列生态** | 庞大生态直接基于 PG 锁构建（如 Graphile Worker、PG-Boss、River） | 极少有纯基于 MySQL 的原生高性能队列组件 | 依赖外置中间件 |

##### 代码对比：并发 Worker 抢占任务

```sql
-- 🐘 PostgreSQL 17：单 SQL 闭环原子提取与状态变更（结合 CTE 与 RETURNING）
WITH next_task AS (
    SELECT task_id 
    FROM job_queue 
    WHERE status = 'QUEUED' 
    ORDER BY priority DESC, id ASC 
    LIMIT 1 
    FOR UPDATE SKIP LOCKED
)
UPDATE job_queue q
SET status = 'RUNNING', started_at = clock_timestamp()
FROM next_task nt
WHERE q.task_id = nt.task_id
RETURNING q.task_id, q.payload;
```

```sql
-- 🐬 MySQL 8.0：必须分步多交互执行
START TRANSACTION;
-- 步骤 1：查询并锁定
SELECT task_id, payload FROM job_queue 
WHERE status = 'QUEUED' 
ORDER BY priority DESC, id ASC 
LIMIT 1 
FOR UPDATE SKIP LOCKED;
-- 步骤 2：应用层拿到 task_id 后再发起 UPDATE
UPDATE job_queue SET status = 'RUNNING' WHERE task_id = ?;
COMMIT;
```

---

### 3.4 咨询锁（Advisory Locks / 原生分布式锁）

PostgreSQL 提供了应用程序级自定义锁——**咨询锁（Advisory Locks）**。它不依赖具体的数据行，而是以一个 64 位整型（或两个 32 位整型）作为锁标识。

- **核心价值**：纯内存轻量操作，完全由应用掌控，**直接替代 Redis 分布式锁**，具备与 DB 同源的高可用与 ACID 保证，无需担心 Redis 与数据库脑裂或锁丢失。
- **生命周期**：
  1. **会话级（Session-level）**：跨事务持续存在，直到显式释放或连接断开。
  2. **事务级（Transaction-level）**：以 `_xact_` 结尾，事务提交（COMMIT）或回滚（ROLLBACK）时**自动释放**。

```sql
-- 1. 尝试获取事务级排他咨询锁（非阻塞）
SELECT pg_try_advisory_xact_lock(100102); -- 返回 true (加锁成功) 或 false (加锁失败)

-- 2. 阻塞式获取事务级咨询锁
SELECT pg_advisory_xact_lock(100102);

-- 3. 双键咨询锁（如：业务类别 101, 资源 ID 5002）
SELECT pg_try_advisory_xact_lock(101, 5002);
```

---

#### 🥊 应用程序分布式锁对比：PG (原生事务/会话级 Advisory Locks) vs MySQL (仅简陋 GET_LOCK)

| 对比维度 | PostgreSQL 17 (Advisory Locks) | MySQL 8.x (`GET_LOCK`) |
| :--- | :--- | :--- |
| **事务级自动释放 (`_xact_`)** | **原生支持**：事务提交或回滚时**自动且绝对释放**，杜绝死锁与锁泄漏 | **完全不支持**：仅会话级，事务回滚时**锁依然被持有**，极易造成锁泄漏！ |
| **锁标识类型** | 支持 64 位 `BIGINT` 整数或两个 32 位 `INT` 复合键（极高吞吐内存比对） | 仅支持字符串名称（如 `GET_LOCK('lock_name', 10)`），字符串 Hash 开销较大 |
| **共享读锁 (Shared Lock)** | 支持排他锁与**共享咨询锁** (`pg_advisory_lock_shared`) | 仅支持独占排他锁，无法实现多读单写分布式并发控制 |
| **死锁检测体系** | **全面纳入 PG 原生死锁检测引擎**（超时自动判定死锁环路并告警回滚） | **不参与死锁检测**，若应用产生逻辑环路将一直卡死直至连接超时 |

##### 代码对比：实现防止定时任务重复运行的分布式锁

```sql
-- 🐘 PostgreSQL 17：事务级锁，极度安全，无锁泄漏隐患
BEGIN;
SELECT pg_try_advisory_xact_lock(998811); -- 尝试加锁
-- 执行业务逻辑...
COMMIT; -- 事务结束，锁自动瞬间释放！即使服务崩溃或发生异常回滚，锁也百分之百自动安全释放！
```

```sql
-- 🐬 MySQL 8.0：繁琐且高风险
SELECT GET_LOCK('daily_cron_job', 0); -- 获取锁
-- 执行业务逻辑...
SELECT RELEASE_LOCK('daily_cron_job'); -- 必须手动释放！
-- ⚠️ 致命缺陷：如果业务在执行中抛出异常、网络抖动或事务被 ROLLBACK，锁不会释放，后续任务全部被永久卡死！
```

---

### 3.5 死锁检测与防死锁设计

- **死锁检测机制**：由参数 `deadlock_timeout` 控制（默认 1 秒）。事务在等待锁时不会立即启动昂贵的锁依赖拓扑环检测，只有当等待时间超过 `deadlock_timeout` 时，PostgreSQL 才会遍历锁图发现环路，并中止代价最小的事务（抛出 `40P01: deadlock_detected`）。
- **最佳防死锁实践**：
  1. 多表或多行更新必须在所有业务事务中遵循**严格统一的排序加锁顺序**（例如按 `id ASC` 排序后再加锁）。
  2. 合理利用 `FOR UPDATE NOWAIT` 或 `FOR UPDATE SKIP LOCKED` 避免进入锁等待队列。

---

## 4. 生产级高并发业务实战

### 场景 1：秒杀与高并发库存扣减架构选型

在电商大促秒杀场景中，面对每秒万级并发对同一商品库存的扣减，不同方案性能与可靠性对比：

```sql
-- 【方案 A: 悲观锁 SELECT ... FOR UPDATE】
-- 事务生命周期内长久持锁，线程排队严重，高并发下连接池迅速打满
BEGIN;
SELECT stock FROM product_stock WHERE product_id = 101 FOR UPDATE;
-- 业务校验: if (stock >= qty)
UPDATE product_stock SET stock = stock - 1 WHERE product_id = 101;
COMMIT;

-- 【方案 B: 乐观锁 CAS (Compare-And-Swap)】
-- 事务不加行锁，提交时比对 version，高并发冲突时重试率极高（90% CPU 浪费在重试上）
UPDATE product_stock 
SET stock = stock - 1, version = version + 1 
WHERE product_id = 101 AND version = 5 AND stock >= 1;

-- 【方案 C: 行级单语句原子更新 + 校验 (生产首选推荐)】
-- 锁粒度仅限制在单个原子 UPDATE 语句执行的微秒级别，配合 RETURNING 完美获取扣减结果
UPDATE product_stock 
SET stock = stock - 1 
WHERE product_id = 101 AND stock >= 1
RETURNING stock;
-- 若返回行数 = 0，说明库存不足直接失败；若返回 1 行，扣减成功！
```

---

### 场景 2：高吞吐任务队列与分布式 Worker 消费派发

**业务背景**：异步工单/短信推送服务，数十个分布式 Worker 并发拉取待处理任务，要求**零冲突、不重复消费、秒级派发**。

```sql
-- 1. 任务表设计
CREATE TABLE async_task_queue (
    task_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    task_type VARCHAR(64) NOT NULL,
    payload JSONB NOT NULL,
    status VARCHAR(32) NOT NULL DEFAULT 'PENDING',
    retry_count INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT clock_timestamp()
);

CREATE INDEX idx_queue_poll ON async_task_queue (created_at) 
WHERE status = 'PENDING';

-- 2. Worker 消费核心 SQL（结合 CTE 与 FOR UPDATE SKIP LOCKED）
WITH locked_tasks AS (
    SELECT task_id
    FROM async_task_queue
    WHERE status = 'PENDING'
    ORDER BY created_at ASC
    LIMIT 10
    FOR UPDATE SKIP LOCKED
)
UPDATE async_task_queue q
SET status = 'PROCESSING'
FROM locked_tasks lt
WHERE q.task_id = lt.task_id
RETURNING q.task_id, q.task_type, q.payload;
-- 即使部署 100 个 Worker 并发拉取，彼此互不阻塞，无任何行锁冲突等待！
```

---

### 场景 3：基于 PostgreSQL 咨询锁实现无 Redis 轻量级分布式任务锁

**业务背景**：定时跑批任务（Cron Job）部署了多个微服务副本，要求同一时刻只有 1 个实例能执行日终清算，防止多节点重复运行。

```sql
CREATE OR REPLACE FUNCTION run_daily_settlement_job(p_job_date DATE) 
RETURNS BOOLEAN AS $$
DECLARE
    -- 将业务标识映射为唯一 64 位数字（例如哈希或固定数值）
    v_lock_id CONSTANT BIGINT := 8899112233;
    v_got_lock BOOLEAN;
BEGIN
    -- 尝试获取事务级咨询锁（非阻塞，事务结束自动归还）
    v_got_lock := pg_try_advisory_xact_lock(v_lock_id);
    
    IF NOT v_got_lock THEN
        RAISE NOTICE 'Another worker is running the daily settlement job. Skipping...';
        RETURN FALSE;
    END IF;

    -- 成功获取分布式锁，执行核心批处理逻辑
    RAISE NOTICE 'Acquired lock. Executing settlement for %', p_job_date;
    
    -- 模拟批处理操作
    PERFORM pg_sleep(2);

    RETURN TRUE;
END;
$$ LANGUAGE plpgsql;

-- 调度器调用
SELECT run_daily_settlement_job(CURRENT_DATE);
```

---

## 5. PostgreSQL 与 MySQL (InnoDB) 事务并发综合对比全景

| 对比维度 | PostgreSQL 17 | MySQL 8.0 (InnoDB) | 深度原理解析与架构差异 |
| :--- | :--- | :--- | :--- |
| **MVCC 实现原理** | **Heap Tuple 多版本存放在堆表**，由 `xmin`/`xmax` 标识，通过 VACUUM 回收 Dead Tuples | **单版本堆表 + Undo Log 回滚段**，旧版本在 Undo 页形成版本链，由 Purge 线程清理 | PG 的 HOT 机制减少了单页更新的开销；MySQL 更新聚簇索引和二级索引开销相对固定，但其堆表不膨胀。 |
| **Repeatable Read 幻读控制** | **快照读原生无幻读**，写操作严格基于快照判定版本冲突（First-Committer-Wins） | **依赖间隙锁（Gap Lock / Next-Key Lock）** 阻塞范围插入来防止幻读 | PG 无间隙锁开销，极大提升了并发写入吞吐；MySQL 间隙锁容易引发难以捉摸的死锁（Deadlock）。 |
| **行级锁粒度** | 提供 `FOR UPDATE` 与 **`FOR NO KEY UPDATE`**，细分主外键约束与常规列 | **仅有独占排他锁 (X-Lock) 与共享锁 (S-Lock)** | PG 更新非主键字段时不会阻塞关联外键的插入/更新；MySQL 一律独占加锁，外键并发性能受限。 |
| **任务队列与并发调度** | **原生长期稳定支持** `SKIP LOCKED` 与 `NOWAIT`，成熟用于各类队列引擎 | MySQL 8.0 之后才引入 `SKIP LOCKED`，早期版本无法高效实现 DB 队列 | PG 在队列场景下生态极佳（如 Graphile Worker、PG-Boss）。 |
| **分布式应用级锁** | **原生内置 Advisory Locks（咨询锁）**，支持会话级与事务级，支持 64 位与多键 | 仅有简陋的 `GET_LOCK()`，**不支持事务级绑定**（事务回滚不会自动释放） | PG 咨询锁完全可以替代 Redis Redlock，具有强一致性和事务自动释放保证。 |
| **事务级 DDL (Transactional DDL)** | **全量原生支持**（`CREATE TABLE`, `ALTER TABLE`, `DROP INDEX` 均可在事务中回滚） | **不支持**（执行任何 DDL 语句均会**隐式强制提交当前事务**） | PG 支持蓝绿发布和安全数据库迁移脚本（执行出错直接 ROLLBACK，不留半拉子脏结构）。 |
| **可串行化隔离 (Serializable)** | **SSI (Serializable Snapshot Isolation)**：基于依赖图分析，零锁开销自动拦截冲突 | **两阶段锁 (2PL)**：对所有读操作隐式强制升级为 `FOR SHARE` 读锁，严重降低吞吐 | PG SSI 具备极高的并发读取性能，只需应用层做好 `40001` 重试即可。 |

---

## 6. 开发者核心避坑与调优准则

1. **应用层必须为 Serializable 和 Repeatable Read 实现重试模板**：
   - 捕获 PostgreSQL 错误码 `40001` (`serialization_failure`) 与 `40P01` (`deadlock_detected`)，配置指数退避重试（Exponential Backoff）。
2. **长事务危害与监控**：
   - 长时间未提交的事务（`idle in transaction`）会阻止 AutoVacuum 回收该事务之后产生的所有 Dead Tuples，导致整库物理膨胀。
   - 必须配置服务端保护参数：`idle_in_transaction_session_timeout = '60s'`。
3. **精准使用行锁，优先使用 `FOR NO KEY UPDATE`**：
   - 在高并发涉及外键关联的业务表中，尽可能将 `SELECT ... FOR UPDATE` 替换为 `FOR NO KEY UPDATE`，避免对从表外键插入造成锁等待。
4. **利用 Transactional DDL 实现零风险发布**：
   - 在迁移脚本（Flyway / Liquibase / 自定义脚本）中始终使用 `BEGIN ... COMMIT` 包裹 DDL，确保迁移失败时自动回滚，不留任何残缺的中间表结构。
