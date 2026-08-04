# Memberships（spaces.members）

すべてのパスは `{BASE_URL}/v1` からの相対パス。boidゲートウェイ経由の場合は `{BASE_URL}` = `$BOID_API_BASE/chat-api`、直接呼び出しの場合は `{BASE_URL}` = `https://chat.googleapis.com`（詳細は [SKILL.md](../SKILL.md) 参照）。

## Membership リソースの主なフィールド

| フィールド | 説明 |
|---|---|
| `name` | `spaces/{space}/members/{member}` 形式のリソース名 |
| `state` | `JOINED` / `INVITED` / `NOT_A_MEMBER` |
| `role` | `ROLE_MEMBER`（通常メンバー） / `ROLE_MANAGER`（スペースマネージャー） / `ROLE_ASSISTANT_MANAGER` |
| `member` | ユニオンフィールド。人間ユーザーまたはChat Appを表す `User` オブジェクト（`name`, `type`（`HUMAN`/`BOT`）, `displayName` 等） |
| `groupMember` | ユニオンフィールド。Googleグループの参照（`name` フィールドのみ） |
| `createTime` | メンバーシップ開始時刻 |
| `deleteTime` | メンバーシップ終了時刻（削除済みの場合） |
| `affiliation` | `INTERNAL` / `EXTERNAL` / `MANAGED_EXTERNAL`（組織との関係） |

## 一覧

```
GET /{parent=spaces/*}/members
```

クエリパラメータ:
- `pageSize` / `pageToken`
- `filter` — 例: `role = "ROLE_MANAGER"`, `member.type = "HUMAN"`
- `showGroups` — Googleグループのメンバーシップを含めるか（デフォルト含めない）
- `showInvited` — `INVITED` 状態のメンバーシップを含めるか
- `useAdminAccess` — `true` にすると、Workspace管理者権限で「自分が参加していないスペース」のメンバーシップも一覧できる（要管理者アカウント + `chat.admin.memberships`/`chat.admin.memberships.readonly` スコープ）。`spaces.members.get`/`create`/`patch`/`delete` でも同様に使える。詳細は [authentication.md](authentication.md) と [pagination-and-errors.md](pagination-and-errors.md) を参照

## 取得

```
GET /{name=spaces/*/members/*}
```

`{member}` 部分にはユーザーIDまたは `app`（Chat App自身を指す特別な値）を指定できる。

## メンバー追加

```
POST /{parent=spaces/*}/members
Content-Type: application/json
```

人間ユーザーを追加:

```json
{
  "member": {
    "name": "users/alice@example.com",
    "type": "HUMAN"
  }
}
```

Chat App自身をスペースに追加（Appのインストール相当）:

```json
{
  "member": {
    "name": "users/app",
    "type": "BOT"
  }
}
```

Googleグループを追加（**ユーザー認証のみ**）:

```json
{
  "groupMember": { "name": "groups/{group_id}" }
}
```

- 人間ユーザー・Chat App自身の追加はユーザー認証・Chat App認証のどちらでも可能
- 追加できるのはそのスペースへの参加権限がある相手のみ。`externalUserAllowed` が `false` のスペースに組織外ユーザーを追加しようとすると失敗する

## メンバーシップ更新（ロール変更）

```
PATCH /{membership.name=spaces/*/members/*}
```

クエリパラメータ `updateMask=role` で `role` をマネージャーに昇格/降格させる用途などに使う。

## メンバー削除

```
DELETE /{name=spaces/*/members/*}
```

自分自身を退出させる場合もこのエンドポイントを使う（`{member}` に自分自身のIDを指定）。スペースの最後のマネージャーを削除しようとするとエラーになることがある。

## ロールの使い分け

- `ROLE_MANAGER` — スペースの設定変更・メンバー管理・（`ANNOUNCEMENT_SPACE` の場合）投稿権限を持つ
- `ROLE_ASSISTANT_MANAGER` — マネージャーの一部権限のみを持つ補助的なロール
- `ROLE_MEMBER` — 通常の参加者
