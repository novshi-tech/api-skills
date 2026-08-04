# 認証

Google Chat APIの認証は、**boidのAPIゲートウェイ経由か、直接呼び出しか**で扱いが大きく異なる。さらにGoogle Chat API自体が「ユーザー認証」と「Chat App（bot）認証」という2系統の認証モデルを持ち、**どちらで呼ぶかによって実行できる操作の範囲が変わる**点がBitbucketなど他のAPIと比べて特徴的。

## Google Chat APIの2つの認証系統（前提知識）

### 1. ユーザー認証（User Authentication）

人間のGoogleアカウントの代理としてOAuth 2.0で呼ぶ方式。「このユーザーがアクセスできるスペース・メッセージ」に対して操作する。ユーザーが所属していないスペースは見えない。

- 主なOAuthスコープ: `chat.spaces`（読み書き）/ `chat.spaces.readonly`、`chat.messages`（読み書き）/ `chat.messages.readonly` / `chat.messages.create`、`chat.memberships`（読み書き）/ `chat.memberships.readonly`
- ユーザー認証でしか呼べない代表的な操作: `spaces:setup`（スペース作成+メンバー招待を1リクエストで行う）、`spaces:completeImport`（**ドメイン全体の委任も別途必須**、`chat.import` スコープ）、添付ファイルアップロード（`attachments:upload`、App認証不可）、メッセージへのリアクション（`spaces.messages.reactions.*`）、カスタム絵文字（`customEmojis.*`）、Googleグループをスペースメンバーとして追加する操作、`spaces.messages.search`
- 管理者権限（`useAdminAccess=true` + Workspace管理者アカウント + `chat.admin.*` 系スコープ）を使うと、`spaces.get/patch/delete`・`spaces.members.*`・`spaces.search` を「自分が参加していないスペース」に対しても実行できる。詳細は [pagination-and-errors.md](pagination-and-errors.md) の共通クエリパラメータを参照

### 2. Chat App（bot）認証（App Authentication）

Chat App（Google Workspace Marketplaceに公開する/組織内限定で使うbot）自身のサービスアカウント資格情報として呼ぶ方式。ユーザーの同意画面を経由せず、Chat Appとして「自分自身が」操作する。

- 主なスコープ: `chat.bot`（非機微スコープ、chatを見てメッセージを送るための基本スコープ）、管理者承認が必要な `chat.app.spaces.create`、`chat.app.spaces`、`chat.app.delete`、`chat.app.messages.readonly`、`chat.app.memberships` など
- Chat App認証でないと使えない代表的な操作: Chat Appとしてのプライベート応答（`privateMessageViewer` を使うメッセージ）、`chat.app.*` スコープが前提の管理者承認済み操作
- Chat App認証でも可能だが要注意な操作の例: `spaces.delete`（管理者承認 + `chat.app.delete` スコープが必要で、**そのAppが作成したスペースに限定**される）、`spaces.create`（`chat.app.spaces.create`/`chat.app.spaces` スコープ、管理者承認必須。**`space.customer`（`customers/{customer}` 形式）フィールドの指定が必須**。人間ユーザーは自動でメンバーにならず、Chat App自身がメンバーになる）
- Chat Appはユーザーからのメンション・スラッシュコマンド・カードのボタン押下などのインタラクションに**同期応答**する場合は追加の認証情報を必要としない（Chatが送ってくるリクエスト自体をイベントとして受け取り、レスポンスとしてメッセージを返すだけで済む）。これはboidゲートウェイ経由の「能動的にAPIを呼ぶ」ケースとは別の話なので混同しないこと

### どちらでも呼べる操作

`spaces.list/get`、`spaces.patch`（一部フィールドはApp認証不可のものもある）、`spaces.messages.create/get/list/update/patch/delete`（メッセージの送信・取得・更新・削除は両方の認証方式で可能）、`spaces.members.list/get/create/delete`（人間ユーザー・Chat App自身の追加はどちらでも可能。Googleグループの追加はユーザー認証のみ）。

