# Docs GraphQL

读取正文优先用封装 action `syncast.docs.readForAgent`，它已经处理 outline、分页和去重。需要结构化创建、重命名、移动、删除或 patch 文档时用本模块 mutation。用户提供完整正文或精确修改时直接使用本模块，不委托内部 Agent 改写或机械搬运。

## Fields

Queries:

- `docPages`
- `docPage(id: String!)`
- `docRead(input: DocReadInput!)`
- `docBlocks(docId: String!)`
- `docMarkdown(docId: String!)`

Mutations:

- `ensureDocPages(inputs: [EnsureDocPageInput!]!)`
- `updateDocPage(id: String!, input: DocPageUpdateInput!)`
- `deleteDocPage(id: String!)`
- `moveDocPage(id: String!, parentId?: String)`
- `moveDocPageBefore(id: String!, beforeId: String!)`
- `moveDocPageAfter(id: String!, afterId: String!)`
- `setDocBlocks(docId: String!, blocks: JSON!)`
- `patchDoc(docId: String!, patch: String!)`

完整内部 schema 仍保留旧 `createDocPage` 供 App/历史调用方创建空白页，但它不属于 Agent-facing schema，外部 `syncast.doc.graphql` 也会拒绝该字段。外部 Agent 新建文档一律使用 `ensureDocPages`；不要把旧入口当作备用路径。

## Queries

List the document tree:

```graphql
query {
  docPages {
    totalCount
    docPages {
      id
      title
      description
      parentId
      docKind
      specInjectionMode
      logicalKey
      containerOnly
      hasContent
      updatedAt
    }
  }
}
```

Read complete page metadata by the real id:

```graphql
query ReadDocPage($id: String!) {
  docPage(id: $id) {
    id
    title
    description
    parentId
    docKind
    specInjectionMode
    logicalKey
    containerOnly
    icon
    cover
    hasContent
    createdAt
    updatedAt
  }
  docMarkdown(docId: $id)
}
```

Read with canonical `docRead`:

```graphql
query DocOutline($input: DocReadInput!) {
  docRead(input: $input) {
    mode
    outline {
      sectionId
      title
      level
      path
      startBlockIndex
      endBlockIndex
      childCount
      preview
    }
  }
}
```

Variables:

```json
{ "input": { "mode": "OUTLINE", "docId": "doc-id" } }
```

## Mutations

一次创建完整文档树并写入初始正文：

```graphql
mutation EnsureDocs($inputs: [EnsureDocPageInput!]!) {
  ensureDocPages(inputs: $inputs) {
    success
    logicalKey
    docId
    created
    adopted
    contentWritten
    ready
    hasMeaningfulContent
    blockCount
    markdownCharCount
  }
}
```

Variables:

```json
{
  "inputs": [
    {
      "logicalKey": "short-drama:episodes",
      "title": "分集剧本",
      "containerOnly": true
    },
    {
      "logicalKey": "short-drama:episode:01",
      "title": "第一集",
      "parentLogicalKey": "short-drama:episodes",
      "description": "第一集完整剧本",
      "initialMarkdown": "# 第一集\n\n## 场景一\n\n清晨，街道上……"
    }
  ]
}
```

`logicalKey` 是创建前即可确定的项目内稳定业务身份，必须跨重试、任务和重放保持不变；不要使用随机 tool-call ID，也不要用可重名的标题代替。`parentLogicalKey` 可以引用同批次父项，因此一轮即可创建完整层级。内容型页面必须提供非空且非等效空的 `initialMarkdown`；只有纯目录页可设 `containerOnly: true`。

旧项目已有目标页面时，先查 `docPages`，确认唯一真实 ID 后通过 `existingDocId` 显式采用。默认禁止同父级同名；确需创建同名新页时，使用不同 `logicalKey` 并显式设置 `allowDuplicateTitle: true`。绝不能仅按标题静默猜测。

每项 `success=true` 且 `ready=true` 表示本次 mutation 已写入并检查正文后置条件，不需要为了机械确认再调用 `docRead`。只有语义审校、引用检查或用户明确要求时才回读。`ensureDocPages` 不覆盖已有有效正文；修改已有文档使用真实 `docId` 调 `patchDoc`。

Move a document under another document:

```graphql
mutation MoveDoc($id: String!, $parentId: String) {
  moveDocPage(id: $id, parentId: $parentId)
}
```

Move to root by passing `parentId: null`.

Reorder among siblings:

```graphql
mutation ReorderDoc($id: String!, $afterId: String!) {
  moveDocPageAfter(id: $id, afterId: $afterId)
}
```

Use `moveDocPageBefore` for before placement. Do not move a document into its own descendant; the underlying operation will fail.

Rename / metadata update:

```graphql
mutation RenameDoc($id: String!, $input: DocPageUpdateInput!) {
  updateDocPage(id: $id, input: $input) {
    id
    title
    description
    updatedAt
  }
}
```

Variables:

```json
{ "id": "doc-id", "input": { "title": "角色设定" } }
```

Patch content:

```graphql
mutation PatchDoc($docId: String!, $patch: String!) {
  patchDoc(docId: $docId, patch: $patch) {
    success
    error
    appliedOperationCount
  }
}
```

## 写入与验收规则

- 创意构思、剧本设计或结构未知时，可先用 `syncast.agent.delegate` 获取草案，并要求它只返回正文/结构、不执行项目写入。外部 Agent 再把确认后的内容写入目标文档。
- 用户已提供完整正文时，新建用 `ensureDocPages`；精确字段更新或确定文本替换用 `updateDocPage` / `patchDoc`。不要把原文交给内部 Agent 转述。删除仍按授权规则确认；用户明确要求的精确替换不重复确认。
- 新建前读取文档树只用于判断是否应显式采用已有真实 ID，不按标题自动复用。`ensureDocPages` 返回 `ready=true` 已完成结构与正文机械验收；只有本次还涉及图标/封面、语义质量或引用真实性时才按真实 `docId` 追加相应回读。
- `setDocBlocks` 是已有整页替换流程的兼容入口；常规修改优先 `patchDoc`，避免不必要地重建 block 身份。
- 普通用户文档使用产品与创作语言，默认不出现 CLI、Bridge、Action、GraphQL、standalone API、传输/provider 或接入实现。明确的集成/开发文档除外。
- 验收覆盖 `title`、`description`、正文、`parentId`、`docKind`、`specInjectionMode`、图标/封面等本次涉及的元数据；description 残留外部接入内容仍算失败。
- 正文或 description 中的 `@{doc:<docId>|...}` / `@{doc-section:<docId>:<sectionId>|...}` 必须使用真实 ID。文档 ID 用 `docPage` 校验，章节 ID 用 `docRead` outline 校验。
