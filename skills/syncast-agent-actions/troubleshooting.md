# Syncast Agent Action 异常处理

## `agent_not_initialized`

外部 Agent 还没有初始化身份。所有外部自动化操作前都必须先调用：

```ts
await window.__syncastAgent.initialize({
  name: "Project Operator Agent",
  description: "负责检查项目、委托内部 Agent，并验收生成结果。",
  emojiAvatar: "🤖",
});
```

## `not_in_project`

bridge 尚未绑定到已初始化的 Syncast 项目页。

请用户打开项目页面后重试：

```ts
await window.__syncastAgent.run("syncast.project.current");
```

## `doc_loading`

项目页面存在，但 Loro 仍在加载。等待片刻后重试。

## `sync_unsafe`

Loro Streams 初始同步状态不安全。不要强制执行 mutation，请用户先在应用中处理同步状态。

## `permission_denied`

当前用户没有相应的项目内容权限。`agent.delegate`、GraphQL mutation 或项目 Imagine submit 等新建/写入操作应停止，不得绕过权限改用直接生成 CLI 并假装结果已进入项目。

控制已经存在的项目 Agent 任务是协作能力：具备项目操作权限的用户可使用 `syncast.agent.followup`、`syncast.agent.thread.*` 和 `syncast.task.cancel`，不要求是原任务发起人。如果这些操作返回 403，通常表示当前账号已不再具备项目操作权限，或页面/后端仍是未更新的旧版本。

内部命名/临时子 Agent 不能通过自己的声明提升 root task 权限。遇到审批或拒绝时按 root 权限处理，不要通过更换 Agent 尝试绕过。

## `validation_failed`

Action input 与当前页面暴露的 schema 不匹配。不要猜字段；先读取完整 JSON Schema：

```bash
syncast project-agent capabilities --action <actionName> --disclosure full
```

GraphQL 字段用 camelCase；Library 本地资源包字段则先运行 `syncast library guide <type>` 查看规范格式。

## `action_not_exposed`

调用了真正的页面内部实现、兼容别名或重复 transport action。不要换名字猜测；重新调用 `window.__syncastAgent.capabilities(...)` 或 `syncast project-agent capabilities`，只使用返回的外部高层 action。`syncast.imagine.submit` 和 `syncast.imagine.submitToChannel` 是公开能力；若它们未出现，通常是页面或 CLI 版本过旧，应更新并重新连接项目。项目工作不得因此回退到直接生成 CLI；只有用户明确改为项目外独立资产任务时才能切换 standalone 路线。

## Agent Skill / binding 异常

- `load_skills` 报项目 custom Skill 不存在或不可用：先确认它已被 Agent 选中，或 Agent 的 `allowLoadSkills=true`。`false` 仅排除未选中的项目 custom Skill；全部内置 Skill、已选 custom Skill、依赖和 `alwaysApply` Skill仍可用。补 binding 或显式开启扩展发现后重新发送任务；运行中的任务不会看到刚修改的配置。
- 新 Agent 意外看到项目全部 custom Skill：检查是否遗漏 `allowLoadSkills` 或写成了 `true`。缺失值为了兼容旧 Agent 按 `true` 解释；新 Agent 应显式写 `false`。
- Agent 更新后意外全部预载：检查 read-modify-write 是否遗漏了 `skills.preload`。旧/缺失值按 `true` 兼容。
- Library 导入报缺失 custom Skill 或 child Agent：项目导入会在写入前做依赖预检。先把缺失资源加入同一 bundle 或安装到项目，再重试；失败不会留下半套 Agent binding。
- `custom_skill_registry_mismatch`：前端会让旧 hash 失效，并用本次完整 Skill 快照自动重试。若错误持续出现，等待项目同步完成并重试，不要手工伪造 hash。

## `timeout`

任务没有在等待窗口内完成。这不代表 Syncast 任务失败。

保存 `ref`，稍后调用：

```ts
await window.__syncastAgent.notifications.list({
  filter: { ref }
});
```

或：

```ts
await window.__syncastAgent.wait(ref, { returnResult: true });
```

## 内部 Agent 失败

读取最终 assistant 消息和工具结果：

```ts
await window.__syncastAgent.wait(ref, { returnResult: true });
```

然后检查项目状态：

```ts
await window.__syncastAgent.run("syncast.project.inspect", {
  limit: 10
});
```

CLI 的 `wait --return-result` 默认只输出最终文本、任务终态和产物 ID。只有排查消息 parts、thinking 或工具轨迹时才加 `--full-result`，避免把大段内部过程带入上下文。
