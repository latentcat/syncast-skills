---
name: syncast-agent-actions
description: 通过浏览器内 Agent Action Layer 操作已打开的 Syncast 项目。当外部 Agent 需要检查项目、委托 Syncast 内部 Agent、为直接 CLI 生成读取项目上下文、排布时间轴 AI 占位 Slot、等待任务完成，或读取项目文档/资源/频道、获取资产下载链接时使用；禁止绕过前端或直接点击 UI。
---

# Syncast Agent Actions

当你需要控制一个已经打开的 Syncast 网页项目时，使用这个 skill。

外部 Agent 的角色是“像人类一样操作项目的使用者”。它应该先检查项目现场，再把业务工作委托给 Syncast 内部 Agent，等待任务完成，然后读取项目数据和产物。不要直接写 Loro，不要直接调用后端 Response API，也不要在 action bridge 可用时通过点击 UI 来完成自动化。

## Skill 更新

本 skill 的发布源是 `latentcat/syncast-skills`。`syncast-cli` npm 包更新不会自动更新本地已安装的 skill 文档。

在使用 `syncast project-agent` 操作项目前，如果本地 skill 缺失、action/schema/example 和 CLI 能力不匹配，或用户要求更新 Syncast Agent 能力，先从发布仓库刷新本 skill：

```bash
npx skills add latentcat/syncast-skills --skill syncast-agent-actions -y
```

如果同时使用 `syncast-cli` skill，也刷新它：

```bash
npx skills add latentcat/syncast-skills --skill syncast-cli -y
```

## 平台理解

Syncast 是一个本地优先、多人协作的 AI 创作工作台。一个项目通常包含文档、资源、画布、时间轴、AI / Agent 频道、工作流、录制和绘画等模块。

外部 Agent 不应把 Syncast 当成单个聊天窗口，而应把它当成“项目创作操作系统”：

1. 先看项目状态、同步状态、任务和积分。
2. 判断当前项目是空项目，还是已有文档、资源、画布、时间轴、频道和任务历史的进行中项目。
3. 判断用户希望“协作创作”还是“完全托管”；不明确时按协作创作处理。
4. 再读文档和资源，理解项目规范、素材和已有产物。
5. 然后判断是否需要画布组织、时间轴剪辑、Imagine 生成、工作流自动化或内部 Agent 协作。
6. 对复杂业务任务，高频委托或询问内部 Agent；外部 Agent 负责理解人类意图、转交任务、等待、读取和验收。

更完整的平台模块和创作路线见 [platform-guide.md](platform-guide.md)。执行实际任务前，先用这份指南判断当前项目处于哪个创作阶段。

## 入口

在浏览器项目页中调用：

```ts
await window.__syncastAgent.initialize({
  name: "外部 Agent 名称",
  description: "说明这个外部 Agent 负责的自动化范围。",
  emojiAvatar: "🤖",
});

await window.__syncastAgent.run(actionName, input, options);
```

返回值统一使用：

```ts
{ ok: true, data: unknown } | { ok: false, error: { code, message, details? } }
```

## CLI Bridge 入口

当外部 Agent 的浏览器工具不允许在页面上下文执行会产生副作用的 `evaluate` 时，不要绕过限制去注入任意 JS。改用 `syncast project-agent` 这条窄口 bridge。它只把结构化 action 请求转发给已打开项目页里的 `window.__syncastAgent`，不暴露通用浏览器控制台，也不替用户点击 UI。

启动本地 bridge：

```bash
syncast project-agent serve
```

用 `serve` 输出的 `syncastAgentBridgeToken` 和 `syncastAgentBridgePort` 打开或刷新 Syncast 项目 URL。页面连接后确认：

```bash
syncast project-agent pages
syncast project-agent capabilities
syncast project-agent capabilities --action syncast.agent.delegate --disclosure full
syncast project-agent capabilities --action syncast.imagine.submitToChannel --disclosure full
syncast project-agent capabilities --action syncast.docs.imagineBlocks.submitBatch --disclosure full
```

随后仍然按本 skill 选择 actionName / input / options，再通过 CLI 发送：

