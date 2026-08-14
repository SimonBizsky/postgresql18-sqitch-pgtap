# Issue + Orca 看板联动工作流

用 GitHub Issues 记录和管理开发计划，用 Orca 工作区看板跟踪执行状态，两者通过
issue 号关联，形成「计划 → 执行 → 交付 → 回顾」闭环。

## 分层体系

| 层级 | 模板 | label | 用途 |
|------|------|-------|------|
| Epic | `1_epic.md` | `epic` | 业务域 / 阶段计划 |
| Story | `2_story.md` | `story` | 用户故事 / 功能需求 |
| Task | `3_task.md` | `task` | 技术任务（含 Item Checklist） |
| Bug | `4_bug.md` | `bug` | 缺陷 / 故障 |

层级关系：**Epic → Story → Task**，通过 issue 正文里的 `#编号` 引用串联；Bug 独立，
修复后可关联到对应 Story/Task。

## 与 Orca 看板的联动

| 维度 | GitHub 侧 | Orca 侧 |
|------|----------|---------|
| 类型（做什么） | issue label（epic/story/task/bug） | worktree 卡片关联 issue 号 |
| 状态（做到哪步） | open / closed（PR 关闭） | `workspace-status` 看板列：`todo` / `in-progress` / `in-review` / `completed` |

两者通过 **issue 号** 串起来。

关键命令：

```bash
# 开 worktree 并关联 issue
orca worktree create --issue <N> --base-branch <ref> --agent <id> --prompt "..."

# 更新看板列
orca worktree set --worktree issue:<N> --workspace-status <id>

# 按 issue 反查 worktree
orca worktree show --worktree issue:<N>
```

## 完整流程（以 Task 为例）

```bash
# 1. 计划：GitHub 网页新建 issue（选 Task 模板，自动打 task label）→ 得到 #N
# 2. 执行：开 worktree 关联 issue 并派发 agent
orca worktree create --issue N --base-branch main --agent pi --prompt "..."
orca worktree set --worktree issue:N --workspace-status in-progress

# 3. 交付：完成后创建 PR，关联关闭 issue
gh pr create --base main --head <branch> --body "Closes #N"

# 4. 回顾：review + merge 后，卡片进完成列
orca worktree set --worktree issue:N --workspace-status completed
orca worktree rm --worktree issue:N --force   # 清理执行现场
```

## 层级协作节奏

1. **建 Epic**：规划阶段，列出 Story 范围与非目标。
2. **拆 Story**：为每个 Epic 建 Story issue，写明验收标准，`#` 引用 Epic。
3. **拆 Task**：为每个 Story 建 Task issue，`#` 引用 Story；Task 是 Orca worktree
   的最小执行单元，一个 Task 对应一个 worktree。
4. **记 Bug**：发现缺陷直接开 Bug issue，修复走「Task 流程」，PR 用 `Closes #N`
   同时关闭 Bug。

## 复用到其他项目

1. 复制模板目录：`cp -r .github/ISSUE_TEMPLATE <目标项目>/.github/`
2. 创建 labels：

   ```bash
   gh label create epic  --color 8B5CF6 --description "Epic 业务域/阶段计划"
   gh label create story --color 3B82F6 --description "Story 用户故事/功能需求"
   gh label create task  --color 10B981 --description "Task 技术任务"
   gh label create bug   --color EF4444 --description "Bug 缺陷/故障" --force
   ```

3. 按需调整 `config.yml` 里的 `contact_links` 链接。

## 注意事项

- issue 模板里的 `labels:` 引用的 label 必须先在仓库存在，否则 GitHub 会忽略该 label。
- `workspace-status` 默认列为 `todo` / `in-progress` / `in-review` / `completed`，
  自定义看板列请用对应配置的 id。
- Orca 的 `--issue` 只做关联记录，不负责创建/关闭 issue；创建与关闭用 `gh` 或网页。
- PR 正文写 `Closes #N`（或 `Fixes #N`），merge 后 issue 自动关闭，实现状态闭环。
