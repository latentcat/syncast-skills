# Agents GraphQL

用于管理项目内自定义 Agent 定义。发起业务任务不需要改这里；直接用 `syncast.agent.delegate` 或 `syncast.agent.chat.submit`。

## Fields

Queries:

- `agents(limit?: Int)`
- `agent(id: String!)`

Mutations:

- `createAgent(input: AgentCreateInput!)`
- `updateAgent(id: String!, input: AgentUpdateInput!)`
- `deleteAgent(id: String!)`

## Queries

```graphql
query {
  agents {
    totalCount
    agents {
      id
      name
      description
      model
      iconName
      colorName
      allowLoadSkills
      skills { skillId skillType }
      updatedAt
    }
  }
}
```

## Mutations

Create an Agent:

```graphql
mutation CreateAgent($input: AgentCreateInput!) {
  createAgent(input: $input) {
    id
    name
    skills { skillId skillType }
  }
}
```

Variables:

```json
{
  "input": {
    "name": "Story Planner",
    "description": "负责项目方案和分镜规划",
    "instructions": "先阅读项目规范，再输出结构化方案。",
    "model": "sonnet-4-6",
    "iconName": "FileText",
    "colorName": "blue",
    "allowLoadSkills": true,
    "skills": [{ "skillId": "builtin-docs", "skillType": "builtin" }]
  }
}
```

Delete only when the user explicitly asks; deleting an Agent may affect saved project setup.
