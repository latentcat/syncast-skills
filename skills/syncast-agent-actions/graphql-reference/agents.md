# Agents GraphQL

用于管理项目内自定义 Agent 定义，以及一个 Agent 对其它 Agent 的命名绑定。外部发起业务任务统一使用 `syncast.agent.delegate`；选择执行器前先从这里读取真实 Agent ID。

## 运行语义

- 每个 Agent 都是一份独立声明，可以单独作为顶层执行器使用。
- `childAgents` 只是对其它既有 Agent ID 的命名调度引用，不会复制声明，也不会产生另一份“子 Agent 模式”。
- `executor: { kind: "model", model }` 的直接模型可发现项目全部 Agent。
- `executor: { kind: "agent", agentId }` 的声明 Agent 只可按名称发现自己的 `childAgents`。
- 两种顶层执行器都能从头创建临时子 Agent。临时子 Agent 不继承父 Agent 的指令或绑定技能；所有子执行都是叶节点。

### Skill binding 与 preload

`Agent.skills` 不是普通的 `SkillRef`，而是 Agent 专用 binding：

- custom `skillId` / `skillType` binding 决定关闭扩展发现时仍明确可用的项目 Skill；全部内置 Skill 始终可按需加载。
- `preload: true` 表示启动时立即注入完整 Skill 说明。
- `preload: false` 表示仍可用，但只在需要时通过 `load_skills` 加载完整说明。
- 旧 binding 没有 `preload` 时按 `true` 解释，保证旧项目升级后行为不变；新 binding 应显式写 `preload: false` 或 `true`。
- Custom Skill 的 `depends` 不使用 `preload`；依赖会跟随被绑定或预载的 Skill 自动进入闭包。
- `alwaysApply=true` 的项目 Skill 始终可用并在启动时加载。

### allowLoadSkills 兼容语义

- `allowLoadSkills: false` 只排除**未选中的项目 custom Skill**；已选 custom Skill、依赖、`alwaysApply` Skill 和全部内置 Skill仍可按需加载。
- `allowLoadSkills: true` 把项目里所有 custom Skill 都加入可发现、可动态加载范围。Skill 很多时会扩大选择空间，只有确实需要跨目录自由发现时才开启。
- 新 Agent 建议显式写 `false`；旧 Agent 缺少该字段时按 `true` 解释，以避免升级后行为突变。
- 该字段不控制启动预载。预载只看 binding 的 `preload` 与 custom Skill 的 `alwaysApply`。

任何读取后再整体更新 `skills` 的操作，都必须在 query 中请求并在 mutation 中原样保留 `preload`。如果省略，原来的 `false` 会变成兼容旧数据的“缺失”，下一次运行会按 `true` 预载。

### 任务快照与权限

- 任务提交时会冻结本次使用的 Agent、命名 Agent 目录和 Skill 快照。运行中修改 Agent/Skill 只影响下一次任务。
- 根任务权限是硬上限；声明 Agent 或命名 Agent 只能继承或进一步收紧，不能提升权限。
- 当前项目 UI 尚未保存 Agent 独立权限配置，所以普通项目 Agent 默认继承根任务权限。

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
      modelProfile
      iconName
      colorName
      allowLoadSkills
      skills { skillId skillType preload }
      childAgents {
        childAgentId
        alias
        displayName
        whenToUse
        handoffContract
        projectSpecStrategy
      }
      updatedAt
    }
  }
}
```

## Mutations

先独立创建 Agent：

```graphql
mutation CreateAgent($input: AgentCreateInput!) {
  createAgent(input: $input) {
    id
    name
    skills { skillId skillType preload }
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
    "modelProfile": {
      "schema_version": 2,
      "context_window_tokens": 200000,
      "effort": "medium",
      "thinking": true
    },
    "iconName": "FileText",
    "colorName": "blue",
    "allowLoadSkills": false,
    "skills": [
      {
        "skillId": "builtin-docs",
        "skillType": "builtin",
        "preload": false
      }
    ]
  }
}
```

更新 Agent 时，`skills` 是替换列表。安全的 read-modify-write 示例：

```graphql
query ReadAgentForUpdate($id: String!) {
  agent(id: $id) {
    id
    name
    skills { skillId skillType preload }
    childAgents {
      childAgentId
      alias
      displayName
      whenToUse
      handoffContract
      projectSpecStrategy
    }
  }
}
```

只修改名称时不要顺手重建 `skills`；确实需要替换 binding 时，把读取到的三个字段完整传回：

```json
{
  "id": "story-planner-agent-id",
  "input": {
    "skills": [
      {
        "skillId": "builtin-docs",
        "skillType": "builtin",
        "preload": false
      }
    ]
  }
}
```

创建另一个独立 Agent 后，再把其 ID 绑定给编排 Agent：

```graphql
mutation BindAgent($id: String!, $input: AgentUpdateInput!) {
  updateAgent(id: $id, input: $input) {
    id
    name
    childAgents {
      childAgentId
      alias
      displayName
      whenToUse
      handoffContract
      projectSpecStrategy
    }
  }
}
```

Variables:

```json
{
  "id": "orchestrator-agent-id",
  "input": {
    "childAgents": [
      {
        "childAgentId": "story-planner-agent-id",
        "alias": "planner",
        "displayName": "Story Planner",
        "whenToUse": "需要项目方案、分镜结构或叙事连续性判断时使用。",
        "handoffContract": "输入项目目标、规范、已有草稿和限制；返回结构化方案、风险与下一步。",
        "projectSpecStrategy": "inherit"
      }
    ]
  }
}
```

绑定 A→B 与 B→A 在数据上是允许的，因为每次被调度的 Agent 都是叶节点，不会在同一次执行里继续沿绑定关系嵌套。通常仍应根据职责保持路由清晰。

执行示例：

```json
{
  "goal": "整理项目方案",
  "executor": { "kind": "agent", "agentId": "orchestrator-agent-id" },
  "wait": false
}
```

如果希望直接模型自行从所有项目 Agent 中选择：

```json
{
  "goal": "整理项目方案，需要时调用最合适的项目 Agent",
  "executor": { "kind": "model", "model": "gemini-3.5-flash" },
  "wait": false
}
```

Delete only when the user explicitly asks; deleting an Agent may affect saved project setup.
