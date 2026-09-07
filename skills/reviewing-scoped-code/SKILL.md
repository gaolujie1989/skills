---
name: reviewing-scoped-code
description: Use only when explicitly invoked by the human to review the current state and quality of a specified code scope on the current branch.
---

# Reviewing Scoped Code

## 目标

审查**当前分支中用户指定范围的现有代码**。

必须先通过真实源码建立 Mental Model，再判断：

* 当前代码到底做什么；
* 架构和职责是否合理；
* 业务规则与状态变化是否清晰；
* 是否存在 correctness / reliability / maintainability 问题；
* 测试是否真正证明关键行为。

核心原则：

> Understand first. Judge second. Show the code.

不要用长篇代码摘要替代源码阅读。

---

# 1. 确定 Review Scope

用户必须指定主要审查范围，例如：

```text
movie/services/catalog.py
movie/services/**
CatalogService
订单审核相关代码
/api/orders + /services/orders
```

将其记录为 **Primary Scope**。

允许读取范围外代码作为 **Context Scope**，包括：

* caller / callee；
* Model / schema；
* shared utility；
* permission / validation；
* configuration；
* tests；
* external adapter。

但不得因为发现相关代码就无限扩大 Review。

默认规则：

```text
Primary Scope
→ 可以产生正式 Finding

Context Scope
→ 用于理解和提供证据
→ 非必要不审查无关质量问题
```

如果发现范围外存在直接导致 Primary Scope 错误的代码，可以明确标记：

`External Dependency Risk`

---

# 2. Walkthrough

开始 Review 前先给 Reviewer 一屏以内的概览。

## Purpose

这部分代码承担什么业务或系统职责？

## Entry Points

谁会进入这部分代码？

例如：

```text
HTTP API
Celery Task
Management Command
Signal
Service Caller
Frontend Request
```

## Primary Flow

例如：

```text
API
→ Serializer
→ OrderService
→ Order Model
→ InventoryService
→ AuditLog
```

## Important Responsibilities

只列真正关键的 3～8 项职责。

## External Dependencies

指出：

* DB；
* Redis；
* 外部 API；
* filesystem；
* queue；
* shared services。

不要在 Walkthrough 阶段输出普通 Code Review Findings。

---

# 3. Mental Model Map

根据真实源码建立当前实现的结构图。

例如：

```text
OrderViewSet.approve()
        ↓
OrderApprovalService.execute()
        ├─ validate_order()
        ├─ check_permission()
        ├─ update_status()
        ├─ InventoryService.reserve()
        └─ AuditLog.create()
```

每个关键节点附源码位置。

优先使用当前 Codex Surface 支持的可点击源码引用。

显示至少包含：

```text
relative/path.py:start-end
```

Mental Model 重点识别：

* Entry Point
* Core Business Logic
* State Owner
* Critical Mutation
* Persistence
* External Side Effect
* Async Boundary
* Permission / Validation
* Error Boundary
* Tests

不要为了完整而罗列所有 helper。

---

# 4. Source Review Route

虽然不构建 Change Stack，但仍必须为 Reviewer 生成**源码阅读路线**。

阅读顺序按照“理解当前实现”的依赖关系决定，而不是文件名。

例如：

| Step | 阅读目标        | READ                                 | 为什么看               |
| ---- | ----------- | ------------------------------------ | ------------------ |
| R1   | 核心数据模型      | `orders/models.py:40-110`            | 理解状态和 invariant    |
| R2   | 核心 Service  | `orders/services/approval.py:30-130` | 理解主要业务行为           |
| R3   | Side Effect | `inventory/services.py:60-120`       | 理解外部影响             |
| R4   | API Entry   | `orders/views.py:100-150`            | 理解入口、权限、validation |
| R5   | Tests       | `tests/orders/...`                   | 检查行为证据             |

## 每一个 Step

必须包含：

### READ

Reviewer 应认真阅读的源码。

### CONTEXT

理解 READ 所需的：

* callers；
* callees；
* types；
* model；
* config；
* tests。

### Reading Focus

只告诉 Reviewer“看什么”，不要替代源码。

例如：

