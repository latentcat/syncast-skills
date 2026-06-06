# Prompt Templates GraphQL

用于管理项目内自定义提示词模板。模板可被富文本输入和 Imagine/Agent 输入复用。只在用户要求维护模板库时使用。

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
query {
  customPromptTemplates {
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

## Mutations

Create a template:

```graphql
mutation CreatePromptTemplate($input: CustomPromptTemplateCreateInput!) {
  createCustomPromptTemplate(input: $input) {
    id
    name
    targetTypes
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
