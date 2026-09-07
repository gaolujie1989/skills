---
name: superpowers-with-matt-tdd
description: 当结合 Superpowers 与 Matt 的 tdd 技能编写 implementation plan，或执行已批准的工作时使用。
---

# Superpowers 结合 Matt TDD

## 概述

保留 Superpowers 的 Spec -> Plan -> Execute -> Review -> Verify -> Finish 纪律，只调整实现规划与 TDD：

1. Implementation plan 使用更大的语义化 Task。
2. 执行阶段使用 Matt 的 `tdd` 策略。

凡是本技能没有明确覆盖的内容，均遵循原 Superpowers skill。

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

- 把 RED 与 GREEN 拆成独立 plan Task。
- 为每个内部函数规划测试，而不是聚焦有价值的 public behavior。
- 重复确认 plan 中已经批准的 test seams。
- 在实现任何行为之前，先写完所有测试。
- 替换执行器的 review 或 verification 流程，而不是只替换其 TDD 策略。

## 边界

本技能调整实现 Task 的粒度与 TDD 策略。它不改变 brainstorming 的提问节奏或设计批准行为，也不决定使用单 Agent 还是 subagent-driven development。

它不替代 writing-plans、executing-plans / subagent-driven-development、requesting-code-review、systematic-debugging、verification-before-completion 或 finishing-a-development-branch。除上述 override 外，其余部分均遵循原 Superpowers。

本技能可以独立使用，也可以与 `superpowers-with-checkpoint-brainstorming` 配合使用。
