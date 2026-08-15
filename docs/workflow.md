# Issue + Orca 看板联动工作流

用 GitHub Issues 记录和管理开发计划，用 Orca 工作区看板跟踪执行状态，两者通过
issue 号关联，形成「计划 → 执行 → 交付 → 回顾」闭环。

## 分层体系

| 层级 | 模板 | 业务编号 | label | 用途 |
|------|------|---------|-------|------|
| Epic | `1_epic.md` | `E01` | `epic` | 业务域 / 阶段计划 |
| Story | `2_story.md` | `S0101` | `story` | 用户故事 / 功能需求（执行单元，对应一个 worktree） |
| Bug | `4_bug.md` | — | `bug` | 缺陷 / 故障 |

Story 是执行单元：一个 Story 对应一个 Orca worktree。Story 内的执行步骤用两级
Checklist 记录，不再单独建 Task issue。

### 编号规则

| 层级 | 格式 | 示例 | 规则 |
|------|------|------|------|
| Epic | `E{NN}` | `E01` | 2 位全局递增 |
| Story | `S{NNNN}` | `S0102` | epic 2 位 + story 2 位 |
| 步骤（一级 Checklist） | `S{NNNN}-{NN}` | `S0102-01` | story 编号 + 步骤 2 位 |
| 子项（二级 Checklist） | `S{NNNN}-{NNNN}` | `S0102-0101` | story 编号 + 步骤 2 位 + 子项 2 位 |

编号放在标题最前，如 `E01 - 目标`、`S0102 - 功能`。
业务编号由人工维护（GitHub 无法自动递增），需在创建时手动填写。

层级关系：**Epic → Story**，通过 issue 正文里的 `#编号` 引用串联；Story 过大时
拆分为多个 Story，保持单个 Story 可在一个 worktree 内完成。Bug 独立，修复后可
关联到对应 Story。

## 与 Orca 看板的联动

| 维度 | GitHub 侧 | Orca 侧 |
|------|----------|---------|
| 类型（做什么） | issue label（epic/story/bug） | worktree 卡片关联 issue 号 |
| 状态（做到哪步） | open / closed（PR 关闭） | `workspace-status` 看板列：`todo` / `in-progress` / `in-review` / `completed` |

两者通过 **issue 号** 串起来。

### 状态映射（两套状态，不自动同步）

issue 本身不会自动出现在看板上——必须 `orca worktree create --issue <N>` 开 worktree
关联后，看板才出现卡片。两套状态各自独立，只有「完成」这一端会自动同步：

| Orca 看板列（worktree 侧，4 态） | 含义 | GitHub issue 侧（2 态） |
|----------------------------------|------|------------------------|
| `todo` | 已建 worktree，未开始 | open |
| `in-progress` | agent 执行中 | open |
| `in-review` | PR 已建，待审核 | open |
| `completed` | 已合并交付 | **closed**（merge 时自动关闭） |

> 中间态（`in-progress` / `in-review`）只在 Orca 看板表达；GitHub issue 只有
> open / closed，无法表达中间态。`completed` ↔ `closed` 是唯一自动同步的点。

关键命令：

```bash
# 开 worktree 并关联 issue
orca worktree create --issue <N> --base-branch <ref> --agent <id> --prompt "..."

# 更新看板列
orca worktree set --worktree issue:<N> --workspace-status <id>

# 按 issue 反查 worktree
orca worktree show --worktree issue:<N>
```

## 完整流程（以 Story 为例）

```bash
# 1. 计划：GitHub 网页新建 issue（Story 模板，自动打 story label）→ 得 #N（看板尚无卡片）
# 2. 开 worktree 关联 issue → 看板出现卡片（todo）
orca worktree create --issue N --base-branch main --agent pi --prompt "..."
# 3. 派发任务 → 移 in-progress
orca worktree set --worktree issue:N --workspace-status in-progress
# 4. 完成建 PR → 移 in-review，并关联关闭 issue
gh pr create --base main --head <branch> --body "Closes #N"
orca worktree set --worktree issue:N --workspace-status in-review
# 5. review + merge 后 → 移 completed（issue 自动 closed），清理执行现场
orca worktree set --worktree issue:N --workspace-status completed
orca worktree rm --worktree issue:N --force
```

## 层级协作节奏

1. **建 Epic**：规划阶段，列出 Story 范围与非目标。
2. **拆 Story**：为每个 Epic 建 Story issue，写明验收标准 + 两级任务 Checklist，
   `#` 引用 Epic。Story 是 Orca worktree 的最小执行单元。
3. **Story 过大**：拆分为多个 Story，保持单个 Story 可在一个 worktree 内完成。
4. **记 Bug**：发现缺陷直接开 Bug issue，修复走 Story 流程，PR 用 `Closes #N`
   同时关闭 Bug。

## 复用到其他项目

1. 复制模板目录：`cp -r .github/ISSUE_TEMPLATE <目标项目>/.github/`
2. 创建 labels：

   ```bash
   gh label create epic  --color 8B5CF6 --description "Epic 业务域/阶段计划"
   gh label create story --color 3B82F6 --description "Story 用户故事/功能需求"
   gh label create bug   --color EF4444 --description "Bug 缺陷/故障" --force
   ```

3. 按需调整 `config.yml` 里的 `contact_links` 链接。

## 注意事项

- issue 模板里的 `labels:` 引用的 label 必须先在仓库存在，否则 GitHub 会忽略该 label。
- `workspace-status` 默认列为 `todo` / `in-progress` / `in-review` / `completed`，
  自定义看板列请用对应配置的 id。
- Orca 的 `--issue` 只做关联记录，不负责创建/关闭 issue；创建与关闭用 `gh` 或网页。
- PR 正文写 `Closes #N`（或 `Fixes #N`），merge 后 issue 自动关闭，实现状态闭环。
