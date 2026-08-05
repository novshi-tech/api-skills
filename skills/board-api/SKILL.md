---
name: board-api
description: board（the-board.jp）のプロジェクト管理/案件管理APIの生のREST APIエンドポイント仕様（`https://api.the-board.jp/v1`）、boidのAPIゲートウェイ経由での呼び出し方、`x-api-key`+`Authorization: Bearer`の二重ヘッダー認証、ページネーション（`X-Total-Count`等のレスポンスヘッダー方式）、エラー形式、リクエスト制限（3req/秒・3000req/日）をまとめたAPIリファレンススキル。`curl`やHTTPクライアント、SDKからboard APIを直接叩くコードをboidサンドボックス内で書く・デバッグする・エンドポイント仕様を確認する場合に使用する。「board APIのエンドポイントを教えて」「boardの案件APIを叩くコードを書いて」「boid経由でboard APIを呼ぶには」「BOID_API_BASEでboardを呼びたい」「board APIの発注のレスポンス形式は」「board APIで見積書を更新するには」「board APIのレスポンスグループって何」など、board APIの仕様そのものに関する質問・実装依頼で使用する。既存の `board` CLIラッパースキル（`board` コマンド経由でのタスク実行、たとえばプロジェクト一覧を取る・発注を作成するなど）を頼まれた場合はこのスキルではなく `board` CLIスキルを使うこと。
---

# board API リファレンス（boid APIゲートウェイ経由）

