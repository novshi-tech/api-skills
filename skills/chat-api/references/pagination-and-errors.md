# ページネーション / エラー形式 / レート制限

## ページネーション

一覧系エンドポイントはGoogle API共通のトークンベースのページネーション形式を返す。

```json
{
  "spaces": [ ... ],
  "nextPageToken": "CAgSBAoCCAo"
}
```

- リクエスト側は `pageSize`（1ページあたりの最大件数）と `pageToken`（前回レスポンスの `nextPageToken`）で制御する
- レスポンスのフィールド名はリソースごとに異なる（`spaces.list` なら `spaces[]`、`spaces.messages.list` なら `messages[]`、`spaces.members.list` なら `memberships[]`）。共通なのは `nextPageToken` という名前と、それがトークン文字列であって連番のページ番号ではない点
- `nextPageToken` が存在しない（省略される、または空文字）場合が最終ページ
- Bitbucketの `next` のような完全URLではなく**不透明な文字列トークン**なので、boidゲートウェイ経由であっても付け替え処理は不要。トークンをそのまま次リクエストの `pageToken` に渡すだけでよい

### 実装パターン（擬似コード、boidゲートウェイ経由）

```python
base = f"{os.environ['BOID_API_BASE']}/chat-api"
url = f"{base}/v1/spaces"
params = {"pageSize": 100}
results = []
while True:
    resp = http_get(url, params=params, cacert=os.environ.get("BOID_API_CA_FILE"))  # Authorizationヘッダは付けない
    data = resp.json()
    results.extend(data.get("spaces", []))
    token = data.get("nextPageToken")
    if not token:
        break
    params["pageToken"] = token
```

直接呼び出し（boid外）の場合もページネーションのロジック自体は同じ（ベースURLが `https://chat.googleapis.com` になるだけ）。

## エラーレスポンス形式（Google Chat API自体が返すもの）

Google API共通のエラーモデル。

```json
{
  "error": {
    "code": 404,
    "message": "Requested entity was not found.",
    "status": "NOT_FOUND",
    "details": [ ... ]
  }
}
```

- `code` — HTTPステータスコードと同じ数値
- `status` — gRPC由来のシンボリックなステータス名（`NOT_FOUND`, `PERMISSION_DENIED` 等）。プログラムから判定する場合はHTTPステータスだけでなくこちらも見た方が確実
- `message` — 開発者向けの英語メッセージ（ユーザー表示用ではない）
- `details[]` — エラーの種類によっては `google.rpc.ErrorInfo` などの追加コンテキストが入る

### 主なHTTPステータス（Google Chat API自体）

| ステータス | `status` | 意味 | 典型的なケース |
|---|---|---|---|
| 400 Bad Request | `INVALID_ARGUMENT` | リクエスト形式不正・バリデーションエラー | 必須フィールド欠落、不正な `spaceType`、不正な `updateMask` |
| 401 Unauthorized | `UNAUTHENTICATED` | 未認証・トークン無効 | アクセストークン未指定/期限切れ |
| 403 Forbidden | `PERMISSION_DENIED` | 権限不足 | スコープ不足、対象スペースの非メンバー、Chat App未インストール、ユーザー認証専用操作をApp認証で呼んだ（逆も同様） |
| 404 Not Found | `NOT_FOUND` | リソースが存在しない | spaceID/messageID/memberIDの誤り、権限がなく見えない場合も404になることがある |
| 409 Conflict | `ALREADY_EXISTS` | 競合 | 同一`requestId`での重複作成、既存メンバーシップへの再追加 |
| 429 Too Many Requests | `RESOURCE_EXHAUSTED` | クォータ超過 | 後述のレート制限 |
| 500系 | `INTERNAL` / `UNAVAILABLE` | Google側の一時的な問題 | リトライ対象 |

## boidゲートウェイが返すエラー（ゲートウェイ経由の場合のみ）

**重要:** ゲートウェイが生成したエラーはGoogle Chat APIの `{"error": {...}}` 形式ではなく、**プレーンテキストのボディ**（`Content-Type: text/plain`）で返る。レスポンスボディが上記のGoogle標準JSON形式でない場合、それはGoogle Chatではなくゲートウェイが弾いたエラーだと判断できる。

