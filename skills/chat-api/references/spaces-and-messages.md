# Spaces / Messages

すべてのパスは `{BASE_URL}/v1` からの相対パス。boidゲートウェイ経由の場合は `{BASE_URL}` = `$BOID_API_BASE/chat-api`、直接呼び出しの場合は `{BASE_URL}` = `https://chat.googleapis.com`（詳細は [SKILL.md](../SKILL.md) と [authentication.md](authentication.md) 参照）。

## Spaces

### DM・グループチャット・スペースの違い

`Space` リソースの `spaceType` フィールドで区別する（`type` フィールドは非推奨、`spaceType` を使う）:

| `spaceType` | 説明 |
|---|---|
| `DIRECT_MESSAGE` | 1対1のダイレクトメッセージ。人間同士、または人間とChat Appの1:1 DM（`singleUserBotDm` フラグで後者を判別）。作成はAPIから直接できず、`spaces:findDirectMessage` で既存のものを探すか、`spaces:setup` で作成する |
| `GROUP_CHAT` | 名前を持たない複数人のグループチャット。`displayName` は不要 |
| `SPACE` | 名前付きの「スペース」（Slackのチャンネルに相当）。作成時 `displayName` が必須 |

`predefinedPermissionSettings` で作成時に `COLLABORATION_SPACE`（全員投稿可）/ `ANNOUNCEMENT_SPACE`（マネージャーのみ投稿可）を指定できる。

### 一覧

```
GET /spaces
```

クエリパラメータ:
- `pageSize` — 1ページの最大件数。**デフォルト100、最大1000**（1000を超える値を指定すると自動的に1000に丸められる）
- `pageToken` — 次ページ取得用トークン
- `filter` — 例: `spaceType = "SPACE"`

呼び出し元の認証が「見えているスペース」の範囲を決める（ユーザー認証ならそのユーザーが参加しているスペース、Chat App認証ならそのAppがインストールされているスペース）。

### 横断検索（`spaces:search`）

```
GET /spaces:search
```

`spaces.list` が「呼び出し元から見えるスペース一覧」を返すのに対し、`spaces:search` は `query` パラメータによる検索条件でスペースを絞り込む専用エンドポイント。クエリパラメータのみを使い、リクエストボディは空にする。

クエリパラメータ:
- `query` — **必須**。例: `spaceType = "SPACE" AND displayName = "プロジェクトX"`
- `useAdminAccess` — `true` にすると、Workspace管理者としての権限で「自分が参加していないスペースも含めた」検索ができる（要 `chat.admin.spaces`/`chat.admin.spaces.readonly` スコープ + 管理者アカウント）。`true` の場合 `customer`（例: `customers/my_customer`）も必須になり、`query` で使えるフィールドが `createTime`/`lastActiveTime`/`spaceHistoryState` などに広がる
- `useAdminAccess=false`（またはデフォルト）の場合は `query` で使えるフィールドが `displayName`/`externalUserAllowed`/`spaceType` に限られ、自分が参加しているスペースのみが対象
- `pageSize` / `pageToken` / `orderBy`

### 取得

```
GET /{name=spaces/*}
```

クエリパラメータ `useAdminAccess=true` で、自分が参加していないスペースも管理者権限で取得できる（要管理者アカウント + `chat.admin.spaces`/`chat.admin.spaces.readonly` スコープ）。

主なレスポンスフィールド: `name`（`spaces/{space}` 形式）, `spaceType`, `displayName`, `spaceThreadingState`（`THREADED_MESSAGES` / `GROUPED_MESSAGES` / `UNTHREADED_MESSAGES`）, `spaceHistoryState`（`HISTORY_ON` / `HISTORY_OFF`）, `spaceDetails.description`, `spaceUri`, `membershipCount`, `createTime`。

### 作成

```
POST /spaces
Content-Type: application/json
```

```json
{
  "displayName": "プロジェクトX",
  "spaceType": "SPACE",
  "predefinedPermissionSettings": "COLLABORATION_SPACE"
}
```

クエリパラメータ `requestId` で冪等性を担保できる（同じ `requestId` での再送は同一リソースを返す）。`SPACE` タイプでは `displayName` 必須。

- **ユーザー認証の場合**: `chat.spaces.create`/`chat.spaces`（インポートモードなら `chat.import`）スコープ。`space.customer` は不要。呼び出したユーザー自身が自動的にメンバーになる
- **Chat App認証の場合**: `chat.app.spaces.create`/`chat.app.spaces` スコープ＋管理者承認が必須。**`space.customer`（`customers/{customer}` 形式）フィールドの指定が必須**（省略不可）。人間ユーザーは自動でメンバーにならず、Chat App自身がメンバーになる（インポートモードではメンバーは追加されない）

