---
name: sql-pgtap
description: 为 PostgreSQL 18 编制 pgTAP 测试文件骨架（unit / integration / performance / security 四类），并用 pg_prove 运行校验。在编写数据库测试、校验迁移结果、把测试接入 sqitch verify 或 CI 时使用。迁移脚本编制请用 sql-sqitch skill。
license: PostgreSQL License
compatibility: PostgreSQL 18；pgTAP 1.x；pg_prove；Sqitch
---

# SQL pgTAP 测试编制（Test Generator）

为 PostgreSQL 18 数据库对象编制 pgTAP 测试文件骨架，覆盖结构验证、函数边界断言、行为链集成、性能基线与安全属性四类测试。

## 约定

- 注释使用中文，标识符使用英文
- 文件路径相对 `{project-root}` 解析
- 与 `sql-sqitch` skill 配套：sqitch 负责迁移脚本（deploy/revert/verify），本 skill 负责测试校验

## Skill 职责

- 类型：代码生成器
- 触发：用户要求「编写测试 / 生成测试文件 / 校验迁移结果」时
- 输入：测试类型 + 目标 schema/对象 + 描述
- 输出：pgTAP 测试文件骨架
- 配套：`sql-sqitch` — 生成对应的 Sqitch 迁移脚本

| 类型 | 编号 | 输出文件 | 说明 |
|------|------|---------|------|
| unit | TTXXX | `{schema}_{object}_unit_{desc}.sql` | 36 固定分组，5 Sections，完整结构验证 |
| integration | AGGDNN + FNNXXX | `{schema}_{object}_integration_{desc}.sql` | A01-A99 函数边界断言 + F01-F07 场景集成 |
| performance | GNNXXX | `{schema}_{object}_performance_{desc}.sql` | G01-G04 NFR 类别，阈值验证 |
| security | SNNXXX | `{schema}_{object}_security_{desc}.sql` | 安全属性验证（待细化） |

## 激活时（On Activation）

向用户确认：

1. **测试类型**：unit / integration / performance / security
2. **目标 schema**：`{schema}`（如 `app`、`audit`、`report`）
3. **目标对象**：`{object}`（如 `users`、`orders`、`user_fn`）
4. **描述**：`{desc}`（如 `mgmt`、`mini` 或自定义，如 `epic1_e2e_workflow`）
5. **待测对象清单**（可选）：列出表、视图、函数、种子数据

信息不足时按上下文推断或最小化询问。

## 输出

写入文件：`test/db/{schema}_{object}_{test_type}_{desc}.sql`

---

## 测试类型

### 1. Unit Test（unit）

文件命名：`{schema}_{object}_unit_{desc}.sql`
编号：`TTXXX` — TT = 分组（01-36），XXX = 组内序号（001-999）

#### 36 个固定分组（5 个 Section）

36 个分组必须全部出现在每个 unit 测试文件中，即使为空也要保留分组头。

**Section A：Tables — 表结构验证（T01-T15）**

| TT | 名称 | 验证内容 | 级别 | pgTAP 函数 |
|----|------|---------|------|-----------|
| T01 | 表存在性验证 | 业务表 + 辅助表（日志/归档等，如适用） | MUST | `has_table()` |
| T02 | 主键验证 | 主键列名、类型 | MUST | `has_pk()`, `col_is_pk()` |
| T03 | 标准字段类型验证 | 项目标准字段（如 id / created_at / updated_at 等通用审计字段；PG 18 的 identity 列按 `attidentity` 校验） | MUST | `col_type_is()` |
| T04 | 业务字段类型验证 | 对象特有业务字段 | MUST | `col_type_is()` |
| T05 | 字段顺序验证 | 列顺序与 Sqitch 定义一致 | MUST | `columns_are()` |
| T06 | NOT NULL 约束验证 | **每字段必须验证**：所有字段逐一使用 `col_not_null()`（NOT NULL 列）或 `col_is_null()`（可 NULL 列），禁止仅验证部分字段 | MUST | `col_not_null()`, `col_is_null()` |
| T07 | DEFAULT 值验证 | **每字段必须验证**：所有字段逐一使用 `col_default_is()`（有默认值）、`col_has_default()`（确认有默认值）、`col_hasnt_default()`（确认无默认值），禁止仅验证部分字段 | MUST | `col_default_is()`, `col_has_default()`, `col_hasnt_default()` |
| T08 | CHECK 约束存在性验证 | **双重验证**：表级 `has_check()` 验证约束总数 + 列级 `col_has_check()` 逐列验证含 CHECK 约束的列 | MUST | `has_check()`, `col_has_check()` |
| T09 | CHECK 约束内容验证 | 约束表达式是否正确 | MUST | `ok()` + `pg_get_constraintdef(oid)` |
| T10 | 表触发器绑定验证 | **表绑定的触发器**（如审计/变更跟踪触发器），**必须使用 `trigger_is()` 5 参数**：`trigger_is(schema, table, trigger_name, func_schema, func_name)`，禁止使用 `has_trigger()` 替代 | MUST | `trigger_is()` |
| T11 | 索引验证 | 索引存在、唯一性（业务表 + 归档表） | SHOULD | `has_index()`, `indexes_are()`, `index_is_unique()` |
| T12 | 归档/历史表 archived_at 列验证 | archived_at 类型/NOT NULL/DEFAULT（如有归档表） | SHOULD | `col_type_is()`, `col_not_null()`, `col_default_is()` |
| T13 | 归档/历史表列对比验证 | 源表所有列是否出现在归档表中 | SHOULD | `ok()` + `information_schema` 列对比 |
| T14 | FK 关系验证 | 外键关系完整性 | MUST | `fk_ok()`, `has_fk()`, `col_is_fk()` |
| T15 | 唯一约束验证 | UNIQUE 约束存在性和列组成 | MUST | `col_is_unique()` |

