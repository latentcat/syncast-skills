---
name: syncast-agent-actions
description: 通过浏览器内 Agent Action Layer 操作已打开的 Syncast 项目。当外部 Agent 需要检查项目、委托 Syncast 内部 Agent、发起 Imagine 生成、排布时间轴 AI 占位 Slot、等待任务完成，或读取项目文档/资源/频道、获取资产远端下载链接时使用；禁止绕过前端或直接点击 UI。
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
```

随后仍然按本 skill 选择 actionName / input / options，再通过 CLI 发送：

```bash
syncast project-agent run syncast.project.inspect --input '{"limit":20}'
syncast project-agent run syncast.agent.delegate --input '{"goal":"请整理项目方案并写入项目文档。","executor":{"kind":"model","model":"gemini-3.5-flash"},"wait":false}'
syncast project-agent wait --ref '{"kind":"agent_chat","projectId":"..."}' --return-result
syncast project-agent asset-download-urls --asset-id "asset-id"
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
- 用户继续操作 UI；外部 Agent 不通过 Playwright click/fill 操作业务 UI。
- 如果当前环境允许原生、可写的 Playwright `page.evaluate`，也可以直接调用 `window.__syncastAgent.run`；如果工具声明 `evaluate` 只读，必须使用 CLI Bridge 或其它正式 bridge。

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
- 项目 Agent 是一份独立声明，可直接作为 `executor: { kind: "agent", agentId }` 使用，也可绑定为另一 Agent 的命名选项。直接模型 `executor: { kind: "model", model }` 能看到项目全部 Agent；声明 Agent 只看到自己的绑定项。两者都能从头创建不继承父指令的临时子 Agent，且所有子执行均为叶节点。
- 外部 Agent 在选择声明 Agent 前，先通过 Agents GraphQL 查询真实 `id` 与 `childAgents`。CLI 的 `--agent-id` 是外部操作者身份，只有 action input 中的 `executor.agentId` 才选择项目内 Agent。
- 查询或更新 Agent 时必须请求并保留 `allowLoadSkills` 与 `skills { skillId skillType preload }`。所有内置 Skill 始终可按需加载；custom binding 决定关闭扩展发现时仍可用的项目 Skill。`preload` 只决定是否在启动时注入完整说明。旧 binding 缺少 `preload` 时按 `true`，新 binding 应显式写 `false` 或 `true`。
- `allowLoadSkills` 是历史兼容字段：`false` 只禁止发现未选中的项目 custom Skill，仍保留已选 custom Skill、依赖、`alwaysApply` Skill 和全部内置 Skill；`true` 扩展到全部项目 custom Skill。新 Agent 建议显式写 `false`，旧 Agent 缺失字段时按 `true`。直接模型仍可发现项目全部 Skill。Custom Skill 的 `depends` 不包含 `preload`。
- 内部 Agent 任务使用提交时的 Agent/Skill 快照；运行中修改 Agent、Skill 或 preload 只影响下一次任务。不要要求当前任务“立即读取”刚刚修改的声明。
- root task 权限是硬上限，命名/临时子 Agent 不能自行升级。当前 UI 项目 Agent 默认继承 root 权限；API 提供的 Agent profile 也只能进一步收紧。
- 外部 Agent 必须先调用 `window.__syncastAgent.initialize({ name, description?, emojiAvatar?, agentId? })` 初始化身份；首次加入会在项目成员中创建归属于当前登录用户的 Agent 身份，后续携带同一个 `agentId` 会认领并延续该身份。后续 action history 和由它发起的任务会记录这个外部操作者。内部 Agent 指的是 Syncast 内部 AI 对话执行者，不等同于外部 Agent。
- `syncast.agent.delegate` 默认使用专用自动化频道，不使用当前人工频道，也不携带历史。只有明确要延续某个频道上下文时，才传 `channelId/channelTitle` 和 `includeHistory: true`。
- `notify` 只是旧调用兼容字段，任务通知始终会记录，新调用可省略。`timeoutMs` 只限制等待时长，超时不会取消任务；保存 `ref` 后用 notifications/result 补拉。
- 只有在需要精确读写项目数据，且不需要内部 Agent 业务推理时，才使用 `syncast.doc.graphql`。它是动态权限：query 是 read，mutation 会要求 edit 并落盘。
- 读取文档优先用 `syncast.docs.readForAgent`，它返回 canonical `docRead` 结构；章节读取使用真实 `sectionId`，分页使用 `nextCursor`，去重使用 `contextKey/loadedContextKeys`。
- GraphQL query/mutation 中的字段名必须使用 **camelCase**（如 `updatedAt`、`parentId`），不要用 Loro 内部的 snake_case（如 `updated_at`）。可先调用 `syncast.doc.graphql.explain` 获取正确示例。
- Agent-facing GraphQL 和资产 action 会过滤隐藏的 3D/model 资产（`glb` / `stl` / `fbx`）。如果 `asset`、`assetId`、文件夹、时间轴、画布或消息里的资产字段返回 null / 缺失，表示该资产不对外部 Agent 可见，不要绕过 action layer 去读 raw Loro 或构造 mutation 反查 `remoteFilename`。
- 输入中引用资产和文档时，使用真实 ID 对应的 `@{asset:<assetId>}`、`@{doc:<docId>|<displayName>}`、`@{doc-section:<docId>:<sectionId>|<displayName>}`；发起图片/视频生成前必须用 `syncast.assets.resolveReferences` 等 action 校验引用。
- 需要把项目资产交给外部 Agent 自行下载时，使用 `syncast.assets.downloadUrls` 或 CLI 便捷命令 `syncast project-agent asset-download-urls --asset-id ...` 获取临时签名 URL。Action 只返回链接和资产元数据，不在浏览器内下载文件；签名 URL 约 1 小时有效，外部 Agent 自行决定下载方式和保存位置。
- 发起生成时使用 `syncast.imagine.submit`，它会走和手动 Imagine 面板一致的前端入队路径；需要指定生成完成后的资源名称时传 `targetAssetName`，一般开启 `optimizePrompt: true`。提交会内置生成前校验，返回 `validation` 和 `submitted.modelPrompt/finalModelInput`；如果 `validation.ok` 不为 true，或存在 `leftoverTokens/unresolvedMentions`，不得认为生成输入有效。
- 当内部 Agent 或 action bridge 返回 `agent_action.approval_requested` 通知时，外部 Agent 可以自行决定是否批准，也可以先询问用户；若批准，调用 `syncast.agent.approval.respond` 或 CLI `syncast project-agent approval respond <approvalId> --approve`。审批只对该外部 Agent 托管的 pending request 有效。
- 如果只是给用户多个生成建议、让用户挑选，或用户尚未确认扣费生成，不要调用 `syncast.imagine.submit`。使用 `syncast.imagine.draftMarkdown`，或在内部 Agent 回复中输出语言标记为 `imagine` 的 fenced code block；内容使用和 `imagine(items)` 完全相同的 JSON 对象，并确保每个 item 都有非空 `model_type` 和 `prompt` / `prompt_raw`。引用资产优先写 `references: [{ asset_id, reference_type }]`；兼容旧别名 `reference_assets` / `referenceAssetIds`，但不要编造 asset id。Syncast 会把它渲染成“待生成 Imagine 参数”控件，显示参考素材，用户可复制或手动打开 Imagine 编辑器。
- Seedance 2.0 待生成视频参数如果要使用首帧/尾帧，优先写 `first_frame` / `last_frame`；兼容 `content[]` 时必须在媒体项里带真实 `asset_id` 和 `role: "first_frame"` / `"last_frame"`。Syncast 会导入到首尾帧模式；普通参考素材 `reference_image/video/audio` 会导入到智能参考模式。
- 一镜到底、视频续接、镜头接镜头、或任何要求上一段尾帧与下一段首帧严格对齐的 Seedance 2.0 首尾帧任务，必须开启几何对齐：调用 `syncast.imagine.submit` 时写 `params: { preserve_geometry: true }`；输出 fenced `imagine` JSON 或 `syncast.imagine.draftMarkdown.items` 时，在对应 item 的模型参数里写 `"preserve_geometry": true`。不要用智能参考模式替代首尾帧模式来做严格续接。
- 如果 Agent 自己负责连续生成一镜到底片段，发起每一段真实生成后必须等待完成，再读取结果或抽取尾帧安排下一段：可在 `syncast.imagine.submit` 传 `wait: true, timeoutMs: 30 * 60 * 1000`，或提交后调用 `window.__syncastAgent.wait(ref, { returnResult: true })` / `syncast.imagine.wait`。只有生成完成后才能把该段结果当作下一段首帧来源。
- 图片模型优先 Nano Banana 2 / OpenAI GPT Image 2；图片生成一般只使用 2K，质量使用 `auto`，这是默认最佳组合。
- 音频中的“音乐 / 歌曲 / BGM / 配乐”优先 Lyria；音频中的“音效 / 环境声 / 拟音 / UI 声 / 冲击音 / loop ambience”优先 `fal-ai/elevenlabs/sound-effects/v2`。这个音效模型最终应使用英文提示词；如用户输入中文，提交链路会自动翻成英文，但外部 Agent 仍应默认直接编写英文音效提示词。Agent Action 传参时使用顶层 `durationSeconds`、`promptInfluence`、`loop`，不要把这些音效参数塞进二级 `params`；`output_format` 不对外暴露，后端固定使用 MP3 44.1kHz / 128kbps。
- 图片超分/修复模型在 `syncast.imagine.models` 的 `category: "upscale"` 中；`recraft-ai/recraft-crisp-upscale` 适合修复 Nano Banana Pro / GPT Image 2 多轮编辑后的鳞片、噪点、颗粒和崩坏质感，不需要 prompt。
- 视频生成模型默认推荐 SeedDance 2.0，除非用户明确要求，否则用 Fast 模式；SeedDance 2.0 / Fast 只允许使用 720P，禁止使用 1080P。复杂动作、多主体、高运动量、怪兽或奇幻动作场景优先 `kittyvibe-seedance2.0global`；快速/低成本 Global 预览用 `kittyvibe-seedance2.0fastglobal`。
- 视频超分/修复模型也在 `category: "upscale"` 中；`topaz/slp-2.5` 适合 AI 生成视频保真增强、去塑料感、提升人脸/材质/文字/logo 清晰度；`fal-ai/topaz/upscale/video` 是 fal 版 Starlight Precise 2.5，适合明确要 fal 或需要 `upscale_factor`、`target_fps`、`compression`、`noise`、`halo`、`grain`、`recover_detail`、`H264_output` 参数时使用；`topaz/ast-2` 适合创意细节重建和 prompt 引导增强。Topaz 直连只用 `target_resolution: "1080p" | "4k"`；fal 版可直接传 `upscale_factor` 覆盖自动换算；Astra 可额外传 `creativity`、`sharp`、`realism`、`prompt`。
- 要在时间轴上排一组待生成块时，使用 `syncast.timeline.generationSlots.createBatch` 创建 draft slots，让用户逐个手动触发；只有在用户明确确认扣费生成时才调用 `syncast.timeline.generationSlots.submit`。
- 为节省上下文，可以建立本地临时项目计划文件缓存操作记录、资产名称与 ID、文档/章节 ID、Channel 和任务 ref；正式规范和剧本仍应写入 Syncast 项目文档。
- 任何会消耗积分、写入项目、运行工作流或访问设备的动作，必须先向用户说明风险并获得确认。
- 使用 `window.__syncastAgent.wait(ref)` 或 `syncast.notifications.list` 判断工作是否完成。
- 如果 action 返回 `not_in_project` 或 `doc_loading`，请让用户打开 Syncast 项目页并等待加载完成。

## 参考

- 平台介绍与创作工作流：[platform-guide.md](platform-guide.md)
- 完整 action 列表：[actions-reference.md](actions-reference.md)
- GraphQL 按模块参考：[graphql-reference/index.md](graphql-reference/index.md)
- 常见工作流：[workflows.md](workflows.md)
- 异常处理：[troubleshooting.md](troubleshooting.md)
