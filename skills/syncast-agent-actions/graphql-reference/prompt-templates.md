# Prompt Templates GraphQL

用于管理项目内自定义提示词模板。模板可被富文本输入和 Imagine/Agent 输入复用。只在用户要求维护模板库时使用。项目内部模板使用创作与产品能力语义，不写 CLI、Bridge、standalone API、外部 Action/GraphQL 名、传输或接入方式。

## Fields

Queries:

- `customPromptTemplates(limit?: Int)`
- `customPromptTemplate(id: String!)`

Mutations:

- `createCustomPromptTemplate(input: CustomPromptTemplateCreateInput!)`
- `updateCustomPromptTemplate(id: String!, input: CustomPromptTemplateUpdateInput!)`
- `deleteCustomPromptTemplate(id: String!)`

## Queries

```graphql
query ListPromptTemplates($limit: Int = 50) {
  customPromptTemplates(limit: $limit) {
    totalCount
    promptTemplates {
      id
      name
      description
      targetTypes
      inputTemplate
      promptTemplate
      variables { key placeholder defaultValue }
      updatedAt
    }
  }
}
```

创建前先按 trim、大小写归一后的完整名称查重；如果 `totalCount` 大于本次返回的 `promptTemplates.length`，以 `totalCount` 作为 `$limit` 重新查询后再查重。唯一同名项使用 `updateCustomPromptTemplate`，不要重复创建。写入后按 mutation 返回的真实 ID 回读完整模板：

```graphql
query ReadPromptTemplate($id: String!) {
  customPromptTemplate(id: $id) {
    id
    name
    description
    targetTypes
    inputTemplate
    promptTemplate
    variables { key placeholder defaultValue }
    createdAt
    updatedAt
  }
}
```

## Mutations

Create a template:

```graphql
mutation CreatePromptTemplate($input: CustomPromptTemplateCreateInput!) {
  createCustomPromptTemplate(input: $input) {
    id
    name
    description
    targetTypes
    inputTemplate
    promptTemplate
    variables { key placeholder defaultValue }
  }
}
```

Variables:

```json
{
  "input": {
    "name": "角色参考图提示词",
    "description": "统一角色设定图风格",
    "targetTypes": ["image"],
    "inputTemplate": "为 {{ character_name }} 生成角色参考图",
    "promptTemplate": "Character reference sheet for {{ character_name }}, consistent costume, cinematic lighting.",
    "variables": [
      {
        "key": "character_name",
        "placeholder": "角色名",
        "defaultValue": "主角"
      }
    ]
  }
}
```

`targetTypes` 可用 `"image"`, `"video"`, `"audio"`, `"agent"`。

验收必须同时检查 `name`、`description`、`targetTypes`、`inputTemplate`、`promptTemplate` 和所有变量的 `key/placeholder/defaultValue`。用户已提供完整模板或精确字段时直接 GraphQL 写入，不委托内部 Agent 改写。所有项目文档引用都使用真实 `docId` / `sectionId`；正文正确但 description 或变量默认值仍含外部接入实现，也不算完成。