```bash
syncast project-agent run syncast.project.inspect --input '{"limit":20}'
syncast project-agent run syncast.agent.delegate --input '{"goal":"请整理项目方案并写入项目文档。","executor":{"kind":"model","model":"gemini-3.5-flash"},"wait":false}'
syncast project-agent run syncast.imagine.submitToChannel --input '{"channelId":"<imagine-channel-id>","modelType":"nano-banana-2","prompt":"生成角色设定图","targetAssetName":"角色A"}'
syncast project-agent run syncast.docs.imagineBlocks.submitBatch --input '{"docId":"<doc-id>"}'
syncast imagine --project <project-id> --folder "/角色/设定" --name "角色A" --prompt "生成角色设定图"
syncast project-agent wait --ref '{"kind":"agent_chat","projectId":"..."}' --return-result
syncast project-agent asset-download-urls --asset-id "asset-id"
syncast project-agent materialize-media-segments --asset-id "audio-asset-id" --segments '[{"segmentId":"part-1","startTimeSeconds":0,"endTimeSeconds":15},{"segmentId":"part-2","startTimeSeconds":15,"endTimeSeconds":30}]'
```

如果内部 Agent 暂停等待工具审批，外部 Agent 是用户托管主体，可以自己审批，也可以先把问题转交给真实用户后再审批。先列出审批通知：

```bash
syncast project-agent notifications --type agent_action.approval_requested --limit 5
```

审批通知会带 `approvalId`、`respondAction` 和 `nextActions`。优先按通知里的 `respondAction` 操作；CLI 便捷命令如下：

```bash
syncast project-agent approval respond <approvalId> --approve
syncast project-agent approval respond <approvalId> --deny --feedback "用户拒绝本次生成。"
```

这会调用项目页里的 `syncast.agent.approval.respond`。只有发起该内部 Agent 任务的同一个外部 Agent 身份可以响应；非 owner 会得到 `approval_actor_mismatch`。用户 UI 仍可兜底处理，谁先响应谁生效。

这条路径的职责边界：

- 本 skill 负责告诉外部 Agent 怎么理解 Syncast 项目、选择 action、组织 GraphQL / input。
- `syncast project-agent` 只负责把请求送进当前浏览器项目页并返回统一 `{ ok, data/error }` 结果。
- 浏览器 bridge 与 CLI 只发现和执行显式标记的外部高层 action。`syncast.imagine.submit` / `submitToChannel`、`syncast.timeline.generationSlots.submit` 和 `syncast.docs.imagineBlocks.submit/submitBatch` 都是公开的用户等价操作；草稿渲染器、重复 transport helper、兼容别名等内部实现不会出现在 capabilities 中，直接调用返回 `action_not_exposed`。
- 用户继续操作 UI；外部 Agent 不通过 Playwright click/fill 操作业务 UI。
- 如果当前环境允许原生、可写的 Playwright `page.evaluate`，也可以直接调用 `window.__syncastAgent.run`；如果工具声明 `evaluate` 只读，必须使用 CLI Bridge 或其它正式 bridge。

### 公开与内部边界

- 公开 action 的标准是“用户在项目 UI 中可以发起的高层操作，且 Action 能复用同一条校验、持久化和任务链路”，不是“是否会写数据”。频道创建、文档/时间轴 GraphQL mutation、Imagine 提交、时间轴 Slot 提交、Agent 委托和任务取消都可以公开，但仍受当前用户权限与 action approval 约束。
- 隐藏项主要是重复 transport helper（`project.capabilities`、typed wait/result、通知 action）、被规范入口替代的兼容别名、内部 Markdown/草稿渲染器，以及含不稳定 raw input 的底层 action。隐藏它们不能导致用户可做的业务操作消失；必须存在公开等价入口。
- `syncast.doc.graphql` 本身是公开能力，不是整层屏蔽。外部调用只禁止 `createAsset` 以及 filename、remoteFilename、storage/provider 等实现字段；正常的项目 query/mutation 继续通过动态 read/edit 权限执行并保存同步。
- 判断某个名字能否调用时，以 `capabilities` 为准；判断能力是否缺失时，要同时检查是否已有公开规范 action 或 bridge method，不能仅凭内部 action 名称下结论。

## 默认流程