**実装前に必ず確認すること:** あるエンドポイントが「ユーザー認証必須」「Chat App認証必須」「どちらでも可」のいずれかは操作ごとに異なる。公式リファレンス（`developers.google.com/workspace/chat/api/reference/rest`）の各メソッドページに認証要件が明記されているため、重要な実装の前に該当メソッドのページで確認すること。本ドキュメントの分類は執筆時点の調査に基づくものであり、Google側の仕様変更で変わりうる。

### `useAdminAccess` パラメータ（管理者アクセス）

`spaces.get/patch/delete`、`spaces.members.*`、`spaces.search` の一部はクエリパラメータ `useAdminAccess=true` をサポートする。これはユーザー認証の一種で、以下の3条件を満たすと「自分が参加していないスペース」も含めて操作できるようになる:

1. 呼び出すユーザーがGoogle Workspace管理者アカウントである
2. `useAdminAccess=true` を指定する
3. `chat.admin.spaces`/`chat.admin.spaces.readonly`/`chat.admin.delete`/`chat.admin.memberships` など `chat.admin.*` 系スコープを認可している（かつ「Chat と スペースの会話を管理」権限を保持している）

Chat App認証では使えない、ユーザー認証専用の仕組み。詳細は [pagination-and-errors.md](pagination-and-errors.md) の共通クエリパラメータを参照。

## boidゲートウェイ経由の場合（サンドボックス化されたジョブから呼ぶ場合）

boid配下のジョブは資格情報そのものを保持しない設計になっている。実際の認証はゲートウェイ（`internal/apigateway`）が代行する。

### クライアント側がやること

- **何もしない。** `Authorization` ヘッダを自分で組み立てて送る必要はない
- 送っても意味がない: ゲートウェイは受け取ったリクエストから `Authorization` / `Cookie` / `Proxy-Authorization` を必ず剥がしてから転送する（クライアント側の値は一切アップストリームに届かない）
- `--cacert "$BOID_API_CA_FILE"` を付けてTLS証明書を検証できるようにする（ゲートウェイは内部CAでTLS終端しているため、これがないと証明書エラーになる）

```bash
curl --cacert "$BOID_API_CA_FILE" \
  "$BOID_API_BASE/chat-api/v1/spaces?pageSize=1"
```

### ゲートウェイが「ユーザー認証」「Chat App認証」のどちらを注入するかは `config.yaml` 次第

boidゲートウェイの `services.<service>.auth` は1つの固定の資格情報しか表現しない。つまり運用者が `chat-api` サービスをどう登録したかによって、そのゲートウェイ経由で呼べる操作の範囲は変わる:

```yaml
services:
  chat-api:
    base_url: https://chat.googleapis.com
    auth:
      kind: bearer
      secret_key: CHAT_APP_ACCESS_TOKEN
```

- `auth.kind` は `bearer` / `basic` / `header` / `query` / `oauth2` から選べる。Google Chat APIはBearerトークン（OAuth 2.0アクセストークン、または `oauth2` kindでリフレッシュ管理させる構成）が基本
- `secret_key` に紐づく実際の値が、ユーザーのOAuthアクセストークンなのか、Chat App（サービスアカウント）のアクセストークンなのかによって、上記「どちらでも呼べる操作」の範囲外の操作（`spaces:setup` やリアクションなど）が使えるかどうかが決まる。`spaces.delete` のようにユーザー認証・Chat App認証の両方に対応があっても、後者は「管理者承認 + `chat.app.delete` スコープ + そのAppが作成したスペース限定」という追加条件があるため、Chat App認証の資格情報だからといって任意のスペースを削除できるとは限らない
- どちらの種類の資格情報が設定されているか不明な場合、`spaces:setup` のようなユーザー認証専用の操作や、Chat App専用の操作をコード上で仮定せず、まずユーザーに確認するか、実際に叩いて403/401が返らないか確認すること
- 資格情報の解決・注入に失敗した場合、ゲートウェイは認証情報なしで転送せず502を返す（fail-closed）
- ゲートウェイは他にも401（job token自体が無効・期限切れ）、403（サービスが未有効化 or read-only jobでの書き込み試行）、404（パス不正）、503（シークレットストア未接続）を返しうる。これらはGoogle Chat自体のエラーではなくゲートウェイが生成したもので、レスポンスボディもGoogleのJSON形式ではなくプレーンテキスト。ステータスごとの切り分けは [pagination-and-errors.md](pagination-and-errors.md) の一覧を参照

