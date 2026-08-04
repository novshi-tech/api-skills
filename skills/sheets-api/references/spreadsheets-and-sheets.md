# Spreadsheets / Sheets（タブ） / batchUpdate

すべてのパスは `{BASE_URL}` からの相対パス。boidゲートウェイ経由の場合は `{BASE_URL}` = `$BOID_API_BASE/sheets-api/v4/spreadsheets`、直接呼び出しの場合は `{BASE_URL}` = `https://sheets.googleapis.com/v4/spreadsheets`（詳細は [SKILL.md](../SKILL.md) と [authentication.md](authentication.md) 参照）。

## Spreadsheets（スプレッドシート自体）

### スプレッドシート作成

```
POST {BASE_URL}
Content-Type: application/json
```

```json
{
  "properties": {
    "title": "新規スプレッドシート",
    "locale": "ja_JP",
    "timeZone": "Asia/Tokyo"
  },
  "sheets": [
    {
      "properties": {
        "title": "Sheet1",
        "gridProperties": { "rowCount": 1000, "columnCount": 26 }
      }
    }
  ]
}
```

- `sheets[]` を省略すると、デフォルトの1シート（`Sheet1` 相当）で作成される
- レスポンスには生成された `spreadsheetId`、`spreadsheetUrl`（`https://docs.google.com/spreadsheets/d/{spreadsheetId}/edit`）、`properties`、`sheets[]`（各シートの `sheetId` を含む）が含まれる
- サービスアカウントで作成した場合、作成者はそのサービスアカウント自身になる。人間が編集するにはその後 Drive API 等で共有権限を付与する必要がある（Sheets APIだけでは共有できない）

### スプレッドシート取得

```
GET {BASE_URL}/{spreadsheetId}
```

クエリパラメータ:
- `fields` — 部分レスポンス（[pagination-and-errors.md](pagination-and-errors.md) 参照）。全シートの全セルデータを含むと非常に大きくなるため、通常は `fields=spreadsheetId,properties,sheets.properties` 程度に絞ることを推奨
- `includeGridData` — `true` にすると各シートのセルデータ（値・書式）も含めて返す。`ranges` と併用してセル範囲を絞り込める（大きなスプレッドシートで全件取得すると非常に重いレスポンスになるため、通常は `values.get`/`batchGet` を使い、この方法は避ける）
- `ranges` — `includeGridData=true` と併用し、取得対象を特定のA1記法範囲に絞る（複数指定可）

**`fields` と `includeGridData` は併用できない（`fields` が優先され `includeGridData` は無視される）。** 公式仕様: "If a field mask is set, the `includeGridData` parameter is ignored."（フィールドマスクが設定されている場合、`includeGridData` パラメータは無視される）。つまり `includeGridData=true` でグリッドデータを取りたい場合は `fields` を指定してはいけない（`fields` を使うなら、必要なら `fields=sheets.data` のようにグリッドデータ自体をフィールドマスクの中で明示的に指定する）。「`fields` で絞り込みつつ `includeGridData=true` も付ける」という併用は意味を持たない誤った使い方なので避けること。

主なレスポンスフィールド:
- `spreadsheetId`, `spreadsheetUrl`
- `properties.title`, `properties.locale`, `properties.timeZone`, `properties.defaultFormat`
- `sheets[].properties.sheetId`（整数、シートの一意識別子）, `sheets[].properties.title`, `sheets[].properties.index`, `sheets[].properties.sheetType`（`GRID`/`OBJECT`）, `sheets[].properties.gridProperties.rowCount`/`columnCount`

### フィルタ指定での取得

```
POST {BASE_URL}/{spreadsheetId}:getByDataFilter
```

`DataFilter`（A1範囲、GridRange、または `developerMetadataLookup`）の配列を指定して該当箇所だけ取得する。`developerMetadata`（シートに紐づく非表示のキー値メタデータ）で範囲を特定したい場合に使う。通常のA1範囲指定なら `values.get`/`batchGet`（[values.md](values.md)）の方がシンプル。

## Sheets（タブ）の追加・削除・複製・コピー

シート（タブ）自体の追加・削除・複製・並べ替え・書式設定などの構造変更は、すべて **`spreadsheets.batchUpdate` 経由**で行う。個別の `POST /sheets` のような専用エンドポイントは（`copyTo` を除き）存在しない。

### batchUpdate の基本構造

```
POST {BASE_URL}/{spreadsheetId}:batchUpdate
Content-Type: application/json
```

```json
{
  "requests": [
    { "addSheet": { "properties": { "title": "新しいタブ" } } },
    { "deleteSheet": { "sheetId": 123456789 } }
  ],
  "includeSpreadsheetInResponse": false,
  "responseIncludeGridData": false
}
```

