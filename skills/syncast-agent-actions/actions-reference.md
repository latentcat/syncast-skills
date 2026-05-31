# Syncast Agent Actions 参考

这份文档是外部 Agent 操作 Syncast 网页的完整能力索引。当前 Action Layer 显式暴露 **49 个 action**，分成两类：

- **原子化能力**：项目上下文、GraphQL/Loro、时间轴、频道、消息、资源、文档、任务、通知、Imagine、内部 Agent 对话。
- **便捷封装路径**：把多个原子能力组合成更适合 Agent 使用的入口，例如 `project.inspect`、`agent.delegate`、`imagine.submit`、`docs.readForAgent`、`wait(..., { returnResult: true })`。

如果你还不了解 Syncast 平台本身，先读 [platform-guide.md](platform-guide.md)。本文件只解释“怎么调用能力”，不替代平台创作流程判断。

如果需要检查运行中的真实能力清单，直接在 Syncast 项目页执行：

```ts
await window.__syncastAgent.initialize({
  name: "Project Operator Agent",
  description: "负责检查项目、委托内部 Agent，并验收生成结果。",
  emojiAvatar: "🤖",
});

await window.__syncastAgent.capabilities();
```

`window.__syncastAgent.capabilities()` 是浏览器 bridge 的便捷入口，等价于默认查询完整 action 清单；如果需要按 `group`、`actionName` 或 `disclosure` 精确筛选，再通过 `window.__syncastAgent.run("syncast.project.capabilities", input)` 调用 action 版本。

所有 action 都通过以下方式调用：

```ts
await window.__syncastAgent.initialize({
  name: "Project Operator Agent",
  description: "负责检查项目、委托内部 Agent，并验收生成结果。",
  emojiAvatar: "🤖",
});

await window.__syncastAgent.run(actionName, input, options);
```

返回值统一为：

```ts
{ ok: true, data: unknown } | { ok: false, error: { code, message, details? } }
```

## 原则

- 外部 Agent 是操作者，不直接绕过 Syncast 前端业务路径。
- 外部 Agent 必须先调用 `window.__syncastAgent.initialize({ name, description?, emojiAvatar?, agentId?, metadata? })` 初始化身份；首次加入会在项目成员里创建一个归属于当前登录用户的 Agent 身份，后续携带同一个 `agentId` 会认领并延续该身份。初始化后 action history 会记录具体 Agent 名称、描述和 Emoji 头像，Agent 发起的长任务也会携带 `initiator`。
- 内部 Agent 是 Syncast 流程内的 AI 对话任务执行者；外部 Agent 是与人类等同的项目操作者。`syncast.agent.delegate` 的语义是“外部 Agent 发起，内部 Agent 执行”。
- 任务身份分两层记录：`initiator` 记录谁发起任务（当前支持外部 Agent / 调试器 / 系统，后续可扩展到协作用户 A/B），`task.type` 或 action 名记录任务业务类型（内部 Agent 对话、Imagine 等）。
- Loro 读写统一走项目已有 monodoc GraphQL 或前端封装好的任务路径。
- 写操作必须经过项目权限检查；GraphQL mutation 会先等待 Streams 初始同步，再执行 `saveAndSync`。
- 长任务不靠反复读进度，优先使用 `wait`、`subscribe`、`notifications.list` 获取完成通知，再按 `ref` 补拉结果。
- 大列表、长正文、消息 parts、模型 schema、GraphQL 字段说明都采用渐进式披露：默认返回 `summary/index`，需要时显式传 `disclosure`、`mode`、`include*`、`limit/offset` 展开。
- 生成图片或视频前必须校验 `@` 引用和 `references` 是否解析到真实资产，避免错误 ID 造成积分浪费。
- 任务应有 Channel 规划：复用少量稳定 Channel，不要每个任务都新建 Channel。

## 完整原子化能力

### 项目上下文

| Action | 权限 | 输入 | 输出 | 用途 |
| --- | --- | --- | --- | --- |
| `syncast.project.current` | read | 无 | `projectId`、`streamId`、route、activeChannelId、sync、permissions | 读取当前项目和权限状态 |
| `syncast.billing.summary` | read | 无 | 当前个人/团队积分余额、扣费策略 | 告诉外部 Agent 当前剩余积分与团队扣费规则 |
| `syncast.project.capabilities` | read | `{ disclosure?, actionName?, group? }` | 默认 action 分组索引；可展开详情 | 让 Agent 自发现能力 |
| `syncast.project.inspect` | read | `{ limit? }` | 必备概要：数量、当前上下文、最近/异常项、后续查询 action | 一次性巡检项目现场，不返回完整明细 |

