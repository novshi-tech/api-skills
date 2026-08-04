# 認証

Google Drive API v3の認証は、**boidのAPIゲートウェイ経由か、直接呼び出しか**で扱いが大きく異なる。

## boidゲートウェイ経由の場合（サンドボックス化されたジョブから呼ぶ場合）

boid配下のジョブは資格情報そのものを保持しない設計になっている。実際の認証はゲートウェイ（`internal/apigateway`）が代行する。

### クライアント側がやること

- **何もしない。** `Authorization` ヘッダを自分で組み立てて送る必要はない
- 送っても意味がない: ゲートウェイは受け取ったリクエストから `Authorization` / `Cookie` / `Proxy-Authorization` を必ず剥がしてから転送する（クライアント側の値は一切アップストリームに届かない）
- `--cacert "$BOID_API_CA_FILE"` を付けてTLS証明書を検証できるようにする（ゲートウェイは内部CAでTLS終端しているため、これがないと証明書エラーになる）

```bash
curl --cacert "$BOID_API_CA_FILE" \
  "$BOID_API_BASE/drive-api/drive/v3/files?pageSize=1&fields=files(id,name)"
```

（疎通確認には `files.list` のような読み取り専用・低権限のエンドポイントを使う。`about` は `fields` を明示的に指定しないと空応答に近くなる点に注意。）

### ゲートウェイ側の設定（参考・デバッグ用）

運用者は boid デーモンの `config.yaml` に次のようなサービス定義を置く（`api-skills` リポジトリの `.boid/project.yaml` ではなく、boid デーモン自体の設定に置く点に注意。project.yaml はリポジトリ由来の信頼境界のため、credential にアクセスできる設定はここには置かない設計になっている）:

```yaml
services:
  drive-api:
    base_url: https://www.googleapis.com
    auth:
      kind: bearer
      secret_key: DRIVE_ACCESS_TOKEN
```

- `auth.kind` は `bearer` / `basic` / `header` / `query` / `oauth2` から選べるが、Google Drive APIの慣例は `bearer`（OAuth 2.0アクセストークンをそのまま `Authorization: Bearer <token>` として注入する）か、`oauth2`（リフレッシュトークンからのアクセストークン自動更新をゲートウェイ側で行う設定）。サービスアカウントを使う運用ではゲートウェイ側でJWT署名・トークン交換まで済ませた上でBearerトークンとして注入する構成が想定される
- `secret_key` はboidのシークレットストア上のキー名（例: `DRIVE_ACCESS_TOKEN`）で、実際のトークン値・リフレッシュトークン・サービスアカウント鍵は `config.yaml` に平文で書かない
- 資格情報の解決・注入に失敗した場合、ゲートウェイは認証情報なしで転送せず502を返す（fail-closed）
- ゲートウェイは他にも401（job token自体が無効・期限切れ）、403（サービスが未有効化 or read-only jobでの書き込み試行）、404（パス不正）、503（シークレットストア未接続）を返しうる。これらはGoogle自体のエラーではなくゲートウェイが生成したもので、レスポンスボディもGoogleのJSON形式ではなくプレーンテキスト。ステータスごとの切り分けは [pagination-and-errors.md](pagination-and-errors.md) の一覧を参照

### サービス名は固定ではない

`drive-api` という名前はboidの組み込みデフォルトではなく、ドキュメント・テストで使われている慣例的な名前にすぎない。実際に何という名前で `services:` に登録されているかは運用者の `config.yaml` 次第。不明な場合はコード内で決め打ちせず、環境や設定から確認するか、ユーザーに確認する。

## 直接呼び出しの場合（boidサンドボックス外から）

`BOID_API_BASE` が環境変数にセットされていない、あるいはboidジョブの外（ローカル開発・CI等）から呼ぶ場合は、通常のGoogle OAuth 2.0認証を自前で扱う。

### 1. OAuth 2.0（ユーザー代理、Authorization Code Grant）

エンドユーザー本人のDriveを操作するアプリで使う。Google Cloud ConsoleでOAuthクライアントを作成し、同意画面経由でアクセストークン・リフレッシュトークンを取得する。

```bash
curl -X GET "https://www.googleapis.com/drive/v3/files" \
  -H "Authorization: Bearer <access_token>"
```

アクセストークンの有効期限は短く（1時間程度）、リフレッシュトークンでの更新が前提。

### 2. サービスアカウント