1. 初始化身份并检查当前项目：

```ts
await window.__syncastAgent.initialize({
  name: "Project Operator Agent",
  description: "负责检查项目、委托内部 Agent，并验收生成结果。",
  emojiAvatar: "🤖",
});

await window.__syncastAgent.run("syncast.project.inspect", {
  limit: 20
});
```

2. 结合 [platform-guide.md](platform-guide.md) 判断项目是空项目还是进行中项目，并确认“协作创作 / 完全托管”模式。

3. 规划 Channel：复用少量稳定 Channel，不要每个任务都新建 Channel；按项目规划、角色设定、场景设定、分镜镜头、生成实验、时间轴 Slot 等主题组织。

4. 把复杂业务工作委托给内部 Agent：

```ts
const started = await window.__syncastAgent.run("syncast.agent.delegate", {
  goal: "请基于当前项目资源生成一个视频项目方案，并写入项目文档。",
  executor: { kind: "model", model: "gemini-3.5-flash" },
  wait: false
});
```

5. 等待完成：

```ts
const done = await window.__syncastAgent.wait(started.data.ref, {
  timeoutMs: 30 * 60 * 1000,
  returnResult: true
});
```

如果等待的是内部 Agent chat/delegate 任务，最终可见回复在 `done.data.result.text`，短摘要在 `done.data.result.textPreview`。CLI 的 `syncast project-agent wait --return-result` 也返回同样的 JSON 结构；默认 JSON 面向 Agent，人工查看可加 `--format human`。

6. 读取并验收结果：

```ts
await window.__syncastAgent.run("syncast.docs.readForAgent", {
  changedSince: started.data.ref.startedAt
});
```

## 规则

