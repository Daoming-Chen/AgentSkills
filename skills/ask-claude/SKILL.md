---
name: ask-claude
description: "将用户明确提出的请求转发给 Claude Code CLI，把 Claude 作为 Codex 编排的外部子代理使用。适用于用户要求问 Claude、让 Claude 看代码/文件/日志/diff、运行验证、写不落盘探针、做 review/风险评估/架构建议，或让 Codex 和 Claude 有限讨论。"
---

# Ask Claude

仅当用户明确要求“问 Claude”“让 Claude 看一下”“让 Claude review”“让 Codex 和 Claude 讨论”时触发。

Claude 是 **Codex 编排的外部子代理**。Claude 可以独立读取、分析、验证和提出建议；Codex 负责上下文选择、安全约束、结果综合，以及最终是否执行任何持久修改。

## 调用命令

默认使用 Claude Opus 4.7，max effort，bypass permission：

```bash
claude -p --model claude-opus-4-7 --effort max --permission-mode bypassPermissions "<prompt>"
```

- `-p`：非交互 print 模式。
- `--model claude-opus-4-7`：指定 Claude Opus 4.7。
- `--effort max`：使用最高 effort。
- `--permission-mode bypassPermissions`：绕过权限确认；即使 bypass，prompt 仍必须明确限制持久写入。

## 前置条件

- 本机必须安装并登录 `claude` CLI。
- 不要向 Claude 传递密钥、凭证、`.env` 内容、生产数据、敏感客户数据，或用户没有明确授权转发的隐私信息。
- 只传递 Claude 完成任务所需的最小上下文。
- 如果仓库中存在 `docs/cross-agent-review.md`，且任务涉及跨代理审查、实现建议或验证，先读取并遵守其中约定。

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
4. 如任务涉及当前仓库，调用 Claude 前后各执行一次 `git status --short`；如果状态变化，立即报告用户并检查原因。
5. 返回 Claude 的关键结论、证据、运行过的命令、Codex 的综合判断，以及建议下一步。

## 模式

| 模式 | 触发场景 | Claude 可做什么 | 输出重点 |
| --- | --- | --- | --- |
| 普通问答 | 用户只想问 Claude 一个问题 | 不读工作区，不跑命令 | 回答、假设、风险、下一步 |
| 工作区读取 | 用户要 Claude 看代码、文件、日志、diff、PRD | 读文件、搜索、查看 git 只读信息 | 结论、证据、关键文件、风险 |
| 探针验证 | 用户要 Claude 复现、跑测试、验证假设 | 运行测试/诊断命令，写不落盘探针 | 结论、步骤、命令输出、未验证风险 |
| 审查 | 用户要 Claude review 计划、代码、架构、diff | 读取材料并做只读检查 | 同意点、不同意点、风险、建议修改、追问 |
| 有限讨论 | 用户明确要求 Codex 和 Claude 来回讨论 | 每轮聚焦一个剩余分歧 | 一致点、分歧、用户决策、下一步 |

## Prompt 构造

使用一个基础 prompt，再按模式补充具体要求：

```text
你是 Claude Opus 4.7，正在以 max effort 作为 Codex 编排的外部子代理工作。

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
- 每一轮都要更短、更聚焦。
- 讨论中不做持久修改。
- 当剩余分歧属于产品判断、用户偏好或低影响实现细节时，停止并交给用户。
- 当前 Codex 会话负责循环编排，不要求 Claude 反向调用 Codex。

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

如果 `claude` 未安装、未登录、认证失败、组织策略限制、调用超时、模型不可用或输出不可用，明确告诉用户 Claude 没有成功参与，并由 Codex 本地给出分析。不要假装 Claude 已回答或审查。

## 输出规范

- 区分 Claude 的原始观点和 Codex 的综合判断。
- 总结 Claude 运行过的命令和关键输出。
- 明确列出 Claude 结论依赖的假设。
- 如果 Claude 违反工作区约束或造成文件变化，优先报告。
