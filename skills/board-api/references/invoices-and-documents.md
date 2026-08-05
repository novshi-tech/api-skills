# 請求（Invoice）/ 案件の書類（Project Document）

売上側（顧客に対する）の請求管理と、案件に紐づく各種帳票（見積書・発注書・納品書・請求書・領収書）を扱う。

**書類系（見積書/発注書/納品書/請求書/領収書）に作成・削除APIはない。** これらは案件の受注確定・納品登録・請求確定といった業務フローに応じてboard側で自動生成される前提で、APIからは「取得」「更新（PATCH）」「ロック」のみ可能。

## 請求（invoices）

案件に対する請求そのもの（1案件に対して分割請求される場合は複数の請求レコードが存在しうる）を横断的に一覧・ステータス変更するリソース。請求書という「帳票」自体は下記の `documents/invoices` が扱う。

### 請求リスト取得 `GET /invoices`

| パラメータ | 説明 |
|---|---|
| `invoice_date_gteq` / `invoice_date_lteq` | 請求日の範囲 |
| `invoice_payment_limit_date_gteq` / `invoice_payment_limit_date_lteq` | 支払期限の範囲 |
| `project_order_status_in[]` | 紐づく案件の受注ステータスでのIN検索（値は[projects-and-costs.md](projects-and-costs.md)の受注ステータス表。複数はカンマ区切り） |
| `invoice_status_in[]` | 請求ステータスでのIN検索（下記の請求ステータス値。複数はカンマ区切り） |
| `project_project_no_eq` | 紐づく案件の案件Noの完全一致 |
| `project_management_no_eq` | 紐づく案件の管理番号の完全一致 |
| `project_name_cont` | 紐づく案件名の部分一致 |
| `project_client_id_eq` | 紐づく顧客IDの完全一致 |
| `project_client_name_cont` / `project_client_name_disp_cont` | 紐づく顧客名・略称の部分一致 |
| `project_tags[]` | 紐づく案件のタグでの絞り込み |
| `updated_at_gteq` / `updated_at_lteq` | 更新日時の範囲 |
| `per_page` / `page` | ページネーション |
| `response_group` | 返却項目の絞り込み。`small`（既定）/`medium`/`large`/`estimate`/`order`/`delivery`/`invoice`/`receipt`/`project_cost`/`all` から選択（[pagination-and-errors.md](pagination-and-errors.md)） |

**同時リクエスト数制限の対象**（`project_project_no_eq` を指定しない一覧取得、[pagination-and-errors.md](pagination-and-errors.md)）。

このリソースには `POST`/`PATCH /invoices/{id}`/`DELETE` が存在しない。**更新できるのはステータスのみ。**

### 請求ステータス変更 `PATCH /invoices/invoice_status/{id}`

ボディの `invoice_status`（必須、`integer`）にステータス値を指定する。

| 値 | 意味 |
|---|---|
| 1 | 未請求 |
| 4 | 請求OK |
| 2 | 請求済 |
| 5 | 一部入金済 |
| 3 | 入金済 |
| 9 | 回収不能 |

```bash
curl -X PATCH --cacert "$BOID_API_CA_FILE" \
  -H "Content-Type: application/json" \
  -d '{"invoice_status":3}' \
  "$BOID_API_BASE/board-api/invoices/invoice_status/123456"
```

## 案件の書類（project_documents: 見積書/発注書/納品書/請求書/領収書）

パスは `/documents/{種別}/{id}` の形。すべて「取得」「更新（PATCH）」「ロック」の3操作のみで、リストAPI・作成API・削除APIはない（＝どの書類が存在するかは案件側（`GET /projects/{id}`、`response_group` によっては書類IDが含まれる）から辿る必要がある）。

| 書類種別 | 取得 | 更新 | ロック |
|---|---|---|---|
| 見積書（estimates） | `GET /documents/estimates/{id}` | `PATCH /documents/estimates/{id}` | `PATCH /documents/estimates/lock_flg/{id}` |
| 発注書（orders、顧客に対して自社が発行する受注書の意） | `GET /documents/orders/{id}` | `PATCH /documents/orders/{id}` | `PATCH /documents/orders/lock_flg/{id}` |
| 納品書（deliveries） | `GET /documents/deliveries/{id}` | `PATCH /documents/deliveries/{id}` | `PATCH /documents/deliveries/lock_flg/{id}` |
| 請求書（invoices、帳票としての請求書。上記の `invoices` リソースとは別物） | `GET /documents/invoices/{id}` | `PATCH /documents/invoices/{id}` | `PATCH /documents/invoices/lock_flg/{id}` |
| 領収書（receipts） | `GET /documents/receipts/{id}` | `PATCH /documents/receipts/{id}` | `PATCH /documents/receipts/lock_flg/{id}` |

```bash
# 見積書取得
curl --cacert "$BOID_API_CA_FILE" \
  "$BOID_API_BASE/board-api/documents/estimates/789012"

# 見積書更新（明細・備考等）
curl -X PATCH --cacert "$BOID_API_CA_FILE" \
  -H "Content-Type: application/json" \
  -d '{"remarks":"納期は別途ご相談ください"}' \
  "$BOID_API_BASE/board-api/documents/estimates/789012"

# 見積書ロック（lock_flgはinteger。0:未ロック 1:ロック済み。booleanではない）
curl -X PATCH --cacert "$BOID_API_CA_FILE" \
  -H "Content-Type: application/json" \
  -d '{"lock_flg":1}' \
  "$BOID_API_BASE/board-api/documents/estimates/lock_flg/789012"
```

**`/documents/invoices/{id}`（帳票としての請求書）と `/invoices`（請求ステータス管理リソース）は別物。** 前者は特定の1通の請求書帳票（PDF相当の元データ）を指し、後者は案件に対する請求サイクル・入金状況を横断管理するリソース。混同しないこと。

## エンドポイント一覧

| メソッド | パス | 概要 |
|---|---|---|
| GET | `/invoices` | 請求リスト取得 |
| PATCH | `/invoices/invoice_status/{id}` | 請求ステータス変更 |
| GET | `/documents/estimates/{id}` | 見積書取得 |
| PATCH | `/documents/estimates/{id}` | 見積書更新 |
| PATCH | `/documents/estimates/lock_flg/{id}` | 見積書ロック |
| GET | `/documents/orders/{id}` | 発注書取得 |
| PATCH | `/documents/orders/{id}` | 発注書更新 |
| PATCH | `/documents/orders/lock_flg/{id}` | 発注書ロック |
| GET | `/documents/deliveries/{id}` | 納品書取得 |
| PATCH | `/documents/deliveries/{id}` | 納品書更新 |
| PATCH | `/documents/deliveries/lock_flg/{id}` | 納品書ロック |
| GET | `/documents/invoices/{id}` | 請求書取得 |
| PATCH | `/documents/invoices/{id}` | 請求書更新 |
| PATCH | `/documents/invoices/lock_flg/{id}` | 請求書ロック |
| GET | `/documents/receipts/{id}` | 領収書取得 |
| PATCH | `/documents/receipts/{id}` | 領収書更新 |
| PATCH | `/documents/receipts/lock_flg/{id}` | 領収書ロック |
