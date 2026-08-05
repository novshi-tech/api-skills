# 案件（Project）/ 案件原価（Project Cost）

案件はboardの中心リソース。見積〜受注〜納品〜請求の一連の書類（[invoices-and-documents.md](invoices-and-documents.md) 参照）は案件に紐づいて生成される。

## 案件（projects）

### 案件リスト取得 `GET /projects`

| パラメータ | 説明 |
|---|---|
| `name_cont` | 案件名の部分一致 |
| `client_id_eq` | 顧客IDの完全一致 |
| `client_name_cont` / `client_name_disp_cont` | 顧客名・顧客略称名の部分一致 |
| `order_status_in[]` | 受注ステータスでのIN検索（下記の受注ステータス値。複数の場合はカンマ区切り、例: `4,5`） |
| `delivery_status_in[]` | 進捗状況でのIN検索（`1`:未着手 `2`:着手中 `3`:納品済 `4`:検収済。複数の場合はカンマ区切り） |
| `project_no_eq` | 案件No（boardが採番する管理番号）の完全一致 |
| `management_no_eq` | 管理番号（自社独自の案件コード）の完全一致 |
| `delivery_date_gteq` / `delivery_date_lteq` | 納品日の範囲 |
| `invoice_date_gteq` / `invoice_date_lteq` | 請求日の範囲 |
| `invoice_timing_kbn_in[]` | 請求タイミング区分でのIN検索 |
| `tags[]` | タグでの絞り込み |
| `created_at_gteq` / `created_at_lteq` | 作成日時の範囲 |
| `updated_at_gteq` / `updated_at_lteq` | 更新日時の範囲 |
| `include_lost_flg` | 失注案件も含める（デフォルト除外） |
| `include_archive_flg` | アーカイブ済みも含める（デフォルト除外） |
| `per_page` / `page` | ページネーション |
| `response_group` | 返却項目の絞り込み。`small`（既定）/`medium`/`large`/`estimate`/`order`/`delivery`/`invoice`/`receipt`/`project_cost`/`all` から選択（[pagination-and-errors.md](pagination-and-errors.md)） |

```bash
curl --cacert "$BOID_API_CA_FILE" \
  "$BOID_API_BASE/board-api/projects?order_status_in[]=4,5&updated_at_gteq=2025-01-01&response_group=medium"
```

**`_in[]` パラメータはクエリ名を繰り返すのではなく、値をカンマ区切りにする。** `order_status_in[]=4&order_status_in[]=5` ではなく `order_status_in[]=4,5`（URLエンコード要。詳細は [pagination-and-errors.md](pagination-and-errors.md)）。

**同時リクエスト数制限の対象。** `project_no_eq` を指定しない一覧取得は、同時未返却リクエストが5以上で `429` になる（[pagination-and-errors.md](pagination-and-errors.md)）。

### 案件取得 `GET /projects/{id}`

`response_group` クエリ対応（一覧取得と同じ選択肢）。

### 案件登録 `POST /projects`

`client_id`（顧客ID）等を含むJSONボディで登録。成功時 `201`。

### 案件更新 `PATCH /projects/{id}`

部分更新。**受注ステータス自体はこのエンドポイントでは変更できない**（下記の専用エンドポイントを使う）。

### 案件削除 `DELETE /projects/{id}`

紐づく書類・原価・発注等が存在する場合は業務仕様上削除できないことがある。

### 受注ステータス変更 `PATCH /projects/order_status/{id}`

案件の受注ステータスだけを変更する専用エンドポイント。ボディの `order_status`（必須、`integer`）にステータス値を指定する。

| 値 | 意味 |
|---|---|
| 1 | 見積中(高) |
| 2 | 見積中(中) |
| 3 | 見積中(低) |
| 8 | 見積中(除) |
| 4 | 受注確定 |
| 5 | 受注済 |
| 9 | 失注 |

```bash
curl -X PATCH --cacert "$BOID_API_CA_FILE" \
  -H "Content-Type: application/json" \
  -d '{"order_status":4}' \
  "$BOID_API_BASE/board-api/projects/order_status/123456"
```

### 案件のロック `PATCH /projects/lock_flg/{id}`

案件を編集ロックする/解除する専用エンドポイント。ロック中は画面・API双方から通常の更新ができなくなる想定（board標準の「ロック」機能。詳細な業務仕様はboardヘルプ参照）。

## 案件原価（project_costs）

案件に紐づく原価（外注費・材料費等）情報。発注（expenditures、[expenditures.md](expenditures.md)）とは別に、案件側で直接原価を記録できるリソース。

### 案件原価リスト取得 `GET /project_costs`

| パラメータ | 説明 |
|---|---|
| `project_id_eq` | 案件IDの完全一致 |
| `created_at_gteq` / `created_at_lteq` | 作成日時の範囲 |
| `invoice_date_gteq` / `invoice_date_lteq` | 請求日の範囲 |
| `payment_date_gteq` / `payment_date_lteq` | 支払日の範囲 |
| `per_page` / `page` | ページネーション |

`response_group` クエリはない。**同時リクエスト数制限の対象**（`project_id_eq` を指定しない一覧取得）。

### 案件原価取得 `GET /project_costs/{id}`

### 案件原価登録 `POST /project_costs`

`project_id`・`description`（内容）・`cost`（金額）等を含むJSONボディで登録。

### 案件原価更新 `PATCH /project_costs/{id}`

### 案件原価削除 `DELETE /project_costs/{id}`

## エンドポイント一覧

| メソッド | パス | 概要 |
|---|---|---|
| GET | `/projects` | 案件リスト取得 |
| POST | `/projects` | 案件登録 |
| GET | `/projects/{id}` | 案件取得 |
| PATCH | `/projects/{id}` | 案件更新 |
| DELETE | `/projects/{id}` | 案件削除 |
| PATCH | `/projects/order_status/{id}` | 受注ステータス変更 |
| PATCH | `/projects/lock_flg/{id}` | 案件のロック |
| GET | `/project_costs` | 案件原価リスト取得 |
| POST | `/project_costs` | 案件原価登録 |
| GET | `/project_costs/{id}` | 案件原価取得 |
| PATCH | `/project_costs/{id}` | 案件原価更新 |
| DELETE | `/project_costs/{id}` | 案件原価削除 |
