---
name: studying-large-ai-changes
description: Use when an existing implementation is already complete and a large or unfamiliar code change must be understood before detailed review, especially when AI-generated changes are too large to read linearly.
---

# Studying Large AI Changes

> `SKILL.md` is the sole normative source. `SKILL.zh.md` is a synchronized translation for human readers and must not define independent behavior.

## Purpose and Boundaries

After implementation is substantially complete, reconstruct a mental model that lets an experienced developer understand the current design, major call flows, important functions, and state ownership before detailed code review.

Optimize for **comprehension, not coverage**. The result should identify the smallest useful reading route through the implementation.

- Describe the implementation that exists; do not redesign, refactor, or perform a full correctness/code-quality review.
- Use current implemented code as the source of truth. Diff, commit range, branch, task files, and changed-file lists establish scope, not the document's narrative.
- Read specs, designs, plans, tickets, and prior code for terminology or intent when useful. When they disagree with current code, describe the code's actual behavior.
- Organize primarily by use case and responsibility, not by changed file or diff hunk. Compare old and new code only when explicitly requested.
- Separate observation, inference, and uncertainty. Support inferred intent with evidence; mark intent as unclear when evidence is insufficient. Never invent design rationale.

## Analysis Workflow

Build the model from outside in:

1. **Establish scope.** Inspect the requested change and repository conventions, including applicable `AGENTS.md` instructions.
2. **Map components and entry points.** Read changed and directly affected components, callers, public interfaces, and important dependencies in their final state.
3. **Trace use cases.** Follow entry points through validation, orchestration, key internal stages, persistence, side effects, and results. Include behavior-changing branches and errors.
4. **Explain ownership and complex logic.** Locate authoritative data, state mutation, transaction/concurrency boundaries, and cross-component effects. Use tests as behavioral evidence and identify test seams; do not substitute test structure for implementation structure.
5. **Select the reading route.** Identify the symbols that explain most of the implementation, then write the study around that model.

Allocate detail by structural importance:

| Level | Meaning | Treatment |
| --- | --- | --- |
| Structural | Defines responsibility, interface, orchestration, business stages, state/transaction ownership, concurrency, side effects, or important algorithms | Explain in detail |
| Supporting | Needed to understand a structural path but not independently design-significant | Explain briefly where used |
| Mechanical | Repetitive CRUD, mappings, simple serializers/helpers, field declarations, obvious adapters, or generated code | Summarize or omit |

As implementation size grows, increase the compression ratio rather than documenting each additional file or function. The study may grow with meaningful complexity, but reading it should cost substantially less than reading the implementation.

## Reconstruction Requirements

Cover each applicable area below. Give each fact a primary home and cross-reference it elsewhere rather than repeating it across sections.

| Area | Required content |
| --- | --- |
| Component boundaries | Important components' responsibilities, non-responsibilities, dependencies, and exposed entry points |
| Public interfaces | Actual important method/function names and signatures, including parameters, types, and return types where available; callers and represented use cases |
| Function structure | Orchestration and internal methods that own meaningful business stages, validation/state transitions, persistence, algorithms, or integration; include shared functions with important fan-in/fan-out |
| Method contracts | Inputs, outputs, responsibility, important rules/errors, state mutation, side effects, and important non-responsibilities |
| Use-case flow | Traceable path from entry point to result, with meaningful branches, ordering, and cross-component interaction |
| Data/state flow | Origins, normalization and validation boundaries, authoritative input/state, transformations, derived data, mutation ownership, and persistence timing |
| Transaction/concurrency | Actual transaction and locking boundaries, idempotency mechanisms, retry assumptions, and concurrency-sensitive sections |
| Side effects | Owner, trigger conditions, and execution timing, including whether effects occur inside or after a transaction |
| Complex logic | Non-trivial state transitions, reconciliation, allocation, synchronization, batching, dependency ordering, retry, or aggregation |
| Test seams | Public/component boundaries where tests verify important runtime and edge behavior |
| Observed decisions | Structural choices visible in code, with explicit distinction between observation and inferred rationale |
| Reading route | Smallest ordered set of symbols a reviewer should open, with locations and reasons |

State locking, idempotency, retry, and other behavioral assumptions only when supported by the implementation. An observed mechanism is not proof of correctness.

### Representation and Function Cards

Choose the representation that makes each relationship easiest to follow:

- **Tables:** compact component maps and ownership inventories.
- **Signatures:** important public interfaces; omit full function bodies.
- **Call trees:** orchestration and key internal stages, excluding trivial helpers.
- **Sequence diagrams:** ordering or cross-component interaction that a call tree cannot clearly express.
- **Pseudocode:** complex logic that prose or call trees cannot adequately explain; do not mechanically translate ordinary loops or ORM syntax.
- **Function Cards:** selected symbols a reviewer is likely to inspect, especially when their contracts are not already clear from the surrounding explanation.

Use this compact Function Card structure:

```text
### <Component.function>
Location: <path + symbol, or verified path:line-range>
Call context: <important callers and callees>
Contract: <purpose, input/output, important rules and errors>
State / effects: <mutations, transaction behavior, external effects when relevant>
Read next: <related symbol and reason, when useful>
```

Include important non-responsibilities in the contract when they clarify a boundary. Omit inapplicable fields and information already explained nearby. Do not create a card for every function.

