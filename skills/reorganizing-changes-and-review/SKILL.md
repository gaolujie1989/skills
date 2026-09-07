---
name: reorganizing-changes-and-review
description: Use only when explicitly invoked by the human to reorganize a large commit returned to the working tree into logical commits and produce an evidence-based review handoff.
---

# 重组大批代码改动并生成审核材料

## 目标

以原始需求为准，扫描全部改动，将其重组为少量、逻辑完整、可独立验证的 commits；随后生成 Review Brief，帮助人工理解整体改了什么、如何实现、是否偏离需求。

事实优先级：用户当前要求 > 已确认需求/Plan > 项目架构与约定 > 代码、测试与原提交。现有实现和测试只是证据，不能反推或覆盖需求。

## 硬规则

- 不丢弃或覆盖工作区内容；禁止在用户工作区执行 `git reset --hard`、`git checkout -- .`、`git restore .`。
- 不使用 `git add .`、`git commit -am`；按明确路径或 hunk 暂存。
- 不按目录、文件类型或“模型/API/测试”机械拆分；按完整业务行为或可观察能力纵向拆分。
- 行为实现、对应测试及必要迁移进入同一 commit。
- 每个 commit 只有一个目的，且不依赖未来 commit 才能编译、迁移或通过相关验证。
- 不夹带无关重构、格式化、生成文件或用户原有改动。
- 不伪造开发过程；测试不得被单独提前提交来假装 TDD。
- Commit message 解释逻辑变化，不逐文件复述 diff；整个分支的解释放入 Review Brief。

## 工作流

### 1. 建立基线

读取完整状态和全部 patch：

```bash
git status --short
git branch --show-current
git rev-parse HEAD
git diff --stat
git diff --name-status
git diff --check
git ls-files --others --exclude-standard
```

若已有 staged 内容，先检查 `git diff --cached`，不得擅自清空 index。必须阅读所有 diff、未跟踪文件、相关需求文档和关键调用链，不能只看文件名或统计。

记录 reset 前基线 `BASE`。若 `ORIG_HEAD` 或 reflog 明确指向原始大提交，记录为 `SOURCE_COMMIT`，仅用于最终树比较，不把原提交信息当作需求。

### 2. 还原需求基线

输出：

- 目标行为与成功标准
- 非目标和明确约束
- 外部契约、数据变化与兼容要求
- 需求中未明确、实现自行假设的事项
- 必须执行的测试、lint、类型检查、构建和迁移检查

先在仓库和任务上下文中查找缺失信息。只有歧义会实质改变行为或 commit 边界时才询问用户。

### 3. 建立改动清单

逐文件、逐 hunk 标注：对应需求、行为作用、依赖、测试证据、迁移/配置、纯重构、超出需求或不确定项。追踪 API、schema、服务、任务、权限和数据流，确定哪些改动必须一起提交。

发现超出需求或无法归属的改动，不静默纳入；默认保留未提交并报告。

### 4. 生成 Commit Map

通常组织为 4–10 个绿色里程碑，不强行凑数量。每项包含：

1. 中文 Conventional Commit 标题
2. 单一目的和可观察结果
3. 精确文件或 hunk
4. 前置依赖
5. 独立验证命令
6. 人工审核重点

优先顺序：可独立成立的基础能力 → 带测试的完整行为切片 → 独立有价值的重构或文档。若某项只有结合后续 commit 才成立，合并边界。

### 5. 逐批暂存和提交

```bash
git add <明确路径>
git add -p <混合多个提交内容的文件>
git diff --cached --stat
git diff --cached
git diff --cached --check
```

新文件需要拆 hunk 时先 `git add -N <path>`。无法形成有效中间状态时合并，不强拆。

Commit message 推荐格式：

```text
<type>(<scope>): <可观察结果>

Why:
- 解决的问题或需求

What:
- 关键行为变化
- 重要实现选择

Verification:
- 实际执行的验证命令
```

只有文件职责隐蔽或存在非显然副作用时才提具体文件；禁止把全部文件清单复制进 message。

### 6. 在干净树验证每个 commit

主工作区还包含后续未提交改动，因此其中的测试结果不能证明当前 commit 独立可用。

每次提交后，在临时 detached worktree 检出当前 `HEAD`，执行该 commit 的验证命令。可复用临时 worktree，但禁止在用户工作区 hard reset。

验证失败时停止后续提交，调整边界或做需求范围内的最小修正。无法干净验证时必须标记限制，不得声称已独立通过。

### 7. 最终核验

运行完整相关检查，并输出：

```bash
git status --short
git log --oneline --decorate -n <本次提交数>
git diff "$BASE"..HEAD --stat
```

若存在 `SOURCE_COMMIT`，执行：

```bash
git diff --exit-code "$SOURCE_COMMIT" HEAD
```

无差异表示仅重组历史；有差异必须逐项说明是有意修正、遗漏还是额外改动。

## Review Brief

重组和验证完成后，必须生成面向人工审核的 Review Brief。默认作为最终输出或 MR/PR 描述，不自动提交进业务 commits。

必须包含：

1. **审核结论摘要**：符合需求、基本符合但需确认、存在偏离、或信息不足。
2. **需求基线**：目标、成功标准、非目标、约束和实现假设。
3. **整体实现说明**：改了什么、为什么这样改、主要调用链/数据流、外部接口与数据库变化。
4. **需求追踪矩阵**：每项需求对应实现位置、commit、测试证据和状态；状态只能是已覆盖、部分覆盖、未覆盖、超出需求、需人工确认。
5. **Commit 导读**：每个 commit 的目的、可观察变化、关键实现选择、边界理由、依赖、审核重点和验证结果。
6. **文件级索引**：覆盖全部变更文件或合理文件组，记录原职责、本次改动、改动原因、对应需求和风险。核心业务文件必须单独说明；迁移、锁文件和生成文件可合并。
7. **偏离与额外改动**：需求不一致、AI 自行假设、超范围重构、架构妥协及保留未提交内容。没有发现也明确写“未发现”。
8. **风险与人工判断点**：业务规则、兼容性、迁移安全、事务、权限、并发、幂等、隐藏副作用、过度设计及测试有效性，按风险排序并定位到 commit、文件或符号。
9. **验证证据**：实际命令和结果；未执行项必须写“未执行”。

Review Brief 必须解释设计和行为，不得只是 diff 摘要，也不得因为测试通过就断言符合需求。

## 独立审核交接

建议由新的 reviewer 上下文重新读取原始需求、完整 `BASE..HEAD` diff、commit 历史、测试结果和 Review Brief，独立判断：

- 是否满足、遗漏或偏离需求
- 是否存在超范围实现或过度设计
- commit 边界是否真实完整且可验证
- Review Brief 是否准确反映代码
- 哪些业务或架构决策必须由人工确认

重组者的结论只能作为导航，不能替代独立审核。

## 红旗

出现以下想法时停止并重做 Commit Map 或 Review Brief：

- “按文件夹拆就行”
- “测试最后统一提交”
- “先全部 add 再挑”
- “中间失败没关系”
- “顺便重构无关代码”
- “工作区测试通过，所以当前 commit 独立通过”
- “把每个文件说明都塞进 commit message”
- “测试通过，所以一定符合需求”

## 输出顺序

执行前：`需求基线` → `改动清单` → `Commit Map` → `风险/歧义`。

执行后：`已创建 commits` → `逐 commit 验证` → `最终验证` → `Review Brief` → `保留改动` → `与 SOURCE_COMMIT 的树差异`。
