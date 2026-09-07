---
name: studying-large-ai-changes
description: 当现有 implementation 已基本完成，需要在详细 review 之前理解大规模或陌生代码，尤其是 AI 生成的修改过大、不适合线性阅读时使用。
---

# Studying Large AI Changes

> `SKILL.md` 是唯一规范源。`SKILL.zh.md` 仅供人工阅读的同步译本，不得定义独立行为。

## 概述

本 Skill 用于 implementation 基本完成之后、进入详细 Code Review 之前。

目标不是解释每一行修改、总结每个文件，也不是比较修改前后的代码。目标是：**把大型 implementation 压缩成一个 mental model，让有经验的开发者无需线性阅读全部代码，就能理解当前代码的设计、implementation structure、主要 call flow、关键 function、state/data movement 和 review boundary。**

优化目标是 **comprehension，而不是 coverage**。

```text
Large implementation
        ↓
Implementation reconstruction
        ↓
Mental model
        ↓
Component / use-case / function structure
        ↓
Targeted human reading
```

这是一个 **implementation comprehension / reconstruction Skill**，不是 Code Review Skill，也不是 change-summary Skill。

## Source of Truth 与范围

**当前已经实现完成的代码**是 source of truth。

diff、commit range、branch、task file 或 changed-file list 只用于发现分析范围。除非用户明确要求比较，否则不要以 “before vs after” 组织文档。

必须读取足够的 final-state 周边代码，以理解：

- changed 和 directly affected component
- entry point 和 public interface
- caller 和重要 dependency
- orchestration 和稳定 internal stage
- persistence 和 state mutation
- transaction 和 concurrency boundary
- side-effect ownership
- cross-component interaction
- 能揭示 runtime behavior 的 tests
- repository pattern、`AGENTS.md` 和相关项目规范

Spec、Detailed Design、Plan、ticket 或旧 implementation 可在需要时用于理解术语和 intent，但不得替代对最终 implementation 的实际观察。

当文档与代码不一致时，描述代码**实际做了什么**。不得静默把 implementation 改写成“它应该存在的架构”。

## 核心规则：Reconstruct，不要 Redesign

描述已经存在的 implementation。

优先：

```text
Observed implementation:
PurchaseOrderService.update() owns the transaction and item synchronization.
```

避免：

```text
PurchaseOrderService should own the transaction and item synchronization.
```

在 reconstruction 阶段不要 refactor、redesign，也不要进行完整 correctness review。

如果无法从代码、tests 或附近文档确认 intent，必须显式标记：

```text
Intent: inferred
Evidence: call structure and tests
```

或者：

```text
Intent: unclear from implementation
```

不得虚构 design rationale。

## 分析策略

不要从第一个 diff hunk 开始一路线性阅读到最后。

从外向内建立模型：

```text
Scope
  ↓
Components
  ↓
Public entry points
  ↓
Use cases
  ↓
Major call chains
  ↓
Key functions
  ↓
Data / state / transaction / side effects
  ↓
Complex logic
  ↓
Recommended review route
```

把 implementation element 分为三个层级：

| Level | 含义 | 处理方式 |
| --- | --- | --- |
| Structural | 定义 responsibility、interface、orchestration、ownership、state flow、transaction、concurrency、side effect 或 cross-component behavior | 详细解释 |
| Supporting | 理解 structural path 所需，但自身并非 design-significant | 在相关位置简要解释 |
| Mechanical | 重复 CRUD、mapping、简单 serializer、field declaration、明显 adapter、generated code、简单 helper | 压缩总结或省略 |

implementation 越大，文档相对越应该被压缩。不要为了说明 8,000 行 implementation，再生成另一份接近 8,000 行的文档。

## 必须重建的内容

只包含适用内容，但所有 design-significant implementation decision 都必须可见。

| 范畴 | 需要重建的内容 |
| --- | --- |
| Component responsibility | 每个重要 component 负责什么、不负责什么、依赖什么、暴露什么 |
| Public interface | 实际的重要 method/function name、parameter、parameter type 和 return type（可获得时） |
| Key internal stage | 表达 business stage、responsibility boundary、稳定 abstraction 或重要 algorithm 的 private/internal method |
| Method contract | Input、output、responsibility、重要 rule、state mutation、error、side effect，以及重要 non-responsibility |
| Use-case call flow | 从 entry point 经 validation/orchestration/persistence/side effect 到 result |
| Data and state flow | 重要数据从哪里产生、在哪里 transform、在哪里成为 authoritative、在哪里修改 persistent state |
| Transaction / concurrency | 实际 transaction boundary、locking、idempotency mechanism、retry assumption 和 concurrency-sensitive section |
| Side-effect ownership | 哪个 component 在什么条件下触发 external 或 cross-domain effect |
| Complex logic | State transition、reconciliation、allocation、synchronization、batching、retry、ordering 等非平凡逻辑 |
| Test seam | 重要 behavior 被验证的 public/component boundary |
| Review route | 能让 reviewer 建立 implementation mental model 的最小有序代码集合 |

## Component Reconstruction

对每个重要 component 使用紧凑结构：

