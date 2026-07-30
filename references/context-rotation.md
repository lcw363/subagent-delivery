# Dev 上下文健康与换班协议

在长阶段开始前，或 Dev 出现上下文压缩、重复读取、范围增长、记忆漂移等信号时读取本文件。目标是限制单个子会话的上下文长度，同时维持单写者、固定验收标准和可恢复 checkpoint。

## 基本规则

- 同一需求、调用链和所有权范围内的相邻串行阶段，只要上下文健康且能稳定容纳，默认复用当前 Dev；出现明显领域切换、独立高风险边界或下述健康失败时才换 fresh Dev。
- 一个阶段可以由 `Dev-A -> Dev-B` 等多个 fresh Dev 串行接力，但任何时刻只能有一个 writer；换班不是并行开发，也不需要 Integrator。
- 阶段在完成前包含至少两个顺序、可验证的安全 checkpoint，或被 spec/Wayfinder 标为大型阶段时，记为 `LONG_STAGE`；派发前记录一个或多个 `planned_rotation_checkpoint` 作为上下文健康评估点，尚无安全点时记录 `PENDING_SAFE_BOUNDARY` 并在出现首个可恢复点时更新。到达计划点时先执行健康检查；上下文健康且没有领域切换或独立高风险边界时继续复用当前 Dev。
- 换班继承原 `task_id`、阶段验收标准、依赖、workspace、timer key 和已固定的绝对 `deadline_at`；尚未触发默认节点时钟时继承 `PENDING_FIRST_IMPLEMENTATION`。不得借换班扩大范围、重置时限或提前解锁后继节点。
- `compression_count` 仅统计运行时明确可观察的压缩事件，按阶段累计并由接力 Dev 继承；事件不可观察时记为 `UNKNOWN`，不得猜测或伪造。`rotation_count` 只记录已完成的换班次数。
- 用户明确只允许一个 Dev/子会话时仍执行健康检查；到达计划评估点且上下文健康时继续复用，只有命中领域切换、独立高风险边界或其他强制换班条件时才停止继续写入，固定 checkpoint/snapshot 并只返回 `BLOCKED` 等待确认。该分支不进入下方换班顺序、不返回 `HANDOFF_READY`、不退休当前 Dev，也不把 writer lease 授予其他会话；当前 Dev 保持闲置且不得继续生产代码。
- 优先在稳定子目标、focused tests 或 checkpoint 边界换班。处于未解决冲突、半完成迁移、不可安全中断的数据操作或无法恢复的中间状态时，先停止新增范围并恢复到安全边界；无法做到则 `BLOCKED`。

## 触发判断

Dev 和主任务在阶段边界及明显上下文事件后检查以下信号：

1. `LONG_STAGE` 到达 `planned_rotation_checkpoint` 且仍有实质工作：执行上下文健康检查并更新交接 artifact；只有出现领域切换、独立高风险边界或下述健康失败时才换班。
2. 第一次可观察的上下文压缩：记为预警，尽快固定 checkpoint/snapshot，并开始维护交接 artifact；若已有稳定 checkpoint 且剩余工作仍明显较多，可主动换班，但不是强制门禁。
3. 同一阶段第二次可观察的上下文压缩：必须在最近安全边界换班。
4. 无论压缩次数，只要出现任一健康失败就必须换班：
   - 不能准确复述当前验收标准、不做内容、固定 checkpoint、已验证证据和剩余任务；
   - 重复查找已经确认的文件、调用链或决策，且无法从现有 artifact 恢复；
   - 先前约束、测试结果或修改归属出现互相矛盾；
   - 实际范围显著超出派发简报，当前上下文已不能稳定容纳剩余工作。

运行时无法直接报告压缩事件时，将 `compression_count` 记为 `UNKNOWN`，依靠 `LONG_STAGE` 计划点和可观察健康信号，不猜测 token 数。非 `LONG_STAGE` 的小阶段没有信号时不机械换班。

## 交接 artifact

交接文档保存在受控 artifact 路径，优先位于仓库外；除非项目流程或用户明确要求，不纳入产品代码 checkpoint、不 stage、不 push。主会话只保留 artifact 路径、hash 和以下摘要，不复制全文。

交接内容必须紧凑且可核验：

- `task_id`、阶段目标、验收标准、明确不做内容；
- `LONG_STAGE`、`planned_rotation_checkpoint`、`context_epoch`、阶段累计 `compression_count|UNKNOWN`、`rotation_count`、换班原因、原 timer key 与绝对 `deadline_at` 或 `PENDING_FIRST_IMPLEMENTATION`；
- `START_SHA`、当前 checkpoint/snapshot、workspace/worktree 绝对路径和当前 Git 状态；
- 已完成、未完成和下一步 1-3 个动作；
- 实际修改文件、文件/模块所有权和仍存在的 dirty/untracked 状态；
- 已确认决策及原因、不能改变的 contract/invariant；
- 已运行测试/HTTP 的准确命令、环境、结果、证据路径及仍未运行项；
- 当前失败、复现方式、阻塞、风险、fixture/服务/端口等环境状态；
- Skill 版本/来源及新 Dev 必须读取的项目规则、spec 和参考文件。

交接不得声称未执行的测试通过，不复制 secret、敏感响应、完整聊天历史或无关日志。

## 换班顺序

1. 旧 Dev 停止新增代码，收敛到可恢复边界；安全且仍有价值时运行 focused tests。
2. 按 commit 模式创建并核对 checkpoint commit 或完整 snapshot，生成交接 artifact，返回 `HANDOFF_READY`。
3. 主任务核对旧 writer 已停止新增写入，记录 artifact 路径/hash、固定 SHA 和 workspace 状态；在旧 Dev 退休前不把 writer lease 授予任何新会话。
4. 使用可用的停止或中断机制退休旧 Dev，确认其退出活跃调度后释放 writer lease，不再向它派发消息或后续任务。任务历史可能仍由运行时保留；这里只保证它退出活跃调度和上下文链，不承诺物理删除历史或进程内存。
5. 以 `fork_turns: "none"` 创建只读 fresh Dev，只提供派发简报、交接 artifact、固定 checkpoint、必要 spec/规则，以及剩余 deadline 或尚未触发的 timer 状态；创建会话不等于授予 writer lease。
6. 新 Dev 在取得 writer lease 前验证目标 SHA、workspace diff/snapshot、修改归属、测试证据与剩余任务，并用紧凑摘要复述理解；不一致时保持只读并返回 `BLOCKED` 或要求主任务修正交接。
7. 核对通过后记录新的 lease owner、授予写入权并递增 `context_epoch`/`rotation_count`，继续同一阶段。退休 Dev 永不重新激活；阶段仍须满足原验证、checkpoint、Review 和完成门禁。

完成标准：新 Dev 无需继承旧聊天即可从固定状态继续，旧 Dev 已退出活跃调度，且没有 writer 重叠、时限重置或证据丢失。
