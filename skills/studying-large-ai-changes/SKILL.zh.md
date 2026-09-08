---
name: studying-large-ai-changes
description: 当现有 implementation 已基本完成，需要在详细 review 之前理解大规模或陌生代码，尤其是 AI 生成的修改过大、不适合线性阅读时使用。
---

# Studying Large AI Changes

> `SKILL.md` 是唯一规范源。`SKILL.zh.md` 仅供人工阅读的同步译本，不得定义独立行为。

## 目标与边界

在 implementation 基本完成后、进入详细 Code Review 之前，重建一个 mental model，让有经验的开发者理解当前设计、主要 call flow、重要 function 和 state ownership。

优化目标是 **comprehension，而不是 coverage**。产出应指出理解 implementation 所需的最小有效阅读路线。

- 描述实际存在的 implementation；不要 redesign、refactor，也不要进行完整的 correctness/code-quality review。
- 以当前已实现的代码为 source of truth。Diff、commit range、branch、task file 和 changed-file list 用于确定范围，不作为文档的叙述结构。
- 需要时阅读 spec、design、plan、ticket 和旧代码以理解术语或 intent。当它们与当前代码不一致时，描述代码的实际行为。
- 主要按 use case 和 responsibility 组织，而不是按 changed file 或 diff hunk 组织。只有用户明确要求时才比较新旧代码。
- 区分 observation、inference 和 uncertainty。推断 intent 时提供证据；证据不足时标记 intent 不明确。不得虚构 design rationale。

## 分析流程

从外向内建立模型：

1. **确定范围。** 检查指定修改及 repository 规范，包括适用的 `AGENTS.md` 指令。
2. **识别 component 与 entry point。** 阅读当前版本中已修改及直接受影响的 component、caller、public interface 和重要 dependency。
3. **追踪 use case。** 从 entry point 沿 validation、orchestration、关键 internal stage、persistence、side effect 追踪到结果，包括会改变行为的 branch 和 error。
4. **解释 ownership 与复杂逻辑。** 定位 authoritative data、state mutation、transaction/concurrency boundary 和跨 component 的 effect。用 tests 提供行为证据并识别 test seam；不要用 test structure 替代 implementation structure。
5. **选择阅读路线。** 找出能解释大部分 implementation 的 symbol，再围绕该模型撰写 study。

按结构重要性分配细节：

| Level | 含义 | 处理方式 |
| --- | --- | --- |
| Structural | 定义 responsibility、interface、orchestration、business stage、state/transaction ownership、concurrency、side effect 或重要 algorithm | 详细解释 |
| Supporting | 理解 structural path 所需，但自身并非 design-significant | 在相关位置简要解释 |
| Mechanical | 重复 CRUD、mapping、简单 serializer/helper、field declaration、明显 adapter 或 generated code | 压缩总结或省略 |

随着 implementation 规模增加，提高压缩比例，而不是逐一记录新增的 file 或 function。Study 可以随有意义的复杂度增长，但阅读成本应明显低于直接阅读 implementation。

## 重建要求

覆盖下列适用内容。每项事实集中在一个主要位置解释，在其他位置引用，避免跨章节重复。

| 范畴 | 必须包含的内容 |
| --- | --- |
| Component boundary | 重要 component 的 responsibility、non-responsibility、dependency 和对外 entry point |
| Public interface | 实际的重要 method/function name 和 signature，包括 parameter、type 和 return type（可获得时）；caller 及对应的 use case |
| Function structure | Orchestration 及负责重要 business stage、validation/state transition、persistence、algorithm 或 integration 的 internal method；包括具有重要 fan-in/fan-out 的 shared function |
| Method contract | Input、output、responsibility、重要 rule/error、state mutation、side effect 和重要 non-responsibility |
| Use-case flow | 从 entry point 到结果的可追踪路径，包括有意义的 branch、ordering 和 cross-component interaction |
| Data/state flow | 来源、normalization 和 validation boundary、authoritative input/state、transformation、derived data、mutation ownership 和 persistence timing |
| Transaction/concurrency | 实际 transaction 和 locking boundary、idempotency mechanism、retry assumption 及 concurrency-sensitive section |
| Side effect | Owner、触发条件和执行时机，包括发生在 transaction 内还是之后 |
| Complex logic | 非平凡的 state transition、reconciliation、allocation、synchronization、batching、dependency ordering、retry 或 aggregation |
| Test seam | Tests 验证重要 runtime 和边界行为的 public/component boundary |
| Observed decision | 代码中可见的 structural choice，明确区分 observation 和推断的 rationale |
| Reading route | Reviewer 应打开的最小有序 symbol 集合，附位置和原因 |

