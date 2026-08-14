---
name: sql-sqitch
description: 使用 Sqitch 为 PostgreSQL 18 编制数据库迁移脚本（deploy/revert/verify）的规范与流程。在新增或修改数据库对象、编写迁移脚本、维护 sqitch.plan、执行 deploy/revert/verify 时使用。支持 table / archive_table / view / base_function / function / helper / cronjob_function / control_function / mview_refresh_function / seed / all_analysis 共 11 种对象类型模板。测试脚本编制请用 sql-pgtap skill。
license: PostgreSQL License
compatibility: PostgreSQL 18；Sqitch 1.x；psql；git
---

# SQL Sqitch 脚本编制（deploy / revert / verify）

为 PostgreSQL 18 数据库对象生成 Sqitch 迁移脚本，覆盖 11 种对象类型，每个 change 输出 deploy / revert / verify 三个 SQL 文件。

## 约定

- 注释使用中文，标识符使用英文
- 文件路径相对 `{project-root}` 解析
- 与 `sql-pgtap` skill 配套：sqitch 负责迁移脚本（deploy/revert/verify），sql-pgtap 负责测试校验
- 每个 sqitch plan 对应一个数据库对象，不做中间迭代（helper / all_analysis 等多对象聚集类型除外）

## Skill 职责

- 类型：代码生成器
- 触发：用户要求「生成 sqitch 脚本」或「编写 deploy/revert/verify」时
- 输入：对象类型 + plan 名 + schema + 对象详情
- 输出：deploy / verify / revert 三个 SQL 文件
- 配套：`sql-pgtap` — 生成对应的 pgTAP 测试文件

| 类型 | 模板分区 | 说明 |
|------|---------|------|
| table | 10 分区 | 业务数据表（8 字段）、日志表（9 字段）、关联表（6 字段） |
| archive_table | 2 分区 | 归档表，`LIKE 源表` + `INCLUDING COMMENTS` + archived_at |
| view | 6 分区 | 视图（物化视图 / 管理视图 / 报告视图 / 分析视图 / 控制视图，统一 6 分区结构） |
| base_function | 7 分区 | 基础函数：工具函数，性能优先，不强制日志记录和异常处理 |
| function | 7 分区 | 普通函数 / INSTEAD OF trigger 函数：完整防御链（校验 → 查询 → 业务 → 异常 → 日志） |
| helper | 多函数聚集 | 多函数聚集 change（调度函数 + val/qry/optfmt 业务函数 + 可选 daemon / 外部调用辅助） |
| cronjob_function | 7 分区 | 定时任务函数：固定签名 + 手工 `{code,msg,data}` 封装 + 异常吞 |
| control_function | 7 分区 | 控制函数：`web_*` schema，通过 PostgREST 向 Web 用户提供服务 |
| mview_refresh_function | 3 分区 | 物化视图刷新函数（函数定义 / 注释 / 附加操作） |
| seed | deploy 4 分区 / revert 4 分区 | 种子数据（会话准备 / 写入 / 复原 / 附加） |
| all_analysis | deploy 3 分区 / revert 3 分区 | 全类型分析视图分支追加（每次新增日志类型生成 2 个 plan） |

> INSTEAD OF trigger 函数是 `function` 的 variant，共享同一 7 分区模板，差异仅在入参来源与附加操作（见 Function 类型）。`helper` 是多函数聚集类型（一个 change 含多个函数），结构不同于单函数模板。

## 激活时（On Activation）

向用户确认：

1. **对象类型**：table / archive_table / view / base_function / function / helper / cronjob_function / control_function / mview_refresh_function / seed / all_analysis
2. **脚本类型**：deploy / verify / revert（或 `all` 生成三个）
3. **Sqitch plan 名**：如 `add_users_table`、`alter_orders_add_status`
4. **目标 schema**：如 `app`、`audit`、`report`
5. **对象详情**（随类型变化，见各模板）

信息不足时按上下文推断或最小化询问。

## 输出

写入文件：

- Deploy：`sqitch/deploy/{plan_name}.sql`
- Verify：`sqitch/verify/{plan_name}.sql`
- Revert：`sqitch/revert/{plan_name}.sql`

## 通用规则

1. **注释语言**：所有注释使用中文
2. **Deploy 提交**：始终 `COMMIT` — deploy 必须持久化
3. **Verify 回滚**：始终 `ROLLBACK` — verify 不得持久化
4. **Revert 提交**：始终 `COMMIT` — revert 必须持久化删除
5. **Seed revert 提交**：始终 `COMMIT` — seed revert 必须持久化删除
6. **Verify 范围**：verify 仅验证对象存在性，详细验证由 pgTAP 测试脚本（sql-pgtap skill）完成
7. **Verify 框架**：所有 verify 脚本使用 pgTAP 框架（`no_plan()` + `finish()`）
8. **幂等性**：deploy 用 `IF NOT EXISTS` / `OR REPLACE` / `ON CONFLICT DO UPDATE`，revert 用 `IF EXISTS`，避免重复执行报错
9. **显式 schema**：脚本内显式写 schema 名，不依赖默认 `search_path`
10. **一个 change 一件事**：每个 plan 只做一个原子变更

## 头格式（Header Format）

所有 deploy/revert/verify 脚本使用统一标准头：

```sql
-- ================================================================
-- URI: sqitch/{deploy|revert|verify}/{plan_name}.sql
-- ================================================================
--
-- Description:   {描述}
-- Sqitch Plan:   {plan_name}
-- Target Schema: {schema}
--
-- ================================================================
```

---

## 对象类型 1：Table

### Table Variants

| Variant | 标准字段 | 数量 |
|---------|---------|------|
| 业务数据表 | id / pid / created_at / created_by / updated_at / updated_by / sys_status / extend | 8 |
| 日志表 | id / pid / trace_id / request_id / log_level / log_type / created_at / created_by / extend | 9 |
| 关联表 | id / {left}_id / {right}_id / created_at / created_by / extend | 6 |

**主键约定（PostgreSQL 18）**，二选一：

- 代理键 + identity 列：`id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY`
- UUID 主键：`id uuid PRIMARY KEY DEFAULT uuidv7()`（PG 18 的 `uuidv7()` 时间有序，适合分布式写入）

**审计字段**（`created_at` / `created_by` / `updated_at` / `updated_by`）：由应用层或项目提供的审计触发器维护；本 skill 不绑定特定审计实现（见分区十）。