**Section B：Views — 视图结构验证（T16-T20）**

| TT | 名称 | 验证内容 | 级别 | pgTAP 函数 |
|----|------|---------|------|-----------|
| T16 | 视图存在性验证 | 普通视图、报告视图、物化视图等 | MUST | `has_view()`, `has_materialized_view()` |
| T17 | 字段类型验证 | **每字段必须验证**：视图/物化视图所有列逐一验证类型，MView 使用 `pg_attribute` 查询 | MUST | `col_type_is()`（普通视图），`pg_attribute` + `ok()`（物化视图） |
| T18 | 字段顺序验证 | **必须使用 `columns_are()`**：验证视图/物化视图所有列名和顺序，禁止使用 `pg_attribute` EXISTS 查询替代 | MUST | `columns_are()` |
| T19 | 物化视图索引验证 | MView 索引存在和唯一性 | SHOULD | `has_index()`, `indexes_are()` |
| T20 | INSTEAD OF 视图触发器绑定验证 | **视图绑定的 INSTEAD OF 触发器**，**必须使用 `trigger_is()` 5 参数**：`trigger_is(schema, view, trigger_name, func_schema, func_name)`，禁止使用 `has_trigger()` 替代 | MUST | `trigger_is()` |

**Section C：Functions — 函数结构验证（T21-T26）**

| TT | 名称 | 验证内容 | 级别 | pgTAP 函数 |
|----|------|---------|------|-----------|
| T21 | 函数存在性和入参验证 | 函数存在、参数类型 | MUST | `has_function()` |
| T22 | 返回值验证 | 返回类型精确匹配 | MUST | `function_returns()` |
| T23 | volatility 验证 | IMMUTABLE/STABLE/VOLATILE | MUST | `volatility_is()` |
| T24 | strict 验证 | STRICT/非 STRICT | MUST | `is_strict()`, `isnt_strict()` |
| T25 | security 验证 | SECURITY DEFINER/INVOKER | MUST | `is_definer()`, `isnt_definer()` |
| T26 | 其他函数属性验证 | 语言、并行安全 | SHOULD | `function_lang_is()` |

**Section D：Seed — 种子数据验证（T27-T30）**

| TT | 名称 | 验证内容 | 级别 | pgTAP 函数 |
|----|------|---------|------|-----------|
| T27 | 种子分布 | 分类树存在、数据条数 | MUST | `is()`（过滤计数） |
| T28 | 种子格式 | JSONB 字段结构 | MUST | `is()`, `matches()` |
| T29 | 种子关联完整性 | 外键关联正确 | MUST | `is()`（join 计数） |
| T30 | 数据存在 | 关键数据项存在 | MUST | `is()`, `ok()` |

**Section E：Cross-Object — 跨对象验证（T31-T36）**

| TT | 名称 | 验证内容 | 级别 | pgTAP 函数 |
|----|------|---------|------|-----------|
| T31 | 表+约束注释验证 | **三段连续编号子结构**：T31xxx 前段表注释（每表 1 条）→ 中段 CHECK 约束注释（业务命名 `ck_*`，CONSTRAINT 形式用 `obj_description(oid, 'pg_constraint')`，逐条用 `(schema, table, conname)` 三元组定位避免归档镜像同名冲突）→ 末段 UNIQUE/FK 注释（INDEX 形式 `CREATE UNIQUE INDEX` 用 `obj_description(oid, 'pg_class')`，CONSTRAINT 形式 `ADD CONSTRAINT` 用 `obj_description(oid, 'pg_constraint')`）。归档/镜像表由 `LIKE` 继承则不单独校验 | SHOULD | `ok()` + `obj_description()` + `pg_constraint` |
| T32 | 表列注释验证 | **所有表所有列必须逐一验证**：每张表的每个字段都使用 `col_description()` 验证注释存在，含全列聚合检查 | SHOULD | `ok()` + `col_description()` |
| T33 | 视图注释验证 | COMMENT ON VIEW | SHOULD | `ok()` + `obj_description()` |
| T34 | 视图列注释验证 | **所有视图所有列必须逐一验证**：视图/物化视图的所有字段使用 `col_description()` 验证注释存在，含全列聚合检查 | SHOULD | `ok()` + `col_description()` |
| T35 | 函数注释验证 | COMMENT ON FUNCTION | SHOULD | `ok()` + `obj_description()` |
| T36 | 相关项验证 | 跨对象依赖、兜底检查 | SHOULD | `has_schema()`, `ok()` |

#### Unit Test 模板