### スペース作成 + メンバー招待をまとめて行う（`spaces:setup`）

```
POST /spaces:setup
Content-Type: application/json
```

```json
{
  "space": { "spaceType": "SPACE", "displayName": "プロジェクトX" },
  "memberships": [
    { "member": { "name": "users/alice@example.com", "type": "HUMAN" } }
  ]
}
```

**ユーザー認証のみ**で呼べる操作。`space.spaceType` ごとに以下の制約がある:

| `spaceType` | `displayName` | `memberships` | `singleUserBotDm` |
|---|---|---|---|
| `SPACE` | 必須 | 任意（0件以上） | 設定しない |
| `GROUP_CHAT` | **設定禁止**（`Space.displayName` を設定しない） | **2件以上必須** | 設定しない |
| `DIRECT_MESSAGE`（人間同士） | 設定禁止 | **ちょうど1件** | `false` を設定 |
| `DIRECT_MESSAGE`（人間とChat Appの1:1 DM） | 設定禁止 | **空（設定しない）** | `true` を設定 |

`memberships` は呼び出し元本人を含めず最大49件まで指定できる（呼び出し元を含めると最大50名構成）。

### 既存のDMを探す

```
GET /spaces:findDirectMessage?name=users/{user}
```

指定したユーザーとの既存のDMスペースを返す（なければ404）。新規作成はしない。DM作成が必要な場合は `spaces:setup` を使う。

### 既存のグループチャットを探す

```
GET /spaces:findGroupChats
```

呼び出し元とメンバー構成が一致する既存のグループチャットスペースを検索する（`spaces:findDirectMessage` のグループチャット版）。ユーザー認証・Chat App認証のどちらでも呼べる。

### 更新

```
PATCH /{space.name=spaces/*}
```

クエリパラメータ `updateMask` で更新対象フィールドをカンマ区切りで指定する（例: `updateMask=displayName,spaceDetails.description`）。ボディには更新したいフィールドのみ含める。`useAdminAccess=true` で管理者権限による更新も可能（一部フィールドはApp認証不可）。

### 削除

```
DELETE /{name=spaces/*}
```

スペース自体とその全メッセージを削除する（取り消し不可）。

- **ユーザー認証**: `chat.delete` スコープ（インポートモードのスペース限定なら `chat.import`）。`useAdminAccess=true` + `chat.admin.delete` スコープ + 管理者アカウントであれば、自分が参加していないスペースも削除可能
- **Chat App認証**: `chat.app.delete` スコープ＋管理者承認が必須。**そのApp自身が作成したスペースに限定**される（他者が作成したスペースは削除できない）

### インポートモードの完了（データ移行用）

```
POST /{name=spaces/*}:completeImport
```

他システムからのメッセージ履歴インポート（`importMode: true` で作成したスペース）を完了し、通常のユーザーに見えるようにする。**ユーザー認証のみ**で、`chat.import` スコープに加えて**ドメイン全体の委任（domain-wide delegation）が必須**（Workspace管理者がサービスアカウントにドメイン全体の委任を許可し、対象ユーザーに成り代わって呼び出す構成が前提）。

## Messages

### 送信

```
POST /{parent=spaces/*}/messages
Content-Type: application/json
```

クエリパラメータ:
- `messageId` / `messageReplyOption` — スレッド指定時の挙動制御（後述）
- `threadKey` — 独自のスレッドキーを使ってスレッドを指定する場合（非推奨方向、`thread.threadKey` をボディで指定する方が新しい）

テキストメッセージ:

```json
{ "text": "デプロイが完了しました" }
```

- 装飾: `*太字*`, `_斜体_`, `~取り消し線~`, `` `等幅` ``
- コードブロック: バッククォート3つで複数行を囲む（`` ``` `` ... `` ``` ``）
- 箇条書き: 行頭に `* item` または `- item`（ネストはインデント4スペースごとに1段階）
- 引用: 行頭に `> text`
- メンション: `<users/{user}>`（特定ユーザー）、`<users/all>`（スペース内全員にメンション）
- レスポンスの `formattedText` は**出力専用**フィールドで、上記のMarkdown記法・メンション・リンク・カスタム絵文字などを反映した整形済みテキストが入る（送信時にリクエストボディへ指定するものではなく、`text` が入力用のプレーンテキストフィールド）

CardsV2（リッチカード）メッセージ:

