---
name: sql-sqitch
description: 'Generate Sqitch deploy/verify/revert SQL scripts for Bizsky database objects. Supports table, archive_table, view, base_function, function, helper, cronjob_function, control_function, mview_refresh_function, seed, all_analysis object types. Use when the user says "generate sqitch script" or "create deploy/revert/verify".'
---

# SQL Sqitch Script Generator

Generate Sqitch deploy/verify/revert SQL scripts following Bizsky conventions.

## Conventions

- All comments in Chinese
- File paths resolve from `{project-root}`
- Reference doc: `docs/sql-script.md`（参考文档，不作为持久化依赖）
- One plan per database object, no intermediate iterations

## Skill Type`技能类型`

- Category: 代码生成器
- Trigger: 当 task 描述包含 `[SKILL:sql-sqitch:{type}]` 时调用
- Input: 对象类型 + plan 名 + schema + 对象详情
- Output: deploy/verify/revert 三个 SQL 文件
- Complement: [SKILL:sql-test] — 生成对应的 pgTAP 测试文件

| Type | 模板分区 | 标准字段 | 说明 |
|------|---------|---------|------|
| table | 10 分区 | 9/10/7 字段 | 业务数据表（9 含 tenant_code）、日志表（10）、关联表（7） |
| archive_table | 2 分区 | LIKE 源表 | 归档表，`INCLUDING COMMENTS` + archived_at |
| view | 6 分区 | — | 视图（物化/管理/报告/分析/control，统一 6 分区结构） |
| base_function | 7 分区 | — | 基础函数：工具函数，性能优先，不强制日志记录和异常处理 |
| function | 7 分区 | — | 普通函数 / INSTEAD OF trigger 函数：thin 编排（外层调 _py/辅助）+ spec-exception 场景 4 + 三分式 val + config 分区二 STRICT+_chain + 显式入参 + trace_id/user_name 双主体 |
| helper | 多函数聚集 | — | helper：多函数聚集 sqitch change（调度函数 + val/qry/optfmt 业务函数 + daemon + _py），调度显式透传 + val 三分式 + COMMENT 每函数后 |
| cronjob_function | 7 分区 | — | 定时任务函数：固定签名 + 手工 {code,msg,data} 封装 + WHEN query_canceled RAISE + WHEN OTHERS log_err 吞（经 async_cronjob_helper 包装执行，不调 log_job） |
| control_function | 7 分区 | — | 控制函数：web_* schema，通过 PostgREST 向 Web 用户提供服务 |
| mview_refresh_function | 3 分区 | — | 物化视图刷新函数（函数定义/注释/附加操作） |
| seed | deploy 4分区 / revert 5分区 | — | 种子数据（GUC/写入/复原/附加） |
| all_analysis | deploy 3分区 / revert 3分区 | — | 全类型分析视图分支追加（每次新增日志类型生成 2 个 plan） |

> 类型模板随 Story 开发逐步沉淀，当前覆盖以上 11 种类型。INSTEAD OF trigger 函数是 `function` 的 variant，共享同一模板。`helper` 是多函数聚集类型（一个 sqitch change 含 10+ 函数），结构不同于单函数模板。

## On Activation

Ask the user for:

1. **Object type**: table / archive_table / view / base_function / function / helper / cronjob_function / control_function / mview_refresh_function / seed / all_analysis
2. **Script type**: deploy / verify / revert（or "all" for all three）
3. **Sqitch plan name**: e.g. `dict_sys_category_table`
4. **Target schema**: e.g. `dict`, `log`, `common`
5. **Object details** (varies by type, see templates below)

If user provides insufficient info, infer from context or ask minimally.

## Output

Write file(s) to:
- Deploy: `sqitch/deploy/{plan_name}.sql`
- Verify: `sqitch/verify/{plan_name}.sql`
- Revert: `sqitch/revert/{plan_name}.sql`

## General Rules

1. **Comment language**: All comments in Chinese
2. **Deploy COMMIT**: Always `COMMIT` — deploy must persist
3. **Verify ROLLBACK**: Always `ROLLBACK` — verify must not persist
4. **Revert COMMIT**: Always `COMMIT` — revert must persist deletions
5. **Revert seed COMMIT**: Always `COMMIT` — revert seed must persist deletions
6. **Template mark**: Only one template file per type retains the `Template:` mark; other files must not add it
7. **Verify scope**: Verify 仅验证对象存在性，详细验证由 pgTAP 测试脚本完成
8. **Verify framework**: 所有 verify 脚本必须使用 pgTAP 框架（`no_plan()` + `finish()`）
9. **Revert/Verify header**: 所有 revert/verify 脚本必须使用与 deploy 相同的标准头格式（URI / Description / Sqitch Plan / Target Schema / Related Story）

## Header Format

所有 deploy/revert/verify 脚本使用统一标准头：

```sql
-- ================================================================
-- URI: sqitch/{deploy|revert|verify}/{plan_name}.sql
-- ================================================================
--
-- Description:   {描述}
-- Sqitch Plan:   {plan_name}
-- Target Schema: {schema}
-- Related Story: Story X.X — {story 描述}
--
-- ================================================================
```

---

## Object Type 1: Table

### Table Variants

| Variant | Standard Fields | Count | Trigger |
|---------|----------------|-------|---------|
| 业务数据表 | tenant_code/id/pid/created_at/created_by/updated_at/updated_by/sys_status/extend | 9 | audit_trigger |
| 日志表 | tenant_code/id/pid/trace_id/request_id/log_level/log_type/created_at/created_by/extend | 10 | audit_trigger |
| 关联表 | tenant_code/id/{left}_id/{right}_id/created_at/created_by/extend | 7 | audit_trigger |

**所有表（业务数据表 / 日志表 / 关联表）必须绑定 `common.audit_trigger()`（BEFORE 触发器）。归档表 / 物化视图不绑定。**

