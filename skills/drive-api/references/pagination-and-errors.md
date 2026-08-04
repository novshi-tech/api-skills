# ページネーション / エラー形式 / レート制限 / `fields` パラメータ

## ページネーション

一覧系エンドポイント（`files.list`, `permissions.list`, `changes.list` 等）の多くは次の形式を返す。

```json
{
  "kind": "drive#fileList",
  "incompleteSearch": false,
  "files": [ ... ],
  "nextPageToken": "~!!~AI9FV7...（不透明な文字列）"
}
```

- `nextPageToken` — 次ページ取得用の不透明なトークン。最終ページには含まれない
- `pageSize` — リクエスト時のクエリパラメータ（`files.list` は最大1000。**デフォルト値は共有ドライブ上の検索では100件だが、非共有ドライブ（マイドライブ）の検索では原則リスト全件が返る**という違いがある。`permissions.list` は最大100で、デフォルトは共有ドライブ上のファイルでは100件、非共有ファイルでは原則全件）。**Bitbucketのようなオフセット式の `page` 番号方式ではなくカーソル方式。** 総件数や現在ページ番号は返らないため、総件数に依存する実装をしないこと
- `incompleteSearch` — trueの場合、全データセットを検索しきれず打ち切られたことを示す（共有ドライブを跨いだ検索などで発生しうる）

### 次ページの取得

```
GET /files?pageToken={前回のnextPageToken}
```

**Google Drive APIの `nextPageToken` はBitbucketの `next` と異なり、完全なURLではなく不透明なトークン文字列のみ。** そのためboidゲートウェイ経由であっても、ホストの付け替えは不要で、そのまま `pageToken` クエリパラメータとして次のリクエストに使い回せる。

### 実装パターン（擬似コード、boidゲートウェイ経由）

```python
import os

base = f"{os.environ['BOID_API_BASE']}/drive-api/drive/v3"
url = f"{base}/files"
params = {"q": "trashed = false", "pageSize": 100, "fields": "files(id,name),nextPageToken"}
results = []
page_token = None
while True:
    if page_token:
        params["pageToken"] = page_token
    resp = http_get(url, params=params, cacert=os.environ.get("BOID_API_CA_FILE"))  # Authorizationヘッダは付けない
    data = resp.json()
    results.extend(data.get("files", []))
    page_token = data.get("nextPageToken")
    if not page_token:
        break
```

### resumableアップロードのセッションURIは別扱い

`uploadType=resumable` のセッション開始で返る `Location` ヘッダの値は、`nextPageToken` と異なり **Google側の絶対URL**（`https://www.googleapis.com/upload/drive/v3/files?uploadType=resumable&upload_id=...`）である。boidゲートウェイ経由の場合、これはBitbucketの `next` URLと同様にホストを剥がしてパス＋クエリだけ `$BOID_API_BASE/drive-api` に付け替える必要がある。詳細は [files.md](files.md) の「resumable」節を参照。

## `fields` パラメータ（部分レスポンス）

Drive APIはデフォルトで全フィールドを返さない（一覧系は特に最小限）。必要なフィールドは `fields` パラメータでFieldMask形式で明示する。**`about`/`comments`/`replies` リソースは `fields` の指定が実質必須**（省略するとほぼ空の応答になる）。

### 構文

- カンマ区切りで複数フィールド: `fields=id,name,mimeType`
- ネストしたオブジェクト: `fields=capabilities/canDownload`
- 配列・オブジェクトのサブフィールド指定は括弧: `fields=permissions(id,role,emailAddress)`
- ワイルドカードで配下全部: `fields=permissions/permissionDetails/*`
- 一覧系（`files.list` 等）はレスポンス自体がラップされているため、配列側も `files(...)` で包む: `fields=files(id,name),nextPageToken`
- 全フィールドが欲しい場合は `fields=*`（デバッグ用途。本番コードでは必要なものだけ指定するのが推奨）

### 例

```
GET /files?fields=files(id,name,mimeType,parents,modifiedTime),nextPageToken
GET /files/{fileId}?fields=id,name,owners(emailAddress),permissions(id,type,role)
```

## エラーレスポンス形式（Google自体が返すもの）

```json
{
  "error": {
    "code": 403,
    "message": "The user does not have sufficient permissions for this file.",
    "errors": [
      {
        "domain": "global",
        "reason": "insufficientFilePermissions",
        "message": "The user does not have sufficient permissions for this file."
      }
    ],
    "status": "PERMISSION_DENIED"
  }
}
```

`code` はHTTPステータスと同じ値。`errors[].reason` に機械可読な原因コードが入るため、リトライ可否の判定はこの `reason` を見て行う。

### 主なHTTPステータスと `reason`

