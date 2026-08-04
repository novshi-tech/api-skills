# Files

すべてのパスは `{BASE_URL}` からの相対パス。boidゲートウェイ経由の場合は `{BASE_URL}` = `$BOID_API_BASE/drive-api/drive/v3`、直接呼び出しの場合は `{BASE_URL}` = `https://www.googleapis.com/drive/v3`（アップロード系のみ `{UPLOAD_BASE_URL}` = `$BOID_API_BASE/drive-api/upload/drive/v3` または `https://www.googleapis.com/upload/drive/v3`）。詳細は [SKILL.md](../SKILL.md) と [authentication.md](authentication.md) 参照。

## ファイル一覧

```
GET /files
```

主なクエリパラメータ:
- `q` — フィルタクエリ（詳細は下記「`q` クエリ構文」）
- `pageSize` — 1ページあたりの件数（最大1000。**デフォルト値は共有ドライブ検索時は100件、非共有ドライブ（マイドライブ）検索時は原則リスト全件**という違いがある。共有ドライブを跨がない通常の検索でも、明示的に指定しておいた方が意図が明確になる）
- `pageToken` — 次ページ取得用トークン（[pagination-and-errors.md](pagination-and-errors.md) 参照）
- `orderBy` — ソート順。例: `modifiedTime desc`, `name`, `folder,name`（カンマ区切りで複数指定可、各フィールドに `desc` を付与可能）。指定可能フィールド: `createdTime`, `folder`, `modifiedByMeTime`, `modifiedTime`, `name`, `name_natural`, `quotaBytesUsed`, `recency`, `sharedWithMeTime`, `starred`, `viewedByMeTime`
- `spaces` — 検索対象スペース。`drive`（デフォルト）/ `appDataFolder`
- `corpora` — 検索範囲。`user`（デフォルト） / `domain`（ドメイン共有アイテム） / `drive`（特定の共有ドライブ、`driveId` 必須） / `allDrives`（非推奨気味、`user,allDrives` のような組み合わせは非推奨。効率のため `user` または `drive` の使用が推奨される）
- `driveId` — 共有ドライブ内を検索する場合に指定
- `includeItemsFromAllDrives` / `supportsAllDrives` — 共有ドライブ配下のアイテムも対象にする場合は両方 `true` にする
- `fields` — 部分レスポンス。一覧系は既定で最小限のフィールドしか返らないため、必要なフィールドは明示する（例: `files(id,name,mimeType,parents,modifiedTime),nextPageToken`）。詳細は [pagination-and-errors.md](pagination-and-errors.md)

```bash
curl --cacert "$BOID_API_CA_FILE" \
  "$BOID_API_BASE/drive-api/drive/v3/files?q=$(python3 -c "import urllib.parse;print(urllib.parse.quote(\"mimeType='application/vnd.google-apps.folder' and trashed=false\"))")&fields=files(id,name),nextPageToken"
```

## ファイル取得

```
GET /files/{fileId}
```

主なクエリパラメータ:
- `fields` — 既定では一部の基本フィールドのみ返る。`id`, `name`, `mimeType`, `parents`, `webViewLink` などフルセットが欲しい場合は `fields=*` またはカンマ区切りで明示指定
- `supportsAllDrives` — 共有ドライブ上のファイルを取得する場合は `true`
- `acknowledgeAbuse` — マルウェア検出済みファイルをそれでもダウンロードする場合に `true`（`alt=media` と併用）

主なレスポンスフィールド:

| フィールド | 説明 |
|---|---|
| `id` | ファイルID |
| `name` | ファイル名（フォルダ内で一意である保証はない） |
| `mimeType` | MIMEタイプ。Google Workspaceネイティブ形式は `application/vnd.google-apps.*`（下記表） |
| `parents[]` | 親フォルダのID配列。マイドライブ配下では通常1件のみ（後述「フォルダ構造」参照） |
| `trashed` / `explicitlyTrashed` | ゴミ箱状態。`trashed` は親フォルダ経由の間接的なゴミ箱移動も含む |
| `size` | バイト単位のサイズ（フォルダ・ショートカット・Google Workspaceネイティブファイルには存在しない） |
| `createdTime` / `modifiedTime` | RFC 3339形式のタイムスタンプ |
| `owners[]` | 所有者情報（共有ドライブ上のファイルには存在しない。共有ドライブはドライブ自体が所有者のため） |
| `webViewLink` | ブラウザでファイルを開くリンク |
| `webContentLink` | バイナリコンテンツのダウンロードリンク |
| `shared` | 他ユーザーと共有済みか |
| `permissions[]` | アクセス権限一覧（`fields` で明示指定した場合のみ返る。詳細は[permissions-and-sharing.md](permissions-and-sharing.md)） |
| `capabilities` | 現在の認証ユーザーがそのファイルに対して実行可能な操作のフラグ集合（`canEdit`, `canShare`, `canDownload`, `canDelete` など） |
| `driveId` | 共有ドライブ配下のファイルの場合、そのドライブID |
| `shortcutDetails` | ショートカットファイルの場合、リンク先の `targetId`/`targetMimeType` |