### GraphQL / Loro 通用入口

这是最重要的通用原子能力。所有没有单独包装成快捷 action 的 Loro 项目数据读写，都应该通过 `syncast.doc.graphql` 走 monodoc GraphQL，而不是外部 Agent 自己改 Loro。

| Action | 权限 | 输入 | 输出 | 用途 |
| --- | --- | --- | --- | --- |
| `syncast.doc.graphql` | dynamic：query=read，mutation=edit | `{ query, variables?, idempotencyKey? }` | GraphQL result | 对当前 Loro doc 执行 query / mutation；能力元数据会标记 `permissionMode: "dynamic"` 和写入风险 |
| `syncast.doc.graphql.schema` | read | `{ mode?, module?, field?, query? }` | 默认模块索引；可查模块/字段/搜索 | 快速了解可用 GraphQL 面 |
| `syncast.doc.graphql.explain` | read | 无 | 常用查询示例 | 避免 Agent 猜字段 |

已显式提示的 GraphQL 字段包括：

- 查询：`assets`、`asset`、`assetBrowse`、`channels`、`channel`、`messages`、`message`、`docPages`、`docPage`、`docRead`、`docBlocks`、`docMarkdown`、`timelines`、`timeline`、`clipsByTimeline`、`clip`
- 写入：`createAsset`、`updateAsset`、`createChannel`、`updateChannel`、`createChatMessage`、`createImagineMessage`、`createDocPage`、`updateDocPage`、`patchDoc`、`setDocBlocks`、`createTimeline`、`createGenerationSlot`、`updateGenerationSlotInput`、`setGenerationSlotOutput`

**GraphQL 字段命名**：monodoc GraphQL 对外暴露 **camelCase** 字段名，不要使用 Loro 内部的 snake_case。常见对应关系：

| GraphQL（正确） | Loro 内部（勿直接写进 query） |
| --- | --- |
| `createdAt` | `created_at` |
| `updatedAt` | `updated_at` |
| `parentId` | `parent_id` |
| `messageType` | `message_type` |
| `taskId` | `task_id` |
| `assetId` | `asset_id` |
| `folderIds` | `folder_ids` |
| `remoteFilename` | `remote_filename` |

常用查询示例（也可调用 `syncast.doc.graphql.explain` 获取）：

```ts
// 文档列表
`query { docPages { docPages { id title parentId updatedAt } } }`

// 频道列表
`query { channels { channels { id name type updatedAt } } }`

// 频道消息
`query($channelId: String!) { messages(channelId: $channelId, limit: 20) { id messageType createdAt } }`

// 资源列表
`query { assets { assets { id name extension updatedAt } totalCount } }`

// 文档 Markdown
`query($docId: String!) { docMarkdown(docId: $docId) }`
```

更完整的 schema 可看 `assets/monodoc-graphql-schema/monodoc.graphql`，或先调用 `syncast.doc.graphql.schema` / `syncast.doc.graphql.explain`。

Mutation 示例：

```ts
await window.__syncastAgent.run("syncast.doc.graphql", {
  query: `mutation($input: ChannelCreateInput!) {
    createChannel(input: $input) { id name type }
  }`,
  variables: { input: { name: "项目方案", type: "chat" } },
  idempotencyKey: "external-agent:planning-channel"
});
```

### 时间轴 / AI 占位 Slot

这些 action 是对 monodoc 时间轴 GraphQL 的 Agent 友好封装。外部 Agent 可以排一组待生成的 draft blocks，让用户在时间轴或 Imagine 面板里逐个手动触发生成；除非用户明确授权，不建议外部 Agent 直接替用户提交所有 slot 生成。

