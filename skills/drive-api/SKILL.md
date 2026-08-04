---
name: drive-api
description: Google Drive API v3 の生のエンドポイント仕様（boidのAPIゲートウェイ経由での呼び出し方、files/permissions/changesの各エンドポイント、アップロード方式、エクスポート、ページネーション、エラー形式）をまとめたAPIリファレンススキル。`curl`やHTTPクライアント、SDKからGoogle Drive APIを直接叩くコードをboidサンドボックス内で書く・デバッグする・エンドポイント仕様を確認する場合に使用する。「Drive APIのエンドポイントを教えて」「Google DriveのファイルアップロードAPIの仕様は」「Drive APIを叩くコードを書いて」「boid経由でDrive APIを呼ぶには」「BOID_API_BASEでDriveを呼びたい」など、Google Drive APIの仕様そのものに関する質問・実装依頼で使用する。既存の `google-drive` CLIラッパースキル（`google-cli` 経由でファイル一覧・検索・アップロード等のタスクを実行するスキル）経由の操作を頼まれた場合はこのスキルではなく `google-drive` スキルを使うこと。
---

# Google Drive API リファレンス（boid APIゲートウェイ経由）

Google Drive API v3の仕様を、**boidのAPIゲートウェイ（`internal/apigateway`）経由で呼び出す**前提でまとめたリファレンス。boidのサンドボックス化されたジョブの中からDrive APIを直接叩くコードを書いたり、レスポンス形式を確認したりする際に使う。

このスキルは **APIの仕様書** であり、社内の `google-drive` CLIの使い方ガイドではない。CLI経由の操作を頼まれた場合は `google-drive` スキル（CLIスキル）を使うこと。

## 最重要: ベースURLはハードコードしない・boidゲートウェイ経由で呼ぶ

Google Drive API自体の素のベースURLは `https://www.googleapis.com/drive/v3`（アップロード専用エンドポイントは `https://www.googleapis.com/upload/drive/v3`）だが、**boid配下のジョブはこのホストに直接アクセスしない。** boidはサンドボックス化されたジョブが認証情報を一切保持しないまま外部APIを呼べるように、認証ゲートウェイ（`internal/apigateway`）を挟む設計になっている。

### 仕組み

1. boidはジョブ起動時に環境変数を自動注入する:
   - `BOID_API_BASE` — 形式は `https://boid-gateway:<port>/api/<job-token>`。ポートはジョブごとに動的に割り当てられるため固定値を仮定しない。**値は必ずこの環境変数から読み、コード中に書き起こさない**
   - `BOID_API_CA_FILE` — ゲートウェイのTLS終端が使う内部CA証明書のパス（Node.jsジョブでは、プロジェクト/ワークスペース側で `NODE_EXTRA_CA_CERTS` を既に設定していない限り、boidが自動でも設定する）
2. Drive APIを呼ぶ側は、`https://www.googleapis.com/drive/v3/...` ではなく次の形でリクエストする:

   ```
   $BOID_API_BASE/<service>/drive/v3/<drive-api-path>
   ```

   アップロード系（`uploadType=media|multipart|resumable`）は素の実装では別ホスト配下の `/upload/drive/v3/...` を叩くが、boidゲートウェイ配下では同じ `base_url`（`https://www.googleapis.com`）にマッピングされているのが前提であれば、`$BOID_API_BASE/<service>/upload/drive/v3/...` という形でパスをそのまま継ぎ足せばよい。

   `<service>` はboidの `config.yaml` の `services:` ブロックで運用者が定義したサービス名。Drive向けの慣例的な名前は **`drive-api`**（`base_url: https://www.googleapis.com` にマッピングされる想定。パスは `/drive/v3/...` や `/upload/drive/v3/...` から始まる）。ただし固定の組み込み名ではないため、実際に何という名前で登録されているかは呼び出し元の `config.yaml` を確認するか、不明ならユーザーに確認すること。

3. ゲートウェイは以下を行う:
   - リクエストパス `/api/<job-token>/<service>/<tail>` をパースし、job tokenを検証する
   - `<service>` がそのjob tokenに許可されたサービス集合に含まれるかを確認する。**`services:` に定義しただけでは足りず、ワークスペース側で当該サービスを有効化していないと403になる**（`boid workspace services add` 等。詳細は [references/pagination-and-errors.md](references/pagination-and-errors.md)）
   - read-only jobの場合、GET/HEAD以外のメソッド（POST/PATCH/PUT/DELETE等）は問答無用で403になる。ファイル作成・更新・削除・アップロード・権限変更などの書き込み系操作をread-only jobから呼ぶことはできない
   - **クライアントが送った `Authorization` / `Cookie` / `Proxy-Authorization` ヘッダは必ず剥がして無視する**（サンドボックス側が本物の資格情報を持つことは想定されていない）
   - `services.<service>.auth` の設定に従って実際の認証情報（OAuthアクセストークンやサービスアカウント資格情報）をシークレットストアから解決し、注入してから実際の `base_url`（`https://www.googleapis.com`）に転送する
   - リクエストの `<tail>` パス（クエリ文字列含む）はバイト単位でそのまま転送される（正規化・デコードし直しなどはしない）。`q` クエリ内のシングルクォートやファイルID中の特殊文字など、パーセントエンコードが必要な箇所は自分で正しくエンコードすること
   - 実際のアップストリームのホスト名はエラー時も含めてサンドボックス側には一切見えない
   - 資格情報の注入に失敗した場合は認証情報なしで転送せず、502で失敗する（fail-closed）。ゲートウェイが返すエラーはGoogleの `{"error": {...}}` 形式ではなく **プレーンテキスト**。詳細なステータス表は [references/pagination-and-errors.md](references/pagination-and-errors.md) を参照

