---
name: sql-test
description: 'Generate pgTAP test file skeletons for Bizsky database objects. Supports unit, integration, performance, and security test types. Use when the user says "generate test" or "create test file".'
---

# SQL Test Generator

Generate pgTAP test file skeletons following Bizsky testing conventions.

## Conventions

- All output in Chinese comments, English identifiers
- File paths resolve from `{project-root}`
- Reference doc: `docs/sql-test.md`（参考文档，不作为持久化依赖）

## Skill Type`技能类型`

- Category: 代码生成器
- Trigger: 当 task 描述包含 `[SKILL:sql-test:{type}]` 时调用
- Input: 测试类型 + 模块 + 子模块 + 目标对象
- Output: pgTAP 测试文件骨架
- Complement: [SKILL:sql-sqitch] — 生成对应的 Sqitch 迁移脚本

| Type | 编号 | 输出文件 | 说明 |
|------|------|---------|------|
| unit | TTXXX | `{module}_{child}_unit_{desc}.sql` | 36 固定分组，5 Sections，完整结构验证 |
| integration | AGGDNN + FNNXXX | `{module}_{child}_integration_{desc}.sql` | A01-A22 函数边界断言 + F01-F07 场景集成 |
| performance | GNNXXX | `{module}_{child}_performance_{desc}.sql` | G01-G04 NFR 类别，阈值验证 |
| security | SNNXXX | `{module}_{child}_security_{desc}.sql` | 安全属性验证（待细化） |

## On Activation

Ask the user for:

1. **Test type**: unit / integration / performance / security
2. **Module**: `{module}` (e.g., common, dict, log)
3. **Child**: `{child}` (e.g., ulid, config, exception)
4. **Desc**: `mini` / `mgmt` / or custom (e.g., `epic1_e2e_workflow`)
5. **Target objects** (optional): list tables, views, functions, seeds to test

If user provides insufficient info, infer from context or ask minimally.

## Output

Write file to: `test/db/{module}_{child}_{test_type}_{desc}.sql`

## Test Types

### 1. Unit Test (unit)

File naming: `{module}_{child}_unit_{desc}.sql`
Numbering: `TTXXX` — TT = group (01-36), XXX = sequence (001-999)

#### 36 Fixed Groups (5 Sections)

All 36 groups MUST appear in every unit test file, even if empty.

**Section A: Tables — 表结构验证（T01-T15）**

| TT | 名称 | 验证内容 | 级别 | pgTAP 函数 |
|----|------|---------|------|-----------|
| T01 | 表存在性验证 | 业务表 + 日志表 + 归档表 | MUST | `has_table()` |
| T02 | 主键验证 | 主键列名、类型 | MUST | `has_pk()`, `col_is_pk()` |
| T03 | 标准字段类型验证 | 项目标准字段（8/9 字段） | MUST | `col_type_is()` |
| T04 | 业务字段类型验证 | 模块特有业务字段 | MUST | `col_type_is()` |
| T05 | 字段顺序验证 | 列顺序与 Sqitch 定义一致 | MUST | `columns_are()` |
| T06 | NOT NULL 约束验证 | **每字段必须验证**：所有字段逐一使用 `col_not_null()`（NOT NULL 列）或 `col_is_null()`（可 NULL 列），禁止仅验证部分字段 | MUST | `col_not_null()`, `col_is_null()` |
| T07 | DEFAULT 值验证 | **每字段必须验证**：所有字段逐一使用 `col_default_is()`（有默认值）、`col_has_default()`（确认有默认值）、`col_hasnt_default()`（确认无默认值），禁止仅验证部分字段 | MUST | `col_default_is()`, `col_has_default()`, `col_hasnt_default()` |
| T08 | CHECK 约束存在性验证 | **双重验证**：表级 `has_check()` 验证约束总数 + 列级 `col_has_check()` 逐列验证含 CHECK 约束的列 | MUST | `has_check()`, `col_has_check()` |
| T09 | CHECK 约束内容验证 | 约束表达式是否正确 | MUST | `ok()` + `pg_get_constraintdef(oid)` |
| T10 | 表触发器绑定验证 | **表绑定的触发器**（如审计触发器），**必须使用 `trigger_is()` 5 参数**：`trigger_is(schema, table, trigger_name, func_schema, func_name)`，禁止使用 `has_trigger()` 替代 | MUST | `trigger_is()` |
| T11 | 索引验证 | 索引存在、唯一性（业务表+归档表） | SHOULD | `has_index()`, `indexes_are()`, `index_is_unique()` |
| T12 | 归档表 archived_at 列验证 | archived_at 类型/NOT NULL/DEFAULT | SHOULD | `col_type_is()`, `col_not_null()`, `col_default_is()` |
| T13 | 归档表列对比验证 | 源表所有列是否出现在归档表中 | SHOULD | `ok()` + `information_schema` 列对比 |
| T14 | FK 关系验证 | 外键关系完整性 | MUST | `fk_ok()`, `has_fk()`, `col_is_fk()` |
| T15 | 唯一约束验证 | UNIQUE 约束存在性和列组成 | MUST | `col_is_unique()` |