**触发器绑定**：表可绑定项目提供的审计/变更跟踪触发器（可选）；归档表、物化视图不绑定。

### Deploy Template（10 分区）

10 分区固定顺序，不可跳过、不可合并。空分区保留分区头 + 注释原因：`-- 无{名称}（{reason}）`

```sql
-- ================================================================
-- URI: sqitch/deploy/{plan_name}.sql
-- ================================================================
--
-- Description:   {表功能说明}
-- Sqitch Plan:   {plan_name}
-- Target Schema: {schema}
--
-- ================================================================

BEGIN;

-- ====================================================================
-- 一、标准字段
-- ====================================================================

CREATE TABLE {schema}.{table} (
    id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,  -- 主键（PG 18 identity）
    -- 或 UUID 主键：id uuid PRIMARY KEY DEFAULT uuidv7(),
    pid         bigint DEFAULT NULL,        -- 父级ID
    created_at  timestamptz NOT NULL DEFAULT now(),  -- 创建时间
    created_by  text NOT NULL,              -- 创建人（应用层/审计触发器维护）
    updated_at  timestamptz,                -- 更新时间
    updated_by  text,                       -- 更新人
    sys_status  integer NOT NULL DEFAULT 1 CHECK (sys_status IN (0,1)), -- 0=停用, 1=正常
    extend      jsonb,                      -- 扩展信息

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

COMMENT ON COLUMN {schema}.{table}.id IS '主键（identity / uuidv7）';
COMMENT ON COLUMN {schema}.{table}.pid IS '父级ID';
-- ...（其他标准字段注释）

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
-- 十、触发器绑定（可选，审计/变更跟踪触发器）
-- ====================================================================

-- CREATE TRIGGER trg_{table}_audit
--     BEFORE INSERT OR UPDATE OR DELETE ON {schema}.{table}
--     FOR EACH ROW EXECUTE FUNCTION {audit_schema}.{audit_func}();
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
--
-- ================================================================

BEGIN;

DROP TABLE IF EXISTS {schema}.{table} CASCADE;

COMMIT;
```

---

## 对象类型 2：Archive Table

### 设计规则

- `LIKE` 源表并带 `INCLUDING DEFAULTS INCLUDING CONSTRAINTS INCLUDING COMMENTS`，继承结构、约束与注释
- 额外 `archived_at TIMESTAMPTZ NOT NULL DEFAULT now()`，记录归档时间
- `PRIMARY KEY (id)`（归档表不复用源表外键）

### Deploy Template（2 分区）

```sql
-- ================================================================
-- URI: sqitch/deploy/{plan_name}.sql
-- ================================================================
--
-- Description:   {源表} 归档表
-- Sqitch Plan:   {plan_name}
-- Target Schema: {schema}
--
-- ================================================================

BEGIN;

-- ====================================================================
-- 一、表结构
-- ====================================================================
CREATE TABLE {schema}.archive_{table} (
    LIKE {schema}.{table} INCLUDING DEFAULTS INCLUDING CONSTRAINTS INCLUDING COMMENTS,
    archived_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (id)
);

-- ====================================================================
-- 二、对象注释
-- ====================================================================
COMMENT ON TABLE {schema}.archive_{table} IS '{源表} 归档表';
COMMENT ON COLUMN {schema}.archive_{table}.archived_at IS '归档时间';

COMMIT;
```

### Verify Template

```sql
-- ================================================================
-- URI: sqitch/verify/{plan_name}.sql
-- ================================================================
--
-- Description:   验证 {源表} 归档表
-- Sqitch Plan:   {plan_name}
-- Target Schema: {schema}
--
-- ================================================================

BEGIN;

SELECT no_plan();
SELECT has_table('{schema}', 'archive_{table}', '{schema}.archive_{table} 归档表应存在');
SELECT * FROM finish();

ROLLBACK;
```

### Revert Template

```sql
-- ================================================================
-- URI: sqitch/revert/{plan_name}.sql
-- ================================================================
--
-- Description:   撤销 {源表} 归档表
-- Sqitch Plan:   {plan_name}
-- Target Schema: {schema}
--
-- ================================================================

BEGIN;

DROP TABLE IF EXISTS {schema}.archive_{table} CASCADE;

COMMIT;
```

---

## 对象类型 3：View

### View Variants

| 类别 | 说明 | 特征 |
|------|------|------|
| 物化视图 (mview) | 配置查询 / 统计报表 | 索引 + 定期刷新 |
| 管理视图 (view) | 基于 CRUD 操作的视图 | INSTEAD OF trigger |
| 报告视图 (report) | CTE 聚合 | 只读 |
| 分析视图 (analysis) | UNION ALL 在线 + 归档 | 只读 |
| 控制视图 (control) | 控制逻辑 | 按需定义 |

### 设计规则

- **CTE 优先**：视图定义使用 CTE（`WITH` / `WITH RECURSIVE`）实现递归和层级，不使用子函数
- 空分区规则与表相同：保留分区头 + 注释原因

### Deploy Template（6 分区）

```sql
-- ================================================================
-- URI: sqitch/deploy/{plan_name}.sql
-- ================================================================
--
-- Description:   {视图功能说明}
-- Sqitch Plan:   {plan_name}
-- Target Schema: {schema}
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
--
-- ================================================================

BEGIN;

-- MVIEW: DROP MATERIALIZED VIEW IF EXISTS {schema}.{view_name} CASCADE;
-- VIEW:  DROP VIEW IF EXISTS {schema}.{view_name} CASCADE;

COMMIT;
```

---

## 对象类型 4：Base Function

### 设计规则

- **性能优先**：避免不必要的异常捕获，核心路径无 try-catch
- **不抛异常**：错误场景静默处理或返回 NULL / 空结果
- **不严格要求日志记录**：写入失败不影响主流程
- **7 分区统一结构**：一～五在函数体内，六～七在脚本层
- 空分区保留分区头 + 注释原因：`-- 无（{reason}）`
- 子分区使用阿拉伯数字编号：2.1, 2.2, 3.1, 3.2 ...

### Deploy Template（7 分区）

```sql
-- ================================================================
-- URI: sqitch/deploy/{plan_name}.sql
-- ================================================================
--
-- Description:   {函数功能说明}
-- Sqitch Plan:   {plan_name}
-- Target Schema: {schema}
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
--
-- ================================================================

BEGIN;

DROP FUNCTION IF EXISTS {schema}.{func_name}({param_types}) CASCADE;

COMMIT;
```

