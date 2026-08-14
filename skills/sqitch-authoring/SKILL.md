---
name: sqitch-authoring
description: 使用 Sqitch 为 PostgreSQL 18 编写数据库迁移脚本（deploy/revert/verify）的规范与流程。在新增或修改数据库变更、编写迁移脚本、维护 sqitch.plan、执行 deploy/revert/verify 时使用。测试校验请用 pgtap-validation skill。
license: PostgreSQL License
compatibility: PostgreSQL 18；Sqitch 1.x；psql；git
---

# Sqitch 脚本编制（deploy / revert / verify）

## 何时使用

- 需要新增、修改或删除数据库对象（表、列、索引、约束、函数、视图、类型、触发器、扩展等）
- 编写或审查 Sqitch 的 `deploy` / `revert` / `verify` 脚本
- 维护 `sqitch.plan`、打发布 tag、回滚或重放迁移

## 前置要求

- PostgreSQL 18（`psql` 可用）
- Sqitch（`sqitch --version`）
- 可连接的数据库 URI：`db:pg:<dbname>` 或 `db:pg://user@host/dbname`

## 核心概念

| 概念 | 说明 |
|------|------|
| change | 一次原子变更，含 `deploy` / `revert` / `verify` 三个脚本 |
| sqitch.plan | 变更的有序列表（含依赖与 tag），必须提交进 git |
| deploy | 正向脚本（DDL），把数据库从旧状态推进到新状态 |
| revert | 逆脚本，精确撤销 deploy 的改动 |
| verify | 部署后断言脚本，检查 deploy 是否生效 |
| tag | 发布点（如 `v1.0.0`），用于 `sqitch revert <tag>` |

## 标准流程

```bash
# 1. 新增变更（snake_case，动词开头）
sqitch add <change_name> -n "<说明>"                                # 无依赖
sqitch add <change_name> --require <父change> -n "<说明>"           # 依赖其他变更

# 2. 编写 deploy/<name>.sql / revert/<name>.sql / verify/<name>.sql

# 3. 部署并校验
sqitch deploy --verify db:pg:<dbname>

# 4. 回滚/重放测试（round-trip）
sqitch revert db:pg:<dbname> && sqitch deploy db:pg:<dbname>

# 5. 提交（deploy/revert/verify + 测试 + plan 一个原子提交）
git add -A && git commit -m "feat: <change 说明>"

# 6. 发布打 tag
sqitch tag v1.0.0 -n "首个发布"
```

## deploy 脚本规范

- 一个 change 只做一件事。
- DDL 尽量幂等：`CREATE TABLE IF NOT EXISTS`、`CREATE INDEX IF NOT EXISTS`、`ADD COLUMN IF NOT EXISTS`。
- 显式写 schema 名，不依赖默认 `search_path`。

```sql
-- deploy/add_users_table.sql
-- 前提：app schema 已由前置 change（如 add_app_schema）建立
CREATE TABLE IF NOT EXISTS app.users (
    id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email       text NOT NULL,
    created_at  timestamptz NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX IF NOT EXISTS users_email_key ON app.users (email);
```

## revert 脚本规范

- 是 deploy 的精确逆操作，顺序相反。
- 同样幂等：`DROP ... IF EXISTS`。

```sql
-- revert/add_users_table.sql
DROP TABLE IF EXISTS app.users CASCADE;
-- 索引随表自动删除，无需单独 DROP
```

## verify 脚本规范

- 用 `DO $$ ... $$` 块断言，失败就 `RAISE EXCEPTION`（fail loudly）。

```sql
-- verify/add_users_table.sql
DO $$
BEGIN
    IF NOT EXISTS (
        SELECT 1 FROM pg_tables
        WHERE schemaname = 'app' AND tablename = 'users'
    ) THEN
        RAISE EXCEPTION 'app.users 表不存在';
    END IF;

    IF NOT EXISTS (
        SELECT 1 FROM pg_attribute a
        JOIN pg_class c ON c.oid = a.attrelid
        JOIN pg_namespace n ON n.oid = c.relnamespace
        WHERE n.nspname = 'app' AND c.relname = 'users'
          AND a.attname = 'email' AND NOT a.attisdropped
    ) THEN
        RAISE EXCEPTION 'app.users.email 列不存在';
    END IF;
END $$;
```

## 命名约定

- change 名：snake_case，动词开头，描述「动作 + 对象」：
  - `add_users_table`、`add_email_uniq_index`、`alter_users_add_role`、`drop_legacy_view`
- tag：语义化版本，如 `v1.0.0`。

## 陷阱（务必注意）

1. **绝不修改已部署的脚本**：一旦 deploy 到环境，该 change 的 deploy/revert/verify 内容即冻结。需要变更就新建 change，否则 plan 与实际状态漂移、revert 失败。
2. **revert 必须精确撤销 deploy**：漏掉对象会导致 round-trip（revert 后再 deploy）报错。
3. **非事务性语句不能混进 change**：PostgreSQL 的 DDL 是事务性的，Sqitch 默认把每个 change 包在一个事务里。以下语句不能在事务中使用，需单独处理：
   - `CREATE DATABASE`
   - `CREATE INDEX CONCURRENTLY` / `DROP INDEX CONCURRENTLY`
   - `VACUUM`
   - `REFRESH MATERIALIZED VIEW CONCURRENTLY`
   - `ALTER TYPE ... ADD VALUE`（PG 12+ 允许，但同一事务内不能立即使用新值）
4. **幂等性**：deploy 用 `IF NOT EXISTS`，revert 用 `IF EXISTS`，避免重复执行报错。
5. **search_path**：脚本里显式写 schema 名，防止对象落到错误 schema。
6. **verify 要 fail loudly**：断言失败必须 `RAISE EXCEPTION`，不能静默通过。

## 验证（完成标准）

- 干净库 `sqitch deploy --verify db:pg:<db>` 成功。
- 已部署库再次 `sqitch verify db:pg:<db>` 成功（幂等）。
- `sqitch revert <tag>` 再 `sqitch deploy <tag>` 的 round-trip 无报错。
- `sqitch log` 历史清晰、依赖正确。