只有 implementation 提供证据时，才描述 locking、idempotency、retry 等行为假设。观察到某种机制不等于证明其正确性。

### 表达方式与 Function Card

选择最容易说明对应关系的表达方式：

- **Table：** 紧凑的 component map 和 ownership 清单。
- **Signature：** 重要 public interface；省略完整 function body。
- **Call tree：** Orchestration 和关键 internal stage，不包含 trivial helper。
- **Sequence diagram：** Call tree 无法清楚表达的 ordering 或 cross-component interaction。
- **Pseudocode：** Prose 或 call tree 无法充分解释的复杂逻辑；不要机械翻译普通 loop 或 ORM syntax。
- **Function Card：** Reviewer 很可能检查的精选 symbol，尤其是其 contract 在上下文中尚不明确时。

使用以下紧凑的 Function Card 结构：

```text
### <Component.function>
Location: <path + symbol，或已核实的 path:line-range>
Call context: <重要 caller 与 callee>
Contract: <purpose、input/output、重要 rule 和 error>
State / effects: <适用的 mutation、transaction behavior、external effect>
Read next: <相关 symbol 及阅读原因，需要时填写>
```

当重要 non-responsibility 有助于明确边界时，将其写入 contract。省略不适用的字段及附近已解释的信息。不要为每个 function 创建 card。

Line number 可靠时使用行号，否则使用 path + symbol。不得编造位置。

## 贯穿示例：更新采购订单

以下是针对一个假设 implementation 的 study 示例，不是规定必须采用的架构。实际 study 中的名称、signature、rule 和 transaction behavior 均应来自代码。

### Mental Model 与 Component

`PurchaseOrderService` 编排订单修改，负责 transition validation、item reconciliation 和 transaction。API 层解析请求并序列化响应。订单 use case 调用库存修改和 audit 记录。

| Component | Owns | 重要 dependency / entry point |
| --- | --- | --- |
| `PurchaseOrderViewSet` | HTTP 入口和响应处理 | Serializer、service；`partial_update()` |
| `PurchaseOrderUpdateSerializer` | Input-shape validation | 生成 service input |
| `PurchaseOrderService` | Business validation、order/item mutation、transaction orchestration | Order/item model、inventory 和 audit service；`update()` |

### Public Interface

```python
PurchaseOrderService.update(
    *,
    order_id: int,
    data: PurchaseOrderUpdateData,
    operator: User,
) -> PurchaseOrder
```

由 `PurchaseOrderViewSet.partial_update()` 调用，对应 `PATCH /purchase-orders/{id}`。接收已验证的更新数据和 operator，返回更新后的订单。Transaction 和 side-effect behavior 见下方流程。

### Use-Case Flow 与 Function Structure

```text
PATCH /purchase-orders/{id}
└── PurchaseOrderViewSet.partial_update()
    ├── PurchaseOrderUpdateSerializer → PurchaseOrderUpdateData
    └── PurchaseOrderService.update() [transaction.atomic]
        ├── _get_for_update() [SELECT order FOR UPDATE]
        ├── _validate_update()
        │   ├── target status changed? → _validate_status_transition()
        │   └── _validate_editable_fields()
        ├── _apply_order_fields()
        ├── _sync_items()
        │   ├── _classify_items()
        │   ├── _create_items()
        │   ├── _update_items()
        │   └── _remove_items()
        ├── _apply_inventory_effects() [CONFIRMED → ORDERED only]
        └── _record_audit()
    → COMMIT → result
```

Validation 基于已锁定的 state 执行。Transition 被拒绝时，在 mutation 之前退出并产生 validation error。Target status 未变化时，仍执行常规 field/item validation 和 update。

### Data、State 与 Effect