**Section B: Views — 视图结构验证（T16-T20）**

| TT | 名称 | 验证内容 | 级别 | pgTAP 函数 |
|----|------|---------|------|-----------|
| T16 | 视图存在性验证 | config_view, report, mview 等 | MUST | `has_view()`, `has_materialized_view()` |
| T17 | 字段类型验证 | **每字段必须验证**：视图/物化视图所有列逐一验证类型，MView 使用 `pg_attribute` 查询 | MUST | `col_type_is()`（普通视图），`pg_attribute` + `ok()`（物化视图） |
| T18 | 字段顺序验证 | **必须使用 `columns_are()`**：验证视图/物化视图所有列名和顺序，禁止使用 `pg_attribute` EXISTS 查询替代 | MUST | `columns_are()` |
| T19 | 物化视图索引验证 | MView 索引存在和唯一性 | SHOULD | `has_index()`, `indexes_are()` |
| T20 | INSTEAD OF 视图触发器绑定验证 | **视图绑定的 INSTEAD OF 触发器**，**必须使用 `trigger_is()` 5 参数**：`trigger_is(schema, view, trigger_name, func_schema, func_name)`，禁止使用 `has_trigger()` 替代 | MUST | `trigger_is()` |

**Section C: Functions — 函数结构验证（T21-T26）**

| TT | 名称 | 验证内容 | 级别 | pgTAP 函数 |
|----|------|---------|------|-----------|
| T21 | 函数存在性和入参验证 | 函数存在、参数类型 | MUST | `has_function()` |
| T22 | 返回值验证 | 返回类型精确匹配 | MUST | `function_returns()` |
| T23 | volatility 验证 | IMMUTABLE/STABLE/VOLATILE | MUST | `volatility_is()` |
| T24 | strict 验证 | STRICT/非STRICT | MUST | `is_strict()`, `isnt_strict()` |
| T25 | security 验证 | SECURITY DEFINER/INVOKER | MUST | `is_definer()`, `isnt_definer()` |
| T26 | 其他函数属性验证 | 语言、并行安全 | SHOULD | `function_lang_is()` |

**Section D: Seed — 种子数据验证（T27-T30）**

| TT | 名称 | 验证内容 | 级别 | pgTAP 函数 |
|----|------|---------|------|-----------|
| T27 | 种子分布 | 分类树存在、数据条数 | MUST | `is()`（过滤计数） |
| T28 | 种子格式 | JSONB 字段结构 | MUST | `is()`, `matches()` |
| T29 | 种子关联完整性 | 外键关联正确 | MUST | `is()`（join 计数） |
| T30 | 数据存在 | 关键数据项存在 | MUST | `is()`, `ok()` |

**Section E: Cross-Object — 跨对象验证（T31-T36）**

