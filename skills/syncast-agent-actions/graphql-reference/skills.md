# Skills GraphQL

用于管理项目内自定义 Skill。只有在用户要求创建/维护项目技能时使用；普通任务执行不需要改 Skill。

## Fields

Queries:

- `customSkills(limit?: Int)`
- `customSkill(id: String!)`

Mutations:

- `createCustomSkill(input: CustomSkillCreateInput!)`
- `updateCustomSkill(id: String!, input: CustomSkillUpdateInput!)`
- `deleteCustomSkill(id: String!)`

## Queries

```graphql
query {
  customSkills {
    totalCount
    skills {
      id
      name
      description
      alwaysApply
      depends { skillId skillType }
      updatedAt
    }
  }
}
```

## Mutations

Create a Skill:

```graphql
mutation CreateSkill($input: CustomSkillCreateInput!) {
  createCustomSkill(input: $input) {
    id
    name
    alwaysApply
  }
}
```

Variables:

```json
{
  "input": {
    "name": "角色设定规范",
    "description": "生成角色设定时必须检查的项目规则。",
    "instructions": "先读取项目规范文档，保持角色名称、年龄、服装和视觉风格一致。",
    "iconName": "FileText",
    "alwaysApply": false,
    "depends": []
  }
}
```

Skill 不支持 Agent 专属字段 `model` 或 `colorName`。
