# Timelines GraphQL

For Agent-facing shot planning, prefer wrapped actions `syncast.timeline.generationSlots.*`. Use raw GraphQL when you need precise timeline/clip/A-Roll/B-Roll operations that wrappers do not cover.

## Fields

Queries:

- `timelines`
- `timeline(id: String!)`
- `clipsByTimeline(timelineId: String!)`
- `clip(timelineId: String!, clipId: String!)`
- `arollList(timelineId: String!)`
- `connections(timelineId: String!)`
- `generationSlotUsagesByAsset(assetId: String!)`

Mutations:

- `createTimeline(input: TimelineCreateInput!)`
- `updateTimeline(id: String!, input: TimelineUpdateInput!)`
- `deleteTimeline(id: String!)`
- `createClip(timelineId: String!, input: ClipCreateInput!)`
- `createGenerationSlot(timelineId: String!, input: GenerationSlotCreateInput!)`
- `updateGenerationSlotInput(timelineId: String!, clipId: String!, input: GenerationSlotInputPatch!)`
- `setGenerationSlotOutput(timelineId: String!, clipId: String!, input: GenerationSlotOutputInput!)`
- `deleteClip(timelineId: String!, clipId: String!)`
- `addToAroll(timelineId: String!, clipId: String!, position?: Int)`
- `createConnection(timelineId: String!, input: ConnectionCreateInput!)`
- `deleteConnection(timelineId: String!, connectionId: String!)`

## Queries

List timelines:

```graphql
query {
  timelines {
    timelines {
      id
      name
      width
      height
      frameRate
      updatedAt
    }
  }
}
```

Read clips with generation slot data:

```graphql
query TimelineClips($timelineId: String!) {
  clipsByTimeline(timelineId: $timelineId) {
    id
    clipType
    frameDuration
    generationSlotData {
      slotLabel
      title
      status
      input
      outputAssetId
      pendingGenerationMessageId
    }
  }
}
```

Find where an asset is used as slot output:

```graphql
query SlotUsages($assetId: String!) {
  generationSlotUsagesByAsset(assetId: $assetId) {
    timelineId
    slotId
    clip { id generationSlotData { title outputAssetId } }
  }
}
```

## Mutations

Create a timeline:

```graphql
mutation CreateTimeline($input: TimelineCreateInput!) {
  createTimeline(input: $input) {
    id
    name
  }
}
```

Create a draft generation slot and add it to A-Roll:

```graphql
mutation CreateSlot($timelineId: String!, $input: GenerationSlotCreateInput!) {
  createGenerationSlot(timelineId: $timelineId, input: $input) {
    id
    clipType
    generationSlotData { slotLabel title status input }
  }
}
```

Variables:

```json
{
  "timelineId": "timeline-id",
  "input": {
    "slotLabel": "S1",
    "title": "男主角",
    "frameDuration": 96,
    "status": "draft",
    "input": {
      "modelType": "nano-banana-2",
      "prompt": "生成男主角角色设定图",
      "targetAssetName": "男主角A"
    },
    "addToAroll": true
  }
}
```

Update a slot input:

```graphql
mutation UpdateSlot($timelineId: String!, $clipId: String!, $input: GenerationSlotInputPatch!) {
  updateGenerationSlotInput(timelineId: $timelineId, clipId: $clipId, input: $input) {
    id
    generationSlotData { title status input }
  }
}
```

Set an adopted output asset:

```graphql
mutation SetSlotOutput($timelineId: String!, $clipId: String!, $input: GenerationSlotOutputInput!) {
  setGenerationSlotOutput(timelineId: $timelineId, clipId: $clipId, input: $input) {
    id
    generationSlotData { outputAssetId outputGenerationMessageId }
  }
}
```

Use External Actions to create and update draft slots. Let the user trigger a
slot in Syncast, or—when the user has authorized the external Agent to execute
it—call the public `syncast.timeline.generationSlots.submit` Action. Do not
generate outside the project and manually adopt the result as a substitute for
the Slot submission chain.