| TT | 名称 | 验证内容 | 级别 | pgTAP 函数 |
|----|------|---------|------|-----------|
| T31 | 表+约束注释验证 | **三段连续编号子结构**：T31xxx 前段表注释（每表 1 条）→ 中段 CHECK 约束注释（业务命名 `ck_*`，CONSTRAINT 形式用 `obj_description(oid, 'pg_constraint')`，逐条用 `(schema, table, conname)` 三元组定位避免 archive 镜像同名冲突）→ 末段 UNIQUE/FK 注释（INDEX 形式 `CREATE UNIQUE INDEX` 用 `obj_description(oid, 'pg_class')`，CONSTRAINT 形式 `ADD CONSTRAINT` 用 `obj_description(oid, 'pg_constraint')`）。`archive_*` 镜像表由 `LIKE` 继承不单独校验 | SHOULD | `ok()` + `obj_description()` + `pg_constraint` |
| T32 | 表列注释验证 | **所有表所有列必须逐一验证**：每张表的每个字段都使用 `col_description()` 验证注释存在，含全列聚合检查和 pid 已启用/未启用标记验证 | SHOULD | `ok()` + `col_description()` |
| T33 | 视图注释验证 | COMMENT ON VIEW | SHOULD | `ok()` + `obj_description()` |
| T34 | 视图列注释验证 | **所有视图所有列必须逐一验证**：视图/物化视图的所有字段使用 `col_description()` 验证注释存在，含全列聚合检查 | SHOULD | `ok()` + `col_description()` |
| T35 | 函数注释验证 | COMMENT ON FUNCTION | SHOULD | `ok()` + `obj_description()` |
| T36 | 相关项验证 | 跨模块依赖、兜底检查 | SHOULD | `has_schema()`, `ok()` |

#### Unit Test Template

Generate file with this structure:

```sql
-- ================================================================
-- URI: test/db/{module}_{child}_unit_{desc}.sql
-- ================================================================
--
-- Unit Test:     {module} {child} {desc} unit tests
-- Description:   {模块描述} 单元测试（Tables + Views + Functions + Seed）
-- Related Story: Story X.X — {Story 名称}
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
--   pg_prove test/db/{module}_{child}_unit_{desc}.sql
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

-- --- 日志表 ---
-- SELECT has_table('log', 'log_{type}', '[T01002] log.log_{type} 表应存在');

-- --- 归档表 ---
-- SELECT has_table('log', 'archive_{type}', '[T01003] log.archive_{type} 归档表应存在');

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

-- 说明: 此分组为空（{module}.{child} 无跨模块副作用需要验证）

SELECT * FROM finish();
ROLLBACK;
```

#### Unit Test Rules

**强制规则：**
1. 36 个分组必须全部出现在文件中，即使为空
2. 5 个 Section 必须按 A→B→C→D→E 顺序排列
3. TT 编号严格按规范，不得自定义或跳号
4. 断言消息必须带 `[Txxxxx]` 前缀（5 位）
5. MUST 级分组必须使用指定 pgTAP 函数，禁止 `ok()` + 子查询替代
6. 所有断言在单个事务内执行（BEGIN / ROLLBACK）
7. 种子测试避免无 WHERE 的 `count(*) = N`
8. T01 必须包含归档表存在性验证（无归档表则标注为空）
9. Test Objects 必须使用 sqitch plan 名竖排列表，每项标注对象分类（Table / Function / View / Seed Data 等）
10. Test Group Overview 必须列出所有非空测试分组及其编号和名称
11. **T08/T09 职责分离**: 仅 CHECK 约束（存在性 + 内容），禁止混入 UNIQUE/FK（后者分别由 T14/T15 负责，避免重叠）
12. **T14 MUST pgTAP 函数**: `has_fk(schema, table)` 存在性 + `fk_ok(schema, table, col, ref_schema, ref_table, ref_col)` 关系正确性
13. **T15 MUST pgTAP 函数**: `has_unique(schema, table)` 存在性 + `col_is_unique(schema, table, column)` 或 `col_is_unique(schema, table, ARRAY[...])` 复合 + `has_index` + `index_is_unique` 唯一索引

**空组处理：**
- 空组必须保留分组头注释，标注原因：`-- 说明: 此分组为空（{reason}）`
- 整个 Section 可跳过（模块无该类对象时）：Section 级注释说明

**分隔符规则：**

| 层级 | 格式 |
|------|------|
| Section | `-- ====================================================================` + `-- Section X: {名称} — {描述}（T{nn}-T{mm}）` + `-- ====================================================================` |
| Group | `-- ────────────────────────────────────────────────────────────────────` + `-- Test Group T{nn}: {名称}` + `-- ────────────────────────────────────────────────────────────────────` |
| Sub-group | `-- --- {描述} ---`（可选，组内对象切换时使用） |

---

### 2. Integration Test (integration)

File naming: `{module}_{child}_integration_{desc}.sql`

#### 编号体系

集成测试文件包含两个独立编号段，**boundary 分组（A）必须放在场景分组（F）之前**：

