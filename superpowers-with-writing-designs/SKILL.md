---
name: superpowers-with-writing-designs
description: Use only when explicitly invoked by the human after a Superpowers spec has been approved and before writing the implementation plan.
---

# Superpowers with Writing Designs

## Overview

Add an optional Detailed Design stage between an approved Superpowers spec and `writing-plans`.

This skill is a **manual extension to Superpowers**. It does not modify, replace, or override any existing Superpowers skill.

Normal flow:

```text
superpowers:brainstorming
        ↓
approved spec
        ↓
[human explicitly invokes this skill]
        ↓
detailed design
        ↓
human review / approval
        ↓
superpowers:writing-plans
        ↓
normal Superpowers implementation workflow
```

Core goal:

> Resolve important implementation-structure decisions before large amounts of code are generated, so humans can review classes, functions, responsibilities, and interactions cheaply and AI implementation drift is reduced.

## Manual Trigger Only

Use this skill **only when explicitly requested by the human**.

Do not automatically invoke it because:

* the task is large
* Services are involved
* multiple modules are involved
* the task is architectural
* significant code generation is expected

Even when Detailed Design would clearly be useful, the human must explicitly invoke this skill.

This skill must not alter the normal Superpowers workflow.

## When to Use

Typical invocation:

```text
The spec is approved.

Use superpowers-with-writing-designs
to create the Detailed Design.

I will review the design before we proceed to writing-plans.
```

An approved spec must already exist.

If the spec has not been approved, return to the normal Superpowers brainstorming/spec workflow rather than creating a Detailed Design prematurely.

## Inputs

Before designing, read:

1. the approved spec
2. relevant existing implementation
3. existing classes, functions, and interfaces
4. established architecture and repository patterns
5. `AGENTS.md`, project conventions, and relevant instructions

The spec is the source of truth for requirements and architecture.

Detailed Design must not silently change an approved spec.

If the spec conflicts with the existing architecture or a sound implementation structure, surface the conflict explicitly rather than silently redefining the requirement.

## Output

Save the design to:

```text
docs/superpowers/designs/YYYY-MM-DD-<feature>-design.md
```

Start with:

```markdown
# <Feature> Detailed Design

**Spec:** `<approved-spec-path>`

## Design Goal

<What implementation structure this design intends to lock down and why.>
```

## Responsibility of Detailed Design

Detailed Design answers:

> **What should the code look like structurally?**

It primarily defines:

* class and component responsibilities
* component boundaries
* public interfaces
* important method signatures
* key internal stage methods
* major call flows
* data flow
* control flow
* transaction boundaries
* concurrency and locking
* side-effect ownership
* cross-component dependencies
* important test seams
* non-trivial algorithms

Detailed Design does not define:

* implementation order
* TDD steps
* commit structure
* complete function bodies
* ordinary local helpers
* mechanical CRUD implementation

Those belong to `writing-plans` and implementation.

## Design Depth

Design only **design-significant structure**.

A decision belongs in Detailed Design when changing it would materially affect one or more of:

* responsibility boundaries
* public APIs
* data flow
* control flow
* transaction or concurrency behavior
* cross-component interfaces
* side-effect ownership
* testability

Otherwise, leave the decision to implementation.

## 1. Component Responsibilities

For each important component, define what it owns and does not own.

Example:

```markdown
## PurchaseOrderService

### Responsibility

Owns:
- purchase-order write operations
- business-rule validation
- status transitions
- consistency between orders and order items

Does not own:
- HTTP request parsing
- response serialization
- list/query presentation
```

Responsibility boundaries should make it easy to determine:

* where logic belongs
* where logic does not belong
* whether a Service has grown too large
* whether multiple abstraction levels are mixed together

## 2. Public Interfaces

Define exact public interfaces for all design-significant components.

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

Public interfaces should normally specify:

* method name
* parameters
* parameter types
* return type

Public interfaces are part of the design contract.

Implementation must not silently redesign them.

## 3. Key Internal Stage Methods

For Services and other orchestration-heavy components, define important private/internal stage methods when they establish meaningful structural boundaries.

Example:

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

Only define internal methods that materially shape the implementation.

Do not pre-design leaf helpers such as:

```python
_to_decimal()
_build_item_map()
_normalize_string()
_extract_ids()
_has_changed()
```

unless the helper itself represents an important domain abstraction.

## 4. Method Contracts

For important methods, document their contracts.

Recommended structure:

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

Focus on ownership and behavioral boundaries rather than implementation details.

Example:

```text
_sync_items()

Responsibilities:
- synchronize order items from the complete incoming item collection
- create new items
- update existing items
- delete existing items absent from the incoming set

Does not:
- validate order-status permissions
- modify the order header
- trigger inventory business side effects
```

## 5. Call Flow

Service public methods should communicate the high-level use-case flow.

Example:

```text
update()
├── _get_for_update()
├── _validate_update()
├── _apply_order_fields()
├── _sync_items()
└── _record_audit()
```

An experienced reviewer should normally be able to understand the major business flow from the public method and its stage-level call chain without reading every helper.

If the call flow itself is difficult to understand, reconsider the responsibility boundaries or abstraction levels.

## Service Design Principles

Service public methods should primarily express high-level business stages.

Prefer flows such as:

```text
load
→ normalize
→ validate
→ mutate
→ synchronize
→ side effects
→ audit
```

Do not mechanically extract helpers merely to reduce line count.

A private method should represent:

* a business stage
* a clear responsibility
* a stable abstraction boundary

rather than merely a few lines of code.

## 6. Cross-Component Flow

When a use case crosses multiple components, document the important interactions.

Example:

