---
name: reorganizing-changes
description: Use when AI-generated or manually accumulated work was committed as one large commit, then returned to the working tree with git reset --mixed, and must be reconstructed into reviewable logical commits.
---

# 将大批未提交改动重组为逻辑提交

## 目标

根据需求，而不是目录或文件类型，把全部改动重组为少量、可审查、可回退、可独立验证的 commits。

事实来源优先级：用户当前要求 > 已确认的需求/Plan > 项目架构与约定 > 代码、测试和 diff。生成代码与测试只是实现证据，不能反过来定义需求。

## 硬规则

* 不丢弃、覆盖或回退工作区内容；禁止对用户工作区执行 `git reset --hard`、`git checkout -- .`、`git restore .`。
* 不使用 `git add .` 或 `git commit -am`；按明确路径或 hunk 暂存。
* 不按“模型/API/测试”“前端/后端”等技术层机械拆分；优先按完整业务行为或可观察能力纵向拆分。
* 行为实现与对应测试进入同一 commit；迁移与依赖它的最小可运行代码进入同一 commit。
* 每个 commit 只有一个目的，且不依赖后续 commit 才能编译、迁移或通过相关检查。
* 不夹带无关重构、格式化、生成文件或用户原有改动。
* 不使用 amend、rebase 或 squash；从当前 reset 后的工作区顺序创建新 commits。
* 这是重建可审查历史，不伪造开发过程；不要把测试单独提前提交来假装 TDD。

## 工作流

### 1. 建立基线

先运行并读取完整输出：

```bash
git status --short
git branch --show-current
git rev-parse HEAD
git diff --stat
git diff --name-status
git diff --check
git ls-files --others --exclude-standard
```

若存在 staged 改动，先检查 `git diff --cached`，不要直接清空 index。必须阅读全部 patch 和未跟踪文件内容，不能只看统计或文件名。

若 `ORIG_HEAD` 或 reflog 明确指向被 reset 的大提交，记录为 `SOURCE_COMMIT`，仅作为最终树快照和辅助证据，不把原提交消息当作需求。

### 2. 还原需求意图

读取当前任务要求、Plan、相关文档和既有实现模式，输出：

* 目标行为与成功标准
* 非目标
* 涉及模块和外部契约
* 需要验证的测试、lint、类型检查、构建或迁移
* 会影响行为或 commit 边界的歧义

需求缺失时先在仓库中查找；只有歧义会实质改变行为或提交边界时才询问用户。

### 3. 建立改动清单

逐文件、逐 hunk 标注：所属需求、行为作用、依赖、对应测试、迁移/配置、纯重构、无关或不确定。追踪调用链、schema、API、任务和测试依赖，确定哪些改动必须一起提交。

发现超出需求的改动时，不静默纳入；保留未提交并报告。

### 4. 生成 Commit Map

通常组织为 4–10 个绿色里程碑，不强行凑数量，也不拆成单文件或分钟级 commits。每项必须包含：

1. 中文 Conventional Commit 标题
2. 单一目的和可观察结果
3. 精确文件或 hunk
4. 前置依赖
5. 独立验证命令

优先顺序：可独立成立的基础能力 → 带测试的完整行为切片 → 独立有价值的无行为重构/文档。若任一 commit 只有结合后续改动才成立，合并边界。

先展示 Commit Map。用户已明确要求直接重组并提交时，展示后继续执行；只在存在实质歧义或风险时暂停。

### 5. 逐批暂存、检查、提交

每个 commit：

```bash
git add <明确路径>
git add -p <包含多个提交内容的文件>
git diff --cached --stat
git diff --cached
git diff --cached --check
```

新文件需要按 hunk 拆分时，可先 `git add -N <path>` 再 `git add -p`。无法形成有效中间状态时合并，不强拆。

提交前确认：staged patch 只对应一个 Commit Map 项；没有半套 API/schema；剩余改动属于后续 commits 或明确保留内容。然后执行：

```bash
git commit -m "feat(scope): 中文说明"
git show --stat --oneline HEAD
```

commit 类型按结果选择 `feat`、`fix`、`refactor`、`test`、`docs`、`chore`，不要按文件类型选择。

### 6. 在干净树中验证每个 commit

主工作区仍含后续未提交改动，因此在那里运行测试只能验证“组合后的工作树”，不能证明当前 commit 独立可用。

每次提交后，在临时 detached worktree 中检出当前 `HEAD` 并执行该 commit 的验证；可复用同一验证 worktree，每次安全地将该临时目录 reset 到新 `HEAD`。禁止在用户工作区使用 hard reset。

验证失败时不要继续提交。重新调整边界，或仅做需求范围内的最小修正。无法进行干净树验证时，明确标记限制，不得声称该 commit 已独立通过。

### 7. 最终核验

运行完整相关检查，并输出：

```bash
git status --short
git log --oneline --decorate -n <本次提交数>
git diff <BASE>..HEAD --stat
```

若记录了 `SOURCE_COMMIT`，执行：

```bash
git diff --exit-code "$SOURCE_COMMIT" HEAD
```

无差异表示只重组了历史；有差异必须逐项说明是有意修正、遗漏还是额外改动。

最终报告：需求摘要、实际 commits、每个 commit 的验证、完整检查、保留的未提交改动和未解决风险。

## 边界判断

* 同一可观察行为及其测试：一起提交。
* 共享基础能力：仅在可复用、独立有效且可验证时单独提交。
* 纯重命名或格式化：仅在能显著降低后续 diff 噪音时独立提交，否则避免。
* 测试基础设施：没有产品行为且可独立复用时可以单独提交。
* 需要未来 commit 才能通过验证：不是有效边界。

## 红旗

出现以下想法时停止并重做 Commit Map：

* “按文件夹拆就行”
* “测试最后统一提交”
* “先全部 add，再从提交里挑”
* “中间失败没关系，最后会修好”
* “顺便把无关代码也重构了”
* “当前工作区测试通过，所以这个 commit 独立通过”

## 输出格式

执行前：`需求基线`、`改动清单`、`Commit Map`、`风险/歧义`。

执行后：`已创建 commits`、`逐 commit 验证`、`最终验证`、`保留改动`、`与 SOURCE_COMMIT 的树差异`。