| Action | 权限 | 输入 | 输出 | 用途 |
| --- | --- | --- | --- | --- |
| `syncast.timelines.list` | read | `{ disclosure?, limit? }` | 时间轴概要列表、后续 action | 先查看项目有哪些时间轴 |
| `syncast.timeline.get` | read | `{ timelineId, disclosure?, includeNonSlotClips? }` | 时间轴概要、AI Slot 列表；`full` 时含完整 input | 查看某条时间轴上的占位块 |
| `syncast.timeline.create` | edit | `{ name, width?, height?, frameRate? }` | 新时间轴 | 创建可排布 AI Slot 的时间轴 |
| `syncast.timeline.generationSlots.create` | edit | `{ timelineId, slot }` | 新 draft slot | 添加单个待生成块，不立即扣费生成 |
| `syncast.timeline.generationSlots.createBatch` | edit | `{ timelineId?, timelineName?, width?, height?, frameRate?, slots[] }` | 时间轴和一组 draft slots | 一次性排一组镜头/角色/素材占位块 |
| `syncast.timeline.generationSlots.updateInput` | edit | `{ timelineId, clipId, patch }` | 更新后的 slot | 修改 prompt、modelType、references、targetAssetName 等 |
| `syncast.timeline.generationSlots.submit` | edit | `{ timelineId, clipId, channelId?, wait?, timeoutMs? }` | Imagine submit 结果 | 明确授权后，直接从 slot input 发起生成 |

`slot` 的核心字段：

```ts
{
  slotLabel?: "S1",
  title?: "男主角",
  prompt: "生成男主角角色设定图",
  modelType?: "nano-banana-2",
  params?: { aspect_ratio: "16:9" },
  // Eleven sound effects only: top-level params, not nested in params.
  durationSeconds?: 4,
  promptInfluence?: 0.3,
  loop?: true,
  references?: [{ assetId: "..." }],
  targetAssetName?: "男主角A",
  frameDuration?: 96
}
```

`targetAssetName` 会写入 slot input。用户后续从这个 slot 手动点击生成时，生成任务会携带该名称；生成完成后新资产会自动命名为这个值。批量生成时会自动加序号，避免多个产物同名。

### 频道和消息

| Action | 权限 | 输入 | 输出 | 用途 |
| --- | --- | --- | --- | --- |
| `syncast.channels.list` | read | `{ offset?, limit? }` | 分页频道列表和 messageCount | 查看所有频道 |
| `syncast.channels.current` | read | 无 | 当前激活频道 | 获取用户当前上下文 |
| `syncast.channels.resolve` | read；创建时要求 edit | `{ channelId?, channelTitle?, purpose?, type?, createIfMissing? }` | 频道摘要 | 自动找到最合适频道，必要时创建 |
| `syncast.channels.create` | edit | `{ channelTitle?, purpose?, type? }` | 新频道摘要 | 通过 GraphQL 创建频道 |
| `syncast.channel.messages.list` | read | `{ channelId?, offset?, limit?, includeParts?, partMode?, includeToolCalls?, includeGeneratedAssets? }` | 默认消息摘要和分页；显式展开 parts/tool calls | 读取频道消息 |
| `syncast.channel.message.get` | read | `{ channelId?, messageId, includeParts?, partMode?, includeToolCalls?, includeGeneratedAssets? }` | 默认单条消息摘要；可展开完整 parts | 按消息 ID 读取结果 |

### 资源 Assets

| Action | 权限 | 输入 | 输出 | 用途 |
| --- | --- | --- | --- | --- |
| `syncast.assets.stats` | read | 无 | 总数、扩展名分布、文件夹概要、最近资源 | 快速判断项目资源情况 |
| `syncast.assets.folders` | read | 无 | 递归文件夹树、根目录资产、未分类资产概要 | 先理解项目的文件夹整理结构 |
| `syncast.assets.browse` | read | `{ folderId?, depth?, includeAssets?, offset?, limit?, includeSubfolders? }` | 默认只返回目录/计数；可分页展开资产 | 像文件系统一样逐层浏览资源 |
| `syncast.assets.list` | read | `{ offset?, limit?, type?, query?, folderId?, includeSubfolders?, disclosure?, includeGenParams? }` | 分页资源摘要；可展开详情/genParams | 分页读取全部资源或指定文件夹资源 |
| `syncast.assets.search` | read | `{ query?, limit? }` | 匹配资源 | 按名称搜索资源 |
| `syncast.assets.get` | read | `{ assetId, disclosure?, includeGenParams? }` | 默认单个资源摘要；可展开 folderIds/genParams | 精确读取资源 |
| `syncast.assets.downloadUrls` | read | `{ assetIds?, assetNames?, references? }` | `downloads[]` 临时签名 URL、`missing[]`、`failures[]` | 给外部 Agent 返回项目资产原文件下载链接，不在 Action 内下载 |
| `syncast.assets.resolveReferences` | read | `{ references: Array<string \| { assetId?, name? }> }` | 资源摘要列表 | 把 Agent 的名称/ID 输入转成生成任务可用引用 |