```text
PurchaseOrderViewSet
    ↓
PurchaseOrderService.update()
    ├── PurchaseOrderQuerySet
    ├── InventoryService
    └── AuditService
```

Clarify:

* who initiates the call
* who owns business validation
* who owns state changes
* who owns side effects
* which components should not directly depend on each other

Use a sequence diagram only when ordering, branching, or interactions are difficult to understand from a simple call chain.

Prefer a concise call tree whenever possible.

## 7. Transaction and Concurrency

When relevant, explicitly define:

```text
Transaction boundary:
- the entire PurchaseOrderService.update use case runs inside transaction.atomic

Locking:
- the order is loaded using select_for_update

Idempotency:
- ...

Concurrency assumptions:
- ...
```

Do not leave transaction boundaries to implementation-time interpretation when they materially affect correctness.

## 8. Complex Logic

Use pseudocode only when the algorithm itself is an important design decision.

Good candidates include:

* state transitions
* reconciliation
* allocation
* collection synchronization
* matching
* batching
* retries
* idempotency
* dependency ordering

Example:

```text
existing_items = index existing items by id
incoming_items = incoming request items

for each incoming item:
    if id exists:
        validate ownership
        update
    else:
        create

delete existing items absent from incoming set
```

Do not write pseudocode for ordinary CRUD.

Prefer:

* signatures
* contracts
* call flows

over function-body-level pseudocode.

## 9. Implementation Freedom

Detailed Design must deliberately preserve reasonable implementation freedom.

Do not normally constrain:

* local variable names
* small helper methods
* equivalent ORM expressions
* ordinary data transformations
* loop implementation
* temporary internal structures
* private implementation details that do not change the approved design

The goal is not to write the code twice.

The goal is:

> Lock important structure while leaving local implementation choices open.

## Design Contract

Once approved by the human, the Detailed Design becomes the implementation-structure contract.

Implementation must not silently change:

* component responsibilities
* public interfaces
* documented key internal boundaries
* major call flows
* transaction boundaries
* side-effect ownership
* cross-component interfaces

Implementation plans may repeat these definitions for task-local context, but must not redesign them.

## Design Drift

If a problem with the Detailed Design is discovered during planning or implementation, do not silently modify the design and continue.

Report:

```text
Design Conflict

Current design:
<current approved design>

Problem:
<why the design is problematic during implementation>

Proposed change:
<recommended design adjustment>

Impact:
<affected classes, methods, tasks, and interfaces>
```

If the change affects an approved structural decision, update the Detailed Design and obtain human approval before continuing with the affected work.

## Relationship to the Spec

The Spec defines:

```text
What should be built?
Why?
What are the requirements?
What is the architecture?
What are the business rules?
```

Detailed Design defines:

```text
What should the implementation structure look like?
Which classes own which responsibilities?
What interfaces exist?
How do major methods collaborate?
```

Do not duplicate large sections of the Spec.

Reference relevant Spec sections instead.

## Relationship to the Plan

The Plan defines:

```text
Which files change?
What tasks exist?
In what order?
What tests are written?
What implementation steps are executed?
What gets committed?
```

Therefore:

```text
Spec
    ↓ requirements / architecture

Detailed Design
    ↓ implementation structure

Plan
    ↓ execution steps

Implementation
```

The Plan may copy exact signatures from the Detailed Design so that task-local executors receive sufficient context.

It must not redesign approved interfaces or structural decisions.

## Output Template

Scale this template to the actual complexity:

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

Do not create empty sections merely to satisfy the template.

## Reviewability

A major goal of Detailed Design is:

> Human review of the design should be substantially cheaper than review of the final implementation.

Prefer:

```text
signatures > implementation code
contracts > long prose
call trees > lengthy explanations
selective pseudocode > full pseudocode
```

Use these two tests.

### Too Shallow

The design is too shallow if the implementer still needs to decide:

* which public methods a Service should expose
* how parameters are passed
* how classes should be split
* which key methods should exist
* who calls whom
* where the transaction boundary belongs
* which component owns side effects

### Too Detailed

The design is too detailed if it already specifies:

* every private helper
* exact loop implementations
* every ORM expression
* every local variable
* near-complete function bodies

## Self-Review

Before presenting the design, verify:

* [ ] The design matches the approved Spec.
* [ ] Every important component has a clear responsibility.
* [ ] Public interfaces are explicitly defined.
* [ ] Important signatures are exact and internally consistent.
* [ ] Service public methods expose readable high-level use-case flows.
* [ ] Key internal stage methods have meaningful responsibilities.
* [ ] Cross-component ownership is clear.
* [ ] Transaction and locking boundaries are explicit where relevant.
* [ ] Complex algorithms are specified where necessary.
* [ ] Unnecessary leaf helpers have not been pre-designed.
* [ ] The document has not turned into an implementation plan.
* [ ] The document has not turned into near-code.
* [ ] Human review is meaningfully cheaper than reviewing the final implementation.

## Approval Gate

After completing the Detailed Design, stop.

Present it to the human for review.

Do not:

* automatically begin implementation
* automatically execute the plan
* proceed to the next phase without explicit approval

If changes are requested:

1. update the design
2. re-check affected signatures
3. re-check affected call flows
4. re-check cross-component dependencies
5. present the revised design for approval

## Handoff

After the human explicitly approves the Detailed Design:

```text
Detailed Design approved.

Next use superpowers:writing-plans.

The plan should read both:

- the approved spec
- the approved detailed design

The Spec is the source of truth for requirements and architecture.
The Detailed Design is the source of truth for implementation structure.
```

Then end this skill and return to the normal Superpowers workflow.