| ステータス | 典型的な `reason` | 意味 |
|---|---|---|
| 400 Bad Request | `invalidParameter`, `badRequest` | クエリ構文不正、必須フィールド欠落 |
| 401 Unauthorized | `authError`, `required` | 未認証・アクセストークン期限切れ |
| 403 Forbidden | `insufficientFilePermissions` | 対象ファイルへの権限不足 |
| 403 Forbidden | `insufficientPermissions` | スコープ不足 |
| 403 Forbidden | `appNotAuthorizedToFile` | `drive.file` スコープで、アプリが作成/共有していないファイルにアクセスしようとした |
| 403 Forbidden | `storageQuotaExceeded` | ユーザーのDrive容量上限超過（アップロード時） |
| 403 Forbidden | `rateLimitExceeded` | プロジェクト単位のレート制限超過 |
| 403 Forbidden | `userRateLimitExceeded` | ユーザー単位のレート制限超過 |
| 403 Forbidden | `dailyLimitExceeded` | 1日あたりのクォータ超過 |
| 404 Not Found | `notFound` | ファイル/権限/変更トークンが存在しない、または権限不足で見えない（非公開リソースは403ではなく404を返すことがある） |
| 409 Conflict | - | 同時更新の競合（リビジョン競合等） |
| 429 Too Many Requests | - | 短時間の呼び出しすぎ（`rateLimitExceeded`/`userRateLimitExceeded` として403で返る場合と、素の429で返る場合がある） |
| 500/503 | `backendError`, `internalError` | Google側の一時的な問題。リトライ対象 |

## boidゲートウェイが返すエラー（ゲートウェイ経由の場合のみ）

**重要:** ゲートウェイが生成したエラーはGoogleの `{"error": {...}}` 形式ではなく、**プレーンテキストのボディ**（`Content-Type: text/plain`）で返る。レスポンスボディが上記のGoogle標準JSON形式でない場合、それはGoogleではなくゲートウェイが弾いたエラーだと判断できる。

| ステータス | ボディの典型例 | 原因 |
|---|---|---|
| 404 | `404 page not found` | リクエストパスが `/api/<job-token>/<service>/<tail>` の形に合っていない、または `.`/`..` を含むパストラバーサル的なパス |
| 401 | `unauthorized: invalid or expired job token` | job token自体が不明・失効（ジョブ終了後は無効化される） |
| 403 | `forbidden: service not permitted for this job token` | `<service>` を `config.yaml` の `services:` に定義しただけでは不十分。ジョブ/ワークスペース側でそのサービスを明示的に有効化していないと出る |
| 403 | `forbidden: read-only job token may only use GET/HEAD` | read-only jobからPOST/PATCH/PUT/DELETEなど書き込み系メソッドを呼んだ。ファイル作成・更新・削除・アップロード・権限変更などはread-only jobからは実行できない |
| 502 | `bad gateway: service X is not configured` | `<service>` という名前が `config.yaml` の `services:` に存在しない（サービス名の誤り） |
| 502 | `bad gateway: api gateway credential resolution failed...` | `secret_key` に対応するシークレットが未設定、またはOAuthトークンのリフレッシュ・サービスアカウントのトークン交換自体が失敗（fail-closed） |
| 502 | `bad gateway: upstream request failed for service X` | 実際のGoogleへの転送時にネットワーク的な失敗。メッセージからは実アップストリームのホスト名は分からないよう意図的に伏せられている |
| 503 | `service unavailable: api gateway has no secret resolver configured` | boidデーモン自体にシークレットストアが配線されていない（運用者側の設定不足） |

401/403がGoogle標準のJSON（`{"error":{...}}`）で返ってきた場合はGoogle側の権限・スコープ問題、プレーンテキストで返ってきた場合は上表のゲートウェイ側の問題として切り分けること。

## レート制限

- 429（またはGoogle標準形式の403+`rateLimitExceeded`/`userRateLimitExceeded`）を受け取った場合は指数バックオフで待機・リトライする。`Retry-After` ヘッダが付与されていればそれを尊重する
- 具体的な制限値（クエリ/100秒あたりの回数等）はGoogle Cloud Consoleのプロジェクト設定・APIごとのデフォルトクォータに依存し変動しうるため、コード側にハードコードしない。エラーの `reason` ベースの動的なバックオフ実装にする
- アップロード系（特にresumable）は失敗時、同じセッションURIに対して再開すればよく、最初からやり直す必要はない（詳細は [files.md](files.md)）
- boidゲートウェイ経由の場合、Google自体のレート制限がそのまま透過するのが基本（ゲートウェイ自体に独自のレート制限機構があるとは限らない）。502/503が返ってきた場合はレート制限ではなく上表のゲートウェイ側の問題を疑う

## 共通クエリパラメータ

| パラメータ | 説明 |
|---|---|
| `fields` | 返却フィールドの絞り込み（本ファイル上部参照） |
| `q` | フィルタクエリ（[files.md](files.md) 参照、`files.list` のみ） |
| `pageSize` | 1ページの件数 |
| `pageToken` | ページング用トークン |
| `supportsAllDrives` | 共有ドライブ上のリソースを対象に含めるか |
| `includeItemsFromAllDrives` | 一覧結果に共有ドライブのアイテムを含めるか（`files.list` のみ、`supportsAllDrives` と併用） |