つまり **クライアント側でDrive APIの認証ヘッダ（`Authorization: Bearer <access_token>`）を組み立てる必要はない（組み立てても剥がされて無視される）。** `$BOID_API_BASE/<service>/...` に対してリクエストを投げるだけでよい。

### curlでの基本形

```bash
# メタデータ系（一覧・取得・作成・更新・削除・コピー・権限）
curl --cacert "$BOID_API_CA_FILE" \
  "$BOID_API_BASE/drive-api/drive/v3/files?pageSize=10"

# アップロード系
curl --cacert "$BOID_API_CA_FILE" \
  -X POST -H "Content-Type: image/jpeg" --data-binary @photo.jpg \
  "$BOID_API_BASE/drive-api/upload/drive/v3/files?uploadType=media"
```

- `--cacert "$BOID_API_CA_FILE"` はゲートウェイが内部CAでTLS終端している場合に必要（省略すると証明書検証エラーになる）。`BOID_API_CA_FILE` が未設定であれば付けなくてよい。Node.jsではプロジェクト側で `NODE_EXTRA_CA_CERTS` を明示的に上書きしていない限り自動で通るため、通常フラグ相当の指定は不要
- 独自の `Authorization` ヘッダは付けない（付けても無視される）
- `drive-api` の部分は運用者が `config.yaml` で定義したサービス名に置き換える。慣例上この名前が使われるが確定ではない
- このドキュメント内のURL例はすべて `$BOID_API_BASE/drive-api` をベースとして記述する。実装時は環境変数をそのまま使い、URLを直書きしない

### boidゲートウェイを経由しない/直接叩く場合

boidのサンドボックス外（ローカル開発、CI、他システムなど）から直接Drive APIを呼ぶ場合は、通常のGoogle OAuth 2.0認証（[references/authentication.md](references/authentication.md) 参照）に従って `https://www.googleapis.com/drive/v3`（アップロードは `https://www.googleapis.com/upload/drive/v3`）を直接叩く。

**判断基準:** `BOID_API_BASE` がセットされていればboidジョブ内なので必ずゲートウェイ経由で呼ぶ。**boidサンドボックス内であることが明らかなのに `BOID_API_BASE` が未設定の場合は、「このジョブにはAPIゲートウェイが配線されていない」ことを意味する。** サンドボックスは資格情報を保持せず外向きの通信も制限されているため、この状態で `https://www.googleapis.com` に直接フォールバックしても成功しない。認証情報を自作したり直接呼び出しにフォールバックしたりせず、処理を止めてユーザーに「このジョブ向けにdrive-api相当のサービスがboidのAPIゲートウェイに登録・有効化されているか」を確認すること。

## 認証

クライアント自身が認証ヘッダを組み立てる必要は通常ない（ゲートウェイが代行する）。ゲートウェイ側の設定例、直接呼び出し時のOAuthスコープ・認証方式、エラー時の切り分けは [references/authentication.md](references/authentication.md) を参照。

## リソース別リファレンス

タスクに応じて該当ファイルを読むこと。全部を毎回読み込む必要はない。

- [references/authentication.md](references/authentication.md) - boidゲートウェイでの認証代行の仕組み、直接呼び出し時のOAuth 2.0スコープ・サービスアカウント認証、ヘッダ形式
- [references/files.md](references/files.md) - files エンドポイント（一覧・取得・作成・更新・削除・コピー）、`q` クエリ構文、アップロード（simple/multipart/resumable）、ダウンロード（`alt=media`）、Workspaceネイティブファイルのエクスポート、変更検知（changes/watch）
- [references/permissions-and-sharing.md](references/permissions-and-sharing.md) - permissions エンドポイントのCRUD、共有設定、role/typeの値、通知メール制御
- [references/pagination-and-errors.md](references/pagination-and-errors.md) - ページネーション形式、エラーレスポンス形式、レート制限、`fields` パラメータによる部分レスポンス

## 注意点

- 日時は基本ISO 8601（RFC 3339、UTC）
- 大半のエンドポイントは `fields` クエリパラメータで返却フィールドを絞り込める（帯域節約に有効。`about`/`comments`/`replies` リソースでは実質必須）。詳細は [references/pagination-and-errors.md](references/pagination-and-errors.md) 参照
- Google Driveのフォルダ構造は厳密な木構造（ツリー）ではなくDAG（有向非巡回グラフ）である点に注意。マイドライブ配下のファイルは基本 `parents` が1件のみだが、共有アイテムなど1つのファイルが複数の親を持つケースが存在しうる。詳細は [references/files.md](references/files.md) 参照
- 本ドキュメントの内容は公開仕様（Google Drive API v3公式ドキュメント）および boid リポジトリ（`internal/apigateway`, `docs/plans/api-gateway.md`, `docs/ja/reference/config-yaml.md`）の調査に基づく記載。Google側の仕様変更や、運用者ごとの `config.yaml` のサービス名・認証設定のカスタマイズにより実際の挙動と差異が出ることがある。重要な実装の前には実際のレスポンス・実際の `config.yaml` で仕様を確認すること