按以下结构生成文件：

```sql
-- ================================================================
-- URI: test/db/{schema}_{object}_unit_{desc}.sql
-- ================================================================
--
-- Unit Test:     {schema} {object} {desc} unit tests
-- Description:   {对象描述} 单元测试（Tables + Views + Functions + Seed）
-- Related Change: {sqitch_change_name}
--
-- Test Objects:
--   {sqitch_plan_name_1}               — {Object Category}
--   {sqitch_plan_name_2}               — {Object Category}
--   ...
--
-- Test Group Overview:
--   T01000  {组名称}
--   T02000  {组名称}
--   ...
--
-- Usage:
--   pg_prove test/db/{schema}_{object}_unit_{desc}.sql
--
-- Dependencies:
--   - pgTAP extension installed
--   - Sqitch deployed to latest change
--
-- ================================================================

\unset AUTOCOMMIT
\set ON_ERROR_STOP true

BEGIN;

SELECT no_plan();

-- ====================================================================
-- Section A: Tables — 表结构验证（T01-T15）
-- ====================================================================

-- ────────────────────────────────────────────────────────────────────
-- Test Group T01: 表存在性验证
-- ────────────────────────────────────────────────────────────────────

-- --- 业务表 ---
-- SELECT has_table('{schema}', '{table}', '[T01001] {schema}.{table} 表应存在');

-- --- 辅助表（如日志/归档表，适用时） ---
-- SELECT has_table('{schema}', '{aux_table}', '[T01002] {schema}.{aux_table} 表应存在');

-- --- 归档表（如适用） ---
-- SELECT has_table('{schema}', '{archive_table}', '[T01003] {schema}.{archive_table} 归档表应存在');

-- ────────────────────────────────────────────────────────────────────
-- Test Group T02: 主键验证
-- ────────────────────────────────────────────────────────────────────

-- ────────────────────────────────────────────────────────────────────
-- Test Group T03: 标准字段类型验证
-- ────────────────────────────────────────────────────────────────────

-- ────────────────────────────────────────────────────────────────────
-- Test Group T04: 业务字段类型验证
-- ────────────────────────────────────────────────────────────────────

-- ────────────────────────────────────────────────────────────────────
-- Test Group T05: 字段顺序验证
-- ────────────────────────────────────────────────────────────────────

-- ────────────────────────────────────────────────────────────────────
-- Test Group T06: NOT NULL 约束验证
-- ────────────────────────────────────────────────────────────────────

-- ────────────────────────────────────────────────────────────────────
-- Test Group T07: DEFAULT 值验证
-- ────────────────────────────────────────────────────────────────────

-- ────────────────────────────────────────────────────────────────────
-- Test Group T08: CHECK 约束存在性验证
-- ────────────────────────────────────────────────────────────────────

-- ────────────────────────────────────────────────────────────────────
-- Test Group T09: CHECK 约束内容验证
-- ────────────────────────────────────────────────────────────────────

-- ────────────────────────────────────────────────────────────────────
-- Test Group T10: 表触发器绑定验证
-- ────────────────────────────────────────────────────────────────────

-- ────────────────────────────────────────────────────────────────────
-- Test Group T11: 索引验证
-- ────────────────────────────────────────────────────────────────────

-- ────────────────────────────────────────────────────────────────────
-- Test Group T12: 归档表 archived_at 列验证
-- ────────────────────────────────────────────────────────────────────

-- ────────────────────────────────────────────────────────────────────
-- Test Group T13: 归档表列对比验证
-- ────────────────────────────────────────────────────────────────────

-- ────────────────────────────────────────────────────────────────────
-- Test Group T14: FK 关系验证（pgTAP has_fk + fk_ok）
-- ────────────────────────────────────────────────────────────────────

-- --- FK 存在性 ---
-- SELECT has_fk('{schema}', '{table}', '[T14001] {schema}.{table} 应有 FK 约束');
-- --- FK 关系正确性 ---
-- SELECT fk_ok('{schema}', '{table}', '{column}', '{ref_schema}', '{ref_table}', '{ref_column}', '[T14002] ...');

-- ────────────────────────────────────────────────────────────────────
-- Test Group T15: 唯一约束验证（pgTAP has_unique + col_is_unique + has_index + index_is_unique）
-- ────────────────────────────────────────────────────────────────────

-- --- UNIQUE 存在性 ---
-- SELECT has_unique('{schema}', '{table}', '[T15001] {schema}.{table} 应有 UNIQUE 约束');
-- --- 唯一索引 ---
-- SELECT has_index('{schema}', '{table}', '{index_name}', ARRAY['{col1}', '{col2}'], '[T15002] ...');
-- SELECT index_is_unique('{schema}', '{table}', '{index_name}', '[T15003] ...');
-- --- 列级 / 复合 UNIQUE ---
-- SELECT col_is_unique('{schema}', '{table}', '{column}', '[T15004] ...');
-- SELECT col_is_unique('{schema}', '{table}', ARRAY['{col1}', '{col2}'], '[T15005] ...');

-- ====================================================================
-- Section B: Views — 视图结构验证（T16-T20）
-- ====================================================================

-- ────────────────────────────────────────────────────────────────────
-- Test Group T16: 视图存在性验证
-- ────────────────────────────────────────────────────────────────────

-- ────────────────────────────────────────────────────────────────────
-- Test Group T17: 字段类型验证
-- ────────────────────────────────────────────────────────────────────

-- ────────────────────────────────────────────────────────────────────
-- Test Group T18: 字段顺序验证
-- ────────────────────────────────────────────────────────────────────

-- ────────────────────────────────────────────────────────────────────
-- Test Group T19: 物化视图索引验证
-- ────────────────────────────────────────────────────────────────────

-- ────────────────────────────────────────────────────────────────────
-- Test Group T20: INSTEAD OF 视图触发器绑定验证
-- ────────────────────────────────────────────────────────────────────

-- ====================================================================
-- Section C: Functions — 函数结构验证（T21-T26）
-- ====================================================================

-- ────────────────────────────────────────────────────────────────────
-- Test Group T21: 函数存在性和入参验证
-- ────────────────────────────────────────────────────────────────────

-- ────────────────────────────────────────────────────────────────────
-- Test Group T22: 返回值验证
-- ────────────────────────────────────────────────────────────────────

-- ────────────────────────────────────────────────────────────────────
-- Test Group T23: volatility 验证
-- ────────────────────────────────────────────────────────────────────

-- ────────────────────────────────────────────────────────────────────
-- Test Group T24: strict 验证
-- ────────────────────────────────────────────────────────────────────

-- ────────────────────────────────────────────────────────────────────
-- Test Group T25: security 验证
-- ────────────────────────────────────────────────────────────────────

-- ────────────────────────────────────────────────────────────────────
-- Test Group T26: 其他函数属性验证
-- ────────────────────────────────────────────────────────────────────

-- ====================================================================
-- Section D: Seed — 种子数据验证（T27-T30）
-- ====================================================================

-- ────────────────────────────────────────────────────────────────────
-- Test Group T27: 种子分布
-- ────────────────────────────────────────────────────────────────────

-- ────────────────────────────────────────────────────────────────────
-- Test Group T28: 种子格式
-- ────────────────────────────────────────────────────────────────────

-- ────────────────────────────────────────────────────────────────────
-- Test Group T29: 种子关联完整性
-- ────────────────────────────────────────────────────────────────────

-- ────────────────────────────────────────────────────────────────────
-- Test Group T30: 数据存在
-- ────────────────────────────────────────────────────────────────────

-- ====================================================================
-- Section E: Cross-Object — 跨对象验证（T31-T36）
-- ====================================================================

-- ────────────────────────────────────────────────────────────────────
-- Test Group T31: 表+约束注释验证
-- ────────────────────────────────────────────────────────────────────
-- 子段编号：前段表注释 → 中段 CHECK 约束注释 → 末段 UNIQUE/FK 注释，连续编号不跳号

-- ────────────────────────────────────────────────────────────────────
-- Test Group T32: 表列注释验证
-- ────────────────────────────────────────────────────────────────────

-- ────────────────────────────────────────────────────────────────────
-- Test Group T33: 视图注释验证
-- ────────────────────────────────────────────────────────────────────

-- ────────────────────────────────────────────────────────────────────
-- Test Group T34: 视图列注释验证
-- ────────────────────────────────────────────────────────────────────

-- ────────────────────────────────────────────────────────────────────
-- Test Group T35: 函数注释验证
-- ────────────────────────────────────────────────────────────────────

-- ────────────────────────────────────────────────────────────────────
-- Test Group T36: 相关项验证（兜底）
-- ────────────────────────────────────────────────────────────────────

-- 说明: 此分组为空（{schema}.{object} 无跨对象副作用需要验证）

SELECT * FROM finish();
ROLLBACK;
```

