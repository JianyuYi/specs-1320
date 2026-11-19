---
status: approved
---

# 工作流派发前拒绝缺失的已选 MCP

工作流运行在开始调度前校验冻结图里显式启用的 MCP 是否仍以完整插件身份安装；缺失时拒绝 `start`、`restart` 和会切换快照的 `resume`，不进入节点调度。只有“插件已不存在”提前到运行门禁；配置不完整、Agent 能力不足等仍按既有约定在 session setup 时让节点失败。

本决策修改 `20260912-workflow-explicit-selection.md` 的 D1 与 D4，把“已选插件缺失”的失败边界从节点创建/运行时提前到运行派发前。它不改变显式选择、Session 持久化、ACP 发送和 Skill/角色物化语义。落地由 desktop#790 承接；合并后本 ADR 转为 `implemented`。

## 继承与修改

| 前序约定 | 处理 | 当前规则 |
| --- | --- | --- |
| D1：工作流使用冻结节点的显式 MCP 集合 | 修改 | 显式集合不变；集合中已选但未安装的插件在 start/restart/resume 派发前被拒绝 |
| D2：Session 自身持久化选择，恢复 fail closed | 继承，不变 | 预检只读取冻结图与已安装插件树，不改写 Session 选择 |
| D3：ACP 发送边界记录 secret-free 身份 | 继承，不变 | 预检错误同样只带节点与 MCP 身份，不带 Setting 实值 |
| D4：安装/授权/Skill 语义分离 | 修改 | 安装仍不等于授权；缺失已选插件从“运行时节点失败”改为“派发前拒绝运行”，配置不完整仍留给 session setup |
| 不变量 1–6 | 继承，不变 | 普通聊天自动选择、空集合不回退、草稿不影响已有运行等保持 |

## 决策

### D1：派发前校验目标图的已选 MCP

`startWorkflowRun`、`restartWorkflowRun` 和保留当前快照的 `resumeWorkflowRunFromFailure` 在派发第一个节点之前，用与创建运行相同的 plugin-aware 解析器读取冻结图，重新校验所有 `enabled: true` 的 MCP 是否仍是已安装的完整插件。缺失时返回 `workflow_mcp_not_found`，参数包含节点 id、节点标题和 MCP id。

门禁只在这些命令真的会改变状态时执行：`Pending` 的 start、非 `Running` 的 restart、`Failed`/`Cancelled` 的 resume。对已运行运行的重复 start、对 `Running` 的 restart 仍按原语义幂等返回，不能把空操作变成依赖错误。

### D2：只提前“缺失安装”，不扩大配置校验

预检只判断插件是否存在，不判断配置是否完整、MCP 是否能启动或 Agent 是否支持该 transport。配置不完整仍是 session setup 的节点失败；Agent 能力不足仍由 runtime 在打开节点会话时 fail closed。这样既能让用户在大范围运行开始前发现依赖已被卸载，又不会把配置错误误报成“重新安装插件即可”。

### D3：切换快照的 resume 校验目标图，且在 worktree 改动之前

切换快照的 resume 与保留当前快照的 resume 一样，必须在任何 worktree 改动之前读取**目标快照**的图并做同样的 MCP 校验。缺失时运行保持 `Cancelled`/原状态，worktree 不被改写，错误仍然点名目标图中的节点和 MCP。

### D4：错误可诊断且不泄露配置

`workflow_mcp_not_found` 只包含节点 id、节点标题（缺省回落为 id）和 MCP id。错误不包含 MCP 的 command、arguments、env、URL、headers 或其他 Setting 实值；预检只读，不创建 Session、不发送 ACP 请求。

## 不变量

1. 缺失的已选 MCP 在派发第一个节点前被拒绝，运行不进入 `Running`。
2. 幂等 start/restart 与“运行时才发现的配置错误”不被预检改写。
3. 切换快照的预检读取目标图，且在 worktree 改动之前完成。
4. 预检只读：不创建 Session、不发送 ACP、不改 Session 选择或 worktree。
5. 错误只暴露节点与依赖身份，不暴露配置实值。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
| --- | --- |
| 保持运行时节点失败 | 用户在大范围运行开始后才发现依赖已被卸载，其他节点已经执行，无法整体回滚 |
| 在预检里同时校验配置完整性和 Agent 能力 | 配置不完整与“插件缺失”的恢复动作不同；提前拒绝会把“去修复配置”误导成“重装插件” |
| 卸载时硬阻止 | 是否允许卸载仍由产品决定；本决策只保证下一次派发前拒绝，不改变卸载流程 |
| 每次节点调度都重新校验 | 节点一旦开始就持有启动时的冻结回执，重复校验会扩大范围并影响进行中的运行 |

## 未解决

- resume 时依赖错误先于 `workflow_run_not_resumable` 返回：用户重装插件后才发现运行不可恢复，后续可先做只读 rollback plan。
- 创建运行时缺 Skill 会被拒、缺 MCP 仍可创建：本决策只覆盖 start/restart/resume，创建时的 MCP 不对称留给后续。
- 同名本地 Skill 遮住已卸载的插件 Skill：属于 Skill 匹配规则，不在本决策范围。

## 核心测试依据

见 `specs/test-cases/desktop/core/agent/session/mcp/workflow-mcp-preflight.md`。
