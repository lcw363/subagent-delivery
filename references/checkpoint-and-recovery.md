# Checkpoint、worktree 与恢复协议

在生成写入 DAG、派发任何 writer、修改工作区、创建 worktree、处理 checkpoint/集成失败，或运行中切换 Skill 版本前读取本文件。

## Commit 模式

- 生成 DAG 前检查用户指令、项目规则、Git 仓库状态和 dirty worktree，固定 `COMMIT_MODE=auto|disabled`。默认 `auto`；用户或项目规则禁止本地 commit 时使用 `disabled`，并强制串行。
- writer 写入前，对计划路径保存 `PRE_TASK_SNAPSHOT`：初始 HEAD、staged/unstaged patch 与 hash、范围内 untracked 原始内容 artifact 的路径/模式/hash、相关 submodule 状态；干净路径记 `CLEAN@START_SHA`。主会话只接收路径和 hash。敏感原文无法安全保存时不得修改其重叠范围。同一 hunk 含无法归属的用户改动时，先在隔离串行 recovery worktree 重建 task-only patch；仍不能安全分离则 `BLOCKED`。
- `auto` 在每个可验证阶段创建 scoped 本地 checkpoint commit，但绝不自动 push。只暂存当前任务可归属的文件或 hunk；禁止 `git add -A`、`git commit -a`、绕过 hooks、修改全局 Git 配置或混入用户改动。
- 无文件变化时记录 `NO_CHANGE`，不创建空 commit。commit 失败时记录 `CHECKPOINT_FAILED`；能生成完整快照且串行继续安全时降级为 `NO_COMMIT_SNAPSHOT`，否则阻塞当前节点及其依赖。
- 已被后继、Integrator 或 Reviewer 使用的 checkpoint 不 amend。修复和后续阶段创建追加 commit。

## Checkpoint 最小字段

所有状态都记录：任务 ID、节点状态、`COMMIT_MODE`、checkpoint 状态、目标工作区或 worktree 绝对路径、初始分支、`START_SHA`/`BASE_SHA`、`PRE_TASK_SNAPSHOT`、实际修改与剩余未提交文件、适用的测试/真实 HTTP 证据摘要、阻塞和残余风险。

按状态补充：

- `COMMITTED`：`PARENT_SHA`、新 `TASK_SHA`/`CHECKPOINT_SHA`/`INTEGRATION_SHA`、本地 ref、commit message、实际提交文件，以及提交后剩余 workspace 状态。
- `NO_CHANGE`：当前 SHA 与没有仓库改动的核对证据。
- `NO_COMMIT_SNAPSHOT`：基线 SHA、staged/unstaged binary patch 路径及 hash、范围内 untracked 文件的路径/模式/内容 hash、相关 submodule 状态。内容保存在受控 artifact 路径；主会话只接收路径和 hash，不复制完整源码或敏感内容。
- `CHECKPOINT_FAILED`：原始错误、已保留的 diff/snapshot/patch 路径、当前 HEAD、恢复或阻塞决定。

恢复 artifact 必须保留可逆所需的精确内容并受控存放；主会话摘要、可见日志和报告必须脱敏。无法安全存放时停止重叠写入。任何 checkpoint 都不能声称未实际执行的测试或 HTTP 验证通过。

## 串行与并行

- 正常串行：任一时刻由唯一活跃 Dev 直接写目标工作区；阶段内可按上下文协议换班，但不创建 Integrator。阶段验证通过后创建并核对 stage checkpoint，完成后才达到 `DEV_PASS`；后继节点从该最新目标状态启动。
- 派发串行 Dev 前若确认目标分支正被其他 writer 持续推进，优先等待其释放，或从固定基线创建隔离 worktree 后再写入。不要让 Dev 在移动中的共享分支上反复接受新基线、重跑验证和重建 checkpoint。
- 正常并行：先固定 `BASE_SHA`，每个 Dev 使用独立 worktree 和本地 `codex/delivery-*` branch/ref，且只写自己的所有权范围。focused tests 和 task commit 完成后返回固定 `TASK_SHA` 并达到 `DEV_READY`。
- Integrator 获取目标工作区独占 writer lease，按依赖顺序消费固定 `TASK_SHA`，核对父 SHA 与写入范围，运行批次验证并创建 integration checkpoint；独立 batch Review/fix loop 对最新 checkpoint 通过后，相关节点才达到 `DEV_PASS`。
- 并行 task ref 和 worktree 保留到集成、Review 与修复结束。只有确认干净且不再用于恢复时才清理 worktree；本地 checkpoint/ref 默认保留并在最终报告列出。

## 资源所有权与收口

