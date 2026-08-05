# 発注（Expenditure）/ 支払（Expenditure Payment）/ 発注の書類（Expenditure Document）

仕入・外注側（発注先に対する）の発注管理。構造は [invoices-and-documents.md](invoices-and-documents.md) の売上側（案件・請求・案件の書類）とほぼ対称。

## 発注（expenditures）

### 発注リスト取得 `GET /expenditures`

| パラメータ | 説明 |
|---|---|
| `name_cont` | 発注名の部分一致 |
| `payee_id_eq` | 発注先IDの完全一致 |
| `payee_name_cont` / `payee_name_disp_cont` | 発注先名・略称の部分一致 |
| `expenditure_status_in[]` | 発注ステータスでのIN検索（下記の発注ステータス値。複数はカンマ区切り） |
| `delivery_status_in[]` | 進捗状況でのIN検索（`1`:未着手 `2`:着手中 `3`:納品済 `4`:検収済。複数はカンマ区切り） |
| `expenditure_no_eq` | 発注No（boardが採番する管理番号）の完全一致 |
| `management_no_eq` | 管理番号（自社独自の発注コード）の完全一致 |
| `tags[]` | タグでの絞り込み |
| `created_at_gteq` / `created_at_lteq` | 作成日時の範囲 |
| `updated_at_gteq` / `updated_at_lteq` | 更新日時の範囲 |
| `include_not_ordered_flg` | 見送りステータスも含める（`0`:除く/既定 `1`:含む。「未発注」ではなく「見送り」扱いの発注を含めるかどうか） |
| `include_archive_flg` | アーカイブ済みも含める |
| `expenditure_order_order_date_gteq` / `expenditure_order_order_date_lteq` | 発注日の範囲 |
| `per_page` / `page` | ページネーション |
| `response_group` | 返却項目の絞り込み。`small`（既定）/`medium`/`large`/`estimate`/`order`/`delivery`/`invoice`/`all` から選択（`project_costs`同様に`project_cost`は無く、`receipt`も無い。[pagination-and-errors.md](pagination-and-errors.md)） |

**同時リクエスト数制限の対象**（`expenditure_no_eq` を指定しない一覧取得、[pagination-and-errors.md](pagination-and-errors.md)）。

### 発注取得 `GET /expenditures/{id}`

`response_group` クエリ対応（一覧取得と同じ選択肢）。

### 発注登録 `POST /expenditures`

`payee_id`（発注先ID）等を含むJSONボディで登録。

### 発注更新 `PATCH /expenditures/{id}`

部分更新。**発注ステータスはこのエンドポイントでは変更できない**（下記の専用エンドポイント）。

### 発注削除 `DELETE /expenditures/{id}`

### 発注ステータス変更 `PATCH /expenditures/expenditure_status/{id}`

ボディの `expenditure_status`（必須、`integer`）にステータス値を指定する。

| 値 | 意味 |
|---|---|
| 1 | 見積中(高) |
| 2 | 見積中(中) |
| 3 | 見積中(低) |
| 8 | 見積中(除) |
| 4 | 発注確定 |
| 5 | 発注済 |
| 9 | 見送り |

```bash
curl -X PATCH --cacert "$BOID_API_CA_FILE" \
  -H "Content-Type: application/json" \
  -d '{"expenditure_status":4}' \
  "$BOID_API_BASE/board-api/expenditures/expenditure_status/123456"
```

### 発注のロック `PATCH /expenditures/lock_flg/{id}`

## 支払（expenditure_payments）

発注先への支払サイクル・支払状況を横断管理するリソース。`invoices`（案件側の請求管理）と対称。

### 支払リスト取得 `GET /expenditure_payments`

| パラメータ | 説明 |
|---|---|
| `invoice_date_gteq` / `invoice_date_lteq` | 請求（先方からの請求）日の範囲 |
| `payment_date_gteq` / `payment_date_lteq` | 支払日の範囲 |
| `expenditure_expenditure_status_in[]` | 紐づく発注の発注ステータスでのIN検索（複数はカンマ区切り） |
| `payment_status_in[]` | 支払ステータスでのIN検索（下記の支払ステータス値。複数はカンマ区切り） |
| `expenditure_expenditure_no_eq` | 紐づく発注の発注Noの完全一致 |
| `expenditure_management_no_eq` | 紐づく発注の管理番号の完全一致 |
| `expenditure_name_cont` | 紐づく発注名の部分一致 |
| `expenditure_payee_id_eq` | 紐づく発注先IDの完全一致 |
| `expenditure_payee_name_cont` / `expenditure_payee_name_disp_cont` | 紐づく発注先名・略称の部分一致 |
| `expenditure_tags[]` | 紐づく発注のタグでの絞り込み |
| `updated_at_gteq` / `updated_at_lteq` | 更新日時の範囲 |
| `per_page` / `page` | ページネーション |
| `response_group` | 返却項目の絞り込み。`small`（既定）/`medium`/`large`/`estimate`/`order`/`delivery`/`invoice`/`all` から選択 |

