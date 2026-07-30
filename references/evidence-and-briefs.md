# 精简简报与证据协议

在首次派发 Dev/Integrator/Reviewer、执行真实 HTTP 验证或生成最终报告前读取本文件。详细日志留在执行子会话或 artifact 中，主会话只保留以下结构化摘要。

## 派发简报

每个子会话只接收：

- `task_id`、角色、目标和输入来源；
- `scope_allowlist`（逐类记录允许改变的行为、API/字段 contract、权限、数据及副作用、schema/迁移、依赖和获准的局部重构；未获准类别为 `none`）、明确不做内容、验收标准和风险；
- 文件/模块所有权、依赖、固定基线 SHA 或 snapshot；
- 测试层级、真实 HTTP 要求、环境与副作用边界；
- 模型角色层级、实际选用模型（可见时）、推理强度、选择/升级原因和 fallback；
- `COMMIT_MODE`、checkpoint 规则、timer key、绝对 `deadline_at`（默认节点时钟尚未触发时为 `PENDING_FIRST_IMPLEMENTATION`）和发布权限；
- `spawn_authority=main_only`，子会话不得创建孙子会话；需要额外角色时返回给主任务统一派发；
- `RESOURCE_LEDGER`：当前节点拥有、共享、任务前已存在或需要保留到恢复/后继节点的服务、进程、端口、fixture、临时目录和 worktree；
- 首次派发时的 `LONG_STAGE`、`planned_rotation_checkpoint`、`context_epoch=0`、`compression_count=0|UNKNOWN`、`rotation_count=0`；接力派发时继承这些字段、换班原因、交接 artifact 路径/hash 和剩余 deadline；
- 必要项目规则与已确认决策，不附整段聊天或其他节点完整日志。

Dev 返回：节点状态、改动摘要、关键文件、每项改动对应的需求或最小支撑理由、简化/注释、自检、测试与 HTTP 摘要、checkpoint 状态/SHA/artifact 路径、`RESOURCE_LEDGER` 清理结果、未完成项、阻塞和风险；没有未授权范围扩张时明确记录 `SCOPE_OK`。

上下文换班时，旧 Dev 额外返回 `HANDOFF_READY`、换班原因、`context_epoch`/`compression_count`/`rotation_count`、固定 checkpoint、交接 artifact 路径/hash 和 workspace 状态；主任务记录旧 Dev 退休、writer lease 释放与新 owner，新 Dev 返回对交接字段的核对结果。详细格式见 [context-rotation.md](context-rotation.md)。

Integrator 额外返回：消费的 `TASK_SHA` 顺序、冲突或范围核对、批次测试、`INTEGRATION_SHA`、未集成节点与目标 workspace 状态。

Reviewer 输入必须固定：`scope_allowlist`、明确不做内容、验收标准、审查范围、base/head SHA 或不可变 snapshot、项目规则、必要测试证据，以及 Standards/Spec 两轴检查要求。单个 Reviewer 在同一会话内依次完成两轴，不创建子会话。输出记录 `code_checkpoint` 和审查范围，把当前改动引入、阻断验收、在当前范围存在真实失败路径，或影响当前路由、调用链、副作用及必要安全的问题列为可执行 P0-P3 finding（等级、紧凑行号、失败路径、影响、最小修复建议）；只有能以 diff 和调用链证明与当前交付无关的范围外历史问题，以及无现实失败路径的理论建议，才在必要时列为残余风险。严重度按实际影响判断，权限、安全、隐私、敏感信息、资金和数据完整性问题不得因类型未被枚举而漏报。另行记录范围扩张、测试缺口和 workspace 前后无变化证据；存在未授权范围扩张时不得输出 `SCOPE_OK` 或 `REVIEW_PASS`。没有只读 custom agent 时，前后证据覆盖 HEAD、staged/unstaged patch、范围内 untracked 路径/模式/内容 hash 和 submodule 状态。没有问题时明确写明审查范围、`SCOPE_OK` 与“未发现已知且可执行的 P0-P3 问题”。

