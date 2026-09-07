---
name: superpowers-with-writing-designs
description: Use only when explicitly invoked by the human after a Superpowers spec has been approved and before writing the implementation plan.
---

# Superpowers with Writing Designs

## 概述

在 Superpowers 已确认的 Spec 与 `writing-plans` 之间，增加一个可选的 Detailed Design 阶段。

本 Skill 是 **Superpowers 的手动扩展**，不修改、不替代任何 Superpowers 原有 Skill。

正常流程：

```text
superpowers:brainstorming
        ↓
approved spec
        ↓
[手动触发本 Skill]
        ↓
detailed design
        ↓
human review / approval
        ↓
superpowers:writing-plans
        ↓
正常 Superpowers 实现流程
```

核心目标：

> 在生成大量代码之前，先确定代码结构中重要的设计决策，让人工可以用较低成本审核类、函数、职责和调用关系，并减少 AI 在实现阶段的设计漂移。

## 手动触发

本 Skill **仅在用户明确要求时使用**。

不要因为以下原因自动触发：

* 任务很大
* 涉及 Service
* 涉及多个模块
* 属于 architectural task
* 预计会生成大量代码

即使明显适合 Detailed Design，也必须由用户主动调用。

本 Skill 不修改 Superpowers 原始流程。

## 使用时机

典型使用方式：

```text
Spec 已确认。

使用 superpowers-with-writing-designs
生成 Detailed Design。

我确认 design 后再进入 writing-plans。
```

调用本 Skill 时，必须已经存在经过确认的 Spec。

如果 Spec 尚未确认，则返回 Superpowers 原有 brainstorming/spec 流程，不应提前进行 Detailed Design。

## 输入

开始设计前读取：

1. 已确认的 Spec
2. 与本需求相关的现有代码
3. 已有类、函数和接口
4. 项目架构和既有模式
5. `AGENTS.md`、项目规范及其他相关说明

Spec 是需求和架构层面的 source of truth。

Detailed Design 不应静默改变已经确认的 Spec。

如果发现 Spec 与现有实现或合理代码设计冲突，应明确指出，而不是自行修改需求或架构。

## 输出

保存到：

```text
docs/superpowers/designs/YYYY-MM-DD-<feature>-design.md
```

文档开头：

```markdown
# <Feature> Detailed Design

**Spec:** `<approved-spec-path>`

## Design Goal

<本 Detailed Design 希望锁定哪些实现结构，以及为什么需要这些约束。>
```

## Detailed Design 的职责

Detailed Design 回答：

> **代码在结构上应该长什么样？**

主要确定：

* 类和组件职责
* 类之间的边界
* public interface
* 重要函数签名
* 关键 internal stage methods
* 主要调用链
* 数据流
* 控制流
* transaction boundary
* concurrency / locking
* side effect ownership
* cross-component dependencies
* 重要 test seams
* 非平凡算法

Detailed Design 不负责：

* 实现顺序
* TDD step
* commit 划分
* 每个函数体的具体实现
* 普通局部 helper
* 机械性的 CRUD 代码

这些属于 `writing-plans` 和 implementation 阶段。

## 设计深度

只设计 **design-significant structure**。

如果一个设计决策变化会明显影响以下任一内容，则应进入 Detailed Design：

* responsibility boundary
* public API
* data flow
* control flow
* transaction / concurrency
* cross-component interface
* side-effect ownership
* testability

否则默认留给实现阶段。

## 1. Component Responsibilities

对于重要组件，明确：

```markdown
## PurchaseOrderService

### Responsibility

负责：
- 采购订单写操作
- 业务规则校验
- 状态流转
- 订单和订单项一致性

不负责：
- HTTP request parsing
- response serialization
- 列表查询展示
```

职责边界应能帮助判断：

* 逻辑属于哪个组件
* 哪些逻辑不应该放进该组件
* 是否存在过大的 Service 或职责混合

## 2. Public Interfaces

所有设计显著的 public interface 都应给出准确函数定义。

例如：

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

函数定义应尽量确定：

* method name
* parameters
* parameter types
* return type

public interface 属于 design contract。

实现阶段不应自行修改。

## 3. Key Internal Stage Methods

对于 Service 等包含明显业务流程的组件，可以设计关键 private/internal stage methods。

例如：

