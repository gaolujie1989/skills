---
name: superpowers-with-strong-agents
description: Use when using Superpowers for brainstorming, writing implementation plans, or executing approved plans with a strong coding agent.
---

# Adapting Superpowers for Strong Agents

## Overview

Preserve Superpowers' Spec -> Plan -> Execute -> Review -> Verify -> Finish discipline while adapting three points for strong coding agents:

1. Brainstorming uses autonomous decisions and checkpoint supervision.
2. Implementation plans use larger semantic tasks.
3. Execution uses Matt's `tdd` strategy.

The goal is not to remove human oversight. Move human attention from routine implementation choices to high-value, user-owned decisions. Everything not explicitly overridden here follows the original Superpowers skill.

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

## Writing Plans: Detailed Content, Larger Tasks

**REQUIRED SUB-SKILL:** Use `superpowers:writing-plans`.

Do not split work mechanically into 2-5 minute actions. Each task should be a semantically complete, verifiable implementation increment suitable for one commit. A strong coding agent may complete several naturally related steps inside one task.

Plans still specify goals and constraints, affected files, interfaces and dependencies, key implementation requirements, verification, and commit boundaries.

Do not create separate tasks merely for:

- writing a failing test;
- verifying RED;
- writing the minimal implementation; or
- verifying GREEN.

Those are internal steps of one implementation task. Use 5-15 minutes only as a rough scale; semantic, dependency, verification, and commit boundaries take precedence over time.

### Test Seams

For tasks requiring TDD, identify the valuable test seams: which public interfaces verify which behaviors. Do not plan tests for every internal function or implementation detail.

Approval of the full plan also approves its listed test seams.

## Execution: Use Matt TDD

Whether execution uses `superpowers:executing-plans` or `superpowers:subagent-driven-development`, preserve that executor's workflow.

**REQUIRED SUB-SKILL:** Use `tdd`, replacing `superpowers:test-driven-development` for implementation tasks.

Within each task:

- test public behavior rather than implementation details;
- test only at approved seams;
- proceed in vertical slices: one behavior test -> minimal implementation -> next behavior;
- observe RED before GREEN;
- avoid duplicate tests for private helpers, internal collaborators, or trivial layers merely for coverage; and
- leave broader refactoring to review rather than expanding the RED -> GREEN loop.

Do not re-confirm test seams already approved in the plan. If implementation requires a material new seam not present in the approved plan, confirm that seam before adding it.

## Common Mistakes

- Asking the user to choose the option the agent has already recommended.
- Treating every missing detail as `ASK` instead of using a safe reversible assumption.
- Asking several related business questions across separate turns.
- Skipping the final spec approval because intermediate confirmations were removed.
- Splitting RED and GREEN into separate plan tasks.
- Replacing an executor's review or verification workflow rather than only its TDD strategy.

## Boundaries

This skill does not choose between single-agent execution and subagent-driven development. It does not replace brainstorming, writing-plans, executing-plans / subagent-driven-development, requesting-code-review, systematic-debugging, verification-before-completion, or finishing-a-development-branch.

Outside the three overrides above, follow Superpowers unchanged.