### Google Workspaceネイティブ mimeType

| mimeType | 種別 |
|---|---|
| `application/vnd.google-apps.document` | Googleドキュメント |
| `application/vnd.google-apps.spreadsheet` | Googleスプレッドシート |
| `application/vnd.google-apps.presentation` | Googleスライド |
| `application/vnd.google-apps.folder` | フォルダ |
| `application/vnd.google-apps.drawing` | Google図形描画 |
| `application/vnd.google-apps.form` | Googleフォーム |
| `application/vnd.google-apps.script` | Apps Scriptプロジェクト |
| `application/vnd.google-apps.shortcut` | ショートカット |

これらのネイティブ形式は `files.get?alt=media`（通常ダウンロード）では取得できない。バイト列が欲しい場合は `files.export`（後述）を使う。

## ファイル作成（メタデータのみ、フォルダ作成含む）

```
POST /files
Content-Type: application/json
```

```json
{
  "name": "新しいフォルダ",
  "mimeType": "application/vnd.google-apps.folder",
  "parents": ["<parent_folder_id>"]
}
```

中身のないメタデータのみのファイル作成（フォルダ作成はこの形が典型）。バイナリコンテンツを伴うファイル作成は後述の「アップロード」を使う（`{UPLOAD_BASE_URL}/files?uploadType=...`）。

## ファイル更新（メタデータ）

```
PATCH /files/{fileId}
```

差分更新可能なフィールドのみボディに含める（`name`, `description`, `starred`, `trashed` など）。**親フォルダの変更は `parents` フィールドの直接上書きではなく専用クエリパラメータで行う:**

```
PATCH /files/{fileId}?addParents={new_parent_id}&removeParents={old_parent_id}
```

（ボディは空でも可。「移動」はこの `addParents`/`removeParents` の組み合わせで表現する。）

コンテンツ自体の更新（新しいバイナリで上書き）は `PATCH {UPLOAD_BASE_URL}/files/{fileId}?uploadType=media` のようにアップロードエンドポイントに対してPATCHする（後述「アップロード」参照）。

## ファイル削除

```
DELETE /files/{fileId}
```

完全削除（ゴミ箱を経由しない）。ゴミ箱に移動したいだけの場合は `PATCH /files/{fileId}` で `{"trashed": true}` を送る。

```
DELETE /files/trash
```

ゴミ箱を空にする（自分がオーナーのファイルのみ対象）。

## ファイルコピー

```
POST /files/{fileId}/copy
```

```json
{
  "name": "コピー先の名前",
  "parents": ["<parent_folder_id>"]
}
```

Google Workspaceネイティブファイル（Docs/Sheets/Slides）の複製にも使う。ショートカットやフォルダはコピー不可（フォルダは再帰コピーAPIが存在しないため、子要素を個別にコピーする必要がある）。

## `q` クエリ構文

`files.list` の `q` パラメータは `フィールド 演算子 値` を `and`/`or`/`not` で組み合わせる構文。

| フィールド | 意味 |
|---|---|
| `name` | ファイル名 |
| `mimeType` | MIMEタイプ |
| `parents` | 親フォルダID（`in` 演算子とセットで使う） |
| `fullText` | 全文検索（ファイル名・説明・インデックス済みコンテンツ） |
| `trashed` | ゴミ箱状態（true/false） |
| `starred` | スター付き（true/false） |
| `modifiedTime` / `createdTime` | 更新日時 / 作成日時 |
| `owners` / `writers` / `readers` | 所有者・編集者・閲覧者（メールアドレスと `in` で比較） |
| `sharedWithMe` | 自分に共有されたファイルか（true/false） |
| `visibility` | `anyoneCanFind` / `anyoneWithLink` / `domainCanFind` / `domainWithLink` / `limited` |
| `properties` | カスタムプロパティ（全アプリから見える公開プロパティ。`has` 演算子とセットで使う） |
| `appProperties` | アプリ専用のカスタムプロパティ（自アプリからのみ見える非公開プロパティ。`has` 演算子とセットで使う） |
| `shortcutDetails.targetId` | ショートカットファイルのリンク先ファイルID |

