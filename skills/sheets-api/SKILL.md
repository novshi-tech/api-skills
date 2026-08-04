---
name: sheets-api
description: Google Sheets API v4の生のエンドポイント仕様（boidのAPIゲートウェイ経由での呼び出し方、スプレッドシート/シート/セル値の各エンドポイント、valueInputOption・A1記法・GridRange、エラー形式・クォータ）をまとめたAPIリファレンススキル。`curl`やHTTPクライアント、SDKからGoogle Sheets APIを直接叩くコードをboidサンドボックス内で書く・デバッグする・エンドポイント仕様を確認する場合に使用する。「Sheets APIのエンドポイントを教えて」「Google SheetsのvalueInputOptionって何」「Sheets APIを叩くコードを書いて」「boid経由でSheets APIを呼ぶには」「BOID_API_BASEでSheetsを呼びたい」など、Google Sheets APIの仕様そのものに関する質問・実装依頼で使用する。既存の `google-cli` ラッパー経由の操作（スプレッドシートの読み書きなどのタスク実行）を頼まれた場合はこのスキルではなく `google-sheets` CLIスキル（`name: google-sheets`）を使うこと。
---

# Google Sheets API リファレンス（boid APIゲートウェイ経由）

Google Sheets API v4の仕様を、**boidのAPIゲートウェイ（`internal/apigateway`）経由で呼び出す**前提でまとめたリファレンス。boidのサンドボックス化されたジョブの中からSheets APIを直接叩くコードを書いたり、レスポンス形式を確認したりする際に使う。

このスキルは **APIの仕様書** であり、社内の `google-cli` を使った `google-sheets` スキルの使い方ガイドではない。CLI経由の操作（スプレッドシートの読み書きなどのタスク実行）を頼まれた場合は `google-sheets` スキル（CLIスキル、`name: google-sheets`）を使うこと。

## 最重要: ベースURLはハードコードしない・boidゲートウェイ経由で呼ぶ

Google Sheets API自体の素のベースURLは `https://sheets.googleapis.com/v4/spreadsheets` だが、**boid配下のジョブはこのホストに直接アクセスしない。** boidはサンドボックス化されたジョブが認証情報を一切保持しないまま外部APIを呼べるように、認証ゲートウェイ（`internal/apigateway`）を挟む設計になっている。

### 仕組み

1. boidはジョブ起動時に環境変数を自動注入する:
   - `BOID_API_BASE` — 形式は `https://boid-gateway:<port>/api/<job-token>`。ポートはジョブごとに動的に割り当てられるため固定値を仮定しない。**値は必ずこの環境変数から読み、コード中に書き起こさない**
   - `BOID_API_CA_FILE` — ゲートウェイのTLS終端が使う内部CA証明書のパス（Node.jsジョブでは、プロジェクト/ワークスペース側で `NODE_EXTRA_CA_CERTS` を既に設定していない限り、boidが自動でも設定する）
2. Sheets APIを呼ぶ側は、`https://sheets.googleapis.com/v4/spreadsheets/...` ではなく次の形でリクエストする:

   ```
   $BOID_API_BASE/<service>/v4/spreadsheets/...
   ```

   `<service>` はboidの `config.yaml` の `services:` ブロックで運用者が定義したサービス名。Sheets向けの慣例的な名前は **`sheets-api`**（`base_url: https://sheets.googleapis.com` にマッピングされる想定。パスは `/v4/spreadsheets/...` から始まる）。ただし固定の組み込み名ではないため、実際に何という名前で登録されているかは呼び出し元の `config.yaml` を確認するか、不明ならユーザーに確認すること。

