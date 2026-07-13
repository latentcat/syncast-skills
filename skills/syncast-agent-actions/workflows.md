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

- 空项目：缺文档、资源、时间轴或明确规范。先和用户讨论项目类型，再让内部 Agent 创建项目规范。
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

## 检查项目

```ts
await window.__syncastAgent.initialize({
  name: "Project Operator Agent",
  description: "负责检查项目、委托内部 Agent，并验收生成结果。",
  emojiAvatar: "🤖",
});

const overview = await window.__syncastAgent.run("syncast.project.inspect", {
  limit: 20
});
```

在决定下一步之前先调用它。返回内容足够外部 Agent 选择频道、检查资源、发现运行中的任务、了解积分概况并读取文档。

检查后先判断缺口：

- 缺项目规范：优先委托内部 Agent 或创建文档。
- 缺参考素材：优先检查资源文件夹或询问用户是否上传。
- 缺视觉结构：可查看画布并规划分镜/参考关系。
- 缺剪辑结构：可查看时间轴并判断是否需要先生成素材。
- 有运行中任务：先等待或读取通知，不要重复发起。

## 使用内部 Agent

外部 Agent 无法直接访问 Syncast 内部 Skill。剧本设计、视频创作、提示词生成、模型参数理解、工具调用建议等，应该高频询问或委托内部 Agent。

项目里的每个自定义 Agent 都是独立声明，可直接运行，也可被其它 Agent 绑定为命名选项。调用前先选择顶层执行器：

- `executor: { kind: "model", model: "gemini-3.5-flash" }`：直接模型可发现项目全部 Agent，也可从头创建临时子 Agent。
- `executor: { kind: "agent", agentId: "..." }`：声明 Agent 只可按名称选择自己的绑定 Agent，也可从头创建临时子 Agent。

绑定不会复制 Agent，也不会形成运行时嵌套；命名或临时子执行都是叶节点。临时子 Agent 不继承声明 Agent 的指令和绑定技能。选择 `agentId` 前，用 Agents GraphQL 查询真实 ID 与 `childAgents`。

查询目录时同时读取 `allowLoadSkills` 和 `skills { skillId skillType preload }`。全部内置 Skill 始终可按需加载；`allowLoadSkills=false` 只排除未选中的项目 custom Skill，不影响已选 binding、依赖与 `alwaysApply` Skill。`preload=false` 表示 Skill 仍可用，只是不在启动时注入完整说明。新 Agent 建议显式写 `allowLoadSkills=false`；旧 Agent 缺失时按 `true`，旧 binding 缺少 `preload` 时也按 `true`。read-modify-write 必须原样保留这些字段。

每次发送都会冻结本次 Agent、命名 Agent 目录和 Skill 快照。任务运行中修改配置只影响下一次任务；子 Agent 权限也始终受 root task 权限上限约束。

```ts
const started = await window.__syncastAgent.run("syncast.agent.delegate", {
  goal: [
    "请基于当前项目状态，帮我创建适合短视频项目的项目规范。",
    "请把规范拆成：整体风格、角色、场景、分镜计划、常用提示词。",
    "如果项目方向是电影感 3D 动画或三维渲染二维感，请在风格规范中明确参考 `$cinematic-3d-animation`，并按该 Skill 写整体视觉原则、固定提示词骨架、正向词、反向控制和验收标准。",
    "请尽量写入项目文档，方便后续通过 @ 文档或章节引用。"
  ].join("\n"),
  executor: { kind: "model", model: "gemini-3.5-flash" },
  wait: false
});
```

不要把所有工作都塞进一个内部 Agent 对话。按项目规划、角色、场景、镜头、生成实验等主题拆开，避免上下文过长。

## 委托内部 Agent

```ts
const started = await window.__syncastAgent.run("syncast.agent.delegate", {
  goal: "请基于当前项目资源生成一个视频项目方案，并写入项目文档。",
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

内部 Agent 的最终可见文本在 `done.data.result.text`，短摘要在 `done.data.result.textPreview`。通过 CLI 调用时，`syncast project-agent wait --return-result` 返回同样结构；人工查看可加 `--format human`。

读取生成的文档：

```ts
await window.__syncastAgent.run("syncast.docs.readForAgent", {
  changedSince: started.data.ref.startedAt
});
```

`syncast.docs.readForAgent` 返回的是 canonical `docRead` 结构；后续读取章节正文时使用 `outline` 中的真实 `sectionId`，继续分页使用 `nextCursor`，重复上下文去重使用 `content.contextKey` / `loadedContextKeys`。

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
