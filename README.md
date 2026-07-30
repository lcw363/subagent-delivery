# subagent-delivery

[中文](./README.md) | [English](./README_EN.md)

用于**需求确认后的开发交付**的 Codex Skill：以 fresh 子会话隔离实现细节，自动创建不推送的阶段 checkpoint，保守调度串并行任务，并完成测试、真实 HTTP 验证、独立 Review 与修复闭环。

## 适用范围

- 输入应是已确认的 OpenSpec change、tickets、PRD tasks 或明确开发需求。
- 本 Skill 不负责需求发现、方案讨论或范围澄清。建议先用当前最强推理档配合 **OpenSpec** 完善需求；大型多阶段任务先用 **Wayfinder** 明确阶段和依赖，再进入本 Skill。
- 仍有影响业务、数据、权限、安全、API 契约或范围的关键歧义时，会停下等待确认。

## 核心保障

- 功能闭环优先，必要安全内建，额外加固按风险决定：先完成最小生产可用主流程及必要失败路径，不为追求绝对安全或理论上的零 Bug 扩大范围、堆叠抽象或无限 Review。
- Dev、Integrator、Reviewer 使用不继承完整历史的 fresh 子会话；主会话只保留关键决策、状态、SHA 和证据摘要。
- 已确认需求是写入范围上限；只做验收所需行为和不可分离的最小支撑，不顺手新增相邻功能、contract、迁移或无关重构，额外必需改动先停下确认。
- 同一需求、调用链和所有权范围内的相邻串行阶段默认复用当前 Dev；只有领域切换、高风险边界或上下文健康失败时才换 fresh Dev。
- 主任务始终沿用当前任务模型；子任务按复杂度在当前模型与可用的低一级模型之间选择。推理强度默认 `medium`，Integrator、独立 Reviewer 和复杂/高风险任务固定为 `high`；`high` 不可用时暂停确认，不静默降级或升级模型，只有用户明确点名的推理档位才覆盖默认值。
- 自动选择一个或多个开发子任务，并在节点完成后自动启动下一 ready 节点。
- 只有主任务创建和调度直接子会话；子会话不再创建孙子会话，需要额外角色时返回主任务统一派发。满足并行白名单的直接子会话仍可并行。
- 写入默认串行。只有依赖、文件、契约、数据库/fixture/端口、集成和回滚均可证明独立时才并行；共享文件、未冻结契约或共享数据资源一律串行。
- 串行任务任一时刻由唯一活跃 Dev 直接写目标工作区；只有并行批次才使用隔离 worktree 和唯一 Integrator。
- 每个阶段默认自动创建本地 checkpoint commit，但不 push、不建 PR、不部署、不归档；用户或项目禁止 commit 时改用串行 snapshot 模式。
- 条件合适时使用 TDD；实现后简化代码并自检，串行链固定后只创建一个最终独立 Review 节点，在同一会话内依次检查 Standards 与 Spec。当前改动引入、阻断验收、在当前范围存在真实失败路径或影响当前调用链及必要安全的 P0-P3 会修复并复查；只有证明与当前交付无关的历史问题和理论建议才记录为残余风险。
- 新增或修改 HTTP API 时，用脱敏后的实际代表参数调用真实路由；Mock 和 schema 检查不能替代接口验收。
- 节点只清理能证明由自己创建、非共享且无后续消费者的服务、端口、fixture、临时目录和 worktree；不终止共享 Codex、MCP、CodeGraph、Docker 或其他任务资源。
- 用户可指定时限或不限时。未指定时，每个实现节点收口、每个并行批次闭环和最终整体收口各默认 60 分钟；阶段内 Dev 换班不重置时钟，超时标记 `STOPPED_INCOMPLETE`，不会伪装成完成。

## 安装与调用

```bash
mkdir -p ~/.agents/skills
git clone https://github.com/lcw363/subagent-delivery.git \
  ~/.agents/skills/subagent-delivery
```

显式调用最可靠：

```text
请使用 $subagent-delivery 完成 openspec/changes/add-order-refund，主会话只保留关键结论。
```

安装后，Codex 在任务匹配 Skill 描述时也可能自动选择它。

## 执行流程

1. 锁定需求、验收标准、测试和发布边界。
2. 按独立验收阶段生成依赖 DAG；相邻串行阶段在同一需求、调用链和所有权范围内复用 Dev，只有领域切换、高风险边界或上下文健康失败时才换班。
3. Dev 完成最小实现、条件性 TDD、focused tests、代码简化、自检和阶段 checkpoint。
4. 长阶段到达计划评估点时检查上下文健康；健康则继续复用当前 Dev，第一次可观察压缩时预备交接，第二次可观察压缩、健康检查失败、领域切换或独立高风险边界才换班。新 Dev 只读核验后才取得写入权，原节点和时限不变。
5. 串行阶段或并行集成批次运行适用回归/compile/build；API 改动补真实 HTTP 参数测试。
6. 串行链固定后由主任务创建一个最终独立 Reviewer；问题交回唯一 writer 修复、重新 checkpoint 并复查，直到通过或时限收口。节点结束时按资源所有权记录清理或保留结果。

例如阶段 C/D 即使已有 checkpoint，只要共享文件、契约未冻结、数据库或 fixture 不能隔离，就必须串行。worktree 创建或集成条件失败时也自动回退串行并保留恢复证据。

## 示例

```text
请使用 $subagent-delivery 交付 Wayfinder 中已确认的 10 个开发阶段：
- 自动判断阶段依赖并持续启动下一任务；除非模块完全独立，否则串行；
- 每阶段自动创建本地 checkpoint commit，但不 push、不部署；
- 新增接口使用测试账号和实际订单参数调用本地 HTTP 路由；
- 最终独立 Review 并闭环当前交付中可执行的 P0-P3；范围外问题列为残余风险，无法在时限内完成时列出未完成清单。
```

完整规则见 [SKILL.md](./SKILL.md)；上下文换班见 [context-rotation.md](./references/context-rotation.md)，恢复协议见 [checkpoint-and-recovery.md](./references/checkpoint-and-recovery.md)，模型策略见 [model-routing.md](./references/model-routing.md)，简报与测试证据见 [evidence-and-briefs.md](./references/evidence-and-briefs.md)。
