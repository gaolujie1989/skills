---
name: reviewing-large-changes
description: Use when reviewing a large, AI-generated, or cross-module change where a flat diff is insufficient to understand intent, dependency order, symbol relationships, and architectural impact.
---

# Reviewing Large Changes

## 目标

帮助人工 Reviewer **理解代码，而不是替人工做最终判断**。

核心流程：

```text
Walkthrough
→ 建立 PR 宏观 Mental Model
→ Change Stack
→ 按业务与代码依赖顺序阅读
→ Code Peek / Call Chain
→ 理解关键函数与执行路径
→ 人工判断设计是否正确
```

不要一开始逐文件、逐行检查问题。先理解系统，再 Review。

---

## 1. 确定 Review 上下文

先确定：

* diff 的 base 与 HEAD；
* 本次需求、设计文档、Plan、验收标准；
* `AGENTS.md` 及相关项目规范；
* changed files / hunks / commits。

优先使用：

```bash
git diff --stat <base>...HEAD
git diff --name-status <base>...HEAD
git diff --find-renames <base>...HEAD
```

需求或设计无法确定时，可以从代码推断，但必须明确标记为 **推断**，不能当成需求事实。

---

## 2. Walkthrough：先建立宏观 Mental Model

不要立即报告普通代码问题。

先回答：

### Intent

* 为什么要改？
* 最终增加或改变了什么业务行为？

### Scope

* 涉及哪些模块、数据模型、API、任务、前端或外部系统？
* 哪些属于核心变化，哪些只是配套变化？

### Flow

描述主要执行路径，例如：

```text
API
→ Serializer
→ Service
→ Model
→ External API
→ Audit Log
```

复杂流程可生成 Mermaid。

### State / Contract

说明：

* 数据结构变化；
* 状态转换；
* API / event / function contract 变化；
* transaction、权限、异步边界。

### Verification

说明新增或修改的测试分别证明什么。

最后用 5～10 条内容给 Reviewer 建立整个 PR 的 Mental Model。

---

## 3. Change Stack：重新组织 Diff

**禁止按照文件名顺序 Review。**

把 changed hunks 按一个完整业务 concern 分成若干 **Cohort**。

例如：

```text
Cohort A：采购单审核
Cohort B：库存同步
Cohort C：审计日志
```

每个 Cohort 再按照依赖关系拆成 **Layer**。

通常优先顺序：

```text
Schema / Migration
        ↓
Model / Domain
        ↓
Service / Business Logic
        ↓
API / Adapter / Task
        ↓
Call Sites / Consumers
        ↓
Frontend
        ↓
Tests
```

根据实际架构调整，不要机械套模板。

输出：

| Layer | Purpose | Code Range      | Depends On | Reviewer 应理解 |
| ----- | ------- | --------------- | ---------- | ------------ |
| 1     | 数据模型    | `x.py:20-80`    | -          | 新增了什么状态和约束   |
| 2     | 审核服务    | `y.py:40-150`   | L1         | 核心业务规则如何实现   |
| 3     | API     | `z.py:30-70`    | L2         | 谁触发业务流程      |
| 4     | Tests   | `test_x.py:...` | L1-L3      | 哪些行为被证明      |

引用代码必须尽量使用 `file:line-range`。

---

## 4. Layer Deep Dive

进入某个 Layer 后，只研究：

1. 本 Layer 修改的 symbol；
2. 理解它们必须知道的直接上下游代码。

不要无边界读取整个仓库。

对重要 function / class 给出：

```text
symbol
├─ Purpose
├─ Inputs
├─ Outputs
├─ Side Effects
├─ Callers
├─ Callees
├─ State / Invariants
└─ Error / Transaction boundaries
```

重点解释**为什么存在以及如何参与业务流程**，不要逐行翻译代码。

---

## 5. Code Peek / Call Chain

Reviewer 可以随时要求：

```text
peek <symbol>
callers <symbol>
callees <symbol>
trace <entrypoint>
open <layer>
next
previous
```

查找 symbol 时：

1. 优先使用可用的 language-aware / code graph / LSP 工具；
2. 没有时使用 repository search、`rg`、`git grep`；
3. 必要时读取未修改代码，因为调用链经常跨出 diff；
4. 区分“真实调用关系”和“根据命名推测的关系”。

`trace` 应输出一条可阅读的执行路径：

```text
POST /orders/{id}/approve
→ PurchaseViewSet.approve()
→ PurchaseApprovalService.execute()
→ validate_status()
→ update_inventory()
→ AuditLog.create()
```

每一步说明：

**调用目的 + 关键状态变化 + side effect。**

---

## 6. 人工设计判断

只有完成 Mental Model 和主要调用链后，才进入设计 Review。

Codex 不应直接替 Reviewer 给出“设计正确”结论。

输出 **Design Judgment Sheet**：

### Requirement

* 是否覆盖需求？
* 是否存在遗漏规则？

### Architecture

* 职责边界是否合理？
* dependency direction 是否合理？
* 是否存在不必要抽象或 YAGNI 问题？

### Data & State

* invariant 是否得到维护？
* transaction / concurrency / idempotency 是否合理？

### Interface

* API / event / function contract 是否清晰？
* 是否存在兼容性问题？

### Failure

* 错误路径、权限、重试、partial failure 如何处理？

### Tests

* 测试真正证明了哪些行为？
* 哪些关键风险没有证据？

### Open Questions

只列出仍需要人工判断的问题。

---

## 输出纪律

首次执行只输出：

```text
# Walkthrough
# Mental Model
# Change Stack
# Recommended Reading Order
# Unknowns
```

之后按 Layer 深入。

不要一次把整个大 PR 全部解释完。

不要把“理解代码”和“寻找 bug”混在第一阶段。

发现明显 P0/P1 风险可以立即标记，但普通 findings 留到 Mental Model 建立完成之后。

最终目标不是：

> “AI 已经 Review 完了。”

而是让人工 Reviewer 能够不用重新从头读代码，就回答：

> 这个功能为什么存在？
> 数据和控制流怎么走？
> 每个关键函数负责什么？
> 谁调用谁？
> 状态在哪里改变？
> 为什么采用这个设计？
> 我是否认可这个设计？