#### Unit Test 规则

**强制规则：**
1. 36 个分组必须全部出现在文件中，即使为空
2. 5 个 Section 必须按 A→B→C→D→E 顺序排列
3. TT 编号严格按规范，不得自定义或跳号
4. 断言消息必须带 `[Txxxxx]` 前缀（5 位）
5. MUST 级分组必须使用指定 pgTAP 函数，禁止 `ok()` + 子查询替代
6. 所有断言在单个事务内执行（BEGIN / ROLLBACK）
7. 种子测试避免无 WHERE 的 `count(*) = N`
8. T01 若存在归档/历史表则必须验证其存在性（无则标注为空）
9. Test Objects 必须使用 sqitch plan 名竖排列表，每项标注对象分类（Table / Function / View / Seed Data 等）
10. Test Group Overview 必须列出所有非空测试分组及其编号和名称
11. **T08/T09 职责分离**：仅 CHECK 约束（存在性 + 内容），禁止混入 UNIQUE/FK（后者分别由 T14/T15 负责，避免重叠）
12. **T14 MUST pgTAP 函数**：`has_fk(schema, table)` 存在性 + `fk_ok(schema, table, col, ref_schema, ref_table, ref_col)` 关系正确性
13. **T15 MUST pgTAP 函数**：`has_unique(schema, table)` 存在性 + `col_is_unique(schema, table, column)` 或 `col_is_unique(schema, table, ARRAY[...])` 复合 + `has_index` + `index_is_unique` 唯一索引