---

## 对象类型 5：Function

### 设计规则

- **入参三分类**：函数入参包含三种来源，统一在分区一（入参校验）中处理
  - **显式入参**：函数签名中的 `IN/OUT` 参数
  - **隐式入参**：PostgreSQL 自动注入的触发器变量（`TG_OP`、`NEW`、`OLD`、`TG_TABLE_NAME` 等）
  - **会话上下文入参**：通过 `current_setting()` 读取的会话配置（如 `app.user_name`，按项目需要）
- **完整防御链**：入参校验 → 资源查询 → 业务逻辑 → 异常处理 → 日志记录
- **统一异常处理**：使用 `EXCEPTION WHEN OTHERS` 捕获转换异常，普通函数 RAISE 重抛，控制函数可返回结构化错误
- **结构化注释**：COMMENT ON FUNCTION 包含参数&返回值、异常处理、设计说明
- **7 分区统一结构**：一～五在函数体内，六～七在脚本层
- 空分区保留分区头 + 注释原因：`-- 无（{reason}）`
- 子分区使用阿拉伯数字编号：2.1, 2.2, 3.1, 3.2 ...
- **资源查询集中原则（强约束）**：分区二的所有资源查询必须集中在统一加载层（如 2.3 一次性加载），下游校验纯变量判断，禁止再查表（`NOT EXISTS SELECT` / `SELECT ... INTO` 均禁止）

### 异常处理规范

普通函数和 INSTEAD OF trigger 函数推荐实现以下异常处理机制：

**1. 异常准备（DECLARE 块）**：
- `v_location CONSTANT TEXT := '{plan_name}'` — 来源位置标识
- `v_log_info JSONB` — 日志信息
- PG 诊断变量：`v_message_text` / `v_sqlstate` / `v_exception_detail` / `v_exception_hint` / `v_exception_context` / `v_schema_name` / `v_table_name` / `v_column_name` / `v_constraint_name` / `v_datatype_name`

**2. 自定义异常（入参校验 / 查询校验分区使用）**：
- 先赋值 `v_log_info := jsonb_build_object('op', TG_OP, 'reason', '{描述}', ...)` 构建业务上下文
- 再抛出 `RAISE EXCEPTION USING MESSAGE = (jsonb_build_object('_location', v_location) || v_log_info)::text, ERRCODE = '{sqlstate}'`
- **禁止**使用不带 ERRCODE 的 `RAISE EXCEPTION`；**禁止**使用简单字符串 MESSAGE
- SQLSTATE 优先使用 PostgreSQL 标准编码（如 `22000` 数据异常、`23000` 完整性约束、`P0001` raise_exception），项目自定义编码按需约定

**3. 统一异常处理（函数末尾 EXCEPTION WHEN OTHERS）**：
- `GET STACKED DIAGNOSTICS` 捕获 PG 异常字段
- 调用项目提供的错误日志函数（如 `{schema}.log_error(...)`）记录日志；无日志表时直接 `RAISE`
- 普通函数 / trigger 函数：`RAISE EXCEPTION USING MESSAGE = ..., ERRCODE = ...` 重抛
- 控制函数：返回结构化错误 JSONB（不重抛）

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
- **Revert**：先 `DROP TRIGGER` 再 `DROP FUNCTION`（见 Known Pitfalls）
- **SECURITY DEFINER（推荐）**：管理视图 trigger 函数建议 `SECURITY DEFINER`，强制写入经 trigger 校验；配套 revoke 表权限 + grant 视图权限实现最小权限

### Deploy Template（7 分区）

```sql
-- ================================================================
-- URI: sqitch/deploy/{plan_name}.sql
-- ================================================================
--
-- Description:   {函数功能说明}
-- Sqitch Plan:   {plan_name}
-- Target Schema: {schema}
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
        -- 如项目提供统一错误日志函数则调用（返回 log_id），否则直接 RAISE
        RAISE EXCEPTION USING
            MESSAGE = (jsonb_build_object(
                '_location', v_location,
                'message_text', v_message_text,
                'sqlstate', v_sqlstate,
                'exception_detail', v_exception_detail,
                'exception_hint', v_exception_hint
            ) || v_log_info)::text,
            ERRCODE = v_sqlstate;
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
--
-- ================================================================

BEGIN;

DROP FUNCTION IF EXISTS {schema}.{func_name}({param_types}) CASCADE;

COMMIT;
```

> **INSTEAD OF trigger 函数的 revert 需先 DROP TRIGGER**：`DROP TRIGGER IF EXISTS trg_{view}_trigger ON {schema}.{view} CASCADE;` 再 `DROP FUNCTION`。

---

## 对象类型 5.5：Cronjob Function

### 设计规则

- **基于普通函数模板**：保持 7 分区结构、完整日志记录和异常规范
- **与普通函数的差异**：
  1. 命名：固定后缀 `_cronjob`
  2. 签名：固定 `(p_params JSONB DEFAULT '{}', OUT o_result JSONB)`
  3. 返回值：**手工封装** `o_result := jsonb_build_object('code', 0, 'msg', 'ok', 'data', o_result)`；业务错误路径手工 `o_result := jsonb_build_object('code', <码>, 'msg', ..., 'log_id', ...)`
  4. 异常处理：`EXCEPTION WHEN query_canceled THEN RAISE`（57014 重抛给调度器）+ `WHEN OTHERS` 记录错误日志 + 吞 + `code≠0`（普通函数重抛，cronjob 吞）
- **thin 编排**：cronjob 函数只编排（调度 daemon/helper），不内联业务逻辑
- **两级异常（核心抗失败设计）**：
  - **对象级**（业务函数内 try）：单个对象失败吞 + 记 errors + log + 继续下个对象
  - **业务线级**（cronjob 每 phase try）：`WHEN query_canceled RAISE` + `WHEN OTHERS` 记录 + 记 log_id + 继续下业务线
- **调度**：经 `pg_cron` 或项目提供的调度 helper 包装执行（按项目约定）
- 空分区保留分区头 + 注释原因：`-- 无（{reason}）`

### Deploy Template（7 分区）