演算子: `=`, `!=`, `contains`, `>`, `<`, `>=`, `<=`, `in`（配列メンバーシップ）, `has`（プロパティのキー/値マッチ）, `and`, `or`, `not`。文字列値はシングルクォートで囲む。

**`contains` の挙動はフィールドによって異なる点に注意:**
- **`name contains 'xxx'` は前方一致（prefix match）のみ。** 部分一致（substring match）ではない。例えば `name` が `HelloWorld` の場合、`name contains 'Hello'` はヒットするが `name contains 'World'` はヒットしない。任意の位置の部分文字列で探したい場合は `fullText` を使うか、クライアント側でフィルタする
- **`fullText contains 'xxx'` はトークン（単語）単位の一致。** 文字列の任意位置の部分一致ではなく、インデックスされた単語トークンに対するマッチ。フレーズ全体で一致させたい場合はダブルクォートで囲む（例: `fullText contains '"hello world"'`）
- `mimeType contains 'xxx'` は文字列の部分一致

`has` の例: `properties has { key='mode' and value='ready' }` のように波括弧内で `key`/`value` を指定する。

例:

```
name = 'quarterly-report.pdf'
name contains 'quarterly'
mimeType = 'application/vnd.google-apps.folder'
mimeType != 'application/vnd.google-apps.folder' and trashed = false
'1a2b3c4d5e' in parents
fullText contains 'important' and trashed = false
modifiedTime > '2026-01-01T00:00:00' and mimeType contains 'image/'
sharedWithMe and name contains 'proposal'
properties has { key='status' and value='approved' }
```

**シングルクォートを含む値のエスケープには `\'` を使う。** URLエンコード（`q` 全体を `%XX` エンコード）も忘れずに行うこと。boidゲートウェイはパス＋クエリをそのまま転送するため、クライアント側で正しくエンコードする責任がある。

## アップロード

アップロード専用エンドポイント（`{UPLOAD_BASE_URL}` = 通常時 `https://www.googleapis.com/upload/drive/v3`、boidゲートウェイ経由なら `$BOID_API_BASE/drive-api/upload/drive/v3`）に対して `uploadType` クエリパラメータで方式を切り替える。新規作成は `POST /files`、既存ファイルのコンテンツ更新は `PATCH /files/{fileId}` に対して行う（パスは共通、メソッドが異なる）。

### 1. simple（`uploadType=media`）

メタデータなし、5MB以下のファイル向け。

```bash
curl -X POST --cacert "$BOID_API_CA_FILE" \
  -H "Content-Type: image/jpeg" \
  --data-binary @photo.jpg \
  "$BOID_API_BASE/drive-api/upload/drive/v3/files?uploadType=media"
```

### 2. multipart（`uploadType=multipart`）

メタデータ（`name`, `parents` 等）とコンテンツを1リクエストで送る、5MB以下向け。

**注意: 以下は multipart リクエストの構造を示す疑似コード/概念例であり、そのままでは実行できない。** `multipart/related` は各パート区切りに `\r\n`（CRLF）を要求するが、シェルのヒアドキュメント（`<<'EOF'`）は通常LF終端になるため境界がGoogle側でパースされない。また `<バイナリデータ>` の部分はプレースホルダであり、実際にバイナリを埋め込むには `cat` でファイルの中身を一時ファイルに組み立てて `--data-binary @tmpfile` で渡すなど、CRLFとバイナリ結合を正しく扱う実装が別途必要。実運用では多くの言語のHTTPクライアント/SDKが multipart 組み立てを担ってくれるため、シェルで自前実装するよりそちらを使う方が確実。

```
POST $BOID_API_BASE/drive-api/upload/drive/v3/files?uploadType=multipart
Content-Type: multipart/related; boundary=foo_bar_baz

--foo_bar_baz
Content-Type: application/json; charset=UTF-8

{"name": "photo.jpg", "parents": ["<parent_folder_id>"]}
--foo_bar_baz
Content-Type: image/jpeg

<バイナリデータ>
--foo_bar_baz--
```

### 3. resumable（`uploadType=resumable`）

5MBを超える大きいファイル、不安定な回線での再開が必要な場合向け。2段階。

**Step 1: セッション開始**

```bash
curl -X POST --cacert "$BOID_API_CA_FILE" \
  -H "Content-Type: application/json; charset=UTF-8" \
  -H "X-Upload-Content-Type: video/mp4" \
  -H "X-Upload-Content-Length: 20000000" \
  -d '{"name": "movie.mp4", "parents": ["<parent_folder_id>"]}' \
  -D - \
  "$BOID_API_BASE/drive-api/upload/drive/v3/files?uploadType=resumable"
```

