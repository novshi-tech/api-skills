# セル値の読み書き（spreadsheets.values）

すべてのパスは `{BASE_URL}/{spreadsheetId}/values` からの相対パス。boidゲートウェイ経由の場合は `{BASE_URL}` = `$BOID_API_BASE/sheets-api/v4/spreadsheets`、直接呼び出しの場合は `{BASE_URL}` = `https://sheets.googleapis.com/v4/spreadsheets`（詳細は [SKILL.md](../SKILL.md) 参照）。

## A1記法によるrange指定

- 単一セル: `A1`
- 矩形範囲: `A1:D5`
- シート指定あり: `Sheet1!A1:D5`
- シート指定なし: `A1:D5` はスプレッドシートの先頭（最初）のシートを対象とする
- 列全体/行全体: `A:A`（A列全体）, `1:1`（1行目全体）
- シート全体: シート名のみ（例: `Sheet1`）
- シート名にスペース・記号・数字始まりなど特殊文字を含む場合は単一引用符で囲む: `'2026年8月'!A1:B2`, `'My Sheet'!A1`
- rangeはURLパスの一部（`{range}` セグメント）として渡すため、`!`, `'`, スペース等は正しくURLエンコードすること(例: `Sheet1!A1:D5` → `Sheet1%21A1%3AD5` 相当。実際にはcurl等のクライアントライブラリのURLエンコード機能に任せるのが安全)
- **より実害の大きい落とし穴:** `values.append`/`values.clear` のURLは `{BASE_URL}/{spreadsheetId}/values/{range}:append` のように、range部分の直後に `:append`/`:clear` というメソッドサフィックスが**コロン付きで**続く。range内の `:`（`A1:D5` のコロン）だけをエンコードして `%3A` にすればよいと思い込み、末尾のメソッドサフィックスの `:` まで一律で `%3A` にエンコードしてしまうと、Googleのルーティングがメソッドを認識できずリクエストが失敗する。**エンコードすべきは `{range}` セグメント内のコロンのみで、`:append`/`:clear` 等のメソッドサフィックスのコロンはエンコードしてはいけない。**（`URLSearchParams`やクエリ文字列用のエンコード関数をパス全体に無差別に適用すると起きがちなミス。パスとクエリを分けて組み立てること）

## 単一範囲の取得（get）

```
GET {BASE_URL}/{spreadsheetId}/values/{range}
```

クエリパラメータ:
- `majorDimension` — `ROWS`（デフォルト）または `COLUMNS`。返る2次元配列 `values` の外側配列が行単位か列単位かを決める
- `valueRenderOption` — `FORMATTED_VALUE`（デフォルト、UI表示上のフォーマット済み文字列。例: 通貨や日付の表示形式が適用された文字列）/ `UNFORMATTED_VALUE`（フォーマットを適用しない生の値。数値はnumber、日付はシリアル値）/ `FORMULA`（セルに数式が入っている場合、計算結果ではなく数式文字列自体を返す）
- `dateTimeRenderOption` — `SERIAL_NUMBER`（デフォルト、Excel/Sheets独自の日付シリアル値）/ `FORMATTED_STRING`（ロケールに応じた文字列）。**`valueRenderOption=FORMATTED_VALUE` の場合はこのパラメータは意味を持たない**（常にフォーマット済み文字列になる）ため、生の日付値が欲しい場合は `valueRenderOption=UNFORMATTED_VALUE` と組み合わせて使う

レスポンス（`ValueRange`）:

```json
{
  "range": "Sheet1!A1:B2",
  "majorDimension": "ROWS",
  "values": [
    ["名前", "金額"],
    ["田中", "1000"]
  ]
}
```

- 末尾の空行・空列は自動的に省略される（`values` の各内側配列の長さが揃わないことがある。存在しないセルは配列にすら含まれない場合がある点に注意してパースすること）
- 範囲内に値が一切ない場合、`values` フィールド自体が存在しないレスポンスになる（空配列 `[]` ではない）

## 単一範囲の更新（update）

```
PUT {BASE_URL}/{spreadsheetId}/values/{range}
Content-Type: application/json
```

クエリパラメータ（必須）:
- `valueInputOption` — `RAW` または `USER_ENTERED`（必須パラメータ、省略不可）

```json
{
  "range": "Sheet1!A1:B2",
  "majorDimension": "ROWS",
  "values": [
    ["名前", "金額"],
    ["田中", "1000"]
  ]
}
```