- 项目方案、视频工作流、文档撰写、剧本设计、提示词生成、模型参数理解、时间轴 Slot 规划等任务，优先使用 `syncast.agent.delegate` 交给内部 Agent。
- 要像用户在已有 AgentChannel 中继续对话，调用 `syncast.agent.delegate` 并传真实 `channelId`；需要读取该频道上下文时同时传 `includeHistory: true`。它与内部 `syncast.agent.chat.submit` 走同一前端消息/任务链路，但只公开稳定、安全的输入面。
- 项目 Agent 是一份独立声明，可直接作为 `executor: { kind: "agent", agentId }` 使用，也可绑定为另一 Agent 的命名选项。直接模型 `executor: { kind: "model", model }` 能看到项目全部 Agent；声明 Agent 只看到自己的绑定项。两者都能从头创建不继承父指令的临时子 Agent，且所有子执行均为叶节点。
- 外部 Agent 在选择声明 Agent 前，先通过 Agents GraphQL 查询真实 `id` 与 `childAgents`。CLI 的 `--agent-id` 是外部操作者身份，只有 action input 中的 `executor.agentId` 才选择项目内 Agent。
- 查询或更新 Agent 时必须请求并保留 `allowLoadSkills` 与 `skills { skillId skillType preload }`。所有内置 Skill 始终可按需加载；custom binding 决定关闭扩展发现时仍可用的项目 Skill。`preload` 只决定是否在启动时注入完整说明。旧 binding 缺少 `preload` 时按 `true`，新 binding 应显式写 `false` 或 `true`。
- `allowLoadSkills` 是历史兼容字段：`false` 只禁止发现未选中的项目 custom Skill，仍保留已选 custom Skill、依赖、`alwaysApply` Skill 和全部内置 Skill；`true` 扩展到全部项目 custom Skill。新 Agent 建议显式写 `false`，旧 Agent 缺失字段时按 `true`。直接模型仍可发现项目全部 Skill。Custom Skill 的 `depends` 不包含 `preload`。
- 内部 Agent 任务使用提交时的 Agent/Skill 快照；运行中修改 Agent、Skill 或 preload 只影响下一次任务。不要要求当前任务“立即读取”刚刚修改的声明。
- root task 权限是硬上限，命名/临时子 Agent 不能自行升级。当前 UI 项目 Agent 默认继承 root 权限；API 提供的 Agent profile 也只能进一步收紧。
- 外部 Agent 必须先调用 `window.__syncastAgent.initialize({ name, description?, emojiAvatar?, agentId? })` 初始化身份；首次加入会在项目成员中创建归属于当前登录用户的 Agent 身份，后续携带同一个 `agentId` 会认领并延续该身份。后续 action history 和由它发起的任务会记录这个外部操作者。内部 Agent 指的是 Syncast 内部 AI 对话执行者，不等同于外部 Agent。
- `syncast.agent.delegate` 默认使用专用自动化频道，不使用当前人工频道，也不携带历史。只有明确要延续某个频道上下文时，才传 `channelId/channelTitle` 和 `includeHistory: true`。
- `timeoutMs` 只限制等待时长，超时不会取消任务；保存 `ref` 后用 bridge 的 `notifications.list` 或 `wait(ref, { returnResult: true })` 补拉。
- 只有在需要精确读写项目数据，且不需要内部 Agent 业务推理时，才使用 `syncast.doc.graphql`。它是动态权限：query 是 read，mutation 会要求 edit 并落盘。
- 读取文档优先用 `syncast.docs.readForAgent`，它返回 canonical `docRead` 结构；章节读取使用真实 `sectionId`，分页使用 `nextCursor`，去重使用 `contextKey/loadedContextKeys`。
- GraphQL query/mutation 中的字段名必须使用 **camelCase**（如 `updatedAt`、`parentId`），不要用 Loro 内部的 snake_case（如 `updated_at`）。可先调用 `syncast.doc.graphql.explain` 获取正确示例。
- Agent-facing GraphQL 和资产 action 会过滤隐藏的 3D/model 资产（`glb` / `stl` / `fbx`），也不暴露存储、缓存、provider workflow 或底层资产创建接口。下载资产用 `syncast.assets.downloadUrls`，不要绕过 action layer 查实现字段。
- 输入中引用资产和文档时，使用真实 ID 对应的 `@{asset:<assetId>}`、`@{doc:<docId>|<displayName>}`、`@{doc-section:<docId>:<sectionId>|<displayName>}`。资产名称转 ID 用带 `query` 的 `syncast.assets.list`，再用 `syncast.assets.get` 校验唯一结果；不要猜 ID。
- 需要把项目资产交给外部 Agent 自行下载时，使用 `syncast.assets.downloadUrls` 或 CLI 便捷命令 `syncast project-agent asset-download-urls --asset-id ...` 获取临时签名 URL。Action 只返回链接和资产元数据，不在浏览器内下载文件；签名 URL 约 1 小时有效，外部 Agent 自行决定下载方式和保存位置。
- 需要把音频或视频的多个时间范围变成可分别引用的普通资产时，使用 `syncast.assets.materializeMediaSegments`，或 CLI 便捷命令 `syncast project-agent materialize-media-segments --asset-id ... --segments ...`。完整源区间直接复用源 `assetId`，不执行裁切；其它区间内容相同时复用已有 Asset，不重复创建或上传；只有不同内容才创建新资产。分段可不等长或重叠，单段失败不回滚其它成功资产。省略 `targetFolderId` / `--folder-id` 时跟随源资产目录；需要安全重试同一外部请求时传稳定的 `idempotencyKey` / `--idempotency-key`，相同 key 不得搭配不同输入。
- 必须先判断生成契约：把 CLI 当作生成 API、只需要文件或 Assets 交付时，使用 `syncast imagine [--project <id>]`；需要像用户在项目 ImagineChannel 中操作、保留提示词/参数/状态/结果消息时，使用 Action Bridge 的 `syncast.imagine.submitToChannel`。两者不可互相冒充。
- `syncast.imagine.submitToChannel` 必须传真实的现有 Imagine `channelId`；`syncast.imagine.submit` 可在未指定频道时自动解析或创建 Agent Imagine 频道。两者都走前端现有 Imagine 入队与消息持久化链路。
- 文档中的待生成 Imagine 块不是普通频道生成，也不能用 `syncast.doc.graphql` 手动改状态来代替。生成单块使用 `syncast.docs.imagineBlocks.submit { docId, blockId }`；等同顶部“批量生成待生成”使用 `syncast.docs.imagineBlocks.submitBatch { docId, blockIds? }`，省略 `blockIds` 时提交该文档全部待生成块。
- 文档批量提交会并行启动每个块的前端入队，不等待上一块完成；单块失败会出现在 `failures` 中，不阻塞其它块。每个返回的 `submissions[].ref` 可独立等待，最终资产仍由同一任务完成链路替换回原文档块。真实模型/provider 并发仍可能受服务端限流。
- 直接 CLI 的 `--folder` 支持名称或路径，缺失段由项目物化流程自动创建；已有真实 ID 时才使用高级参数 `--folder-id`。引用项目内图片使用 `--reference-asset <asset-id>`，CLI 内部解析存储对象。它会把提示词保存在 Asset `gen_params`，但不会创建 ImagineChannel 消息。
- Imagine draft compiler、typed wait/result、通知 action，以及被公开高层 action 取代的兼容 list/search/get 与 `agent.chat.submit` 仍属于内部实现。不要从 browser bridge 或 `project-agent run` 猜这些名字；使用 capabilities 返回的规范入口。
- 内部 Agent 的 `imagine` 工具与外部 CLI 是两套消费方契约：内部 Agent 在实时项目中只传 `asset_name` 和已确认存在的 `target_folder_id`；不传路径、名称或猜测 ID。它引用项目素材时只写真实 `asset_id` / `reference_type`，文件定位由执行器处理。
- 内部 Agent 只看到创作参数和上述项目语义字段；传输、provider、计费、回调、调试及源文件探测参数全部由 Syncast 管线负责。`media_understand` 中项目资产也只传 `asset_id`，项目外媒体才传 URL。时间轴 Skill 需要的 `source_slot_id` / `source_timeline_id` / `source_slot_label` 是受支持的项目来源元数据，可由内部 Agent 使用，但不属于外部 CLI 参数。
- 当内部 Agent 的 Imagine 需要确认时，应返回可点击的 Imagine 草稿，不自动执行，也不额外弹通用审批问题。用户已明确要求生成时，内部 Agent必须传 `request_review=false`；自动审查或完全访问权限会直接执行，包括明确授权的批量生成。
- 其它内部 Agent action 若返回 `agent_action.approval_requested` 通知，外部 Agent 可以自行决定是否批准，也可以先询问用户；若批准，调用 `syncast.agent.approval.respond` 或 CLI `syncast project-agent approval respond <approvalId> --approve`。审批只对该外部 Agent 托管的 pending request 有效。
- 外部 Agent 可用 `syncast.imagine.models`、`syncast.imagine.estimateCredits`、`syncast.imagine.optimizePrompt` 做发现和准备；随后根据是否需要项目频道历史，选择 Action Bridge submit 或直接 `syncast imagine`。
- 要在时间轴上排一组待生成块时，使用 `syncast.timeline.generationSlots.createBatch` 创建 draft slots。协作模式默认让用户检查后手动触发；用户已授权 Agent 代为执行时，可调用公开的 `syncast.timeline.generationSlots.submit`，其效果等同用户点击该 Slot 的生成按钮。
- 为节省上下文，可以建立本地临时项目计划文件缓存操作记录、资产名称与 ID、文档/章节 ID、Channel 和任务 ref；正式规范和剧本仍应写入 Syncast 项目文档。
- 消耗积分、写入项目、运行工作流或访问外部数据时，必须遵守当前 Agent 权限 profile 和 action approval 结果。用户已明确下达生成命令、且 profile 允许自动执行时，不要再追加一轮通用确认。
- 使用 `window.__syncastAgent.wait(ref)` 或 `window.__syncastAgent.notifications.list(...)` 判断工作是否完成；bridge 会强制限定当前项目，审批通知只对当前外部 Agent 自己托管的请求可见。
- 如果 action 返回 `not_in_project` 或 `doc_loading`，请让用户打开 Syncast 项目页并等待加载完成。

## 参考

- 平台介绍与创作工作流：[platform-guide.md](platform-guide.md)
- 完整 action 列表：[actions-reference.md](actions-reference.md)
- GraphQL 按模块参考：[graphql-reference/index.md](graphql-reference/index.md)
- 常见工作流：[workflows.md](workflows.md)
- 异常处理：[troubleshooting.md](troubleshooting.md)