サーバー間連携・バッチ処理・ドメイン全体の委任（domain-wide delegation）を伴う運用で使う。Google Cloud Consoleでサービスアカウントを作成し、JSON鍵ファイルからJWTを組み立ててOAuth 2.0トークンエンドポイント（`https://oauth2.googleapis.com/token`）と交換する。ユーザーの個人Driveを操作する場合はドメイン全体の委任設定と `subject`（代理するユーザーのメールアドレス）の指定が必要。共有ドライブ（Shared Drive）やサービスアカウント自身が所有者のファイルであれば委任なしでも動作する。

```bash
curl -X GET "https://www.googleapis.com/drive/v3/files" \
  -H "Authorization: Bearer <service_account_access_token>"
```

### OAuth 2.0スコープ

用途に応じて必要最小限のスコープを選ぶこと。

| スコープ | 説明 |
|---|---|
| `https://www.googleapis.com/auth/drive` | Drive内の全ファイルの表示・管理（フル権限。**制限付き（restricted）スコープ**に分類され、Googleの審査（OAuth verification、場合によりCASAセキュリティ評価）が最も厳しい） |
| `https://www.googleapis.com/auth/drive.readonly` | Drive内の全ファイルの表示・ダウンロードのみ（**制限付き（restricted）スコープ**。書き込み不可でも全ファイル閲覧が可能なためセキュリティ評価の対象になる） |
| `https://www.googleapis.com/auth/drive.file` | アプリ自身が作成した、またはユーザーがアプリと共有したファイルのみ作成・変更可能（**非機微（non-sensitive）スコープ**。基本的なOAuth審査のみで済み、`drive`/`drive.readonly`/`drive.metadata*` 等の制限付きスコープより大幅に審査が軽い） |
| `https://www.googleapis.com/auth/drive.metadata` | ファイルのメタデータの表示・管理（コンテンツ自体は不可。**制限付き（restricted）スコープ**） |
| `https://www.googleapis.com/auth/drive.metadata.readonly` | ファイルのメタデータの表示のみ（**制限付き（restricted）スコープ**。コンテンツ自体にアクセスしないが、メタデータ全件閲覧が可能なため制限付き扱い） |
| `https://www.googleapis.com/auth/drive.appdata` | アプリ専用の隠しフォルダ（appDataFolder）へのアクセスのみ（別名 `drive.appfolder`。**非機微（non-sensitive）スコープ**） |
| `https://www.googleapis.com/auth/drive.scripts` | Apps Scriptプロジェクトの動作変更（**制限付き（restricted）スコープ**） |
| `https://www.googleapis.com/auth/drive.apps.readonly` | Drive上でファイルを開けるアプリの一覧の表示のみ（**機微（sensitive）スコープ**） |

**注意:** Googleのスコープ分類上、`drive`/`drive.readonly`/`drive.metadata`/`drive.metadata.readonly`/`drive.scripts` はいずれも**制限付き（restricted）スコープ**に該当し、読み取り専用かどうかにかかわらずGoogleのセキュリティ評価（追加審査、場合によっては年次のサードパーティ監査）の対象になる。「`drive` だけが厳しく、`readonly`/`metadata` 系は緩い」という単純な区分けではない点に注意。**推奨:** 新規実装では可能な限り `drive.file`（非機微スコープ。基本審査のみで済む）を使う。Drive全体の閲覧・メタデータアクセスがどうしても必要な場合は、制限付きスコープの審査要件（セキュリティ評価が発生しうること）を事前に見積もった上で `drive.readonly`/`drive.metadata`/`drive.metadata.readonly`/`drive` のうち必要最小限のものを選ぶ。

## 認証エラー時のレスポンス（直接呼び出しの場合）

| ステータス | 意味 |
|---|---|
| 401 Unauthorized | トークン未指定・無効・期限切れ |
| 403 Forbidden | トークンは有効だがスコープ不足、対象ファイルへの権限がない、レート制限超過など |

Google自体が返すエラーの詳細な形式（`reason` フィールドでの原因分類）は [pagination-and-errors.md](pagination-and-errors.md) を参照。

ゲートウェイ経由の場合のエラー（401/403/404/502/503それぞれの原因の切り分け）は [pagination-and-errors.md](pagination-and-errors.md) の「boidゲートウェイが返すエラー」を参照。ゲートウェイ経由のエラーは上表のGoogle標準の意味とは原因が異なることが多いので混同しないこと。