```json
{
  "cardsV2": [
    {
      "cardId": "unique-card-id",
      "card": {
        "header": { "title": "デプロイ結果" },
        "sections": [
          {
            "widgets": [
              { "textParagraph": { "text": "ステータス: <b>成功</b>" } },
              {
                "buttonList": {
                  "buttons": [
                    { "text": "詳細を見る", "onClick": { "openLink": { "url": "https://example.com" } } }
                  ]
                }
              }
            ]
          }
        ]
      }
    }
  ]
}
```

- `text` と `cardsV2` は併用可能（`fallbackText` はカードを描画できないクライアント向けの代替テキスト）
- カード内のテキストはデフォルトでは一部のHTMLタグのサブセット（`<b>`, `<i>` 等）で装飾する形式だが、対応ウィジェットで `textSyntax` 設定を使ってMarkdown記法を有効化することもできる。「カードはMarkdownを使えない」わけではなく、デフォルトの書式がプレーンテキストの `text` フィールドと異なる点に注意

### スレッド指定

```json
{
  "text": "返信です",
  "thread": { "threadKey": "my-thread-key" }
}
```

- `thread.name`（`spaces/{space}/threads/{thread}` 形式、既存スレッドのリソース名）または `thread.threadKey`（呼び出し元が任意に決める文字列キー）のどちらかで指定する
- `messageReplyOption` クエリパラメータ:
  - `MESSAGE_REPLY_OPTION_UNSPECIFIED`（デフォルト、常に新規スレッド扱い）
  - `REPLY_MESSAGE_FALLBACK_TO_NEW_THREAD`（該当スレッドがあれば返信、なければ新規スレッド作成）
  - `REPLY_MESSAGE_OR_FAIL`（該当スレッドがなければエラー）
- `spaceThreadingState` が `UNTHREADED_MESSAGES` のスペースではスレッド指定は無視される

### 取得

```
GET /{name=spaces/*/messages/*}
```

### 一覧

```
GET /{parent=spaces/*}/messages
```

クエリパラメータ:
- `pageSize`（デフォルト25、最大1000） / `pageToken`
- `filter` — `createTime` と `thread.name` で絞り込み可能。例: `createTime > "2026-01-01T00:00:00Z" AND thread.name = "spaces/{space}/threads/{thread}"`。`thread.name` は1クエリにつき1件のみ
- `orderBy` — `createTime ASC`（デフォルト）または `createTime DESC`
- `showDeleted` — 削除済みメッセージ（本文は空、削除メタデータのみ）を含めるか

### 更新

```
PATCH /{message.name=spaces/*/messages/*}
PUT   /{message.name=spaces/*/messages/*}
```

`PATCH` は `updateMask` で差分更新、`PUT` は全体を置き換える。自分（呼び出し元の認証主体）が送信したメッセージのみ更新可能なのが基本。

### 削除

```
DELETE /{name=spaces/*/messages/*}
```

### メッセージ検索（`spaces.messages.search`）

```
POST /{parent=spaces/*}/messages:search
```

`parent` には単一のスペース（`spaces/{space}`）、または全スペースを横断検索するための特殊値 `spaces/-` を指定できる。全スペース横断で検索したい場合は次のパスになる:

```
POST /spaces/-/messages:search
```

**ユーザー認証のみ**（`chat.messages.readonly`/`chat.messages` スコープ）。ボディに `filter`（必須、最大1000文字。`createTime`/`sender.name`/`space.name`/`attachment`/`annotations.user_mentions.user.name` 等で絞り込み可能）、`pageSize`（デフォルト25、最大100）、`pageToken`、`orderBy`（`createTime` デフォルト、または `relevance`。降順のみ）、`view`（`SEARCH_MESSAGES_VIEW_BASIC` デフォルト or `SEARCH_MESSAGES_VIEW_FULL`）を指定する。

### カードの差し替え

```
POST /{name=spaces/*/messages/*}:replaceCards
```

既存メッセージの `cardsV2` だけを更新する（ボタン押下後にカードの見た目を更新する用途などで使う）。

### リアクション

```
POST   /{parent=spaces/*/messages/*}/reactions
GET    /{parent=spaces/*/messages/*}/reactions
DELETE /{name=spaces/*/messages/*/reactions/*}
```

**ユーザー認証のみ**で呼べる操作。ボディ例: `{ "emoji": { "unicode": "👍" } }`。

### メッセージのピン留め（`spaces.messages.messagePins`）

```
POST   /{parent=spaces/*/messages/*}/messagePins
GET    /{parent=spaces/*/messages/*}/messagePins
DELETE /{name=spaces/*/messages/*/messagePins/*}
```

