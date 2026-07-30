---
name: subagent-delivery
description: 用于已确认需求、OpenSpec change 或 tickets 的隔离子会话开发交付；不用于需求发现、方案讨论或范围澄清。包含测试、独立 Review 与修复闭环。
---

# Subagent Delivery

只在需求与验收边界已经确认后使用。主会话负责锁定范围、调度、状态与最终结论；Dev、Integrator、Reviewer 使用不继承完整历史的全新子会话，只接收完成当前职责所需的精简简报。

## 核心原则

功能闭环优先，必要安全内建，额外加固按风险决定。先完成已确认范围内的最小生产可用实现，确保主流程、必要失败路径、状态变化和外部副作用可验收且逻辑闭环；与功能不可分离的权限、数据一致性、事务、信任边界输入校验、敏感信息保护和失败恢复必须同步实现。此后只补与实际风险相称的常规安全、边界和回归处理，不为追求绝对安全、穷举所有理论边界或证明完全无 Bug 而扩大范围、增加无明确收益的抽象或无限延长 Review/fix loop。

默认采用快速交付基线：

- 一个边界清晰的任务只派发一个串行 Dev，完成后只建立一个最终独立 Review 节点；不为机械步骤拆分子任务，也不增加重复的预审或“终审”。
- 验证以 focused/regression tests、生产 compile/build、适用的 OpenSpec strict validate 和 diff-check 为最小充分集合。已确认会被范围外历史错误阻塞的全量 `testCompile` 或全量测试不重复运行，只记录阻塞证据。
- Review 以业务可用、流程闭环、外部副作用不重复、当前改动不破坏真实路径为准。理论风险、极端全量扫描、证明绝对唯一或额外安全门禁，没有现实失败路径时只记录残余风险，不推动扩 scope。
- 修复当前改动引入或暴露的 P0-P2，以及明确可复现、落在当前范围且有实际影响的 P3；不修复范围外历史问题或纯理论建议。修复后只重跑受影响验证，并对新 fixed point 复查。
- 共享目标分支持续被其他 writer 推进时，优先等待其结束或切换到隔离 worktree；不要在移动中的共享分支上反复重算基线、重做测试和重写 checkpoint。
- 只有主任务可创建和调度子会话。Dev、Integrator、Reviewer、Fixer、测试或环境子会话不得继续创建孙子会话；需要额外角色时返回精简请求，由主任务按现有并行白名单创建直接子会话。

## 不可变门禁

- 若业务规则、数据、权限、安全、API 契约或范围仍有关键歧义，停止受影响节点，返回 OpenSpec、Wayfinder 或用户确认，不把猜测写进生产代码。
- 把已确认需求视为写入范围上限：只实现验收所需行为及不可分离的最小支撑改动。最小支撑必须同时满足“不产生独立可见的产品行为/contract/权限/数据副作用/依赖变化”和“不做就无法满足当前验收”；不满足任一条件即属额外范围。未经确认不新增相邻功能、接口/字段/状态、权限、schema/迁移、依赖、兼容分支或重构清理；不影响当前交付的列入不做清单，确为验收必需的只暂停受影响节点并请求确认。
- 支持时以 `fork_turns: "none"` 启动 fresh Dev、Integrator、Reviewer；主会话只保留决策、状态、固定 SHA、证据摘要和未决项。
- 同一工作区同一时刻只有一个 writer。纯串行任务任一时刻由唯一活跃 Dev 直接写目标工作区，阶段内允许按协议换班但不创建 Integrator；只有并行批次才由唯一 Integrator 写目标工作区。Reviewer 始终只读。
- 写入任务默认串行。只有依赖、写入文件、共享契约、数据库/fixture/端口等资源、集成与回滚都能证明独立时才并行；“不同模块”或“已有 checkpoint”本身不构成证明。
- 默认每阶段创建本地 checkpoint commit，但不等于发布授权。除非用户明确要求，不 push、不建 PR、不部署、不归档，也不 rebase、squash、reset 或清理 checkpoint。
- Dev 自检不能替代最终独立 Review。当前改动引入、阻断验收、在当前范围存在真实失败路径，或影响当前路由、调用链、副作用及必要安全的 P0-P2，以及明确可复现且有实际影响的 P3，属于可执行问题，必须修复并对新 fixed point 复查；范围外历史问题和无现实失败路径的理论建议只记录残余风险。可执行问题超出写入授权时暂停受影响节点并请求确认，不擅自扩大范围。

