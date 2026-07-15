# Syncast Agent Action 工作流

## 平台创作路线

外部 Agent 接手 Syncast 项目时，应先把项目理解成一个创作工作台，而不是单个对话框。典型创作路线是：

1. 明确项目目标和交付物。
2. 建立或读取项目规范、剧本、角色设定、分镜说明。
3. 检查资源库中的参考图、视频、音频和生成结果。
4. 查看画布是否已有视觉结构、参考关系或分镜。
5. 查看时间轴是否已有剪辑结构。
6. 再决定是否委托内部 Agent、发起 Imagine 生成、运行工作流或编辑项目内容。

更完整的平台说明见 [platform-guide.md](platform-guide.md)。

## 接手模式判断

每次开始都先判断项目状态：

- 空项目：缺文档、资源、时间轴或明确规范。用户尚未给出结构时先讨论项目类型并让内部 Agent 返回规范草案；用户已给出完整正文时直接写入。
- 进行中项目：已有文档、资源、画布、时间轴或频道消息。先读现有内容，避免重复创建和重复生成。

随后判断模式：

- 协作创作：外部 Agent 和用户一起决策，高频汇报与确认，适合探索项目方向。
- 完全托管：用户已授权外部 Agent 在目标和预算内推进，适合明确任务。托管也要记录操作、校验引用、控制成本并阶段性汇报。

不明确时默认“协作创作”。

## Channel 规划

所有右侧 Imagine Panel / Agent Panel 里的 Channel 用户都能看到。为了让用户容易回看，外部 Agent 不要每个任务都新建 Channel。

推荐少量稳定 Channel：

- 项目规划：项目目标、规范、总体路线。
- 角色设定：角色资料、参考图、固定提示词。
- 场景设定：空间、光线、氛围、场景参考。
- 分镜与镜头：剧本、镜头、运镜、转场、多镜头片段。
- 生成实验：图片/视频生成尝试、失败原因、改进记录。
- 时间轴 Slot：待生成镜头块、关键帧和用户手动生成入口。

同一主题的连续任务复用同一个 Channel；只有没有合适 Channel 时才创建。

## 写入前先判断产物受众

创建或修改 custom Skill、Agent instructions、项目规范、提示词模板和文档前，必须先选择一种受众：

1. 外部 Agent 操作/集成指南：允许出现 CLI、Bridge、公开 Action、GraphQL 与接入方式；除非用户明确要求集成文档，否则不要写入项目内部。
2. 项目内部 Skill/Agent/规范/模板：只写业务规则、能力语义、模型和创作参数；不得出现 CLI、Bridge、standalone API、外部 Action 名、GraphQL mutation、传输或接入方式。
3. 用户阅读文档：使用产品和创作语言，不暴露接入实现；明确的开发/集成文档除外。

检查范围包括名称/标题、description、正文/instructions、模板字段和元数据。项目内部 Skill 可以写“使用项目内 Imagine 能力，模型为 `oai-gpt-image-2`，比例为 16:9，分辨率为 2K”，不能写“调用 `syncast.imagine.submit`，不能调用 standalone CLI”。

## 选择委托还是直接 GraphQL

- 创意构思、剧本设计、未知结构、需要项目内专业 Skill 判断：委托内部 Agent，并明确要求只返回草案或判断，不执行机械写入。
- 用户已提供完整正文、精确字段更新、确定文本替换、删除指定段落：直接使用项目 GraphQL；不要让内部 Agent 转述或改写。删除在执行方式上仍走 GraphQL，但按删除授权规则确认。
- 用户明确要求的同范围、非计费创建或编辑已经授权，不再确认一次。只有扩大范围、额外积分消耗、删除内容或目标不明确时询问；会造成未请求内容丢失的整段覆盖按删除处理，用户明确要求的精确替换不重复确认。

## 检查项目

```ts
await window.__syncastAgent.initialize({
  name: "Project Operator Agent",
  description: "负责检查项目、委托内部 Agent，并验收生成结果。",
  emojiAvatar: "🤖",
});

const overview = await window.__syncastAgent.run("syncast.project.inspect", {
  limit: 10
});
```

在决定下一步之前先调用它。返回内容足够外部 Agent 选择频道、检查资源、发现运行中的任务、了解积分概况并读取文档。

检查后先判断缺口：

- 缺项目规范：结构未知时委托内部 Agent 返回草案；用户已有完整内容时直接创建文档。
- 缺参考素材：优先检查资源文件夹或询问用户是否上传。
- 缺视觉结构：可查看画布并规划分镜/参考关系。
- 缺剪辑结构：可查看时间轴并判断是否需要先生成素材。
- 有运行中任务：先等待或读取通知，不要重复发起。

## 使用内部 Agent