| Data / effect | Ownership 与语义 |
| --- | --- |
| Input | Serializer 将 HTTP payload 验证并规范化为 `PurchaseOrderUpdateData` |
| Order state | Service 在 transaction 内验证并修改已锁定的订单 |
| Item collection | 输入集合是 authoritative snapshot；`_sync_items()` 持久化 reconciliation 结果 |
| Inventory | `_apply_inventory_effects()` 在 `CONFIRMED → ORDERED` 时触发同步库存修改，执行于订单 transaction 内 |
| Audit | `_record_audit()` 在 transaction 内记录 mutation |

Observed：inventory effect 由 state transition 守卫；未观察到独立 idempotency key。从已检查的 implementation 无法明确 retry behavior。同步执行 inventory effect 的 rationale 并不明确；根据 call structure，可以推断其可能意在耦合订单和库存修改。

### Function Card 与复杂逻辑

```text
### PurchaseOrderService._sync_items()
Location: purchase/services/purchase_order.py — _sync_items
Call context: update() → _sync_items() → _classify_items(),
              _create_items(), _update_items(), _remove_items()
Contract: 已锁定的订单 + 完整输入 item collection → None。
          按 item ID 匹配；输入中缺失的已持久化 item 会被删除。
          新 item 不带 ID。_classify_items() 在写入 item 前
          拒绝未知或重复 ID。
State / effects: 在 caller 的 transaction 内创建、更新和删除
                 PurchaseOrderItem 记录；无 external effect。
Read next: _classify_items()，检查 identity validation 和分组。
```

Reconciliation 语义：

```text
写入前对完整输入分类：
    无 ID → create
    已知且唯一的 ID → update
    未知或重复 ID → validation error
将输入 ID 集合中缺失的已持久化 item 归入 remove
执行 create / update / remove 分组
```

相关 service tests 为省略即删除、拒绝无效 identity 和 transition 触发 inventory 的行为提供证据。

### Recommended Review Route

实际 study 中，应为下列每个 symbol 附上已核实的位置。

1. `PurchaseOrderService.update()` — Orchestration 和 transaction boundary。
2. `_validate_status_transition()` — Business transition rule。
3. `_sync_items()` 和 `_classify_items()` — Snapshot 和 identity 语义。
4. `_apply_inventory_effects()` — Cross-domain effect 和触发条件。
5. `PurchaseOrderUpdateSerializer.validate()` — API 侧输入约束。
6. 相关 service tests — Service boundary 上预期的边界行为。

Mechanical field 和 response mapping 可以延后阅读。

## 输出

将 study 保存到：

```text
docs/superpowers/studies/YYYY-MM-DD-<feature>-implementation-study.md
```

按需使用以下提纲。省略空的或不适用的章节，合并重叠解释。在 use case 中包含相关 signature、call tree 和精选 Function Card；共用的 ownership 规则放在跨用例章节。

```markdown
# <Feature> Implementation Study

**Scope:** `<branch / commit range / task / changed area>`

## Mental Model

## Component Map

## Use Cases

### <Use Case>

## Shared Data, State, Transactions, and Side Effects

## Complex Logic

## Observed Decisions and Unclear Intent

## Recommended Review Route
```

以有序阅读路线结尾，每一站包含位置、symbol 和原因。找出能解释大部分 implementation 的最小集合，而不是列出全部 changed file。主路线之后可按需列出延后阅读的 mechanical 区域。

## Self-Review 与完成条件

提交前确认：

- Study 描述当前代码，并明确区分 observation、inference 和 uncertainty。
- 开发者能识别主要 responsibility、public entry point、重要 internal stage 和端到端 runtime call chain。
- 适用的 authoritative data、state mutation、transaction/concurrency boundary、side effect 和相关 error 已清楚呈现。
- Signature、symbol、位置和行为描述均有代码依据；tests 用作证据。
- Mechanical 细节及重复解释已压缩；Function Card 和 pseudocode 帮助理解，而不是重复代码。
- 阅读路线明显降低人工阅读成本，指出应优先检查的少量代码；可行时约为 10–20%。

如果这些问题仍未回答，应加深 structural explanation。如果文档逐一讲解每个 helper、serializer field、local variable、普通 CRUD 操作、重复 test 或近乎完整的 function body，应继续压缩。

完成 Implementation Study 后停止。不要自动开始 Code Review、refactoring 或 implementation change。用户先检查 mental model，再决定哪些部分需要详细阅读。
