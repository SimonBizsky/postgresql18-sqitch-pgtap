# 参考 Skill（Reference）

本目录存放从 **bizsky** 项目（`/Users/s1mon/mywork/bizsky/.claude/skills/`）复制的
SQL 开发 skill，作为本项目 skills 的**参考来源与泛化基础**。

## 来源

| Skill | 作用 | 说明 |
|-------|------|------|
| `sql-sqitch` | Sqitch `deploy`/`verify`/`revert` 脚本生成器 | 11 种对象类型模板：table / archive_table / view / base_function / function / helper / cronjob_function / control_function / mview_refresh_function / seed / all_analysis |
| `sql-test` | pgTAP 测试骨架生成器 | unit / integration / performance / security 四类，36 个固定分组（T01-T36） |
| `sql-check` | 脚本合规审计器 | 检查 Sqitch 脚本 + 测试文件是否符合对应模板 |

## 注意事项

这些 skill 深度绑定 bizsky 项目特定规范，泛化为通用 PostgreSQL 18 skill 时需剥离：

- 多租户字段 `tenant_code`、审计触发器 `common.audit_trigger()`
- GUC 会话变量（`bizsky.user_name`、`bizsky.bypass_audit` 等）
- 异常编码体系（5 位 SQLSTATE = 来源 + schema + 分类）
- Story / BMAD 流程（`Story X.X`、`[SKILL:...]` 触发语法）
- 模块划分（`common` / `dict` / `log` schema）

## 与本项目 skills 的对应关系

| 参考 Skill | 对应本项目 Skill |
|-----------|-----------------|
| `sql-sqitch` | `skills/sql-sqitch`（脚本编制） |
| `sql-test` | `skills/sql-pgtap`（测试校验） |
| `sql-check` | `skills/sql-check`（审核） |