**空组处理：**
- 空组必须保留分组头注释，标注原因：`-- 说明: 此分组为空（{reason}）`
- 整个 Section 可跳过（对象无该类对象时）：Section 级注释说明

**分隔符规则：**

| 层级 | 格式 |
|------|------|
| Section | `-- ====================================================================` + `-- Section X: {名称} — {描述}（T{nn}-T{mm}）` + `-- ====================================================================` |
| Group | `-- ────────────────────────────────────────────────────────────────────` + `-- Test Group T{nn}: {名称}` + `-- ────────────────────────────────────────────────────────────────────` |
| Sub-group | `-- --- {描述} ---`（可选，组内对象切换时使用） |

---

### 2. Integration Test（integration）

文件命名：`{schema}_{object}_integration_{desc}.sql`

#### 编号体系

集成测试文件包含两个独立编号段，**boundary 分组（A）必须放在场景分组（F）之前**：

| 编号段 | 格式 | 说明 |
|--------|------|------|
| Boundary | `AGGDNN` | 1 函数 plan = 1 分组，边界断言全覆盖 |
| Scenario | `FNNXXX` | F01-F07 场景选取，行为链验证 |

#### Boundary 分组（A01-A99）

每个函数 plan（普通函数 / 触发器函数 / 定时任务函数）对应一个独立 A 分组，按 sqitch.plan 部署顺序编号。

**编号格式：** `[AGGDNN]` — GG = 分组（01-99），D = 维度（1-8），NN = 组内序号（01-99）

**8 个固定维度（子分组）：**

| D | 维度 | 说明 | 典型用例数 |
|---|------|------|-----------|
| 1 | Happy path | 成功路径 + 返回值精确验证 | 1-5 |
| 2 | Input boundary | NULL/空串/超长/非法格式/边界值 | 5-15 |
| 3 | Error path | 每个 RAISE EXCEPTION / 每个失败分支逐一验证 | 3-15 |
| 4 | Side effect | 日志/状态变更/物化视图刷新 | 2-10 |
| 5 | State transition | 状态机转换（不适用时标空） | 3-10 |
| 6 | Output schema | 返回值类型/结构/字段精确匹配 | 1-5 |
| 7 | Idempotency | 重复调用安全性（如 MERGE 幂等 upsert） | 1-3 |
| 8 | Concurrency | 并发/锁/序列行为 | 1-3 |

**分组结构示例：**

```sql
-- ====================================================================
-- Boundary: 函数边界断言（A01-A99）
-- ====================================================================

-- ────────────────────────────────────────────────────────────────────
-- A01: order_submit 边界断言
-- ────────────────────────────────────────────────────────────────────

-- --- A01-1: Happy path ---
-- [A01101] 提交成功 → o_is_ok=true
-- [A01102] 金额计算正确 → o_total 精确匹配

-- --- A01-2: Input boundary ---
-- [A01201] p_customer_id = NULL → o_is_ok=false
-- [A01202] p_amount <= 0 → o_is_ok=false

-- --- A01-3: Error path ---
-- [A01301] 库存不足 → o_is_ok=false
-- [A01302] 客户不存在 → o_is_ok=false

-- --- A01-4: Side effect ---
-- [A01401] 成功时 order_log 有 status='created' 记录
-- [A01402] 失败时 order_log 有 error 字段

-- --- A01-5: State transition ---
-- 说明: 此子分组为空（order_submit 无状态转换）

-- --- A01-6: Output schema ---
-- [A01601] 返回 3 列: o_is_ok(BOOLEAN), o_order_id(BIGINT), o_total(NUMERIC)

-- --- A01-7: Idempotency ---
-- 说明: 此子分组为空（重复调用产生新订单，天然非幂等；如为 upsert 场景用 MERGE 验证）

-- --- A01-8: Concurrency ---
-- 说明: 此子分组为空（或按需验证并发下单）
```

**Boundary 分组规则：**

1. 每个函数 plan 必须有独立 A 分组，即使测试用例较少
2. 8 个维度子分组必须全部出现，不适用时保留子分组头 + `说明: 此子分组为空（{reason}）`
3. INSTEAD OF trigger 函数的边界测试放在 A 分组中，不再使用 F02 场景
4. 每个错误路径（RAISE EXCEPTION / o_is_ok=false）必须有独立测试用例
5. Side effect 维度验证函数执行后的副作用（日志记录、状态变更、mview 刷新）
6. 编号按 sqitch.plan 部署顺序排列

---

#### 场景分类（F01-F07）

编号：`FNNXXX` — NN = 场景（01-07），XXX = 组内序号（001-999）

| 场景 | 名称 | 验证链路 |
|------|------|---------|
| F01 | 配置/元数据管理集成 | 写入 → 刷新 → 读取 |
| F02 | 触发器行为集成 | INSERT/UPDATE/DELETE → trigger → 预期结果 |
| F03 | 日志服务集成 | 日志写入 → 日志表 → 查询/过滤 |
| F04 | 异步调度集成 | 调度 → 执行 → 状态 |
| F05 | 归档迁移集成 | 迁移 → 数据迁移 → 归档校验 |
| F06 | 跨对象端到端 | 写入 → 查询 → 状态流转 → 归档 |
| F07 | 结果集完整性集成 | 数据写入 → 查询 → 结果集精确匹配 |

