---
name: chat-api
description: Google Chat API（`chat.googleapis.com/v1`）の生のエンドポイント仕様（boidのAPIゲートウェイ経由での呼び出し方、スペース/メッセージ/メンバーシップ/スレッド/カードの各エンドポイント、ユーザー認証とChat App（bot）認証の違い、Incoming Webhook、ページネーション、エラー形式）をまとめたAPIリファレンススキル。`curl`やHTTPクライアント、SDKからGoogle Chat APIを直接叩くコードをboidサンドボックス内で書く・デバッグする・エンドポイント仕様を確認する場合に使用する。「Google Chat APIのエンドポイントを教えて」「Chat APIのメッセージ送信のレスポンス形式は」「Chat APIを叩くコードを書いて」「boid経由でGoogle Chat APIを呼ぶには」「BOID_API_BASEでChatを呼びたい」「CardsV2の仕様は」など、Google Chat APIの仕様そのものに関する質問・実装依頼で使用する。既存の `google-chat` CLIラッパー経由の操作（スペース一覧やタスク管理などのタスク実行）を頼まれた場合はこのスキルではなく `google-chat` CLIスキル（`name: google-chat`、`google-cli` を使う）を使うこと。
---

# Google Chat API リファレンス（boid APIゲートウェイ経由）

Google Chat API（`https://chat.googleapis.com/v1`）の仕様を、**boidのAPIゲートウェイ（`internal/apigateway`）経由で呼び出す**前提でまとめたリファレンス。boidのサンドボックス化されたジョブの中からGoogle Chat APIを直接叩くコードを書いたり、レスポンス形式を確認したりする際に使う。

このスキルは **APIの仕様書** であり、社内の `google-chat` CLI（`google-cli`）の使い方ガイドではない。CLI経由の操作を頼まれた場合は `google-chat` スキル（CLIスキル、`name: google-chat`）を使うこと。

## 最重要: ベースURLはハードコードしない・boidゲートウェイ経由で呼ぶ

Google Chat API自体の素のベースURLは `https://chat.googleapis.com`（バージョンパスは `/v1/...`）だが、**boid配下のジョブはこのホストに直接アクセスしない。** boidはサンドボックス化されたジョブが認証情報を一切保持しないまま外部APIを呼べるように、認証ゲートウェイ（`internal/apigateway`）を挟む設計になっている。

### 仕組み

1. boidはジョブ起動時に環境変数を自動注入する:
   - `BOID_API_BASE` — 形式は `https://boid-gateway:<port>/api/<job-token>`。ポートはジョブごとに動的に割り当てられるため固定値を仮定しない。**値は必ずこの環境変数から読み、コード中に書き起こさない**
   - `BOID_API_CA_FILE` — ゲートウェイのTLS終端が使う内部CA証明書のパス（Node.jsジョブでは、プロジェクト/ワークスペース側で `NODE_EXTRA_CA_CERTS` を既に設定していない限り、boidが自動でも設定する）
2. Google Chat APIを呼ぶ側は、`https://chat.googleapis.com/v1/...` ではなく次の形でリクエストする:

   ```
   $BOID_API_BASE/<service>/<chat-api-path>
   ```

   `<service>` はboidの `config.yaml` の `services:` ブロックで運用者が定義したサービス名。Chat向けの慣例的な名前は **`chat-api`**（`base_url: https://chat.googleapis.com` にマッピングされ、パスは `/v1/...` から始まる想定）。ただし固定の組み込み名ではないため、実際に何という名前で登録されているかは呼び出し元の `config.yaml` を確認するか、不明ならユーザーに確認すること。

