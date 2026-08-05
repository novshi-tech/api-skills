# 顧客（Client）/ 発注先（Payee）

顧客系（売上側の取引先）と発注先系（仕入・外注側の取引先）は、構造がほぼ対称（顧客=Payer側、発注先=Payee側）。

## 顧客（clients）

### 顧客リスト取得 `GET /clients`

| パラメータ | 説明 |
|---|---|
| `name_cont` | 顧客名の部分一致 |
| `name_disp_cont` | 顧客略称名の部分一致 |
| `invoice_system_number_eq` | インボイス制度の登録番号（適格請求書発行事業者番号）の完全一致 |
| `tags[]` | タグでの絞り込み（複数の場合はカンマ区切り、例: `タグA,タグB`。URLエンコード要） |
| `include_archive_flg` | アーカイブ済みも含める（デフォルト除外） |
| `updated_at_gteq` / `updated_at_lteq` | 更新日時の範囲 |
| `custom_no_eq` | カスタム番号（自社独自の顧客コード）の完全一致 |
| `per_page` / `page` | ページネーション（[pagination-and-errors.md](pagination-and-errors.md)） |
| `response_group` | 返却項目の絞り込み。`small`（既定、id/name/住所/支払条件等の基本項目のみ）/`large`（全項目）の2択（[pagination-and-errors.md](pagination-and-errors.md)） |

```bash
curl --cacert "$BOID_API_CA_FILE" \
  "$BOID_API_BASE/board-api/clients?name_cont=サンプル&response_group=large"
```

主な項目（`response_group` により変動）: `id`（顧客ID、読み取り専用）、`name`（顧客名、必須）、`name_disp`（顧客略称名）、`zip`/`pref`/`address1`/`address2`（住所）、`tel`/`fax`、`title`（敬称、デフォルト `御中`）、`payment_term_id`（デフォルト支払条件ID）ほか。

### 顧客取得 `GET /clients/{id}`

**`response_group` クエリは無い**（一覧取得（`GET /clients`）のみが対応するクエリで、詳細取得には存在しない）。

### 顧客登録 `POST /clients`

リクエストボディ（JSON）に `name` 等を指定。`Content-Type: application/json` 必須。成功時 `201 Created`、レスポンスボディに登録後の顧客オブジェクト（`id` 含む）。

```bash
curl -X POST --cacert "$BOID_API_CA_FILE" \
  -H "Content-Type: application/json" \
  -d '{"name":"サンプル株式会社","name_disp":"サンプル"}' \
  "$BOID_API_BASE/board-api/clients"
```

### 顧客更新 `PATCH /clients/{id}`

部分更新。指定したフィールドのみ更新される。

### 顧客削除 `DELETE /clients/{id}`

紐づく案件・書類が存在する場合、削除できず `422` になる可能性がある（board側の業務仕様に依存。実装前にboard公式ヘルプで削除可否の業務仕様を確認すること）。

## 顧客支社（client_branches）

顧客の支社・部門情報。`GET /client_branches?client_id_eq={顧客ID}` のように親リソースのIDで絞り込む。

| パラメータ | 説明 |
|---|---|
| `client_id_eq` | 顧客IDの完全一致 |
| `include_archive_flg` | アーカイブ済みも含める |
| `updated_at_gteq` / `updated_at_lteq` | 更新日時の範囲 |
| `per_page` / `page` | ページネーション |

CRUD（`GET`一覧/詳細、`POST`登録、`PATCH`更新、`DELETE`削除）は顧客と同様のパターン。`response_group` クエリはない。

## 顧客担当者（contacts）

顧客企業側の担当者情報。`GET /contacts?client_id_eq={顧客ID}` で絞り込む。パラメータ・CRUDパターンは `client_branches` と同様（`client_id_eq`/`include_archive_flg`/`updated_at_gteq`/`updated_at_lteq`/`per_page`/`page`）。

## 発注先（payees）

顧客と対称の構造。発注（外注・仕入）先の企業情報。

### 発注先リスト取得 `GET /payees`

| パラメータ | 説明 |
|---|---|
| `name_cont` / `name_disp_cont` | 発注先名・略称の部分一致 |
| `invoice_system_number_eq` | インボイス登録番号の完全一致 |
| `tags[]` | タグでの絞り込み（複数の場合はカンマ区切り） |
| `include_archive_flg` | アーカイブ済みも含める |
| `updated_at_gteq` / `updated_at_lteq` | 更新日時の範囲 |
| `custom_no_eq` | カスタム番号の完全一致 |
| `per_page` / `page` | ページネーション |
| `response_group` | 返却項目の絞り込み。`small`（既定）/`large`の2択（`GET /payees` のみ対応。`GET /payees/{id}` には無い） |

CRUD（`GET`/`POST`/`PATCH`/`DELETE`）は `clients` と同じパターン。ただし `response_group` は一覧取得にのみ存在し、詳細取得（`GET /payees/{id}`）には無い点は `clients` と共通。

## 発注先支社（payee_branches）/ 発注先担当者（payee_contacts）

`client_branches`/`contacts` と対称。それぞれ `payee_id_eq` で親の発注先IDに絞り込む。パラメータは `payee_id_eq`/`include_archive_flg`/`updated_at_gteq`/`updated_at_lteq`/`per_page`/`page`。CRUDパターンも同様。

## エンドポイント一覧

| メソッド | パス | 概要 |
|---|---|---|
| GET | `/clients` | 顧客リスト取得 |
| POST | `/clients` | 顧客登録 |
| GET | `/clients/{id}` | 顧客取得 |
| PATCH | `/clients/{id}` | 顧客更新 |
| DELETE | `/clients/{id}` | 顧客削除 |
| GET | `/client_branches` | 顧客支社リスト取得 |
| POST | `/client_branches` | 顧客支社登録 |
| GET | `/client_branches/{id}` | 顧客支社取得 |
| PATCH | `/client_branches/{id}` | 顧客支社更新 |
| DELETE | `/client_branches/{id}` | 顧客支社削除 |
| GET | `/contacts` | 顧客担当者リスト取得 |
| POST | `/contacts` | 顧客担当者登録 |
| GET | `/contacts/{id}` | 顧客担当者取得 |
| PATCH | `/contacts/{id}` | 顧客担当者更新 |
| DELETE | `/contacts/{id}` | 顧客担当者削除 |
| GET | `/payees` | 発注先リスト取得 |
| POST | `/payees` | 発注先登録 |
| GET | `/payees/{id}` | 発注先取得 |
| PATCH | `/payees/{id}` | 発注先更新 |
| DELETE | `/payees/{id}` | 発注先削除 |
| GET | `/payee_branches` | 発注先支社リスト取得 |
| POST | `/payee_branches` | 発注先支社登録 |
| GET | `/payee_branches/{id}` | 発注先支社取得 |
| PATCH | `/payee_branches/{id}` | 発注先支社更新 |
| DELETE | `/payee_branches/{id}` | 発注先支社削除 |
| GET | `/payee_contacts` | 発注先担当者リスト取得 |
| POST | `/payee_contacts` | 発注先担当者登録 |
| GET | `/payee_contacts/{id}` | 発注先担当者取得 |
| PATCH | `/payee_contacts/{id}` | 発注先担当者更新 |
| DELETE | `/payee_contacts/{id}` | 発注先担当者削除 |