```text
## PurchaseOrderService

Role:
Orchestrates purchase-order mutation use cases.

Owns:
- update orchestration
- state-transition validation
- item synchronization
- transaction boundary

Does not own:
- HTTP request parsing
- response serialization

Depends on:
- PurchaseOrder
- PurchaseOrderItem
- InventoryService
- AuditService

Public entry points:
- update(...)
- cancel(...)
```

关注当前 responsibility boundary，不要做 class-by-class inventory。

## Public Interfaces

重要 entry point 尽量展示准确 signature：

```python
PurchaseOrderService.update(
    *,
    order_id: int,
    data: PurchaseOrderUpdateData,
    operator: User,
) -> PurchaseOrder
```

对每个重要 public entry point，说明：

- 谁调用它
- 它代表什么 use case
- 主要 input/output
- transaction behavior
- 主要 state mutation
- 主要 side effect

不要复制完整 function body。

## Function-Level Structure

只对 structurally significant path 重建 function structure。

一个 function 通常在以下情况属于 significant：

- public use-case entry point
- orchestration method
- validation 或 state-transition owner
- transaction 或 locking boundary
- reconciliation/synchronization/allocation stage
- 带有重要 behavior 的 persistence boundary
- side-effect trigger
- cross-component integration point
- complex algorithm
- 具有重要 fan-in 或 fan-out 的 shared function

优先使用简洁 call tree：

```text
PurchaseOrderService.update()

├── _get_for_update()
├── _validate_update()
│   ├── _validate_status_transition()
│   └── _validate_editable_fields()
├── _apply_order_fields()
├── _sync_items()
│   ├── _classify_items()
│   ├── _create_items()
│   ├── _update_items()
│   └── _remove_items()
├── _apply_inventory_effects()
└── _record_audit()
```

不要因为 trivial helper 存在就全部列出。

## Function Cards

对 reviewer 很可能需要打开阅读的 function 创建 Function Card。

使用紧凑格式：

```text
### PurchaseOrderService._sync_items()

Location:
`purchase/services/purchase_order.py` — `_sync_items`

Called by:
`PurchaseOrderService.update`

Calls:
`_classify_items`, `_create_items`, `_update_items`, `_remove_items`

Purpose:
Synchronizes persisted items against the complete incoming collection.

Input:
Locked order + incoming item collection.

Output:
None.

Important rules:
- incoming collection is authoritative
- missing persisted items are removed
- item identity is matched by ...

State mutation:
Creates, updates, and deletes PurchaseOrderItem rows.

Side effects:
None outside persistence.

Why it matters:
Owns item reconciliation semantics.

Read next:
`_classify_items` only if identity matching or duplicate handling needs inspection.
```

如果 line number 可靠，添加 `path:line-range`；否则使用 path + symbol。不得编造 line number。

## Use-Case 与 Call-Flow Reconstruction

implementation 应优先按 **use case** 组织，而不是按 file 组织。

示例：

```text
### Update Purchase Order

PATCH /purchase-orders/{id}
        ↓
PurchaseOrderViewSet.partial_update()
        ↓
PurchaseOrderUpdateSerializer
        ↓
PurchaseOrderService.update()
        ↓
_get_for_update()
        ↓
_validate_update()
        ↓
_apply_order_fields()
        ↓
_sync_items()
        ↓
_apply_inventory_effects()
        ↓
_record_audit()
        ↓
COMMIT
```

当 branch 会改变 behavior 时，展示有意义的 branch：

```text
target_status changed?
├── no  → normal field/item update
└── yes → validate transition
          ↓
          apply transition
          ↓
          trigger transition-owned side effects
```

只有 ordering 或 cross-component interaction 无法用 call tree 清楚表达时，才使用 sequence diagram。

## Data、State、Transaction 与 Side Effects

对于 business-heavy implementation，必须明确重建 ownership。

### Data / State Flow

```text
HTTP payload
   ↓
Serializer validated_data
   ↓
PurchaseOrderUpdateData
   ↓
PurchaseOrderService.update()
   ├── order fields
   ├── item collection
   └── target status
          ↓
Persistent state
```

识别：

- authoritative input/state
- normalization boundary
- validation ownership
- state mutation ownership
- derived data
- persistence timing

### Transaction / Concurrency

```text
PurchaseOrderService.update()
└── transaction.atomic
    ├── SELECT order FOR UPDATE
    ├── validate against locked state
    ├── mutate order
    ├── synchronize items
    ├── trigger in-transaction effects
    └── audit
```

只有 implementation 有证据支持时，才描述 locking、idempotency、retry 或 concurrency assumption。

### Side-Effect Ownership

```text
Inventory mutation

Owner:
PurchaseOrderService._apply_inventory_effects()

Trigger:
CONFIRMED → ORDERED

Execution:
Inside the order transaction.

Idempotency:
Guarded by state transition; no separate idempotency key observed.
```

除非用户要求 review，否则不要判断这样做是否正确。

## Complex Logic

只有普通 prose 或 call tree 无法表达清楚时，才使用 selective pseudocode。

