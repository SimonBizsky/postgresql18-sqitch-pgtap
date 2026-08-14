---
name: sql-check
description: 'Audit Sqitch and test scripts against their corresponding skill templates. Checks deploy/verify/revert scripts for template compliance, naming conventions, and structural correctness. Also validates pgTAP test files against spec-tdd conventions. Use when the user says "check scripts" or "audit scripts" or task contains [SKILL:sql-check].'
---

# SQL Script Compliance Auditor

Audit Sqitch deploy/verify/revert scripts and pgTAP test files against their corresponding skill templates.

## Conventions

- All output in Chinese
- File paths resolve from `{project-root}`
- Reference skills: `sql-sqitch` (Sqitch 模板) + `sql-test` (测试模板)
- Reference spec: `_bmad-output/specs/spec-tdd.md`

## Skill Type`技能类型`

- Category: 代码审计器
- Trigger: 当 task 描述包含 `[SKILL:sql-check]` 时调用
- Input: Story 文件路径（提取 plan 表和 skill 类型映射）
- Output: 合规报告（通过/小偏离/大偏离 + 修复建议）
- Complement: [SKILL:sql-sqitch] + [SKILL:sql-test] — 大偏离时建议用对应 skill 重新编制

## On Activation

Ask the user for:

1. **Story file path**: 从中提取 Sqitch Plan 表（plan 名称 + skill 类型映射）
2. **Check scope** (optional):
   - `sqitch` — 仅检查 Sqitch 脚本
   - `test` — 仅检查测试脚本
   - `all`（默认）— 检查全部

If user provides a story file, automatically extract the plan table and check all corresponding scripts.

## Output Format

```
============================================================
SQL Script Compliance Report`脚本合规报告`
============================================================

Story: {story_name}
Date: {audit_date}
Scope: sqitch + test

--- Sqitch Scripts ---

[{status}] {plan_name}
  Type: {skill_type}
  Deploy: {status_icon} {result}
  Verify: {status_icon} {result}
  Revert: {status_icon} {result}
  Issues:
    - {issue_1}
    - {issue_2}
  Fix: [SKILL:sql-sqitch:{type}] （仅大偏离时显示）

--- Test Scripts ---

[{status}] {test_file}
  Type: {test_type}
  Issues:
    - {issue_1}

============================================================
Summary: {pass_count} passed, {minor_count} minor, {major_count} major
============================================================
```

Status icons:
- `[PASS]` ✅ — 合规
- `[MINOR]` ⚠️ — 小偏离（格式/注释问题，手动修复）
- `[MAJOR]` ❌ — 大偏离（结构错误/未使用 skill，建议重新编制）

---

## Check Rules: Sqitch Scripts`Sqitch 脚本检查规则`

### Common Checks (all types)

| # | Check | Severity | Description |
|---|-------|----------|-------------|
| C01 | Header Format | MINOR | 标准头完整（URI/Description/Sqitch Plan/Target Schema/Related Story） |
| C02 | Header URI | MINOR | URI 路径与实际文件路径一致 |
| C03 | Header Consistency | MINOR | deploy/verify/revert 三文件 Header 信息一致 |
| C04 | BEGIN/COMMIT | MAJOR | deploy + revert 必须 COMMIT；verify 必须 ROLLBACK |
| C05 | File Existence | MAJOR | deploy/verify/revert 三文件必须同时存在 |
| C06 | Non-empty | MAJOR | 所有文件必须有实际内容（非空 revert/verify） |
| C07 | sqitch.plan Match | MAJOR | sqitch.plan 中 requires 与 Story Plan 表一致 |

### Type-Specific Deploy Checks

#### table (10 分区)

| # | Check | Severity | Description |
|---|-------|----------|-------------|
| T01 | Partition Count | MAJOR | 必须包含 10 个分区（一~十） |
| T02 | Partition Order | MAJOR | 分区顺序：标准字段→业务字段→表注释→标准字段注释→业务字段注释→业务字段约束→业务字段约束注释→索引→索引注释→触发器绑定 |
| T03 | Standard Fields | MAJOR | 包含完整标准字段（8/9/6 视变体而定） |
| T04 | Audit Trigger | MAJOR | 分区十必须绑定 `common.audit_trigger()` |
| T05 | Column Comments | MINOR | 所有列必须有 COMMENT ON COLUMN |
| T06 | Constraint Comments | MINOR | 所有约束必须有 COMMENT ON CONSTRAINT |

