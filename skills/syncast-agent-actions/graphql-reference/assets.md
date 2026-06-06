# Assets GraphQL

用于精确读取和整理项目资源、文件夹。生成前引用校验优先用封装 action `syncast.assets.resolveReferences`；需要批量整理资源树时再用本模块 mutation。

## Fields

Queries:

- `asset(id: String!)`
- `assets(folderId?: String, limit?: Int)`
- `assetsByIds(ids: [String!]!)`
- `folders(parentId?: String)`
- `assetBrowse(input: AssetBrowseInput!)`

Mutations:

- `createAsset(input: AssetCreateInput!)`
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
    remoteFilename
    genParams
  }
}
```

## Mutations

Ensure a folder path and move assets into it:

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