适合的内容包括：

- state machine 和 transition guard
- collection reconciliation
- allocation/distribution
- dependency ordering
- synchronization
- batching
- retry/idempotency
- non-trivial aggregation

示例：

```text
existing_by_id = persisted items indexed by identity

for incoming item:
    if identity exists:
        update existing item
    else:
        create item

delete persisted items not present in incoming identities
```

不要把普通 loop 或 ORM syntax 机械翻译成 pseudocode。

## Observed Implementation Decisions

记录 final implementation 中实际可观察到的重要 structural choice。

例如：

```text
- The Service owns the transaction boundary.
- Serializer performs input-shape validation; business transition validation remains in Service.
- Incoming item collections are treated as authoritative snapshots.
- Inventory effects are triggered synchronously from the order use case.
```

如果 rationale 不明确，不要写 “because”。区分 observation 与 inference：

```text
Observed:
Inventory effects execute inside the transaction.

Possible rationale:
Not explicit in code; may be intended to keep order and inventory mutation coupled.
```

应谨慎使用 inferred rationale。

## Recommended Review Route

文档最后必须给出一个有序阅读路线，以最小化人工阅读成本。

示例：

```text
1. PurchaseOrderService.update()
   Why: main orchestration and transaction boundary.

2. _validate_status_transition()
   Why: defines business state-transition rules.

3. _sync_items()
   Why: owns reconciliation semantics.

4. _apply_inventory_effects()
   Why: cross-domain side-effect boundary.

5. PurchaseOrderUpdateSerializer.validate()
   Why: defines API-side input restrictions.

6. Relevant service tests
   Why: confirms expected edge behavior.
```

Review Route 应找出能解释大部分 implementation 的最小 symbol 集合，而不是简单列出全部 changed file。

主路线之后，可以按需增加：

```text
Secondary reading:
- mechanical serializers
- admin/display changes
- straightforward model fields
- repetitive tests
```

## 输出

将 study 保存到：

```text
docs/superpowers/studies/YYYY-MM-DD-<feature>-implementation-study.md
```

按需使用以下 section，不要创建空 section：

```markdown
# <Feature> Implementation Study

**Scope:** `<branch / commit range / task / changed area>`

## Mental Model

## Component Map

## Use Cases

## Key Components

### <Component>

#### Responsibility

#### Public Interfaces

#### Function Structure

#### Function Cards

## Cross-Component Call Flows

## Data / State Flow

## Transaction / Concurrency

## Side-Effect Ownership

## Complex Logic

## Observed Implementation Decisions

## Unclear Implementation Intent

## Recommended Review Route

## Low-Priority / Mechanical Areas
```

优先使用 table 表达紧凑 inventory，signature 表达 interface，call tree 表达 orchestration，Function Card 表达重要 symbol，selective pseudocode 表达真正 complex 的 logic。

## 禁止事项

不要：

- 解释每个 changed file
- 解释每个 function
- 按 diff hunk 逐段叙述
- 以 old-vs-new comparison 为中心
- 生成泛泛的 “what changed” summary
- 复制大段代码
- 把 implementation 改写成理想架构
- 在没有证据时推断 requirement 或 rationale
- 进行完整 bug/code-quality review
- 对 mechanical code 与 structural code 投入同等篇幅
- 把 tests 当作 implementation structure；tests 只作为 behavioral evidence
- 用自信措辞掩盖 uncertainty
- 生成一份阅读成本接近直接读代码的文档

## 深度判断

如果有经验的开发者读完后仍不能回答以下问题，说明 study 太浅：

- major component 和 responsibility 是什么？
- public use-case entry point 是什么？
- 主要 runtime call chain 是什么？
- 哪些 function 拥有重要 business stage？
- 重要 data 在哪里成为 authoritative？
- persistent state 在哪里以及如何被修改？
- transaction、locking、concurrency 和 side effect 在哪里被 ownership？
- 哪 10–20% 的代码应该优先阅读？

如果文档解释了以下内容，说明 study 太细：

- 每个 helper
- 每个 serializer field
- ordinary CRUD
- local variable
- equivalent ORM expression
- obvious mapping code
- repetitive tests
- near-complete function body

## Self-Review

提交 study 前确认：

- 文档描述的是**当前 final implementation**，而不是 change history
- diff/commit 信息只用于 scope discovery，而不是 narrative structure
- 重要 component responsibility 和 boundary 已明确
- 重要 public interface 和 function-level structure 准确
- major use-case call chain 可以端到端追踪
- 重要 data/state mutation、transaction、concurrency 和 side-effect ownership 可见
- Function Card 仅用于真正提高理解效率的 symbol
- complex logic 只做 selective explanation
- observation、inference 和 uncertainty 明确区分
- mechanical code 已被压缩或省略
- Recommended Review Route 明显减少人工需要阅读的代码量
- 文档优化目标是 comprehension，而不是 coverage

## Completion

完成 Implementation Study 后停止。

不要自动开始 Code Review、refactoring 或 implementation change。

用户应先 review reconstructed mental model，再决定哪些 component 或 function 值得进入详细检查。