外部 Agent 无法直接访问 Syncast 内部 Skill。剧本设计、视频创作、未知提示词结构、模型参数理解、工具调用建议等需要创作判断的任务，可以询问或委托内部 Agent。已有完整内容的创建和精确修改不属于委托范围。

项目里的每个自定义 Agent 都是独立声明，可直接运行，也可被其它 Agent 绑定为命名选项。调用前先选择顶层执行器：

- `executor: { kind: "model", model: "gemini-3.5-flash" }`：直接模型可发现项目全部 Agent，也可从头创建临时子 Agent。
- `executor: { kind: "agent", agentId: "..." }`：声明 Agent 只可按名称选择自己的绑定 Agent，也可从头创建临时子 Agent。

绑定不会复制 Agent，也不会形成运行时嵌套；命名或临时子执行都是叶节点。临时子 Agent 不继承声明 Agent 的指令和绑定技能。选择 `agentId` 前，用 Agents GraphQL 查询真实 ID 与 `childAgents`。

查询目录时同时读取 `allowLoadSkills` 和 `skills { skillId skillType preload }`。全部内置 Skill 始终可按需加载；`allowLoadSkills=false` 只排除未选中的项目 custom Skill，不影响已选 binding、依赖与 `alwaysApply` Skill。`preload=false` 表示 Skill 仍可用，只是不在启动时注入完整说明。新 Agent 建议显式写 `allowLoadSkills=false`；旧 Agent 缺失时按 `true`，旧 binding 缺少 `preload` 时也按 `true`。read-modify-write 必须原样保留这些字段。

每次发送都会冻结本次 Agent、命名 Agent 目录和 Skill 快照。任务运行中修改配置只影响下一次任务；子 Agent 权限也始终受 root task 权限上限约束。

```ts
const started = await window.__syncastAgent.run("syncast.agent.delegate", {
  goal: [
    "请基于当前项目状态，返回适合短视频项目的项目规范草案。",
    "请把规范拆成：整体风格、角色、场景、分镜计划、常用提示词。",
    "如果项目方向是电影感 3D 动画或三维渲染二维感，请在风格规范中明确参考 `$cinematic-3d-animation`，并按该 Skill 写整体视觉原则、固定提示词骨架、正向词、反向控制和验收标准。",
    "只负责创作判断并返回正文，不要创建或修改项目文档。"
  ].join("\n"),
  executor: { kind: "model", model: "gemini-3.5-flash" },
  wait: false
});
```

不要把所有工作都塞进一个内部 Agent 对话。按项目规划、角色、场景、镜头、生成实验等主题拆开，避免上下文过长。

## 委托内部 Agent

```ts
const started = await window.__syncastAgent.run("syncast.agent.delegate", {
  goal: "请基于当前项目资源返回一个视频项目方案草案，只负责创作判断，不要执行项目写入。",
  executor: { kind: "agent", agentId: "<project-agent-id>" },
  wait: false
});
```

默认情况下，`syncast.agent.delegate` 会使用专用自动化频道，不会把任务发进用户当前正在看的人工频道，也不会携带整段频道历史。只有要延续某个明确频道上下文时，才传 `channelId/channelTitle` 和 `includeHistory: true`。

然后等待：

```ts
const done = await window.__syncastAgent.wait(started.data.ref, {
  timeoutMs: 30 * 60 * 1000,
  returnResult: true
});
```

内部 Agent 的最终可见文本在 `done.data.result.text`，短摘要在 `done.data.result.textPreview`。浏览器自动化应只把最终文本、任务终态和产物 ID 返回外部上下文，不直接返回包含 parts/tool 轨迹的完整 `done`。CLI 的 `syncast project-agent wait --return-result` 默认输出精简 JSON；仅调试时使用 `--full-result`。

如果委托只返回了草案，外部 Agent 在确认目标文档后用项目 GraphQL 写入。随后读取并验收文档：

```ts
await window.__syncastAgent.run("syncast.docs.readForAgent", {
  changedSince: started.data.ref.startedAt
});
```

`syncast.docs.readForAgent` 返回的是 canonical `docRead` 结构；后续读取章节正文时使用 `outline` 中的真实 `sectionId`，继续分页使用 `nextCursor`，重复上下文去重使用 `content.contextKey` / `loadedContextKeys`。

## 创建或更新 custom Skill

custom Skill 写入使用固定流程，不能只看 mutation 是否成功：

