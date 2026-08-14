---
name: sql-pgtap
description: 使用 pgTAP 为 PostgreSQL 18 编写数据库测试，并用 pg_prove 运行校验。在编写数据库测试、校验迁移结果、把测试接入 sqitch verify 或 CI 时使用。迁移脚本编写请用 sql-sqitch skill。
license: PostgreSQL License
compatibility: PostgreSQL 18；pgTAP 1.x；pg_prove
---

# pgTAP 校验（pg_prove 测试）

## 何时使用

- 为数据库对象（表、列、约束、索引、函数、视图）编写测试。
- 用 `pg_prove` 运行测试、校验迁移是否生效。
- 把测试接入 sqitch verify 或 CI/CD。

## 前置要求

- PostgreSQL 18
- pgTAP 扩展（测试库中执行 `CREATE EXTENSION pgtap`）
- `pg_prove`（随 pgTAP 分发）

## 测试库准备

```sql
-- 在测试/scratch 库执行一次
CREATE EXTENSION IF NOT EXISTS pgtap;
```

> 使用专用测试库，不要在生产库上跑测试。

## 测试文件结构

每个测试文件一个完整计划，包裹在事务里并在结尾回滚，保证测试不污染数据库：

```sql
-- test/users_table.sql
BEGIN;

SELECT plan(4);  -- 断言总数，必须与实际一致

SELECT has_schema('app');
SELECT has_table('app', 'users');
SELECT has_column('app', 'users', 'email');
SELECT col_is_pk('app', 'users', 'id');

SELECT * FROM finish();
ROLLBACK;
```

## 常用断言函数

| 函数 | 用途 |
|------|------|
| `has_schema(name)` | schema 存在 |
| `has_table(schema, name)` | 表存在 |
| `has_column(schema, table, col)` | 列存在 |
| `col_is_pk(schema, table, col)` | 列是主键 |
| `col_not_null(schema, table, col)` | 列非空 |
| `col_type_is(schema, table, col, type)` | 列类型 |
| `has_index(schema, table, idx)` | 索引存在 |
| `has_unique(schema, table, cols)` | 唯一约束 |
| `has_foreign_key(schema, table, fk)` | 外键 |
| `has_function(schema, func)` | 函数存在 |
| `has_view(schema, view)` | 视图存在 |
| `is(actual, expected, desc)` | 值相等断言 |
| `ok(cond, desc)` | 布尔断言 |
| `throws_ok(sql, errcode, desc)` | 期望抛错 |

## 运行测试

```bash
# 单文件
pg_prove -d <dbname> test/users_table.sql

# 整个目录
pg_prove -d <dbname> test/*.sql

# 常用选项
pg_prove -d <dbname> --set ON_ERROR_STOP=1 test/
```

认证：通过 `PGPASSWORD`、`~/.pgpass` 或本地 trust 提供凭据。

## 与 sqitch 集成

测试文件放在 `test/`，与对应 change 一起提交；CI 里先 `sqitch deploy` 再 `pg_prove`：

```bash
sqitch deploy --verify db:pg:$PGDATABASE
pg_prove -d $PGDATABASE test/
```

## 陷阱

1. **没装 pgTAP 扩展**：报 `function plan() does not exist` → 先 `CREATE EXTENSION pgtap`。
2. **plan 计数不符**：`plan(n)` 的数字必须与实际测试数一致，否则 fail。
3. **忘记 ROLLBACK**：测试会污染测试库，影响后续测试。
4. **标识符大小写**：pgTAP 按实际大小写匹配，注意引号与大小写（`has_table('app','Users')` 与 `users` 不同）。
5. **权限**：`pg_prove` 用当前用户连接，权限不足会报错。
6. **测试隔离**：断言应在事务内，破坏性操作要能在 `ROLLBACK` 下撤销。

## 验证（完成标准）

- `pg_prove -d <db> test/` 退出码 0，全部通过。
- 测试后测试库无残留对象（ROLLBACK 生效）。
- 每个新增/修改的 change 都有对应测试覆盖。
