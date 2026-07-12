# Syncast Agent Actions 参考

这份文档是外部 Agent 操作 Syncast 网页的完整能力索引。当前外部 Agent Action Layer 显式暴露 **33 个高层 action**，分成两类：

- **项目业务能力**：项目上下文、GraphQL/Loro、时间轴草稿、频道、消息、资源、文档、任务、模型查询和内部 Agent 委托。
- **Bridge 便捷方法**：身份初始化、能力发现、等待/取结果、通知和历史；这些 transport 方法不再重复注册成 action。

内部 Syncast Agent 的 `imagine` 工具、页面 Imagine 提交实现、原始 chat submit、typed wait/result、通知 action 和兼容 list/search/get 别名不属于外部接口。它们不会出现在外部 capabilities 中，直接通过 browser/CLI bridge 调用会返回 `action_not_exposed`。

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

`window.__syncastAgent.capabilities()` 是唯一的外部能力发现入口，并支持 `group`、`actionName`、`disclosure` 精确筛选。不要通过 action 再套一层 capabilities。

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
- 内部 Agent 任务在提交时冻结本次 Agent、可用 Agent 目录与 Skill 快照；任务运行中修改项目 Agent/Skill 只影响下一次任务。
- root task 权限是硬上限。声明 Agent/命名 Agent 可以继承或进一步收紧，但不能提升 root 权限；当前项目 UI 中的 Agent 默认继承 root 权限。
- 声明 Agent 始终可按需加载全部内置 Skill；custom Skill 默认范围是已选 binding、依赖与 `alwaysApply` Skill。`allowLoadSkills=true` 才额外开放未选中的项目 custom Skill；新 Agent 建议显式写 `false`，旧 Agent 缺失字段按 `true`。binding 的 `preload` 只控制启动时是否注入完整说明，旧 binding 缺少 `preload` 时按 `true`。
- 长任务不靠反复读进度，优先使用 bridge 的 `wait`、`subscribe`、`notifications.list`；`wait(ref, { returnResult: true })` 可一次返回通知和结果。
- 大列表、长正文、消息 parts、模型 schema、GraphQL 字段说明都采用渐进式披露：默认返回 `summary/index`，需要时显式传 `disclosure`、`mode`、`include*`、`limit/offset` 展开。
- 生成图片或视频前必须校验 `@` 引用和 `references` 是否解析到真实资产，避免错误 ID 造成积分浪费。
- 任务应有 Channel 规划：复用少量稳定 Channel，不要每个任务都新建 Channel。

## 完整原子化能力

### 项目上下文

| Action | 权限 | 输入 | 输出 | 用途 |
| --- | --- | --- | --- | --- |
| `syncast.project.current` | read | 无 | `projectId`、activeChannelId、sync、permissions | 读取当前项目和权限状态，不暴露 Streams/路由实现 |
| `syncast.billing.summary` | read | 无 | 当前个人/团队积分余额、扣费策略 | 告诉外部 Agent 当前剩余积分与团队扣费规则 |
| `syncast.project.inspect` | read | `{ limit? }` | 必备概要：数量、当前上下文、最近/异常项、后续查询 action | 一次性巡检项目现场，不返回完整明细 |

### GraphQL / Loro 通用入口

这是最重要的通用原子能力。所有没有单独包装成快捷 action 的 Loro 项目数据读写，都应该通过 `syncast.doc.graphql` 走 monodoc GraphQL，而不是外部 Agent 自己改 Loro。

| Action | 权限 | 输入 | 输出 | 用途 |
| --- | --- | --- | --- | --- |
| `syncast.doc.graphql` | dynamic：query=read，mutation=edit | `{ query, variables?, idempotencyKey? }` | GraphQL result | 对当前 Loro doc 执行 query / mutation；能力元数据会标记 `permissionMode: "dynamic"` 和写入风险 |
| `syncast.doc.graphql.schema` | read | `{ mode?, module?, field?, query? }` | 默认模块索引；可查模块/字段/搜索 | 快速了解可用 GraphQL 面 |
| `syncast.doc.graphql.explain` | read | 无 | 常用查询示例 | 避免 Agent 猜字段 |