1. **查重**：先查询 `customSkills` 的 `id/name/description/alwaysApply/depends`；按规范化后的完整名称查同名 Skill。唯一同名项应更新而不是重复创建；多个候选或名称意图不明确时先确认。
2. **确认元数据**：明确 `alwaysApply`、每个 `depends { skillId skillType }` 是否真实存在，以及是否需要绑定某个 Agent。`depends` 不含 `preload`；Agent binding 才有 `preload`。
3. **内容隔离**：在写入前扫描 `name`、`description`、`instructions`、依赖与相关 Agent 元数据。项目内部 Skill 不得包含 CLI、Bridge、standalone API、外部 Action/GraphQL 名或接入方式，除非用户明确要求集成 Skill。
4. **创建/更新**：通过 `syncast.doc.graphql` 执行 mutation。用户给出的完整 instructions 原样写入，不经内部 Agent 改写。
5. **完整回读**：按返回的真实 `id` 查询 `customSkill(id)`，必须回读 `id name description instructions iconName alwaysApply depends { skillId skillType } createdAt updatedAt`。
6. **引用验真**：提取 `instructions` 和 `description` 中的 `@{doc:<docId>|...}` / `@{doc-section:<docId>:<sectionId>|...}`；逐个用文档查询校验真实 `docId`，章节引用还要从 outline 校验真实 `sectionId`。不得把标题、slug 或猜测值当 ID。
7. **绑定验收**：需要绑定 Agent 时，更新前先读取并保留 `allowLoadSkills`、全部 `skills { skillId skillType preload }` 和其它声明字段；更新后回读 Agent，确认新 binding、`preload` 和原有绑定都正确。`alwaysApply=true` 不等于必须额外绑定。

详细 query/mutation 见 [graphql-reference/skills.md](graphql-reference/skills.md)。最终验收必须同时覆盖正文、描述和元数据；任何一处仍残留外部接入内容都不算完成。

## 创建或精确修改文档/模板

- 新建文档使用 `ensureDocPages`，以稳定、项目级唯一的 `logicalKey` 标识业务目标，并在同一次 mutation 传入有效 `initialMarkdown`。多篇父子文档放入同一批次，通过 `parentLogicalKey` 建树。
- 查现有标题/路径只用于选择是否显式采用已有真实 `existingDocId`；不得按标题静默复用。默认禁止同父级同名，确需同名新页时才显式设置 `allowDuplicateTitle=true`。
- `success=true` 且 `ready=true` 已完成写入后置条件检查，不做机械 `docRead`。只有语义审校、引用验真或图标/封面等额外元数据验收时再回读。修改已有有效正文使用真实 `docId` 调 `patchDoc`。
- 用户阅读文档检查 `title/description/body`；项目规范还要检查 `docKind/specInjectionMode`。所有 `@` 文档引用使用真实 ID。
- 提示词模板先查同名项，写入后回读完整 `name/description/targetTypes/inputTemplate/promptTemplate/variables`，对所有文本字段做受众隔离检查。
- 用户给出完整正文、精确字段更新或确定删除目标时直接 GraphQL，不把机械搬运委托给内部 Agent。删除仍按授权规则确认；精确替换不重复确认。

## 校验 @ 引用

发起图片/视频生成前，先解析资产引用。支持的常用文本格式：

```text
@{asset:<assetId>}
@{doc:<docId>|<displayName>}
@{doc-section:<docId>:<sectionId>|<displayName>}
```

资产引用校验：先按名称搜索，再用唯一 ID 读取详情。

```ts
const matches = await window.__syncastAgent.run("syncast.assets.list", {
  query: "主角参考图",
  limit: 10
});

await window.__syncastAgent.run("syncast.assets.get", {
  assetId: "<unique-asset-id>"
});
```

如果没有唯一真实 assetId，停止生成并向用户说明。不要带着错误 ID 发起图片或视频任务。

## 发起 Imagine

先读与本次产物相关的项目规范，再查当前 Assets 目录，确定用户可读的目标路径和资源名称：

```ts
await window.__syncastAgent.run("syncast.docs.readForAgent", {
  mode: "index"
});

const tree = await window.__syncastAgent.run("syncast.assets.browse", {
  depth: 2,
  includeAssets: false
});
```

发起前建议先估算积分：

```ts
await window.__syncastAgent.run("syncast.imagine.estimateCredits", {
  modelType: "nano-banana-2",
  count: 2
});
```

项目内生成统一走公开的 Imagine submit Action。未指定现有频道时默认使用
`syncast.imagine.submit`，让前端自动解析或创建 Agent Imagine Channel：

```bash
syncast project-agent run syncast.imagine.submit \
  --input '{"modelType":"nano-banana-2","prompt":"请基于选中的项目资源生成一个电影感项目概念板。","targetAssetName":"项目概念板A","targetFolderId":"<verified-folder-id>"}'
```

`targetFolderId` 必须来自当前项目的真实目录；没有明确目录时省略。引用项目内图片时把真实 Asset ID 放入 Action `references`。如果所需本地文件尚未进入项目，先请用户导入目标项目，不得因此切回直接 CLI。只有用户明确选择项目外独立资产时，才使用 `syncast-cli` 的 standalone 生成命令。

模型建议：

