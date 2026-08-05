# 認証

board APIの認証は、**boidのAPIゲートウェイ経由か、直接呼び出しか**で扱いが大きく異なる。

## board API自体の認証方式（前提知識）

board APIは、Bitbucketのようなトークン1本の方式やfreeeのようなOAuth 2.0ではなく、**「APIキー」と「APIトークン」という2つの独立した値を毎回セットで送る**方式を取る。

```
x-api-key: API-KEY
Authorization: Bearer API-TOKEN
```

- **APIキー（`x-api-key`）** — boardのアカウント単位で1つだけ発行される。リクエスト数制限（後述）はこのAPIキー単位で管理される
- **APIトークン（`Authorization: Bearer ...`）** — 複数発行可能。トークンごとに利用可能なエンドポイント（読み取り専用/書き込み可否、リソース単位の許可など）を細かく指定できる。用途ごとに最小権限のトークンを分けて発行することが推奨されている
- 両方とも [開発者用API設定](https://the-board.jp/api_settings) 画面で発行する（board管理画面へのログインが必要）

**どちらか一方だけでは認証できない。** `x-api-key` を送らなければ401/403、`Authorization` を送らなければ同様にエラーになる。

## boidゲートウェイ経由の場合（サンドボックス化されたジョブから呼ぶ場合）

boid配下のジョブは資格情報そのものを保持しない設計になっている。実際の認証はゲートウェイ（`internal/apigateway`）が代行する。

### クライアント側がやること

- **何もしない。** `x-api-key` / `Authorization` ヘッダを自分で組み立てて送る必要はない
- 送っても意味がない: ゲートウェイは受け取ったリクエストから `Authorization` / `Cookie` / `Proxy-Authorization` を必ず剥がしてから転送する（クライアント側の値は一切アップストリームに届かない。ただし `x-api-key` は予約ヘッダーの剥がし対象に含まれていない可能性があるため、独自に付けたつもりのない値が万一混入していないかは疑っておくこと）
- `--cacert "$BOID_API_CA_FILE"` を付けてTLS証明書を検証できるようにする（ゲートウェイは内部CAでTLS終端しているため、これがないと証明書エラーになる）

```bash
curl --cacert "$BOID_API_CA_FILE" \
  "$BOID_API_BASE/board-api/clients?per_page=1"
```

### 要注意: ゲートウェイの `auth.kind` は1サービスにつき1方式

boidの `services.<name>.auth` は `bearer` / `basic` / `header` / `query` / `oauth2` のいずれか1種類しか指定できない（`docs/ja/reference/config-yaml.md` 参照）。board APIが要求する「`x-api-key` と `Authorization: Bearer` を両方同時に注入する」という要件をこの仕組みだけで満たせるかは**未検証**。

- 単一の `auth.kind: header` は1つのヘッダー名にしか値を注入できないため、これだけでは `x-api-key` か `Authorization` のどちらか一方しか賄えない
- `config.yaml` のドキュメント上、1サービスに複数の `auth` ブロックを併記する記法は見当たらない
- **`x-api-key` はゲートウェイの剥がし対象ヘッダーに含まれていない（確定）。** ゲートウェイが強制的に剥がすのは `Authorization` / `Cookie` / `Proxy-Authorization` の3つのみで、`x-api-key` はクライアントが送った値がそのまま素通しされる。したがって理論上は「`auth.kind: bearer` で `Authorization: Bearer <APIトークン>` をゲートウェイに注入させ、`x-api-key` はサンドボックス側のコードが自分で付ける」という組み合わせで両ヘッダーを揃えることは可能。ただしこれはサンドボックスにAPIキー（資格情報の一部）を持たせることになり、boidの「サンドボックスは資格情報を一切保持しない」という設計思想には反する。採用する場合はAPIキー自体の漏洩・ログ出力リスクをユーザーと合意した上で行うこと
- 運用者側でこれ以外の回避策（例えばboard側のAPIトークン発行時にAPIキー相当の値を固定文字列として埋め込む、リバースプロキシをもう一段挟む等）を取っている可能性もあるが、`board-cli` や boid のコードからは確認できていない

**実装前に、実際の `config.yaml` の `services.board-api`（またはそれに相当する名前）の定義を確認すること。** 想定される設定例（要検証・未確認）:

```yaml
services:
  board-api:
    base_url: https://api.the-board.jp/v1
    auth:
      kind: header       # 例: x-api-key 側だけを固定注入する場合
      header: x-api-key
      secret_key: BOARD_API_KEY
```

このような設定だとしても `Authorization: Bearer` 側がどう注入されるかは別途確認が必要。疎通確認（`GET /users` 等の軽いエンドポイント）をせずに実装を進めないこと。うまく通らない場合は、boardサービスがこのboid環境向けにまだ正しく配線されていない可能性が高いので、ユーザーに確認する。

### サービス名は固定ではない

`board-api` という名前はboidの組み込みデフォルトではなく、他スキル（`bitbucket-api` 等）との一貫性のために本ドキュメントで採用した慣例的な名前にすぎない。実際に何という名前で `services:` に登録されているかは運用者の `config.yaml` 次第。不明な場合はコード内で決め打ちせず、環境や設定から確認するか、ユーザーに確認する。

## 直接呼び出しの場合（boidサンドボックス外から）

`BOID_API_BASE` が環境変数にセットされていない、あるいはboidジョブの外（ローカル開発・CI等）から呼ぶ場合は、`x-api-key` と `Authorization: Bearer` の両方を自分で組み立てて送る。

```bash
curl -X GET "https://api.the-board.jp/v1/clients?per_page=1" \
  -H "x-api-key: <API-KEY>" \
  -H "Authorization: Bearer <API-TOKEN>"
```

- POST/PATCHでは `Content-Type: application/json` も必須。付けないと `415 Unsupported Media Type` になる
- APIキー・APIトークンは平文でコードに書き起こさず、環境変数やシークレットストアから読むこと

## 認証エラー時のレスポンス（直接呼び出しの場合）

| ステータス | 意味 |
|---|---|
| 401 Unauthorized | APIキー・APIトークンのいずれかが未指定・無効 |
| 403 Forbidden | APIキー・APIトークンは有効だが、そのAPIトークンに対象エンドポイントの利用権限が設定されていない |

401・403どちらも複数の原因が考えられるため、実際のレスポンスボディの `message` を確認すること。判別しづらい場合はboard APIファーストガイドの「APIキー、APIトークンを指定しているのにエラーになります」を参照する（board公式ヘルプ、URLは省略）。

ゲートウェイ経由の場合のエラー（401/403/404/502/503それぞれの原因の切り分け）は [pagination-and-errors.md](pagination-and-errors.md) の「boidゲートウェイが返すエラー」を参照。ゲートウェイ経由のエラーは上表のboard標準の意味とは原因が異なることが多いので混同しないこと。