3. ゲートウェイは以下を行う:
   - リクエストパス `/api/<job-token>/<service>/<tail>` をパースし、job tokenを検証する
   - `<service>` がそのjob tokenに許可されたサービス集合に含まれるかを確認する。**`services:` に定義しただけでは足りず、ワークスペース側で当該サービスを有効化していないと403になる**（`boid workspace services add` 等。詳細は [references/pagination-and-errors.md](references/pagination-and-errors.md)）
   - read-only jobの場合、GET/HEAD以外のメソッド（POST/PATCH/PUT/DELETE等）は問答無用で403になる。メッセージ送信・スペース作成・メンバー追加など書き込み系操作をread-only jobから呼ぶことはできない
   - **クライアントが送った `Authorization` / `Cookie` / `Proxy-Authorization` ヘッダは必ず剥がして無視する**（サンドボックス側が本物の資格情報を持つことは想定されていない）
   - `services.<service>.auth` の設定に従って実際の認証情報をシークレットストアから解決し、注入してから実際の `base_url`（`https://chat.googleapis.com`）に転送する
   - リクエストの `<tail>` パス（クエリ文字列含む）はバイト単位でそのまま転送される（正規化・デコードし直しなどはしない）。リソース名に含まれるスラッシュや `:setup` / `:search` のようなカスタムメソッドの `:` はそのまま正しくエンコードすること
   - 実際のアップストリームのホスト名はエラー時も含めてサンドボックス側には一切見えない
   - 資格情報の注入に失敗した場合は認証情報なしで転送せず、502で失敗する（fail-closed）。ゲートウェイが返すエラーはGoogle Chat APIの `{"error": {...}}` 形式ではなく **プレーンテキスト**。詳細なステータス表は [references/pagination-and-errors.md](references/pagination-and-errors.md) を参照

つまり **クライアント側でGoogle OAuthのアクセストークンを組み立てる必要はない（組み立てても剥がされて無視される）。** `$BOID_API_BASE/<service>/...` に対してリクエストを投げるだけでよい。

### curlでの基本形

```bash
curl --cacert "$BOID_API_CA_FILE" \
  "$BOID_API_BASE/chat-api/v1/spaces"
```

- `--cacert "$BOID_API_CA_FILE"` はゲートウェイが内部CAでTLS終端している場合に必要（省略すると証明書検証エラーになる）。`BOID_API_CA_FILE` が未設定であれば付けなくてよい。Node.jsではプロジェクト側で `NODE_EXTRA_CA_CERTS` を明示的に上書きしていない限り自動で通るため、通常フラグ相当の指定は不要
- 独自の `Authorization` ヘッダは付けない（付けても無視される）
- `chat-api` の部分は運用者が `config.yaml` で定義したサービス名に置き換える。慣例上この名前が使われるが確定ではない
- このドキュメント内のURL例はすべて `$BOID_API_BASE/chat-api` をベースとして記述する。実装時は環境変数をそのまま使い、URLを直書きしない

### ユーザー認証とChat App（bot）認証の違いは、ゲートウェイ経由でも解消しない

Google Chat APIには「**ユーザー認証**（人間のユーザーの代理としてOAuthで呼ぶ）」と「**Chat App（bot）認証**（サービスアカウント/Chat App自身の資格情報として呼ぶ）」の2系統があり、**どちらの認証で呼ぶかによって実行できる操作が異なる**。boidゲートウェイはこの区別を消してくれるわけではなく、`services.<service>.auth` にどちらの種類の資格情報が設定されているかで、実際に何ができるかが決まる。

- ユーザー認証・Chat App認証のどちらでも可能: `spaces.list/get`、`spaces.messages.create/get/list/update/delete`（メッセージ送信はどちらの認証でも可能）、`spaces.members.list/get`、`spaces.delete`（App認証は管理者承認+`chat.app.delete`スコープが必要で、かつ**そのAppが作成したスペースに限定**される）
- **Chat App認証でないと使えない**操作の例: Chat Appとしてのプライベートメッセージ（`privateMessageViewer` を使う応答）、`chat.bot` スコープが前提の一部の応答系操作、管理者承認が必要な `chat.app.*` スコープの操作
- **ユーザー認証でないと使えない**操作の例: `spaces:setup`、`spaces:completeImport`（要ドメイン全体の委任）、添付ファイルアップロード（`attachments:upload`）、リアクション（`spaces.messages.reactions`）、カスタム絵文字（`customEmojis`）、Googleグループのメンバーシップ追加、`spaces.messages.search`
- 詳細な使い分けは [references/authentication.md](references/authentication.md) を参照

つまりコードを書く前に「このゲートウェイのサービス設定はユーザー認証とChat App認証のどちらの資格情報を注入する構成か」を確認する必要がある。不明な場合は憶測で実装せずユーザーに確認すること。

### boidゲートウェイを経由しない/直接叩く場合