场景选取按对象需要，不强制全部使用，但**必须覆盖所有适用的场景**并包含充分的测试用例和断言，禁止仅使用 2 个场景敷衍。每个场景应有明确的测试步骤、充分的断言覆盖，不得仅有骨架占位。

#### Integration Test 模板

```sql
-- ================================================================
-- URI: test/db/{schema}_{object}_integration_{desc}.sql
-- ================================================================
--
-- Integration Test: {schema} {object} {desc} integration
-- Description:     {对象描述} 集成测试（函数边界断言 + 行为链验证）
-- Related Change:  {sqitch_change_name}
--
-- Test Objects:
--   {sqitch_plan_name_1}               — {Object Category}
--   {sqitch_plan_name_2}               — {Object Category}
--   ...
--
-- Test Group Overview:
--   A01     {函数名} 边界断言
--   A02     {函数名} 边界断言
--   ...
--   F01     {场景名称}
--   F06     {场景名称}
--   F07     {场景名称}
--
-- Usage:
--   pg_prove test/db/{schema}_{object}_integration_{desc}.sql
--
-- Dependencies:
--   - pgTAP extension installed
--   - Sqitch deployed to latest change
--
-- ================================================================

\unset AUTOCOMMIT
\set ON_ERROR_STOP true

BEGIN;

SELECT no_plan();

-- ====================================================================
-- Boundary: 函数边界断言（A01-A99）
-- ====================================================================

-- ────────────────────────────────────────────────────────────────────
-- A01: {函数名} 边界断言
-- ────────────────────────────────────────────────────────────────────

-- --- A01-1: Happy path ---
-- [A01101] ...

-- --- A01-2: Input boundary ---
-- [A01201] ...

-- --- A01-3: Error path ---
-- [A01301] ...

-- --- A01-4: Side effect ---
-- [A01401] ...

-- --- A01-5: State transition ---
-- 说明: 此子分组为空（{reason}）

-- --- A01-6: Output schema ---
-- [A01601] ...

-- --- A01-7: Idempotency ---
-- 说明: 此子分组为空（{reason}）

-- --- A01-8: Concurrency ---
-- 说明: 此子分组为空（{reason}）

-- ... A02-A99 同理 ...

-- ====================================================================
-- Scenario F01: {场景名称}
-- ====================================================================

SELECT * FROM finish();
ROLLBACK;
```

#### Integration Test 规则

**通用规则：**
1. 所有数据变更必须在事务内（BEGIN / ROLLBACK）
2. 临时对象使用 `test.` schema（或约定的测试 schema）
3. 推荐函数：`throws_ok()`, `throws_like()`, `throws_matching()`, `lives_ok()`, `is()`, `ok()`, `set_eq()`, `bag_eq()`, `is_empty()`
4. 禁止 `ok(true, ...)` 无意义断言
5. Test Objects 必须列出所有涉及的 sqitch plan，使用竖排列表 + 对象分类
6. Test Group Overview 必须列出所有 A 分组和 F 场景编号及名称，A 分组标识符**必须使用 sqitch plan name**（与 Test Objects 列表保持一致），禁止使用 PG 全限定名 `schema.object`；段头格式 `-- A01: {sqitch_plan_name} — {一句话功能描述}`

**Boundary 分组（A）规则：**
7. 每个函数 plan 必须有独立 A 分组，按 sqitch.plan 部署顺序编号
8. 8 个维度子分组必须全部出现，不适用时保留头 + 空原因说明
9. 每个 RAISE EXCEPTION / 每个失败分支必须有独立测试用例
10. Side effect 维度必须验证函数执行后的日志/状态变更
11. 消息前缀格式：`[AGGDNN]`（A + 2 位组号 + 1 位维度 + 2 位序号）

**场景分组（F）规则：**
12. 场景选取按对象需要，不强制全部使用，但必须覆盖所有适用场景
13. 消息前缀格式：`[F0X001]`
14. **分支全覆盖**：INSTEAD OF trigger 的边界测试已移至 A 分组，F 场景聚焦跨对象行为链

---

### 3. Performance Test（performance）

文件命名：`{schema}_{object}_performance_{desc}.sql`
编号：`GNNXXX` — NN = NFR 类别（01-04），XXX = 组内序号（001-999）

#### NFR 类别（G01-G04）

| 类别 | 名称 | 典型 NFR |
|------|------|---------|
| G01 | 吞吐量测试 | ≥10000 ID/s, >1000 条/秒 |
| G02 | 延迟测试 | <50ms, ≤5ms |
| G03 | 批量测试 | 50000 行 ≤30s |
| G04 | 查询性能 | 报告查询 ≤5s |

#### Performance Test 模板