GraphQL 字段采用按模块渐进式披露。先调用 `syncast.doc.graphql.schema` 查看模块和字段摘要；需要写具体 query/mutation 时，再按需加载对应文件：

| 模块 | 参考文件 | 典型字段 |
| --- | --- | --- |
| Assets | [graphql-reference/assets.md](graphql-reference/assets.md) | `assetBrowse`、`assetsByIds`、`organizeAssets`、`moveAssetsToFolder` |
| Docs | [graphql-reference/docs.md](graphql-reference/docs.md) | `docRead`、`createDocPage`、`moveDocPage`、`patchDoc` |
| Channels | [graphql-reference/channels.md](graphql-reference/channels.md) | `channels`、`messages`、`createChannel`、`deleteMessage` |
| Timelines | [graphql-reference/timelines.md](graphql-reference/timelines.md) | `clipsByTimeline`、`generationSlotUsagesByAsset`、`createGenerationSlot` |
| Canvas | [graphql-reference/canvas.md](graphql-reference/canvas.md) | `pageWithElements`、`createElement`、`moveElement` |
| Agents | [graphql-reference/agents.md](graphql-reference/agents.md) | `agents`、`createAgent`、`deleteAgent` |
| Skills | [graphql-reference/skills.md](graphql-reference/skills.md) | `customSkills`、`createCustomSkill` |
| Prompt Templates | [graphql-reference/prompt-templates.md](graphql-reference/prompt-templates.md) | `customPromptTemplates`、`createCustomPromptTemplate` |
| Imagine Optimize Presets | [graphql-reference/imagine-optimize-presets.md](graphql-reference/imagine-optimize-presets.md) | `imagineOptimizePresets`、`createImagineOptimizePreset` |

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

存储对象、缩略图对象和校验值不在外部 Agent GraphQL 契约中。下载资产使用 `syncast.assets.downloadUrls`。

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

这些 action 是对 monodoc 时间轴 GraphQL 的 Agent 友好封装。外部 Agent 可以排一组待生成的 draft blocks，让用户在时间轴或 Imagine 面板里逐个手动触发生成；页面内部的 slot submit 不属于外部 API。