## 1. 锁定输入

1. 按“用户当前指令 > 已确认 spec/tickets > 项目规则 > 现有行为”确定范围；形成 `scope_allowlist`，逐类记录允许改变的行为、API/字段 contract、权限、数据及副作用、schema/迁移、依赖和获准的局部重构，未获准类别明确为 `none`；同时记录验收标准、明确不做内容、风险、测试与发布边界。
2. 有 OpenSpec 时先验证 change。只有当前任务授权维护 artifact 或项目流程明确要求时，才修复机械性的结构错误；任何语义、范围、验收标准变化都必须先确认。仅在被授权时更新 task 状态。
3. 在生成 DAG 或派发任何 writer 前，完整读取 [checkpoint-and-recovery.md](references/checkpoint-and-recovery.md)，检查 dirty 状态并固定 `COMMIT_MODE`、任务前基线和恢复方式；再记录用户指定的单任务、串行/并行、环境、真实接口与时限要求。
4. 派发任何子会话前，完整读取 [model-routing.md](references/model-routing.md)：主任务沿用当前模型，子任务按复杂度在当前模型与低一级模型之间选择；推理强度默认 `medium`，Integrator、Reviewer 和复杂/高风险任务使用 `high`，用户明确点名的档位优先，不硬编码具体模型版本。
5. 形成精简交付简报，禁止把整段主会话历史复制给子会话。未指定的行为使用本 Skill 默认值。

完成标准：需求输入、验收边界、发布权限和待确认项足以让 Dev 独立执行。

## 2. 拆分与调度

1. 用户指定只用一个开发节点/任务、不要拆分或不要并行时直接遵循；同一节点仍可按协议更换 fresh Dev。用户指定只用一个 Dev/子会话时禁止换班；只说“一个”而未明确单位时按单一子会话处理，命中强制换班条件则 `BLOCKED` 等待确认。
2. 只有存在可独立验收的阶段或单个上下文难以稳定容纳时才拆分；优先复用已确认的 OpenSpec/Wayfinder 阶段，不拆无独立价值的机械步骤。同一需求、调用链和所有权范围内的相邻串行阶段，只要上下文健康且能稳定容纳，就复用当前 Dev；出现明显领域切换、独立高风险边界或上下文健康失败时才换 fresh Dev。阶段若在完成前包含至少两个顺序、可验证的安全 checkpoint，或已被 spec/Wayfinder 标为大型阶段，则标记 `LONG_STAGE` 并预设阶段内上下文健康评估点；评估点本身不强制换班，暂时找不到安全边界时记为 `PENDING_SAFE_BOUNDARY`，不新增开发节点。
3. 共享文件、未冻结 API/DTO/schema/核心抽象、共享数据库或 fixture、迁移顺序、不可隔离环境任一存在时，建立顺序边。先串行冻结共享契约，再重新评估后续任务。
4. 对满足全部并行白名单的 ready 节点，可用隔离 worktree 并行；只读分析、测试设计和只读审计在不争用环境时可更积极并行。worktree 能力失败则保留证据并自动回退串行。
5. 主任务维护依赖 DAG 和 ready queue：一个节点结束后自动启动下一 ready 节点；多个 ready 节点仅在白名单成立时并行。局部 `FAIL`/`BLOCKED` 只阻塞该节点及其依赖。
6. 并行时发现文件或语义重叠，停止后启动的冲突 writer，保存 checkpoint；先集成前一个结果，再从最新目标 checkpoint 串行重启受影响任务，不让主会话硬合并语义冲突。
7. MySQL、Docker、第三方 Test Mode、测试账号和 fixture 仅在已授权、非生产、资源隔离且副作用可回滚时作为环境节点提前并行；环境就绪不等于接口验收通过。
8. 一个阶段同一时刻仍只有一个 writer，必要时可由多个 fresh Dev 串行接力。派发 `LONG_STAGE` 前或运行中出现上下文压缩、重复取证、范围增长等信号时，完整读取 [context-rotation.md](references/context-rotation.md)；到达预设评估点时先检查上下文健康，健康则继续复用当前 Dev，只有出现领域切换、独立高风险边界或命中强制健康条件时才执行 checkpoint、交接和退休。换班不解锁依赖，也不重置任务 ID、验收标准或 deadline。
9. 所有子会话派发都由主任务统一完成。子会话需要额外分析、测试或 Review 角色时，只返回目标、输入、依赖和建议并行性，不自行 spawn；主任务仍可把多个满足白名单的直接子会话并行派发。