| 编号段 | 格式 | 说明 |
|--------|------|------|
| Boundary | `AGGDNN` | 1 函数 plan = 1 分组，边界断言全覆盖 |
| Scenario | `FNNXXX` | F01-F07 场景选取，行为链验证 |

#### Boundary Groups (A01-A99)

每个函数 plan（base_function / function / trigger / cronjob_function）对应一个独立 A 分组，按 sqitch.plan 部署顺序编号。

**编号格式：** `[AGGDNN]` — GG = 分组（01-99），D = 维度（1-8），NN = 组内序号（01-99）

**8 个固定维度（子分组）：**

| D | 维度 | 说明 | 典型用例数 |
|---|------|------|-----------|
| 1 | Happy path | 成功路径 + 返回值精确验证 | 1-5 |
| 2 | Input boundary | NULL/空串/超长/非法格式/边界值 | 5-15 |
| 3 | Error path | 每个 RAISE EXCEPTION / 每个失败分支逐一验证 | 3-15 |
| 4 | Side effect | 日志/审计/状态变更/物化视图刷新 | 2-10 |
| 5 | State transition | 状态机转换（不适用时标空） | 3-10 |
| 6 | Output schema | 返回值类型/结构/字段精确匹配 | 1-5 |
| 7 | Idempotency | 重复调用安全性 | 1-3 |
| 8 | Concurrency | 并发/锁/序列行为 | 1-3 |

**分组结构示例：**

```sql
-- ====================================================================
-- Boundary: 函数边界断言（A01-A22）
-- ====================================================================

-- ────────────────────────────────────────────────────────────────────
-- A13: minio_presign 边界断言
-- ────────────────────────────────────────────────────────────────────

-- --- A13-1: Happy path ---
-- [A13101] GET presign 成功 → o_is_ok=true
-- [A13102] PUT presign 成功 → o_is_ok=true

-- --- A13-2: Input boundary ---
-- [A13201] p_id = NULL → o_is_ok=false
-- [A13202] p_id = '' → o_is_ok=false

-- --- A13-3: Error path ---
-- [A13301] operation='DELETE' → o_is_ok=false
-- [A13302] 配置缺失 → o_is_ok=false

-- --- A13-4: Side effect ---
-- [A13401] 成功时 log_minio 有 is_ok=1 记录
-- [A13402] 失败时 log_minio 有 is_ok=0 + error 字段

-- --- A13-5: State transition ---
-- 说明: 此子分组为空（minio_presign 无状态转换）

-- --- A13-6: Output schema ---
-- [A13601] 返回 3 列: o_is_ok(BOOLEAN), o_url(TEXT), o_expires_at(TIMESTAMPTZ)

-- --- A13-7: Idempotency ---
-- 说明: 此子分组为空（只读操作，天然幂等）

-- --- A13-8: Concurrency ---
-- 说明: 此子分组为空（无写操作，无并发风险）
```

**Boundary 分组规则：**

1. 每个函数 plan 必须有独立 A 分组，即使测试用例较少
2. 8 个维度子分组必须全部出现，不适用时保留子分组头 + `说明: 此子分组为空（{reason}）`
3. INSTEAD OF trigger 函数的边界测试放在 A 分组中，不再使用 F02 场景
4. 每个错误路径（RAISE EXCEPTION / o_is_ok=false）必须有独立测试用例
5. Side effect 维度验证函数执行后的副作用（日志记录、审计记录、mview 刷新）
6. 编号按 sqitch.plan 部署顺序排列

---

#### Scenario Categories (F01-F07)

Numbering: `FNNXXX` — NN = scenario (01-07), XXX = sequence (001-999)

#### Scenario Categories (F01-F07)

| 场景 | 名称 | 验证链路 |
|------|------|---------|
| F01 | 配置管理集成 | config write → mview refresh → read |
| F02 | 触发器行为集成 | INSERT/UPDATE/DELETE → trigger → 预期结果 |
| F03 | 日志服务集成 | log_write → log table → query/filter |
| F04 | 异步调度集成 | schedule → execute → status |
| F05 | 归档迁移集成 | migrate → 数据迁移 → archive verify |
| F06 | 跨模块端到端 | config → log → audit → async → archive |
| F07 | 结果集完整性集成 | 数据写入 → 查询 → 结果集精确匹配 |