| Action | 权限 | 输入 | 输出 | 用途 |
| --- | --- | --- | --- | --- |
| `syncast.timelines.list` | read | `{ disclosure?, limit? }` | 时间轴概要列表、后续 action | 先查看项目有哪些时间轴 |
| `syncast.timeline.get` | read | `{ timelineId, disclosure?, includeNonSlotClips? }` | 时间轴概要、AI Slot 列表；`full` 时含完整 input | 查看某条时间轴上的占位块 |
| `syncast.timeline.create` | edit | `{ name, width?, height?, frameRate? }` | 新时间轴 | 创建可排布 AI Slot 的时间轴 |
| `syncast.timeline.generationSlots.create` | edit | `{ timelineId, slot }` | 新 draft slot | 添加单个待生成块，不立即扣费生成 |
| `syncast.timeline.generationSlots.createBatch` | edit | `{ timelineId?, timelineName?, width?, height?, frameRate?, slots[] }` | 时间轴和一组 draft slots | 一次性排一组镜头/角色/素材占位块 |
| `syncast.timeline.generationSlots.updateInput` | edit | `{ timelineId, clipId, patch }` | 更新后的 slot | 修改 prompt、modelType、references、targetAssetName、targetFolderId 等 |

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
  targetFolderId?: "existing-folder-id",
  frameDuration?: 96
}
```

`targetAssetName` 和 `targetFolderId` 会写入 slot input。用户后续从这个 slot 手动点击生成时，生成任务会沿用名称和已验证的文件夹 ID；生成完成后新资产会直接落入该目录。批量生成时名称会自动加序号。

### 频道和消息

| Action | 权限 | 输入 | 输出 | 用途 |
| --- | --- | --- | --- | --- |
| `syncast.channels.list` | read | `{ offset?, limit? }` | 分页频道列表和 messageCount | 查看所有频道 |
| `syncast.channels.current` | read | 无 | 当前激活频道 | 获取用户当前上下文 |
| `syncast.channels.resolve` | read | `{ channelId?, channelTitle?, type? }` | 已存在的频道摘要 | 自动找到最合适的现有频道；找不到时返回错误，不隐式创建 |
| `syncast.channels.create` | edit | `{ channelTitle, type? }` | 新频道摘要 | 通过 GraphQL 创建明确命名的频道 |
| `syncast.channel.messages.list` | read | `{ channelId?, offset?, limit?, includeParts?, includeToolCalls?, includeGeneratedAssets? }` | 默认消息摘要和分页；显式展开安全的 part/tool-call 摘要 | 读取频道消息 |
| `syncast.channel.message.get` | read | `{ channelId?, messageId, includeParts?, includeToolCalls?, includeGeneratedAssets? }` | 单条消息与安全 part 摘要 | 按消息 ID 读取结果 |

### 资源 Assets

| Action | 权限 | 输入 | 输出 | 用途 |
| --- | --- | --- | --- | --- |
| `syncast.assets.folders` | read | 无 | 递归文件夹树、根目录资产、未分类资产概要 | 先理解项目的文件夹整理结构 |
| `syncast.assets.browse` | read | `{ folderId?, path?, depth?, includeAssets?, offset?, limit?, includeSubfolders? }` | 默认只返回目录/计数；可按名称路径浏览并分页展开资产 | 像文件系统一样逐层浏览资源 |
| `syncast.assets.list` | read | `{ offset?, limit?, type?, query?, folderId?, includeSubfolders?, disclosure?, includeGenParams? }` | 分页资源摘要；可展开详情/genParams | 分页读取全部资源、按名称搜索或读取指定文件夹资源 |
| `syncast.assets.get` | read | `{ assetId, disclosure?, includeGenParams? }` | 默认单个资源摘要；可展开 folderIds/genParams | 精确读取资源 |
| `syncast.assets.frames` | read | `{ assetId, startTimeSeconds?, endTimeSeconds?, maxFrames?, includeContactSheet? }` | 关键帧/contact sheet | 按需读取视频视觉上下文 |
| `syncast.assets.downloadUrls` | read | `{ assetIds?, assetNames?, references? }` | `downloads[]` 临时签名 URL、`missing[]`、`failures[]` | 给外部 Agent 返回项目资产原文件下载链接，不在 Action 内下载 |

`folderId` 可传真实文件夹 ID，也可传特殊值：`__ROOT__` 表示文件夹外根目录资产，`__UNCATEGORIZED__` 表示既不在文件夹中也不在根目录索引中的未分类资产。需要包含子文件夹时设置 `includeSubfolders: true`。

生成前引用校验建议：先按名称搜索，再读唯一 ID：

```ts
const matches = await window.__syncastAgent.run("syncast.assets.list", {
  query: "主角参考图",
  limit: 10
});

