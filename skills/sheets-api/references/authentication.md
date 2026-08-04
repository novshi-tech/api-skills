# 認証

Google Sheets APIの認証は、**boidのAPIゲートウェイ経由か、直接呼び出しか**で扱いが大きく異なる。

## boidゲートウェイ経由の場合（サンドボックス化されたジョブから呼ぶ場合）

boid配下のジョブは資格情報そのものを保持しない設計になっている。実際の認証はゲートウェイ（`internal/apigateway`）が代行する。

### クライアント側がやること

- **何もしない。** `Authorization` ヘッダを自分で組み立てて送る必要はない
- 送っても意味がない: ゲートウェイは受け取ったリクエストから `Authorization` / `Cookie` / `Proxy-Authorization` を必ず剥がしてから転送する（クライアント側の値は一切アップストリームに届かない）
- `--cacert "$BOID_API_CA_FILE"` を付けてTLS証明書を検証できるようにする（ゲートウェイは内部CAでTLS終端しているため、これがないと証明書エラーになる）

```bash
curl --cacert "$BOID_API_CA_FILE" \
  "$BOID_API_BASE/sheets-api/v4/spreadsheets/{spreadsheetId}?fields=spreadsheetId,properties.title"
```

（疎通確認には書き込み系ではなく `spreadsheets.get` のような読み取り専用エンドポイントを使う。対象の `spreadsheetId` はテスト用に共有権限を付与済みのものを使うこと。認証情報がサービスアカウントの場合、サービスアカウントのメールアドレスに対象スプレッドシートの閲覧/編集権限を明示的に共有しておく必要がある点に注意）

### ゲートウェイ側の設定（参考・デバッグ用）

運用者は boid デーモンの `config.yaml` に次のようなサービス定義を置く（呼び出し元リポジトリの `.boid/project.yaml` ではなく、boid デーモン自体の設定に置く点に注意。project.yaml はリポジトリ由来の信頼境界のため、credential にアクセスできる設定はここには置かない設計になっている）:

```yaml
services:
  sheets-api:
    base_url: https://sheets.googleapis.com
    auth:
      kind: oauth2
      secret_key: SHEETS_SERVICE_ACCOUNT
```

- `auth.kind` は `bearer` / `basic` / `header` / `query` / `oauth2` から選べる。Google系APIでは慣例として `oauth2` が使われることが多い。**ただし `auth.kind: oauth2` がサービスアカウントのJSON鍵から具体的にどうアクセストークンを解決するか（JWTベアラーフローの詳細、トークンのキャッシュ・再利用の挙動等）はboid側の実装として本ドキュメントでは未検証であり、確定した仕様として書けない。** 実際の解決方式は `internal/apigateway` の実装やboidの設定リファレンス（`docs/ja/reference/config-yaml.md`）を確認すること
- `secret_key` はboidのシークレットストア上のキー名（例: `SHEETS_SERVICE_ACCOUNT`）で、実際のサービスアカウント鍵やトークン値は `config.yaml` に平文で書かない
- 資格情報の解決・注入に失敗した場合、ゲートウェイは認証情報なしで転送せず502を返す（fail-closed）
- ゲートウェイは他にも401（job token自体が無効・期限切れ）、403（サービスが未有効化 or read-only jobでの書き込み試行）、404（パス不正）、503（シークレットストア未接続）を返しうる。これらはGoogle自体のエラーではなくゲートウェイが生成したもので、レスポンスボディもGoogleのJSON形式ではなくプレーンテキスト。ステータスごとの切り分けは [pagination-and-errors.md](pagination-and-errors.md) の一覧を参照

### サービス名は固定ではない

`sheets-api` という名前はboidの組み込みデフォルトではなく、ドキュメント・テストで使われている慣例的な名前にすぎない。実際に何という名前で `services:` に登録されているかは運用者の `config.yaml` 次第。不明な場合はコード内で決め打ちせず、環境や設定から確認するか、ユーザーに確認する。

## 直接呼び出しの場合（boidサンドボックス外から）