```sql
-- ================================================================
-- URI: sqitch/deploy/{plan_name}.sql
-- ================================================================
--
-- Description:   {任务功能说明}
-- Sqitch Plan:   {plan_name}
-- Target Schema: {schema}
-- Function Type: cronjob
--
-- ================================================================

BEGIN;

CREATE OR REPLACE FUNCTION {schema}.{func_name}_cronjob(
    IN  p_params JSONB DEFAULT '{}',
    OUT o_result JSONB
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_location          CONSTANT TEXT := '{schema}.{func_name}_cronjob';
    v_message_text      TEXT;
    v_sqlstate          TEXT;
    v_exception_detail  TEXT;
    v_exception_hint    TEXT;
    v_exception_context TEXT;
    v_log_id            TEXT;   -- 错误日志 ID（如项目提供错误日志函数）
BEGIN
    -- ====================================================================
    -- 一、入参校验
    -- ====================================================================

    -- ====================================================================
    -- 二、资源查询校验与整理
    -- ====================================================================

    -- ====================================================================
    -- 三、业务逻辑处理（将业务指标放入 o_result）
    -- ====================================================================

    -- --- 3.1 子逻辑名称 ------------------------------------------------

    -- ====================================================================
    -- 四、返回值（成功路径，手工封装）
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
            RAISE;                           -- 重抛，调度器消歧「用户中止 / 系统超时」
        WHEN OTHERS THEN                     -- 业务错误：记录错误日志（抗回滚）+ 吞 + code≠0
            GET STACKED DIAGNOSTICS
                v_message_text      = MESSAGE_TEXT,
                v_sqlstate          = RETURNED_SQLSTATE,
                v_exception_detail  = PG_EXCEPTION_DETAIL,
                v_exception_hint    = PG_EXCEPTION_HINT,
                v_exception_context = PG_EXCEPTION_CONTEXT;
            -- 如项目提供错误日志函数，则记录并取回 log_id；否则 v_log_id := NULL
            o_result := jsonb_build_object(
                'code', COALESCE(v_sqlstate, 'P0001'),
                'msg', v_message_text,
                'log_id', v_log_id
            );
            RETURN;                          -- 吞，不抛；调度器读 code≠0 → 失败
END;
$$;

-- ====================================================================
-- 七、注释
-- ====================================================================

COMMENT ON FUNCTION {schema}.{func_name}_cronjob(JSONB) IS
    E'{任务功能说明}\n\n'
    '参数&返回值:'
    '  IN  p_params(JSONB): 任务参数'
    '  OUT o_result(JSONB): {code:0,msg:''ok'',data:{...}} 成功 / {code,msg,log_id} 业务失败'
    '\n\n'
    '设计说明:'
    '  cronjob 函数，由调度器（pg_cron 等）包装执行'
    '  业务错误吞（记录日志 + code≠0）；query_canceled (57014) 重抛给调度器';

COMMIT;
```

### Verify Template

```sql
-- ================================================================
-- URI: sqitch/verify/{plan_name}.sql
-- ================================================================
--
-- Description:   验证 {任务功能说明}
-- Sqitch Plan:   {plan_name}
-- Target Schema: {schema}
--
-- ================================================================

BEGIN;

SELECT no_plan();
SELECT has_function('{schema}', '{func_name}_cronjob', ARRAY['jsonb'], '{func_name}_cronjob 函数应存在');
SELECT * FROM finish();

ROLLBACK;
```

### Revert Template

```sql
-- ================================================================
-- URI: sqitch/revert/{plan_name}.sql
-- ================================================================
--
-- Description:   撤销 {任务功能说明}
-- Sqitch Plan:   {plan_name}
-- Target Schema: {schema}
--
-- ================================================================

BEGIN;

DROP FUNCTION IF EXISTS {schema}.{func_name}_cronjob(JSONB) CASCADE;

COMMIT;
```

---

## 对象类型 5.6：Helper（多函数聚集 change）

### 适用场景

一个 sqitch change 聚集多个相关函数（典型 10~50+），如 `{module}_helper_func` 含调度入口 + val/qry/optfmt 业务函数 + 可选 daemon 守护函数 + 可选外部调用辅助。不同于单函数模板（function / cronjob_function），helper 是**函数集合**。

### 设计规则

- **调度函数（公共入口）**：`{schema}.{module}_helper(p_model, p_para)` 用 `CASE p_model` 分发到内部业务函数，透传显式标量参数
- **业务函数三步式**：每个 `{model}_{business}` 拆为 `_val` / `_qry` / `_optfmt` 三函数（或合并为 `_valqryoptfmt`）
  - **val（校验）**：三分式 `>>>` 分隔的 `v_violation` + `v_log_info` JSONB + RAISE jsonb
  - **qry（查询整理）**：分区二 `2.1 config STRICT` / `2.3 资源查一次` / `2.9 整理 COALESCE`
  - **optfmt（操作+格式化）**：执行操作 + log + 返回 data
- **daemon 两级异常**：对象级 try 在 optfmt（单个对象失败吞 + 继续）+ 业务线级 try 在 cronjob（每 phase 失败继续）
- **外部调用辅助（可选）**：`LANGUAGE plpython3u SECURITY DEFINER`，纯外部调用，异常透传
- **COMMENT 每函数后紧跟**：不集中末尾，每个 `CREATE OR REPLACE FUNCTION` 后立即写 `COMMENT ON FUNCTION`
- **显式标量参数**：内部业务函数参数显式标量，不用打包 JSONB 隐式取字段
- **幂等 upsert 用 MERGE（PG 18）**：需要幂等写入的场景优先用 `MERGE ... WHEN MATCHED THEN UPDATE ... WHEN NOT MATCHED THEN INSERT`

### Deploy Template（四段聚集）