### サービス名は固定ではない

`chat-api` という名前はboidの組み込みデフォルトではなく、ドキュメント・テストで使われている慣例的な名前にすぎない。実際に何という名前で `services:` に登録されているかは運用者の `config.yaml` 次第。不明な場合はコード内で決め打ちせず、環境や設定から確認するか、ユーザーに確認する。

## 直接呼び出しの場合（boidサンドボックス外から）

`BOID_API_BASE` が環境変数にセットされていない、あるいはboidジョブの外（ローカル開発・CI等）から呼ぶ場合は、通常のGoogle Chat API認証を自前で扱う。

### 1. ユーザー認証（OAuth 2.0 Authorization Code Grant）

Google Cloud ConsoleでOAuthクライアントを作成し、Authorization Code Grant（+ 必要ならリフレッシュトークン）でユーザーの同意を得たアクセストークンを取得する。

```bash
curl -X GET "https://chat.googleapis.com/v1/spaces" \
  -H "Authorization: Bearer <user_access_token>"
```

必要なスコープ例: `https://www.googleapis.com/auth/chat.spaces`, `https://www.googleapis.com/auth/chat.messages`。

### 2. Chat App（bot）認証（サービスアカウント）

Chat Appとして登録したサービスアカウントの秘密鍵からJWTを発行し、Google OAuth 2.0トークンエンドポイントでアクセストークンに交換する（`google-auth-library` 等のクライアントライブラリが通常これを代行する）。

```bash
curl -X POST "https://chat.googleapis.com/v1/spaces/{space}/messages" \
  -H "Authorization: Bearer <service_account_access_token>" \
  -H "Content-Type: application/json" \
  -d '{"text": "Hello from the app"}'
```

必要なスコープ例: `https://www.googleapis.com/auth/chat.bot`。ドメイン全体の委任（domain-wide delegation）を設定していれば、特定ユーザーに成り代わって呼ぶことも可能だがこれは別途Workspace管理者の設定が必要。

### 3. Incoming Webhook（`key`/`token`によるクエリ認証）

スペースに個別に設定するWebhook URLは、`Authorization` ヘッダではなく **URL自体に埋め込まれた `key` と `token` クエリパラメータ**が資格情報になる。OAuthフロー自体が不要な代わりに、投稿先スペースが固定され、できる操作もメッセージ送信のみに限られる。詳細は [spaces-and-messages.md](spaces-and-messages.md) の「Incoming Webhook」を参照。

## 認証エラー時のレスポンス（直接呼び出しの場合）

Google Chat APIは他のGoogle APIと共通のエラー形式（`{"error": {"code", "message", "status", "details"}}`）を返す。

| ステータス | `status` | 意味 |
|---|---|---|
| 401 | `UNAUTHENTICATED` | アクセストークン未指定・無効・期限切れ |
| 403 | `PERMISSION_DENIED` | トークンは有効だが対象リソースへの権限がない（スコープ不足、対象スペースの非メンバー、Chat App未インストール等） |

ゲートウェイ経由の場合のエラー（401/403/404/502/503それぞれの原因の切り分け）は [pagination-and-errors.md](pagination-and-errors.md) の「boidゲートウェイが返すエラー」を参照。ゲートウェイ経由のエラーは上表のGoogle Chat標準の意味とは原因が異なることが多いので混同しないこと。
