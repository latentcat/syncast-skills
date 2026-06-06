# Canvas GraphQL

用于精确读写画布页面和元素。复杂视觉排版通常更适合内部 Agent 或人类 UI 操作；本模块适合把已有资源、文档引用或结构化元素放到画布上。

## Fields

Queries:

- `pages`
- `page(id: String!)`
- `pageWithElements(pageId: String!)`
- `element(id: String!)`
- `elementsByPage(pageId: String!)`
- `elementWithAsset(id: String!)`

Mutations:

- `createPage(input: PageCreateInput!)`
- `updatePage(id: String!, input: PageUpdateInput!)`
- `deletePage(id: String!)`
- `createElement(input: ElementCreateInput!)`
- `updateElement(id: String!, input: ElementUpdateInput!)`
- `deleteElement(id: String!)`
- `deleteElements(ids: [String!]!)`
- `moveElement(id: String!, x: Float!, y: Float!)`

## Queries

List pages:

```graphql
query {
  pages {
    pages { id name createdAt updatedAt }
  }
}
```

Read a page with elements:

```graphql
query PageWithElements($pageId: String!) {
  pageWithElements(pageId: $pageId) {
    page { id name }
    elements {
      id
      type
      x
      y
      width
      height
      rotation
      zIndex
      assetId
      docId
      sectionId
      text
    }
  }
}
```

## Mutations

Create a canvas page:

```graphql
mutation CreatePage($input: PageCreateInput!) {
  createPage(input: $input) { id name }
}
```

Create an asset element:

```graphql
mutation CreateElement($input: ElementCreateInput!) {
  createElement(input: $input) {
    id
    type
    assetId
    x
    y
    width
    height
  }
}
```

Variables:

```json
{
  "input": {
    "type": "image",
    "assetId": "asset-id",
    "x": 120,
    "y": 80,
    "width": 480,
    "height": 270
  }
}
```

Create a doc reference element:

```json
{
  "input": {
    "type": "doc_ref",
    "docId": "doc-id",
    "sectionId": "optional-section-id",
    "x": 120,
    "y": 380,
    "width": 360,
    "height": 160
  }
}
```

Use `updateElement` for size, style, text, `assetId`, `docId`, `sectionId`, and position fields. Use `moveElement` only for a simple x/y move.