3. ゲートウェイは以下を行う:
   - リクエストパス `/api/<job-token>/<service>/<tail>` をパースし、job tokenを検証する
   - `<service>` がそのjob tokenに許可されたサービス集合に含まれるかを確認する。**`services:` に定義しただけでは足りず、ワークスペース側で当該サービスを有効化していないと403になる**（`boid workspace services add` 等。詳細は [references/pagination-and-errors.md](references/pagination-and-errors.md)）
   - read-only jobの場合、GET/HEAD以外のメソッド（POST/PUT/DELETE等）は問答無用で403になる。スプレッドシート作成、シート追加・削除、セル値の更新・追記・クリア、`batchUpdate` など書き込み系操作をread-only jobから呼ぶことはできない
   - **クライアントが送った `Authorization` / `Cookie` / `Proxy-Authorization` ヘッダは必ず剥がして無視する**（サンドボックス側が本物の資格情報を持つことは想定されていない）
   - `services.<service>.auth` の設定に従って実際の認証情報をシークレットストアから解決し、注入してから実際の `base_url`（`https://sheets.googleapis.com`）に転送する
   - リクエストの `<tail>` パス（クエリ文字列含む）はバイト単位でそのまま転送される（正規化・デコードし直しなどはしない）。A1記法のrange（`Sheet1!A1:D5` の `!` や、シート名にスペース・日本語を含む場合の `'シート1'!A1` 等）はURLパスの一部として渡すため、正しくパーセントエンコードすること
   - 実際のアップストリームのホスト名はエラー時も含めてサンドボックス側には一切見えない
   - 資格情報の注入に失敗した場合は認証情報なしで転送せず、502で失敗する（fail-closed）。ゲートウェイが返すエラーはGoogleの `{"error": {...}}` 形式ではなく **プレーンテキスト**。詳細なステータス表は [references/pagination-and-errors.md](references/pagination-and-errors.md) を参照

つまり **クライアント側でGoogle OAuthの認証ヘッダを組み立てる必要はない（組み立てても剥がされて無視される）。** `$BOID_API_BASE/<service>/v4/spreadsheets/...` に対してリクエストを投げるだけでよい。

### curlでの基本形

```bash
curl --cacert "$BOID_API_CA_FILE" \
  "$BOID_API_BASE/sheets-api/v4/spreadsheets/{spreadsheetId}/values/{range}"
```

- `--cacert "$BOID_API_CA_FILE"` はゲートウェイが内部CAでTLS終端している場合に必要（省略すると証明書検証エラーになる）。`BOID_API_CA_FILE` が未設定であれば付けなくてよい。Node.jsではプロジェクト側で `NODE_EXTRA_CA_CERTS` を明示的に上書きしていない限り自動で通るため、通常フラグ相当の指定は不要
- 独自の `Authorization` ヘッダは付けない（付けても無視される）
- `sheets-api` の部分は運用者が `config.yaml` で定義したサービス名に置き換える。慣例上この名前が使われるが確定ではない
- このドキュメント内のURL例はすべて `$BOID_API_BASE/sheets-api` をベースとして記述する。実装時は環境変数をそのまま使い、URLを直書きしない

### boidゲートウェイを経由しない/直接叩く場合

boidのサンドボックス外（ローカル開発、CI、他システムなど）から直接Sheets APIを呼ぶ場合は、通常のGoogle OAuth 2.0認証（[references/authentication.md](references/authentication.md) 参照）に従って `https://sheets.googleapis.com` を直接叩く。

**判断基準:** `BOID_API_BASE` がセットされていればboidジョブ内なので必ずゲートウェイ経由で呼ぶ。**boidサンドボックス内であることが明らかなのに `BOID_API_BASE` が未設定の場合は、「このジョブにはAPIゲートウェイが配線されていない」ことを意味する。** サンドボックスは資格情報を保持せず外向きの通信も制限されているため、この状態で `https://sheets.googleapis.com` に直接フォールバックしても成功しない。認証情報を自作したり直接呼び出しにフォールバックしたりせず、処理を止めてユーザーに「このジョブ向けにsheets-api相当のサービスがboidのAPIゲートウェイに登録・有効化されているか」を確認すること。

### スプレッドシートの一覧・検索・共有・エクスポート・削除は対象外（Drive APIの領分）

