# Docs GraphQL

读取正文优先用封装 action `syncast.docs.readForAgent`，它已经处理 outline、分页和去重。需要结构化创建、重命名、移动、删除或 patch 文档时用本模块 mutation。

## Fields

Queries:

- `docPages`
- `docPage(id: String!)`
- `docRead(input: DocReadInput!)`
- `docBlocks(docId: String!)`
- `docMarkdown(docId: String!)`

Mutations:

- `createDocPage(title: String!, parentId?: String, docKind?: String, specInjectionMode?: String, idempotencyKey?: String)`
- `updateDocPage(id: String!, input: DocPageUpdateInput!)`
- `deleteDocPage(id: String!)`
- `moveDocPage(id: String!, parentId?: String)`
- `moveDocPageBefore(id: String!, beforeId: String!)`
- `moveDocPageAfter(id: String!, afterId: String!)`
- `setDocBlocks(docId: String!, blocks: JSON!)`
- `patchDoc(docId: String!, patch: String!)`

## Queries

List the document tree:

```graphql
query {
  docPages {
    totalCount
    docPages {
      id
      title
      parentId
      docKind
      hasContent
      updatedAt
    }
  }
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

Create a child doc:

```graphql
mutation CreateDoc($title: String!, $parentId: String, $key: String) {
  createDocPage(title: $title, parentId: $parentId, idempotencyKey: $key) {
    id
    title
    parentId
  }
}
```

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

For business writing, delegate to `syncast.agent.delegate` instead of crafting large patches manually.