- 指定した `range` の矩形に対して**上書き**する。`range` より `values` が小さければその分だけ更新される。**`range` が複数セルを指す矩形範囲の場合、`values` はその `range` 内に収まらなければならない**（`range` を超えて自動的に拡張されることはない）。超過すると400エラーになる（例: `Invalid data[0]: Requested writing within range ['Sheet1'!A1:B2], but tried writing to column [C]`）
- **`range` が単一セル（例: `A1`）を指す場合に限り**、そのセルを起点として `values` の行数・列数ぶん自動的に拡張される（この場合のみ `range` は実質的な「開始位置」のヒントとして働く）
- レスポンス（`UpdateValuesResponse`）: `spreadsheetId`, `updatedRange`, `updatedRows`, `updatedColumns`, `updatedCells`。`?includeValuesInResponse=true` を付けると更新後の値も含めて返す
- 書き込み先の行・列がシートの `gridProperties.rowCount`/`columnCount` を超える場合は、`range` の指定内容にかかわらず400エラー（`exceeds grid limits` 系のメッセージ）になる。**`values.append` の `insertDataOption=INSERT_ROWS` はあくまで既存グリッド内で行を挿入する指定であり、グリッド自体の行数・列数を拡張するものではない点に注意**（グリッドが足りなければ `INSERT_ROWS` を指定していても `exceeds grid limits` になりうる）。グリッド自体を事前に広げるには、`batchUpdate` の `appendDimension`（`AppendDimensionRequest`。指定した次元に行/列を追加）や `updateSheetProperties`（`gridProperties.rowCount`/`columnCount` の変更）を使う（詳細は [spreadsheets-and-sheets.md](spreadsheets-and-sheets.md) 参照）

### valueInputOption の意味

| 値 | 動作 |
|---|---|
| `RAW` | 入力を一切解析せず、文字列としてそのままセルに入れる。`"=1+2"` はテキストの `=1+2` になり計算式として評価されない。`"1000"` も数値化されず文字列として入る場合がある点に注意 |
| `USER_ENTERED` | スプレッドシートUIに人間が直接入力したのと同じように解析される。`"=1+2"` は計算式として評価され、`"1000"` は数値、`"2026/8/4"` は日付として認識される |

数式を書き込みたい場合や、Sheets側の型推論（数値・日付・パーセンテージ等）に任せたい場合は `USER_ENTERED` を使う。文字列として厳密にそのまま入れたい場合（先頭ゼロ付きのコード番号など）は `RAW` を使う。

## 値の追記（append）

```
POST {BASE_URL}/{spreadsheetId}/values/{range}:append
Content-Type: application/json
```

クエリパラメータ:
- `valueInputOption`（必須）— `RAW` / `USER_ENTERED`
- `insertDataOption` — `OVERWRITE`（デフォルト。指定範囲より下にある既存データがあれば、それを上書きする形で追記）/ `INSERT_ROWS`（既存の行の間に新しい行を挿入してから書き込む。既存データを壊したくない場合に使う）
- `includeValuesInResponse` — `true` で追記後の実際の値を含めて返す

```json
{
  "values": [
    ["新規行1", "値A"],
    ["新規行2", "値B"]
  ]
}
```

- `range` は「探索の起点となるテーブル」を指定するために使う（例: `Sheet1!A1:B1` のようにヘッダ行を指定すると、そのテーブルの直下の最初の空行から追記される）。**追記先の実際の行番号はAPIが自動判定する**ため、事前に「今何行目まで埋まっているか」を調べてから `update` で書き込む必要がない
- レスポンス（`AppendValuesResponse`）: `tableRange`（追記対象と判定された既存テーブルの範囲）、`updates`（`UpdateValuesResponse` と同形式、実際に書き込まれた範囲・件数）

## クリア（clear）

```
POST {BASE_URL}/{spreadsheetId}/values/{range}:clear
```

**リクエストボディは空でなければならない**（公式仕様: "The request body must be empty"）。空文字列やContent-Lengthなしで送る。`{}` のようなJSONオブジェクトも本来不要で、余計なボディを送らないのが安全（値の指定はできないため、送っても無視されるかエラーになりうる）。指定範囲の値のみをクリアする。**セルの書式（背景色、罫線等）は保持され、値だけが消える。** 書式ごと削除したい場合は `batchUpdate` の `updateCells`（`fields: "userEnteredValue,userEnteredFormat"` 等）を使う（[spreadsheets-and-sheets.md](spreadsheets-and-sheets.md) 参照）。

