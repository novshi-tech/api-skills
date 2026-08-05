# 計上データ（Analysis）/ マスタ系リソース

いずれも参照専用（一覧取得のみ、作成・更新・削除APIはない）。

## 計上データ（analyses）

案件・発注の売上/原価計上を、計上年月（`report_ym`）単位で横断的に参照するためのリソース。会計連携・レポーティング用途で使う。

### 計上データリスト取得 `GET /analyses`

| パラメータ | 説明 |
|---|---|
| `report_ym_gteq` / `report_ym_lteq` | 計上年月の範囲（例: `2025-01`） |
| `analysis_data_kbn_in[]` | 計上データ区分（売上/原価等）でのIN検索 |
| `order_status_in[]` | 紐づく案件の受注ステータスでのIN検索 |
| `expenditure_status_in[]` | 紐づく発注の発注ステータスでのIN検索 |
| `include_auto_renewal_flg` | 自動契約更新のシミュレーションデータを含めるか（`0`:含めない/既定、`1`:含める）。実データではなくシミュレーション上の計上を含めるかどうかの指定 |
| `invoice_status_in[]` | 請求ステータスでのIN検索 |
| `payment_status_in[]` | 支払ステータスでのIN検索 |
| `updated_at_gteq` / `updated_at_lteq` | 更新日時の範囲 |
| `per_page` / `page` | ページネーション |

```bash
curl --cacert "$BOID_API_CA_FILE" \
  "$BOID_API_BASE/board-api/analyses?report_ym_gteq=2025-01&report_ym_lteq=2025-12"
```

`response_group` クエリはない。**「案件No」「発注No」の指定による絞り込みができないため、同時リクエスト数制限（[pagination-and-errors.md](pagination-and-errors.md)）の対象一覧には明示的に含まれていないが、期間を絞らずに全件取得しようとするとレスポンスサイズ10MB制限に抵触しやすいので `report_ym_*` で範囲を絞ることを推奨する。**

## マスタ系リソース

いずれも `GET` のみ、`name_cont`（名称の部分一致）・`include_archive_flg`（アーカイブ済みも含める）・`per_page`/`page` という共通の絞り込みパターンを持つ。

### ユーザー `GET /users`

boardアカウントに登録されているユーザー（担当者として案件・発注に割り当てられる）。

| パラメータ | 説明 |
|---|---|
| `email_eq` | メールアドレスの完全一致 |
| `per_page` / `page` | ページネーション |

`name_cont`/`include_archive_flg` は持たない（`users` のみ他のマスタと違うパラメータセット）。

### グループ `GET /groups`

ユーザーのグループ（部署・チーム等）。`name_cont`/`include_archive_flg`/`per_page`/`page`。

### 支払条件 `GET /payment_terms`

顧客・発注先に設定するデフォルト支払条件（`clients`/`payees` の `payment_term_id` が参照する）。`name_cont`/`include_archive_flg`/`per_page`/`page`。

### 案件区分 `GET /project_types`

案件を分類するためのマスタ。

| パラメータ | 説明 |
|---|---|
| `project_type_kbn_eq` | 案件区分1〜3のどれかを指定（`1`/`2`/`3`）。未指定時は `1` 指定時と同等 |
| `name_cont` | 名称の部分一致 |
| `include_archive_flg` | アーカイブ済みも含める |
| `per_page` / `page` | ページネーション |

### 発注区分 `GET /expenditure_types`

発注を分類するためのマスタ。`expenditure_type_kbn_eq`（発注区分1〜3、`1`/`2`/`3`。未指定時は`1`指定時と同等）/`name_cont`/`include_archive_flg`/`per_page`/`page`。

### 会計区分 `GET /accounting_types`

会計連携用の区分マスタ。`accounting_type_kbn_eq`（会計区分1〜3、`1`/`2`/`3`。未指定時は`1`指定時と同等）/`name_cont`/`include_archive_flg`/`per_page`/`page`。

### カスタム書類送付方法 `GET /document_send_channels`

見積書・請求書等の送付方法（郵送/メール送付/手渡し等、アカウントごとにカスタマイズ可能）のマスタ。`name_cont`/`include_archive_flg`/`per_page`/`page`。

## エンドポイント一覧

| メソッド | パス | 概要 |
|---|---|---|
| GET | `/analyses` | 計上データリスト取得 |
| GET | `/users` | ユーザーリスト取得 |
| GET | `/groups` | グループリスト取得 |
| GET | `/payment_terms` | 支払条件リスト取得 |
| GET | `/project_types` | 案件区分リスト取得 |
| GET | `/expenditure_types` | 発注区分リスト取得 |
| GET | `/accounting_types` | 会計区分リスト取得 |
| GET | `/document_send_channels` | カスタム書類送付方法リスト取得 |