场景选取按模块需要，不强制全部使用，但**必须覆盖所有适用的场景**并包含充分的测试用例和断言，禁止仅使用 2 个场景敷衍。每个场景应有明确的测试步骤、充分的断言覆盖，不得仅有骨架占位。

#### Integration Test Template

```sql
-- ================================================================
-- URI: test/db/{module}_{child}_integration_{desc}.sql
-- ================================================================
--
-- Integration Test: {module} {child} {desc} integration
-- Description:     {模块描述} 集成测试（函数边界断言 + 行为链验证）
-- Related Story:   Story X.X — {Story 名称}
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
--   pg_prove test/db/{module}_{child}_integration_{desc}.sql
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
-- Boundary: 函数边界断言（A01-A22）
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

-- ... A02-A22 同理 ...

-- ====================================================================
-- Scenario F01: {场景名称}
-- ====================================================================

SELECT * FROM finish();
ROLLBACK;
```

#### Integration Test Rules

**通用规则：**
1. 所有数据变更必须在事务内（BEGIN / ROLLBACK）
2. 清理 DELETE 前必须 `SET LOCAL bizsky.bypass_audit = 'true'`
3. bypass_audit 测试后必须 `SET LOCAL bizsky.bypass_audit = 'false'` 重置，避免泄漏
4. 临时对象使用 `test.` schema
5. 推荐函数：`throws_ok()`, `throws_like()`, `throws_matching()`, `lives_ok()`, `is()`, `ok()`, `set_eq()`, `bag_eq()`, `is_empty()`
6. 禁止 `ok(true, ...)` 无意义断言
7. Test Objects 必须列出所有涉及的 sqitch plan，使用竖排列表 + 对象分类
8. Test Group Overview 必须列出所有 A 分组和 F 场景编号及名称，A 分组标识符**必须使用 sqitch plan name**（与 Test Objects 列表保持一致），禁止使用 PG 全限定名 `schema.object`；段头格式 `-- A01: {sqitch_plan_name} — {一句话功能描述}`

**Boundary 分组（A）规则：**
9. 每个函数 plan 必须有独立 A 分组，按 sqitch.plan 部署顺序编号
10. 8 个维度子分组必须全部出现，不适用时保留头 + 空原因说明
11. 每个 RAISE EXCEPTION / 每个失败分支必须有独立测试用例
12. Side effect 维度必须验证函数执行后的日志/审计/状态变更
13. 消息前缀格式：`[AGGDNN]`（A + 2位组号 + 1位维度 + 2位序号）

**Scenario 分组（F）规则：**
14. 场景选取按模块需要，不强制全部使用，但必须覆盖所有适用场景
15. 消息前缀格式：`[F0X001]`
16. **分支全覆盖**：INSTEAD OF trigger 的边界测试已移至 A 分组，F 场景聚焦跨对象行为链

---

### 3. Performance Test (performance)

File naming: `{module}_{child}_performance_{desc}.sql`
Numbering: `GNNXXX` — NN = NFR category (01-04), XXX = sequence (001-999)

#### NFR Categories (G01-G04)

| 类别 | 名称 | 典型 NFR |
|------|------|---------|
| G01 | 吞吐量测试 | ≥10000 ID/s, >1000 条/秒 |
| G02 | 延迟测试 | <50ms, ≤5ms |
| G03 | 批量测试 | 50000 行 ≤30s |
| G04 | 查询性能 | 报告查询 ≤5s |

#### Performance Test Template

```sql
-- ================================================================
-- URI: test/db/{module}_{child}_performance_{desc}.sql
-- ================================================================
--
-- Performance Test: {module} {child} {desc} performance
-- Description:     {模块描述} 性能基线测试（NFR 验证）
-- Related Story:   Story X.X — {Story 名称}
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
--   pg_prove test/db/{module}_{child}_performance_{desc}.sql
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

#### Performance Test Rules

1. 首选 `performs_ok()` / `performs_within()`，复杂场景用临时表计时
2. 阈值为本地测试的 3-5 倍（CI 环境波动）
3. 测试数据在事务内创建和清理
4. 大量数据使用 `generate_series`
5. 清理 DELETE 使用 `bypass_audit`

---

### 4. Security Test (security)

File naming: `{module}_{child}_security_{desc}.sql`
Numbering: `SNNXXX` — NN = category, XXX = sequence

> 安全测试规范待细化，当前仅提供基础骨架。后期根据实际安全需求沉淀。

#### Security Test Template

```sql
-- ================================================================
-- URI: test/db/{module}_{child}_security_{desc}.sql
-- ================================================================
--
-- Security Test:  {module} {child} {desc} security
-- Description:    {模块描述} 安全测试
-- Related Story:  Story X.X — {Story 名称}
--
-- Test Objects:
--   {sqitch_plan_name_1}               — {Object Category}
--
-- Test Group Overview:
--   S01000  {安全测试描述}
--
-- Usage:
--   pg_prove test/db/{module}_{child}_security_{desc}.sql
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