```sql
-- ================================================================
-- URI: sqitch/deploy/{module}_helper_func.sql
-- ================================================================
--
-- Description:   {module} helper 函数集合（调度 + val/qry/optfmt + daemon + 外部辅助）
-- Sqitch Plan:   {module}_helper_func
-- Target Schema: {schema}
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
    v_new   JSONB := COALESCE(p_para, '{}'::jsonb);
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
            ERRCODE = 'P0001';
    END IF;
    RETURN jsonb_build_object('ok', true);
END;
$$;

COMMENT ON FUNCTION {schema}._{module}_helper_{model}_{business}_val({type1}, {type2}) IS
    'val 校验（三分式 v_violation + RAISE jsonb）';

-- --- 2.2 _qry（查询整理：config STRICT / 资源查一次 / COALESCE）---
CREATE OR REPLACE FUNCTION {schema}._{module}_helper_{model}_{business}_qry(
    p_field1 {type1}
) RETURNS JSONB
LANGUAGE plpgsql
AS $$
DECLARE
    v_location CONSTANT TEXT := '{module}_helper_{model}_{business}_qry';
    v_config   JSONB;
    -- 诊断变量（_chain 用）
    v_message_text TEXT; v_sqlstate TEXT; v_exception_detail TEXT;
    v_exception_hint TEXT; v_exception_context TEXT;
    v_resource {resource_type};  -- 2.3 资源查一次
BEGIN
    -- 2.1 config STRICT + 异常链（资源缺失抛错）
    BEGIN
        SELECT value INTO STRICT v_config
        FROM {schema}.{config_view}
        WHERE category = '{config_key}' AND label = 'project' AND active = 1;
    EXCEPTION WHEN OTHERS THEN
        GET STACKED DIAGNOSTICS
            v_message_text = MESSAGE_TEXT, v_sqlstate = RETURNED_SQLSTATE,
            v_exception_detail = PG_EXCEPTION_DETAIL, v_exception_hint = PG_EXCEPTION_HINT,
            v_exception_context = PG_EXCEPTION_CONTEXT;
        RAISE EXCEPTION USING
            MESSAGE = (jsonb_build_object(
                '_chain', jsonb_build_object(
                    'message_text', v_message_text, 'sqlstate', v_sqlstate,
                    'exception_detail', v_exception_detail, 'exception_hint', v_exception_hint,
                    'exception_context', v_exception_context
                ),
                '_location', v_location
            ))::text,
            ERRCODE = 'P0001';
    END;

    -- 2.2 必要字段校验（STRICT 仅保证 row，不保证 JSONB key）
    -- 2.3 资源查一次（纯加载，不抛异常）
    -- 2.4+ 纯变量判断（不再查表）
    -- 2.9 整理 COALESCE（应用 config 默认值）
    RETURN jsonb_build_object('config', v_config, 'resource', v_resource);
END;
$$;

COMMENT ON FUNCTION {schema}._{module}_helper_{model}_{business}_qry({type1}) IS
    'qry 查询整理（config STRICT / 资源查一次 / 整理 COALESCE）';

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
    -- 业务操作（调外部辅助 / DML / MERGE 幂等 upsert）
    -- 记录日志（如有日志函数）
    -- 返回 data
    RETURN v_data;
END;
$$;

COMMENT ON FUNCTION {schema}._{module}_helper_{model}_{business}_optfmt({type1}, JSONB) IS
    'optfmt 操作+格式化（执行 + log + 返回 data）';

-- ====================================================================
-- 三、daemon 函数（守护进程批量，可选）
-- ====================================================================

-- --- 3.1 daemon valqry（扫描 + config STRICT）--------------------
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
    -- config STRICT + 异常链（同 qry 2.1）
    RETURN jsonb_build_object('rows', v_rows, 'config', v_config);
END;
$$;

COMMENT ON FUNCTION {schema}._{module}_daemon_{business}_valqry(TEXT) IS
    'daemon valqry（扫描候选 + config STRICT）';

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
    -- 循环 rows + 对象级 try（单个对象失败吞 + errors + log + 继续下个对象）
    FOR v_row IN SELECT * FROM jsonb_array_elements(p_rows) LOOP
        BEGIN
            -- 对象级操作（调外部辅助）
            PERFORM {schema}._{module}_{op}_ext(v_row, p_endpoint);
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
-- 四、外部调用辅助（plpython3u，可选）
-- ====================================================================

CREATE OR REPLACE FUNCTION {schema}._{module}_{op}_ext(
    p_row      JSONB,
    p_endpoint TEXT
) RETURNS JSONB
LANGUAGE plpython3u
SECURITY DEFINER
AS $$
# 纯外部调用，异常透传上层（optfmt 对象级 try 捕获）
# OUT 必须 return dict（赋值方式返回 null）
return {'ok': True}
$$;

COMMENT ON FUNCTION {schema}._{module}_{op}_ext(JSONB, TEXT) IS
    '外部调用辅助（plpython3u + SECURITY DEFINER，纯调用异常透传）';

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
-- 外部调用辅助（如有）
SELECT has_function('{schema}', '_{module}_{op}_ext', ARRAY['jsonb', 'text'], '外部调用辅助应存在');
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
--
-- ================================================================

BEGIN;

-- 每函数 DROP（注意重载用 DROP ROUTINE；推荐按依赖倒序：外部辅助 → daemon → 业务 → 调度）
DROP FUNCTION IF EXISTS {schema}._{module}_{op}_ext(JSONB, TEXT) CASCADE;
DROP FUNCTION IF EXISTS {schema}._{module}_daemon_{business}_optfmt(JSONB, TEXT, JSONB) CASCADE;
DROP FUNCTION IF EXISTS {schema}._{module}_daemon_{business}_valqry(TEXT) CASCADE;
DROP FUNCTION IF EXISTS {schema}._{module}_helper_{model}_{business}_optfmt({type1}, JSONB) CASCADE;
DROP FUNCTION IF EXISTS {schema}._{module}_helper_{model}_{business}_qry({type1}) CASCADE;
DROP FUNCTION IF EXISTS {schema}._{module}_helper_{model}_{business}_val({type1}, {type2}) CASCADE;
DROP FUNCTION IF EXISTS {schema}.{module}_helper(TEXT, JSONB) CASCADE;

COMMIT;
```

---

## 对象类型 6：Control Function

### 设计规则

- **PostgREST 暴露**：函数签名和返回值需兼容 PostgREST RPC 调用
- **严格输入验证**：所有入参必须校验，非法输入返回明确错误
- **结构化返回**：返回 JSONB 或复合类型，便于前端解析
- **7 分区统一结构**：一～五在函数体内，六～七在脚本层
- 空分区保留分区头 + 注释原因：`-- 无（{reason}）`

### Deploy Template（7 分区）