- 子会话创建本地服务、进程、端口、fixture、临时目录或 worktree 前，在 `RESOURCE_LEDGER` 记录 `owner_task`、`owner_session`、类型、标识、创建证据、是否共享、消费者、清理条件和清理方式；任务前已存在的资源标记 `PREEXISTING`。
- CodeGraph 按仓库而非节点管理。主任务维护 `CODEGRAPH_REGISTRY[repo_root]`，至少记录服务端点或 PID、启动时间/命令、`owner_task`/`owner_session`、`shared=true`、消费者租约与创建证据。每个节点使用前先查询并复用该唯一可用实例；存在登记实例时不得另启 CodeGraph，也不得重复初始化索引或等价服务。只有主任务明确授权的首个节点可创建实例，创建后立即登记并对后续节点共享。
- 使用 CodeGraph 的节点登记并在收口时释放自己的消费者租约。若节点能证明实例由自己启动、所有消费者已释放且不再用于恢复或后继节点，则在收口时主动关闭该进程并记录 `CLEANED`；否则记录 `RETAINED_SHARED` 或 `RETAINED_FOR_RECOVERY`。不能证明 PID、启动时间、命令和所有权一致时，不终止进程。
- 节点收口时，只清理能证明由当前节点创建、`shared=false`、没有后续消费者且不再用于恢复的资源。清理本地进程时同时核对 PID、启动时间和命令；清理 fixture、临时目录或 worktree 时核对创建标记和路径。
- 不终止共享 Codex、MCP、CodeGraph、Docker daemon 或其他任务的进程，也不清理仍被测试、Review、恢复或后继节点使用的资源；上述 CodeGraph 条件满足时关闭的是已无消费者的自建实例，不是共享实例。
- 无法证明所有权或安全清理条件时不自动处理，只记录标识、当前状态、潜在占用和建议的人工清理方式。
- 清理结果记录 `CLEANED|RETAINED_SHARED|RETAINED_FOR_RECOVERY|OWNERSHIP_UNKNOWN|CLEANUP_FAILED`；清理失败不能伪装成成功，也不能用破坏性命令强制收口。

## 失败与退化

1. 并行前预检 Git、固定基线、worktree 写入、本地 ref 与目标工作区集成条件。任一条件不成立，保存错误并从最新安全 checkpoint 自动回退串行；只有用户明确禁止串行时才 `BLOCKED`。
2. 部分 worktree 创建失败时，不再启动新的并行 writer。已运行任务先停止可安全中断的工作并保存可恢复 checkpoint；随后释放 writer、按依赖集成可用结果，再把剩余节点串行重排。
3. worktree 失败 checkpoint 至少记录 worktree 路径、基线 SHA、实际修改文件、patch/snapshot 路径和测试证据。没有可恢复证据时不能把节点标为通过。
4. 实际文件或语义重叠时停止后启动的冲突 writer，保留其 diff；先完成并集成前一个任务，再从最新目标 checkpoint 串行重启后一个任务。
5. `TASK_SHA` 无法干净集成时，不自动解决语义冲突。先保留 commit、ref、patch 与冲突证据，再执行与当前集成方式对应的安全 abort，并核对 HEAD、index 和 workspace 已回到预集成状态；核对成功后释放 writer lease，并从最近已验证 checkpoint 串行重做。abort 失败或状态不一致时 `BLOCKED`，禁止用 reset 硬恢复。
6. 串行任务通常不建 worktree；仅当原 Dev 中断、目标 dirty 状态无法安全归属，且从最后 checkpoint 重建是最小安全恢复方式时，允许临时专用 recovery worktree。恢复后仍只有一个 writer，并在 checkpoint 中说明例外。

## 运行中升级与回退

- 已经运行的子会话不会自动加载新版 Skill。让当前 writer 到达最近的安全 checkpoint 边界；从下一节点启动 fresh 子会话并读取最新版规则。若旧规则会破坏单写者、数据或发布边界，立即停止受影响 writer并保存恢复证据。
- 版本切换 checkpoint 要记录当前 Skill 版本/来源、writer lease、commit 模式、固定 ref、artifact 路径和下一 DAG 状态；不要在漂移中的阶段切换审查基线。
- 因上下文健康触发的阶段内换班也必须先固定并核对 checkpoint/snapshot，且继承原 timer key 和绝对 deadline；默认节点时钟尚未触发时继承 `PENDING_FIRST_IMPLEMENTATION`。换班细节遵循 `context-rotation.md`，不能借此重置阶段或 Dev 收口时钟。
- 回退前检查依赖该 checkpoint 的后续阶段。`auto` 模式按反向依赖顺序优先创建可审计的 revert commit；`disabled` 模式使用保存的反向 patch。未经明确授权不执行破坏性 reset、rebase 或强制清理。

完成标准：每个交接点都有可定位、可核验、可恢复的固定状态，且没有把无关改动或敏感内容带入 checkpoint。