await window.__syncastAgent.run("syncast.assets.get", {
  assetId: "<unique-asset-id>"
});
```

如果同名资产不唯一或 ID 不存在，停止生成并向用户说明。用于输入文本的常见引用格式：

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
| `syncast.docs.readForAgent` | read | `{ mode?, docId?, query?, target?, sectionId?, startBlockIndex?, endBlockIndex?, cursor?, loadedContextKeys?, skipLoaded?, maxChars?, changedSince?, limit?, offset? }` | 直接返回 canonical `docRead` 结构：`documents/outline/hits/content/contextKey/nextCursor` | 给外部 Agent 渐进式读取内部 Agent 写出的文档 |

`syncast.docs.readForAgent` 默认只返回索引，不返回正文，并且语义与 monodoc GraphQL 的 `docRead` 保持一致。典型流程是：先 `{ mode: "index" }` 看文档地图，再 `{ mode: "outline", docId }` 取真实 `sectionId` / `startBlockIndex` / `endBlockIndex`，最后 `{ mode: "content", docId, target: "section", sectionId, maxChars, loadedContextKeys }` 读取必要正文。继续读取使用返回的 `nextCursor`，去重使用 `contextKey`。全文只在用户明确需要时使用 `{ mode: "content", target: "full" }`。

文档写入如果是简单结构化 mutation，可用 `syncast.doc.graphql` 的 `createDocPage`、`updateDocPage`、`moveDocPage`、`moveDocPageBefore`、`moveDocPageAfter`、`deleteDocPage`、`patchDoc`、`setDocBlocks`。如果是业务性内容创作，优先用 `syncast.agent.delegate` 让内部 Agent 完成。

项目骨架或模板式初始化不要直接写 Loro、IndexedDB 或后端数据库。复用已发布的模板包时，走 App/Library/CLI 的项目模板包导入路线；如果用户明确要求外部 Agent 在当前项目内创建结构，使用 GraphQL 的幂等 mutation：Docs 用 `createDocPage(..., idempotencyKey)` 建树，再用 `patchDoc` 或 `setDocBlocks` 写内容；资源目录用 `ensureFolderPath(input: { path: "/Shots/Act 1" })` 创建或复用文件夹，需要搬运资产时再用 `moveAssetsToFolder` 的 `folderPath`。

### 任务和通知

| Action | 权限 | 输入 | 输出 | 用途 |
| --- | --- | --- | --- | --- |
| `syncast.tasks.list` | read | `{ status?: Array<"pending" \| "processing" \| "deferred" \| "completed" \| "failed" \| "cancelled">, type?: Array<"chat" \| "imagine">, channelId?, disclosure?, limit? }` | 默认运行中/失败任务摘要；可显式查 completed | 查看当前网页任务 |
| `syncast.task.status` | read | `{ taskId }` | 单个任务摘要 | 查询任务状态 |
| `syncast.task.cancel` | edit | `{ taskId }` | `{ cancelled: true }` | 走现有前端路径取消任务 |

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
| `syncast.imagine.optimizePrompt` | read | `{ prompt, modelType, params?, references?, firstFrameAssetId?, lastFrameAssetId?, locale? }` | `{ optimizedPrompt, rawOptimizedPrompt, modelType }` | 复用人类前端“优化提示词”按钮链路 |

`syncast.imagine.models` 默认等价于 `{ disclosure: "recommended", category: "all", includeSchemas: true }`。推荐图片模型优先 `nano-banana-2` 和 `oai-gpt-image-2`；图片一般只使用 2K，质量使用 `auto`。推荐视频生成模型只推荐 SeedDance 2.0；常规生成使用 `kittyvibe-seedance2.0pro`，快速预览或低成本需求使用 `kittyvibe-seedance2.0fast`。复杂动作、多主体、高运动量、怪兽或奇幻动作场景推荐 `kittyvibe-seedance2.0global`；快速/低成本 Global 预览推荐 `kittyvibe-seedance2.0fastglobal`。推荐音频模型分两类：音乐/配乐优先 `lyria-3-clip` / `lyria-3-pro`，音效/环境声/拟音/UI 声优先 `fal-ai/elevenlabs/sound-effects/v2`。这个音效模型最终应提交英文提示词；如果用户原文是中文，提交链路会自动翻成英文，但外部 Agent 仍应优先直接编写英文声音描述。Eleven 音效参数使用顶层 `durationSeconds`、`promptInfluence`、`loop`；不要暴露或传入 `output_format`，后端固定使用 MP3 44.1kHz / 128kbps。SeedDance 2.0 / Fast 只允许 720P，禁止 1080P。图片和视频超分/修复模型归在 `category: "upscale"`：`recraft-ai/recraft-crisp-upscale` 适合修复 Nano Banana Pro、GPT Image 2 等模型多轮编辑后的鳞片、噪点、颗粒和崩坏质感；`topaz/slp-2.5` 适合 AI 生成视频保真增强、去塑料感、提升人脸/材质/文字/logo 清晰度；`fal-ai/topaz/upscale/video` 是 fal 版 Starlight Precise 2.5，支持 `upscale_factor`、`target_fps`、`compression`、`noise`、`halo`、`grain`、`recover_detail`、`H264_output`；`topaz/ast-2` 适合创意细节重建和 prompt 引导增强，支持 `creativity`、`sharp`、`realism`、`prompt`。如果用户明确需要其它模型，再调用 `{ disclosure: "all", includeSchemas: false }` 查看其它模型名称；只有真正要使用某个非推荐模型时，才调用 `{ disclosure: "all", includeSchemas: true }` 获取完整 schema。

积分估算来自前端本地定价表，真实扣费以后端 reserve / settle 为准。外部 Agent 在批量生成前可调用 `syncast.billing.summary` 和 `syncast.imagine.estimateCredits`。

提示词优化会复用前端的 `imagine_prompt_optimize` Response API 和白名单过滤逻辑。输入中的资产引用应使用内部 token 形式 `@{asset:<assetId>}`；如果外部 Agent 只有人类可读名称，先用 `syncast.assets.list { query }` 找到唯一 assetId，再写入 prompt 或传入 `references`。优化结果会经过 sanitize：不允许模型新增未在原始 prompt / references / 首尾帧中的资产或文档 token。

外部 `project-agent` 只负责读取模型、估算和可选的提示词优化，不暴露页面内部的草稿编译或提交实现。真正生成统一使用：

```bash
syncast imagine --project <project-id> --folder <name-or-path> --name <asset-name> --prompt <text>
```

`--folder` 接受名称或路径，缺失目录段由项目物化流程创建；已有真实 ID 时可用高级参数 `--folder-id`。引用本地文件用 `--reference-image`，引用项目内图片 ID 用 `--reference-asset`；不要自行获取或传递项目存储 key，也不要把页面内部的目录、时间轴来源或 provider 回调字段当成 CLI 参数。

视频生成建议：

- 资产参考模式：先生成/整理角色、场景、物件参考图；生成视频时用 `references` 或 `@{asset:<assetId>}` 引入，并在 prompt 中明确剧情动作。
- 多镜头合成：使用 SeedDance 2.0 时，可把多个镜头组织到一个 4 到 15 秒片段中，描述不同运镜和转场，通常比逐镜头割裂生成更利于一致性。

### 内部 Agent 对话

| Action | 权限 | 输入 | 输出 | 用途 |
| --- | --- | --- | --- | --- |
| `syncast.agent.delegate` | edit | `{ goal?, prompt?, executor?, channelId?, channelTitle?, includeHistory?, wait?, timeoutMs? }` | `{ localTaskId, messageId, ref }`；等待时包含 notification/result | 外部 Agent 最推荐使用的业务委托入口 |
| `syncast.agent.approval.respond` | edit | `{ approvalId, approved, feedback? }` | 审批结果摘要 | 响应该外部 Agent 托管的内部 Agent action approval |

`syncast.agent.delegate` 允许只传 `goal`，它会自动创建/复用专用的自动化 chat channel，不会默认使用人类当前正在查看的频道，也不会默认携带整段频道历史。只有当外部 Agent 明确需要延续某个频道上下文时，才传 `channelId/channelTitle` 和 `includeHistory: true`。

`executor` 显式选择顶层执行器，建议外部 Agent 总是传入，避免任务受目标频道当前 UI 选择影响：

```json
{ "kind": "model", "model": "gemini-3.5-flash" }
```

直接模型会看到项目内全部独立 Agent，可按名称调用其中任意一个，也可从头创建临时子 Agent。

```json
{ "kind": "agent", "agentId": "<project-agent-id>" }
```

声明 Agent 只会看到显式绑定给它的命名 Agent，同时仍可从头创建临时子 Agent。项目 Agent 始终只有一份独立声明：既可直接使用，也可被其它 Agent 绑定；绑定不会复制声明或创建“子模式”。任何子执行都是叶节点，不继续展开自身绑定。省略命名 Agent ID 创建的临时子 Agent 不继承父 Agent 的指令或绑定技能。

先用 `syncast.doc.graphql` 查询 `agents { agents { id name model allowLoadSkills skills { skillId skillType preload } childAgents { childAgentId alias displayName whenToUse } } }` 再填写 `executor.agentId`。读取后更新 Agent 时必须保留 `allowLoadSkills` 和每个 binding 的 `preload`；前者缺失按旧 Agent 的 `true` 解释，后者缺失按旧 binding 的 `true` 解释。CLI 的全局 `--agent-id` 是外部操作者身份，不是项目内执行器；不要混用。

任务使用提交时快照。任务运行中修改 Agent 指令、Skill 内容、binding 或 preload，不会改变当前任务，只会从下一次发送开始生效。命名 Agent 的权限也不能超过 root task；当前 UI 创建的项目 Agent 没有独立持久化权限，默认继承 root。

`timeoutMs` 只控制等待窗口，超时不会取消任务，必须保留 `ref` 后续补拉。

可通过 CLI 查看机器可读字段：

```bash
syncast project-agent capabilities --action syncast.agent.delegate --disclosure full
```

当 `syncast.agent.delegate` 使用 `wait: true`，返回的 `data.result.text` 是内部 Agent 最终可见回复，`data.result.textPreview` 是短摘要。异步提交后需要结果时，使用 `window.__syncastAgent.wait(ref, { returnResult: true })` 或 CLI 的 `syncast project-agent wait --ref <json> --return-result`。

当内部 Agent 返回 `agent_action.approval_requested` 通知时，通知里会包含 `approvalId` / `respondAction` / `nextActions`。同一个外部 Agent 身份可以调用 `syncast.agent.approval.respond` 批准或拒绝；非 owner 会得到 `approval_actor_mismatch`。

## Browser Bridge 便捷 API

除了 `run(actionName, input, options)`，浏览器 bridge 还暴露以下便捷 API：

| API | 用途 |
| --- | --- |
| `window.__syncastAgent.initialize({ name, description?, emojiAvatar?, agentId?, metadata? })` | 初始化外部 Agent 身份；未初始化前不能执行 action |
| `window.__syncastAgent.currentAgent()` | 查看当前 session 中的外部 Agent 身份 |
| `window.__syncastAgent.clearAgent()` | 清除当前外部 Agent 身份，通常只用于调试 |
| `window.__syncastAgent.capabilities()` | 获取所有 action 名称、说明、权限 |
| `window.__syncastAgent.wait(ref, { timeoutMs?, returnResult? })` | 等待任务通知；`returnResult: true` 时同时补拉结果，Agent 文本位于 `data.result.text` |
| `window.__syncastAgent.subscribe(filter, callback)` | 订阅完成/失败通知，返回 unsubscribe |
| `window.__syncastAgent.notifications.list({ filter?, limit? })` | 拉取当前项目通知；审批通知只返回当前外部 Agent 自己托管的请求 |
| `window.__syncastAgent.history.list({ limit? })` | 拉取当前项目、当前外部 Agent 自己的公开 Action 调用历史 |
| `window.__syncastAgent.history.clear()` | 只清理当前外部 Agent 自己的调用历史，不影响页面内部或其他操作者 |

`history` 记录的是外部 action 调用历史，不是内部 Agent 的思考过程。内部 Agent 的业务输出仍然通过 channel message / docs / task result 读取；如果这个内部 Agent 任务是由外部 Agent 触发，任务摘要中的 `initiator` 会指向外部 Agent。

外部 Agent 身份会显示在项目成员列表里，并缩进挂在创建它的前端用户下方。它不直接参与权限计算，实际权限仍来自主人用户；这避免 Agent 身份绕过项目协作权限。

## CLI Bridge 对应命令

如果外部 Agent 不能直接在页面 main world 调用 `window.__syncastAgent`，使用 `syncast project-agent`：

注意：页面内部仍有 Imagine 提交实现，但它不属于 `window.__syncastAgent`
或 `project-agent` 的公共能力。CLI 生图统一使用
`syncast imagine --project <id> --folder <path> --name <name> --prompt <text>`；
缺失文件夹路径会在项目物化时自动创建。

| CLI | 对应浏览器 API |
| --- | --- |
| `syncast project-agent serve` | 启动本地窄口 bridge，等待项目页注册 |
| `syncast project-agent pages` | 查看已注册项目页 |
| `syncast project-agent initialize` | `window.__syncastAgent.initialize(...)` |
| `syncast project-agent capabilities` | `window.__syncastAgent.capabilities()` |
| `syncast project-agent run <action>` | `window.__syncastAgent.run(actionName, input, options)` |
| `syncast project-agent asset-download-urls --asset-id <id>` | `window.__syncastAgent.run("syncast.assets.downloadUrls", input)` |
| `syncast project-agent notifications --type <type>` | `window.__syncastAgent.notifications.list(...)` |
| `syncast project-agent approval respond <approvalId> --approve/--deny` | `window.__syncastAgent.run("syncast.agent.approval.respond", input)` |
| `syncast project-agent wait --ref <json>` | `window.__syncastAgent.wait(ref, options)` |
| `syncast project-agent history` | `window.__syncastAgent.history.list(...)` |

CLI Bridge 是 transport，不是新的业务 API。它不会替代本文件里的 action 语义，也不会暴露任意 `evaluate`。

CLI 默认输出稳定 JSON，便于外部 Agent 解析。内部 Agent 的返回文本读取 `data.result.text`；人工查看可加全局参数 `--format human` 获得摘要输出。

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
  executor: { kind: "model", model: "gemini-3.5-flash" },
  wait: false
});

const done = await window.__syncastAgent.wait(started.data.ref, {
  timeoutMs: 30 * 60 * 1000,
  returnResult: true
});

const agentText = done.data.result?.text;

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

### 3. 发起 Imagine 并归档到项目

```bash
syncast imagine \
  --project <project-id> \
  --folder "视觉/概念图" \
  --name "电影感概念图" \
  --prompt "请基于项目资源生成一张电影感概念图。"