| ステータス | ボディの典型例 | 原因 |
|---|---|---|
| 404 | `404 page not found` | リクエストパスが `/api/<job-token>/<service>/<tail>` の形に合っていない、または `.`/`..` を含むパストラバーサル的なパス |
| 401 | `unauthorized: invalid or expired job token` | job token自体が不明・失効（ジョブ終了後は無効化される） |
| 403 | `forbidden: service not permitted for this job token` | `<service>` を `config.yaml` の `services:` に定義しただけでは不十分。ジョブ/ワークスペース側でそのサービスを明示的に有効化していないと出る |
| 403 | `forbidden: read-only job token may only use GET/HEAD` | read-only jobからPOST/PATCH/PUT/DELETEなど書き込み系メソッドを呼んだ。メッセージ送信・スペース作成・メンバー追加などはread-only jobからは実行できない |
| 502 | `bad gateway: service X is not configured` | `<service>` という名前が `config.yaml` の `services:` に存在しない（サービス名の誤り） |
| 502 | `bad gateway: api gateway credential resolution failed...` | `secret_key` に対応するシークレットが未設定、またはシークレット解決自体が失敗（fail-closed） |
| 502 | `bad gateway: upstream request failed for service X` | 実際のGoogle Chatへの転送時にネットワーク的な失敗。メッセージからは実アップストリームのホスト名は分からないよう意図的に伏せられている |
| 503 | `service unavailable: api gateway has no secret resolver configured` | boidデーモン自体にシークレットストアが配線されていない（運用者側の設定不足） |

401/403がGoogle標準のJSON（`{"error": {...}}`）で返ってきた場合はGoogle Chat側の権限問題、プレーンテキストで返ってきた場合は上表のゲートウェイ側の問題として切り分けること。加えてChat APIでは、403がGoogle標準JSONで返っていても「ユーザー認証専用の操作をApp認証の資格情報で呼んだ（またはその逆）」というAPI仕様レベルの原因があり得るので、[authentication.md](authentication.md) の認証方式ごとの操作範囲も併せて確認すること。

## レート制限 / クォータ

Google Chat APIのクォータは3種類ある:

- **プロジェクト単位**（Chat App/OAuthクライアントごと、直近60秒間）: メッセージ書き込み系（create/patch/delete）は3,000リクエスト、メッセージ読み取り系（get/list）は3,000リクエスト、スペース書き込み系（create/patch/delete）は60リクエスト、メンバーシップ操作は操作の種類により300〜3,000リクエスト、が目安（Google Cloud Consoleの「割り当てとシステムの上限」で正確な値・現状の消費量を確認できる）
- **スペース単位**（直近1秒間、そのスペースを操作する全アプリ・全ユーザーで共有）: 通常の読み取りは15リクエスト/秒、通常の書き込みは1リクエスト/秒、リアクション作成は5リクエスト/秒、データインポート書き込みは10リクエスト/秒が目安
- **ユーザー単位**: 個々のユーザーからのクエリレートにも上限がある

429を受け取った場合は指数バックオフ（`min(((2^n)+random_ms), max_backoff)`、ジッター込み、上限は数十秒程度）でリトライすること。具体的な数値はGoogle側の裁量で変更されうるためコードにハードコードせず、429ベースの動的なバックオフ実装にする。

boidゲートウェイ経由の場合、Google Chat自体のレート制限がそのまま透過するのが基本（ゲートウェイ自体に独自のレート制限機構があるとは限らない）。502/503が返ってきた場合はレート制限ではなく上表のゲートウェイ側の問題を疑う。

Incoming Webhookは上記とは別に「スペースの全Webhook合算で1リクエスト/秒」という、より厳しい固有の制限がある（詳細は [spaces-and-messages.md](spaces-and-messages.md) の「Incoming Webhook」参照）。

## 共通クエリパラメータ

| パラメータ | 説明 |
|---|---|
| `pageSize` | 1ページの件数上限 |
| `pageToken` | 前回レスポンスの `nextPageToken` を渡してページ送り |
| `filter` | リソースごとに使えるフィールドが異なる（`spaces-and-messages.md`/`membership.md` 参照） |
| `updateMask` | PATCH系メソッドで更新対象フィールドをカンマ区切りで指定 |
| `useAdminAccess` | `spaces.get/patch/delete`、`spaces.members.*`、`spaces.search` で使える。`true` にすると、Workspace管理者アカウント + `chat.admin.*` 系スコープを条件に、呼び出し元が参加していないスペースも含めて操作できる（ユーザー認証のみ、Chat App認証では使えない）。詳細は [authentication.md](authentication.md) を参照 |
| `fields` | Google API共通の部分レスポンス（partial response）パラメータ。レスポンスに含めるフィールドをカンマ区切り・ドット区切りで指定し、不要なフィールドを削らせてレスポンスサイズを減らせる（例: `fields=spaces(name,displayName),nextPageToken`）。すべてのメソッドで使える汎用パラメータで、Chat API固有のものではない |