**Google Sheets API自体には、スプレッドシートを一覧・検索するエンドポイントは存在しない。** `spreadsheetId` を指定して個別に読み書きする以外のファイル操作（例: 「マイドライブ内のスプレッドシートを一覧して」「タイトルで検索して」「スプレッドシートをCSV/XLSXとしてエクスポートして」「他ユーザーに共有権限を付与して」「スプレッドシート自体を削除・ゴミ箱に移動して」）は、Sheets APIではなく **Google Drive API**（`files.list`/`files.get`/`files.export`/`permissions.create`/`files.delete` 等）の領分であり、本スキルの範囲外。これらはリポジトリ内の別スキル [skills/drive-api/](../drive-api/SKILL.md) を参照すること。

- スプレッドシート「自体」の作成・取得・シート構造変更・セル値の読み書きは本スキル（Sheets API）
- スプレッドシート「ファイル」としての一覧・検索・共有・エクスポート・削除・Trash操作はDrive API（[skills/drive-api/](../drive-api/SKILL.md)）
- `spreadsheets.create` で作成したスプレッドシートも、サービスアカウントで作成した場合は作成者がそのサービスアカウント自身になるため、人間が編集・閲覧できるようにするには結局Drive APIで共有権限を付与する必要がある（[references/spreadsheets-and-sheets.md](references/spreadsheets-and-sheets.md) 参照）

## 認証

クライアント自身が認証ヘッダを組み立てる必要は通常ない（ゲートウェイが代行する）。ゲートウェイ側の設定例、直接呼び出し時の認証方式（OAuth 2.0スコープ、サービスアカウント等）、エラー時の切り分けは [references/authentication.md](references/authentication.md) を参照。

## リソース別リファレンス

タスクに応じて該当ファイルを読むこと。全部を毎回読み込む必要はない。

- [references/authentication.md](references/authentication.md) - boidゲートウェイでの認証代行の仕組み、直接呼び出し時の認証方式（OAuthスコープ・サービスアカウント）、ヘッダ形式
- [references/spreadsheets-and-sheets.md](references/spreadsheets-and-sheets.md) - `spreadsheets.create`/`get`、シート（タブ）の追加・削除・複製・コピー、`batchUpdate` の構造とGridRange
- [references/values.md](references/values.md) - セル値の読み書き（`values.get`/`update`/`append`/`clear`/`batchGet`/`batchUpdate`/`batchClear`）、`valueInputOption`、`majorDimension`、A1記法
- [references/pagination-and-errors.md](references/pagination-and-errors.md) - 大規模スプレッドシートでの分割取得の考え方、エラーレスポンス形式、クォータ/レート制限、`fields` パラメータ

## 注意点

- Sheets APIの一覧的な取得（例: 巨大な範囲の `values.get`）は、Bitbucketのような `next` カーソル方式のページネーションを持たない。範囲（range）そのものを分割してリクエストする設計になっている点に注意（詳細は [references/pagination-and-errors.md](references/pagination-and-errors.md)）
- A1記法（`Sheet1!A1:D5`）と、`batchUpdate` の構造変更系リクエストで使う `GridRange`（0始まりの `sheetId`/`startRowIndex`/`endRowIndex`/`startColumnIndex`/`endColumnIndex`）は別物であり、混同しないこと。値の読み書きはA1記法、書式設定やセルの構造操作はGridRangeを使う（詳細は [references/spreadsheets-and-sheets.md](references/spreadsheets-and-sheets.md)）
- `spreadsheetId` はURL等から取得できる不変の識別子だが、`sheetId`（シート/タブごとの整数ID）はスプレッドシート内で一意であり、シート名（タイトル）とは別物。シート名は変更されうるため、コード内でシートを一意に指すなら可能な限り `sheetId` を使う
- レスポンスサイズを絞りたい場合は `fields` パラメータで部分レスポンスを指定できる。詳細は [references/pagination-and-errors.md](references/pagination-and-errors.md)
- 本ドキュメントの内容は公開仕様（Google Sheets API v4公式ドキュメント）および boid リポジトリ（`internal/apigateway`, `docs/plans/api-gateway.md`, `docs/ja/reference/config-yaml.md`）の調査に基づく記載。Google側の仕様変更や、運用者ごとの `config.yaml` のサービス名・認証設定のカスタマイズにより実際の挙動と差異が出ることがある。重要な実装の前には実際のレスポンス・実際の `config.yaml` で仕様を確認すること