**同時リクエスト数制限の対象**（`expenditure_expenditure_no_eq` を指定しない一覧取得）。

`invoices` と同様、`POST`/`PATCH /expenditure_payments/{id}`/`DELETE` は存在しない。**更新できるのはステータスとロックのみ。**

### 支払ステータス変更 `PATCH /expenditure_payments/payment_status/{id}`

ボディの `payment_status`（必須、`integer`）にステータス値を指定する。

| 値 | 意味 |
|---|---|
| 1 | 請求書未受領 |
| 2 | 請求書受領済 |
| 4 | 振込予約済 |
| 3 | 支払済 |

```bash
curl -X PATCH --cacert "$BOID_API_CA_FILE" \
  -H "Content-Type: application/json" \
  -d '{"payment_status":3}' \
  "$BOID_API_BASE/board-api/expenditure_payments/payment_status/123456"
```

### 支払のロック `PATCH /expenditure_payments/lock_flg/{id}`

ボディの `lock_flg` は `integer`（`0`:未ロック `1`:ロック済み）。

## 発注の書類（expenditure_documents: 見積依頼書/発注書/検収書/支払通知書）

案件の書類（[invoices-and-documents.md](invoices-and-documents.md) の `/documents/{種別}/{id}`）と対称。「取得」「更新」「ロック」のみで作成・削除APIはない。

| 書類種別 | 取得 | 更新 | ロック |
|---|---|---|---|
| 見積依頼書（estimates） | `GET /expenditure_documents/estimates/{id}` | `PATCH /expenditure_documents/estimates/{id}` | `PATCH /expenditure_documents/estimates/lock_flg/{id}` |
| 発注書（orders、発注先に対して自社が発行する発注書） | `GET /expenditure_documents/orders/{id}` | `PATCH /expenditure_documents/orders/{id}` | `PATCH /expenditure_documents/orders/lock_flg/{id}` |
| 検収書（deliveries） | `GET /expenditure_documents/deliveries/{id}` | `PATCH /expenditure_documents/deliveries/{id}` | `PATCH /expenditure_documents/deliveries/lock_flg/{id}` |
| 支払通知書（invoices） | `GET /expenditure_documents/invoices/{id}` | `PATCH /expenditure_documents/invoices/{id}` | `PATCH /expenditure_documents/invoices/lock_flg/{id}` |

```bash
# 発注書（発注先向け）取得
curl --cacert "$BOID_API_CA_FILE" \
  "$BOID_API_BASE/board-api/expenditure_documents/orders/456789"
```

**`/documents/orders/{id}`（案件の書類、顧客向けの発注書＝受注書）と `/expenditure_documents/orders/{id}`（発注の書類、発注先向けの発注書）は別物。** どちらも「orders」というキーだが、宛先（顧客/発注先）が異なる点に注意。

## エンドポイント一覧

| メソッド | パス | 概要 |
|---|---|---|
| GET | `/expenditures` | 発注リスト取得 |
| POST | `/expenditures` | 発注登録 |
| GET | `/expenditures/{id}` | 発注取得 |
| PATCH | `/expenditures/{id}` | 発注更新 |
| DELETE | `/expenditures/{id}` | 発注削除 |
| PATCH | `/expenditures/expenditure_status/{id}` | 発注ステータス変更 |
| PATCH | `/expenditures/lock_flg/{id}` | 発注のロック |
| GET | `/expenditure_payments` | 支払リスト取得 |
| PATCH | `/expenditure_payments/payment_status/{id}` | 支払ステータス変更 |
| PATCH | `/expenditure_payments/lock_flg/{id}` | 支払のロック |
| GET | `/expenditure_documents/estimates/{id}` | 見積依頼書取得 |
| PATCH | `/expenditure_documents/estimates/{id}` | 見積依頼書更新 |
| PATCH | `/expenditure_documents/estimates/lock_flg/{id}` | 見積依頼書ロック |
| GET | `/expenditure_documents/orders/{id}` | 発注書取得 |
| PATCH | `/expenditure_documents/orders/{id}` | 発注書更新 |
| PATCH | `/expenditure_documents/orders/lock_flg/{id}` | 発注書ロック |
| GET | `/expenditure_documents/deliveries/{id}` | 検収書取得 |
| PATCH | `/expenditure_documents/deliveries/{id}` | 検収書更新 |
| PATCH | `/expenditure_documents/deliveries/lock_flg/{id}` | 検収書ロック |
| GET | `/expenditure_documents/invoices/{id}` | 支払通知書取得 |
| PATCH | `/expenditure_documents/invoices/{id}` | 支払通知書更新 |
| PATCH | `/expenditure_documents/invoices/lock_flg/{id}` | 支払通知書ロック |