状态含义：

- `DEV_READY`：并行 Dev 已完成实现、自检、focused tests 和 task checkpoint，可交给 Integrator；不能解锁依赖节点。
- `DEV_PASS`：串行阶段完成适用验证和 stage checkpoint；并行批次还必须完成集成验证、batch checkpoint 及强制 batch `REVIEW_PASS`。只有达到该状态才解锁依赖节点。
- `REVIEW_PASS`：独立 Reviewer 对固定 `code_checkpoint` 与范围复查后，没有已知且可执行的 P0-P3 问题。
- HTTP acceptance 状态：`HTTP_PASS`、`HTTP_FAIL`、`HTTP_BLOCKED`、`HTTP_STALE` 或 `HTTP_NOT_APPLICABLE`；它与代码、Review 状态分别记录。
- `FAIL` / `BLOCKED`：实现或验证失败，或缺少继续所需的权限、环境、凭据、fixture、用户决定。
- `STOPPED_INCOMPLETE`：达到时限后收口，仍有未完成项；它不是通过或完成状态。
- `HANDOFF_READY`：当前 writer 已停止新增写入，固定 checkpoint/snapshot 和交接 artifact 已核验，可安全切换 fresh writer；它不是阶段通过状态。

## 3. 开发、测试与 checkpoint

在首次派发子会话、独立 Review、真实 HTTP 验证或最终汇报前，完整读取 [evidence-and-briefs.md](references/evidence-and-briefs.md)。

Dev 必须：

- 先完成最小生产可用功能闭环，不扩大范围，不修改无关 contract；主流程、必要失败路径、状态变化和外部副作用必须可验收。每项代码、配置、依赖和数据变更都必须能直接追溯到 `scope_allowlist` 或不可分离的最小支撑，无法说明必要性就不修改。只在兼容且可用时调用其他 Skill，不因缺失而降低交付标准。
- 测试缝和预期行为清晰时使用 `$tdd`；Bug 修复优先先写能复现问题的回归测试。测试缝会改变设计时先确认，否则按最小实现加事后测试并说明原因。
- 开发中跑 focused tests。实现后用 `$code-simplifier` 或等价规则简化本次改动，补充只解释关键“为什么”的注释，重跑受影响测试，再做需求遗漏、异常路径、测试缺口与复杂度自检。
- 串行阶段收口或并行批次集成后只运行一次适用的受影响回归、类型检查、编译或构建；最终验证只补尚未覆盖的整体验证，避免机械重复重测试。
- 默认验证集合为目标测试、生产 compile/build、适用的 OpenSpec strict validate 和 diff-check。全量测试只在项目规则、用户或验收标准明确要求，且不存在已确认的范围外基线阻塞时运行；已知无关 `testCompile`/全量失败不反复重试、不纳入当前修复范围。
- 创建本地服务、端口、fixture、临时目录或 worktree 前登记资源所有权；节点收口时只清理能证明由当前节点创建、非共享且无后续消费者的资源，遵循 [checkpoint-and-recovery.md](references/checkpoint-and-recovery.md)。

新增或修改 HTTP API 时，建立以固定 code checkpoint 和可用环境为输入的独立 acceptance 节点；交付完成前必须取得对最终 fixed point 仍有效的 `HTTP_PASS`。真实调用、证据、阻塞、wire type、凭据和过期处理遵循 [evidence-and-briefs.md](references/evidence-and-briefs.md)；HTTP 节点状态不抹除已取得的代码或 Review 状态，但会约束显式依赖节点与整体完成。

## 4. 独立 Review 与修复