```python
def _get_for_update(
    *,
    order_id: int,
) -> PurchaseOrder:
    ...

def _validate_update(
    *,
    order: PurchaseOrder,
    data: PurchaseOrderUpdateData,
    operator: User,
) -> None:
    ...

def _apply_order_fields(
    *,
    order: PurchaseOrder,
    data: PurchaseOrderUpdateData,
) -> None:
    ...

def _sync_items(
    *,
    order: PurchaseOrder,
    items: list[PurchaseOrderItemData],
) -> None:
    ...
```

只定义影响整体代码结构的重要阶段方法。

不要提前设计叶子 helper，例如：

```python
_to_decimal()
_build_item_map()
_normalize_string()
_extract_ids()
_has_changed()
```

除非这些函数本身表达重要 domain abstraction。

## 4. Method Contracts

对于重要方法，说明其 contract。

建议包含：

```text
Inputs:
- ...

Returns:
- ...

Responsibilities:
- ...

Business rules:
- ...

Errors:
- ...

Side effects:
- ...
```

重点说明职责边界，而不是描述每一行实现。

例如：

```text
_sync_items()

Responsibilities:
- 根据完整 incoming item 集合同步订单项
- 创建新增 item
- 更新已有 item
- 删除不存在于 incoming set 的 item

Does not:
- 校验订单状态权限
- 修改订单 header
- 触发库存业务副作用
```

## 5. Call Flow

Service public method 应尽可能体现完整 use case 的高层执行流程。

例如：

```text
update()
├── _get_for_update()
├── _validate_update()
├── _apply_order_fields()
├── _sync_items()
└── _record_audit()
```

一个有经验的 Reviewer 应尽可能仅通过 public method 和调用链理解主要业务流程。

如果 call flow 本身很难理解，通常意味着职责或抽象层级仍需要调整。

## Service 设计原则

Service public method 应主要表达高层业务阶段。

优先采用类似：

```text
load
→ normalize
→ validate
→ mutate
→ synchronize
→ side effects
→ audit
```

不要为了降低函数行数机械拆 helper。

一个 private method 应代表：

* 一个业务阶段
* 一个明确职责
* 一个稳定 abstraction boundary

而不是仅仅代表几行代码。

## 6. Cross-Component Flow

当 use case 跨越多个组件时，明确关键调用关系。

例如：

```text
PurchaseOrderViewSet
    ↓
PurchaseOrderService.update()
    ├── PurchaseOrderQuerySet
    ├── InventoryService
    └── AuditService
```

重点确定：

* 谁发起调用
* 谁拥有业务规则
* 谁拥有数据变更
* 谁负责副作用
* 哪些组件不应该直接互相依赖

只有当顺序、条件或交互较复杂时才使用 sequence diagram。

简单流程优先使用 call tree。

## 7. Transaction and Concurrency

如果相关，明确：

```text
Transaction boundary:
- PurchaseOrderService.update 整体运行于 transaction.atomic

Locking:
- 订单通过 select_for_update 获取

Idempotency:
- ...

Concurrency assumptions:
- ...
```

不要把 transaction boundary 留给 implementation 阶段自行决定，如果它会影响业务正确性。

## 8. Complex Logic

只有当算法本身属于设计决策时才写伪代码。

适合：

* 状态流转
* reconciliation
* allocation
* item synchronization
* matching
* batching
* retry
* idempotency
* dependency ordering

例如：

```text
existing_items = index existing items by id
incoming_items = incoming request items

for incoming item:
    if id exists:
        validate ownership
        update
    else:
        create

delete existing items absent from incoming set
```

普通 CRUD 不需要写伪代码。

优先：

* signature
* contract
* call flow

而不是函数体级伪代码。

## 9. Implementation Freedom

Detailed Design 必须明确保留合理的实现自由。

默认不约束：

* local variable names
* 小型 helper
* 等价 ORM 写法
* 普通数据转换
* loop implementation
* 内部临时数据结构
* 不影响设计边界的 private implementation details

目标不是提前把代码写一遍。

目标是：

> 锁定重要结构，同时保留局部实现自由。

## Design Contract

Design 经用户确认后，视为 implementation structure contract。

实现阶段不应静默改变：

* component responsibilities
* public interfaces
* documented key internal boundaries
* major call flows
* transaction boundaries
* side-effect ownership
* cross-component interfaces