boidのサンドボックス外（ローカル開発、CI、他システムなど）から直接Google Chat APIを呼ぶ場合は、通常のGoogle OAuth 2.0認証（[references/authentication.md](references/authentication.md) 参照）に従って `https://chat.googleapis.com` を直接叩く。

**判断基準:** `BOID_API_BASE` がセットされていればboidジョブ内なので必ずゲートウェイ経由で呼ぶ。**boidサンドボックス内であることが明らかなのに `BOID_API_BASE` が未設定の場合は、「このジョブにはAPIゲートウェイが配線されていない」ことを意味する。** サンドボックスは資格情報を保持せず外向きの通信も制限されているため、この状態で `https://chat.googleapis.com` に直接フォールバックしても成功しない。認証情報を自作したり直接呼び出しにフォールバックしたりせず、処理を止めてユーザーに「このジョブ向けにchat-api相当のサービスがboidのAPIゲートウェイに登録・有効化されているか」を確認すること。

### Incoming Webhookは別扱い

スペースに設定した「Incoming Webhook」のURL（`https://chat.googleapis.com/v1/spaces/{space}/messages?key=...&token=...`）にメッセージを直接POSTする方式は、通常のAPI呼び出しとは認証モデルが根本的に異なる（`key`/`token` というクエリパラメータ自体が資格情報であり、`services.<service>.auth` によるヘッダ注入の対象ではない）。boidゲートウェイの標準的な `chat-api` サービス定義がこの用途を想定しているとは限らないため、webhook経由での送信が必要な場合はゲートウェイ設定がこれをどう扱う想定かを事前にユーザー・運用者に確認すること。詳細は [references/spaces-and-messages.md](references/spaces-and-messages.md) の「Incoming Webhook」を参照。

## 認証

クライアント自身が認証ヘッダを組み立てる必要は通常ない（ゲートウェイが代行する）。ユーザー認証/Chat App認証の違い、ゲートウェイ側の設定例、直接呼び出し時の認証方式、エラー時の切り分けは [references/authentication.md](references/authentication.md) を参照。

## リソース別リファレンス

タスクに応じて該当ファイルを読むこと。全部を毎回読み込む必要はない。

- [references/authentication.md](references/authentication.md) - boidゲートウェイでの認証代行の仕組み、ユーザー認証とChat App認証の違い・スコープ、直接呼び出し時の認証方式
- [references/spaces-and-messages.md](references/spaces-and-messages.md) - spaces（DM/グループチャット/スペースの違い、CRUD、`spaces.search`、`spaces.findGroupChats`）、messages（テキスト/CardsV2、スレッド、リアクション、`spaces.messages.search`、`spaces.messagePins`）、`spaces.spaceEvents`、`users.spaces`（既読状態）、`customEmojis`、Incoming Webhook
- [references/membership.md](references/membership.md) - spaces.members のCRUD、ロール、Googleグループメンバーシップ
- [references/pagination-and-errors.md](references/pagination-and-errors.md) - ページネーション形式、エラーレスポンス形式、レート制限/クォータ、boidゲートウェイが返すエラー一覧、共通クエリパラメータ（`useAdminAccess`, `fields` 等）

## 注意点

- リソース名はGoogle API共通の resource name 規約に従う（`spaces/{space}`, `spaces/{space}/messages/{message}`, `spaces/{space}/members/{member}` など）。パス変数にこれらの値を埋め込む際は `spaces/` などのプレフィックスを含めた完全なリソース名なのか、末尾のIDだけなのかをエンドポイントごとに確認すること（メソッドによって `{name=spaces/*}` のように完全修飾名をパスに埋め込む形が多い）
- 日時は基本RFC 3339（ISO 8601、UTC）
- カスタムメソッドは `resource:methodName` の形（例: `spaces:setup`, `spaces:findDirectMessage`, `spaces/{space}/messages/{message}:replaceCards`）。RESTfulなCRUDパスと混同しないこと
- 本ドキュメントの内容は公開仕様（`developers.google.com/workspace/chat`）および boid リポジトリ（`internal/apigateway`, `docs/plans/api-gateway.md`, `docs/ja/reference/config-yaml.md`）の調査に基づく記載。Google Chat側の仕様変更や、運用者ごとの `config.yaml` のサービス名・認証設定のカスタマイズにより実際の挙動と差異が出ることがある。重要な実装の前には実際のレスポンス・実際の `config.yaml` で仕様を確認すること