```text
approval.py:42-55
→ 看进入 Service 前有哪些 invariant

approval.py:57-75
→ 看状态什么时候发生修改

approval.py:77-91
→ 看 external side effect 与 transaction 的关系
```

### Checkpoint

Reviewer 看完真实源码后应能够回答：

* 这个组件负责什么？
* 什么不属于它的职责？
* 输入假设是什么？
* 状态在哪里变化？
* side effect 在哪里发生？
* failure 如何传播？

---

# 5. Code Peek

Reviewer 可以要求：

```text
peek <symbol>
callers <symbol>
callees <symbol>
trace <entrypoint>
```

`peek` 输出：

```text
Symbol
├─ Purpose
├─ Definition
├─ Inputs / Outputs
├─ Important Callers
├─ Important Callees
├─ State Mutation
├─ Side Effects
└─ Error / Transaction Boundary
```

所有重要 symbol 必须附源码位置。

优先使用：

1. language-aware / LSP / code graph；
2. repository search；
3. `rg` / `git grep`。

不要仅根据名称推断调用关系。

---

# 6. Runtime Trace

选择最重要的真实执行路径。

至少包含主要 Happy Path，例如：

```text
POST /orders/{id}/approve
→ OrderViewSet.approve()
→ OrderApprovalService.execute()
→ InventoryService.reserve()
→ Order.save()
→ AuditLog.create()
```

重要模块还应增加一个 Failure Path：

```text
OrderApprovalService.execute()
→ InventoryService.reserve()
→ External API raises
→ ?
```

每一步指出：

* 源码位置；
* 关键输入；
* 关键判断；
* state mutation；
* side effect；
* exception propagation。

Reviewer 应沿 Runtime Trace 再快速阅读一次真实源码。

---

# 7. Review Dimensions

建立 Mental Model 后才能正式 Review。

## Correctness

检查：

* 实现是否与代码自身表达的 contract 一致；
* branch / edge case 是否正确；
* state transition 是否完整；
* error path 是否可能留下错误状态；
* null / empty / boundary 输入；
* 调用顺序是否产生行为错误。

## Business Rules

如果存在 Requirement / Design / Spec：

* 是否实现全部规则；
* 是否有遗漏；
* 是否有规则前后不一致。

没有正式需求时，不猜业务需求。

区分：

```text
Code Defect
Design Concern
Requirement Unknown
```

## Architecture

检查：

* responsibility 是否清晰；
* service / model / API 边界；
* dependency direction；
* coupling / cohesion；
* abstraction 是否真正有价值；
* 是否存在明显 YAGNI / over-engineering；
* 是否存在重复或绕过既有 abstraction。

## Data & State

检查：

* invariant；
* database constraint；
* transaction boundary；
* concurrency；
* locking；
* idempotency；
* consistency；
* duplicate writes；
* partial failure。

## Interface

检查：

* API / function contract；
* validation；
* permission；
* compatibility；
* error semantics；
* serialization；
* caller 是否正确理解 callee contract。

## Reliability

检查：

* exception handling；
* retry；
* timeout；
* rollback；
* external dependency failure；
* asynchronous consistency；
* observability。

## Maintainability

只报告真正影响维护的质量问题：

* 职责严重混杂；
* 复杂控制流；
* 隐含 contract；
* 重复业务规则；
* misleading naming；
* mutation 难以追踪；
* 难以测试的设计。

不要为了风格偏好产生 Finding。

---

# 8. Tests as Evidence

不要只运行测试然后说“通过”。

将关键行为映射到测试：

| Behavior / Risk | Test                     | Evidence |
| --------------- | ------------------------ | -------- |
| 正常审核            | `test_approve_pending`   | Covered  |
| 重复审核            | `test_reapprove`         | Covered  |
| 外部调用失败回滚        | `test_inventory_failure` | Covered  |
| 并发审核            | —                        | Missing  |

同时检查测试质量：

* 测试是否验证真实 behavior；
* 是否只验证 mock interaction；
* failure path 是否覆盖；
* boundary 是否覆盖；
* 测试是否可能在错误实现下依然通过。

测试通过只能作为 Evidence。