- 图片：优先 Nano Banana 2 或 OpenAI GPT Image 2；一般只使用 2K，质量使用 `auto`。
- 视频：优先 SeedDance 2.0，默认 Fast 模式；SeedDance 2.0 / Fast 只允许 720P，禁止 1080P。
- 除非用户明确要求，或内部 Agent 给出充分理由，不使用其它模型。

视频生成策略：

- 资产参考模式：为角色、场景和关键物件准备参考图，生成时 `@` 这些资产，并用文本描述剧情和动作。
- 多镜头合成生成：使用 SeedDance 2.0 时，可以把多个镜头组织到一个 4 到 15 秒片段中，以获得更好一致性和更自然转场。

## 排布时间轴 AI 占位 Slot

如果用户希望先看到一组待生成镜头/角色块，再由人类逐个点击生成，使用 draft slot，不要直接批量扣费：

```ts
await window.__syncastAgent.run("syncast.timeline.generationSlots.createBatch", {
  timelineName: "角色设定草案",
  frameRate: 30,
  slots: [
    {
      slotLabel: "S1",
      title: "男主角",
      prompt: "生成男主角角色设定图，半身，电影感。",
      modelType: "nano-banana-2",
      targetAssetName: "男主角A",
      targetFolderId: "existing-folder-id",
      durationSeconds: 4
    },
    {
      slotLabel: "S2",
      title: "女主角",
      prompt: "生成女主角角色设定图，半身，电影感。",
      modelType: "nano-banana-2",
      targetAssetName: "女主角A",
      targetFolderId: "existing-folder-id",
      durationSeconds: 4
    }
  ]
});
```

用户或已获授权的外部 Agent 从 Slot 生成时，都会沿用 Slot input 里的 `targetAssetName` 和 `targetFolderId`。协作模式可停在 draft；需要代用户触发时调用：

```ts
await window.__syncastAgent.run("syncast.timeline.generationSlots.submit", {
  timelineId: "<timeline-id>",
  clipId: "<slot-clip-id>",
});
```

## 生成文档内 Imagine 块

文档里已经有待生成 Imagine 块时，不要把参数复制到直接 CLI 或普通 ImagineChannel；那样不会把结果回填原块。单块生成：

```ts
await window.__syncastAgent.run("syncast.docs.imagineBlocks.submit", {
  docId: "<doc-id>",
  blockId: "<imagine-block-id>"
});
```

等同人类点击文档顶部“批量生成待生成”：

```ts
const batch = await window.__syncastAgent.run(
  "syncast.docs.imagineBlocks.submitBatch",
  { docId: "<doc-id>" }
);
```

如只生成选中的块，传 `blockIds`。批量 action 并行启动各块入队，返回独立的 `submissions[].ref`；单块失败记录在 `failures`，不会阻塞其它块。后续分别等待这些 ref，并检查原文档块是否已被生成资产替换。

## 关键帧生成

复杂镜头或场景变化建议先生成关键帧。关键帧需要严格命名和记录引用：

- 命名示例：`镜头03_关键帧01_街角远景`、`镜头03_关键帧02_角色近景`。
- 缓存每个关键帧对应的 assetId、场景位置、角色角度和参考来源。
- 如果关键帧之间空间、角度或角色不一致，先修正关键帧，再生成视频。

## 临时项目计划文件

为了节省上下文，可以在本地维护临时计划文件，记录：

- 操作记录和决策原因。
- 资产名称到 assetId 的映射。
- 文档 ID、章节 ID 和用途。
- Channel 名称/ID、任务 ref、生成结果位置。

正式项目规范、剧本、角色和场景内容仍应写入 Syncast 文档；本地文件只做外部 Agent 速查缓存。

## 读取频道结果

```ts
await window.__syncastAgent.run("syncast.channel.messages.list", {
  channelId: "channel-id",
  limit: 20
});
```

如果已知消息 ID：

```ts
await window.__syncastAgent.run("syncast.channel.message.get", {
  channelId: "channel-id",
  messageId: "message-id"
});
```

## 超时后恢复

如果等待调用超时，保留返回的 `ref` 或原始的 `started.data.ref`。

```ts
await window.__syncastAgent.notifications.list({
  filter: { ref: savedRef },
  limit: 10
});
```

如果暂时没有通知，稍后用同一个 ref 等待并补拉结果：

```ts
await window.__syncastAgent.wait(savedRef, {
  returnResult: true
});
```

## 汇报给用户

完成查看或任务后，不要只返回 action 的原始 JSON。应按项目创作语境汇报：

- 当前项目处于哪个阶段。
- 找到了哪些文档、资源、画布、时间轴或生成结果。
- 内部 Agent 或 Imagine 产物在哪里。
- 是否消耗了积分，或下一步是否可能消耗积分。
- 下一步建议是什么，哪些需要用户确认。