实现计划可以为了 task-local context 重复这些定义，但不应重新设计它们。

## Design Drift

如果后续 Plan 或 Implementation 发现 Detailed Design 存在问题，不允许直接自行修改设计并继续实现。

应该报告：

```text
Design Conflict

Current design:
<当前设计>

Problem:
<为什么该设计在实现时存在问题>

Proposed change:
<建议的新设计>

Impact:
<受影响的类、函数、task、接口>
```

如果改动会影响已经确认的设计结构，则先更新 Detailed Design，并由用户确认后再继续。

## 与 Spec 的关系

Spec 决定：

```text
What should be built?
Why?
What are the requirements?
What is the architecture?
What are the business rules?
```

Detailed Design 决定：

```text
What should the implementation structure look like?
Which classes own which responsibilities?
What interfaces exist?
How do major methods collaborate?
```

Detailed Design 不应重复大量 Spec 内容。

需要引用时，引用 Spec section 即可。

## 与 Plan 的关系

Plan 决定：

```text
Which files change?
What tasks exist?
In what order?
What tests are written?
What implementation steps are executed?
What gets committed?
```

因此：

```text
Spec
    ↓ requirements / architecture

Detailed Design
    ↓ implementation structure

Plan
    ↓ execution steps

Implementation
```

Plan 中可以复制 Detailed Design 中的 exact signatures，以确保 task-local executor 获得必要上下文。

但 Plan 不应重新设计已经确认的接口和结构。

## 输出模板

根据实际复杂度裁剪以下结构：

```markdown
# <Feature> Detailed Design

**Spec:** `...`

## Design Goal

## Component Overview

## <Component 1>

### Responsibility

### Public Interface

### Key Internal Methods

### Method Contracts

### Call Flow

### Transaction / Concurrency

## <Component 2>

...

## Cross-Component Flow

## Complex Logic

## Implementation Freedom

## Design Decisions

## Open Design Conflicts
```

不要为了满足模板而创建空章节。

## Reviewability

Detailed Design 的核心价值之一是：

> 人工 Review Design 应明显比 Review 最终代码便宜。

优先使用：

```text
signatures > implementation code
contracts > long prose
call trees > lengthy explanations
selective pseudocode > full pseudocode
```

两个判断标准：

### 太粗

如果 implementer 仍然需要自行决定：

* Service 应该有哪些 public methods
* 参数怎么传
* 类怎么拆
* 关键函数怎么拆
* 谁调用谁
* transaction 在哪里
* 副作用属于谁

说明 Detailed Design 不够详细。

### 太细

如果 Detailed Design 已经规定：

* 所有 private helper
* 每个循环怎么写
* 每个 ORM expression
* 每个局部变量
* 几乎完整函数体

说明 Detailed Design 太细。

## Self-Review

提交给用户前检查：

* [ ] 与已确认 Spec 一致
* [ ] 所有重要组件职责清晰
* [ ] public interfaces 已明确
* [ ] 重要函数签名准确且一致
* [ ] Service public methods 能体现完整 use-case 高层流程
* [ ] 关键 internal stage methods 有明确职责
* [ ] cross-component ownership 清楚
* [ ] transaction / locking 在需要时已明确
* [ ] complex algorithms 在必要时已有伪代码
* [ ] 没有设计无意义的 leaf helpers
* [ ] 没有提前写 implementation plan
* [ ] 没有把 Detailed Design 写成准代码
* [ ] 文档明显比最终代码更容易人工 Review

## Approval Gate

完成 Detailed Design 后，停止。

向用户提交 design 进行审核。

不要：

* 自动开始 implementation
* 自动执行 plan
* 未经用户确认继续进入下一阶段

如果用户要求修改：

1. 修改 design
2. 检查受影响的 signatures
3. 检查受影响的 call flows
4. 检查 cross-component dependencies
5. 再次提交用户确认

## Handoff

用户明确确认 Detailed Design 后：

```text
Detailed Design 已确认。

下一阶段使用 superpowers:writing-plans。
Plan 应同时读取：

- approved spec
- approved detailed design

Spec 作为 requirements / architecture source of truth。
Detailed Design 作为 implementation structure source of truth。
```

然后结束本 Skill，返回正常 Superpowers 流程。
