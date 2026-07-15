# Skills GraphQL

用于管理项目内自定义 Skill。只有在用户要求创建/维护项目技能时使用；普通任务执行不需要改 Skill。项目内部 Skill 只描述业务规则、项目内能力语义、模型与创作参数，不写 CLI、Bridge、standalone API、外部 Action/GraphQL 名或接入方式；用户明确要求集成 Skill 时除外。

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
query ListSkills($limit: Int = 50) {
  customSkills(limit: $limit) {
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

创建前必须先执行列表查询并按 trim、大小写归一后的完整名称查重。如果 `totalCount` 大于本次返回的 `skills.length`，以 `totalCount` 作为 `$limit` 重新查询后再查重，不能只检查前 50 项。唯一同名项应走 `updateCustomSkill`；多个近似/同名候选时先确认，不能重复创建。

Read one complete Skill after create/update:

```graphql
query ReadSkill($id: String!) {
  customSkill(id: $id) {
    id
    name
    description
    instructions
    iconName
    alwaysApply
    depends { skillId skillType }
    createdAt
    updatedAt
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
    description
    instructions
    iconName
    alwaysApply
    depends { skillId skillType }
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

Update an existing Skill:

```graphql
mutation UpdateSkill($id: String!, $input: CustomSkillUpdateInput!) {
  updateCustomSkill(id: $id, input: $input) {
    id
    name
    description
    instructions
    iconName
    alwaysApply
    depends { skillId skillType }
    updatedAt
  }
}
```

## 标准验收流程

1. 创建前查询同名 Skill，避免重复；更新前先回读现有完整字段，避免遗漏元数据。
2. 写入前明确 `alwaysApply`、`depends`，并决定是否需要在 Agents GraphQL 中绑定到具体 Agent。`alwaysApply=true` 时通常不需要为“可用性”重复绑定；需要控制特定 Agent 的预载时再绑定并明确 `preload`。
3. mutation 后使用返回的真实 `id` 执行 `ReadSkill`，不能只相信 mutation 的精简返回。
4. 同时检查 `name`、`description`、`instructions`、`iconName`、`alwaysApply`、`depends` 和相关 Agent binding；正文正确但 description 仍含 CLI 也属于失败。
5. 对 `description/instructions` 中所有 `@{doc:<docId>|...}` 和 `@{doc-section:<docId>:<sectionId>|...}`，逐个查询真实文档；章节还要通过 `docRead` outline 校验 `sectionId`。
6. 项目内部内容应使用能力语义。例如写“使用项目内 Imagine 能力，模型为 `oai-gpt-image-2`，比例 16:9，分辨率 2K”，不要写 `syncast.imagine.submit`、`syncast project-agent` 或禁止 standalone CLI 的接入规则。
7. 如果需要绑定 Agent，更新 Agent 前必须读取并保留 `allowLoadSkills`、全部 `skills { skillId skillType preload }`、`childAgents` 和其它声明字段；更新后再回读确认。
