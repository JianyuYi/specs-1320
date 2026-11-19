# 工作流派发前 MCP 预检

## Workflow Mcp Preflight Must Reject Missing Selections Before Dispatch

### 风险

如果运行在已选 MCP 插件被卸载后仍能进入 `Running`，用户会在大范围节点执行后才发现依赖缺失，其他节点已经产生副作用；如果预检同时校验配置或 Agent 能力，又会把可修复的配置错误误报为“插件缺失”，改变既有失败边界。

### 前置状态

- 运行已冻结，图中有 `enabled: true` 的 MCP 绑定。
- 对应插件已卸载或从未安装。
- 运行分别为 `Pending`、`Cancelled`/`Failed`（可 resume）或 `Running`（幂等命令）。

### 触发

调用 `startWorkflowRun`、`restartWorkflowRun` 或 `resumeWorkflowRunFromFailure`。

### 必须成立

- 缺失已选 MCP 时，三个命令都在派发第一个节点前失败，返回 `workflow_mcp_not_found`，错误带节点 id、节点标题和 MCP id。
- 运行保持原状态（`Pending` 的 start 不进入 `Running`，可 resume 的保持 `Cancelled`/`Failed`）。
- 对已运行运行的重复 start、对 `Running` 的 restart 仍幂等返回，不返回依赖错误。
- 预检只读：不创建 Session、不发送 ACP 请求、不改 Session 选择或 worktree。
- 配置不完整、Agent 不支持该 transport 等仍留给 session setup 的节点失败，不被预检提前拒绝。

### 禁止结果

- 缺失已选 MCP 的运行进入 `Running` 并派发节点。
- 幂等 start/restart 被误判为依赖错误。
- 预检改写 worktree、Session 选择或产生 ACP 流量。
- 错误包含 command、env、URL、headers 或 Setting 实值。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 卸载已选 MCP 后 start 被拒绝且运行保持 `Pending` | `ora_backend::workflow::run::operations::precheck_tests::a_missing_mcp_plugin_blocks_start`（desktop#790） | `Partial` |
| 同一检查覆盖 restart 与保留当前快照的 resume | `a_removed_skill_plugin_blocks_start_and_restart`、`a_removed_skill_plugin_blocks_a_resume`（desktop#790；当前为 Skill 路径，MCP 参数形状由 `a_missing_mcp_plugin_blocks_start` 覆盖） | `Partial` |
| 幂等 start/restart 不被依赖错误改写 | `an_idempotent_start_is_not_turned_into_a_dependency_error`（desktop#790） | `Partial` |
| 预检只读，不产生 Session/ACP/worktree 副作用 | 尚无直接断言 | `Missing` |
| 错误不包含 Setting 实值 | 契约参数只含节点与 MCP id，尚无专门 secret 测试 | `Missing` |

### 决策依据

- `20261119-workflow-mcp-preflight.md` D1–D4。
- 该决策不变量 1–5。

## Snapshot Switch Resume Must Validate The Target Graph Before Rollback

### 风险

切换快照的 resume 如果只校验当前分支，或先改 worktree 再校验目标图，会在目标快照缺少 MCP 时留下被改写的 worktree，并让运行状态与错误不一致。

### 前置状态

- 运行处于可 resume 状态，当前快照与目标快照的 MCP 绑定不同。
- 目标快照包含未安装的已选 MCP。

### 触发

调用 `resumeWorkflowRunFromFailure` 并指定 `snapshot_id`，走快照切换分支。

### 必须成立

- 在任何 worktree 改动之前读取目标快照的图并校验 MCP。
- 缺失时返回 `workflow_mcp_not_found`，运行保持 `Cancelled`/原状态。
- worktree 保持调用前的状态，不被 checkout、reset 或改写。

### 禁止结果

- 先把 worktree 切到目标快照，再报依赖缺失。
- 只校验当前快照，漏掉目标图的缺失 MCP。
- 运行进入 `Running` 后在 session setup 才失败。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 切换快照 resume 在 worktree 改动前校验目标图，并拒绝缺失 MCP | 尚无；desktop#790 待补 | `Missing` |

### 决策依据

- `20261119-workflow-mcp-preflight.md` D3。
- 该决策不变量 3–4。