不能作为“实现正确”的结论。

---

# 9. Findings

Finding 必须：

* 有具体源码证据；
* 有实际影响；
* 能说明触发条件；
* 不报告纯风格偏好；
* 不重复同一个根因。

严重级别：

### P0 — Critical

可能导致严重数据损坏、安全事故或系统不可用。

### P1 — High

现实场景下可能导致错误业务结果、数据不一致或明显 regression。

### P2 — Medium

真实缺陷或显著设计问题，但影响受限。

### P3 — Low

值得修复的维护性或局部质量问题。

每条 Finding 使用：

```text
[P1] 标题

Source:
path/file.py:42-67

Evidence:
具体代码行为。

Trigger:
什么情况下发生。

Impact:
为什么重要。

Reasoning:
代码为什么会产生该问题。

Suggested direction:
修复方向，而不是无必要重写整个实现。
```

如果没有发现值得报告的问题：

明确写：

`No material findings in the reviewed scope.`

不要为了显得有价值强行产生 Finding。

---

# 10. Quality Assessment

Findings 之后给当前代码一个结构化的**现状评估**。

使用：

```text
Strong
Acceptable
Needs Attention
Risky
```

分别评价：

| Dimension        | Assessment | Evidence |
| ---------------- | ---------- | -------- |
| Correctness      |            |          |
| Architecture     |            |          |
| Data / State     |            |          |
| Failure Handling |            |          |
| Testability      |            |          |
| Maintainability  |            |          |

必须引用具体源码或测试依据。

不要生成没有证据的数字评分，例如 `8.3/10`。

---

# 11. Human Judgment

最后单独列出需要 Reviewer 人工判断的问题：

## Design Decisions

例如：

* Service 是否承担过多职责？
* 当前 transaction scope 是否是业务真正需要的原子边界？
* 某 abstraction 是否值得长期保留？

## Requirement Questions

代码无法回答的业务问题。

## Accepted Trade-offs

当前设计有明显 trade-off，但不能仅凭代码判断对错。

AI 可以提供：

* 源码事实；
* 调用关系；
* 风险；
* trade-off；
* 测试证据。

最终架构与业务判断由 Human Reviewer 完成。

---

# 输出格式

首次运行默认输出：

```text
# Review Scope

# Walkthrough

# Mental Model

# Source Review Route

# Runtime Trace

# Findings

# Tests as Evidence

# Quality Assessment

# Human Judgment / Open Questions
```

对于较大范围，`Source Review Route` 先给完整目录，但不要展开所有源码。

Reviewer 可以继续：

```text
open R2
next
previous
peek <symbol>
callers <symbol>
callees <symbol>
trace <entrypoint>
explain finding P1-2
```

---

# Review Boundary

这是 **Current State Review**，不是 Change Review。

默认：

```text
不需要 base branch
不需要分析 git diff
不需要 Change Stack
不需要解释“这次改了什么”
```

审查的是：

> 当前分支、当前指定范围里的代码现在是什么状态。

如果用户明确要求结合历史、commit 或其他 branch，再读取 diff/history。

---

# 输出纪律

必须：

```text
Source first
Evidence first
Understand before judge
Scope stays bounded
Tests are evidence
Human owns final judgment
```

禁止：

* 逐文件机械总结；
* 为所有函数生成解释；
* 用自然语言摘要代替源码；
* 没看 caller/callee 就判断局部代码；
* 将代码范围无限扩张；
* 把 stylistic preference 当 defect；
* 因为测试通过就宣称实现正确；
* 没有证据就给 Finding；
* 为了输出数量制造低价值问题。

最终成功标准：

Reviewer 看完路线和关键源码后，能够回答：

1. 这部分代码承担什么职责？
2. 从哪里进入？
3. 核心执行路径是什么？
4. 关键状态在哪里读取和修改？
5. 依赖谁，又被谁依赖？
6. transaction / failure / side effect 如何工作？
7. 当前设计的主要优点和风险是什么？
8. 测试证明了什么、没有证明什么？
9. 当前有哪些真正值得修复的问题？
10. 我是否认可这部分代码当前的设计与质量？