`folderId` 可传真实文件夹 ID，也可传特殊值：`__ROOT__` 表示文件夹外根目录资产，`__UNCATEGORIZED__` 表示既不在文件夹中也不在根目录索引中的未分类资产。需要包含子文件夹时设置 `includeSubfolders: true`。

生成前引用校验建议：

```ts
const resolved = await window.__syncastAgent.run("syncast.assets.resolveReferences", {
  references: ["主角参考图", { assetId: "asset-id" }]
});
```

如果返回中有任意引用未解析到真实资产，停止生成并向用户说明。用于输入文本的常见引用格式：

获取远端下载链接时使用 `syncast.assets.downloadUrls`。它只为当前项目能解析到的资产返回原文件签名 URL，不下载文件；URL 约 1 小时有效：

```ts
await window.__syncastAgent.run("syncast.assets.downloadUrls", {
  assetIds: ["asset-id"],
  references: ["主角参考图"]
});
```

```text
@{asset:<assetId>}
@{doc:<docId>|<displayName>}
@{doc-section:<docId>:<sectionId>|<displayName>}
```

### 文档 Docs

| Action | 权限 | 输入 | 输出 | 用途 |
| --- | --- | --- | --- | --- |
| `syncast.docs.list` | read | `{ offset?, limit? }` | 分页文档索引 + 字数/标题统计 | 查看项目文档树入口 |
| `syncast.docs.get` | read | `{ docId, disclosure?, includeContent?, limit? }` | 默认文档摘要；可选 outline/Markdown | 读取单个文档 |
| `syncast.docs.search` | read | `{ query?, limit? }` | 匹配文档摘要 | 搜索标题和 Markdown 内容 |
| `syncast.docs.readForAgent` | read | `{ mode?, docId?, query?, target?, sectionId?, startBlockIndex?, endBlockIndex?, cursor?, loadedContextKeys?, skipLoaded?, maxChars?, changedSince?, limit?, offset? }` | 直接返回 canonical `docRead` 结构：`documents/outline/hits/content/contextKey/nextCursor` | 给外部 Agent 渐进式读取内部 Agent 写出的文档 |

`syncast.docs.readForAgent` 默认只返回索引，不返回正文，并且语义与 monodoc GraphQL 的 `docRead` 保持一致。典型流程是：先 `{ mode: "index" }` 看文档地图，再 `{ mode: "outline", docId }` 取真实 `sectionId` / `startBlockIndex` / `endBlockIndex`，最后 `{ mode: "content", docId, target: "section", sectionId, maxChars, loadedContextKeys }` 读取必要正文。继续读取使用返回的 `nextCursor`，去重使用 `contextKey`。全文只在用户明确需要时使用 `{ mode: "content", target: "full" }`。

文档写入如果是简单结构化 mutation，可用 `syncast.doc.graphql` 的 `createDocPage`、`updateDocPage`、`patchDoc`、`setDocBlocks`。如果是业务性内容创作，优先用 `syncast.agent.delegate` 让内部 Agent 完成。

### 任务和通知

| Action | 权限 | 输入 | 输出 | 用途 |
| --- | --- | --- | --- | --- |
| `syncast.tasks.list` | read | `{ status?: Array<"pending" \| "processing" \| "deferred" \| "completed" \| "failed" \| "cancelled">, type?: Array<"chat" \| "imagine">, channelId?, disclosure?, limit? }` | 默认运行中/失败任务摘要；可显式查 completed | 查看当前网页任务 |
| `syncast.task.status` | read | `{ taskId }` | 单个任务摘要 | 查询任务状态 |
| `syncast.task.wait` | read | `{ ref, timeoutMs? }` | 完成/失败通知 | 等待任意任务完成 |
| `syncast.task.result` | read | `{ ref }` | 消息或任务摘要 | 按 ref 补拉结果 |
| `syncast.task.cancel` | edit | `{ taskId }` | `{ cancelled: true }` | 走现有前端路径取消任务 |
| `syncast.notifications.list` | read | `{ filter?, limit?, disclosure? }` | 默认通知摘要；可展开 data | 超时后恢复、补拉已完成事件 |