board（[the-board.jp](https://the-board.jp)）が提供する案件・見積・発注・請求管理APIの仕様を、**boidのAPIゲートウェイ（`internal/apigateway`）経由で呼び出す**前提でまとめたリファレンス。boidのサンドボックス化されたジョブの中からboard APIを直接叩くコードを書いたり、レスポンス形式を確認したりする際に使う。

このスキルは **APIの仕様書** であり、社内の `board` CLIの使い方ガイドではない。CLI経由の操作（`board project list` のようなコマンド実行）を頼まれた場合は `board` スキル（CLIスキル、`name: board`）を使うこと。

## 対応範囲

board APIが公開しているリソース全体（顧客・案件・発注・請求・書類・マスタ）を対象とする。ドメインは大きく以下に分かれる:

- **顧客（Client）** — 顧客（clients）・顧客支社（client_branches）・顧客担当者（contacts）
- **発注先（Payee）** — 発注先（payees）・発注先支社（payee_branches）・発注先担当者（payee_contacts）
- **案件（Project）** — 案件（projects）・案件原価（project_costs）・受注ステータス変更・案件ロック
- **売上側の書類** — 請求（invoices、ステータス変更のみ。作成・更新（本体）・削除はできない）・案件の書類（project_documents: 見積書/発注書/納品書/請求書/領収書。取得・更新・ロックのみで作成/削除APIはない）
- **発注側の書類** — 発注（expenditures）・支払（expenditure_payments、ステータス変更とロックのみ。作成・更新（本体）・削除はできない）・発注の書類（expenditure_documents: 見積依頼書/発注書/検収書/支払通知書。取得・更新・ロックのみ）
- **計上・マスタ** — 計上データ（analyses、参照専用）・ユーザー（users）・グループ（groups）・支払条件（payment_terms）・案件区分（project_types）・発注区分（expenditure_types）・会計区分（accounting_types）・カスタム書類送付方法（document_send_channels）。マスタ系はいずれも参照専用（一覧取得のみ）

## 最重要: ベースURLはハードコードしない・boidゲートウェイ経由で呼ぶ

board API自体の素のベースURLは `https://api.the-board.jp/v1` だが、**boid配下のジョブはこのホストに直接アクセスしない。** boidはサンドボックス化されたジョブが認証情報を一切保持しないまま外部APIを呼べるように、認証ゲートウェイ（`internal/apigateway`）を挟む設計になっている。

### 仕組み

1. boidはジョブ起動時に環境変数を自動注入する:
   - `BOID_API_BASE` — 形式は `https://boid-gateway:<port>/api/<job-token>`。ポートはジョブごとに動的に割り当てられるため固定値を仮定しない。**値は必ずこの環境変数から読み、コード中に書き起こさない**
   - `BOID_API_CA_FILE` — ゲートウェイのTLS終端が使う内部CA証明書のパス（Node.jsジョブでは、プロジェクト/ワークスペース側で `NODE_EXTRA_CA_CERTS` を既に設定していない限り、boidが自動でも設定する）
2. board APIを呼ぶ側は、`https://api.the-board.jp/v1/...` ではなく次の形でリクエストする:

   ```
   $BOID_API_BASE/<service>/<board-api-path>
   ```

   `<service>` はboidの `config.yaml` の `services:` ブロックで運用者が定義したサービス名。慣例的な名前は **`board-api`**（`base_url: https://api.the-board.jp/v1` にマッピングされる想定）。ただし固定の組み込み名ではないため、実際に何という名前で登録されているかは呼び出し元の `config.yaml` を確認するか、不明ならユーザーに確認すること。

3. ゲートウェイは以下を行う:
   - リクエストパス `/api/<job-token>/<service>/<tail>` をパースし、job tokenを検証する
   - `<service>` がそのjob tokenに許可されたサービス集合に含まれるかを確認する。**`services:` に定義しただけでは足りず、ワークスペース側で当該サービスを有効化していないと403になる**（`boid workspace services add board-api` 等。詳細は [references/pagination-and-errors.md](references/pagination-and-errors.md)）
   - read-only jobの場合、GET/HEAD以外のメソッド（POST/PATCH/DELETE等）は問答無用で403になる。顧客登録・案件更新・発注ステータス変更・書類ロックなど書き込み系操作をread-only jobから呼ぶことはできない
   - **クライアントが送った `Authorization` / `Cookie` / `Proxy-Authorization` ヘッダは必ず剥がして無視する**（サンドボックス側が本物の資格情報を持つことは想定されていない）
   - `services.<service>.auth` の設定に従って実際の認証情報をシークレットストアから解決し、注入してから実際の `base_url`（`https://api.the-board.jp/v1`）に転送する
   - リクエストの `<tail>` パス（クエリ文字列含む）はバイト単位でそのまま転送される
   - 実際のアップストリームのホスト名はエラー時も含めてサンドボックス側には一切見えない
   - 資格情報の注入に失敗した場合は認証情報なしで転送せず、502で失敗する（fail-closed）。ゲートウェイが返すエラーはboardのJSONエラー形式ではなく **プレーンテキスト**。詳細なステータス表は [references/pagination-and-errors.md](references/pagination-and-errors.md) を参照

つまり **クライアント側でboardの認証ヘッダを組み立てる必要はない（組み立てても剥がされて無視される）。** `$BOID_API_BASE/<service>/...` に対してリクエストを投げるだけでよい。

### 要注意: board APIは2種類のヘッダーを同時に要求する

board APIはOAuthではなく、**APIキー（`x-api-key`）とAPIトークン（`Authorization: Bearer`）という2つの独立したヘッダーを毎回同時に**要求する（[references/authentication.md](references/authentication.md) 参照）。一方、boidゲートウェイの `services.<name>.auth.kind` は `bearer` / `basic` / `header` / `query` / `oauth2` のいずれか**1種類**を1サービスに対して注入する設計になっている（`docs/ja/reference/config-yaml.md` 参照、複数ヘッダーを同時注入する仕組みは同ドキュメント上には見当たらない）。つまり **board APIを単一の `services:` エントリだけでフルに賄えるかは未検証の懸念点**であり、運用者側で以下のいずれかの対応を取っている可能性がある:

- `auth.kind: header` でどちらか一方（例えば `x-api-key`）だけを固定注入し、もう一方はboard側のAPIトークン発行設定で緩和している
- ゲートウェイ側に複数ヘッダー注入の拡張が入っている（`config-yaml.md` に明記されていない変更）
- 実運用では未対応で、boardサービス自体がまだ有効化されていない

**実装前に、実際の `config.yaml` の `services.board-api`（またはそれに相当する名前）の定義を確認し、`x-api-key`/`Authorization` の両方が正しく転送される構成になっているかをユーザーに確認すること。** 疎通確認せずに「動くはず」で実装を進めない。

### curlでの基本形

```bash
curl --cacert "$BOID_API_CA_FILE" \
  "$BOID_API_BASE/board-api/clients?per_page=1"
```

- `--cacert "$BOID_API_CA_FILE"` はゲートウェイが内部CAでTLS終端している場合に必要（省略すると証明書検証エラーになる）。`BOID_API_CA_FILE` が未設定であれば付けなくてよい
- 独自の `x-api-key` / `Authorization` ヘッダは付けない（付けても無視される）
- `board-api` の部分は運用者が `config.yaml` で定義したサービス名に置き換える。慣例上この名前が使われるが確定ではない
- このドキュメント内のURL例はすべて `$BOID_API_BASE/board-api` をベースとして記述する。実装時は環境変数をそのまま使い、URLを直書きしない

### boidゲートウェイを経由しない/直接叩く場合

boidのサンドボックス外（ローカル開発、CI、他システムなど）から直接board APIを呼ぶ場合は、通常のboard API認証（[references/authentication.md](references/authentication.md) 参照）に従って `https://api.the-board.jp/v1` を直接叩く。

**判断基準:** `BOID_API_BASE` がセットされていればboidジョブ内なので必ずゲートウェイ経由で呼ぶ。**boidサンドボックス内であることが明らかなのに `BOID_API_BASE` が未設定の場合は、「このジョブにはAPIゲートウェイが配線されていない」ことを意味する。** サンドボックスは資格情報を保持せず外向きの通信も制限されているため、この状態で `https://api.the-board.jp/v1` に直接フォールバックしても成功しない。認証情報を自作したり直接呼び出しにフォールバックしたりせず、処理を止めてユーザーに「このジョブ向けにboard-api相当のサービスがboidのAPIゲートウェイに登録・有効化されているか」を確認すること。

## 認証

クライアント自身が認証ヘッダを組み立てる必要は通常ない（ゲートウェイが代行する）。board独自の「APIキー」「APIトークン」の概念、直接呼び出し時のヘッダー形式、発行方法は [references/authentication.md](references/authentication.md) を参照。

## リソース別リファレンス

タスクに応じて該当ファイルを読むこと。全部を毎回読み込む必要はない。

- [references/authentication.md](references/authentication.md) - boidゲートウェイでの認証代行の仕組み（2ヘッダー問題含む）、直接呼び出し時のAPIキー/APIトークンの発行・ヘッダー形式
- [references/clients-and-payees.md](references/clients-and-payees.md) - 顧客系（clients/client_branches/contacts）と発注先系（payees/payee_branches/payee_contacts）のCRUD
- [references/projects-and-costs.md](references/projects-and-costs.md) - 案件（projects）、案件原価（project_costs）、受注ステータス変更、案件ロックのCRUD
- [references/invoices-and-documents.md](references/invoices-and-documents.md) - 請求（invoices、ステータス変更）と案件の書類（project_documents: 見積書/発注書/納品書/請求書/領収書）の取得・更新・ロック
- [references/expenditures.md](references/expenditures.md) - 発注（expenditures）、支払（expenditure_payments、ステータス変更）、発注の書類（expenditure_documents: 見積依頼書/発注書/検収書/支払通知書）の取得・更新・ロック
- [references/masters-and-analyses.md](references/masters-and-analyses.md) - 計上データ（analyses、参照専用）とマスタ系（users/groups/payment_terms/project_types/expenditure_types/accounting_types/document_send_channels、いずれも参照専用）
- [references/pagination-and-errors.md](references/pagination-and-errors.md) - `per_page`/`page` とレスポンスヘッダー（`X-Total-Count`等）によるページネーション、エラーレスポンス形式（`message`/`errors[]`）、レート制限（3req/秒・3000req/日・リスト取得系の同時実行数制限）、boidゲートウェイが返すエラー

## 注意点

- **レスポンスグループ（`response_group`）はデフォルトで最小限。** 一部エンドポイント（`clients`/`payees`/`projects`/`expenditures`/`invoices`/`expenditure_payments`）は `response_group` クエリで返却フィールドを絞り込めるが、**クエリを省略した場合のデフォルトは常に `small` で最小限の項目しか返らない**。選択肢は `small`/`large` の2択のみのエンドポイントと、`small`/`medium`/`large`/書類種別ごとの絞り込み（`estimate`/`order`等）/`all` まで細かく選べるエンドポイントがある（詳細は [references/pagination-and-errors.md](references/pagination-and-errors.md)）。APIドキュメント上は全項目が記載されているため、実装時に「ドキュメントにあるフィールドが返ってこない」と混乱しやすい。必要な項目が返らない場合はまず `response_group` を明示指定しているか確認すること
- **更新系メソッドは一律 `PATCH`。** freeeやBitbucketのような `PUT` は使わず、`clients`/`projects`/`expenditures` 等のリソース更新はすべて `PATCH`（部分更新）
- **書類系（project_documents/expenditure_documents）に作成・削除APIはない。** 見積書・発注書・納品書・請求書・領収書・見積依頼書・検収書・支払通知書はいずれも案件/発注の作成に伴って自動生成される前提で、APIからは「取得」「更新（PATCH）」「ロック（`lock_flg`エンドポイント）」のみ可能。新規作成・削除はできない
- **ステータス変更は専用エンドポイント。** 受注ステータス（`PATCH /projects/order_status/{id}`）、発注ステータス（`PATCH /expenditures/expenditure_status/{id}`）、請求ステータス（`PATCH /invoices/invoice_status/{id}`）、支払ステータス（`PATCH /expenditure_payments/payment_status/{id}`）は、リソース本体の更新エンドポイント（`PATCH /projects/{id}` 等）とは別の専用パスになっている。通常の更新エンドポイントでステータスフィールドを直接書き換えることはできない
- **リスト取得系の同時リクエスト数制限に注意。** 「案件No」「発注No」を指定しないリスト取得（projects/invoices/project_costs/expenditures/expenditure_paymentsの一覧）は、未返却の同時リクエストが5以上になると `429 Too Many Requests` になる。並列にページを取りに行く実装は避け、逐次処理にすること。詳細は [references/pagination-and-errors.md](references/pagination-and-errors.md)
- **レスポンスサイズの上限は10MB。** board APIはAWS API Gateway経由のため、レスポンスが10MBを超えると `500 Internal server error`（`{"message":"Internal server error"}`のみ）になる。大量データを取る場合は `per_page` を絞るかフィルタパラメータで対象を絞り込むこと
- **`per_page` のデフォルトは `10`、上限は `100`。** 全リスト取得APIで共通。省略すると10件しか返らないので、必要な件数を明示的に指定すること（[references/pagination-and-errors.md](references/pagination-and-errors.md)）
- 日時・日付のクエリフィルタは `_gteq`/`_lteq` サフィックス（例: `updated_at_gteq`）、部分一致は `_cont` サフィックス（例: `name_cont`）、IN検索は `_in[]` サフィックス（例: `order_status_in[]`。ただし複数値は同名パラメータの繰り返しではなくカンマ区切り）という命名規則で統一されている。ステータス系のIN検索・ステータス変更エンドポイントのボディ値は日本語ラベルではなく整数コード（各referenceファイルの対応表参照）
- 本ドキュメントの内容は board公式APIドキュメント（OpenAPI仕様、`board-cli` リポジトリの `docs/board_openapi.json` として取得したもの）と、boid リポジトリ（`internal/apigateway`, `docs/ja/reference/config-yaml.md`, `docs/plans/api-gateway.md`）の調査に基づく記載。boid側のサービス設定は運用者ごとにカスタマイズされるため、実際の挙動と差異が出ることがある。特に前述の「2ヘッダー問題」は実際の `config.yaml` を確認しないと解決しない未検証の懸念点であり、重要な実装の前には実際のレスポンス・実際の `config.yaml` で仕様を確認すること