```sql
-- ================================================================
-- URI: test/db/{schema}_{object}_performance_{desc}.sql
-- ================================================================
--
-- Performance Test: {schema} {object} {desc} performance
-- Description:     {对象描述} 性能基线测试（NFR 验证）
-- Related Change:  {sqitch_change_name}
--
-- Test Objects:
--   {sqitch_plan_name_1}               — {Object Category}
--   {sqitch_plan_name_2}               — {Object Category}
--   ...
--
-- Test Group Overview:
--   G01000  {吞吐量测试描述}
--   G02000  {延迟测试描述}
--
-- Usage:
--   pg_prove test/db/{schema}_{object}_performance_{desc}.sql
--
-- Dependencies:
--   - pgTAP extension installed
--   - Sqitch deployed to latest change
--
-- ================================================================

\unset AUTOCOMMIT
\set ON_ERROR_STOP true

BEGIN;

SELECT no_plan();

-- ────────────────────────────────────────────────────────────────────
-- NFR G01: 吞吐量测试
-- ────────────────────────────────────────────────────────────────────

-- PREPARE g01_{desc} AS SELECT ...;
-- SELECT performs_ok('g01_{desc}', {ms}, '[G01001] ...应 ≤{N}ms');

SELECT * FROM finish();
ROLLBACK;
```

#### Performance Test 规则

1. 首选 `performs_ok()` / `performs_within()`，复杂场景用临时表计时
2. 阈值为本地测试的 3-5 倍（CI 环境波动）
3. 测试数据在事务内创建和清理
4. 大量数据使用 `generate_series`
5. 大量 INSERT 后执行 `ANALYZE` 再计时

---

### 4. Security Test（security）

文件命名：`{schema}_{object}_security_{desc}.sql`
编号：`SNNXXX` — NN = 类别，XXX = 组内序号

> 安全测试规范待细化，当前仅提供基础骨架。后期根据实际安全需求沉淀。

#### Security Test 模板

```sql
-- ================================================================
-- URI: test/db/{schema}_{object}_security_{desc}.sql
-- ================================================================
--
-- Security Test:  {schema} {object} {desc} security
-- Description:    {对象描述} 安全测试
-- Related Change: {sqitch_change_name}
--
-- Test Objects:
--   {sqitch_plan_name_1}               — {Object Category}
--
-- Test Group Overview:
--   S01000  {安全测试描述}
--
-- Usage:
--   pg_prove test/db/{schema}_{object}_security_{desc}.sql
--
-- Dependencies:
--   - pgTAP extension installed
--   - Sqitch deployed to latest change
--
-- ================================================================

\unset AUTOCOMMIT
\set ON_ERROR_STOP true

BEGIN;

SELECT no_plan();

-- ────────────────────────────────────────────────────────────────────
-- Security Tests
-- ────────────────────────────────────────────────────────────────────

-- TODO: Add security test cases

SELECT * FROM finish();
ROLLBACK;
```

---

## 常用 pgTAP 断言函数速查

| 函数 | 用途 |
|------|------|
| `plan(n)` / `no_plan()` / `finish()` | 计划声明与收尾 |
| `has_schema(name)` | schema 存在 |
| `has_table(schema, name)` | 表存在 |
| `has_pk(schema, table)` / `col_is_pk(schema, table, col)` | 主键存在 / 列是主键 |
| `has_column(schema, table, col)` / `columns_are(schema, table, cols[])` | 列存在 / 列集合与顺序 |
| `col_type_is(schema, table, col, type)` | 列类型 |
| `col_not_null(schema, table, col)` / `col_is_null(schema, table, col)` | 非空 / 可空 |
| `col_default_is(schema, table, col, default)` / `col_has_default(...)` / `col_hasnt_default(...)` | 默认值 |
| `has_check(schema, table)` / `col_has_check(schema, table, col)` | CHECK 约束 |
| `has_unique(schema, table)` / `col_is_unique(schema, table, col[, ...])` | 唯一约束 |
| `has_fk(schema, table)` / `col_is_fk(...)` / `fk_ok(...)` | 外键 |
| `has_index(schema, table, idx)` / `indexes_are(...)` / `index_is_unique(...)` | 索引 |
| `trigger_is(schema, table, trigger, func_schema, func_name)` | 触发器绑定 |
| `has_view(schema, view)` / `has_materialized_view(schema, view)` | 视图 / 物化视图存在 |
| `has_function(schema, func, arg_types[])` / `function_returns(...)` | 函数存在 / 返回类型 |
| `volatility_is(...)` / `is_strict(...)` / `isnt_strict(...)` / `is_definer(...)` / `isnt_definer(...)` / `function_lang_is(...)` | 函数属性 |
| `is(actual, expected, desc)` / `ok(cond, desc)` | 值相等 / 布尔断言 |
| `matches(actual, pattern, desc)` | 正则匹配 |
| `set_eq(...)` / `bag_eq(...)` / `is_empty(...)` | 集合比较 |
| `throws_ok(sql, errcode, desc)` / `throws_like(...)` / `throws_matching(...)` / `lives_ok(...)` | 异常断言 |
| `performs_ok(sql, ms, desc)` / `performs_within(...)` | 性能计时 |

## Known Pitfalls（常见陷阱）

