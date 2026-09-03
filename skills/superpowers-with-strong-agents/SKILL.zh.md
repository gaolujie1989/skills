---
name: superpowers-with-strong-agents
description: 当使用 Superpowers 进行 brainstorming、编写 implementation plan，或由强 coding agent 执行已确认计划时使用。
---

# 面向强 Agent 调整 Superpowers

## 概述

保留 Superpowers 的 Spec -> Plan -> Execute -> Review -> Verify -> Finish 纪律，只针对强 coding agent 调整三点：

1. Brainstorming 使用自主决策与 checkpoint supervision。
2. Implementation plan 使用更大的语义化 Task。
3. 执行阶段使用 Matt 的 `tdd` 策略。

目标不是取消人工监督，而是把人的注意力从常规实现选择转移到高价值、由用户拥有的决策。凡是本技能没有明确覆盖的内容，均遵循原 Superpowers skill。

## Brainstorming：优先使用 Checkpoint Supervision

**REQUIRED SUB-SKILL：** 使用 `superpowers:brainstorming`。

保留其范围分类、设计质量、书面 spec 要求，以及实现前的最终批准 gate。只按下文覆盖它的提问节奏和逐节批准行为。

### 核心原则

**Recommendation is a decision, not a question.（推荐即决策，而不是问题。）**

当某个选项明显更好时，直接采用、记录并继续。不要先给出推荐项，再要求用户选择同一个推荐项。

### 决策策略

依据可观察条件对每个未决选择分类：

| 类型 | 条件 | 行为 |
| --- | --- | --- |
| `AUTO` | 根据既有约束、惯例、YAGNI/KISS 或工程实践，存在明显最佳选项；该选择属于实现细节、可逆，且不改变需求语义。 | 直接选择，记录决策与理由，然后继续。 |
| `ASSUME` | 信息缺失，但存在安全、常规、可逆的默认值。 | 采用默认值，记录假设及影响，然后继续。 |
| `ASK` | 没有明显更优的选项，或选择取决于产品意图、业务语义、范围、兼容性、迁移或删除行为、安全、权限、显著成本、需求冲突，或只有用户掌握的信息。 | 记录下来，留到下一次 Decision Checkpoint。 |

仅仅存在多个实现选项，并不意味着该决策属于 `ASK`。

### 批量澄清

覆盖 brainstorming 默认的“一次只问一个问题”。

在累积相关 `ASK` 决策的同时，继续所有未被阻塞的设计工作。当未决事项阻塞后续设计时，在一个简洁的 Decision Checkpoint 中集中提出。优先进行一次有意义的澄清，而不是连续多次小打断。

### Decision Checkpoint

在最终确定架构型 spec 之前，汇总：

- Agent 自主做出的推荐决策；
- Agent 自主采用的假设；
- 用户已经做出的决策；
- 仍需要用户输入的 `ASK` 决策。

使用一份紧凑的 ledger，例如：

| ID | 主题 | 选择 | 来源 | 理由 / 影响 |
| --- | --- | --- | --- | --- |
| D1 | 异步处理 | Celery | Agent recommendation | 符合现有基础设施 |
| A1 | 分页大小 | 50，可配置 | Agent assumption | 低影响、可逆默认值 |
| D2 | 取消时的库存处理 | 立即恢复 | User decision | 必需的业务行为 |

如果没有剩余 `ASK` 决策，说明不存在阻塞性的用户决策，然后直接编写 spec；不要再请求一次中间确认。如果仍有 `ASK` 决策，应一次性提出，合并用户回答后再编写 spec。

### 设计与 Spec 批准

对于 architectural path，覆盖逐个设计章节审批的规则。把架构、组件、数据流、错误处理和测试发展成一份连贯设计，不要仅仅为了确认而在每一节后停下来。

完整 spec 必须包含简洁的 `Decisions` 章节，记录对实现有实质约束的选择。Spec 本身是持久决策记录；除非项目明确要求，否则不要另建一份长期维护的 decision document。

在 planning 前，把完整 spec 提交用户做一次最终 review。尽可能一次性合并用户要求的修改，重新检查 spec，并保留实现前必须最终批准的 gate。

对于 bounded work，同样应用 `AUTO` / `ASSUME` / `ASK` 策略和批量澄清，但仍保留 `superpowers:brainstorming` 要求的简短 chat 内设计及批准 gate。

## Writing Plans：保持内容详细，放大 Task

**REQUIRED SUB-SKILL：** 使用 `superpowers:writing-plans`。

不要机械拆成 2-5 分钟的动作。每个 Task 应是语义完整、可验证、适合作为一次 commit 的实现增量。强 coding agent 可以在一个 Task 内连续完成多个自然相关的步骤。

Plan 仍需明确目标与约束、涉及文件、接口与依赖、关键实现要求、验证方式和 commit 边界。

不要仅仅因为 TDD 而把以下过程拆成独立 Task：

- 编写 failing test；
- 验证 RED；
- 编写最小实现；
- 验证 GREEN。

这些是同一个实现 Task 的内部步骤。5-15 分钟仅作为粗略尺度；语义、依赖、验证和 commit 边界优先于时间。

### Test Seams

对需要 TDD 的 Task，标出有价值的 test seams：通过哪些 public interface 验证哪些行为。不要为每个内部函数或实现细节规划测试。

用户批准整个 plan，即视为同时批准其中列出的 test seams。

## Execute：使用 Matt TDD

无论执行选择 `superpowers:executing-plans` 还是 `superpowers:subagent-driven-development`，都保留该执行器自身的流程。

**REQUIRED SUB-SKILL：** 使用 `tdd`，在实现 Task 中替代 `superpowers:test-driven-development`。

在每个 Task 内：

- 测试 public behavior，而不是 implementation details；
- 只在已批准的 seam 上测试；
- 使用 vertical slices：一个行为测试 -> 最小实现 -> 下一个行为；
- RED 先于 GREEN；
- 不要仅为覆盖率而重复测试 private helper、内部协作者或简单层；
- 把更大范围的 refactor 留到 review，不要扩大 RED -> GREEN 循环。

不要重复确认 plan 中已经批准的 test seams。如果实现必须引入已批准 plan 中没有的重要新 seam，应在添加前确认。

## 常见错误

- 要求用户选择 Agent 已经明确推荐的选项。
- 把每个缺失细节都当成 `ASK`，而不采用安全、可逆的假设。
- 把多个相关业务问题拆到多个回合询问。
- 因为取消了中间确认，就跳过最终 spec 批准。
- 把 RED 与 GREEN 拆成独立 plan Task。
- 替换执行器的 review 或 verification 流程，而不是只替换其 TDD 策略。

## 边界

本技能不决定使用单 Agent 还是 subagent-driven development。它不替代 brainstorming、writing-plans、executing-plans / subagent-driven-development、requesting-code-review、systematic-debugging、verification-before-completion 或 finishing-a-development-branch。

除上述三项 override 外，其余部分均遵循原 Superpowers。