レスポンスの `Location` ヘッダにセッションURI（有効期限1週間）が返る。**このURIはGoogle側の絶対URL（`www.googleapis.com/upload/drive/v3/files?uploadType=resumable&upload_id=...`）なので、boidゲートウェイ経由の場合はホストを捨ててパス＋クエリだけ取り出し、`$BOID_API_BASE/drive-api` に付け替えてから使う**（[pagination-and-errors.md](pagination-and-errors.md) の `next` URL付け替えパターンと同様の扱い）。

**Step 2: データ送信（一括、または `Content-Range` を付けたチャンク分割）**

```bash
curl -X PUT --cacert "$BOID_API_CA_FILE" \
  -H "Content-Length: 20000000" \
  --data-binary @movie.mp4 \
  "<付け替え済みセッションURI>"
```

チャンク分割時は `Content-Range: bytes START-END/TOTAL` を付ける。中断からの再開を確認するには、空ボディの `PUT` に `Content-Range: */TOTAL` を付けて送り、`308 Resume Incomplete` の `Range` ヘッダで再開位置を確認する。セッション切れの場合は `404` が返るのでStep 1からやり直す。

## ダウンロード（バイナリファイル）

```
GET /files/{fileId}?alt=media
```

PDF・画像・動画など、Google Workspaceネイティブでない通常ファイルのバイト列取得に使う。

```bash
curl --cacert "$BOID_API_CA_FILE" \
  "$BOID_API_BASE/drive-api/drive/v3/files/{fileId}?alt=media" -o output.bin
```

ウイルススキャンで警告が出る大きいファイルは通常403になるため、`acknowledgeAbuse=true` を追加する必要がある場合がある。

## エクスポート（Google Workspaceネイティブファイル）

```
GET /files/{fileId}/export?mimeType={target_mime_type}
```

Docs/Sheets/Slides/Drawingsのようなネイティブ形式は直接ダウンロードできないため、`files.export` で変換後のバイト列を取得する。**`files.export`（同期API）でエクスポートできるコンテンツは10MBまでという上限がある。** それを超える場合は後述の `files.download`（非同期API）を使う。

### 変換先 mimeType 対応表

| ソース | エクスポート可能な mimeType |
|---|---|
| Googleドキュメント (`application/vnd.google-apps.document`) | `application/pdf`, `text/plain`, `text/markdown`, `application/rtf`, `application/zip`（HTML一式）, `application/epub+zip`, `application/vnd.openxmlformats-officedocument.wordprocessingml.document`（.docx）, `application/vnd.oasis.opendocument.text`（.odt） |
| Googleスプレッドシート (`application/vnd.google-apps.spreadsheet`) | `application/pdf`, `text/csv`（**先頭シートのみ**）, `text/tab-separated-values`（**先頭シートのみ**）, `application/zip`（HTML一式）, `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`（.xlsx）, `application/vnd.oasis.opendocument.spreadsheet`（.ods） |
| Googleスライド (`application/vnd.google-apps.presentation`) | `application/pdf`, `text/plain`, `image/jpeg`（**先頭スライドのみ**）, `image/png`（**先頭スライドのみ**）, `image/svg+xml`（**先頭スライドのみ**）, `application/vnd.openxmlformats-officedocument.presentationml.presentation`（.pptx）, `application/vnd.oasis.opendocument.presentation`（.odp） |
| Google図形描画 (`application/vnd.google-apps.drawing`) | `application/pdf`, `image/jpeg`, `image/png`, `image/svg+xml` |
| Apps Scriptプロジェクト (`application/vnd.google-apps.script`) | `application/vnd.google-apps.script+json`（.json） |
| Google Vids (`application/vnd.google-apps.vid`) | `video/mp4`（ただし `files.export` では非対応。後述の `files.download` を使う必要がある） |

**画像/SVGエクスポート（Googleスライドの `image/jpeg`/`image/png`/`image/svg+xml`）は先頭スライドのみ、CSV/TSVエクスポート（Googleスプレッドシートの `text/csv`/`text/tab-separated-values`）は先頭シートのみが対象になる。** 全スライド・全シートが必要な場合は `application/pdf` や `application/zip`（HTML一式）などファイル全体を変換する形式を使うか、シート単位でAPIを分けて呼ぶ必要がある。

```bash
curl --cacert "$BOID_API_CA_FILE" \
  "$BOID_API_BASE/drive-api/drive/v3/files/{fileId}/export?mimeType=application/pdf" \
  -o exported.pdf
```

