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

当前用户没有编辑此项目的权限。读取类 action 仍可能可用；`agent.delegate`、`imagine.submit` 或 GraphQL mutation 等写入操作应停止。

内部命名/临时子 Agent 不能通过自己的声明提升 root task 权限。遇到审批或拒绝时按 root 权限处理，不要通过更换 Agent 尝试绕过。

## `validation_failed`

Action input 与当前页面暴露的 schema 不匹配。不要猜字段；先读取完整 JSON Schema：

```bash
syncast project-agent capabilities --action <actionName> --disclosure full
```

GraphQL 字段用 camelCase；Library 本地资源包字段则先运行 `syncast library guide <type>` 查看规范格式。

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
await window.__syncastAgent.run("syncast.notifications.list", {
  filter: { ref }
});
```

或：

```ts
await window.__syncastAgent.run("syncast.task.result", { ref });
```

## 内部 Agent 失败

读取最终 assistant 消息和工具结果：

```ts
await window.__syncastAgent.run("syncast.agent.chat.result", { ref });
```

然后检查项目状态：

```ts
await window.__syncastAgent.run("syncast.project.inspect", {
  limit: 20
});
```
