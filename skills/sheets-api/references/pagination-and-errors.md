# 大規模データの分割取得 / エラーレスポンス形式 / クォータ / fields パラメータ

## Bitbucketのような「ページネーション」はない

Google Sheets APIには、`next` カーソルや `page`/`pagelen` のような一覧系ページネーション機構はない。`values.get`/`batchGet` は指定したA1範囲（`Sheet1!A1:D5` 等）に含まれるセルを一度にすべて返す設計になっている。

大きなシート（数万行など）を扱う場合は、以下のいずれかの方針で自前に分割する。

- **range自体を分割する**: `A1:Z1000`, `A1001:Z2000` のように行範囲を区切って複数回 `values.get` を呼ぶ（または `values.batchGet` に複数rangeをまとめて渡し1回のHTTPリクエストで済ませる）
- **列全体/シート全体を指定して1回で取る**: `Sheet1` や `A:Z` のように広めに指定し、レスポンス側で末尾の空行・空列が自動省略されることを利用する（ただしレスポンスサイズが大きくなりすぎる場合は行範囲の分割を検討する）
- 現在のデータ量が事前に分からない場合は、まず `spreadsheets.get`（`fields=sheets.properties.gridProperties` 程度に絞る）でシートの `rowCount`/`columnCount` を取得してから、それに応じてrangeを組み立てるとよい

いずれの方式でも、Bitbucketの `next` リンクのように「レスポンスに次ページへのURLが埋め込まれている」ということはないため、`values.md` に書いた `range`/`majorDimension` の指定だけで完結する。boidゲートウェイ経由でも、絶対URLの付け替えのような処理は不要（`range` 自体を都度組み立てて `$BOID_API_BASE/sheets-api/...` に対してリクエストするだけでよい）。

### `spreadsheetUrl` はboidサンドボックスから直接到達できない

Sheets APIのレスポンス（`spreadsheets.create`/`get` 等）には `spreadsheetUrl`（`https://docs.google.com/spreadsheets/d/{spreadsheetId}/edit` 形式のURL）が含まれることがある。これはBitbucketの `next` と同様、**Googleの実ホスト（`docs.google.com`）を指す絶対URLであり、boidサンドボックス内からこのURLへ直接アクセスすることはできない**（サンドボックスの外向き通信はboidゲートウェイ経由に限定されており、`docs.google.com` はSheets/Drive等のAPIホストとは別ドメインでゲートウェイの転送対象でもない）。`spreadsheetUrl` はあくまで「人間がブラウザで開くためのリンク」として扱い、ユーザーへの提示（チャット出力やコミットメッセージ等）に使うだけにとどめ、ジョブ内のコードからこのURLに対してHTTPリクエストを送るような使い方はしないこと。スプレッドシートの内容取得・操作は引き続き `$BOID_API_BASE/sheets-api/v4/spreadsheets/{spreadsheetId}/...` 経由で行う。

## fields パラメータでの部分レスポンス

`spreadsheets.get` や `batchUpdate` のレスポンスは、シート全体の書式・条件付き書式・フィルタ等まで含めると非常に大きくなりうる。`fields` クエリパラメータ（Google API共通のフィールドマスク構文）で必要な部分だけに絞り込める。

```
GET {BASE_URL}/{spreadsheetId}?fields=spreadsheetId,properties.title,sheets.properties
```

- カンマ区切りで複数フィールドを指定
- `.` でネストしたフィールドを指定（例: `sheets.properties.sheetId`）
- 配列フィールドに対する指定は配列の各要素に適用される（`sheets.properties` は「各シートのproperties」を意味する）
- `values.get` 自体は元々レスポンスが軽い（指定範囲の値のみ）ため `fields` の効果は薄いが、レスポンスに多くの補助情報を含む `batchUpdate`（`includeSpreadsheetInResponse=true` 時）では帯域節約に有効
- **`fields` と `spreadsheets.get` の `includeGridData` は併用できない。** `fields`（フィールドマスク）を指定すると `includeGridData` パラメータは無視される（詳細は [spreadsheets-and-sheets.md](spreadsheets-and-sheets.md) 参照）。グリッドデータを絞り込みたい場合は `includeGridData=true` を使わず、`fields=sheets.data` のようにフィールドマスク側でグリッドデータを明示的に指定する

## エラーレスポンス形式（Google Sheets API自体が返すもの）