```sql
-- ================================================================
-- URI: sqitch/deploy/{plan_name}.sql
-- ================================================================
--
-- Description:   {控制函数功能说明}（PostgREST RPC）
-- Sqitch Plan:   {plan_name}
-- Target Schema: {schema}
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
    v_sqlstate          TEXT;
    v_message_text      TEXT;
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

    -- 异常处理：控制函数返回结构化错误，不重抛
    EXCEPTION WHEN OTHERS THEN
        GET STACKED DIAGNOSTICS
            v_message_text = MESSAGE_TEXT,
            v_sqlstate     = RETURNED_SQLSTATE;
        RETURN jsonb_build_object(
            'code', v_sqlstate,
            'msg', v_message_text,
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

### Verify / Revert Template

与 Function 类型相同（verify + revert），不再重复。

---

## 对象类型 7：MView Refresh Function

### 设计规则

- **CONCURRENTLY 优先**：默认走 `REFRESH MATERIALIZED VIEW CONCURRENTLY`（不阻塞读），失败回退到非 CONCURRENTLY
- **永不传播异常（关键）**：mview_refresh 被 trigger 调用，异常传播会破坏用户 DML；外层必须 `EXCEPTION WHEN OTHERS THEN RAISE LOG` 兜底
- **RAISE LOG 兜底**：刷新失败 = 物化视图损坏 / schema 错配 / 系统级问题（锁/OOM/磁盘），属于运维事件，直接写 PG Server Log
- **JSON 结构化**：RAISE LOG 输出 `event / _location / mview / sqlstate / err` 字段，便于日志系统机器解析与告警

### Deploy Template（3 分区）

```sql
-- ================================================================
-- URI: sqitch/deploy/{plan_name}.sql
-- ================================================================
--
-- Description:   物化视图刷新函数 {schema}.{func_name}()
-- Sqitch Plan:   {plan_name}
-- Target Schema: {schema}
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
--
-- ================================================================

BEGIN;

DROP FUNCTION IF EXISTS {schema}.{func_name}() CASCADE;

COMMIT;
```

---

## 对象类型 8：Seed

### Seed 特殊规则

- **Deploy**：必须用 `DO $$ ... $$;` + DECLARE（需要 RETURNING id 传递给子级），`ON CONFLICT DO UPDATE` 幂等写入（PG 18 复杂 upsert 可用 `MERGE`）
- **Verify**：必须依靠唯一字段信息验证（分类用 code + pid，数据用 category_id + label），可用 count 计数
- **Revert**：移除基础数据时必须依靠唯一字段信息（非宽泛过滤）；移除层级数据时可级联
- **会话准备（可选）**：仅当项目有审计触发器需要自动填充 `created_by` 等字段时，设置会话变量（如 `set_config('app.user_name', '_seed', true)`）；否则省略

### Deploy Template（4 分区）

```sql
-- ================================================================
-- URI: sqitch/deploy/{plan_name}.sql
-- ================================================================
--
-- Description:   {种子数据说明}
-- Sqitch Plan:   {plan_name}
-- Target Schema: {schema}
--
-- ================================================================

BEGIN;

-- ====================================================================
-- 一、会话准备（可选，仅当审计触发器需要）
-- ====================================================================
-- SELECT set_config('app.user_name', '_seed', true);
-- 或 -- 无（{reason}）

-- ====================================================================
-- 二、种子数据写入
-- ====================================================================
DO $$
DECLARE
    v_{root}_id {id_type};
    v_{child}_id {id_type};
BEGIN
    -- --- 2.1 根分类（查找或创建）---------------------------------------
    INSERT INTO {schema}.{category_table} (pid, code, created_by)
    VALUES (NULL, '{code}', '_seed')
    ON CONFLICT (code, pid) DO UPDATE SET updated_by = EXCLUDED.created_by
    RETURNING id INTO v_{root}_id;

    -- --- 2.2 子分类树 --------------------------------------------------
    INSERT INTO {schema}.{category_table} (pid, code, created_by)
    VALUES (v_{root}_id, '{child_code}', '_seed')
    ON CONFLICT (code, pid) DO UPDATE SET updated_by = EXCLUDED.created_by
    RETURNING id INTO v_{child}_id;

    -- --- 2.N 数据种子 ---------------------------------------------
    INSERT INTO {schema}.{data_table} (category_id, label, value, created_by)
    VALUES (v_{child}_id, '{label}', '{value}'::jsonb, '_seed')
    ON CONFLICT (category_id, label) DO UPDATE SET
        value = EXCLUDED.value,
        updated_by = EXCLUDED.created_by;
END;
$$;

-- ====================================================================
-- 三、会话复原（可选）
-- ====================================================================
-- SELECT set_config('app.user_name', '{原值}', true);
-- 或 -- 无（同一）

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
--
-- ================================================================

BEGIN;

SELECT no_plan();

-- 分类验证（唯一字段：code + pid）
SELECT ok(EXISTS(
    SELECT 1 FROM {schema}.{category_table} WHERE code = '{code}' AND pid IS NULL
), '{分类} 应存在');

-- 数据计数验证（唯一字段：category path + label）
SELECT is(
    (SELECT count(*) FROM {schema}.{data_table} d
     WHERE d.category_id IN (
         SELECT sc.id FROM {schema}.{category_table} sc
         WHERE sc.code = '{child_code}' AND sc.pid IN (...)
     ))::bigint,
    {expected_count},
    '{category} 应有 {N} 条种子数据');

SELECT * FROM finish();

ROLLBACK;
```

### Revert Template（4 分区）

```sql
-- ================================================================
-- URI: sqitch/revert/{plan_name}.sql
-- ================================================================
--
-- Description:   撤销 {种子数据说明}
-- Sqitch Plan:   {plan_name}
-- Target Schema: {schema}
--
-- ================================================================

BEGIN;

-- ====================================================================
-- 一、会话准备（可选，仅当审计触发器需要）
-- ====================================================================
-- SELECT set_config('app.user_name', '_seed', true);
-- 或 -- 无（{reason}）

-- ====================================================================
-- 二、移除基础数据（按唯一字段定位）
-- ====================================================================
DELETE FROM {schema}.{data_table}
WHERE label = '{label}'
  AND category_id IN (
      SELECT sc.id FROM {schema}.{category_table} sc
      WHERE sc.code = '{child_code}' AND sc.pid IN (...)
  );

-- ====================================================================
-- 三、移除层级数据（叶到根，可级联）
-- ====================================================================
DELETE FROM {schema}.{category_table} WHERE code IN ('{leaf_codes}')
    AND pid IN (SELECT id FROM {schema}.{category_table} WHERE code = '{parent_code}');