```

这是唯一的外部 CLI 生成入口；生成产物由项目物化流程写入指定目录。

### 4. 通过资源名称找到唯一 ID

```ts
const refs = await window.__syncastAgent.run("syncast.assets.list", {
  query: "主视觉",
  limit: 10
});
```

同名结果不唯一时让用户选择；不要猜 assetId。

### 5. 超时后恢复

```ts
const notifications = await window.__syncastAgent.notifications.list({
  filter: { ref: savedRef },
  limit: 10
});

const result = await window.__syncastAgent.wait(savedRef, {
  returnResult: true
});
```

等待超时不等于任务失败。保存 `ref`，稍后用 bridge 通知和 `wait(..., { returnResult: true })` 补拉。

## 什么时候用哪条路径

- 想知道项目里有什么：用 `syncast.project.inspect`。
- 想让 Syncast 自己生成方案、文档、视频思路：用 `syncast.agent.delegate`。
- 想发起图片/视频/音频生成：用 `syncast imagine --project ... --folder ... --name ...`。
- 想知道余额或生成预计消耗：用 `syncast.billing.summary` / `syncast.imagine.estimateCredits`。
- 想查资源：用 `assets.list/get/browse`；想查文档：统一用 `docs.readForAgent`。
- 想做精确 Loro 数据读写：用 `syncast.doc.graphql`。
- 想等长任务完成：用 `window.__syncastAgent.wait(ref, { returnResult: true })`。
- 想恢复一个之前超时的任务：用 bridge 的 `notifications.list` + `wait(..., { returnResult: true })`。
