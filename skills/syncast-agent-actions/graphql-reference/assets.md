# Assets GraphQL

用于精确读取和整理项目资源、文件夹。名称找 ID 优先用 `syncast.assets.list { query }`，再用 `syncast.assets.get` 确认唯一资产；需要批量整理资源树时再用本模块 mutation。

Agent-facing asset queries 会过滤隐藏的 3D/model 资产（`glb` / `stl` / `fbx`）。过滤覆盖 top-level `asset/assets/assetsByIds/folders/assetBrowse`，也覆盖时间轴、画布、频道消息等 nested `asset` / `assetId` / asset ID map 输出；返回 null 或缺失时不要绕过 action layer 反查 raw Loro。

## Fields

Queries:

- `asset(id: String!)`
- `assets(folderId?: String, limit?: Int)`
- `assetsByIds(ids: [String!]!)`
- `folders(parentId?: String)`
- `assetBrowse(input: AssetBrowseInput!)`

Mutations:

- `updateAsset(id: String!, input: AssetUpdateInput!)`
- `deleteAsset(id: String!)`
- `deleteAssets(ids: [String!]!)`
- `createFolder(input: FolderCreateInput!)`
- `ensureFolderPath(input: EnsureFolderPathInput!)`
- `updateFolder(input: FolderUpdateInput!)`
- `moveFolder(input: FolderMoveInput!)`
- `deleteFolder(input: FolderDeleteInput!)`
- `moveAssetToFolder(assetId: String!, folderId?: String)`
- `moveAssetsToFolder(input: MoveAssetsToFolderInput!)`
- `updateAssets(input: [AssetBatchUpdateInput!]!)`
- `organizeAssets(input: OrganizeAssetsInput!)`
- `setAssetFavorite(id: String!, isFavorite: Boolean!)`

## Queries

Browse like a file tree:

```graphql
query BrowseAssets($input: AssetBrowseInput!) {
  assetBrowse(input: $input) {
    currentFolder { id name path parentId }
    folders { id name path childFolderCount directAssetCount recursiveAssetCount }
    assets { id name extension width height duration rating updatedAt }
    pageInfo { nextCursor hasMore }
    summary { directAssetCount recursiveAssetCount imageCount videoCount audioCount folderCount }
  }
}
```

Variables:

```json
{ "input": { "path": "/Characters", "depth": 1, "includeAssets": true, "assetLimit": 50 } }
```

Resolve known IDs:

```graphql
query AssetsByIds($ids: [String!]!) {
  assetsByIds(ids: $ids) {
    id
    name
    extension
    genParams
  }
}
```

## Mutations

Ensure a folder path:

```graphql
mutation EnsureFolder($input: EnsureFolderPathInput!) {
  ensureFolderPath(input: $input) {
    folderId
    path
    createdCount
    folders { id name path parentId created }
  }
}
```

Variables:

```json
{
  "input": {
    "path": "/Shots/Act 1"
  }
}
```

For an internal Syncast Agent `imagine` call, pass the returned `folderId` as
`target_folder_id`; the internal Agent contract never accepts a folder path. For
external CLI generation, prefer the user-facing `syncast imagine --folder
<name-or-path>` and let project materialization create missing path segments.
If no explicit destination is required, omit the folder option and use Assets root.

Move assets into an existing or newly-created folder path:

```graphql
mutation MoveAssets($input: MoveAssetsToFolderInput!) {
  moveAssetsToFolder(input: $input) {
    success
    folderId
    movedAssetIds
    error
  }
}
```

Variables:

```json
{
  "input": {
    "assetIds": ["asset-a", "asset-b"],
    "folderPath": "/Shots/Act 1"
  }
}
```

Batch organize in one mutation:

```graphql
mutation OrganizeAssets($input: OrganizeAssetsInput!) {
  organizeAssets(input: $input) {
    success
    ensuredFolders { folderId path createdCount }
    movedAssets { success folderId movedAssetIds error }
    updatedAssets { success updatedAssetIds error }
    errors
  }
}
```

Use `deleteFolders.strategy` carefully:

- `detach_assets`: remove folder, leave assets unfiled.
- `move_assets_to_root`: keep assets in root asset order.
- `delete_assets`: destructive asset deletion.
