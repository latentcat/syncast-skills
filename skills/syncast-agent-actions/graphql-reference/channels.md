# Channels GraphQL

用于精确读写 AI / Imagine 频道和消息。业务对话优先使用 `syncast.agent.delegate` 或 `syncast.agent.chat.submit`；本模块适合读取、迁移或轻量结构化创建。

## Fields

Queries:

- `channels(type?: String)`
- `channel(id: String!)`
- `messages(channelId: String!, limit?: Int)`
- `message(channelId: String!, messageId: String!)`

Mutations:

- `createChannel(input: ChannelCreateInput!)`
- `updateChannel(id: String!, input: ChannelUpdateInput!)`
- `deleteChannel(id: String!)`
- `createChatMessage(input: ChatMessageCreateInput!)`
- `createImagineMessage(input: ImagineMessageCreateInput!)`
- `updateImagineMessage(channelId: String!, messageId: String!, input: ImagineMessageUpdateInput!)`
- `deleteMessage(channelId: String!, messageId: String!)`

## Queries

List channels:

```graphql
query {
  channels {
    channels {
      id
      name
      type
      updatedAt
    }
  }
}
```

Read recent messages:

```graphql
query Messages($channelId: String!) {
  messages(channelId: $channelId, limit: 20) {
    id
    messageType
    taskId
    taskStatus
    assetId
    createdAt
    message
    input
  }
}
```

## Mutations

Create a stable planning channel:

```graphql
mutation CreateChannel($input: ChannelCreateInput!) {
  createChannel(input: $input) {
    id
    name
    type
  }
}
```

Variables:

```json
{ "input": { "name": "项目方案", "type": "chat" } }
```

Update an Imagine message result metadata:

```graphql
mutation UpdateImagineMessage(
  $channelId: String!
  $messageId: String!
  $input: ImagineMessageUpdateInput!
) {
  updateImagineMessage(channelId: $channelId, messageId: $messageId, input: $input) {
    id
    assetId
    assetIdMap
  }
}
```

Avoid deleting user-visible channels/messages unless explicitly requested.