1. `has_view()` 不检测 MView — 必须使用 `has_materialized_view()`
2. MView 列不在 `information_schema.columns` — 使用 `pg_attribute` + `pg_class`
3. Domain 类型 — `col_type_is()` 检查底层基础类型，非 DOMAIN 名
4. DEFAULT 值格式 — `information_schema` 返回含类型转换后缀（如 `'recurring'::text`）
5. `has_function()` 带参数 — 必须指定精确类型数组：`ARRAY['bytea', 'integer']`
6. `has_function()` 无参数 — 使用 `ARRAY[]::text[]`
7. `json_matches_schema` 返回 NULL — 用 `IS NOT TRUE`，非 `IS FALSE`
8. CHECK 约束格式 — PostgreSQL 存储 `IN (0,1)` 为 `= ANY (ARRAY[0, 1])`
9. CHECK 约束计数 — 使用 `>= N` 而非 `= N`
10. 归档表列对比 — 排除 `archived_at` 列后再对比
11. 约束注释 — 通过 `pg_constraint.oid` + `obj_description()` 获取
12. 列注释 — 使用 `col_description(table_oid, attnum)`
13. 后台任务会话隔离 — `pg_cron` 等后台任务，ROLLBACK 不回滚其写入，需显式清理
14. `count(*)` 类型 — pgTAP `is()` 要求两边类型一致，`count(*)` 返回 bigint，需 `::integer` 转换
15. `pg_temp` schema 名 — 实际名称为 `pg_temp_N`，匹配时使用 `LIKE 'pg_temp%'`
16. `PERFORM` 仅限 PL/pgSQL — 在纯 SQL 脚本中使用 `SELECT` 而非 `PERFORM`
17. `has_unique()` 不带列名 — 仅验证表是否有任意唯一约束，不够精确；T15 统一使用 `col_is_unique()` 指定具体列
18. **INSTEAD OF trigger 测试**：对视图执行 INSERT/UPDATE/DELETE 而非底层表，使用 `throws_ok()` 验证触发器拒绝逻辑（配合实际定义的自定义错误码，如 `P0001`）
19. **触发器门控测试**：先验证前置条件不满足时抛错，再验证数据不合规时抛错，最后验证合规数据通过
20. **软删除/恢复测试**：DELETE 触发恢复逻辑时，验证字段恢复到默认值且行仍存在（非物理删除）
21. **计算列保护测试**：UPDATE 视图时修改计算列应被拒绝
22. **INSTEAD OF trigger 返回值**：DELETE 恢复操作返回 NULL（行未实际删除），物理删除返回 OLD；集成测试需验证行是否存在性
23. **`ok()` + `FROM` 语法错误**：pgTAP `ok(boolean, desc)` 第一参数是标量布尔，不能内嵌 `FROM`；查表断言必须包成子查询 `ok((SELECT ... IS NOT NULL FROM ...), desc)`
24. **归档/镜像表约束**：由 `LIKE ... INCLUDING` 创建的镜像表，约束注释与源表同名重复；T31 仅校验源表，不单独校验镜像表
25. **MAX/MIN 不可聚合类型**：PG `MAX(boolean)` / `MAX(jsonb)` 等不存在，条件聚合需先 cast 到 text 再 cast 回
26. **`->` 操作符优先级**：测试数据准备 SQL 中 `value->'key'::text` 易误写，应 `(value->'key')::text`
27. **`col_default_is()` 不兼容 json 类型**：`json` 无 `=` 运算符（jsonb 有），`col_default_is()` 对 json 列报 `operator does not exist: json = json`；用 `information_schema.columns` 查询替代（`SELECT column_default::text FROM information_schema.columns WHERE ...`），jsonb 列正常使用 `col_default_is()`
28. **`col_is_not_null()` 不存在**：pgTAP 只有 `col_not_null(schema, table, column, desc)` 和 `col_is_null(schema, table, column, desc)`，不存在 `col_is_not_null` 函数
29. **NOT NULL/DEFAULT 计数**：表结构验证的分组标题必须标注正确的列总数和 NOT NULL/nullable 或 DEFAULT/no-DEFAULT 分计数（如「14 列：10 NOT NULL + 4 nullable」），与实际 `information_schema.columns` 一致
30. **identity 列（PG 18）**：`GENERATED ... AS IDENTITY` 列的默认值是内部序列 `nextval`，`col_default_is()` 无法精确断言；应通过 `information_schema.columns.is_identity = 'YES'` 或 `pg_attribute.attidentity`（`'a'`=ALWAYS / `'d'`=BY DEFAULT）校验
31. **`uuidv7()`（PG 18）**：作为默认值使用时（如 `id uuid DEFAULT uuidv7()`），用 `col_has_default()` 断言有默认值，再单独验证生成的 UUID 为版本 7（`uuid_extract_version() = 7`），避免直接比较具体值
32. **MERGE 幂等性测试**：upsert 场景优先用 `MERGE` 语句验证幂等（同一键重复执行不产生重复行、计数不变），比「先 DELETE 再 INSERT」更贴合实际语义

## 验证（完成标准）

- `pg_prove -d <db> test/db/` 退出码 0，全部通过
- 测试后测试库无残留对象（ROLLBACK 生效，后台任务写入已显式清理）
- 每个新增/修改的 change 都有对应测试覆盖
- 单元测试 36 分组齐全、5 Section 顺序正确、断言带 `[Txxxxx]` 前缀