Google APIs共通のエラー形式（`google.rpc.Status` 由来）。

```json
{
  "error": {
    "code": 400,
    "message": "Invalid range: Sheet3!A1:B2",
    "status": "INVALID_ARGUMENT",
    "errors": [
      {
        "message": "Invalid range: Sheet3!A1:B2",
        "domain": "global",
        "reason": "badRequest"
      }
    ]
  }
}
```

- `error.code` — HTTPステータスコードと同じ数値
- `error.status` — gRPC由来のステータス名（`INVALID_ARGUMENT`, `PERMISSION_DENIED`, `NOT_FOUND`, `RESOURCE_EXHAUSTED`, `UNAUTHENTICATED` 等）。HTTPステータスコードだけでなくこの文字列でも原因を判別できる
- `error.errors[]` — 旧来のDiscovery API形式のエラー詳細（`reason` に `badRequest`, `notFound` 等が入る）。新しいAPIレスポンスでは省略されることもある

### 主なHTTPステータス（Google Sheets API自体）

| ステータス | `status` | 意味 | 典型的なケース |
|---|---|---|---|
| 400 Bad Request | `INVALID_ARGUMENT` | リクエスト形式不正 | 存在しないシート名を含むrange、不正なA1記法、`valueInputOption` 未指定 |
| 401 Unauthorized | `UNAUTHENTICATED` | 未認証・トークン無効 | アクセストークン未指定/期限切れ |
| 403 Forbidden | `PERMISSION_DENIED` | 権限不足 | 対象スプレッドシートが認証主体（サービスアカウント/ユーザー）に共有されていない、スコープ不足 |
| 404 Not Found | `NOT_FOUND` | リソースが存在しない | `spreadsheetId` の誤り、削除済みスプレッドシート |
| 429 Too Many Requests | `RESOURCE_EXHAUSTED` | クォータ超過 | 後述 |
| 500系 | `INTERNAL` / `UNAVAILABLE` | Google側の一時的な問題 | リトライ対象（指数バックオフ推奨） |

`403`は「認証は通っているがそのスプレッドシートへのアクセス権がない」ケースが大半で、Bitbucketの非公開リポジトリのように404にフォールバックすることはなく、素直に403として返る。

## boidゲートウェイが返すエラー（ゲートウェイ経由の場合のみ）

**重要:** ゲートウェイが生成したエラーはGoogleの `{"error": {...}}` 形式ではなく、**プレーンテキストのボディ**（`Content-Type: text/plain`）で返る。レスポンスボディが上記のGoogle標準JSON形式（`error.code`/`error.status` を持つ）でない場合、それはGoogleではなくゲートウェイが弾いたエラーだと判断できる。

| ステータス | ボディの典型例 | 原因 |
|---|---|---|
| 404 | `404 page not found` | リクエストパスが `/api/<job-token>/<service>/<tail>` の形に合っていない、または `.`/`..` を含むパストラバーサル的なパス |
| 401 | `unauthorized: invalid or expired job token` | job token自体が不明・失効（ジョブ終了後は無効化される） |
| 403 | `forbidden: service not permitted for this job token` | `<service>` を `config.yaml` の `services:` に定義しただけでは不十分。ジョブ/ワークスペース側でそのサービスを明示的に有効化していないと出る |
| 403 | `forbidden: read-only job token may only use GET/HEAD` | read-only jobからPOST/PUT/DELETEなど書き込み系メソッドを呼んだ。スプレッドシート作成、`values.update`/`append`/`clear`、`batchUpdate` などはread-only jobからは実行できない |
| 502 | `bad gateway: service X is not configured` | `<service>` という名前が `config.yaml` の `services:` に存在しない（サービス名の誤り） |
| 502 | `bad gateway: api gateway credential resolution failed...` | `secret_key` に対応するシークレットが未設定、またはサービスアカウント鍵・OAuthトークンの解決自体が失敗（fail-closed） |
| 502 | `bad gateway: upstream request failed for service X` | 実際のGoogleへの転送時にネットワーク的な失敗。メッセージからは実アップストリームのホスト名は分からないよう意図的に伏せられている |
| 503 | `service unavailable: api gateway has no secret resolver configured` | boidデーモン自体にシークレットストアが配線されていない（運用者側の設定不足） |