通知类型包括：

- `task.completed`、`task.failed`、`task.cancelled`
- `agent_chat.completed`、`agent_chat.failed`
- `imagine.completed`、`imagine.failed`
- `doc.changed`

### Imagine 生成

| Action | 权限 | 输入 | 输出 | 用途 |
| --- | --- | --- | --- | --- |
| `syncast.imagine.models` | read | `{ disclosure?, category?, includeSchemas? }`，`category` 支持 `image` / `video` / `audio` / `upscale` / `all` | 渐进式模型披露 | 默认只看推荐模型；需要时再展开完整模型 |
| `syncast.imagine.estimateCredits` | read | `{ modelType, params?, count? }` | 本地积分估算、当前余额、是否足够 | 发起生成前预估成本 |
| `syncast.imagine.draftMarkdown` | read | `{ items: {"1": {model_type, prompt, ...}} }` 或 `{ drafts: [{ index?, title?, targetAssetName?, modelType?, prompt?, params?, input? }] }`；每项最终必须有 `model_type` 和 `prompt` | `markdown` + `items`，语言标记为 `imagine` | 给用户多个待生成方案，不创建任务、不扣费 |
| `syncast.imagine.optimizePrompt` | read | `{ prompt, modelType, params?, references?, firstFrameAssetId?, lastFrameAssetId?, locale? }` | `{ optimizedPrompt, rawOptimizedPrompt, modelType }` | 复用人类前端“优化提示词”按钮链路 |
| `syncast.imagine.submit` | edit | `{ modelType, prompt, params?, durationSeconds?, promptInfluence?, loop?, references?, count?, channelId?, targetAssetName?, optimizePrompt?, optimizePromptLocale?, sourceSlot?, wait?, timeoutMs? }` | `{ messageId, taskIds, ref, billingEstimate, validation, promptOptimization, submitted }`；等待时包含 notification/result | 可选先优化提示词，再通过现有前端 Imagine 入队路径发起生成 |
| `syncast.imagine.submitToChannel` | edit | 同 `submit` | 同 `submit` | 明确表达“提交到解析后的频道” |
| `syncast.imagine.wait` | read | `{ ref, timeoutMs? }` | 完成/失败通知 | 等待生成完成 |
| `syncast.imagine.result` | read | `{ ref }` | 生成消息摘要 | 读取生成结果 |

`syncast.imagine.models` 默认等价于 `{ disclosure: "recommended", category: "all", includeSchemas: true }`。推荐图片模型优先 `nano-banana-2` 和 `oai-gpt-image-2`；图片一般只使用 2K，质量使用 `auto`。推荐视频生成模型只推荐 SeedDance 2.0；常规生成使用 `kittyvibe-seedance2.0pro`，快速预览或低成本需求使用 `kittyvibe-seedance2.0fast`。复杂动作、多主体、高运动量、怪兽或奇幻动作场景推荐 `kittyvibe-seedance2.0global`；快速/低成本 Global 预览推荐 `kittyvibe-seedance2.0fastglobal`。推荐音频模型分两类：音乐/配乐优先 `lyria-3-clip` / `lyria-3-pro`，音效/环境声/拟音/UI 声优先 `fal-ai/elevenlabs/sound-effects/v2`。这个音效模型最终应提交英文提示词；如果用户原文是中文，提交链路会自动翻成英文，但外部 Agent 仍应优先直接编写英文声音描述。Eleven 音效参数使用顶层 `durationSeconds`、`promptInfluence`、`loop`；不要暴露或传入 `output_format`，后端固定使用 MP3 44.1kHz / 128kbps。SeedDance 2.0 / Fast 只允许 720P，禁止 1080P。图片和视频超分/修复模型归在 `category: "upscale"`：`recraft-ai/recraft-crisp-upscale` 适合修复 Nano Banana Pro、GPT Image 2 等模型多轮编辑后的鳞片、噪点、颗粒和崩坏质感；`topaz/slp-2.5` 适合 AI 生成视频保真增强、去塑料感、提升人脸/材质/文字/logo 清晰度；`topaz/ast-2` 适合创意细节重建和 prompt 引导增强，支持 `creativity`、`sharp`、`realism`、`prompt`。如果用户明确需要其它模型，再调用 `{ disclosure: "all", includeSchemas: false }` 查看其它模型名称；只有真正要使用某个非推荐模型时，才调用 `{ disclosure: "all", includeSchemas: true }` 获取完整 schema。

