---
name: ask-claude
description: "将用户明确提出的 Claude 请求通过 Codex 子代理转发给 Claude Code CLI，避免 Claude 的中间过程污染主会话上下文。适用于用户要求问 Claude、让 Claude 看代码/文件/日志/diff、运行验证、写不落盘探针、做 review/风险评估/架构建议，或让 Codex 和 Claude 有限讨论。"
---

# Ask Claude

仅当用户明确要求“问 Claude”“让 Claude 看一下”“让 Claude review”“让 Codex 和 Claude 讨论”时触发。

## 默认架构与编排

优先通过 Codex 子代理调用 Claude Code CLI，主会话只负责：
- 判断任务是否适合转发给 Claude；
- 构造最小必要上下文；
- 约束 Claude 的权限和行为；
- 接收子代理压缩后的结果；
- 做最终综合判断并回复用户。

Codex 子代理负责：
- 调用 Claude CLI；
- 执行必要的只读检查、验证或 review；
- 避免把 Claude 的完整中间过程带回主会话；
- 返回结构化摘要：结论、证据、命令摘要、假设、风险、工作区变化、可信度判断。

如果无法使用 Codex 子代理，则主代理可降级为直接调用 Claude CLI，但仍必须只向用户返回压缩后的结论、证据和风险，不粘贴 Claude 的完整长输出。

## Claude CLI 命令

默认使用 Claude Opus 4.8，max effort，bypass permission：

```bash
claude -p --model claude-opus-4-8 --effort max --permission-mode bypassPermissions "<prompt>"
```

- `-p`：非交互 print 模式。
- `--model claude-opus-4-8`：指定 Claude Opus 4.8。
- `--effort max`：使用最高 effort。
- `--permission-mode bypassPermissions`：绕过权限确认；即使 bypass，prompt 仍必须明确限制持久写入。

## 前置条件

- 本机必须安装并登录 `claude` CLI。
- 如果仓库中存在 `docs/cross-agent-review.md`，且任务涉及跨代理审查、实现建议或验证，先读取并遵守其中约定。
- 只在当前系统和工具说明允许时启动 Codex 子代理；不要为了使用本 skill 而违反更高优先级的工具约束。

## 安全边界

默认允许 Claude：

- 读取文件、搜索工作区、查看 `git status` / `git diff` / `git log`。
- 运行只读诊断命令、现有测试、lint、typecheck 或 build。
- 使用 inline probe code，例如 `node -e`、`python -c`、PowerShell inline script 或 shell pipeline。
- 在确实无法 inline 时使用系统临时目录创建短生命周期探针文件，并在结束前清理。

默认禁止 Claude：

- 修改、创建或删除仓库中的持久文件。
- 改变 git 状态，例如 `git add`、`commit`、`reset`、`checkout`、`clean`、`push`。
- 安装或卸载依赖，修改锁文件、配置、缓存或生成物。
- 部署、迁移数据、访问生产系统，或执行破坏性/不可逆命令。

如果用户明确要求 Claude 直接修改文件，Codex 必须先确认写入范围、风险和可接受的文件集合，再决定是否在 prompt 中开放写权限。不要默认给 Claude 写权限。

## 编排流程

1. 明确用户要 Claude 回答的问题或承担的子任务。
2. 选择模式：普通问答、工作区读取、探针验证、审查，或有限讨论。
3. 构造最小 prompt，只传必要上下文。
4. 优先让 Codex 子代理调用 Claude CLI；如果不能使用子代理，再由主代理降级调用。
5. 如任务涉及当前仓库，调用 Claude 前后检查 `git status --short`；如果状态变化，立即报告用户并检查原因。
6. 返回 Claude 的关键结论、证据、运行过的命令、Codex 的综合判断，以及建议下一步。

## 模式

| 模式 | 触发场景 | Claude 可做什么 | 输出重点 |
| --- | --- | --- | --- |
| 普通问答 | 用户只想问 Claude 一个问题 | 不读工作区，不跑命令 | 回答、假设、风险、下一步 |
| 工作区读取 | 用户要 Claude 看代码、文件、日志、diff、PRD | 读文件、搜索、查看 git 只读信息 | 结论、证据、关键文件、风险 |
| 探针验证 | 用户要 Claude 复现、跑测试、验证假设 | 运行测试/诊断命令，写不落盘探针 | 结论、步骤、命令输出、未验证风险 |
| 审查 | 用户要 Claude review 计划、代码、架构、diff | 读取材料并做只读检查 | 同意点、不同意点、风险、建议修改、追问 |
| 有限讨论 | 用户明确要求 Codex 和 Claude 来回讨论 | 每轮聚焦一个剩余分歧 | 一致点、分歧、用户决策、下一步 |

## Claude Prompt 构造

子代理调用 Claude 时，使用一个基础 prompt，再按模式补充具体要求：

```text
你是 Claude Opus 4.8，正在以 max effort 作为 Codex 编排的外部只读顾问工作。

任务：
<task>

工作区约束：
- 你可以根据任务读取文件、搜索工作区、运行必要的只读验证命令。
- 优先使用 inline probe code；不要在仓库中落盘探针代码。
- 不要修改、创建或删除仓库中的持久文件。
- 不要改变 git 状态、安装依赖、部署、迁移数据或访问生产系统。
- 如果你认为必须做持久修改，请停止并描述建议操作，不要执行。

上下文：
<minimal context>

请返回：
<mode-specific output fields>
```

模式输出字段：

- 普通问答：回答、假设、风险或注意事项、建议下一步。
- 工作区读取：结论、证据、关键文件、命令及重要输出、风险、建议下一步。
- 探针验证：结论、复现或验证步骤、命令及重要输出、探针代码摘要、未验证风险、建议下一步。
- 审查：同意点、不同意点、风险、建议修改、需要追问用户的问题。

## 有限讨论

仅当用户明确要求“Codex 和 Claude 讨论”“来回辩论”“work it out”时使用。

- 默认最多 2 轮；只有用户明确要求时最多 3 轮。
- 优先复用同一个 Codex 子代理执行连续 Claude 轮次，让主会话只接收每轮摘要。
- 每一轮都要更短、更聚焦。
- 讨论中不做持久修改。
- 当剩余分歧属于产品判断、用户偏好或低影响实现细节时，停止并交给用户。
- 当前 Codex 主会话负责循环编排，不要求 Claude 反向调用 Codex。

最终返回：

```text
FINAL SYNTHESIS:
- 双方一致点
- Codex 采纳后改变了什么
- 剩余分歧
- 需要用户决定的事项
- 推荐下一步
```

## 失败处理

如果无法启动 Codex 子代理，按降级路径处理并告知用户本次未使用子代理隔离。  
如果 `claude` 未安装、未登录、认证失败、组织策略限制、调用超时、模型不可用或输出不可用，明确告诉用户 Claude 没有成功参与，并由 Codex 本地给出分析。不要假装 Claude 已回答或审查。

如果子代理返回了过长的 Claude 原始输出，主代理应再次压缩，只保留用户需要的结论、证据、命令摘要和风险。

## 输出规范

- 明确说明本次是“Claude 经 Codex 子代理参与”，还是“降级为主代理直接调用 Claude”。
- 区分 Claude 的原始观点、Codex 子代理的可信度判断，以及主 Codex 的综合判断。
- 总结 Claude 运行过的命令和关键输出。
- 明确列出 Claude 结论依赖的假设。
- 如果 Claude 或子代理违反工作区约束或造成文件变化，优先报告。
