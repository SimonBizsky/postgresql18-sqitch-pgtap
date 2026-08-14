# PostgreSQL 18 开发 Skills

一组用于 PostgreSQL 18 数据库开发的开源 [Agent Skills](https://agentskills.io/specification)，
围绕 **Sqitch**（迁移脚本编制）+ **pgTAP / pg_prove**（测试校验）+ **Git** 工作流。

## 包含的 Skills

| Skill | 用途 | 路径 |
|-------|------|------|
| `sqitch-authoring` | 脚本编制：编写 Sqitch `deploy` / `revert` / `verify` 迁移脚本 | [skills/sqitch-authoring](./skills/sqitch-authoring) |
| `pgtap-validation` | 校验：编写并运行 pgTAP 测试（`pg_prove`） | [skills/pgtap-validation](./skills/pgtap-validation) |

## 安装使用

这些 skills 遵循 [Agent Skills 标准](https://agentskills.io/specification)，可被支持的 agent 直接加载。

**Pi**：

```bash
# 复制到全局 skills 目录
cp -r skills/* ~/.pi/agent/skills/

# 或按路径临时加载
pi --skill skills/sqitch-authoring --skill skills/pgtap-validation
```

**Claude Code**：

```bash
cp -r skills/* ~/.claude/skills/
```

**其他 harness**：参考其 skills 目录配置，把本仓库的 `skills/` 加入加载路径。

## 目录结构

```
skills/
├── sqitch-authoring/     # 脚本编制 skill
│   └── SKILL.md
└── pgtap-validation/     # 校验 skill
    └── SKILL.md

reference/               # 参考 skill（来自 bizsky，供泛化参考）
├── sql-sqitch/
├── sql-test/
├── sql-check/
└── README.md
```

## 前置要求（使用这些 skills 的目标环境）

- PostgreSQL 18
- Sqitch（含 PostgreSQL 支持）
- pgTAP + `pg_prove`

> 本仓库只提供 skills，不含运行时环境；工具链请在目标环境自行安装。

## License

[PostgreSQL License](./LICENSE)
