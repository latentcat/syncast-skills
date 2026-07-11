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

`depends` 只是 Skill 依赖引用，每项只有 `skillId` 和 `skillType`，不要在这里写 `preload`。`preload` 只属于 Agents GraphQL 中某个 Agent 的 `skills` binding；`alwaysApply=true` 才表示这个项目 Skill 对所有执行器始终可用并在启动时加载。
