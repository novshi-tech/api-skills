# ページネーション / レスポンスグループ / エラー形式 / レート制限

## ページネーション

一覧系エンドポイント（`GET /clients` 等）はJSON配列をそのまま返す（Bitbucketのような `{values: [...], next: ...}` の封筒構造ではない）。ページ情報はレスポンス**ヘッダー**に載る。

```
X-Total-Count: 245
X-Page: 1
X-Per-Page: 50
```

リクエスト側は `per_page` / `page` クエリパラメータで制御する。

```bash
curl --cacert "$BOID_API_CA_FILE" \
  "$BOID_API_BASE/board-api/clients?per_page=50&page=2"
```

- **`per_page` は全リスト取得APIで共通、デフォルトは `10`、上限は `100`。** デフォルトのまま呼ぶと10件しか返らず「全件取れていない」事故になりやすいので、必要な件数を明示的に指定する
- `X-Total-Count` を `per_page` で割って総ページ数を算出し、`page` をインクリメントして辿る
- レスポンスボディ自体はページ番号や総件数を含まない配列のみなので、**ページネーション情報は必ずヘッダーから読む**こと（ボディだけパースする実装だと総件数が分からない）

### 実装パターン（擬似コード、boidゲートウェイ経由）

```python
import math

base = f"{os.environ['BOID_API_BASE']}/board-api"
per_page = 50
page = 1
results = []
while True:
    resp = http_get(f"{base}/clients?per_page={per_page}&page={page}",
                     cacert=os.environ.get("BOID_API_CA_FILE"))  # x-api-key/Authorizationヘッダは付けない
    results.extend(resp.json())
    total = int(resp.headers.get("X-Total-Count", "0"))
    if page * per_page >= total:
        break
    page += 1
```

**リスト取得系の同時実行に注意。** 「案件No」「発注No」を指定しない一覧取得（`projects`/`invoices`/`project_costs`/`expenditures`/`expenditure_payments`）は、未返却の同時リクエストが5以上になると `429 Too Many Requests` を返す（後述のレート制限とは別枠のガード）。ページを並列に取得する実装は避け、上記のように逐次処理にすること。

## レスポンスグループ（`response_group`）

一覧・詳細取得の一部のエンドポイントは、クエリパラメータに `response_group` を持つ。**クエリパラメータ自体を省略した場合のデフォルトは常に `small`** で、返却されるのは最小限の項目のみ。APIドキュメント上は各エンドポイントで全項目が記載されているため、「ドキュメント上のフィールドが返ってこない」と感じたら、まず `response_group` を明示指定しているか確認すること。

```bash
curl --cacert "$BOID_API_CA_FILE" \
  "$BOID_API_BASE/board-api/projects?response_group=medium"
```

対応状況と選択肢はエンドポイントごとに異なる:

| エンドポイント | 選択肢 |
|---|---|
| `GET /clients`, `GET /payees` | `small`（既定）/ `large` |
| `GET /projects`, `GET /projects/{id}`, `GET /invoices`, `GET /expenditure_payments` | `small`（既定）/ `medium` / `large` / `estimate` / `order` / `delivery` / `invoice` / `receipt` / `project_cost` / `all` |
| `GET /expenditures`, `GET /expenditures/{id}` | `small`（既定）/ `medium` / `large` / `estimate` / `order` / `delivery` / `invoice` / `all`（`receipt`/`project_cost` は無い） |

- `medium` は `small` の上位互換（`small` の全項目 + 追加項目）、`large` は書類系を除く全項目
- `estimate`/`order`/`delivery`/`invoice`/`receipt`/`project_cost` は「`small` + 該当書類（または案件原価）の情報」だけを追加で含める省帯域オプション。特定の書類だけ欲しい場合に有効
- `all` は `large` + すべての書類情報
- 上記以外のエンドポイント（`client_branches`/`contacts`/`payee_branches`/`payee_contacts`/`project_costs`/`analyses`/`clients・payeesの詳細取得`/マスタ系）には `response_group` クエリ自体が存在しない

## エラーレスポンス形式（board自体が返すもの）

シンプルな `message` のみの形と、バリデーションエラー時の `errors[]` 付きの形の2パターンがある。

```json
{ "message": "APIキーまたはAPIトークンが無効です。" }
```

```json
{
  "message": "パラメータが正しくありません。",
  "errors": [
    { "field": "name", "code": "blank", "description": "顧客名を入力してください。" }
  ]
}
```

- `errors[].field` — バリデーションエラーが発生したフィールド名
- `errors[].code` — エラー種別を表す短いコード（例: `blank`）
- `errors[].description` — 人間向けの日本語エラーメッセージ

### 主なHTTPステータス（board自体）

| ステータス | 意味 | 典型的なケース |
|---|---|---|
| 200 OK | 成功（GET/PATCH） | |
| 201 Created | 成功（POST） | |
| 204 No Content | 成功（本文なしのDELETE等） | |
| 400 Bad Request | リクエスト形式不正 | |
| 401 Unauthorized | APIキー・APIトークンが未指定/無効 | |
| 403 Forbidden | APIトークンの権限不足 | |
| 404 Not Found | リソースが存在しない | `id` の誤り |
| 415 Unsupported Media Type | `Content-Type: application/json` が未指定/不正 | POST/PATCHで発生 |
| 422 Unprocessable Entity | バリデーションエラー | 必須項目欠落、形式不正。`errors[]` を参照 |
| 429 Too Many Requests | レート制限超過 | 後述 |
| 500 Internal Server Error | boardサーバー側の問題、またはレスポンスサイズ超過 | レスポンスが10MBを超える場合も500になり `{"message":"Internal server error"}` のみが返る |
| 503 Service Unavailable | メンテナンス中、または障害時 | |

