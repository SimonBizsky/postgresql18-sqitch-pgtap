---
name: sql-check
description: 审核 Sqitch 脚本（deploy/verify/revert）与 pgTAP 测试文件是否符合对应 skill 模板规范，输出合规报告（通过/小偏离/大偏离 + 修复建议）。在用户要求检查或审核脚本合规性时使用。
license: PostgreSQL License
compatibility: PostgreSQL 18；Sqitch；pgTAP；pg_prove
---

# sql-check（脚本合规审核）

> 内容待从 `reference/sql-check` 泛化。当前仅固定 skill 名称与职责。

## 职责

审核两个 skill 的产物是否符合模板规范：

- **sql-sqitch 产物**：`deploy` / `verify` / `revert` 脚本
- **sql-pgtap 产物**：pgTAP 测试文件

输出合规报告：通过 / 小偏离 / 大偏离 + 修复建议（大偏离时建议用对应 skill 重新编制）。
