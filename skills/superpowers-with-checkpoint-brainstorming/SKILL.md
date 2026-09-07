---
name: superpowers-with-checkpoint-brainstorming
description: Use only when explicitly invoked by the human to brainstorm with Superpowers while reducing interruptions from routine choices, repeated clarification, and section approvals.
---

# Superpowers with Checkpoint Brainstorming

## Overview

Preserve Superpowers' Spec -> Plan -> Execute -> Review -> Verify -> Finish discipline while adapting brainstorming to autonomous decisions and checkpoint supervision.

Move human attention from routine implementation choices to high-value, user-owned decisions. Everything not explicitly overridden here follows the original Superpowers skill.

## Brainstorming: Prefer Checkpoint Supervision

**REQUIRED SUB-SKILL:** Use `superpowers:brainstorming`.

Preserve its scope classification, design quality, written-spec requirements, and final pre-implementation approval gate. Override its question cadence and per-section approval behavior as described below.

### Core Principle

**A recommendation is a decision, not a question.**

When one option is clearly preferable, select it, record it, and continue. Do not present the recommended option and then ask the user to select it.

### Decision Policy

Classify each unresolved choice by an observable condition:

| Class | Condition | Action |
| --- | --- | --- |
| `AUTO` | One option is clearly best under existing constraints, conventions, YAGNI/KISS, or engineering practice; the choice is an implementation detail, reversible, and does not change requirement semantics. | Choose it, record the decision and rationale, and continue. |
| `ASSUME` | Information is missing, but a safe, conventional, reversible default exists. | Adopt the default, record the assumption and impact, and continue. |
| `ASK` | No option is clearly superior, or the choice depends on product intent, business semantics, scope, compatibility, migration or deletion behavior, security, permissions, significant cost, conflicting requirements, or information only the user has. | Record it for the next Decision Checkpoint. |

The existence of multiple implementation options does not by itself make a decision `ASK`.

### Batch Clarification

Override brainstorming's default one-question-at-a-time behavior.

Continue all design work that is not blocked while accumulating related `ASK` decisions. When unresolved decisions block further design, present them together in one concise Decision Checkpoint. Prefer one meaningful clarification round over a sequence of small interruptions.

### Decision Checkpoint

Before finalizing an architectural spec, summarize:

- recommended decisions made autonomously;
- assumptions made autonomously;
- decisions already made by the user; and
- remaining `ASK` decisions requiring user input.

Use one compact ledger, for example:

| ID | Topic | Choice | Source | Rationale / impact |
| --- | --- | --- | --- | --- |
| D1 | Async processing | Celery | Agent recommendation | Matches existing infrastructure |
| A1 | Page size | 50, configurable | Agent assumption | Low-impact, reversible default |
| D2 | Cancellation inventory | Restore immediately | User decision | Required business behavior |

If no `ASK` decisions remain, state that no blocking user decisions remain and proceed directly to the spec; do not request another intermediate confirmation. If `ASK` decisions remain, ask them together, incorporate the answers, and then write the spec.

### Design and Spec Approval

For the architectural path, override approval after each design section. Develop architecture, components, data flow, error handling, and testing as one coherent design without stopping after each section merely for confirmation.

The completed spec must contain a concise `Decisions` section for choices that materially constrain implementation. The spec is the durable decision record; do not create a separate long-lived decision document unless the project explicitly requires one.

Present the complete spec for one final user review before planning. Incorporate requested changes in one pass where practical, re-check the spec, and retain the mandatory final approval gate before implementation.

For bounded work, apply the same `AUTO` / `ASSUME` / `ASK` policy and batch clarification, but keep the short in-chat design and approval gate required by `superpowers:brainstorming`.

## Common Mistakes

- Asking the user to choose the option the agent has already recommended.
- Treating every missing detail as `ASK` instead of using a safe reversible assumption.
- Asking several related business questions across separate turns.
- Skipping the final spec approval because intermediate confirmations were removed.

## Boundaries

This skill changes brainstorming's decision policy, question cadence, and per-section approval behavior. It preserves scope classification, design quality, written-spec requirements, and the final approval gate.

It does not change implementation task granularity, TDD strategy, executor selection, review, verification, or branch completion. Outside the overrides above, follow Superpowers unchanged.

This skill can be used independently or together with `superpowers-with-matt-tdd`.
