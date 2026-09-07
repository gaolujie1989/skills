---
name: studying-large-ai-changes
description: Use when an existing implementation is already complete and a large or unfamiliar code change must be understood before detailed review, especially when AI-generated changes are too large to read linearly.
---

# Studying Large AI Changes

> `SKILL.md` is the sole normative source. `SKILL.zh.md` is a synchronized translation for human readers and must not define independent behavior.

## Overview

Use this skill after implementation is substantially complete and before detailed code review.

The goal is not to explain every changed line, summarize every file, or compare old and new code. The goal is to **compress a large implementation into a mental model that lets an experienced developer understand the current design, implementation structure, major call flows, important functions, state/data movement, and review boundaries without reading the entire change linearly**.

Optimize for **comprehension, not coverage**.

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

This is an **implementation comprehension / reconstruction skill**, not a Code Review skill and not a change-summary skill.

## Source of Truth and Scope

The **current implemented code** is the source of truth.

Use the diff, commit range, branch, task files, or changed-file list only to discover scope. Do not organize the document around "before vs after" unless the user explicitly asks for comparison.

Read enough surrounding final-state code to understand:

- changed and directly affected components
- entry points and public interfaces
- callers and important dependencies
- orchestration and stable internal stages
- persistence and state mutation
- transaction and concurrency boundaries
- side-effect ownership
- cross-component interaction
- tests that reveal intended runtime behavior
- repository patterns, `AGENTS.md`, and relevant project conventions

Spec, Detailed Design, Plan, tickets, or prior implementation may be read for terminology and intent when useful, but they must not replace observation of the final implementation.

When documentation and code disagree, describe what the code actually does. Do not silently rewrite the implementation into the architecture that "should" exist.

## Core Rule: Reconstruct, Do Not Redesign

Describe the implementation that exists.

Prefer:

```text
Observed implementation:
PurchaseOrderService.update() owns the transaction and item synchronization.
```

Avoid:

```text
PurchaseOrderService should own the transaction and item synchronization.
```

Do not refactor, redesign, or perform a full correctness review while reconstructing the implementation.

If intent cannot be established from code, tests, or nearby documentation, mark it explicitly:

```text
Intent: inferred
Evidence: call structure and tests
```

or:

```text
Intent: unclear from implementation
```

Never invent design rationale.

## Analysis Strategy

Do not read a large change linearly from the first diff hunk to the last.

Build the model from outside in:

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

Classify implementation elements into three levels:

| Level | Meaning | Treatment |
| --- | --- | --- |
| Structural | Defines responsibility, interface, orchestration, ownership, state flow, transaction, concurrency, side effect, or cross-component behavior | Explain in detail |
| Supporting | Important to understand a structural path but not independently design-significant | Explain briefly where used |
| Mechanical | Repetitive CRUD, mappings, trivial serializers, field declarations, obvious adapters, generated code, simple helpers | Summarize or omit |

The document must become smaller as the implementation becomes larger. Do not produce a second 8,000-line artifact to explain an 8,000-line implementation.

## What Must Be Reconstructed

Include only applicable sections, but make every design-significant implementation decision visible.

| Area | Reconstruct |
| --- | --- |
| Component responsibility | What each important component owns, does not own, depends on, and exposes |
| Public interface | Actual important method/function names, parameters, parameter types, and return types where available |
| Key internal stages | Private/internal methods that represent business stages, responsibility boundaries, stable abstractions, or important algorithms |
| Method contract | Input, output, responsibility, important rules, state mutation, errors, side effects, and important non-responsibilities |
| Use-case call flow | Entry point through validation/orchestration/persistence/side effects to result |
| Data and state flow | Where important data originates, is transformed, becomes authoritative, and mutates persistent state |
| Transaction / concurrency | Actual transaction boundaries, locking, idempotency mechanisms, retry assumptions, and concurrency-sensitive sections |
| Side-effect ownership | Which component triggers external or cross-domain effects and under what conditions |
| Complex logic | State transition, reconciliation, allocation, synchronization, batching, retry, ordering, or other non-trivial logic |
| Test seam | Public/component boundaries where important behavior is verified |
| Review route | Smallest ordered set of code that gives a reviewer the implementation mental model |

## Component Reconstruction

For every important component, use a compact structure such as:

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

Focus on current responsibility boundaries, not class-by-class inventory.

## Public Interfaces

Show exact signatures for important entry points when practical:

```python
PurchaseOrderService.update(
    *,
    order_id: int,
    data: PurchaseOrderUpdateData,
    operator: User,
) -> PurchaseOrder
```

For each important public entry point, state:

- who calls it
- what use case it represents
- major input/output
- transaction behavior
- major state mutation
- major side effects

Do not reproduce full function bodies.

## Function-Level Structure

Reconstruct function structure only for structurally significant paths.

A function is usually significant if it is one or more of:

- a public use-case entry point
- an orchestration method
- a validation or state-transition owner
- a transaction or locking boundary
- a reconciliation/synchronization/allocation stage
- a persistence boundary with meaningful behavior
- a side-effect trigger
- a cross-component integration point
- a complex algorithm
- a shared function with important fan-in or fan-out

Prefer a compact call tree:

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

Do not include trivial helpers merely because they exist.

## Function Cards

Create Function Cards for functions a reviewer is likely to open and inspect.

Use this compact form:

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

Add `path:line-range` when line numbers are reliable; otherwise use path + symbol. Do not fabricate line numbers.

## Use-Case and Call-Flow Reconstruction

Organize the implementation primarily by **use case**, not by file.

Example:

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

Show meaningful branches when they change behavior:

```text
target_status changed?
├── no  → normal field/item update
└── yes → validate transition
          ↓
          apply transition
          ↓
          trigger transition-owned side effects
```

Use a sequence diagram only when ordering or cross-component interaction cannot be expressed clearly with a call tree.

## Data, State, Transaction, and Side Effects

For business-heavy implementations, explicitly reconstruct ownership.

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

Identify:

- authoritative input/state
- normalization boundaries
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

State actual locking, idempotency, retry, or concurrency assumptions only when supported by the implementation.

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

Do not judge whether this is correct unless the user asks for review.

## Complex Logic

Use selective pseudocode only when normal prose or a call tree is insufficient.

Good candidates:

- state machines and transition guards
- collection reconciliation
- allocation/distribution
- dependency ordering
- synchronization
- batching
- retry/idempotency
- non-trivial aggregation

Example:

```text
existing_by_id = persisted items indexed by identity

for incoming item:
    if identity exists:
        update existing item
    else:
        create item

delete persisted items not present in incoming identities
```

Do not translate ordinary loops or ORM syntax into pseudocode.

## Observed Implementation Decisions

Record important structural choices that are visible in the final implementation.

Examples:

```text
- The Service owns the transaction boundary.
- Serializer performs input-shape validation; business transition validation remains in Service.
- Incoming item collections are treated as authoritative snapshots.
- Inventory effects are triggered synchronously from the order use case.
```

If rationale is not explicit, do not write "because". Separate observation from inference:

```text
Observed:
Inventory effects execute inside the transaction.

Possible rationale:
Not explicit in code; may be intended to keep order and inventory mutation coupled.
```

Use inferred rationale sparingly.

## Recommended Review Route

End with an ordered reading route that minimizes human reading cost.

Example:

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

The route should identify the smallest set of symbols that explains most of the implementation. Do not simply list all changed files.

After the primary route, optionally include:

```text
Secondary reading:
- mechanical serializers
- admin/display changes
- straightforward model fields
- repetitive tests
```

## Output

Save the study to:

```text
docs/superpowers/studies/YYYY-MM-DD-<feature>-implementation-study.md
```

Use sections as needed; do not create empty sections.

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

Prefer tables for compact inventories, signatures for interfaces, call trees for orchestration, Function Cards for important symbols, and selective pseudocode for genuinely complex logic.

## What Not to Do

Do not:

- explain every changed file
- explain every function
- narrate the diff hunk by hunk
- center the document on old-vs-new comparison
- produce a generic "what changed" summary
- copy large code blocks
- rewrite the implementation into an ideal architecture
- infer requirements or rationale without evidence
- perform a full bug/code-quality review
- spend equal attention on mechanical and structural code
- treat tests as the implementation structure; use them as behavioral evidence
- hide uncertainty behind confident prose
- create a document so detailed that reading it costs nearly as much as reading the code

## Depth Test

The study is too shallow if, after reading it, an experienced developer still cannot answer:

- What are the major components and responsibilities?
- What are the public use-case entry points?
- What are the main runtime call chains?
- Which functions own the important business stages?
- Where does important data become authoritative?
- Where and how is persistent state mutated?
- Where are transaction, locking, concurrency, and side effects owned?
- Which 10–20% of the code should be read first?

The study is too detailed if it explains:

- every helper
- every serializer field
- ordinary CRUD
- local variables
- equivalent ORM expressions
- obvious mapping code
- repetitive tests
- near-complete function bodies

## Self-Review

Before submitting the study, confirm:

- the document describes the **current final implementation**, not the history of the change
- diff/commit information was used for scope discovery, not as the narrative structure
- important component responsibilities and boundaries are explicit
- important public interfaces and function-level structure are accurate
- major use-case call chains are traceable end to end
- important data/state mutation, transaction, concurrency, and side-effect ownership are visible
- Function Cards are limited to symbols that materially improve understanding
- complex logic is explained selectively
- observation, inference, and uncertainty are clearly distinguished
- mechanical code has been compressed or omitted
- the Recommended Review Route materially reduces how much code a human must read
- the document is optimized for comprehension, not coverage

## Completion

After writing the Implementation Study, stop.

Do not automatically begin Code Review, refactoring, or implementation changes.

The user should first review the reconstructed mental model and decide which components or functions deserve detailed inspection.