401/403がGoogle標準のJSON（`error.code`/`error.status` を持つ）で返ってきた場合はGoogle側の認証・権限問題、プレーンテキストで返ってきた場合は上表のゲートウェイ側の問題として切り分けること。

## クォータ / レート制限

Google Sheets APIのクォータは**読み取りと書き込みで別枠**として管理される。

| 項目 | 上限 |
|---|---|
| 読み取りリクエスト（プロジェクト全体） | 300 リクエスト / 分 |
| 読み取りリクエスト（ユーザー・プロジェクトごと） | 60 リクエスト / 分 |
| 書き込みリクエスト（プロジェクト全体） | 300 リクエスト / 分 |
| 書き込みリクエスト（ユーザー・プロジェクトごと） | 60 リクエスト / 分 |

- ここでいう「ユーザー」は、OAuthユーザー代理の場合は認証したエンドユーザー、サービスアカウントの場合はそのサービスアカウント自身を指す
- `values.get` は読み取り、`values.update`/`append`/`clear`/`batchUpdate`（値・構造どちらの `batchUpdate` も）は書き込みとしてカウントされる
- **日次（1日あたり）の上限は存在しない。** 上記の分単位のクォータ内に収まっていれば、1日に発行できるリクエスト数そのものに上限はない
- **推奨ペイロードサイズは最大2MB。** これを超えるとリクエスト速度が低下する（エラーになるとは限らないが、大量データの一括書き込みは分割してこの目安に収めることが推奨される）
- **1リクエストあたりの処理時間には180秒の上限がある。** これを超える処理時間がかかるリクエスト（非常に大きい範囲への一括操作等）はタイムアウトエラーになる
- 429を受け取った場合は公式に**指数バックオフ**が推奨されている（`min((2^n + random_ms), max_backoff)`。`max_backoff` は用途に応じて32〜64秒程度に設定し、到達後はその間隔で維持する）。固定間隔でのリトライは避ける
- クォータ超過は通常429 `RESOURCE_EXHAUSTED`として返るが、**403 `rateLimitExceeded`/`userRateLimitExceeded`（旧来のDiscovery API系エラー形式）で返ってくる場合もある**ため、429だけでなく403のエラー内容（`error.errors[].reason`）もクォータ関連かどうか確認するとよい
- **GoogleはSheets APIのレート制限時に `Retry-After` ヘッダを返さない。** Bitbucket API（[bitbucket-api](../../bitbucket-api/references/pagination-and-errors.md)参照）のように `Retry-After` を尊重して待機時間を決める、という実装はSheets APIには使えない。上記の指数バックオフをクライアント側で自前に実装する必要がある
- `values.batchGet`/`batchUpdate`/`batchClear` で複数範囲・複数シートをまとめることで、個別に `values.get`/`update` を繰り返すより実際のHTTPリクエスト数（＝クォータ消費）を大きく削減できる。大量のセル操作を行う場合は可能な限りbatch系エンドポイントに寄せることを推奨
- クォータの具体的な数値は将来Google側の裁量で変更されうるため、コード側にハードコードした前提を持たせず、429ベースの動的なバックオフ実装にする
- boidゲートウェイ経由の場合、Google自体のクォータがそのまま透過するのが基本（ゲートウェイ自体に独自のレート制限機構があるとは限らない）。502/503が返ってきた場合はクォータ超過ではなく上表のゲートウェイ側の問題を疑う（クォータ超過は429（まれに403）として透過される）

## 共通クエリパラメータ

Google APIs共通で使えるクエリパラメータ（Sheets API固有ではなく、Google APIs全般の慣習）。

| パラメータ | 説明 |
|---|---|
| `fields` | 部分レスポンス用のフィールドマスク（上記「fields パラメータでの部分レスポンス」参照） |
| `quotaUser` | クォータ集計の単位を任意の文字列で指定する。サービスアカウントを複数の論理的な「ユーザー」に見せかけてクォータを分離したい場合などに使う（エンドユーザーのIPアドレスやセッションIDを渡すのが典型） |
| `prettyPrint` | `true`（デフォルト）でレスポンスJSONを整形して返す。帯域を節約したい場合は `false` を指定する |
| `alt` | レスポンス形式。通常は `json`（デフォルト）のみを使えばよい |
| `key` | APIキー単体での認証時に使う（[authentication.md](authentication.md) の「APIキーのみでの利用」参照）。OAuth/サービスアカウント認証時は不要 |