DELETE FROM {schema}.{category_table} WHERE code = '{parent_code}';

-- ====================================================================
-- 四、会话复原（可选）
-- ====================================================================
-- SELECT set_config('app.user_name', '{原值}', true);
-- 或 -- 无（同一）

COMMIT;
```

---

## 对象类型 9：All Analysis（全类型分析视图分支追加）

### 适用场景

新增日志类型（注册 `{log_table}` + `archive_{log_table}` 表）后，向 `{schema}.{all_analysis_view}` 和 `{schema}.{full_analysis_view}` 追加 UNION ALL 分支。每次新增日志类型生成 **2 个独立 plan**，每个 plan 对应一个视图。

### 设计规则

- **每个 plan 对应一个视图**：`{all_analysis_add_plan}` 和 `{full_analysis_add_plan}`
- **Deploy**：读取当前视图定义，追加新分支后 `CREATE OR REPLACE VIEW`
- **Revert**：包含**完整的前版本视图定义**（不含新分支），`CREATE OR REPLACE VIEW` 还原
- **Verify**：仅需验证视图存在且可查询（smoke test），不需要详细结构验证
- **Revert 必须完整**：revert 脚本中的视图定义必须与上一个版本的完整定义一致，不能省略任何分支
- **列结构一致性**：追加分支必须保持与前版本分支完全相同的列名、顺序与类型

### 依赖关系

```
{log_table}_table ──→ archive_{log_table}_table ──→ {all_analysis_add_plan}
                                                  ──→ {full_analysis_add_plan}
```

两个 all_analysis plan 都依赖对应的 archive_table plan。`{full_analysis_add_plan}` 依赖 `{all_analysis_add_plan}`（部署顺序保证在线表先合并）。

### 新增分支模板

**在线视图 — 追加 1 个分支：**

```sql
SELECT {列清单}, to_jsonb(b.*) AS details
FROM {schema}.{log_table} b
```

**完整视图（在线 + 归档）— 追加 2 个分支：**

```sql
SELECT 0 AS is_archived,
       {列清单}, to_jsonb(b.*) AS details
FROM {schema}.{log_table} b
UNION ALL
SELECT 1 AS is_archived,
       {列清单}, to_jsonb(a.*) AS details
FROM {schema}.archive_{log_table} a
```

### Plan 1：{all_analysis_add_plan}

#### Deploy Template（3 分区）

```sql
-- ================================================================
-- URI: sqitch/deploy/{all_analysis_add_plan}.sql
-- ================================================================
--
-- Description:   追加日志类型 {log_table} 到 {all_analysis_view} 视图
-- Sqitch Plan:   {all_analysis_add_plan}
-- Target Schema: {schema}
--
-- ================================================================

BEGIN;

-- ====================================================================
-- 一、视图定义
-- ====================================================================

CREATE OR REPLACE VIEW {schema}.{all_analysis_view} AS
-- --- 前版本分支（原样保留，不可修改）---
{previous_branches}
-- --- 新增 {log_table} 分支 ---
UNION ALL
SELECT {列清单}, to_jsonb(b.*) AS details
FROM {schema}.{log_table} b;

-- ====================================================================
-- 二、视图注释
-- ====================================================================

COMMENT ON VIEW {schema}.{all_analysis_view} IS '全类型日志分析视图（在线表 UNION ALL）';

-- ====================================================================
-- 三、列注释
-- ====================================================================

-- 无（列注释由首次创建视图的 plan 设置，后续仅追加分支）

COMMIT;
```

#### Revert Template（3 分区）

```sql
-- ================================================================
-- URI: sqitch/revert/{all_analysis_add_plan}.sql
-- ================================================================
--
-- Description:   撤销日志类型 {log_table} 的 {all_analysis_view} 分支
-- Sqitch Plan:   {all_analysis_add_plan}
-- Target Schema: {schema}
--
-- ================================================================

BEGIN;

-- ====================================================================
-- 一、视图定义（还原为前版本完整定义）
-- ====================================================================

CREATE OR REPLACE VIEW {schema}.{all_analysis_view} AS
{previous_full_definition};

-- ====================================================================
-- 二、视图注释
-- ====================================================================

COMMENT ON VIEW {schema}.{all_analysis_view} IS '全类型日志分析视图（在线表 UNION ALL）';

-- ====================================================================
-- 三、列注释
-- ====================================================================

-- 无（同 deploy）

COMMIT;
```

#### Verify Template

```sql
-- ================================================================
-- URI: sqitch/verify/{all_analysis_add_plan}.sql
-- ================================================================
--
-- Description:   验证 {all_analysis_view} 视图存在
-- Sqitch Plan:   {all_analysis_add_plan}
-- Target Schema: {schema}
--
-- ================================================================

BEGIN;

SELECT no_plan();
SELECT has_view('{schema}', '{all_analysis_view}', '{schema}.{all_analysis_view} 视图应存在');
SELECT * FROM finish();

ROLLBACK;
```

### Plan 2：{full_analysis_add_plan}

#### Deploy Template（3 分区）

```sql
-- ================================================================
-- URI: sqitch/deploy/{full_analysis_add_plan}.sql
-- ================================================================
--
-- Description:   追加日志类型 {log_table} 到 {full_analysis_view} 视图
-- Sqitch Plan:   {full_analysis_add_plan}
-- Target Schema: {schema}
--
-- ================================================================

BEGIN;

-- ====================================================================
-- 一、视图定义
-- ====================================================================

CREATE OR REPLACE VIEW {schema}.{full_analysis_view} AS
-- --- 前版本分支（原样保留，不可修改）---
{previous_branches}
-- --- 新增 {log_table} 在线 + 归档分支 ---
UNION ALL
SELECT 0 AS is_archived,
       {列清单}, to_jsonb(b.*) AS details
FROM {schema}.{log_table} b
UNION ALL
SELECT 1 AS is_archived,
       {列清单}, to_jsonb(a.*) AS details
FROM {schema}.archive_{log_table} a;

-- ====================================================================
-- 二、视图注释
-- ====================================================================

COMMENT ON VIEW {schema}.{full_analysis_view} IS '全类型日志完整分析视图（在线 + 归档 UNION ALL）';

-- ====================================================================
-- 三、列注释
-- ====================================================================

-- 无（列注释由首次创建视图的 plan 设置，后续仅追加分支）