- `requests[]` に任意の数の Request オブジェクトを詰め込める。**すべてのリクエストが単一のトランザクションとして原子的に適用される**（1つでも失敗すると全体がロールバックされ何も適用されない）
- 各要素は `{ "<requestType>": { ... } }` という1キーのオブジェクト。`addSheet` / `deleteSheet` / `duplicateSheet` の他にも、セルの書式設定（`repeatCell`, `updateCells`）、条件付き書式（`addConditionalFormatRule`）、フィルタ（`setBasicFilter`）、行列の挿入・削除（`insertDimension`, `deleteDimension`）、セル結合（`mergeCells`）など数十種類のRequest型がある。Sheets APIの「構造変更・書式設定全般」はこの `requests[]` 経由の1つの仕組みに集約される設計になっている
- レスポンスは `BatchUpdateSpreadsheetResponse`。`replies[]` に `requests[]` と同じ順序・同じ件数で各リクエストの結果が入る（`addSheet` なら `{ "addSheet": { "properties": {...} } }` のように生成された `sheetId` 等を含む。多くのRequest型は空オブジェクト `{}` を返す）

### シート追加（AddSheetRequest）

```json
{ "requests": [
  { "addSheet": { "properties": {
      "title": "2026-08",
      "index": 0,
      "gridProperties": { "rowCount": 100, "columnCount": 10 }
  } } }
] }
```

`properties.sheetId` を明示的に指定することも可能（省略時はGoogle側が採番）。生成された `sheetId` はレスポンスの `replies[0].addSheet.properties.sheetId` から取得する。

**グリッドの上限に注意:** シートの `gridProperties.rowCount`/`columnCount` を超える範囲へ書き込もうとすると（`values.update`/`append` や `updateCells` 等）、`exceeds grid limits` 系の400エラーになる。グリッド自体を広げるには、この `addSheet` 時の `gridProperties` で最初から余裕を持たせるか、既存シートに対して `appendDimension`（`AppendDimensionRequest`。指定した次元に行/列を追加してグリッドを広げる）または `updateSheetProperties`（`gridProperties.rowCount`/`columnCount` を直接変更）を使う。**`values.append` の `insertDataOption=INSERT_ROWS`（[values.md](values.md)参照）は既存グリッド内での行挿入にすぎず、グリッド自体の拡張にはならない**（グリッドが足りなければこれを指定していても `exceeds grid limits` になりうる）点に注意。

### シート削除（DeleteSheetRequest）

```json
{ "requests": [ { "deleteSheet": { "sheetId": 123456789 } } ] }
```

シート名（タイトル）ではなく `sheetId`（整数）で指定する点に注意。最後の1枚のシートは削除できない（少なくとも1シートは残す必要がある）。

### シート複製（DuplicateSheetRequest）

```json
{ "requests": [ { "duplicateSheet": {
    "sourceSheetId": 0,
    "insertSheetIndex": 1,
    "newSheetName": "コピー"
} } ] }
```

同一スプレッドシート内でのシート複製。値・書式ともに複製される。

### 別スプレッドシートへのシートコピー（copyTo）

```
POST {BASE_URL}/{spreadsheetId}/sheets/{sheetId}:copyTo
Content-Type: application/json
```

```json
{ "destinationSpreadsheetId": "{別のspreadsheetId}" }
```

`batchUpdate` 系ではなく、`spreadsheets.sheets.copyTo` という専用エンドポイント（唯一の例外）。コピー先スプレッドシートに対する編集権限が別途必要。

## GridRange（構造変更・書式設定で使う範囲指定）

`batchUpdate` の多くのRequest型は、A1記法ではなく **`GridRange`**（0始まりのインデックスによる範囲指定）を使う。

```json
{
  "sheetId": 0,
  "startRowIndex": 0,
  "endRowIndex": 10,
  "startColumnIndex": 0,
  "endColumnIndex": 5
}
```

- `sheetId` — 対象シート（`GridRange` には人間可読なシート名は含まれず、必ず整数の `sheetId` で指定する）
- `startRowIndex`/`startColumnIndex` は0始まり・**inclusive（含む）**、`endRowIndex`/`endColumnIndex` は**exclusive（含まない）**。例えば `A1:B2`（2行2列）相当は `startRowIndex:0, endRowIndex:2, startColumnIndex:0, endColumnIndex:2`
- 各フィールドを省略すると「無限」を意味する（例: `startRowIndex` を省略し `endRowIndex` のみ指定すると先頭行から指定行までの全列、全フィールド省略でシート全体）

### A1記法との使い分け

| 用途 | 使う表現 |
|---|---|
| セル値の読み書き（`values.get`/`update`/`append`/`clear` 等） | A1記法の文字列（例: `Sheet1!A1:D5`） |
| セルの書式設定、条件付き書式、フィルタ、行列の挿入・削除・結合など構造変更（`batchUpdate` の各Request） | `GridRange`（0始まり整数、`sheetId` 指定） |

**A1記法をGridRangeに変換する固定APIは存在しない。** シート名→`sheetId` の対応は `spreadsheets.get` の `sheets[].properties` から事前に取得しておき、行・列番号は1始まり(A1)→0始まり(GridRange)への変換をアプリ側で行う必要がある。

## developerMetadata

シートやセル範囲、スプレッドシート全体に対して、UIには表示されないキー値メタデータを付与できる仕組み。

```
GET  {BASE_URL}/{spreadsheetId}/developerMetadata/{metadataId}
POST {BASE_URL}/{spreadsheetId}/developerMetadata:search
```

メタデータの作成・更新・削除も `batchUpdate` の `createDeveloperMetadata`/`updateDeveloperMetadata`/`deleteDeveloperMetadata` Requestで行う。テンプレート管理や、シートの並び替え後も安定してシートを識別したい場合などに使う。