レスポンス（`ClearValuesResponse`）:

```json
{
  "spreadsheetId": "...",
  "clearedRange": "Sheet1!A1:B10"
}
```

- `clearedRange` は実際にクリアされたA1範囲。リクエストの `range` が列全体・シート全体などシートの実サイズより大きい場合、シートの実際の範囲に丸められた値が返る

## 複数範囲の一括取得（batchGet）

```
GET {BASE_URL}/{spreadsheetId}/values:batchGet
```

クエリパラメータ（`ranges` は複数指定可、繰り返しクエリパラメータとして渡す）:

```
?ranges=Sheet1!A1:B2&ranges=Sheet2!A1:C10&majorDimension=ROWS&valueRenderOption=UNFORMATTED_VALUE
```

レスポンス:

```json
{
  "spreadsheetId": "...",
  "valueRanges": [
    { "range": "Sheet1!A1:B2", "majorDimension": "ROWS", "values": [...] },
    { "range": "Sheet2!A1:C10", "majorDimension": "ROWS", "values": [...] }
  ]
}
```

`valueRanges[]` は `ranges` クエリで指定した順序と対応する。1回のAPIコールで複数シート・複数範囲をまとめて取れるため、複数範囲を個別に `values.get` するよりリクエスト数を節約できる（クォータの節約にもなる。[pagination-and-errors.md](pagination-and-errors.md) 参照）。

## 複数範囲の一括更新（batchUpdate）

```
POST {BASE_URL}/{spreadsheetId}/values:batchUpdate
Content-Type: application/json
```

```json
{
  "valueInputOption": "USER_ENTERED",
  "data": [
    { "range": "Sheet1!A1:B2", "values": [["a", "b"], ["c", "d"]] },
    { "range": "Sheet2!A1", "values": [["single"]] }
  ],
  "includeValuesInResponse": false,
  "responseValueRenderOption": "FORMATTED_VALUE"
}
```

- `data[]` の各要素が独立した `ValueRange`。異なるシートへの書き込みを1リクエストにまとめられる
- **`spreadsheets.batchUpdate`（構造変更用、[spreadsheets-and-sheets.md](spreadsheets-and-sheets.md)）とは別物のエンドポイント。** どちらも「batchUpdate」という名前だが、こちらは `values:batchUpdate` でセル値専用、あちらは `{spreadsheetId}:batchUpdate` でシート構造・書式全般用。混同しないこと
- レスポンス（`BatchUpdateValuesResponse`）: `spreadsheetId`, `totalUpdatedRows`, `totalUpdatedColumns`, `totalUpdatedCells`, `totalUpdatedSheets`, `responses[]`（各 `data[]` 要素に対応する `UpdateValuesResponse`）

## 複数範囲の一括クリア（batchClear）

```
POST {BASE_URL}/{spreadsheetId}/values:batchClear
```

```json
{ "ranges": ["Sheet1!A1:B10", "Sheet2!A1:C1"] }
```

レスポンス（`BatchClearValuesResponse`）:

```json
{
  "spreadsheetId": "...",
  "clearedRanges": ["Sheet1!A1:B10", "Sheet2!A1:C1"]
}
```

## DataFilterベースのバリアント（batchGetByDataFilter / batchUpdateByDataFilter / batchClearByDataFilter）

上記の `values:batchGet`/`values:batchUpdate`/`values:batchClear` にはそれぞれ `values:batchGetByDataFilter` 等の姉妹エンドポイントがある。A1範囲文字列の代わりに `DataFilter`（A1範囲、GridRange、または `developerMetadataLookup`）で対象を指定できる。シート名や範囲ではなく developerMetadata のキーで対象範囲を特定したいような高度なユースケース向けで、通常のセル読み書きでは上記の素のバリアントで十分。

## majorDimension の意味（再掲）

`values` の2次元配列における「外側配列＝行か列か」を決めるパラメータ。

- `ROWS`（デフォルト）: `values[0]` が1行目、`values[0][0]` がA1セル、`values[0][1]` がB1セル
- `COLUMNS`: `values[0]` がA列（1列目）、`values[0][0]` がA1セル、`values[0][1]` がA2セル

読み取り・書き込みとも同じ意味。書き込み時にデータを列方向で持っている場合は `majorDimension=COLUMNS` を指定すれば転置せずにそのまま送れる。
