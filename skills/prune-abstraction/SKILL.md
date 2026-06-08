---
name: prune-abstraction
description: "以高 refactor 预算校准代码 abstraction density：inline 装饰性 helper、wrapper、adapter、冗余 carrier 和临时变量，保留真正的 semantic boundary。当用户抱怨过度 abstraction、Clean Code 教条、helper 膨胀、过度 indirection、control flow 碎片化，或要求代码更直接、更 procedural、更接近 C 风格、更 compact、更高 density、更 inline、更榨干时使用。"
---

# 裁剪 Abstraction

## 核心原则

优化 semantic density，而不是追求最少行数。目标形态是一条 inline 的执行主线，加上少量真正承载语义重量的 helper。默认怀疑没有 reuse 的 helper。保留那些命名稳定 domain concept、保护 contract，或能显著降低 state pressure 的 abstraction。

这个 skill 用来校准 abstraction density；除非用户明确要求这个方向，否则它不替代代码库原本的常规风格。

## 挂钩工作流

1. 编辑前先阅读周边代码和测试。
2. 运行 scope hook 和 mainline hook。
3. Inline 装饰性 abstraction，并删除装饰性 data carrier。
4. 运行 structure hook、helper inventory hook、touched helper hook 和 smell hook。
5. 运行 readability rollback hook。
6. 运行 validation hook。
7. 最终回复前运行 exit record hook。

## Inline 与保留

当 abstraction 只是装饰性时，inline 它：

- 只调用一次、只转发参数的 wrapper。
- `_maybe_*`、`_commit_*`、`_resolve_*`、`_merge_*`、adapter 或 stage helper，如果它们只是在隐藏一个 `if`、一次直接调用，或一个平凡的对象构造。
- 函数体没有揭示新 concept 的 helper。
- 只是为了在本地 helper 层之间传值而引入的 data carrier。
- 冗余临时变量，例如 `result = f(); return result`。
- 只使用一次，又没有澄清 concept、缩短复杂表达式、避免重复计算、收窄类型或改善错误信息的 alias。

当 abstraction 确实降低 cognitive load 时，保留它：

- 它是真正被 reuse 的，而不只是同一次 refactor 中制造出来的 helper chain reuse。
- 它命名了稳定的 domain concept。
- 它保护了细微的 invariant、公共 contract、validation、protocol rule、parse rule、serialization shape 或 error structure。
- 它包含 algorithm logic，并且局部命名能降低读者的 state pressure。
- 它避免主流程同时追踪过多 live value。

## 结构检查

不要删掉明显 wrapper 后就停下。重新检查 abstraction topology：

- 删除那些只是在连接刚被删除 helper 的 carrier。
- Inline 仍然遮挡执行主线的剩余 wrapper。
- 如果一个 helper 只是喂给下一个 helper，就合并这对 helper。
- 确认主函数现在承担编排职责，而不是读起来像一张含糊的动词表。

优先使用 guard clause，而不是深层嵌套。只有在仍然显然易懂时才使用推导式和表达式。不要把条件、error contract 或多字段对象构造变成 expression puzzle。

## Hooks

Hooks 是强制检查点。如果某个 hook 没有通过，就继续编辑或 rollback，直到通过。Hooks 应用于用户请求的范围和你触碰过的文件；只有当调用点或 ownership boundary 要求时才扩大范围。保持 hook 通用：根据语言和代码库调整 private/helper 检测方式，不要硬编码文件名、模块名或 framework 假设。

### Loop Protocol

不要只运行一次 hook。单次检查本身就是 smell。使用循环：

1. 第一次编辑后至少运行两轮 prune loop：structure hook -> helper inventory hook -> touched helper hook -> smell hook；如果 loop 发现没有正当理由的 abstraction，就继续编辑。
2. 持续运行 prune loop，直到完整一轮没有发现缺少 survival reason 的 helper、carrier 或 smell。
3. Prune loop 后运行 readability rollback hook。如果 rollback 改变了 helper topology 或新增 boundary，再运行 prune loop。
4. 最终 loop 之后再运行 validation hook，不要在收敛前验证。
5. 最后运行 exit record hook。

即使是很小的编辑，也要对 touched area 做两次心智或可视检查。较大的 refactor 则持续循环，直到 helper inventory 稳定。

### Scope Hook

编辑前，明确 touched scope：请求实际覆盖的文件、模块、函数或组件。除非用户要求或 dependency graph 需要，否则不要扩展到整个仓库。

### Mainline Hook

编辑前，识别主执行线：maintainer 必须优先理解的顺序。Refactor 应该让这条主线更直接。

### Structure Hook

第一次编辑后，重新检查 abstraction topology：

- 只是连接已删除 helper 的 carrier。
- 仍然遮挡执行主线的 wrapper。
- 一个 helper 只是喂给下一个 helper 的 helper pair。
- 仍然读起来像含糊 dispatch table 的主函数。

Inline 或合并这个 hook 发现的任何装饰性层级。

### Helper Inventory Hook

在可用时用结构化工具列出 touched 文件里的 private/local helper；不可用时使用语言感知的搜索。根据语言调整模式：Python private function、TypeScript local function、private method、internal class、shell helper、test fixture helper，以及类似的 local abstraction 都算。

对编辑区域内或附近保留的每个 helper，至少验证一个 survival reason：

- 在装饰性 helper chain 之外被 reuse。
- 稳定 domain concept。
- invariant、contract、protocol、validation、parse、serialization 或 error structure。
- 显著降低 live state pressure。

如果都不成立，就 inline 它。

### Touched Helper Hook

每个 definition 或 call site 发生变化的 helper 都必须重新审计。不要因为它在 refactor 前就存在，就假设它仍然值得保留。

### Smell Hook

主动在 touched area 搜索 abstraction smell：

- 包装单个 expression 的一行 helper。
- 单调用 helper。
- 只守护一次调用的 `_maybe_*` helper。
- 只复制字段的 `_merge_*` helper。
- 只构造一个公共对象的 `_failure_*` 或 builder helper。
- `return bool(expr)` wrapper。
- `x = f(...); return x`。
- 只用一次、没有 type/error/concept 价值的 alias。

每个 smell 都必须被移除，或用 survival reason 明确证明。

### Readability Rollback Hook

Inline 后，检查相反方向的失败模式：

- Expression puzzle。
- 带有过多 live state 的巨大函数。
- 重复的细微 contract。
- 丢失的 domain vocabulary。
- 超过用户请求范围的编辑面。

只 rollback 那些确实降低 state pressure 或命名真实 concept 的 boundary。

### Validation Hook

优先用 focused test 验证。当实现触碰 shared behavior、公共 contract、import 或跨模块流程时，再运行更广的检查。

### Exit Record Hook

结束前，准备好说明：

- 哪些 helper/carrier 被删除或 inline。
- Touched area 里还剩哪些 helper。
- 每个值得一提的剩余 helper 为什么存活。

这份记录是自检；除非用户要求完整清单，否则面向用户的最终回复保持简洁。

## 退出条件

结果必须至少满足一项：

- 主执行线明显更直接。
- 继续 inline 会破坏已命名的 semantic boundary、重复细微 contract，或制造 state pressure/expression puzzle 成本。

只删除几个 helper，却保留原来的 abstraction topology，是不够的。
