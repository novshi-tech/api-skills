# Permissions（共有設定）

すべてのパスは `{BASE_URL}/files/{fileId}` からの相対パス。boidゲートウェイ経由の場合は `{BASE_URL}` = `$BOID_API_BASE/drive-api/drive/v3`、直接呼び出しの場合は `{BASE_URL}` = `https://www.googleapis.com/drive/v3`（詳細は [SKILL.md](../SKILL.md) 参照）。共有ドライブのルート自体への権限設定も同じ `permissions` エンドポイントを共有ドライブID（`{driveId}`）に対して使う。

## 一覧取得

```
GET /files/{fileId}/permissions
```

主なクエリパラメータ:
- `fields` — 既定では最小限のフィールドしか返らないため、`fields=permissions(id,type,role,emailAddress,domain)` のように明示する
- `supportsAllDrives` — 共有ドライブ上のファイルの場合は `true`
- `includePermissionsForView` — `published` を指定するとウェブ公開に関する権限も含める
- `pageSize` — 1ページあたりの最大件数（最大100。100を超える値を指定しても100に丸められる。デフォルトは共有ドライブ上のファイルでは100件、非共有ファイルでは原則全件）
- `pageToken` — 前回レスポンスの `nextPageToken` を渡してページングを続ける（[pagination-and-errors.md](pagination-and-errors.md) 参照）

## 個別取得

```
GET /files/{fileId}/permissions/{permissionId}
```

## 権限の付与（共有）

```
POST /files/{fileId}/permissions
Content-Type: application/json
```

```json
{
  "type": "user",
  "role": "writer",
  "emailAddress": "someone@example.com"
}
```

主なクエリパラメータ:
- `sendNotificationEmail` — `type=user`/`group` の場合、共有通知メールを送るか（デフォルト `true`）。大量共有時のスパム防止でメール本文を付けたい場合は `emailMessage` ボディフィールドと併用する
- `transferOwnership` — `role=owner` を付与しオーナー移譲する場合は `true` にする（Google Workspaceアカウント間のみ、個人Gmailアカウントへのオーナー移譲はUI経由の別フローが必要な場合がある）
- `moveToNewOwnersRoot` — オーナー移譲時、ファイルを新オーナーのマイドライブ直下に移動するか
- `supportsAllDrives` — 共有ドライブ上のファイルの場合は `true`

### `role` の値

| role | 意味 |
|---|---|
| `owner` | 所有者（マイドライブのファイルのみ。共有ドライブアイテムには存在しない） |
| `organizer` | 整理者（主に共有ドライブ自体に対して） |
| `fileOrganizer` | ファイル整理者（共有ドライブ内でアイテムの移動・整理が可能） |
| `writer` | 編集者 |
| `commenter` | コメント可能（閲覧＋コメント） |
| `reader` | 閲覧のみ |

### `type` の値

| type | 意味 | 必須の追加フィールド |
|---|---|---|
| `user` | 特定の個人 | `emailAddress` |
| `group` | Googleグループ | `emailAddress`（グループのメールアドレス） |
| `domain` | 組織ドメイン全体 | `domain` |
| `anyone` | リンクを知っている全員（インターネット全体） | なし |

`type=domain` / `type=anyone` の場合、`allowFileDiscovery` で検索可能にするかどうかを制御できる（`false` にすると「リンクを知っている人のみ」、`true` にすると検索結果にも出る）。

### 期限付き共有

```json
{
  "type": "user",
  "role": "reader",
  "emailAddress": "someone@example.com",
  "expirationTime": "2026-12-31T00:00:00.000Z"
}
```

`expirationTime` はRFC 3339形式。`reader`/`commenter` の `user`/`group` タイプにのみ設定可能で、現在時刻から最大1年先までという制約がある（Google Workspace管理者の設定によりさらに短く制限される場合もある）。

## 権限の更新

```
PATCH /files/{fileId}/permissions/{permissionId}
```

部分更新（`role` の変更など）。`type`/`emailAddress`/`domain` は作成後に変更できない（削除して作り直す）。

```json
{ "role": "commenter" }
```

## 権限の削除（共有解除）

```
DELETE /files/{fileId}/permissions/{permissionId}
```

オーナー権限（`role=owner`）の削除はできない（オーナーを外したい場合は先に別ユーザーへオーナー移譲する）。

## 共有関連のその他フィールド（`files` リソース側）

権限自体は `permissions` サブリソースだが、共有状態を素早く確認したい場合は `files.get` で以下のフィールドを見るとよい。

| フィールド | 説明 |
|---|---|
| `shared` | 誰かと共有済みかどうかの真偽値 |
| `ownedByMe` | 自分がオーナーかどうか |
| `viewersCanCopyContent` | 閲覧者がコピー・ダウンロード・印刷できるか（非推奨気味、`capabilities.canDownload` を見る方が確実） |
| `writersCanShare` | 編集者がさらに他者と共有できるか |
| `capabilities.canShare` | 現在の認証ユーザーがこのファイルの共有設定を変更できるか |
| `permissionIds[]` | このファイルに直接・間接に紐づく権限IDの一覧（`fields` で明示指定時のみ） |

## 共有ドライブ（Shared Drive）でのメンバー管理との違い

共有ドライブ自体へのメンバー追加（`role=organizer` 等をドライブ全体に付与）も同じ `permissions` エンドポイントを使うが、`fileId` の代わりに共有ドライブのID（`driveId`）を渡す点、および `supportsAllDrives=true` を必ず付ける点に注意。共有ドライブそのものの作成・取得・一覧は `drives` リソース（`GET /drives`, `POST /drives` 等）が別途存在するが、本ドキュメントでは詳細を割愛する（必要になった際に公式リファレンスで確認すること）。