#### archive_table (2 分区)

| # | Check | Severity | Description |
|---|-------|----------|-------------|
| A01 | Partition Count | MAJOR | 必须包含 2 个分区（一、表结构 + 二、对象注释） |
| A02 | LIKE Source | MAJOR | 使用 `LIKE log.log_{type} INCLUDING DEFAULTS INCLUDING CONSTRAINTS INCLUDING COMMENTS` |
| A03 | archived_at | MAJOR | 包含 `archived_at TIMESTAMPTZ NOT NULL DEFAULT now()` |
| A04 | No Audit Trigger | MAJOR | 归档表不绑定审计触发器 |

#### view (6 分区)

| # | Check | Severity | Description |
|---|-------|----------|-------------|
| V01 | Partition Count | MAJOR | 必须包含 6 个分区（一~六） |
| V02 | Partition Order | MAJOR | 分区顺序：视图定义→索引→索引注释→视图注释→列注释→附加操作 |
| V03 | Column Comments | MINOR | 所有列必须有 COMMENT ON COLUMN |
| V04 | View Comment | MINOR | 必须有 COMMENT ON VIEW/MATERIALIZED VIEW |

#### base_function / function / control_function (7 分区)

| # | Check | Severity | Description |
|---|-------|----------|-------------|
| F01 | Partition Count | MAJOR | 必须包含 7 个分区（一~七） |
| F02 | Partition Order | MAJOR | 分区顺序：入参校验→资源查询→业务逻辑→返回值→特殊函数→注释→附加操作 |
| F03 | v_location | MAJOR | 声明 `v_location CONSTANT TEXT := '{plan_name}'` |
| F04 | Function Comment | MINOR | 必须有 COMMENT ON FUNCTION |
| F05 | INSTEAD OF Binding | MAJOR | trigger 函数末尾必须有 `CREATE TRIGGER ... INSTEAD OF ... ON {view}` 绑定（仅 function 类型 trigger variant） |

#### mview_refresh_function (3 分区)

| # | Check | Severity | Description |
|---|-------|----------|-------------|
| M01 | Partition Count | MAJOR | 必须包含 3 个分区（一~三） |
| M02 | Partition Order | MAJOR | 分区顺序：函数定义→注释→附加操作 |
| M03 | REFRESH | MAJOR | 包含 `REFRESH MATERIALIZED VIEW` 语句 |

#### seed (deploy 4 分区 / revert 5 分区)

| # | Check | Severity | Description |
|---|-------|----------|-------------|
| S01 | Deploy Partitions | MAJOR | deploy 必须包含 4 分区（GUC准备→种子数据写入→GUC复原→附加操作） |
| S02 | Revert Partitions | MAJOR | revert 必须包含 5 分区（GUC准备→数据移除→分类移除→GUC复原→附加操作） |
| S03 | GUC Save/Restore | MAJOR | deploy/revert 都包含 `set_config('bizsky.user_name', ...)` + 复原 |
| S04 | bypass_audit | MAJOR | revert 包含 `SET LOCAL bizsky.bypass_audit = 'true'` |
| S05 | Idempotent | MAJOR | INSERT 使用 `ON CONFLICT` 或条件检查确保幂等 |
| S06 | MView Refresh | MINOR | deploy 末尾调用对应 `*_mview_refresh()` |

### Verify/Revert Checks

| # | Check | Severity | Description |
|---|-------|----------|-------------|
| R01 | Verify Framework | MAJOR | verify 使用 pgTAP（`no_plan()` + `finish()` + `ROLLBACK`） |
| R02 | Verify Scope | MINOR | verify 仅检查对象存在性（不测试行为） |
| R03 | Revert Logic | MAJOR | revert 正确 DROP 对应对象（表/视图/函数/种子数据） |
| R04 | Revert Order | MAJOR | 种子 revert 先删 data 后删 category（从叶到根） |

---

## Check Rules: Test Scripts`测试脚本检查规则`

### Unit Test Checks