## boidゲートウェイが返すエラー（ゲートウェイ経由の場合のみ）

**重要:** ゲートウェイが生成したエラーはboardの上記JSON形式ではなく、**プレーンテキストのボディ**（`Content-Type: text/plain`）で返る。レスポンスボディが上記のboard標準JSON形式でない場合、それはboardではなくゲートウェイが弾いたエラーだと判断できる。

| ステータス | ボディの典型例 | 原因 |
|---|---|---|
| 404 | `404 page not found` | リクエストパスが `/api/<job-token>/<service>/<tail>` の形に合っていない、または `.`/`..` を含むパストラバーサル的なパス |
| 401 | `unauthorized: invalid or expired job token` | job token自体が不明・失効（ジョブ終了後は無効化される） |
| 403 | `forbidden: service not permitted for this job token` | `<service>` を `config.yaml` の `services:` に定義しただけでは不十分。ジョブ/ワークスペース側でそのサービスを明示的に有効化していないと出る |
| 403 | `forbidden: read-only job token may only use GET/HEAD` | read-only jobからPOST/PATCH/DELETEなど書き込み系メソッドを呼んだ |
| 502 | `bad gateway: service X is not configured` | `<service>` という名前が `config.yaml` の `services:` に存在しない（サービス名の誤り） |
| 502 | `bad gateway: api gateway credential resolution failed...` | `secret_key` に対応するシークレットが未設定、またはシークレット解決自体が失敗（fail-closed）。board APIは`x-api-key`/`Authorization`の2ヘッダーを要求するため、片方しか設定されていない構成だとこのエラー、またはboard側の401になる可能性がある |
| 502 | `bad gateway: upstream request failed for service X` | 実際のboardへの転送時にネットワーク的な失敗。メッセージからは実アップストリームのホスト名は分からないよう意図的に伏せられている |
| 503 | `service unavailable: api gateway has no secret resolver configured` | boidデーモン自体にシークレットストアが配線されていない（運用者側の設定不足） |

401/403がboard標準のJSONで返ってきた場合はboard側の権限問題、プレーンテキストで返ってきた場合は上表のゲートウェイ側の問題として切り分けること。

## レート制限

board APIは以下の制限を持つ:

- **3000リクエスト / 1日**（リセットはJSTではなくUTC基準）
- **3リクエスト / 秒**
- **リスト取得系APIの同時リクエスト数は4まで**（「案件No」「発注No」を指定しない `projects`/`invoices`/`project_costs`/`expenditures`/`expenditure_payments` の一覧のみが対象。同時実行が5以上で429）

429を受け取った場合、レスポンスボディの `message` で原因を切り分けられる:

| `message` | 原因 |
|---|---|
| `Too Many Requests` | 秒間リクエスト数の上限超過 |
| `Limit Exceeded` | 1日あたりのリクエスト数の上限超過 |
| `リスト取得系APIの同時リクエスト数の上限を超えています` | 同時実行数の上限超過 |

上限緩和には対応していない。日次上限を頻繁に超える場合はポーリング間隔を見直す、リクエスト数自体を減らす（`updated_at_gteq` 等の差分取得フィルタを使う）などクライアント側での対処が必要。

秒間制限（3req/秒）に対しては、クライアント側で意図的にスロットリングする実装を推奨する（例: 1リクエストごとに最低 `1/3` 秒空ける）。

## 共通クエリパラメータの命名規則

一覧系エンドポイントのフィルタ用クエリパラメータは、boardの内部実装（Ransack風）に由来する命名規則で統一されている。

| サフィックス | 意味 | 例 |
|---|---|---|
| `_eq` | 完全一致 | `client_id_eq=123` |
| `_cont` | 部分一致（LIKE） | `name_cont=サンプル` |
| `_gteq` | 以上（Greater Than or Equal） | `updated_at_gteq=2025-01-01` |
| `_lteq` | 以下（Less Than or Equal） | `updated_at_lteq=2025-12-31` |
| `_in[]` | IN検索（複数値） | `order_status_in[]=4,5` |

- **`_in[]` はパラメータ名を繰り返すのではなく、値をカンマ区切りにする。** `order_status_in[]=4&order_status_in[]=5` ではなく `order_status_in[]=4,5`（URLエンコード要）。`tags[]` も同様にカンマ区切り
- `include_archive_flg` / `include_lost_flg` / `include_not_ordered_flg` / `include_auto_renewal_flg` のような `include_*_flg` パラメータは `integer`型（`0`:除く/既定、`1`:含める）で、`true`/`false` の文字列では通らない。デフォルトでは除外される対象（アーカイブ済み・失注・見送り等）を含める指定
- ステータス系のIN検索（`order_status_in[]`/`expenditure_status_in[]`/`invoice_status_in[]`/`payment_status_in[]`等）の値は、`response_group`同様「日本語ラベル」ではなく**整数コード**。各リソースのreferenceファイルに掲載の対応表を参照すること
- 日付は `YYYY-MM-DD` 形式、日時は基本ISO 8601