Use reliable line numbers when available; otherwise use path + symbol. Never fabricate locations.

## Worked Example: Update Purchase Order

The following is an illustrative study of one hypothetical implementation, not a prescribed architecture. In an actual study, derive names, signatures, rules, and transaction behavior from the code.

### Mental Model and Components

`PurchaseOrderService` orchestrates order mutations and owns transition validation, item reconciliation, and the transaction. The API layer parses requests and serializes responses. Inventory mutation and audit recording are invoked by the order use case.

| Component | Owns | Important dependencies / entry points |
| --- | --- | --- |
| `PurchaseOrderViewSet` | HTTP entry and response handling | Serializer, service; `partial_update()` |
| `PurchaseOrderUpdateSerializer` | Input-shape validation | Produces service input |
| `PurchaseOrderService` | Business validation, order/item mutation, transaction orchestration | Order/item models, inventory and audit services; `update()` |

### Public Interface

```python
PurchaseOrderService.update(
    *,
    order_id: int,
    data: PurchaseOrderUpdateData,
    operator: User,
) -> PurchaseOrder
```

Called by `PurchaseOrderViewSet.partial_update()` for `PATCH /purchase-orders/{id}`. It accepts validated update data and the operator, then returns the updated order. Transaction and side-effect behavior are shown in the flow below.

### Use-Case Flow and Function Structure

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

Validation runs against locked state. A rejected transition exits before mutation and produces a validation error. When target status is unchanged, normal field/item validation and updates still run.

### Data, State, and Effects

| Data / effect | Ownership and semantics |
| --- | --- |
| Input | HTTP payload is validated and normalized by the serializer into `PurchaseOrderUpdateData` |
| Order state | Service validates and mutates the locked order within the transaction |
| Item collection | Incoming collection is the authoritative snapshot; `_sync_items()` persists the reconciliation |
| Inventory | `_apply_inventory_effects()` triggers synchronous inventory mutation on `CONFIRMED → ORDERED`, inside the order transaction |
| Audit | `_record_audit()` records the mutation inside the transaction |

Observed: inventory effects are guarded by a state transition; no separate idempotency key is observed. Retry behavior is unclear from the inspected implementation. The rationale for synchronous inventory effects is not explicit; coupling order and inventory mutation is a possible inference from the call structure.

### Function Card and Complex Logic

```text
### PurchaseOrderService._sync_items()
Location: purchase/services/purchase_order.py — _sync_items
Call context: update() → _sync_items() → _classify_items(),
              _create_items(), _update_items(), _remove_items()
Contract: Locked order + complete incoming item collection → None.
          Match by item ID; missing persisted items are removed.
          New items have no ID. Unknown or duplicate IDs are rejected
          by _classify_items() before item writes.
State / effects: Creates, updates, and deletes PurchaseOrderItem rows
                 within the caller's transaction; no external effects.
Read next: _classify_items() for identity validation and partitioning.
```

Reconciliation semantics:

```text
classify the complete input before writing:
    no ID → create
    known, unique ID → update
    unknown or duplicate ID → validation error
classify persisted items absent from incoming IDs as remove
apply the create / update / remove groups
```

Relevant service tests provide evidence for deletion by omission, invalid identity rejection, and transition-triggered inventory behavior.

### Recommended Review Route

In an actual study, attach a verified location to each symbol below.

1. `PurchaseOrderService.update()` — orchestration and transaction boundary.
2. `_validate_status_transition()` — business transition rules.
3. `_sync_items()` and `_classify_items()` — snapshot and identity semantics.
4. `_apply_inventory_effects()` — cross-domain effects and trigger conditions.
5. `PurchaseOrderUpdateSerializer.validate()` — API-side input restrictions.
6. Relevant service tests — expected edge behavior at the service boundary.

Mechanical fields and response mappings can be deferred.

## Output

Save the study to:

```text
docs/superpowers/studies/YYYY-MM-DD-<feature>-implementation-study.md
```

Use the following outline as needed. Omit empty/inapplicable sections and merge overlapping explanations. Within use cases, include relevant signatures, call trees, and selected Function Cards. Put shared ownership rules in the cross-cutting section.

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

End with the ordered reading route: location + symbol + reason for each stop. Identify the smallest set explaining most of the implementation, not all changed files. Optionally list deferred mechanical areas after the primary route.

## Self-Review and Completion

Before submitting, confirm:

- The study describes current code, with observation, inference, and uncertainty clearly distinguished.
- A developer can identify major responsibilities, public entry points, important internal stages, and end-to-end runtime call chains.
- Authoritative data, state mutation, transaction/concurrency boundaries, side effects, and relevant errors are visible where applicable.
- Signatures, symbols, locations, and behavioral claims are grounded in code; tests serve as evidence.
- Mechanical details and repeated explanations are compressed; Function Cards and pseudocode add understanding rather than duplicate code.
- The reading route materially reduces human reading cost and identifies which small portion of the code to inspect first, roughly 10–20% where practical.

If these questions remain unanswered, deepen the structural explanation. If the document walks through every helper, serializer field, local variable, ordinary CRUD operation, repetitive test, or near-complete function body, compress it.

After writing the Implementation Study, stop. Do not automatically begin Code Review, refactoring, or implementation changes. The user first reviews the mental model and decides where detailed inspection is needed.