| # | Check | Severity | Description |
|---|-------|----------|-------------|
| U01 | 36 Groups | MAJOR | 必须包含 T01-T36 全部 36 个分组 |
| U02 | 5 Sections | MAJOR | 必须包含 Section A-E 全部 5 个 Section |
| U03 | TTXXX Numbering | MINOR | 编号格式 `TTXXX`（TT=01-36, XXX=001-999） |
| U04 | Section Delimiters | MINOR | Section 用 `===` 分隔线，Group 用 `---` 分隔线 |
| U05 | Empty Group Annotation | MINOR | 空组保留分组头 + `-- 说明: 此分组为空（{reason}）` |
| U06 | MUST Functions | MAJOR | MUST 级分组必须使用指定 pgTAP 函数（见 spec-tdd 函数映射表） |
| U07 | Transaction Wrap | MAJOR | 所有断言在 `BEGIN` / `ROLLBACK` 内 |
| U08 | Assertion Message | MINOR | 每个断言包含编号前缀：`[T01001]` |
| U09 | File Header | MINOR | 标准头完整（URI/Type/Description/Related Story/Test Objects/Test Group Overview） |
| U10 | pgTAP Framework | MAJOR | 使用 `\unset AUTOCOMMIT` + `\set ON_ERROR_STOP true` + `no_plan()` + `finish()` |

### Integration Test Checks

| # | Check | Severity | Description |
|---|-------|----------|-------------|
| I01 | FNNXXX Numbering | MINOR | 编号格式 `FNNXXX`（NN=01-07, XXX=001-999） |
| I02 | Scenario Labels | MINOR | 每个场景标注 F01-F07 场景名称 |
| I03 | Assertion Fill | MAJOR | 所有断言已填充实际验证逻辑（非占位符） |
| I04 | Data Isolation | MINOR | 使用 `test.` 前缀或 `LIKE 'test.%'` 模式 |
| I05 | Cleanup | MINOR | 测试数据在 ROLLBACK 或显式 DELETE 中清理 |
| I06 | bypass_audit | MINOR | 清理 DELETE 前设置 `SET LOCAL bizsky.bypass_audit = 'true'` |
| I07 | File Header | MINOR | 标准头完整 |

### Performance Test Checks

| # | Check | Severity | Description |
|---|-------|----------|-------------|
| P01 | GNNXXX Numbering | MINOR | 编号格式 `GNNXXX`（NN=01-04, XXX=001-999） |
| P02 | NFR Labels | MINOR | 每个测试标注 G01-G04 NFR 类别 |
| P03 | Timing Functions | MAJOR | 使用 `performs_ok()` 或 `performs_within()` |
| P04 | Threshold Realistic | MINOR | 阈值标注 CI 环境倍率说明 |
| P05 | ANALYZE | MINOR | 大量 INSERT 后包含 ANALYZE |
| P06 | File Header | MINOR | 标准头完整 |

---

## Execution Workflow

### Step 1: Parse Story

1. 读取 Story 文件
2. 从 Sqitch Plan 注册清单提取：plan 名称、object 类型、schema、depends on
3. 从 Task 分解提取：每个 subtask 对应的 `[SKILL:sql-sqitch:{type}]` / `[SKILL:sql-test:{type}]`
4. 建立 plan → skill_type 映射

### Step 2: Check Sqitch Scripts

对每个 plan 执行：

1. 检查 deploy/verify/revert 三文件是否存在
2. 读取每个文件内容
3. 执行 Common Checks (C01-C07)
4. 根据 plan 的 skill_type 执行 Type-Specific Checks
5. 执行 Verify/Revert Checks (R01-R04)
6. 汇总结果

### Step 3: Check Test Scripts

从 Story 的 Task 1 提取测试文件列表，对每个文件：

1. 检查文件存在
2. 读取文件内容
3. 根据测试类型执行对应检查规则
4. 汇总结果

### Step 4: Generate Report

按 Output Format 输出合规报告：
- 每个脚本独立列出状态和问题
- 大偏离标注建议重编的 skill 命令
- 末尾汇总统计

### Step 5: Fix Recommendations

对大偏离脚本，输出修复命令：
```
修复建议：
  → [SKILL:sql-sqitch:view] 重新编制 log_log_system_analysis
  → [SKILL:sql-test:unit] 重新编制 test/db/log_sys_unit_mgmt.sql
```