积分估算来自前端本地定价表，真实扣费以后端 reserve / settle 为准。外部 Agent 在批量生成前应先调用 `syncast.billing.summary` 和 `syncast.imagine.estimateCredits`；`syncast.imagine.submit` 返回的 `billingEstimate` 可用于记录本次预计消耗。

如果只是给用户建议、备选方案或需要用户确认，不要调用 `syncast.imagine.submit`。用 `syncast.imagine.draftMarkdown` 返回 Markdown，或自行输出同等格式；每个 item 必须包含非空 `model_type` 和 `prompt` / `prompt_raw`。引用资产优先写 `references: [{ "asset_id": "...", "reference_type": "image" }]`；兼容 `reference_assets` / `referenceAssetIds` 等旧别名，但不要编造 asset id：

```imagine
{
  "1": {"asset_name": "角色方案 A", "model_type": "nano-banana-2", "prompt": "...", "references": [{"asset_id": "asset-id", "reference_type": "image"}]},
  "2": {"asset_name": "角色方案 B", "model_type": "nano-banana-2", "prompt": "...", "references": [{"asset_id": "asset-id", "reference_type": "image"}]}
}
```

这段内容在 Syncast Agent 记录中会显示为“待生成 Imagine 参数”控件，并展示引用到的参考素材。用户可以复制参数，也可以手动打开 Imagine 编辑器；在用户提交前不会生成、不会扣费。

Seedance 2.0 待生成视频参数如果要使用首尾帧，优先写 `first_frame` / `last_frame`；如果沿用 Ark 兼容 `content[]`，媒体项必须包含真实 `asset_id` 和 `role: "first_frame"` / `"last_frame"`。Syncast 会把它导入到首尾帧模式；`role: "reference_image"` / `"reference_video"` / `"reference_audio"` 会导入到智能参考模式。

提示词优化会复用前端的 `imagine_prompt_optimize` Response API 和白名单过滤逻辑。输入中的资产引用应使用内部 token 形式 `@{asset:<assetId>}`；如果外部 Agent 手里只有人类可读名称，先调用 `syncast.assets.resolveReferences` 解析成 assetId，再写入 prompt 或传入 `references`。优化结果会经过 sanitize：不允许模型新增未在原始 prompt / references / 首尾帧中的资产或文档 token。

`syncast.imagine.submit` 支持 `optimizePrompt: true`。开启后会先执行同一条优化链路，再用优化后的 prompt 提交生成；返回中的 `promptOptimization` 会包含 original / optimized / raw optimized，`submitted` 会记录实际提交给生成器的 `modelType`、`prompt`、`params`、`references`、`count` 等参数。对 `fal-ai/elevenlabs/sound-effects/v2`，这条链路还会在提交前自动把中文 prompt 翻成英文，并把顶层 `durationSeconds`、`promptInfluence`、`loop` 合并为后端模型输入；`output_format` 由后端固定，不是 Agent 参数。外部 Agent 如果想模拟人类“先写参数参考提示词 -> 点优化提示词 -> 再生成”的流程，优先使用这个封装。

`syncast.imagine.submit` 内置生成前校验，不需要外部 Agent 额外调用 preflight。校验会在原始 prompt、优化后 prompt 和最终模型输入上检查 `@{asset:...}`、`@{doc:...}`、`@{doc-section:...}` 是否可解析。提交成功返回的 `validation` 至少应满足：

```ts
validation.ok === true
validation.leftoverTokens.length === 0
validation.unresolvedMentions.length === 0
```

`submitted.modelPrompt` 是实际模型提示词文本，`submitted.finalModelInput` 是最终入队的模型输入。外部 Agent 应优先看这些字段，而不是只看原始 `submitted.prompt`。

`references` 可以传资源 ID、资源名称，或 `{ assetId, name }`。Action 会自动解析资源，不需要外部 Agent 自己拼前端内部数据。

`targetAssetName` 用于指定生成完成后创建的 Syncast 资产名称，例如 `{ targetAssetName: "男主角A" }`。它不传给模型，只作为前端任务元数据进入完成处理链路；如果生成复用了同一个 `remoteFilename` 的既有资产，也会把该资产重命名为目标名称。批量生成会自动追加序号。

视频生成建议：

