# GraphQL Reference Index

本目录按模块拆分 monodoc GraphQL 参考，供外部 Agent 按需加载。不要把所有文件一次性读入上下文；先用 `syncast.doc.graphql.schema` 看模块索引，再加载相关模块文件。

所有调用都通过：

```ts
await window.__syncastAgent.run("syncast.doc.graphql", {
  query: "...",
  variables: { ... },
  idempotencyKey: "stable-key-for-create-mutations"
});
```

CLI Bridge 等价调用：

```bash
syncast project-agent run syncast.doc.graphql --input '{"query":"query { docPages { totalCount } }"}'
```

## Modules

| 模块 | 文件 | 典型用途 |
| --- | --- | --- |
| Assets | [assets.md](assets.md) | 资源查询、批量移动、文件夹整理、收藏/评分 |
| Docs | [docs.md](docs.md) | 文档列表、正文读取、创建/移动/改名/patch |
| Channels | [channels.md](channels.md) | AI 频道和消息的精确读写 |
| Timelines | [timelines.md](timelines.md) | 时间轴、clip、A-Roll/B-Roll、generation slot |
| Canvas | [canvas.md](canvas.md) | 画布页面和元素读写 |
| Agents | [agents.md](agents.md) | 自定义 Agent 定义 |
| Skills | [skills.md](skills.md) | 自定义 Skill 定义 |
| Prompt Templates | [prompt-templates.md](prompt-templates.md) | 自定义提示词模板 |

## Rules

- Query 是 read；mutation 是 edit，会先等待 Streams 初始同步，再 `saveAndSync`。
- 字段名用 GraphQL camelCase：`parentId`、`updatedAt`、`assetId`，不要用内部 snake_case。
- 创建类 mutation 尽量传 `idempotencyKey`，避免重试时重复创建。
- 业务创作优先 `syncast.agent.delegate`；只有明确的数据结构变更才用 raw GraphQL。
- 完整 SDL 在 `assets/monodoc-graphql-schema/monodoc.graphql`。
