# Imagine Optimize Presets GraphQL

用于管理项目内 Imagine 提示词优化预设（Sparkles 菜单中的自定义改写风格）。只在用户要求创建/修改/删除优化预设，或根据 before/after 例子固化改写风格时使用。

与 Prompt Template 区分：Prompt Template 生成用户要发送的内容；Optimize Preset 只定义「如何把用户输入改写成更适合模型的提示词」。

## Fields

Queries:

- `imagineOptimizePresets(limit?: Int)`
- `imagineOptimizePreset(id: String!)`

Mutations:

- `createImagineOptimizePreset(input: ImagineOptimizePresetCreateInput!)`
- `updateImagineOptimizePreset(id: String!, input: ImagineOptimizePresetUpdateInput!)`
- `deleteImagineOptimizePreset(id: String!)`

## Queries

```graphql
query {
  imagineOptimizePresets(limit: 50) {
    totalCount
    presets {
      id
      name
      description
      scopes
      baseStyle
      mergeMode
      intent
      instructions
      formatRules
      expansion
      examples { before after }
      updatedAt
    }
  }
}
```

## Mutations

Create a preset:

```graphql
mutation CreateOptimizePreset($input: ImagineOptimizePresetCreateInput!) {
  createImagineOptimizePreset(input: $input) {
    id
    name
    scopes
    baseStyle
    mergeMode
  }
}
```

Variables:

```json
{
  "input": {
    "name": "Cinematic faithful polish",
    "description": "Faithful Seedance rewrite with cinematic camera, lighting, and blocking.",
    "scopes": ["video_seedance"],
    "baseStyle": "controlled",
    "mergeMode": "extend",
    "intent": "Preserve user facts while adding cinematic production detail.",
    "instructions": "Add concise camera framing, movement, lighting, environment, blocking, and material detail derived from the examples. Do not introduce unrelated story events or change character identity.",
    "expansion": "expand",
    "examples": [
      {
        "before": "girl on rainy street",
        "after": "Medium shot of a girl standing on a rain-slick city street at night, soft neon reflections on the pavement, subtle handheld camera movement, natural facial expression, realistic wet fabric detail."
      }
    ]
  }
}
```

## Field notes

- `scopes`: prefer fine-grained values (`image_nano_banana`, `image_non_nano`, `video_seedance`, `video_non_seedance`, `audio_music`); empty array means all models.
- `baseStyle`: `controlled` or `creative`; only for multi-route models such as `video_seedance`.
- `mergeMode`: `extend` (default, stack on system default) or `replace` (custom style only; hard safety/token rules still apply).
- `expansion`: `faithful`, `expand`, or `rich`; omit when uncertain.
- Put long `instructions`, `formatRules`, and `examples` in mutation `variables`, not inline in the query string.