## Known Pitfalls

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
13. bypass_audit — 集成/性能测试清理 DELETE 前必须 `SET LOCAL bizsky.bypass_audit = 'true'`
14. bypass_audit 泄漏 — 测试 bypass 后必须 `SET LOCAL bizsky.bypass_audit = 'false'` 重置
15. pg_cron 会话隔离 — ROLLBACK 不回滚 cron.job，需显式清理
16. `count(*)` 类型 — pgTAP `is()` 要求两边类型一致，`count(*)` 返回 bigint，需 `::integer` 转换
17. `pg_temp` schema 名 — 实际名称为 `pg_temp_N`，匹配时使用 `LIKE 'pg_temp%'`
18. `PERFORM` 仅限 PL/pgSQL — 在纯 SQL 脚本中使用 `SELECT` 而非 `PERFORM`
19. `has_unique()` 不带列名 — 仅验证表是否有任意唯一约束，不够精确；T15 统一使用 `col_is_unique()` 指定具体列
20. **INSTEAD OF trigger 测试**: 对视图执行 INSERT/UPDATE/DELETE 而非底层表，使用 `throws_ok()` 验证触发器拒绝逻辑（60229/60229 错误码）
21. **触发器三重门控测试**: 先验证 schema 不存在时抛 `60229`，再验证数据不合规时抛 `60229`，最后验证合规数据通过
22. **default_value 恢复测试**: DELETE 有 default_value 的行后，验证 value/sys_status/sort/description 恢复为 default_value 中的值，且行仍存在（非物理删除）
23. **计算列保护测试**: UPDATE 视图时修改 value_json_schema/default_value_json_schema/details 列应抛 `60229`
24. **INSTEAD OF trigger 返回值**: DELETE 恢复操作返回 NULL（行未实际删除），物理删除返回 OLD；集成测试需验证行是否存在性
25. **`ok()` + `FROM` 语法错误**: pgTAP `ok(boolean, desc)` 第一参数是标量布尔，不能内嵌 `FROM`；查表断言必须包成子查询 `ok((SELECT ... IS NOT NULL FROM ...), desc)`
26. **archive 镜像表约束**: `log.archive_*` 由 `LIKE log.*` 创建，约束注释与 `log.*` 同名重复；T31 仅校验 `log.*`，不单独校验 archive.*
27. **MAX/MIN 不可聚合类型**: PG `MAX(boolean)` / `MAX(jsonb)` 等不存在，测试构造数据时若涉及条件聚合需先 cast 到 text 再 cast 回（参考 sqitch Known Pitfall #13）
28. **`->` 操作符优先级**: 测试数据准备 SQL 中 `value->'key'::text` 易误写，应 `(value->'key')::text`（参考 sqitch Known Pitfall #14）
29. **`col_default_is()` 不兼容 json 类型**: `json` 无 `=` 运算符（jsonb 有），`col_default_is()` 对 json 列报 `operator does not exist: json = json`；用 `information_schema.columns` 查询替代（`SELECT column_default::text FROM information_schema.columns WHERE ...`），jsonb 列正常使用 `col_default_is()`
30. **`col_is_not_null()` 不存在**: pgTAP 只有 `col_not_null(schema, table, column, desc)` 和 `col_is_null(schema, table, column, desc)`，不存在 `col_is_not_null` 函数
31. **NOT NULL/DEFAULT 计数**: 表结构验证的分组标题必须标注正确的列总数和 NOT NULL/nullable 或 DEFAULT/no-DEFAULT 分计数（如"14 列：10 NOT NULL + 4 nullable"），与实际 `information_schema.columns` 一致