メッセージのピン留め・一覧・解除を行う。既存メッセージのみピン留め可能（メッセージ作成とピン留めを同一リクエストではできない）。1スペースあたり最大100件までピン留め可能。ユーザー認証（`chat.spaces.pins`/`chat.spaces` スコープ）・Chat App認証のどちらでも利用可能。

### 添付ファイル

```
GET  /{name=spaces/*/messages/*/attachments/*}
POST https://chat.googleapis.com/upload/v1/{parent=spaces/*}/attachments:upload
GET  /media/{resourceName=**}
```

アップロードは通常の `/v1/...` ではなく `/upload/v1/...` という別パス（multipart形式）になる点に注意。ダウンロードは `media` リソース経由。

**アップロードは常にユーザー認証必須（Chat App認証では呼べない）**。`chat.messages.create`/`chat.messages` スコープ（インポートモードなら `chat.import`）が必要。アップロードできるファイルサイズの上限は**200MB**（一部のファイル形式はGoogle Chat側でブロックされる）。

## spaces.spaceEvents（変更イベントの取得）

```
GET /{name=spaces/*/spaceEvents/*}
GET /{parent=spaces/*}/spaceEvents
```

メッセージ作成/更新/削除、メンバーシップ変更、リアクション、スペース設定の更新など、スペース内で起きた変更をイベントとして取得できるリソース。ポーリングでの変更検知やWebhook/Pub Sub連携の代替として使う。

## users.spaces（既読状態）

```
GET   /{name=users/*/spaces/*/spaceReadState}
PATCH /{spaceReadState.name=users/*/spaces/*/spaceReadState}
```

呼び出し元ユーザーのスペースごとの既読状態（`SpaceReadState`、最後に既読になったメッセージの時刻等）を取得・更新する。`{user}` 部分は `me`（呼び出し元本人のエイリアス）・メールアドレス・ユーザーIDのいずれかで指定できる（例: `users/me/spaces/{space}/spaceReadState`）。**ユーザー認証のみ**（`chat.users.readstate`/`chat.users.readstate.readonly` スコープ）。スレッド単位の既読状態は `users.spaces.threads.threadReadState` で別途扱う。

## customEmojis（カスタム絵文字）

```
POST   /customEmojis
GET    /customEmojis/{customEmoji}
GET    /customEmojis
DELETE /customEmojis/{customEmoji}
```

組織のカスタム絵文字の作成・取得・一覧・削除。`list` の `pageSize` はデフォルト25、最大200。**ユーザー認証のみ**（`chat.customemojis`/`chat.customemojis.readonly` スコープ）。2026年8月時点でDeveloper Preview機能として位置付けられており、仕様変更の可能性がある点に注意。

## Incoming Webhook（別のAPIサーフェス）

Chat UIの「アプリと統合機能を管理」からスペースごとに発行できるURL:

```
https://chat.googleapis.com/v1/spaces/{space}/messages?key={key}&token={token}
```

- `key` はGoogle Cloudプロジェクト単位で共有される値、`token` はWebhookごとに一意で機密（URL自体が資格情報なので絶対に公開しないこと）
- 通常のAPIとは異なり `Authorization` ヘッダではなく、この `key`/`token` クエリパラメータ自体が認証情報になる
- POSTするボディは通常のメッセージ作成と同じ形（`{"text": "..."}` や `{"cardsV2": [...]}`）。**スレッドへの返信も可能**——通常の `spaces.messages.create` と同様に `thread.threadKey`（や `thread.name`）と `messageReplyOption` クエリパラメータを併用すれば、新規スレッドではなく既存スレッドへの返信として投稿できる（一方向の新規投稿しかできないわけではない）
- **できないこと**: メッセージの取得・一覧・更新・削除、スペース情報の取得、メンバーシップ操作。レスポンスにも `name` と `thread.name` しか含まれず、送信したメッセージの全フィールドは返らない
- レート制限は「そのスペースの全Webhook合算で1リクエスト/秒」とAPI全体のクォータより厳しい
- **boidゲートウェイとの関係**: `key`/`token` はクエリパラメータであり `Authorization` ヘッダではないため、ゲートウェイのヘッダ剥奪ロジックの対象外でそのまま転送される。ただし通常の `chat-api` サービス定義は「1つの固定資格情報をヘッダとして注入する」ことを想定しており、Webhookのように呼び出し先ごとに異なる `key`/`token` を持つ用途を想定した設計になっているとは限らない。boid経由でWebhookを叩きたい場合、その `key`/`token` を含んだURLをどう扱う構成になっているか（別サービス名で登録されているのか、単に `<tail>` としてそのまま転送する想定か）は運用者の `config.yaml` 次第なので、事前に確認すること