日志表绑定规则（含 `log.log_audit` 自身）：触发器内部通过 3.1 审计写入条件 `NOT v_is_log_table` 显式跳过日志表，避免 log_audit 写 log_audit 的循环审计（[见：spec-audit Anti-recursion](../../_bmad-output/specs/spec-audit.md#anti-recursion)）。

### Deploy Template (10-Partition)

<!-- Skill Note: Known Patterns 仅作为 skill 生成指导，不出现在实际脚本中 -->
<!-- P1 业务数据表 9 标准字段(含 tenant_code), P2 日志表 10 标准字段, P3 关联表 7 标准字段, P4 审计触发器 -->

10 分区固定顺序，不可跳过、不可合并。空分区保留分区头 + 注释原因：`-- 无{name}（{reason}）`

```sql
-- ================================================================
-- URI: sqitch/deploy/{plan_name}.sql
-- ================================================================
--
-- Description:   {表功能说明}
-- Sqitch Plan:   {plan_name}
-- Target Schema: {schema}
-- Related Story: Story X.X
--
-- ================================================================

BEGIN;

-- ====================================================================
-- 一、标准字段
-- ====================================================================

CREATE TABLE {schema}.{table} (
    tenant_code TEXT NOT NULL DEFAULT 'system', -- 1. 租户标识（多租户预留，DEFAULT 'system'，Post-MVP RLS 启用；audit_trigger INSERT 时从 GUC bizsky.tenant_code 填充）
    id          TEXT PRIMARY KEY,          -- 2. ULID，触发器生成
    pid         TEXT DEFAULT NULL,         -- 3. 父级ID
    created_at  TIMESTAMPTZ NOT NULL,      -- 4. 触发器设置
    created_by  TEXT NOT NULL,             -- 5. 触发器设置
    updated_at  TIMESTAMPTZ,               -- 6. 触发器设置
    updated_by  TEXT,                      -- 7. 触发器设置
    sys_status  INTEGER NOT NULL DEFAULT 1 CHECK (sys_status IN (0,1)), -- 8. 0=停用, 1=正常
    extend      JSONB,                     -- 9. 扩展信息

-- ====================================================================
-- 二、业务字段（仅类型，约束移至分区六）
-- ====================================================================

    -- {business_field} {TYPE},

-- ====================================================================
-- 三、表注释
-- ====================================================================
);

COMMENT ON TABLE {schema}.{table} IS '{表功能说明}';

-- ====================================================================
-- 四、标准字段注释
-- ====================================================================

COMMENT ON COLUMN {schema}.{table}.tenant_code IS '租户标识（多租户预留，DEFAULT system，Post-MVP RLS 启用，audit_trigger 从 GUC bizsky.tenant_code 填充）';
COMMENT ON COLUMN {schema}.{table}.id IS '主键（ULID），由触发器生成';
COMMENT ON COLUMN {schema}.{table}.pid IS '父级ID';
-- ... (其他标准字段注释)

-- ====================================================================
-- 五、业务字段注释
-- ====================================================================

-- COMMENT ON COLUMN {schema}.{table}.{column} IS '{说明}';

-- ====================================================================
-- 六、业务字段约束（NOT NULL / DEFAULT / CHECK / UNIQUE / FK 集中）
-- ====================================================================
-- 注：NOT NULL / DEFAULT 是列级属性，无对应 COMMENT 语法（PG 限制）
--     CHECK / UNIQUE / FK 可命名后通过 COMMENT ON CONSTRAINT 注释（见分区七）
-- 空集时使用：-- 无业务字段约束（{reason}）

-- --- 6.1 NOT NULL ------------------------------------------------
-- ALTER TABLE {schema}.{table} ALTER COLUMN {column} SET NOT NULL;

-- --- 6.2 DEFAULT -------------------------------------------------
-- ALTER TABLE {schema}.{table} ALTER COLUMN {column} SET DEFAULT {value};

-- --- 6.3 CHECK ---------------------------------------------------
-- ALTER TABLE {schema}.{table} ADD CONSTRAINT ck_{table}_{desc} CHECK (...);

-- --- 6.4 UNIQUE / FK ---------------------------------------------
-- ALTER TABLE {schema}.{table} ADD CONSTRAINT uk_{table}_{desc} UNIQUE (...);
-- ALTER TABLE {schema}.{table} ADD CONSTRAINT fk_{table}_{desc} FOREIGN KEY (...) REFERENCES ...;

-- ====================================================================
-- 七、业务字段约束注释（仅 CHECK / UNIQUE / FK；NOT NULL / DEFAULT 无注释语法）
-- ====================================================================

-- COMMENT ON CONSTRAINT ck_{table}_{desc} ON {schema}.{table} IS '{说明}';
-- 或 -- 无（同六）

-- ====================================================================
-- 八、索引
-- ====================================================================

-- CREATE INDEX idx_{table}_{column} ON {schema}.{table} ({column});
-- 或 -- 无索引（{reason}）

-- ====================================================================
-- 九、索引注释
-- ====================================================================

-- COMMENT ON INDEX idx_{table}_{column} IS '{说明}';
-- 或 -- 无（同八）

-- ====================================================================
-- 十、触发器绑定
-- ====================================================================

CREATE TRIGGER trg_{table}_audit
    BEFORE INSERT OR UPDATE OR DELETE ON {schema}.{table}
    FOR EACH ROW EXECUTE FUNCTION common.audit_trigger();
-- 或 -- 无触发器（{reason}）

COMMIT;
```

### Verify Template

```sql
-- ================================================================
-- URI: sqitch/verify/{plan_name}.sql
-- ================================================================
--
-- Description:   验证 {表功能说明}
-- Sqitch Plan:   {plan_name}
-- Target Schema: {schema}
-- Related Story: Story X.X — {story 描述}
--
-- ================================================================

BEGIN;

SELECT no_plan();
SELECT has_table('{schema}', '{table}', '{schema}.{table} 表应存在');
SELECT * FROM finish();

ROLLBACK;
```

### Revert Template

```sql
-- ================================================================
-- URI: sqitch/revert/{plan_name}.sql
-- ================================================================
--
-- Description:   撤销 {表功能说明}
-- Sqitch Plan:   {plan_name}
-- Target Schema: {schema}
-- Related Story: Story X.X — {story 描述}
--
-- ================================================================

BEGIN;

DROP TABLE IF EXISTS {schema}.{table} CASCADE;

COMMIT;
```

---

## Object Type 2: Archive Table

### Deploy Template (2-Partition)

<!-- Skill Note: P1 表结构 (LIKE + archived_at), P2 对象注释 (TABLE + COLUMN) -->

```sql
-- ================================================================
-- URI: sqitch/deploy/{plan_name}.sql
-- ================================================================
--
-- Description:   {源日志表} 归档表
-- Sqitch Plan:   {plan_name}
-- Target Schema: log
-- Related Story: Story X.X
--
-- ================================================================

BEGIN;

-- ====================================================================
-- 一、表结构
-- ====================================================================
CREATE TABLE log.archive_{type} (
    LIKE log.log_{type} INCLUDING DEFAULTS INCLUDING CONSTRAINTS INCLUDING COMMENTS,
    archived_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (id)
);

-- ====================================================================
-- 二、对象注释
-- ====================================================================
COMMENT ON TABLE log.archive_{type} IS '{源日志表} 归档表';
COMMENT ON COLUMN log.archive_{type}.archived_at IS '归档时间';

COMMIT;
```

### Verify Template

```sql
-- ================================================================
-- URI: sqitch/verify/{plan_name}.sql
-- ================================================================
--
-- Description:   验证 {源日志表} 归档表
-- Sqitch Plan:   {plan_name}
-- Target Schema: log
-- Related Story: Story X.X — {story 描述}
--
-- ================================================================

BEGIN;

SELECT no_plan();
SELECT has_table('log', 'archive_{type}', 'log.archive_{type} 归档表应存在');
SELECT * FROM finish();

ROLLBACK;
```

### Revert Template

```sql
-- ================================================================
-- URI: sqitch/revert/{plan_name}.sql
-- ================================================================
--
-- Description:   撤销 {源日志表} 归档表
-- Sqitch Plan:   {plan_name}
-- Target Schema: log
-- Related Story: Story X.X — {story 描述}
--
-- ================================================================

BEGIN;

DROP TABLE IF EXISTS log.archive_{type} CASCADE;

COMMIT;
```

---

## Object Type 3: View

### View Variants

| 类别 | 说明 | 特征 |
|------|------|------|
| 物化视图 (mview) | 配置查询 / 统计报表 | 索引 + 定期刷新 |
| 管理视图 (view) | 基于 CRUD 操作的视图 | INSTEAD OF trigger |
| 报告视图 (report) | CTE 聚合 | 只读 |
| 分析视图 (analysis) | UNION ALL 在线+归档 | 只读 |
| Control 视图 (control) | 控制逻辑 | 按需定义 |

### View Column Templates`视图列模版`

管理视图根据业务特征使用不同列模版：

| Template | 列数 | 适用场景 | 差异说明 |
|----------|------|---------|---------|
| A-Data | 13 | 数据 CRUD 管理视图（config/custom 等数据子模块） | 全列，含 path + value_json_schema + default_value + default_value_json_schema |
| A-S | 9 | JSON Schema 管理视图（json_schema 子模块） | 移除 4 列（见下方说明） |

#### Template A-Data (13 列)

| 序号 | 列名 | 类型 | 来源 | 说明 |
|------|------|------|------|------|
| 1 | id | text | sys_data.id | 数据条目唯一标识 |
| 2 | category | text | 递归拼接 code | 所属分类路径 |
| 3 | path | text | 递归拼接 data.label | 数据层级路径 |
| 4 | label | text | sys_data.label | 数据条目标签 |
| 5 | active | integer | LEAST(chain) | 综合生效状态 |
| 6 | value | json | sys_data.value | 业务值 |
| 7 | value_json_schema | json | 关联查询 | value 的 JSON Schema 校验规则 |
| 8 | default_value | json | sys_data.default_value | 默认值 |
| 9 | default_value_json_schema | json | 关联查询 | default_value 的 JSON Schema 校验规则 |
| 10 | sys_status | integer | sys_data.sys_status | 数据条目原始状态码 |
| 11 | sort | integer | sys_data.sort | 排序权重 |
| 12 | description | jsonb | sys_data.description | 描述（多语言） |
| 13 | details | jsonb | to_jsonb(sys_data) | 原始行完整快照 |

#### Template A-S (9 列)

| 序号 | 列名 | 类型 | 来源 | 说明 |
|------|------|------|------|------|
| 1 | id | text | sys_data.id | 数据条目唯一标识 |
| 2 | category | text | 递归拼接 code | 所属分类路径 |
| 3 | label | text | sys_data.label | 数据条目标签 |
| 4 | active | integer | LEAST(chain) | 综合生效状态 |
| 5 | value | json | sys_data.value | JSON Schema 定义内容 |
| 6 | sys_status | integer | sys_data.sys_status | 数据条目原始状态码 |
| 7 | sort | integer | sys_data.sort | 排序权重 |
| 8 | description | jsonb | sys_data.description | 描述（多语言） |
| 9 | details | jsonb | to_jsonb(sys_data) | 原始行完整快照 |

**Template A-S 移除列及原因：**
- `path` — json_schema 为扁平结构，无数据层级路径
- `value_json_schema` — json_schema 条目自身即为 Schema，无外部校验规则
- `default_value` — Schema 条目无默认值概念，固定为 NULL
- `default_value_json_schema` — 无默认值，固定为 NULL

### Design Rules

- **CTE 优先**：视图定义使用 CTE（WITH / WITH RECURSIVE）实现递归和层级，不使用子函数
- 空分区规则与表相同：保留分区头 + 注释原因

### Deploy Template (6-Partition)

```sql
-- ================================================================
-- URI: sqitch/deploy/{plan_name}.sql
-- ================================================================
--
-- Description:   {视图功能说明}
-- Sqitch Plan:   {plan_name}
-- Target Schema: {schema}
-- Related Story: Story X.X
--
-- ================================================================

BEGIN;

-- ====================================================================
-- 一、视图定义
-- ====================================================================

-- MVIEW: CREATE MATERIALIZED VIEW {schema}.{view_name} AS ...
-- VIEW:  CREATE [OR REPLACE] VIEW {schema}.{view_name} AS ...
-- 使用 CTE 实现递归/层级，不使用子函数

-- ====================================================================
-- 二、索引
-- ====================================================================

-- MVIEW: CREATE UNIQUE INDEX uk_{view_name} ON {schema}.{view_name} (...);
-- MVIEW: CREATE INDEX idx_{view_name}_{column} ON {schema}.{view_name} ({column});
-- VIEW:  -- 无索引（视图不支持索引）

-- ====================================================================
-- 三、索引注释
-- ====================================================================

-- COMMENT ON INDEX {schema}.uk_{view_name} IS '{说明}';
-- 或 -- 无（同二）

-- ====================================================================
-- 四、视图注释
-- ====================================================================

-- MVIEW: COMMENT ON MATERIALIZED VIEW {schema}.{view_name} IS '{说明}';
-- VIEW:  COMMENT ON VIEW {schema}.{view_name} IS '{说明}';

-- ====================================================================
-- 五、列注释
-- ====================================================================

-- COMMENT ON COLUMN {schema}.{view_name}.{column} IS '{说明}';

-- ====================================================================
-- 六、附加操作
-- ====================================================================

-- MVIEW: REFRESH MATERIALIZED VIEW {schema}.{view_name};
-- 或 -- 无附加操作（{reason}）

COMMIT;
```

### Verify Template

```sql
-- ================================================================
-- URI: sqitch/verify/{plan_name}.sql
-- ================================================================
--
-- Description:   验证 {视图功能说明}
-- Sqitch Plan:   {plan_name}
-- Target Schema: {schema}
-- Related Story: Story X.X — {story 描述}
--
-- ================================================================

BEGIN;

SELECT no_plan();
-- MVIEW
-- SELECT has_materialized_view('{schema}', '{view_name}', '{schema}.{view_name} 物化视图应存在');
-- VIEW
-- SELECT has_view('{schema}', '{view_name}', '{schema}.{view_name} 视图应存在');
SELECT * FROM finish();

ROLLBACK;
```

### Revert Template

```sql
-- ================================================================
-- URI: sqitch/revert/{plan_name}.sql
-- ================================================================
--
-- Description:   撤销 {视图功能说明}
-- Sqitch Plan:   {plan_name}
-- Target Schema: {schema}
-- Related Story: Story X.X — {story 描述}
--
-- ================================================================

BEGIN;

-- MVIEW: DROP MATERIALIZED VIEW IF EXISTS {schema}.{view_name} CASCADE;
-- VIEW:  DROP VIEW IF EXISTS {schema}.{view_name} CASCADE;

COMMIT;
```

---

## Object Type 4: Base Function

### Design Rules

- **性能优先**：避免不必要的异常捕获，核心路径无 try-catch
- **不抛异常**：错误场景静默处理或返回 NULL/空结果
- **不严格要求日志记录**：写入失败不影响主流程
- **7 分区统一结构**：一～五在函数体内，六～七在脚本层
- 空分区保留分区头 + 注释原因：`-- 无（{reason}）`
- 子分区使用阿拉伯数字编号：2.1, 2.2, 3.1, 3.2 ...

### Deploy Template (7-Partition)

```sql
-- ================================================================
-- URI: sqitch/deploy/{plan_name}.sql
-- ================================================================
--
-- Description:   {函数功能说明}
-- Sqitch Plan:   {plan_name}
-- Target Schema: {schema}
-- Related Story: Story X.X — {story 描述}
--
-- ================================================================

BEGIN;

CREATE OR REPLACE FUNCTION {schema}.{func_name}(
    -- IN/OUT 参数
)
LANGUAGE plpgsql
AS $$
DECLARE
    -- 局部变量
BEGIN
    -- ====================================================================
    -- 一、入参校验
    -- ====================================================================
    -- 或 -- 无（{reason}）

    -- ====================================================================
    -- 二、资源查询校验与整理
    -- ====================================================================
    -- 或 -- 无（{reason}）

    -- ====================================================================
    -- 三、业务逻辑处理
    -- ====================================================================

    -- --- 3.1 子逻辑名称 ------------------------------------------------

    -- ====================================================================
    -- 四、返回值处理
    -- ====================================================================
    -- 或 -- 无（{reason}）

    -- ====================================================================
    -- 五、特殊内部函数
    -- ====================================================================
    -- 无
END;
$$;

-- ====================================================================
-- 六、注释
-- ====================================================================

COMMENT ON FUNCTION {schema}.{func_name}({param_types}) IS
    E'{函数功能说明}\n\n'
    '参数&返回值:'
    '  IN  p_{param}(TYPE): 参数说明'
    '  OUT o_{param}(TYPE): 返回值说明'
    '\n\n'
    '设计说明:'
    '  设计要点说明';

-- ====================================================================
-- 七、附加操作
-- ====================================================================
-- 或 -- 无附加操作（{reason}）

COMMIT;
```

### Verify Template

```sql
-- ================================================================
-- URI: sqitch/verify/{plan_name}.sql
-- ================================================================
--
-- Description:   验证 {函数功能说明}
-- Sqitch Plan:   {plan_name}
-- Target Schema: {schema}
-- Related Story: Story X.X — {story 描述}
--
-- ================================================================

BEGIN;

SELECT no_plan();
-- 无参函数
SELECT has_function('{schema}', '{func_name}', ARRAY[]::text[], '{func_name} 应存在');
-- 有参函数
-- SELECT has_function('{schema}', '{func_name}', ARRAY['type1', 'type2'], '{func_name} 应存在');
SELECT * FROM finish();

ROLLBACK;
```

### Revert Template

```sql
-- ================================================================
-- URI: sqitch/revert/{plan_name}.sql
-- ================================================================
--
-- Description:   撤销 {函数功能说明}
-- Sqitch Plan:   {plan_name}
-- Target Schema: {schema}
-- Related Story: Story X.X — {story 描述}
--
-- ================================================================

BEGIN;

DROP FUNCTION IF EXISTS {schema}.{func_name}({param_types}) CASCADE;

COMMIT;
```

---

## Object Type 5: Function

### Design Rules

- **入参三分类**：函数入参包含三种来源，统一在分区一（入参校验）中处理
  - **显式入参**：函数签名中的 `IN/OUT` 参数
  - **隐式入参**：PostgreSQL 自动注入的触发器变量（`TG_OP`、`NEW`、`OLD`、`TG_TABLE_NAME` 等）
  - **GUC 入参**：通过 `current_setting()` 读取的会话配置（如 `bizsky.user_name`、`bizsky.bypass_audit`）
- **完整防御链**：入参校验 → 资源查询 → 业务逻辑 → 异常处理 → 日志记录
- **统一异常处理**：场景 4，使用 `EXCEPTION WHEN OTHERS` + `log.log_err()` 捕获转换异常，普通函数必须 RAISE 重抛，控制函数可返回结构化错误
- **结构化注释**：COMMENT ON FUNCTION 包含参数&返回值、异常处理、设计说明
- **7 分区统一结构**：一～五在函数体内，六～七在脚本层
- 空分区保留分区头 + 注释原因：`-- 无（{reason}）`
- 子分区使用阿拉伯数字编号：2.1, 2.2, 3.1, 3.2 ...
- **thin 编排（普通函数）**：外层函数只负责编排（调内层 _py + 辅助 log_operation），不内联所有逻辑
  - 内层 `_py` 函数（LANGUAGE plpython3u）：纯外部调用（boto3/HTTP），异常透传上层，不做业务校验
  - 辅助函数（`log_operation` / `jsonb_textarray` 等）：纯工具，被外层调用
- **spec exception 三件套（分区一/二使用）**：
  - `v_log_info JSONB`（DECLARE 段声明）— 业务上下文
  - `_chain` JSONB — 配合 `GET STACKED DIAGNOSTICS` 捕获底层异常链（10 个字段）
  - 三分式 RAISE — `>>>` 分隔（field/category/reason）的 v_violation + v_log_info + RAISE jsonb MESSAGE
- **config 读分区二 2.1/2.2（资源查询）**：
  - 2.1：`SELECT INTO STRICT` + `EXCEPTION WHEN OTHERS` 包裹（缺行抛 80139 + `_chain`）
  - 2.2：必要字段校验（STRICT 仅保证 row 存在，不保证 JSONB key 存在，缺失抛 80139）
- **显式入参（spec §480）**：函数签名参数必须显式标量（`p_endpoint TEXT`、`p_rows JSONB`），不使用打包 JSONB 隐式取字段
- **trace_id + user_name 双主体（分区一）**：
  - `trace_id`：分区一设置（gen ulid 或从入参/GUC 透传）
  - `user_name`：按优先级 `Operator(固定) > Delegator(原 GUC)` 取值，分区一固定

### Exception Handling Specification`异常处理规范`

普通函数和 INSTEAD OF trigger 函数必须实现以下异常处理机制：

**1. 异常准备（DECLARE 块，必须）**：
- `v_location CONSTANT TEXT := '{plan_name}'` — 来源位置标识
- `v_log_info JSONB` — 日志信息
- 10 个 PG 诊断变量：`v_message_text` / `v_sqlstate` / `v_exception_detail` / `v_exception_hint` / `v_exception_context` / `v_schema_name` / `v_table_name` / `v_column_name` / `v_constraint_name` / `v_datatype_name`
- `r_err RECORD` — log_err 返回值

**2. 自定义异常（场景 2，入参校验/查询校验分区使用）**：
- 先赋值 `v_log_info := jsonb_build_object('op', TG_OP, 'reason', '{描述}', ...)` 构建业务上下文
- 再抛出 `RAISE EXCEPTION USING MESSAGE = (jsonb_build_object('_location', v_location) || v_log_info)::text, ERRCODE = '{sqlstate}'`
- JSON **必须**包含 `_location` 字段（通过 `jsonb_build_object('_location', v_location) || v_log_info` 合并）
- 示例：
  ```sql
  v_log_info := jsonb_build_object('op', TG_OP, 'reason', 'label is immutable');
  RAISE EXCEPTION USING
      MESSAGE = (jsonb_build_object('_location', v_location) || v_log_info)::text,
      ERRCODE = '60229';
  ```
- 自定义异常会被场景 4 的 `EXCEPTION WHEN OTHERS` 捕获，经 `log.log_err()` 记录后重抛
- **禁止**使用不带 ERRCODE 的 `RAISE EXCEPTION`
- **禁止**使用简单字符串 MESSAGE（如 `USING MESSAGE = '标签不可变'`）

**3. 统一异常处理（场景 4，函数末尾 EXCEPTION WHEN OTHERS）**：
- `GET STACKED DIAGNOSTICS` 捕获 10 个 PG 异常字段
- 调用 `log.log_err(10 个异常字段, v_location)` 记录日志并获取转换后信息
- 普通函数 / trigger 函数：`RAISE EXCEPTION USING MESSAGE = r_err.o_pg_message_text || '(' || v_location || ':' || r_err.o_log_id || ')', ERRCODE = r_err.o_pg_sqlstate` 重抛
- 控制函数：返回结构化错误 JSONB（不重抛）

**4. 异常编码约定**（遵循 `spec-exception.md`）：

5 位 SQLSTATE 编码 = **来源(1) + schema(2) + 分类(2)**：

| 位 | 取值 | 含义 |
|---|---|---|
| 第 1 位（来源） | `6` | 用户来源错误（入参/校验失败） |
|                | `8` | 系统来源错误（资源/内部错误） |
| 第 2-3 位（schema） | `01` | common 模块 |
|                     | `02` | dict 模块 |
|                     | `03` | log 模块 |
| 第 4-5 位（分类） | `19` | 鉴权失败（保留，view_trigger 当前不使用） |
|                   | `29` | 入参校验失败（分区一） |
|                   | `39` | 资源查询失败（分区二） |

**常用编码**：

| Code | 模块 | 触发场景 |
|---|---|---|
| `60129` | common | 入参校验失败（视图 trigger 字段保护/不可变/计算列/枚举校验等） |
| `60229` | dict   | 入参校验失败（schema 不存在、字段不可变、default_value 保护等） |
| `60239` | dict   | 资源查询失败（**用户责任**：path 解析、code 唯一、删除许可、schema 变更兼容等用户操作引发的校验） |
| `60329` | log    | 入参校验失败 |
| `80139` | common | 资源查询失败（配置加载 STRICT 失败，场景 3 异常链传播） |
| `80239` | dict   | 资源查询失败（**系统责任**：config 加载、根节点存在、seed 资源健康检查等系统资源校验） |
| `80339` | log    | 资源查询失败 |
| `80000` | —      | 系统内部错误（兜底） |

**分区对应关系**：
- 分区一（入参校验）抛 `6{schema}29`
- 分区二（资源查询）按**责任归属**抛：
  - 系统资源校验（config 加载、根节点/seed 资源）→ `8{schema}39`
  - 用户操作引发的资源校验（path 解析、code 唯一、删除许可、schema 变更兼容）→ `6{schema}39`
- 分区三（业务逻辑）通常转入场景 4 统一处理，自定义异常按场景归类

### Function Variants

| Variant | 返回类型 | 特征 |
|---------|---------|------|
| 普通函数 | 自定义 | 标准模板 |
| INSTEAD OF trigger | trigger | 隐式入参 TG_OP/NEW/OLD；分区七含触发器绑定；revert 先 DROP TRIGGER |

**INSTEAD OF trigger 函数**通过 plan 名模式 `_view_trigger` 识别，与普通函数共享 7 分区模板，差异仅在入参来源和附加操作：
- **入参**：无显式参数，隐式入参为 `TG_OP`（操作类型）、`NEW`（新行）、`OLD`（旧行），均在分区一校验
- **分区一～三**：内含 `IF TG_OP = 'INSERT' THEN ... ELSIF TG_OP = 'UPDATE' THEN ... ELSIF TG_OP = 'DELETE' THEN ... END IF;` 分支，每条分支按校验→查询→业务逻辑组织
- **分区四**（返回值）：INSERT/UPDATE → `RETURN NEW`；DELETE 恢复 → `RETURN NULL`；DELETE 物理删除 → `RETURN OLD`
- **分区七**（附加操作）：`CREATE TRIGGER ... INSTEAD OF INSERT OR UPDATE OR DELETE ON {view} FOR EACH ROW EXECUTE FUNCTION ...`
- **Revert**：先 `DROP TRIGGER` 再 `DROP FUNCTION`（Known Pitfall #6）
- 具体模式（三重门控、default_value 保护、immutable_schemas 等）参考 `spec-dict.md` Management View Trigger 章节

### View Trigger Validation Patterns`视图 Trigger 校验模式`

INSTEAD OF trigger 函数必须遵守以下 4 个统一模式（参考 `common_cache_target_view_trigger.sql` 作为模版）：

**前置 — SECURITY DEFINER（安全基础，必须）**：
- 所有管理视图 trigger 函数**必须** `SECURITY DEFINER`
- **原因**：强制所有写入经 trigger 校验（分区一字段保护 + 分区二 schema 门控 + 资源校验），调用者无法绕过视图直接写 `sys_category`/`sys_data`（脏数据防护）
- **配套**：revoke 表权限 + grant 视图权限 → 最小权限（PostgREST 角色只能通过视图写）
- **TODO**：`SET search_path` 防劫持待安全模块补全（当前不做，`log_err` 已有参照）
- 示例：
  ```sql
  CREATE OR REPLACE FUNCTION dict.xxx_view_trigger()
  RETURNS trigger
  LANGUAGE plpgsql
  SECURITY DEFINER          -- 必需：以 owner 权限执行，强制 trigger 校验
  AS $$ ... $$;
  ```

**模式 1 — CASE 链 + 单 RAISE（DRY，分区一）**：
- INSERT/UPDATE 字段校验使用 CASE 链找首个违规字段，配合单个 RAISE 模板
- **三部分编码**：`'<字段名>>><分类>>><描述>'`（field/category/reason，用 `>>` 分隔）
  - 字段名：违规字段（如 `id`、`path`、`created_at`）
  - 分类：`只读字段` / `计算列` / `无推荐值必填字段` / `不可变字段` 等
  - 描述：完整中文 reason（如 `INSERT 时必须为 NULL`、`UPDATE 时不能修改（移动需先删除后新增）`）
- v_log_info 输出三字段：`field`/`category`/`reason`（各用 `split_part(v_violation, '>>', N)` 解析）
- INSERT 校验集合：只读字段（id/审计字段）+ 计算列必须为 NULL；无推荐值必填字段必须为 NOT NULL
- UPDATE 校验集合：id + 审计字段 + 业务不可变字段（含 `IS DISTINCT FROM` 比较）
- **⚠ json 类型比较 pitfall**：`json` 类型无 `=` 运算符，`IS DISTINCT FROM` 内部用 `=` 会抛 `42883 operator does not exist: json = json`。json 字段（value / default_value / value_schema / default_value_schema）比较必须 `::text`：`(NEW.value::text) IS DISTINCT FROM (OLD.value::text)`。jsonb 虽有 `=` 运算符，但为统一建议也 `::text`
- 必须把 `id` 放在 CASE 链首位（id 不可变是所有 trigger 的基础约束）
- 三部分格式优势：分类独立便于聚合分析，reason 可按字段定制（尤其 UPDATE 不可变字段的差异化提示）
- 示例：
  ```sql
  -- INSERT：CASE 链三部分编码（只读字段/计算列必须 NULL，必填字段必须 NOT NULL）
  v_violation := CASE
      WHEN NEW.id              IS NOT NULL THEN 'id>>只读字段>>INSERT 时必须为 NULL'
      WHEN NEW.created_at      IS NOT NULL THEN 'created_at>>只读字段>>INSERT 时必须为 NULL'
      WHEN NEW.relation_exists IS NOT NULL THEN 'relation_exists>>计算列>>INSERT 时必须为 NULL'
      WHEN NEW.code            IS NULL     THEN 'code>>无推荐值必填字段>>INSERT 时必须为 NOT NULL'
      ELSE NULL
  END;
  IF v_violation IS NOT NULL THEN
      v_log_info := jsonb_build_object(
          'op',       TG_OP,
          'field',    split_part(v_violation, '>>', 1),
          'category', split_part(v_violation, '>>', 2),
          'reason',   split_part(v_violation, '>>', 3)
      );
      RAISE EXCEPTION USING
          MESSAGE = (jsonb_build_object('_location', v_location) || v_log_info)::text,
          ERRCODE = '60129';
  END IF;

  -- UPDATE：CASE 链三部分编码（reason 可定制差异化提示）
  v_violation := CASE
      WHEN NEW.id         IS DISTINCT FROM OLD.id         THEN 'id>>不可变字段>>UPDATE 时不能修改'
      WHEN NEW.created_at IS DISTINCT FROM OLD.created_at THEN 'created_at>>不可变字段>>UPDATE 时不能修改'
      WHEN NEW.path       IS DISTINCT FROM OLD.path       THEN 'path>>不可变字段>>UPDATE 时不能修改（移动需先删除后新增）'
      ELSE NULL
  END;
  ```

**模式 2 — 资源查询 STRICT + 场景 3 异常链（分区二）**：
- 配置加载（`common>{module}>config>project` 等系统配置）必须使用 `SELECT INTO STRICT` + `EXCEPTION WHEN OTHERS` 包裹
- **按需加载优化**：仅当推荐值字段存在 NULL 时才加载（`IF TG_OP = 'INSERT' AND (NEW.xxx IS NULL OR NEW.yyy IS NULL) THEN`），全非 NULL 时跳过 DB 查询
- 任何失败（缺行、视图不存在、权限等）按 spec-exception 场景 3 抛 80139（系统错误/资源查询失败）
- 异常 MESSAGE **必须**包含 `_chain` JSONB（10 个 GET STACKED DIAGNOSTICS 字段）+ `_location` + 业务上下文
- 禁止使用硬编码 fallback（如 `COALESCE(..., 30)`）掩盖配置缺失
- STRICT 仅保证 row 存在，**不保证 JSONB key 存在**。读时必须显式校验必要字段（防御历史数据/手工改库），缺失抛 80139：
  ```sql
  IF NOT (v_config ? 'default_xxx' AND v_config ? 'default_yyy') THEN
      v_log_info := jsonb_build_object(
          'op', TG_OP,
          'reason', 'xxx 配置缺少必要字段',
          'category', 'common>xxx>config',
          'missing_keys', (
              SELECT jsonb_agg(k) FROM (VALUES
                  ('default_xxx'),
                  ('default_yyy')
              ) AS t(k)
              WHERE NOT v_config ? t.k
          )
      );
      RAISE EXCEPTION USING
          MESSAGE = (jsonb_build_object('_location', v_location) || v_log_info)::text,
          ERRCODE = '80139';
  END IF;
  ```
- 配合 JSON Schema required 数组（写时防御）形成双层保护：写时 schema 阻断 + 读时 trigger 兜底
- 示例：
  ```sql
  IF TG_OP = 'INSERT' THEN
      BEGIN
          SELECT value INTO STRICT v_config
          FROM dict.cgservice_mview
          WHERE category = 'common>cache>config' AND label = 'project' AND active = 1;
      EXCEPTION WHEN OTHERS THEN
          GET STACKED DIAGNOSTICS
              v_message_text      = MESSAGE_TEXT,
              v_sqlstate          = RETURNED_SQLSTATE,
              v_exception_detail  = PG_EXCEPTION_DETAIL,
              v_exception_hint    = PG_EXCEPTION_HINT,
              v_exception_context = PG_EXCEPTION_CONTEXT,
              v_schema_name       = SCHEMA_NAME,
              v_table_name        = TABLE_NAME,
              v_column_name       = COLUMN_NAME,
              v_constraint_name   = CONSTRAINT_NAME,
              v_datatype_name     = PG_DATATYPE_NAME;
          v_log_info := jsonb_build_object(
              'op', TG_OP,
              'reason', 'cache 配置读取失败（common>cache>config>project）',
              'category', 'common>cache>config'
          );
          RAISE EXCEPTION USING
              MESSAGE = (jsonb_build_object(
                  '_chain', jsonb_build_object(
                      'message_text',      v_message_text,
                      'sqlstate',          v_sqlstate,
                      'exception_detail',  v_exception_detail,
                      'exception_hint',    v_exception_hint,
                      'exception_context', v_exception_context,
                      'schema_name',       v_schema_name,
                      'table_name',        v_table_name,
                      'column_name',       v_column_name,
                      'constraint_name',   v_constraint_name,
                      'datatype_name',     v_datatype_name
                  ),
                  '_location', v_location
              ) || v_log_info)::text,
              ERRCODE = '80139';
      END;
  END IF;
  ```

**模式 3 — 分区二应用默认值（入参整理 / 整理层）**：
- 依赖资源查询结果的默认值应用，**必须**在分区二完成（紧跟 STRICT 加载之后）
- **整理层概念**：分区二的默认值应用是"整理层"（仅赋值，不抛异常），与校验层（RAISE）显式分离；整理后的 NEW 才能进入分区三的业务逻辑，保证分区三纯净
- 形式：`NEW.{field} := COALESCE(NEW.{field}, (v_config->>'{key}')::{type});`
- **json 类型 cast**：config 推荐值（`v_config->'key'`）是 jsonb，若目标视图列是 json 类型，必须 `::json` cast（如 `COALESCE(NEW.value_schema, (v_config->'default_xxx_value_schema')::json)`）；标量字段（boolean/integer）用 `->>` + `::type`；jsonb 字段（如 description）用 `->'key'` 直接取
- 示例：
  ```sql
  -- 应用配置默认值到 NEW（入参整理，依赖资源查询结果）
  NEW.prewarm_on_start  := COALESCE(NEW.prewarm_on_start,  (v_config->>'default_prewarm_on_start')::boolean);
  NEW.warm_interval_min := COALESCE(NEW.warm_interval_min, (v_config->>'default_warm_interval_min')::integer);
  NEW.sort              := COALESCE(NEW.sort, 0);
  -- jsonb 字段（视图列也是 jsonb）
  NEW.description       := COALESCE(NEW.description, v_config->'default_xxx_description');
  -- json 字段（视图列是 json，config 推荐值是 jsonb，需 ::json cast）
  NEW.value_schema      := COALESCE(NEW.value_schema, (v_config->'default_xxx_value_schema')::json);
  ```

**⚠ 资源查询集中原则（强约束）**：分区二的所有资源查询（category_id / schema_value / config / 计数 / 唯一性等）**必须集中在统一加载层**（如 2.3 按需统一加载），下游校验（2.4+）**纯变量判断，禁止再查表**（`NOT EXISTS SELECT` / `SELECT ... INTO` 均禁止）。违反会导致同一资源多次查询（性能）+ 校验逻辑分散（难维护）。模式：2.3 一次性加载所有变量（纯加载不抛异常）→ 2.4+ 用 `v_xxx IS NULL` / `v_xxx <> 1` 等纯变量判断。参考 `dict_custom_category_view_trigger.sql` 2.3 统一加载 + 2.4-2.9 纯变量。

**复杂场景参考**：多层级分类树（CTE 一次性 path 解析）、schema 变更向后兼容（`json_matches_schema` 校验已有数据）、行级 schema 行结构/title 前缀校验等复杂模式，参考 `dict_custom_category_view_trigger.sql` 作为进阶案例。

**模式 4 — 分区三 NEW.* 纯净（业务逻辑）**：
- 分区三的 INSERT/UPDATE 语句的 VALUES/SET 子句**必须**直接使用 `NEW.xxx`
- **禁止**在分区三内嵌 COALESCE 默认值（默认值已在分区二应用到 NEW）
- **禁止**在分区三内重新加载配置（资源查询集中在分区二）
- 例外：DELETE 恢复（用 OLD.default_value 还原）允许使用 COALESCE，因为这是数据还原而非默认值应用
- 示例（合规）：
  ```sql
  INSERT INTO common.cache_target (
      object_type, schema_name, object_name,
      prewarm_on_start, warm_interval_min, max_size_mb,
      sort, sys_status, description
  ) VALUES (
      NEW.object_type, NEW.schema_name, NEW.object_name,
      NEW.prewarm_on_start, NEW.warm_interval_min, NEW.max_size_mb,
      NEW.sort, NEW.sys_status, NEW.description
  )
  RETURNING id INTO NEW.id;
  ```

**模式 5 — mview 刷新原则（分区三刷新范围）**：
- 分区三 DML 后刷新 mview，但**刷新范围必须按 mview 数据范围匹配**，禁止无脑刷新所有 mview：
  - **数据 trigger（操作 sys_data）**：只刷新**含该数据**的 mview。核查：trigger 操作的 sys_data（label + category）是否在 mview WHERE 范围内。例：module 数据（label=M00-M07）只在 `cgservice_module_mview`（不在 `cgservice_mview` 的 `label='project'` / `exception_mview` 的 source·common_code·custom_code / `custom_mview` 的 rel>sys>custom）
  - **分类 trigger（操作 sys_category）**：**保守刷新**所有相关 mview（即使当前 UPDATE sort/description 不进 mview，保留防御未来 schema 解禁等变化）
- **核查基准**：mview `details = to_jsonb(sys_data)`，**不含 sys_category 的 sort/description**。判断数据 trigger 是否需刷新某 mview，看操作数据是否在 mview WHERE 范围
- **反模式**：module trigger 刷新 cgservice_mview + exception_mview（module 数据不在）、dict_config_config 刷新 exception + custom（config project 不在），均属冗余

### Deploy Template (7-Partition)

```sql
-- ================================================================
-- URI: sqitch/deploy/{plan_name}.sql
-- ================================================================
--
-- Description:   {函数功能说明}
-- Sqitch Plan:   {plan_name}
-- Target Schema: {schema}
-- Related Story: Story X.X — {story 描述}
--
-- ================================================================

BEGIN;

CREATE OR REPLACE FUNCTION {schema}.{func_name}(
    -- IN/OUT 参数
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_location CONSTANT TEXT := '{plan_name}';
    v_log_info          JSONB;
    v_message_text      TEXT;
    v_sqlstate          TEXT;
    v_exception_detail  TEXT;
    v_exception_hint    TEXT;
    v_exception_context TEXT;
    v_schema_name       TEXT;
    v_table_name        TEXT;
    v_column_name       TEXT;
    v_constraint_name   TEXT;
    v_datatype_name     TEXT;
    r_err               RECORD;
BEGIN
    -- ====================================================================
    -- 一、入参校验
    -- ====================================================================

    -- ====================================================================
    -- 二、资源查询校验与整理
    -- ====================================================================

    -- ====================================================================
    -- 三、业务逻辑处理
    -- ====================================================================

    -- --- 3.1 子逻辑名称 ------------------------------------------------

    -- ====================================================================
    -- 四、返回值处理
    -- ====================================================================

    -- ====================================================================
    -- 五、特殊内部函数
    -- ====================================================================
    -- 无

    EXCEPTION WHEN OTHERS THEN
        GET STACKED DIAGNOSTICS
            v_message_text      = MESSAGE_TEXT,
            v_sqlstate          = RETURNED_SQLSTATE,
            v_exception_detail  = PG_EXCEPTION_DETAIL,
            v_exception_hint    = PG_EXCEPTION_HINT,
            v_exception_context = PG_EXCEPTION_CONTEXT,
            v_schema_name       = SCHEMA_NAME,
            v_table_name        = TABLE_NAME,
            v_column_name       = COLUMN_NAME,
            v_constraint_name   = CONSTRAINT_NAME,
            v_datatype_name     = PG_DATATYPE_NAME;
        SELECT * INTO r_err FROM log.log_err(
            v_message_text, v_sqlstate, v_exception_detail,
            v_exception_hint, v_exception_context,
            v_schema_name, v_table_name, v_column_name,
            v_constraint_name, v_datatype_name,
            v_location
        );
        RAISE EXCEPTION USING
            MESSAGE = r_err.o_pg_message_text || '(' || v_location || ':' || r_err.o_log_id || ')',
            ERRCODE = r_err.o_pg_sqlstate;
END;
$$;

-- ====================================================================
-- 六、注释
-- ====================================================================

COMMENT ON FUNCTION {schema}.{func_name}({param_types}) IS
    E'{函数功能说明}\n\n'
    '参数&返回值:'
    '  IN  p_{param}(TYPE): 参数说明'
    '  OUT o_{param}(TYPE): 返回值说明'
    '\n\n'
    '异常处理:'
    '  统一异常处理说明'
    '\n\n'
    '设计说明:'
    '  设计要点说明';

-- ====================================================================
-- 七、附加操作
-- ====================================================================
-- 或 -- 无附加操作（{reason}）

COMMIT;
```

### Verify Template

```sql
-- ================================================================
-- URI: sqitch/verify/{plan_name}.sql
-- ================================================================
--
-- Description:   验证 {函数功能说明}
-- Sqitch Plan:   {plan_name}
-- Target Schema: {schema}
-- Related Story: Story X.X — {story 描述}
--
-- ================================================================

BEGIN;

SELECT no_plan();
SELECT has_function('{schema}', '{func_name}', ARRAY['type1', 'type2'], '{func_name} 应存在');
SELECT * FROM finish();

ROLLBACK;
```

### Revert Template

```sql
-- ================================================================
-- URI: sqitch/revert/{plan_name}.sql
-- ================================================================
--
-- Description:   撤销 {函数功能说明}
-- Sqitch Plan:   {plan_name}
-- Target Schema: {schema}
-- Related Story: Story X.X — {story 描述}
--
-- ================================================================

BEGIN;

DROP FUNCTION IF EXISTS {schema}.{func_name}({param_types}) CASCADE;

COMMIT;
```

---

## Object Type 5.5: Cronjob Function

### Design Rules

- **基于普通函数模板**：保持 7 分区结构、完整日志记录和异常规范
- **与普通函数的 4 处差异（T15 新模板，经 `common.async_cronjob_helper` 包装执行）**：
  1. 命名：固定后缀 `_cronjob`
  2. 签名：固定 `(p_params JSONB DEFAULT '{}', OUT o_result JSONB)`
  3. 返回值：**手工封装** `o_result := jsonb_build_object('code',0,'msg','ok','data',o_result)`；业务错误路径手工 `o_result := jsonb_build_object('code',<5位码>,'msg',...,'log_id',...)`——**不调 `log.log_job()`**（execution 级记录归 helper 写）
  4. 异常处理：`EXCEPTION WHEN query_canceled THEN RAISE`（57014 重抛给 helper 消歧 60161/80161）+ `WHEN OTHERS` 调 `log.log_err()` 收尾（抗回滚）+ 吞 + code≠0（普通函数 log_err 后 RAISE 重抛；cronjob 吞）
- **Spec 参考**：`_bmad-output/specs/spec-async.md` § Cronjob Function Template + § Cronjob Helper & Termination
- 空分区保留分区头 + 注释原因：`-- 无（{reason}）`
- 子分区使用阿拉伯数字编号：2.1, 2.2, 3.1, 3.2 ...
- **thin 编排（cronjob 只编排，不内联业务）**：cronjob 函数仅负责：
  - 分区一：trace_id + user_name（双主体）
  - 分区二/三：调 daemon/helper（daemon valqry 扫描 + config STRICT+_chain / daemon optfmt rows JSONB + 对象级 try + 汇总 log）
  - 分区四：o_result 合并（多 phase 累计）
  - 分区六：模板六 EXCEPTION
  - **禁止内联**：config 读 + 汇总 log + _py 外部调用 + DML（这些归 daemon/helper）
- **daemon helper 调用模式（分区二/三）**：
  - **daemon valqry**：`_{module}_daemon_{business}_valqry(...)` — 扫描候选行 + config STRICT+_chain（80139）+ 返回 rows JSONB
  - **daemon optfmt**：`_{module}_daemon_{business}_optfmt(p_rows, p_endpoint, ...)` — 循环 rows + config 显式标量 + 对象级 try + 汇总 log
- **两级异常（核心抗失败设计）**：
  - **对象级**（daemon optfmt 内 try）：单个对象 `_py` 失败吞 + 记 errors + log + 继续下个对象（一个对象失败不影响整批）
  - **业务线级**（cronjob 每 phase try）：`WHEN query_canceled RAISE`（中止必须传播）+ `WHEN OTHERS GET STACKED DIAGNOSTICS` + `log_err` + 记 log_id + 继续下业务线（一个 phase 失败不影响其他 phase）
- **trace_id + user_name（分区一）**：
  - `trace_id`：`gen_ulid()` 生成（cronjob 异步上下文无 GUC trace_id）
  - `user_name`：`TASK(TASK)` 固定枚举（异步任务身份）；或按优先级 `Operator(固定) > Delegator(原 GUC)`
- **模板六 EXCEPTION（cronjob 标准异常块）**：
  - `WHEN query_canceled THEN RAISE` — 57014 透传给 `async_cronjob_helper` 消歧 60161（用户中止）/ 80161（系统 timeout）
  - `WHEN OTHERS THEN` — `GET STACKED DIAGNOSTICS`（5 字段）+ `log.log_err` 收尾（抗回滚）+ `o_result := {code, msg, log_id}` + `RETURN` 吞

### Developer Contract`开发约定`

- **一~三**：开发人员负责业务逻辑，将业务数据放入 `o_result`（原始指标 JSONB）
- **四**：手工封装 `o_result := jsonb_build_object('code', 0, 'msg', 'ok', 'data', o_result);`（成功路径；**不调 log_job**——execution 记录归 helper）
- **五**：开发人员负责
- **六**：框架自动（`WHEN query_canceled RAISE` + `WHEN OTHERS` → `log.log_err` 收尾 + 吞 + code≠0），开发人员不修改

### Deploy Template (7-Partition)

```sql
-- ================================================================
-- URI: sqitch/deploy/{module}_{child}_{desc}_cronjob_func.sql
-- ================================================================
--
-- Description:   {任务功能说明}
-- Sqitch Plan:   {module}_{child}_{desc}_cronjob_func
-- Target Schema: {module}
-- Related Story: Story X.X — {story 描述}
-- Function Type: cronjob（普通函数变体，经 async_cronjob_helper 包装执行）
--
-- ================================================================

BEGIN;

CREATE OR REPLACE FUNCTION {module}.{child}_{desc}_cronjob(
    IN  p_params JSONB DEFAULT '{}',
    OUT o_result JSONB
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_location          CONSTANT TEXT := '{module}.{child}_{desc}_cronjob';
    v_message_text      TEXT;
    v_sqlstate          TEXT;
    v_exception_detail  TEXT;
    v_exception_hint    TEXT;
    v_exception_context TEXT;
    v_log_id            TEXT;   -- log.log_err 返回的错误日志 ID
BEGIN
    -- ====================================================================
    -- 一、入参校验
    -- ====================================================================

    -- ====================================================================
    -- 二、资源查询校验与整理
    -- ====================================================================

    -- ====================================================================
    -- 三、业务逻辑处理（开发人员将业务指标放入 o_result）
    -- ====================================================================

    -- --- 3.1 子逻辑名称 ------------------------------------------------

    -- ====================================================================
    -- 四、返回值（成功路径，手工封装；execution 记录归 helper，函数不调 log_job）
    -- ====================================================================
    o_result := jsonb_build_object('code', 0, 'msg', 'ok', 'data', o_result);

    -- ====================================================================
    -- 五、特殊内部函数
    -- ====================================================================
    -- 无

    -- ====================================================================
    -- 六、统一异常兜底（cronjob 模板）
    -- ====================================================================
    EXCEPTION
        WHEN query_canceled THEN            -- 57014（statement_timeout 限时 / pg_cancel 命令中止）
            RAISE;                           -- 重抛，async_cronjob_helper 消歧 60161/80161
        WHEN OTHERS THEN                     -- 业务错误：log.log_err 收尾（抗回滚）+ 吞 + code≠0
            GET STACKED DIAGNOSTICS
                v_message_text      = MESSAGE_TEXT,
                v_sqlstate          = RETURNED_SQLSTATE,
                v_exception_detail  = PG_EXCEPTION_DETAIL,
                v_exception_hint    = PG_EXCEPTION_HINT,
                v_exception_context = PG_EXCEPTION_CONTEXT;
            SELECT o_log_id INTO v_log_id FROM log.log_err(
                v_message_text, v_sqlstate, v_exception_detail, v_exception_hint, v_exception_context,
                NULL, NULL, NULL, NULL, NULL,
                p_location => v_location
            );
            o_result := jsonb_build_object(
                'code', COALESCE(v_sqlstate, '80000'),
                'msg', v_message_text,
                'log_id', v_log_id
            );
            RETURN;                          -- 吞，不抛；helper 读 code≠0 → execution failed
END;
$$;

-- ====================================================================
-- 七、注释
-- ====================================================================

COMMENT ON FUNCTION {module}.{child}_{desc}_cronjob(JSONB) IS
    E'{任务功能说明}\n\n'
    '参数&返回值:'
    '  IN  p_params(JSONB): 任务参数（来自 async_cronjob.func_params）'
    '  OUT o_result(JSONB): {code:0,msg:''ok'',data:{...}} 成功 / {code:<5位码>,msg,log_id} 业务失败'
    '\n\n'
    '设计说明:'
    '  cronjob 函数，由 common.async_cronjob_helper 包装执行（经 pg_cron 调度）'
    '  业务错误吞（log_err 收尾 + code≠0）；query_canceled (57014) 重抛给 helper 消歧 60161/80161'
    '  execution 级日志归 helper 抗回滚 dblink 写；本函数不调 log.log_job';

COMMIT;
```

### Verify Template

```sql
-- ================================================================
-- URI: sqitch/verify/{module}_{child}_{desc}_cronjob_func.sql
-- ================================================================
--
-- Description:   验证 {任务功能说明}
-- Sqitch Plan:   {module}_{child}_{desc}_cronjob_func
-- Target Schema: {module}
-- Related Story: Story X.X — {story 描述}
--
-- ================================================================

BEGIN;

SELECT no_plan();
SELECT has_function('{module}', '{child}_{desc}_cronjob', ARRAY['jsonb'], '{child}_{desc}_cronjob 函数应存在');
SELECT * FROM finish();

ROLLBACK;
```

### Revert Template

```sql
-- ================================================================
-- URI: sqitch/revert/{module}_{child}_{desc}_cronjob_func.sql
-- ================================================================
--
-- Description:   撤销 {任务功能说明}
-- Sqitch Plan:   {module}_{child}_{desc}_cronjob_func
-- Target Schema: {module}
-- Related Story: Story X.X — {story 描述}
--
-- ================================================================

BEGIN;

DROP FUNCTION IF EXISTS {module}.{child}_{desc}_cronjob(JSONB) CASCADE;

COMMIT;
```

---

## Object Type 5.6: Helper（多函数聚集 sqitch change）

### 适用场景

一个 sqitch change 聚集多个相关函数（典型 10~50+），如 `minio_helper_func` 含调度入口 + val/qry/optfmt 业务函数 + daemon 守护函数 + _py 辅助。不同于单函数模板（function / cronjob_function），helper 是**函数集合**。

### Design Rules

- **调度函数（公共入口）**：`{schema}.{module}_helper(p_model, p_para)` 用 `CASE p_model` 分发到内部业务函数，透传显式标量参数（`v_new ->> 'field'`）
- **业务函数三步式**：每个 `{model}_{business}` 拆为 `_val` / `_qry` / `_optfmt` 三函数（或合并为 `_valqryoptfmt`）
  - **val（校验）**：三分式 `>>>` 分隔的 `v_violation` + `v_log_info` JSONB + RAISE jsonb
  - **qry（查询整理）**：分区二 `2.1 config STRICT+_chain` / `2.3 资源查一次` / `2.9 整理 COALESCE`
  - **optfmt（操作+格式化）**：执行操作 + log + 返回 data
- **daemon 两级异常**：对象级 try 在 optfmt（_py 失败吞 + 继续）+ 业务线级 try 在 cronjob（每 phase 失败继续）
- **_py 辅助（可选）**：boto3 / 外部调用，`LANGUAGE plpython3u SECURITY DEFINER`，纯调用，异常透传
- **COMMENT 每函数后紧跟**：不集中末尾，每个 `CREATE OR REPLACE FUNCTION` 后立即写 `COMMENT ON FUNCTION`
- **显式标量参数（spec §480）**：内部业务函数参数显式标量，不用打包 JSONB 隐式取字段

### Deploy Template（四段聚集）

```sql
-- ================================================================
-- URI: sqitch/deploy/{module}_helper_func.sql
-- ================================================================
--
-- Description:   {module} helper 函数集合（调度 + val/qry/optfmt + daemon + _py）
-- Sqitch Plan:   {module}_helper_func
-- Target Schema: {schema}
-- Related Story: Story X.X — {story 描述}
-- Function Type: helper（多函数聚集 sqitch change）
--
-- ================================================================

BEGIN;

-- ====================================================================
-- 一、调度函数（公共入口）
-- ====================================================================

CREATE OR REPLACE FUNCTION {schema}.{module}_helper(
    IN p_model TEXT,
    IN p_para  JSONB DEFAULT '{}'
) RETURNS JSONB
LANGUAGE plpgsql
AS $$
DECLARE
    v_result JSONB;
BEGIN
    -- CASE p_model 透传显式标量参数到内部业务函数（v_new ->> 'field'）
    v_result := CASE p_model
        WHEN '{model}_{business}_{opt}' THEN
            {schema}._{module}_helper_{model}_{business}_{opt}(
                (v_new ->> 'field1')::{type1},
                (v_new ->> 'field2')::{type2}
            )
        -- ... 其他 model 分支
        ELSE NULL
    END;
    RETURN v_result;
END;
$$;

COMMENT ON FUNCTION {schema}.{module}_helper(TEXT, JSONB) IS
    E'{module} helper 调度入口\n\n'
    '参数&返回值:'
    '  IN  p_model(TEXT): 业务模型路由键（{model}_{business}_{opt}）'
    '  IN  p_para(JSONB): 业务参数（透传到内部函数）'
    '  返回值: JSONB（内部函数返回）';

-- ====================================================================
-- 二、业务函数（val / qry / optfmt 三步，或 valqryoptfmt 合并）
-- ====================================================================

-- --- 2.1 _val（校验：三分式 v_violation + v_log_info + RAISE jsonb）-----
CREATE OR REPLACE FUNCTION {schema}._{module}_helper_{model}_{business}_val(
    p_field1 {type1},
    p_field2 {type2}
) RETURNS JSONB
LANGUAGE plpgsql
AS $$
DECLARE
    v_location  CONSTANT TEXT := '{module}_helper_{model}_{business}_val';
    v_violation TEXT;
    v_log_info  JSONB;
BEGIN
    -- CASE 链三分式（>>> 分隔 field/category/reason）
    v_violation := CASE
        WHEN p_field1 IS NULL THEN 'field1>>必填字段>>不能为 NULL'
        WHEN p_field2 IS NULL THEN 'field2>>必填字段>>不能为 NULL'
        ELSE NULL
    END;
    IF v_violation IS NOT NULL THEN
        v_log_info := jsonb_build_object(
            'op',       '{business}',
            'field',    split_part(v_violation, '>>', 1),
            'category', split_part(v_violation, '>>', 2),
            'reason',   split_part(v_violation, '>>', 3)
        );
        RAISE EXCEPTION USING
            MESSAGE = (jsonb_build_object('_location', v_location) || v_log_info)::text,
            ERRCODE = '6{schema}29';
    END IF;
    RETURN jsonb_build_object('ok', true);
END;
$$;

COMMENT ON FUNCTION {schema}._{module}_helper_{model}_{business}_val({type1}, {type2}) IS
    'val 校验（三分式 v_violation + RAISE jsonb）';

-- --- 2.2 _qry（查询整理：config STRICT+_chain / 资源查一次 / COALESCE）---
CREATE OR REPLACE FUNCTION {schema}._{module}_helper_{model}_{business}_qry(
    p_field1 {type1}
) RETURNS JSONB
LANGUAGE plpgsql
AS $$
DECLARE
    v_location CONSTANT TEXT := '{module}_helper_{model}_{business}_qry';
    v_config   JSONB;
    -- 10 个 GET STACKED DIAGNOSTICS 变量（_chain 用）
    v_message_text TEXT; v_sqlstate TEXT; v_exception_detail TEXT;
    v_exception_hint TEXT; v_exception_context TEXT;
    v_schema_name TEXT; v_table_name TEXT; v_column_name TEXT;
    v_constraint_name TEXT; v_datatype_name TEXT;
    v_resource {resource_type};  -- 2.3 资源查一次
BEGIN
    -- 2.1 config STRICT + _chain（80139）
    BEGIN
        SELECT value INTO STRICT v_config
        FROM dict.{module}_mview
        WHERE category = 'common>{module}>config' AND label = 'project' AND active = 1;
    EXCEPTION WHEN OTHERS THEN
        GET STACKED DIAGNOSTICS
            v_message_text = MESSAGE_TEXT, v_sqlstate = RETURNED_SQLSTATE,
            v_exception_detail = PG_EXCEPTION_DETAIL, v_exception_hint = PG_EXCEPTION_HINT,
            v_exception_context = PG_EXCEPTION_CONTEXT, v_schema_name = SCHEMA_NAME,
            v_table_name = TABLE_NAME, v_column_name = COLUMN_NAME,
            v_constraint_name = CONSTRAINT_NAME, v_datatype_name = PG_DATATYPE_NAME;
        RAISE EXCEPTION USING
            MESSAGE = (jsonb_build_object(
                '_chain', jsonb_build_object(
                    'message_text', v_message_text, 'sqlstate', v_sqlstate,
                    'exception_detail', v_exception_detail, 'exception_hint', v_exception_hint,
                    'exception_context', v_exception_context, 'schema_name', v_schema_name,
                    'table_name', v_table_name, 'column_name', v_column_name,
                    'constraint_name', v_constraint_name, 'datatype_name', v_datatype_name
                ),
                '_location', v_location
            ))::text,
            ERRCODE = '80139';
    END;

    -- 2.2 必要字段校验（STRICT 仅保证 row，不保证 JSONB key）
    -- 2.3 资源查一次（纯加载，不抛异常）
    -- 2.4+ 纯变量判断（不再查表）
    -- 2.9 整理 COALESCE（应用 config 默认值）
    RETURN jsonb_build_object('config', v_config, 'resource', v_resource);
END;
$$;

COMMENT ON FUNCTION {schema}._{module}_helper_{model}_{business}_qry({type1}) IS
    'qry 查询整理（config STRICT+_chain / 资源查一次 / 整理 COALESCE）';

-- --- 2.3 _optfmt（操作 + log + 返回 data）-------------------------------
CREATE OR REPLACE FUNCTION {schema}._{module}_helper_{model}_{business}_optfmt(
    p_field1 {type1},
    p_config JSONB
) RETURNS JSONB
LANGUAGE plpgsql
AS $$
DECLARE
    v_data JSONB;
BEGIN
    -- 业务操作（调 _py / DML）
    -- log_operation(...)
    -- 返回 data
    RETURN v_data;
END;
$$;

COMMENT ON FUNCTION {schema}._{module}_helper_{model}_{business}_optfmt({type1}, JSONB) IS
    'optfmt 操作+格式化（执行 + log + 返回 data）';

-- ====================================================================
-- 三、daemon 函数（守护进程批量，可选）
-- ====================================================================

-- --- 3.1 daemon valqry（扫描 + config STRICT+_chain）--------------------
CREATE OR REPLACE FUNCTION {schema}._{module}_daemon_{business}_valqry(
    p_endpoint TEXT
) RETURNS JSONB
LANGUAGE plpgsql
AS $$
DECLARE
    v_rows   JSONB;
    v_config JSONB;
BEGIN
    -- 扫描候选行（SELECT jsonb_agg(...) INTO v_rows）
    -- config STRICT+_chain（同 qry 2.1）
    RETURN jsonb_build_object('rows', v_rows, 'config', v_config);
END;
$$;

COMMENT ON FUNCTION {schema}._{module}_daemon_{business}_valqry(TEXT) IS
    'daemon valqry（扫描候选 + config STRICT+_chain）';

-- --- 3.2 daemon optfmt（rows JSONB + 对象级 try + 汇总 log）--------------
CREATE OR REPLACE FUNCTION {schema}._{module}_daemon_{business}_optfmt(
    p_rows     JSONB,
    p_endpoint TEXT,
    p_config   JSONB
) RETURNS JSONB
LANGUAGE plpgsql
AS $$
DECLARE
    v_row JSONB;
    v_errors JSONB := '[]'::jsonb;
    v_summary JSONB;
BEGIN
    -- 循环 rows + 对象级 try（_py 失败吞 + errors + log + 继续下个对象）
    FOR v_row IN SELECT * FROM jsonb_array_elements(p_rows) LOOP
        BEGIN
            -- 对象级操作（调 _py）
            PERFORM {schema}._{module}_{op}_py(v_row, p_endpoint);
        EXCEPTION WHEN OTHERS THEN
            -- 对象级吞：记 errors + log + 继续下个对象
            v_errors := v_errors || jsonb_build_object('row', v_row, 'err', SQLERRM);
        END;
    END LOOP;
    -- 汇总 log
    RETURN jsonb_build_object('processed', jsonb_array_length(p_rows), 'errors', v_errors);
END;
$$;

COMMENT ON FUNCTION {schema}._{module}_daemon_{business}_optfmt(JSONB, TEXT, JSONB) IS
    'daemon optfmt（rows 循环 + 对象级 try + 汇总 log）';

-- ====================================================================
-- 四、_py 辅助（boto3 / 外部，可选）
-- ====================================================================

CREATE OR REPLACE FUNCTION {schema}._{module}_{op}_py(
    p_row      JSONB,
    p_endpoint TEXT
) RETURNS JSONB
LANGUAGE plpython3u
SECURITY DEFINER
AS $$
import boto3  # inline glob bootstrap（非 sys_bootstrap_python 子上下文）
# 纯外部调用，异常透传上层（optfmt 对象级 try 捕获）
# OUT 必须 return dict（赋值方式返回 null）
return {'ok': True}
$$;

COMMENT ON FUNCTION {schema}._{module}_{op}_py(JSONB, TEXT) IS
    '_py 辅助（plpython3u + boto3 + SECURITY DEFINER，纯调用异常透传）';

COMMIT;
```

### Verify Template

```sql
-- ================================================================
-- URI: sqitch/verify/{module}_helper_func.sql
-- ================================================================
--
-- Description:   验证 {module} helper 函数集合
-- Sqitch Plan:   {module}_helper_func
-- Target Schema: {schema}
-- Related Story: Story X.X — {story 描述}
--
-- ================================================================

BEGIN;

SELECT no_plan();
-- 调度函数
SELECT has_function('{schema}', '{module}_helper', ARRAY['text', 'jsonb'], '{module}_helper 调度函数应存在');
-- 业务函数（每函数 has_function，显式参数类型数组）
SELECT has_function('{schema}', '_{module}_helper_{model}_{business}_val', ARRAY['{type1}', '{type2}'], '_val 应存在');
SELECT has_function('{schema}', '_{module}_helper_{model}_{business}_qry', ARRAY['{type1}'], '_qry 应存在');
SELECT has_function('{schema}', '_{module}_helper_{model}_{business}_optfmt', ARRAY['{type1}', 'jsonb'], '_optfmt 应存在');
-- daemon 函数（如有）
SELECT has_function('{schema}', '_{module}_daemon_{business}_valqry', ARRAY['text'], 'daemon valqry 应存在');
SELECT has_function('{schema}', '_{module}_daemon_{business}_optfmt', ARRAY['jsonb', 'text', 'jsonb'], 'daemon optfmt 应存在');
-- _py 辅助（如有）
SELECT has_function('{schema}', '_{module}_{op}_py', ARRAY['jsonb', 'text'], '_py 应存在');
SELECT * FROM finish();

ROLLBACK;
```

### Revert Template

```sql
-- ================================================================
-- URI: sqitch/revert/{module}_helper_func.sql
-- ================================================================
--
-- Description:   撤销 {module} helper 函数集合
-- Sqitch Plan:   {module}_helper_func
-- Target Schema: {schema}
-- Related Story: Story X.X — {story 描述}
--
-- ================================================================

BEGIN;

-- 每函数 DROP（注意重载用 DROP ROUTINE；推荐按依赖倒序：_py → daemon → 业务 → 调度）
DROP FUNCTION IF EXISTS {schema}._{module}_{op}_py(JSONB, TEXT) CASCADE;
DROP FUNCTION IF EXISTS {schema}._{module}_daemon_{business}_optfmt(JSONB, TEXT, JSONB) CASCADE;
DROP FUNCTION IF EXISTS {schema}._{module}_daemon_{business}_valqry(TEXT) CASCADE;
DROP FUNCTION IF EXISTS {schema}._{module}_helper_{model}_{business}_optfmt({type1}, JSONB) CASCADE;
DROP FUNCTION IF EXISTS {schema}._{module}_helper_{model}_{business}_qry({type1}) CASCADE;
DROP FUNCTION IF EXISTS {schema}._{module}_helper_{model}_{business}_val({type1}, {type2}) CASCADE;
DROP FUNCTION IF EXISTS {schema}.{module}_helper(TEXT, JSONB) CASCADE;

COMMIT;
```

---

## Object Type 6: Control Function

### Design Rules

- **PostgREST 暴露**：函数签名和返回值需兼容 PostgREST RPC 调用
- **严格输入验证**：所有入参必须校验，非法输入返回明确错误
- **结构化返回**：返回 JSONB 或复合类型，便于前端解析
- **7 分区统一结构**：一～五在函数体内，六～七在脚本层
- 空分区保留分区头 + 注释原因：`-- 无（{reason}）`
- 子分区使用阿拉伯数字编号：2.1, 2.2, 3.1, 3.2 ...

### Deploy Template (7-Partition)

```sql
-- ================================================================
-- URI: sqitch/deploy/{plan_name}.sql
-- ================================================================
--
-- Description:   {控制函数功能说明}（PostgREST RPC）
-- Sqitch Plan:   {plan_name}
-- Target Schema: {schema}
-- Related Story: Story X.X — {story 描述}
--
-- ================================================================

BEGIN;

CREATE OR REPLACE FUNCTION {schema}.{func_name}(
    -- IN 参数（PostgREST 兼容类型）
)
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
DECLARE
    v_location CONSTANT TEXT := '{plan_name}';
    v_log_info          JSONB;
    -- 异常处理变量（同 function 类型）
BEGIN
    -- ====================================================================
    -- 一、入参校验
    -- ====================================================================
    -- PostgREST 入参必须严格校验

    -- ====================================================================
    -- 二、资源查询校验与整理
    -- ====================================================================

    -- ====================================================================
    -- 三、业务逻辑处理
    -- ====================================================================

    -- --- 3.1 子逻辑名称 ------------------------------------------------

    -- ====================================================================
    -- 四、返回值处理
    -- ====================================================================

    -- ====================================================================
    -- 五、特殊内部函数
    -- ====================================================================
    -- 无

    -- 异常处理：场景 4 统一异常处理（控制函数 log_err 后返回结构化错误，不重抛）
    EXCEPTION WHEN OTHERS THEN
        GET STACKED DIAGNOSTICS ...;
        SELECT * INTO r_err FROM log.log_err(...);
        RETURN jsonb_build_object(
            'code', r_err.o_pg_sqlstate,
            'msg', r_err.o_msg_cn,
            'data', NULL
        );
END;
$$;

-- ====================================================================
-- 六、注释
-- ====================================================================

COMMENT ON FUNCTION {schema}.{func_name}({param_types}) IS
    E'{控制函数功能说明}\n\n'
    '参数&返回值:'
    '  IN  p_{param}(TYPE): 参数说明'
    '  返回值: JSONB 结构说明'
    '\n\n'
    '异常处理:'
    '  统一异常处理说明'
    '\n\n'
    '设计说明:'
    '  PostgREST RPC 调用说明';

-- ====================================================================
-- 七、附加操作
-- ====================================================================
-- PostgREST 权限配置（GRANT EXECUTE 等）
-- 或 -- 无附加操作（{reason}）

COMMIT;
```

### Verify / Revert Templates

与 Function 类型相同（verify + revert），不再重复。

---

## Object Type 7: MView Refresh Function

### Design Rules

- **CONCURRENTLY 优先**：默认走 `REFRESH MATERIALIZED VIEW CONCURRENTLY`（不阻塞读），失败回退到非 CONCURRENTLY
- **永不传播异常（关键）**：mview_refresh 被 trigger 调用（category_view_trigger 等），异常传播会破坏用户 DML；外层必须 `EXCEPTION WHEN OTHERS THEN RAISE LOG` 兜底
- **RAISE LOG 兜底**：刷新失败 = 物化视图损坏 / schema 错配 / 系统级问题（锁/OOM/磁盘），属于运维事件；直接走 PG Server Log，不尝试 log.log_system（避免同 DB 同病 + 嵌套 EXCEPTION 噪音）
- **JSON 结构化**：RAISE LOG 输出 `event / _location / mview / sqlstate / err` 字段，便于 ELK/Loki 机器解析与告警

### Deploy Template (3-Partition)

<!-- Skill Note: Known Patterns 仅作为 skill 生成指导，不出现在实际脚本中 -->

```sql
-- ================================================================
-- URI: sqitch/deploy/{plan_name}.sql
-- ================================================================
--
-- Description:   物化视图刷新函数 {schema}.{func_name}()
-- Sqitch Plan:   {plan_name}
-- Target Schema: {schema}
-- Related Story: Story X.X — {story 描述}
--
-- ================================================================

BEGIN;

-- ====================================================================
-- 一、函数定义
-- ====================================================================
CREATE OR REPLACE FUNCTION {schema}.{func_name}()
RETURNS void
LANGUAGE plpgsql
AS $$
BEGIN
    BEGIN
        -- 优先 CONCURRENTLY（不阻塞读），事务块中或无数据时回退非 CONCURRENTLY
        BEGIN
            REFRESH MATERIALIZED VIEW CONCURRENTLY {schema}.{mview_name};
        EXCEPTION WHEN feature_not_supported OR object_not_in_prerequisite_state THEN
            REFRESH MATERIALIZED VIEW {schema}.{mview_name};
        END;
    EXCEPTION WHEN OTHERS THEN
        -- 兜底层：刷新失败写 PG Server Log，不传播到 trigger 调用方（避免破坏用户 DML）
        -- mview 刷新失败属于系统底层问题（损坏/schema 错配/锁/OOM/磁盘），由运维处理
        RAISE LOG USING MESSAGE = jsonb_build_object(
            'event',     'mview_refresh_failed',
            '_location', '{plan_name}',
            'mview',     '{schema}.{mview_name}',
            'sqlstate',  SQLSTATE,
            'err',       SQLERRM
        )::text;
    END;
END;
$$;

-- ====================================================================
-- 二、注释
-- ====================================================================
COMMENT ON FUNCTION {schema}.{func_name}() IS '刷新物化视图 {schema}.{mview_name}（CONCURRENTLY 优先 + RAISE LOG 兜底，不抛异常）';

-- ====================================================================
-- 三、附加操作
-- ====================================================================
-- 首次部署时刷新，确保种子数据可见
SELECT {schema}.{func_name}();

COMMIT;
```

### Verify Template

```sql
-- ================================================================
-- URI: sqitch/verify/{plan_name}.sql
-- ================================================================
--
-- Description:   验证物化视图刷新函数 {schema}.{func_name}()
-- Sqitch Plan:   {plan_name}
-- Target Schema: {schema}
-- Related Story: Story X.X — {story 描述}
--
-- ================================================================

BEGIN;

SELECT no_plan();
SELECT has_function('{schema}', '{func_name}', ARRAY[]::text[],
    '{schema}.{func_name}() 函数应存在');
SELECT * FROM finish();

ROLLBACK;
```

### Revert Template

```sql
-- ================================================================
-- URI: sqitch/revert/{plan_name}.sql
-- ================================================================
--
-- Description:   撤销物化视图刷新函数 {schema}.{func_name}()
-- Sqitch Plan:   {plan_name}
-- Target Schema: {schema}
-- Related Story: Story X.X — {story 描述}
--
-- ================================================================

BEGIN;

DROP FUNCTION IF EXISTS {schema}.{func_name}() CASCADE;

COMMIT;
```

---

## Object Type 8: Seed

### Seed Special Rules

- **Deploy**: 必须用 `DO $$ ... $$;` + DECLARE（需要 RETURNING id 传递给子分类），ON CONFLICT DO UPDATE 幂等写入
- **Verify**: 必须依靠唯一字段信息验证（分类用 code + pid，数据用 category_id + label），可用 count 计数
- **Revert**: 移除基础数据时必须依靠唯一字段信息（非 created_by 宽泛过滤）；移除层级数据时可级联

### Deploy Template (4-Partition)

```sql
-- ================================================================
-- URI: sqitch/deploy/{plan_name}.sql
-- ================================================================
--
-- Description:   {种子数据说明}
-- Sqitch Plan:   {plan_name}
-- Target Schema: {schema}
-- Related Story: Story X.X
--
-- ================================================================

BEGIN;

-- ====================================================================
-- 一、GUC准备
-- ====================================================================
SELECT set_config('bizsky._tmp_saved_user',
    COALESCE(current_setting('bizsky.user_name', true), ''), true);
SELECT set_config('bizsky.user_name', '_seed', true);

-- ====================================================================
-- 二、种子数据写入
-- ====================================================================
DO $$
DECLARE
    v_{root}_id TEXT;
    v_{child}_id TEXT;
BEGIN
    -- --- 2.1 根分类（查找或创建）---------------------------------------
    INSERT INTO dict.sys_category (pid, code, created_by)
    VALUES (NULL, '{code}', '_seed')
    ON CONFLICT (code, pid) DO UPDATE SET updated_by = EXCLUDED.created_by
    RETURNING id INTO v_{root}_id;

    -- --- 2.2 子分类树 --------------------------------------------------
    INSERT INTO dict.sys_category (pid, code, created_by)
    VALUES (v_{root}_id, '{child_code}', '_seed')
    ON CONFLICT (code, pid) DO UPDATE SET updated_by = EXCLUDED.created_by
    RETURNING id INTO v_{child}_id;

    -- --- 2.N Data 种子数据 ---------------------------------------------
    INSERT INTO dict.sys_data (category_id, label, value, created_by)
    VALUES (v_{child}_id, '{label}', '{value}'::json, '_seed')
    ON CONFLICT (category_id, label) DO UPDATE SET
        value = EXCLUDED.value,
        updated_by = EXCLUDED.created_by;
END;
$$;

-- ====================================================================
-- 三、GUC复原
-- ====================================================================
SELECT set_config('bizsky.user_name',
    current_setting('bizsky._tmp_saved_user', true), true);

-- ====================================================================
-- 四、附加操作
-- ====================================================================
-- SELECT {schema}.{mview_refresh_func}();
-- 或 -- 无附加操作（{reason}）

COMMIT;
```

### Verify Template

```sql
-- ================================================================
-- URI: sqitch/verify/{plan_name}.sql
-- ================================================================
--
-- Description:   验证 {种子数据说明}
-- Sqitch Plan:   {plan_name}
-- Target Schema: {schema}
-- Related Story: Story X.X — {story 描述}
--
-- ================================================================

BEGIN;

SELECT no_plan();

-- 分类验证（唯一字段：code + pid）
SELECT ok(EXISTS(
    SELECT 1 FROM dict.sys_category WHERE code = '{code}' AND pid IS NULL
), '{分类} 应存在');

-- 数据计数验证（唯一字段：category path + label）
SELECT is(
    (SELECT count(*) FROM dict.sys_data d
     WHERE d.category_id IN (
         SELECT sc.id FROM dict.sys_category sc
         WHERE sc.code = '{child_code}' AND sc.pid IN (...)
     ))::bigint,
    {expected_count},
    '{category} 应有 {N} 条种子数据');

SELECT * FROM finish();

ROLLBACK;
```

### Revert Template (5-Partition)

```sql
-- ================================================================
-- URI: sqitch/revert/{plan_name}.sql
-- ================================================================
--
-- Description:   撤销 {种子数据说明}
-- Sqitch Plan:   {plan_name}
-- Target Schema: {schema}
-- Related Story: Story X.X — {story 描述}
--
-- ================================================================

BEGIN;

-- ====================================================================
-- 一、GUC准备
-- ====================================================================
SET LOCAL bizsky.bypass_audit = 'true';
SELECT set_config('bizsky._tmp_saved_user',
    COALESCE(current_setting('bizsky.user_name', true), ''), true);
SELECT set_config('bizsky.user_name', '_seed', true);

-- ====================================================================
-- 二、移除基础数据（sys_data，按唯一字段定位）
-- ====================================================================
DELETE FROM dict.sys_data
WHERE label = '{label}'
  AND category_id IN (
      SELECT sc.id FROM dict.sys_category sc
      WHERE sc.code = '{child_code}' AND sc.pid IN (...)
  );

-- ====================================================================
-- 三、移除层级数据（sys_category，叶到根，可级联）
-- ====================================================================
DELETE FROM dict.sys_category WHERE code IN ('{leaf_codes}'...)
    AND pid IN (SELECT id FROM dict.sys_category WHERE code = '{parent_code}');
DELETE FROM dict.sys_category WHERE code = '{parent_code}';

-- ====================================================================
-- 四、GUC复原
-- ====================================================================
SET LOCAL bizsky.bypass_audit = 'false';
SELECT set_config('bizsky.user_name',
    current_setting('bizsky._tmp_saved_user', true), true);

-- ====================================================================
-- 五、附加操作
-- ====================================================================
-- SELECT {schema}.{mview_refresh_func}();
-- 或 -- 无附加操作（{reason}）

COMMIT;
```

---

## Object Type 9: All Analysis（全类型分析视图分支追加）

### 适用场景

新增日志类型（注册 `log_{type}` + `archive_{type}` 表）后，向 `log.log_all_analysis` 和 `log.log_all_full_analysis` 追加 UNION ALL 分支。每次新增日志类型生成 **2 个独立 plan**，每个 plan 对应一个视图。

### 设计规则

- **每个 plan 对应一个视图**：`log_log_all_analysis_add_{type}` 和 `log_log_all_full_analysis_add_{type}`
- **Deploy**：读取当前视图定义，追加新分支后 `CREATE OR REPLACE VIEW`
- **Revert**：包含**完整的前版本视图定义**（不含新分支），`CREATE OR REPLACE VIEW` 还原
- **Verify**：仅需验证视图存在且可查询（smoke test），不需要详细结构验证
- **Revert 必须完整**：revert 脚本中的视图定义必须与上一个版本的完整定义一致，不能省略任何分支

### Plan 命名

| Plan 名 | 目标视图 | 追加分支数 | 说明 |
|---------|---------|-----------|------|
| `log_log_all_analysis_add_{type}` | `log.log_all_analysis` | 1 | `log_{type}` 在线表 |
| `log_log_all_full_analysis_add_{type}` | `log.log_all_full_analysis` | 2 | `log_{type}` 在线 + `archive_{type}` 归档 |

### 依赖关系

```
log_log_{type}_table ──→ log_archive_{type}_table ──→ log_log_all_analysis_add_{type}
                                                     ──→ log_log_all_full_analysis_add_{type}
```

两个 all_analysis plan 都依赖对应的 archive_table plan。`log_log_all_full_analysis_add_{type}` 依赖 `log_log_all_analysis_add_{type}`（部署顺序保证在线表先合并）。

### 视图列结构

**log_all_analysis（10 列）**

| 序号 | 列名 | 类型 | 来源 |
|------|------|------|------|
| 1 | id | text | log_{type}.id |
| 2 | pid | text | log_{type}.pid |
| 3 | trace_id | text | log_{type}.trace_id |
| 4 | request_id | text | log_{type}.request_id |
| 5 | log_level | integer | log_{type}.log_level |
| 6 | log_type | text | log_{type}.log_type |
| 7 | created_at | timestamptz | log_{type}.created_at |
| 8 | created_by | text | log_{type}.created_by |
| 9 | extend | jsonb | log_{type}.extend |
| 10 | details | jsonb | to_jsonb(log_{type}.*) |

**log_all_full_analysis（11 列）** — 增加 `is_archived` 列（在线=0，归档=1），其余同上。

### 新增分支模板

**log_all_analysis — 追加 1 个分支：**

```sql
SELECT b.id, b.pid, b.trace_id, b.request_id, b.log_level, b.log_type,
       b.created_at, b.created_by, b.extend, to_jsonb(b.*) AS details
FROM log.log_{type} b
```

**log_all_full_analysis — 追加 2 个分支：**

```sql
SELECT 0 AS is_archived,
       b.id, b.pid, b.trace_id, b.request_id, b.log_level, b.log_type,
       b.created_at, b.created_by, b.extend, to_jsonb(b.*) AS details
FROM log.log_{type} b
UNION ALL
SELECT 1 AS is_archived,
       a.id, a.pid, a.trace_id, a.request_id, a.log_level, a.log_type,
       a.created_at, a.created_by, a.extend, to_jsonb(a.*) AS details
FROM log.archive_{type} a
```

### Plan 1: log_log_all_analysis_add_{type}

#### Deploy Template (3-Partition)

```sql
-- ================================================================
-- URI: sqitch/deploy/log_log_all_analysis_add_{type}.sql
-- ================================================================
--
-- Description:   追加日志类型 {type} 到 log_all_analysis 视图
-- Sqitch Plan:   log_log_all_analysis_add_{type}
-- Target Schema: log
-- Related Story: Story X.X — {story 描述}
--
-- ================================================================

BEGIN;

-- ====================================================================
-- 一、视图定义
-- ====================================================================

CREATE OR REPLACE VIEW log.log_all_analysis AS
-- --- 前版本分支（原样保留，不可修改）---
{previous_branches}
-- --- 新增 {type} 分支 ---
UNION ALL
SELECT b.id, b.pid, b.trace_id, b.request_id, b.log_level, b.log_type,
       b.created_at, b.created_by, b.extend, to_jsonb(b.*) AS details
FROM log.log_{type} b;

-- ====================================================================
-- 二、视图注释
-- ====================================================================

COMMENT ON VIEW log.log_all_analysis IS '全类型日志分析视图（在线表 UNION ALL）';

-- ====================================================================
-- 三、列注释
-- ====================================================================

-- 无（列注释由首次创建视图的 plan 设置，后续仅追加分支）

COMMIT;
```

#### Revert Template (3-Partition)

```sql
-- ================================================================
-- URI: sqitch/revert/log_log_all_analysis_add_{type}.sql
-- ================================================================
--
-- Description:   撤销日志类型 {type} 的 log_all_analysis 分支
-- Sqitch Plan:   log_log_all_analysis_add_{type}
-- Target Schema: log
-- Related Story: Story X.X — {story 描述}
--
-- ================================================================

BEGIN;

-- ====================================================================
-- 一、视图定义（还原为前版本完整定义）
-- ====================================================================

CREATE OR REPLACE VIEW log.log_all_analysis AS
{previous_full_definition};

-- ====================================================================
-- 二、视图注释
-- ====================================================================

COMMENT ON VIEW log.log_all_analysis IS '全类型日志分析视图（在线表 UNION ALL）';

-- ====================================================================
-- 三、列注释
-- ====================================================================

-- 无（同 deploy）

COMMIT;
```

#### Verify Template

```sql
-- ================================================================
-- URI: sqitch/verify/log_log_all_analysis_add_{type}.sql
-- ================================================================
--
-- Description:   验证 log_all_analysis 视图存在
-- Sqitch Plan:   log_log_all_analysis_add_{type}
-- Target Schema: log
-- Related Story: Story X.X — {story 描述}
--
-- ================================================================

BEGIN;

SELECT no_plan();
SELECT has_view('log', 'log_all_analysis', 'log.log_all_analysis 视图应存在');
SELECT * FROM finish();

ROLLBACK;
```

### Plan 2: log_log_all_full_analysis_add_{type}

#### Deploy Template (3-Partition)

```sql
-- ================================================================
-- URI: sqitch/deploy/log_log_all_full_analysis_add_{type}.sql
-- ================================================================
--
-- Description:   追加日志类型 {type} 到 log_all_full_analysis 视图
-- Sqitch Plan:   log_log_all_full_analysis_add_{type}
-- Target Schema: log
-- Related Story: Story X.X — {story 描述}
--
-- ================================================================

BEGIN;

-- ====================================================================
-- 一、视图定义
-- ====================================================================

CREATE OR REPLACE VIEW log.log_all_full_analysis AS
-- --- 前版本分支（原样保留，不可修改）---
{previous_branches}
-- --- 新增 {type} 在线 + 归档分支 ---
UNION ALL
SELECT 0 AS is_archived,
       b.id, b.pid, b.trace_id, b.request_id, b.log_level, b.log_type,
       b.created_at, b.created_by, b.extend, to_jsonb(b.*) AS details
FROM log.log_{type} b
UNION ALL
SELECT 1 AS is_archived,
       a.id, a.pid, a.trace_id, a.request_id, a.log_level, a.log_type,
       a.created_at, a.created_by, a.extend, to_jsonb(a.*) AS details
FROM log.archive_{type} a;

-- ====================================================================
-- 二、视图注释
-- ====================================================================

COMMENT ON VIEW log.log_all_full_analysis IS '全类型日志完整分析视图（在线 + 归档 UNION ALL）';

-- ====================================================================
-- 三、列注释
-- ====================================================================

-- 无（列注释由首次创建视图的 plan 设置，后续仅追加分支）

COMMIT;
```

#### Revert Template (3-Partition)

```sql
-- ================================================================
-- URI: sqitch/revert/log_log_all_full_analysis_add_{type}.sql
-- ================================================================
--
-- Description:   撤销日志类型 {type} 的 log_all_full_analysis 分支
-- Sqitch Plan:   log_log_all_full_analysis_add_{type}
-- Target Schema: log
-- Related Story: Story X.X — {story 描述}
--
-- ================================================================

BEGIN;

-- ====================================================================
-- 一、视图定义（还原为前版本完整定义）
-- ====================================================================

CREATE OR REPLACE VIEW log.log_all_full_analysis AS
{previous_full_definition};

-- ====================================================================
-- 二、视图注释
-- ====================================================================

COMMENT ON VIEW log.log_all_full_analysis IS '全类型日志完整分析视图（在线 + 归档 UNION ALL）';

-- ====================================================================
-- 三、列注释
-- ====================================================================

-- 无（同 deploy）

COMMIT;
```

> **关键**：revert 中的 `{previous_full_definition}` 必须是 deploy 前的**完整视图定义**（所有分支）。生成 revert 时从 `pg_get_viewdef` 获取或从上一个 deploy 脚本中复制。

#### Verify Template

```sql
-- ================================================================
-- URI: sqitch/verify/log_log_all_full_analysis_add_{type}.sql
-- ================================================================
--
-- Description:   验证 log_all_full_analysis 视图存在
-- Sqitch Plan:   log_log_all_full_analysis_add_{type}
-- Target Schema: log
-- Related Story: Story X.X — {story 描述}
--
-- ================================================================

BEGIN;

SELECT no_plan();
SELECT has_view('log', 'log_all_full_analysis', 'log.log_all_full_analysis 视图应存在');
SELECT * FROM finish();

ROLLBACK;
```

---

- **COALESCE PATCH**: All config_view_trigger UPDATE operations use `COALESCE(NEW.field, OLD.field)` for patch semantics
- **Seed override runtime**: All seed `ON CONFLICT DO UPDATE` overwrites runtime changes
- **config_mview_refresh()**: All triggers call this after mutation; seed deploys call it after data write
- **log.log_err()**: 场景 4 统一异常处理，普通函数 log_err 后必须 RAISE 重抛，控制函数可返回结构化错误。log_err 内部经 **dblink 命名常驻连接**（独立事务）写 log_system，避免调用方事务回滚撤销日志（详见 Known Pitfalls #15/#16）
- **Report views**: Two patterns — CTE aggregation or `_impl()` function passthrough
- **Analysis views**: UNION ALL merge online + archive tables, no triggers
- **All-analysis 视图更新规范**: 新增日志类型时，必须使用 `all_analysis` 类型生成 2 个 plan：`log_log_all_analysis_add_{type}` 和 `log_log_all_full_analysis_add_{type}`，分别更新对应视图。Plan 依赖对应的日志归档表 plan。Revert 必须包含完整的前版本视图定义
- **audit_trigger GUC auto-fill**: 审计触发器必须在 INSERT 时自动填充 `trace_id`/`request_id`（从 `current_setting('bizsky.trace_id', true)` / `current_setting('bizsky.request_id', true)` 读取），并使用 `EXCEPTION WHEN undefined_column` 保护不包含这些列的表
- **Seed identity GUC**: 种子脚本开头必须执行 `set_config('bizsky.user_name', '_seed', true)`，确保审计触发器通过 GUC 自动填充 `created_by = '_seed'`，而非仅依赖 INSERT 显式传值
- **Seed mview 刷新顺序**: 任何种子脚本若部署在 mview_refresh_func 之后，必须在脚本末尾调用对应的 mview 刷新函数，确保新写入的种子数据立即可见于物化视图
- **INSTEAD OF trigger 标准保护**: 所有数据管理视图 trigger 共享标准保护逻辑（id 自动生成、label 不可变、default_value 只读、计算列保护、DELETE=恢复），详见 `spec-dict.md` Management View Trigger 章节
- **INSTEAD OF trigger 三重门控**: INSERT/UPDATE 时执行 schema 存在 → `jsonschema_is_valid()` → `json_matches_schema()` 三重校验；json_schema 视图例外：仅用 `jsonschema_is_valid()` 校验 schema 本身
- **INSTEAD OF trigger 完整恢复**: DELETE 时 `default_value IS NOT NULL` → 从 default_value 恢复 value/sys_status/sort/description，`RETURN NULL`（行未删除）

## Known Pitfalls

1. **json_matches_schema NULL**: Returns NULL (not FALSE) on validation failure — use `IS NOT TRUE`
2. **pg_cron session isolation**: `ROLLBACK` doesn't undo cron.job entries — explicit cleanup needed
3. **MView refresh timing**: Deploy must execute REFRESH after MView creation; seed must refresh after data write; 任何种子脚本部署在 mview_refresh_func 之后时，必须在末尾调用刷新函数
4. **COMMENT string format**: Use `E'...\\n\\n'` (with `E` prefix) for multi-line COMMENTs, not `''` empty strings
5. **Seed created_by**: All seed data uses `created_by = '_seed'`，且脚本开头必须 `set_config('bizsky.user_name', '_seed', true)` 确保触发器 GUC 一致
6. **Revert order**: 含触发器绑定的函数 — DROP TRIGGER before DROP FUNCTION
7. **Revert seed order**: DELETE sys_data before sys_category (leaf to root)
8. **Verify scope**: Verify does existence check only; detailed validation via pgTAP tests
9. **has_function() params**: With params use `ARRAY['type1', 'type2']`; no params use `ARRAY[]::text[]`
10. **Domain types**: `col_type_is()` checks underlying base type, not DOMAIN name
11. **Archive table INCLUDING COMMENTS**: 归档表 LIKE 必须包含 `INCLUDING COMMENTS` 以继承源表列注释，然后只需添加 `archived_at` 注释
12. **mview_refresh_function 异常兜底**: mview_refresh 被 trigger 调用，**必须**外层 `EXCEPTION WHEN OTHERS THEN RAISE LOG`（JSON 结构化），不能传播异常；否则会破坏用户 DML 事务。失败原因属于运维事件，不尝试 log.log_system 兜底（同 DB 同病）
13. **MAX/MIN 不可聚合类型**: PG 聚合函数 MAX/MIN **不支持** boolean / json / jsonb / xml 等不可排序类型，会报 `function max(jsonb) does not exist`。条件聚合（`MAX(CASE WHEN ... END)`）合并多分类配置时，需先 cast 到 text 再聚合再 cast 回：`MAX(CASE WHEN ... THEN (value->'x')::text END)::jsonb`
14. **`->` 操作符优先级陷阱**: `value->'key'::text` 被解析为 `value -> ('key'::text)`（先 cast 后取值），不是预期 `((value->'key')::text)`。cast 必须加括号 `(value->'key')::text`，否则条件聚合会因类型不匹配而失败
15. **log_err 自治事务用 dblink（非 pg_background）**: log_err 被 trigger EXCEPTION 调用后 trigger 重抛，调用方事务回滚会撤销同步 INSERT log_system（实测表恒空）。log_err 通过 **dblink 命名常驻连接**（`dblink_get_connections()` + `array_position` 检查复用，避免 duplicate 42710）在独立事务执行 `INSERT ... RETURNING id` 同步取回 log_id。**不用 pg_background**：其 worker 受全局 `max_worker_processes`（默认 8）限制，并发耗尽抛 `could not register background process`，log_err 兜底 LOG_FALLBACK 丢日志（vibhorkum/pg_background 官方文档明示 autonomous logging 是 best-effort）。dblink 受 `max_connections`（100+）约束，远宽于 worker 池。log_err 兜底时 `o_log_id := 'LOG_FALLBACK'`（非 NULL 占位），保护 trigger 消息 `|| o_log_id` 不致 MESSAGE 变 NULL（PG「RAISE option cannot be null」）
16. **函数级 SET search_path（调用其他 schema 的扩展函数）**: 函数内调用其他 schema 的扩展函数（如 dblink 系列在 public），**必须**函数级 `SET search_path = pg_catalog, public, log, dict, common`（参照 common_audit_trigger 模式）。否则在 cronjob 等受限 search_path 上下文，unqualified 函数解析失败（sqlstate 42883 `function does not exist`）。log_err 曾因 cronjob 上下文 search_path 不含 public，`dblink_get_connections()` 找不到，archive cronjob 的异常日志全部兜底 LOG_FALLBACK
17. **兜底值常量化（避免魔法值）**: 函数内的默认值/兜底值（如 err_level=40、enabled=true、sqlstate='80000'、http_status=200）用 DECLARE `CONSTANT` 集中声明（`c_default_*` / `c_fallback_*`），不在赋值/COALESCE 中写魔法值。多个兜底值有联动时用 `jsonb_build_object` 引用常量（如 c_fallback JSONB 引用 c_fallback_sqlstate/c_default_http_status），保证单一来源