任何原因产生后续代码 checkpoint 时，只要改动落入原审查范围或相关调用链，旧 `REVIEW_PASS` 立即变为 `REVIEW_STALE` 并必须复查；只有能以 diff 和调用链证明后续改动无关时才可保留。最终 `REVIEW_PASS` 必须对最终 fixed point 有效。

## 测试证据

按风险选取最小充分层级，并记录准确命令、环境、结果、失败数与未覆盖项：

1. Dev 开发中的 focused/regression tests；
2. 简化或修复后的受影响测试；
3. 串行阶段或并行集成批次的一次适用回归、类型检查、compile/build；
4. 适用的 OpenSpec strict validate 与 diff-check；
5. 只有项目规则、用户或验收标准明确要求时，才补尚未覆盖的全量验证。

已确认会被范围外历史错误阻塞的全量 `testCompile`、全量测试或构建只记录一次阻塞证据，后续 fixed point 未触及该阻塞调用链时不重复运行。历史无关失败不能伪装成通过，也不能未经授权扩大修复范围。未运行的检查写明 `NOT_RUN` 和原因。

## 真实 HTTP 证据

新增或修改 HTTP API 时，使用本地服务或明确授权的非生产环境与可回滚 fixture 调用真实路由。证据至少包括：

- method、route、环境、服务版本或 SHA、时间；
- 脱敏后的实际代表参数与认证方式；
- 预期业务结果、实际状态码和关键响应字段；
- 数据库、消息、账务或第三方 Test Mode 等副作用核对；
- fixture 创建方式、清理/回滚结果；
- `PASS`、`FAIL` 或 `BLOCKED`，以及原始日志/artifact 路径。

请求和响应 wire type 必须遵循已确认 API contract。对于超出客户端安全整数范围的 64 位 ID，可在证据、日志或客户端解析层保留原始字符串以防精度丢失；若 contract 定义请求为 number，不能为了记录方便擅自改成 string。

Token 从环境或授权的 secret 工具读取，不写入命令、补丁、日志或主会话；请求/响应中的凭据、个人信息和支付数据要脱敏。未经明确授权不访问生产环境，不执行不可回滚的真实交易或数据修改。

把真实 HTTP 建模为依赖固定 code checkpoint 和环境准备的独立 acceptance 节点；它可与该 fixed point 的只读 Review 并行。每条结果记录 `code_checkpoint`。后续 checkpoint 若改变 endpoint contract、认证授权、相关调用链、持久化/消息/第三方副作用或 fixture，旧 `HTTP_PASS` 立即变为 `HTTP_STALE` 并必须重跑；只有能以 diff 和调用链证明后续改动无关时，才可保留原结果。最终 `HTTP_PASS` 必须对最终 fixed point 有效。

服务、凭据或有效 fixture 缺失时 HTTP 节点为 `BLOCKED`，只阻塞显式依赖它的节点与整体完成，不回退已取得的代码或 Review 状态。Mock、MockMvc、单测、schema、静态 curl 示例或“服务应当能启动”都不能替代实际接口调用。

## 最终汇报

- 最终报告仅列：范围/节点状态、关键文件、自动化与 HTTP 证据、checkpoint、Review/fix loop、当前 P0-P3、资源清理/保留状态、未完成/阻塞/风险、commit/push/deploy/archive 状态。
- 分开声明 `CODE_COMPLETE`、`TEST_PASS`、`HTTP_PASS|HTTP_FAIL|HTTP_BLOCKED|HTTP_STALE|HTTP_NOT_APPLICABLE`、`REVIEW_PASS|REVIEW_STALE`、`RELEASED|RELEASE_NOT_REQUESTED`；一个状态不能代替另一个。`STOPPED_INCOMPLETE` 必须列出恢复所需的下一步和固定 checkpoint。

完成标准：主会话能用精简摘要审计每个结论，同时完整过程与敏感数据不会污染主上下文。
