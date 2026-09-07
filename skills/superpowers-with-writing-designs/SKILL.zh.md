---
name: superpowers-with-writing-designs
description: 仅在用户明确调用、需要在编写 implementation plan 之前创建 Detailed Design 时使用。
---

# Superpowers with Writing Designs

> `SKILL.md` 是唯一规范源。`SKILL.zh.md` 是供人工阅读的同步译本，不得定义独立行为。

## 概述

在 Superpowers Spec 与 `superpowers:writing-plans` 之间增加一个可选的 Detailed Design 阶段：

```text
Spec → Detailed Design → 人工 review / approval → Plan → Implementation
```

本 Skill 扩展 Superpowers，但不替代或覆盖任何现有 Superpowers Skill。

Detailed Design 在生成大量代码前解决 implementation structure 决策，降低人工 review 成本并减少 implementation drift。

## 信息来源与边界

开始设计前，读取相关 Spec、implementation、class、function、interface、architecture、repository pattern、`AGENTS.md` 和项目规范。

Spec 是 requirements 和 architecture 的 source of truth。应引用 Spec，而不是重复其内容。不得静默修改或虚构 requirement；如果 Spec、现有代码和合理 implementation structure 之间存在冲突，应明确指出。

Detailed Design 回答：

> 代码在结构上应该是什么样？

如果一个决策的变化会显著影响 responsibility、public API、data flow、control flow、transaction、concurrency、side-effect ownership、cross-component interface 或 testability，则应纳入 Detailed Design。

以下内容留给 Plan 或 Implementation：

- implementation order、TDD step 和 commit structure
- 完整 function body 和机械性 CRUD
- local variable、普通 data transformation 和等价 ORM expression
- 不改变已确认结构边界的 leaf helper 和 private detail

目标是锁定重要结构，而不是把代码写两遍。

## 必须包含的设计内容

根据 use case 调整文档规模。只包含适用 section，但必须明确所有 design-significant decision。

| 范畴 | 需要确定的内容 |
| --- | --- |
| Component responsibility | 每个重要 component 负责和不负责什么，以及其 dependency 和 boundary |
| Public interface | 准确的 method name、parameter、parameter type 和 return type |
| Key internal stage | 仅定义代表 business stage、responsibility 或稳定 abstraction boundary 的 private/internal method |
| Method contract | Input、return、responsibility、business rule、error、side effect，以及必要时明确不负责的内容 |
| Call flow 与 data flow | 谁发起 use case、谁负责 validation、谁修改 state，以及主要 stage 如何协作 |
| Transaction 与 concurrency | correctness 依赖的 transaction boundary、locking、idempotency 和 concurrency assumption |
| Complex logic | 仅为非平凡的 state transition、reconciliation、allocation、synchronization、batching、retry 或 dependency ordering 编写 pseudocode |
| Test seam | 可验证 behavior 的重要 public 或 component boundary |

包含 orchestration 的 Service，其 public method 应体现高层 business stage。不要仅为减少行数而拆分 helper。

示例：

```python
class PurchaseOrderService:
    @classmethod
    def update(
        cls,
        *,
        order_id: int,
        data: PurchaseOrderUpdateData,
        operator: User,
    ) -> PurchaseOrder:
        ...
```

```text
update()
├── _get_for_update()
├── _validate_update()
├── _apply_order_fields()
├── _sync_items()
└── _record_audit()
```

对于 `_sync_items()` 这类重要 stage，应简洁说明其 contract：

```text
Responsibilities:
- 根据完整 incoming item collection 进行 synchronization
- 按需 create、update 和 remove item

Does not:
- validate order-status permission
- modify order header
- trigger inventory side effect
```

默认使用简洁 call tree。只有当 ordering、branching 或 cross-component interaction 无法清晰表达时才使用 sequence diagram。

## 设计深度

如果 implementer 仍需决定重要 component boundary、public method、parameter shape、主要 stage method、call ownership、transaction boundary 或 side-effect ownership，说明设计太浅。

如果设计规定了每个 helper、loop、ORM expression、local variable 或近乎完整的 function body，说明设计太细。

## 输出

将设计保存到：

```text
docs/superpowers/designs/YYYY-MM-DD-<feature>-design.md
```

按需使用以下结构，不要创建空 section：

```markdown
# <Feature> Detailed Design

**Spec:** `<spec-path>`

## Design Goal

## Component Overview

## <Component>

### Responsibility
### Public Interface
### Key Internal Methods
### Method Contracts
### Call Flow
### Transaction / Concurrency

## Cross-Component Flow
## Complex Logic
## Implementation Freedom
## Design Decisions
## Open Design Conflicts
```

优先使用 signature 而不是 implementation code，contract 而不是长篇 prose，call tree 而不是冗长 explanation，selective pseudocode 而不是完整 pseudocode。

## Design Contract 与 Drift

用户批准 Detailed Design 后，它成为 implementation structure 的 source of truth。Plan 可以为 task-local context 复制准确定义，但不得重新设计。

如果 Planning 或 Implementation 暴露出重大问题，停止受影响的工作并报告：

```text
Design Conflict

Current design:
<approved design>

Problem:
<why it is problematic>

Proposed change:
<recommended adjustment>

Impact:
<affected components, methods, tasks, and interfaces>
```

如果变更涉及 responsibility、interface、major flow、transaction、side effect 或 cross-component contract，应更新 Detailed Design 并获得用户批准，再继续受影响的工作。

## Self-Review

提交设计前，确认：

- 与 Spec 一致，没有静默增加 requirement
- 重要 responsibility、interface、signature、flow 和 ownership 明确且一致
- correctness 所需的 transaction、locking、idempotency 和 complex logic 已定义
- internal method 表达有意义的 stage，而不是机械拆分
- 非结构性选择仍保留 implementation freedom
- 文档既不是 implementation plan，也不是 near-code
- 人工 review 明显比 review 最终 implementation 更便宜

## Approval 与 Handoff

完成 Detailed Design 后停止，并提交用户 review。未经明确批准，不得开始 Plan 或 Implementation。

如果用户要求修改，更新设计并重新检查受影响的 signature、flow、dependency 和 contract，再次提交 revision。

批准后，使用 `superpowers:writing-plans`。Plan 必须同时读取 Spec 和 Detailed Design：

- Spec：requirements 和 architecture source of truth
- Detailed Design：implementation-structure source of truth

然后返回正常 Superpowers workflow。