1. 单任务或纯串行链只在全部实现与可执行自动化测试完成后进入一个最终独立 Review/fix loop；禁止在此之前增加重复预审。每次范围内修复形成新 fixed point 后，由该最终 Review 节点复查受影响调用链和累计 diff。HTTP acceptance 状态单独记录，不阻止代码 Review 形成独立结论。
2. 并行任务必须在 Integrator 合并、批次验证和 batch checkpoint 后做独立 Review；该 Review/fix loop 通过后批次才达到 `DEV_PASS` 并解锁后继。只有资金、权限、安全、迁移、并发、共享核心契约、难回滚副作用或大型子任务才增加合并前 Review。
3. 最后一次批次 Review 明确覆盖全部累计 diff 与整体调用链时，可同时作为最终 Review。
4. 优先使用一个只读 custom Reviewer。该 Reviewer 在同一会话内依次完成 Standards 与 Spec 两轴并统一报告；只有 `$code-review` 或其他 Review Skill 支持单会话、无嵌套子会话模式时才调用，否则把等价审查规则直接写入 Reviewer 简报。Reviewer 必须把全部 diff 与 `scope_allowlist`、明确不做内容逐项核对；任何已经写入的未确认行为、contract、权限、schema/迁移、依赖或无关重构本身就是可复现的范围缺陷，至少列为 P3，并按实际影响提高等级；在移除或获得确认前不得给出 `SCOPE_OK` 或 `REVIEW_PASS`。不可用时，Review 前后核对规范化 workspace diff/内容指纹；发生非预期变化则该 Review 无效。
5. 严重度按实际影响判断，以下类型不是穷举：P0 为灾难性数据、安全、资金或全局可用性事故；P1 为阻断核心需求、主流程或发布，或造成重大权限、安全、隐私、敏感信息、资金、数据完整性风险；P2 为重要场景的真实正确性、兼容性、可靠性或非阻断安全问题；P3 为低影响但可复现、应在当前范围修复的局部缺陷。当前改动引入、阻断验收、在当前范围存在现实失败路径，或影响当前路由、调用链、副作用及必要安全的 P0-P2，以及明确可复现且有实际影响的 P3，属于可执行 finding；范围外历史问题、无现实失败路径的理论建议、为证明绝对唯一而提出的全量扫描或额外门禁，只列为残余风险。已经写入的未授权范围扩张按上一条处理。
6. 可执行问题交给当前 writer lease owner；若修复超出写入授权，暂停受影响节点并请求确认，不得因其表面上属于历史代码或范围外文件而降为残余风险。已证明与当前路由、调用链、副作用及必要安全无关的历史 P2/P3 不自动修复。当前 writer 已退休或不存在时，从最新 fixed point 按交接协议创建 fresh Fixer，禁止重新激活退休会话。合并前问题仍归对应任务，批次/跨任务/整体问题仍归 Integrator 职责。替换 writer 时依次停止旧 writer、固定 checkpoint/workspace、完成交接、退休并释放 lease，再启动只读 fresh replacement；其核验通过后才记录新 lease owner 并授权写入。修复后重跑相关测试、创建新 checkpoint，并让独立 Reviewer 审查新的 fixed point，禁止在旧结论上口头关闭。

## 5. 时限与完成

- 用户可指定总时限、阶段时限或不限时。指定时限是硬上限；调度时固定绝对 `deadline_at`，每个子会话使用“用户总时限剩余值”和当前节点时限中的较早者，禁止自行重置。
- 未指定时，每个实现节点的收口时钟从第一版功能实现完成起算；此前记录 `deadline_at=PENDING_FIRST_IMPLEMENTATION`，触发时固定一次绝对 deadline，阶段内 Dev 换班继承该时钟。每个并行批次时钟从 Integrator 收到首个固定 `TASK_SHA` 起算，最终时钟从全部实现节点已通过或阻塞并进入整体收口起算，各默认 60 分钟墙钟时间。等待、重试和换班不暂停、不重置；长 DAG 没有额外的默认总时限；最后批次 Review 兼作最终 Review 时只使用一个时钟。
- 不启动明显无法在剩余时间内安全结束的操作。到达 deadline 后停止所有可安全中断的工作，只允许完成保护数据/安全或保存可恢复 checkpoint 所必需的清理，并标记 `STOPPED_INCOMPLETE`。超时后不能新授予 pass 或“已完成”；超时前已取得的状态保留为历史证据，但不改变整体未完成结论。
- 只有范围内功能与逻辑闭环完成、必需自动化通过、HTTP 不适用或 `HTTP_PASS` 对最终 fixed point 仍有效，且 `REVIEW_PASS` 对最终 fixed point 仍有效、没有已知可执行 P0-P3 时，才能声明交付完成。完成条件不是证明系统绝对安全或完全无 Bug；范围外历史问题和理论风险按残余风险如实列出。
- 最终只汇总：范围与节点状态、关键文件、自动化及 HTTP 证据、checkpoint、Review 与修复、P0-P3、未完成/风险、commit/push/deploy/archive 状态。明确区分代码完成、测试通过、接口验收和上线完成。