- 资产参考模式：先生成/整理角色、场景、物件参考图；生成视频时用 `references` 或 `@{asset:<assetId>}` 引入，并在 prompt 中明确剧情动作。
- 多镜头合成：使用 SeedDance 2.0 时，可把多个镜头组织到一个 4 到 15 秒片段中，描述不同运镜和转场，通常比逐镜头割裂生成更利于一致性。

### 内部 Agent 对话

| Action | 权限 | 输入 | 输出 | 用途 |
| --- | --- | --- | --- | --- |
| `syncast.agent.chat.submit` | edit | `{ prompt, channelId?, channel?, attachments?, rawPrompt?, includeHistory?, wait?, notify?, timeoutMs? }` | `{ localTaskId, messageId, ref }`；等待时包含 notification/result | 通过现有前端聊天路径发起内部 Agent 对话 |
| `syncast.agent.chat.wait` | read | `{ ref, timeoutMs? }` | 完成/失败通知 | 等待内部 Agent 回复 |
| `syncast.agent.chat.result` | read | `{ ref }` | assistant 消息摘要 | 读取内部 Agent 最终消息 |
| `syncast.agent.delegate` | edit | `{ goal?, prompt?, channel?, attachments?, includeHistory?, wait?, notify?, timeoutMs? }` | `{ localTaskId, messageId, ref }`；等待时包含 notification/result | 外部 Agent 最推荐使用的业务委托入口 |

`syncast.agent.delegate` 允许只传 `goal`，它会自动创建/复用专用的自动化 chat channel，不会默认使用人类当前正在查看的频道，也不会默认携带整段频道历史。只有当外部 Agent 明确需要延续某个频道上下文时，才传 `channelId/channelTitle` 和 `includeHistory: true`。

## Browser Bridge 便捷 API

除了 `run(actionName, input, options)`，浏览器 bridge 还暴露以下便捷 API：

| API | 用途 |
| --- | --- |
| `window.__syncastAgent.initialize({ name, description?, emojiAvatar?, agentId?, metadata? })` | 初始化外部 Agent 身份；未初始化前不能执行 action |
| `window.__syncastAgent.currentAgent()` | 查看当前 session 中的外部 Agent 身份 |
| `window.__syncastAgent.clearAgent()` | 清除当前外部 Agent 身份，通常只用于调试 |
| `window.__syncastAgent.capabilities()` | 获取所有 action 名称、说明、权限 |
| `window.__syncastAgent.wait(ref, { timeoutMs?, returnResult? })` | 等待任务通知；`returnResult: true` 时同时补拉结果 |
| `window.__syncastAgent.subscribe(filter, callback)` | 订阅完成/失败通知，返回 unsubscribe |
| `window.__syncastAgent.notifications.list({ filter?, limit? })` | 拉取已记录通知 |
| `window.__syncastAgent.history.list({ limit? })` | 拉取 Agent Action 调用历史 |
| `window.__syncastAgent.history.clear()` | 清空本页 session 中的调用历史 |

`history` 记录的是外部 action 调用历史，不是内部 Agent 的思考过程。内部 Agent 的业务输出仍然通过 channel message / docs / task result 读取；如果这个内部 Agent 任务是由外部 Agent 触发，任务摘要中的 `initiator` 会指向外部 Agent。

外部 Agent 身份会显示在项目成员列表里，并缩进挂在创建它的前端用户下方。它不直接参与权限计算，实际权限仍来自主人用户；这避免 Agent 身份绕过项目协作权限。

## CLI Bridge 对应命令

如果外部 Agent 不能直接在页面 main world 调用 `window.__syncastAgent`，使用 `syncast project-agent`：

| CLI | 对应浏览器 API |
| --- | --- |
| `syncast project-agent serve` | 启动本地窄口 bridge，等待项目页注册 |
| `syncast project-agent pages` | 查看已注册项目页 |
| `syncast project-agent initialize` | `window.__syncastAgent.initialize(...)` |
| `syncast project-agent capabilities` | `window.__syncastAgent.capabilities()` |
| `syncast project-agent run <action>` | `window.__syncastAgent.run(actionName, input, options)` |
| `syncast project-agent asset-download-urls --asset-id <id>` | `window.__syncastAgent.run("syncast.assets.downloadUrls", input)` |
| `syncast project-agent wait --ref <json>` | `window.__syncastAgent.wait(ref, options)` |
| `syncast project-agent history` | `window.__syncastAgent.history.list(...)` |