正確な最新の対応表はGoogle側で追加・変更されることがあるため、重要な実装の前に公式ドキュメント（Export MIME types for Google Workspace documents）で確認すること。

### 10MB超のエクスポート・Google Vidsは `files.download`（非同期API）を使う

```
POST /files/{fileId}/download?mimeType={target_mime_type}
```

`files.export` は10MBまでという制約があるが、それを超えるファイルをエクスポートしたい場合や、**Google Vids（`video/mp4` へのエクスポート）のように `files.export` 自体が対応していないケース**では、v3で追加された非同期の `files.download` を使う。これはlong-running operation（LRO）方式で、同期的にバイト列を返さない点が `files.export` と異なる：

1. `POST /files/{fileId}/download`（ボディは空。エクスポート先の変換フォーマットが必要な場合は `mimeType` クエリパラメータを付ける。特定リビジョンを対象にする場合は `revisionId` も指定可能）を呼ぶと、`Operation` オブジェクト（`name` に不透明なoperation ID）が返る
2. このoperationをポーリングし、完了（`done: true`）したら結果からダウンロード用のURLを取得して実際のバイト列を取得する
3. operationの有効期限は作成から24時間

「10MBを超えるエクスポートはできない」という制約は `files.export`（同期API）にのみ当てはまり、`files.download`（非同期API）で回避できる。

## フォルダ構造（`parents` と DAG）

Google Driveは厳密な木構造（ツリー）ではなく **DAG（有向非巡回グラフ）** である。マイドライブ配下では通常ファイルは1つの親（`parents` 配列の要素数1）しか持たないが、仕組み上は複数の親を持つファイルが存在しうる（レガシーな「複数フォルダに追加」機能の名残や、一部の共有経路）。`parents` 配列は常に1要素だと仮定したコードは書かないこと。

フォルダ自体も `mimeType = 'application/vnd.google-apps.folder'` を持つ通常の `files` リソースであり、専用のFoldersリソースは存在しない。ルート（マイドライブのトップ）は特別なエイリアス `root` で参照できる（例: `'root' in parents`）。

## 変更検知（Changes）

差分同期を行いたい場合、`files.list` のポーリングではなく `changes` コレクションを使う。

### 開始トークン取得

```
GET /changes/startPageToken
```

メソッド名は `changes.getStartPageToken` だが、実際のURLパスは `/changes/startPageToken`（`getStartPageToken` という文字列はパスには出てこない）。レスポンスの `startPageToken` を初回の起点として保存する。

### 変更一覧取得

```
GET /changes?pageToken={page_token}
```

主なクエリパラメータ:
- `pageToken` — 前回保存した `startPageToken`（または前回レスポンスの `nextPageToken`）
- `spaces` — `drive` / `appDataFolder`
- `includeRemoved` — 削除済みアイテムの変更も含めるか
- `restrictToMyDrive` — マイドライブ内の変更のみに絞るか
- `includeItemsFromAllDrives` / `supportsAllDrives` — 共有ドライブ配下の変更も対象にするか

レスポンス:

```json
{
  "changes": [
    { "fileId": "...", "removed": false, "file": { "...": "..." }, "time": "..." }
  ],
  "nextPageToken": "...",
  "newStartPageToken": "..."
}
```

`nextPageToken` がある間はページングを続け、最終ページで返る `newStartPageToken` を次回ポーリングの起点として永続化する（`nextPageToken` と `newStartPageToken` は同時には返らない）。

### プッシュ通知（`files.watch` / `changes.watch`）

ポーリングの代わりにWebhook通知を受け取りたい場合に使う。

```
POST /changes/watch?pageToken={page_token}
```

```json
{
  "id": "一意なチャンネルID（UUID推奨）",
  "type": "web_hook",
  "address": "https://example.com/webhook-endpoint（HTTPS必須）"
}
```

通知は変更内容そのものを含まず、「変更があった」ことだけを次のヘッダで伝える。受信側はこれをトリガーに `changes.list` を呼んで実際の差分を取得する。

| ヘッダ | 説明 |
|---|---|
| `X-Goog-Channel-ID` | チャンネルID |
| `X-Goog-Resource-ID` | 監視対象リソースの不透明なID |
| `X-Goog-Resource-State` | `sync`（購読確立直後の初回通知） / `change`（変更あり） |
| `X-Goog-Channel-Expiration` | チャンネルの有効期限 |

boidサンドボックス内のジョブが外部からのWebhookを直接受信するアーキテクチャになっているとは限らない点に注意（受信用のパブリックエンドポイントを別途用意する必要がある）。単純なバッチ処理であればポーリング方式（`changes.list` + `pageToken` の永続化）の方が実装が容易。