`BOID_API_BASE` が環境変数にセットされていない、あるいはboidジョブの外（ローカル開発・CI等）から呼ぶ場合は、通常のGoogle OAuth 2.0認証を自前で扱う。

### 1. サービスアカウント（サーバー間連携）

CI/CDやバッチ処理など、特定のユーザーに代理せずアプリケーション自身としてアクセスする場合に使う。GCPコンソールでサービスアカウントを作成しJSON鍵を発行、JWTベアラーフローでアクセストークンを取得する。

```bash
curl -X GET "https://sheets.googleapis.com/v4/spreadsheets/{spreadsheetId}" \
  -H "Authorization: Bearer <access_token>"
```

サービスアカウントは対象スプレッドシートに対する暗黙のアクセス権を持たないため、**対象スプレッドシートをサービスアカウントのメールアドレス（`xxx@xxx.iam.gserviceaccount.com`）に共有（閲覧者/編集者）しておく必要がある。** 共有していないスプレッドシートに対しては、サービスアカウント自体が有効でも403（`PERMISSION_DENIED`）になる。

### 2. OAuth 2.0（ユーザー代理、Authorization Code Grant）

エンドユーザー自身のスプレッドシートを本人の代わりに操作する場合に使う。OAuth同意画面経由でユーザーの同意を得てアクセストークン（+リフレッシュトークン）を取得する。

```bash
curl -X GET "https://sheets.googleapis.com/v4/spreadsheets/{spreadsheetId}" \
  -H "Authorization: Bearer <access_token>"
```

アクセストークンの有効期限は短く（通常1時間程度）、リフレッシュトークンでの更新が前提。

### 3. APIキーのみでの利用は原則不可

Sheets APIは基本的に非公開データ（個々のユーザーのスプレッドシート）を扱うAPIであり、APIキー単体（OAuthなし）では認証できない。公開設定が「ウェブに公開」（Publish to web）されたスプレッドシートに限らず、**「リンクを知っている全員」に公開（一般公開のアクセス権）設定にしただけのスプレッドシートでも、その読み取りに限りAPIキー単体で読める場合がある。** どちらの公開設定であっても書き込みは不可で、一般的な用途（自分のスプレッドシートの読み書き）では常にOAuth 2.0（サービスアカウントまたはユーザー代理）が必要と考えてよい。

### OAuth 2.0 スコープ

用途に応じて最小権限のスコープを選ぶ。

| スコープ | 説明 |
|---|---|
| `https://www.googleapis.com/auth/spreadsheets.readonly` | すべてのGoogleスプレッドシートの閲覧のみ |
| `https://www.googleapis.com/auth/spreadsheets` | すべてのGoogleスプレッドシートの閲覧・編集・作成・削除 |
| `https://www.googleapis.com/auth/drive.readonly` | Google Drive全体の読み取り（スプレッドシートのメタデータ取得やファイル一覧に必要な場合がある） |
| `https://www.googleapis.com/auth/drive.file` | アプリ自身が作成した、またはユーザーが明示的に開いたファイルのみへのアクセス |
| `https://www.googleapis.com/auth/drive` | Google Drive全体への完全アクセス |

`spreadsheets`/`spreadsheets.readonly` はセンシティブスコープに分類され、OAuth同意画面の審査（アプリ公開時）が必要になる場合がある。個人利用・社内限定利用（テストユーザー登録の範囲内）であれば審査不要で使える。

## 認証エラー時のレスポンス（直接呼び出しの場合）

| ステータス | 意味 |
|---|---|
| 401 Unauthorized | トークン未指定・無効・期限切れ |
| 403 Forbidden (`PERMISSION_DENIED`) | トークンは有効だが対象スプレッドシートへの共有権限がない、またはスコープ不足 |

ゲートウェイ経由の場合のエラー（401/403/404/502/503それぞれの原因の切り分け）は [pagination-and-errors.md](pagination-and-errors.md) の「boidゲートウェイが返すエラー」を参照。ゲートウェイ経由のエラーは上表のGoogle標準の意味とは原因が異なることが多いので混同しないこと。