CLI Bridge 是 transport，不是新的业务 API。它不会替代本文件里的 action 语义，也不会暴露任意 `evaluate`。

## 开发者调试器

在 Syncast 的 `Preferences -> Developer -> Agent Action Debugger` 打开调试器后，项目页右下角会显示一个类似 GraphQL Debugger 的面板：

- 左侧按「封装好的操作符」和「原子操作符」分组展示；原子操作符再按项目、GraphQL/Loro、频道、资源、文档、任务通知、Imagine、内部 Agent 分类。
- 右侧可以填写 `Input JSON` 和 `Options JSON`，由人类手动模拟 Agent 调用。
- 执行结果会直接显示在 `Result` 区域。
- `History` 区域会记录外部 Agent、手动调试器、`wait` 调用产生的历史，包括输入、options、返回值、耗时、成功/失败状态。
- 手动执行会自动带上 `{ "debugSource": "manual" }`，外部 Agent 默认记录为 `agent`。

## 便捷封装路径

### 1. 一次性巡检项目

```ts
const overview = await window.__syncastAgent.run("syncast.project.inspect", {
  limit: 20
});
```

返回的是“概要索引”，不是完整项目 dump：包含项目上下文、各类数量、资源类型分布、当前/最近频道、最近文档、运行中任务、最近失败任务、模型数量和少量示例，并告诉 Agent 下一步应该调用哪些 action 深挖。外部 Agent 开始任何操作前都应该先走这条。

### 2. 委托内部 Agent 完成业务任务

```ts
const started = await window.__syncastAgent.run("syncast.agent.delegate", {
  goal: "请基于当前项目资源生成一个视频项目方案，并写入项目文档。",
  wait: false,
  notify: true
});

const done = await window.__syncastAgent.wait(started.data.ref, {
  timeoutMs: 30 * 60 * 1000,
  returnResult: true
});

const docsIndex = await window.__syncastAgent.run("syncast.docs.readForAgent", {
  mode: "index",
  changedSince: started.data.ref.startedAt
});

// 选中需要验收的 docId 后，再按章节读取必要正文。
const outline = await window.__syncastAgent.run("syncast.docs.readForAgent", {
  mode: "outline",
  docId: "<docId>"
});
```

这是外部 Agent 的主路径：外部 Agent 负责指挥和验收，内部 Agent 负责 Syncast 业务产出。

### 3. 发起 Imagine 并等待结果

```ts
const started = await window.__syncastAgent.run("syncast.imagine.submit", {
  modelType: "nano-banana-2",
  prompt: "请基于项目资源生成一张电影感概念图。",
  references: [{ name: "主视觉.png" }],
  count: 2,
  wait: false
});

const result = await window.__syncastAgent.wait(started.data.ref, {
  timeoutMs: 20 * 60 * 1000,
  returnResult: true
});
```

### 4. 通过资源名称快速转引用

```ts
const refs = await window.__syncastAgent.run("syncast.assets.resolveReferences", {
  references: ["主视觉.png", { name: "角色设定.png" }]
});
```

用于避免外部 Agent 手动猜 assetId。

### 5. 超时后恢复

```ts
const notifications = await window.__syncastAgent.run(
  "syncast.notifications.list",
  {
    filter: { ref: savedRef },
    limit: 10
  }
);

const result = await window.__syncastAgent.run("syncast.task.result", {
  ref: savedRef
});
```

等待超时不等于任务失败。保存 `ref`，稍后用通知列表或 result action 补拉。

## 什么时候用哪条路径

- 想知道项目里有什么：用 `syncast.project.inspect`。
- 想让 Syncast 自己生成方案、文档、视频思路：用 `syncast.agent.delegate`。
- 想发起图片/视频/音频生成：用 `syncast.imagine.submit`。
- 想知道余额或生成预计消耗：用 `syncast.billing.summary` / `syncast.imagine.estimateCredits`。
- 想查资源、文档、频道、任务：用对应 list/search/get 原子 action。
- 想做精确 Loro 数据读写：用 `syncast.doc.graphql`。
- 想等长任务完成：用 `window.__syncastAgent.wait(ref, { returnResult: true })`。
- 想恢复一个之前超时的任务：用 `syncast.notifications.list` + `syncast.task.result`。