COMMIT;
```

#### Revert Template（3 分区）

```sql
-- ================================================================
-- URI: sqitch/revert/{full_analysis_add_plan}.sql
-- ================================================================
--
-- Description:   撤销日志类型 {log_table} 的 {full_analysis_view} 分支
-- Sqitch Plan:   {full_analysis_add_plan}
-- Target Schema: {schema}
--
-- ================================================================

BEGIN;

-- ====================================================================
-- 一、视图定义（还原为前版本完整定义）
-- ====================================================================

CREATE OR REPLACE VIEW {schema}.{full_analysis_view} AS
{previous_full_definition};

-- ====================================================================
-- 二、视图注释
-- ====================================================================

COMMENT ON VIEW {schema}.{full_analysis_view} IS '全类型日志完整分析视图（在线 + 归档 UNION ALL）';

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
-- URI: sqitch/verify/{full_analysis_add_plan}.sql
-- ================================================================
--
-- Description:   验证 {full_analysis_view} 视图存在
-- Sqitch Plan:   {full_analysis_add_plan}
-- Target Schema: {schema}
--
-- ================================================================

BEGIN;

SELECT no_plan();
SELECT has_view('{schema}', '{full_analysis_view}', '{schema}.{full_analysis_view} 视图应存在');
SELECT * FROM finish();

ROLLBACK;
```

---

## 跨类型补充规则

- **报告视图**：两种模式 — CTE 聚合，或 `_impl()` 函数透传
- **分析视图**：UNION ALL 合并在线 + 归档表，无触发器
- **All-analysis 视图更新规范**：新增日志类型时，必须使用 `all_analysis` 类型生成 2 个 plan，分别更新在线/完整视图；revert 必须包含完整的前版本视图定义
- **Seed mview 刷新顺序**：任何种子脚本若部署在 mview_refresh_func 之后，必须在脚本末尾调用对应的 mview 刷新函数，确保新写入的种子数据立即可见于物化视图
- **INSTEAD OF trigger 标准保护**：所有数据管理视图 trigger 应共享标准保护逻辑（id 自动生成、审计字段保护、计算列保护、DELETE=恢复语义），按项目约定沉淀
- **幂等 upsert（PG 18）**：需要「存在则更新、不存在则插入」的场景优先用 `MERGE ... WHEN MATCHED THEN UPDATE ... WHEN NOT MATCHED THEN INSERT ...`，比「先 DELETE 再 INSERT」更贴合实际语义且天然幂等

## Known Pitfalls

1. **json_matches_schema 返回 NULL**：校验失败返回 NULL（非 FALSE），用 `IS NOT TRUE` 判断
2. **pg_cron 会话隔离**：`ROLLBACK` 不回滚后台任务（pg_cron）的写入，需显式清理
3. **MView 刷新时机**：deploy 创建 MView 后必须执行 REFRESH；seed 写入后必须刷新；任何种子脚本部署在 mview_refresh_func 之后时，必须在末尾调用刷新函数
4. **COMMENT 字符串格式**：多行 COMMENT 用 `E'...\n\n'`（带 `E` 前缀），不用空字符串拼接
5. **Revert 顺序**：含触发器绑定的函数 — 先 DROP TRIGGER 再 DROP FUNCTION
6. **Revert seed 顺序**：先 DELETE 数据行（data），再 DELETE 层级（category，叶到根）
7. **Verify 范围**：verify 只做存在性检查，详细验证交给 pgTAP 测试
8. **has_function() 参数**：有参用 `ARRAY['type1', 'type2']`；无参用 `ARRAY[]::text[]`
9. **Domain 类型**：`col_type_is()` 检查底层基础类型，非 DOMAIN 名
10. **归档表 INCLUDING COMMENTS**：归档表 `LIKE` 必须包含 `INCLUDING COMMENTS` 以继承源表列注释，然后只需添加 `archived_at` 注释
11. **mview_refresh_function 异常兜底**：被 trigger 调用时，**必须**外层 `EXCEPTION WHEN OTHERS THEN RAISE LOG`（JSON 结构化），不能传播异常，否则破坏用户 DML 事务
12. **MAX/MIN 不可聚合类型**：PG 聚合函数 MAX/MIN **不支持** boolean / json / jsonb / xml 等不可排序类型，条件聚合需先 cast 到 text 再聚合再 cast 回
13. **`->` 操作符优先级陷阱**：`value->'key'::text` 被解析为 `value -> ('key'::text)`，cast 必须加括号 `(value->'key')::text`
14. **函数级 SET search_path（调用其他 schema 的扩展函数）**：函数内调用其他 schema 的扩展函数，**必须**函数级 `SET search_path`（如 `SET search_path = pg_catalog, public, {schema}`）；否则在受限 search_path 上下文（如 cronjob），unqualified 函数解析失败（42883）
15. **兜底值常量化（避免魔法值）**：函数内的默认值/兜底值（错误级别、启用标志、默认 SQLSTATE 等）用 DECLARE `CONSTANT` 集中声明，不在赋值/COALESCE 中写魔法值
16. **非事务性语句不能混进 change**：`CREATE INDEX CONCURRENTLY` / `DROP INDEX CONCURRENTLY` / `REFRESH MATERIALIZED VIEW CONCURRENTLY` / `ALTER TYPE ... ADD VALUE` 等不能在事务块内执行，需单独处理或放到事务外
17. **identity 列（PG 18）**：`GENERATED ... AS IDENTITY` 列的默认值是内部序列，revert/deploy 往返时注意 identity 序列归属；verify 用 `pg_attribute.attidentity`（`'a'`=ALWAYS / `'d'`=BY DEFAULT）校验
18. **uuidv7()（PG 18）**：`id uuid DEFAULT uuidv7()` 作为默认值时，验证生成值为版本 7（`uuid_extract_version() = 7`），避免直接比较具体值
19. **MERGE 幂等性**：upsert 场景优先用 `MERGE` 验证幂等（同一键重复执行不产生重复行、计数不变）

## 验证（完成标准）

- 干净库 `sqitch deploy --verify db:pg:<db>` 成功
- 已部署库再次 `sqitch verify db:pg:<db>` 成功（幂等）
- `sqitch revert <tag>` 再 `sqitch deploy <tag>` 的 round-trip 无报错
- 每个 change 的 deploy / revert / verify 三脚本齐全、头格式统一、注释为中文
- `sqitch log` 历史清晰、依赖正确
