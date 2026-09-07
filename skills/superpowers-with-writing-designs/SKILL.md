---
name: superpowers-with-writing-designs
description: Use only when explicitly invoked by the human to create a Detailed Design before writing the implementation plan.
---

# Superpowers with Writing Designs

> `SKILL.md` is the canonical source. `SKILL.zh.md` is a synchronized, human-readable translation and must not define independent behavior.

## Overview

Add an optional Detailed Design stage between a Superpowers Spec and `superpowers:writing-plans`:

```text
Spec → Detailed Design → human review / approval → Plan → Implementation
```

This skill extends Superpowers without replacing or overriding existing Superpowers skills.

Detailed Design resolves implementation-structure decisions before substantial code is generated, making human review cheaper and reducing implementation drift.

## Sources and Boundaries

Before designing, read the relevant Spec, implementation, classes, functions, interfaces, architecture, repository patterns, `AGENTS.md`, and project conventions.

The Spec is the source of truth for requirements and architecture. Reference it instead of duplicating it. Do not silently change or invent requirements; surface conflicts between the Spec, existing code, and a sound implementation structure.

Detailed Design answers:

> What should the code look like structurally?

Include a decision when changing it would materially affect responsibilities, public APIs, data or control flow, transactions, concurrency, side-effect ownership, cross-component interfaces, or testability.

Leave these to the Plan or Implementation:

- implementation order, TDD steps, and commit structure
- complete function bodies and mechanical CRUD
- local variables, ordinary transformations, and equivalent ORM expressions
- leaf helpers and private details that do not change an approved structural boundary

The goal is to lock important structure without writing the code twice.

## Required Design Content

Scale the document to the use case. Include only applicable sections, but make every design-significant decision explicit.

| Area | Define |
| --- | --- |
| Component responsibilities | What each important component owns and does not own; its dependencies and boundary |
| Public interfaces | Exact method names, parameters, parameter types, and return types |
| Key internal stages | Private/internal methods only when they represent a business stage, responsibility, or stable abstraction boundary |
| Method contracts | Inputs, returns, responsibilities, business rules, errors, side effects, and explicit non-responsibilities where useful |
| Call and data flow | Who initiates the use case, who validates, who changes state, and how major stages collaborate |
| Transactions and concurrency | Transaction boundary, locking, idempotency, and concurrency assumptions when correctness depends on them |
| Complex logic | Selective pseudocode for non-trivial state transitions, reconciliation, allocation, synchronization, batching, retries, or dependency ordering |
| Test seams | Important public or component boundaries through which behavior can be verified |

Public methods on orchestration-heavy Services should communicate high-level business stages. Do not extract helpers merely to reduce line count.

Example:

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

For a significant stage such as `_sync_items()`, state its contract concisely:

```text
Responsibilities:
- synchronize from the complete incoming item collection
- create, update, and remove items as required

Does not:
- validate order-status permissions
- modify the order header
- trigger inventory side effects
```

Use a concise call tree by default. Use a sequence diagram only when ordering, branching, or cross-component interaction would otherwise be unclear.

## Design Depth

The design is too shallow if the implementer still must decide important component boundaries, public methods, parameter shapes, major stage methods, call ownership, transaction boundaries, or side-effect ownership.

The design is too detailed if it specifies every helper, loop, ORM expression, local variable, or near-complete function body.

## Output

Save the design to:

```text
docs/superpowers/designs/YYYY-MM-DD-<feature>-design.md
```

Use this structure as needed; do not create empty sections:

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

Prefer signatures over implementation code, contracts over long prose, call trees over lengthy explanations, and selective pseudocode over full pseudocode.

## Design Contract and Drift

Once the human approves the Detailed Design, it becomes the source of truth for implementation structure. The Plan may copy exact definitions for task-local context, but must not redesign them.

If planning or implementation exposes a material problem, stop the affected work and report:

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

Update the Detailed Design and obtain human approval before continuing work affected by a change to responsibilities, interfaces, major flows, transactions, side effects, or cross-component contracts.

## Self-Review

Before presenting the design, verify:

- it matches the Spec without silently adding requirements
- important responsibilities, interfaces, signatures, flows, and ownership are explicit and consistent
- transaction, locking, idempotency, and complex logic are defined where correctness requires them
- internal methods represent meaningful stages rather than arbitrary extraction
- implementation freedom remains for non-structural choices
- the document is neither an implementation plan nor near-code
- human review is materially cheaper than reviewing the final implementation

## Approval and Handoff

After completing the Detailed Design, stop and present it for human review. Do not begin the Plan or Implementation without explicit approval.

When changes are requested, update the design, re-check affected signatures, flows, dependencies, and contracts, then present the revision.

After approval, use `superpowers:writing-plans`. The Plan must read both the Spec and Detailed Design:

- Spec: requirements and architecture source of truth
- Detailed Design: implementation-structure source of truth

Then return to the normal Superpowers workflow.
