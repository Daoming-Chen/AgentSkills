# AgentSkills

一组可复用的 AI Agent 技能（Skills），用于增强 AI 编码助手的能力。

## 项目结构

```
skills/
├── prune-abstraction/   # 抽象修枝，重构过度间接的代码
│   ├── SKILL.md         # 技能定义与使用说明
│   └── agents/
│       └── openai.yaml  # OpenAI Codex 接口配置
└── ask-claude/          # 将 Claude Code CLI 作为外部子代理调用
    ├── SKILL.md         # 技能定义与使用说明
    └── agents/
        └── openai.yaml  # OpenAI Codex 接口配置
```

## 已有技能

### prune-abstraction

抽象修枝：移除只装饰流程的 helper 和过多间接层，同时保留表达领域概念、契约、不变量或错误形状的语义边界。

适用场景：
- 用户抱怨代码过度抽象、Clean Code 味太重、helper 函数森林
- 希望代码更直接、更过程式、更紧凑，或更容易不跳转阅读
- 需要把碎片化控制流整理成可顺读的主执行路径

### ask-claude

将用户请求转发给 Claude Code CLI，把 Claude 作为 Codex 编排的外部子代理使用。

适用场景：
- 问 Claude 问题或让 Claude 看代码/文件/日志/diff
- 运行只读验证、写不落盘探针
- 代码 review、风险评估、架构建议
- Codex 与 Claude 有限讨论

前置条件：本机需安装并登录 `claude` CLI。

## 添加新技能

在 `skills/` 下创建新目录，包含：
- `SKILL.md` — 技能定义（名称、描述、触发条件、执行流程）
- `agents/` — 各 Agent 平台的接口配置（如 `openai.yaml`）

## License

MIT
