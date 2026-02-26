# CSV仕様書 V3（PromptPalette）

## 1. 対象
- SFW/NSFWタグ辞書CSV
- 代表ファイル:
  - `public/data/default_tags.csv`
  - `public/data/csv/default_tags_v1.csv`
  - `public/data/csv/nsfw_tags_v1.csv`

## 2. 列仕様
`id,category,type,safety,en,ja,zh-CN,zh-TW`

- `id`: 一意キー（推奨: `sfw_0001`, `nsfw_0001` 形式）
- `category`: `|` 区切り複数カテゴリ
- `type`: 現在は `word` を基本
- `safety`: `safe` または `nsfw`
- `en/ja/zh-CN/zh-TW`: 表示ラベル

## 3. category設計
3軸で解釈する。
- Main: 必須（1つ）
- World: 任意（0〜1推奨）
- Context: 任意（複数可）

例: `Clothing|Modern|Uniform`

## 4. バリデーション方針
`npm run validate:csv -- <path>` で検証。

段階:
1. 構文チェック
- split/trim
- 空トークン検出
- 同一行重複トークン検出

2. 語彙チェック
- 許可語彙との照合
- `safe` 行で NSFW拡張語彙を禁止
- 禁止トークン（例: `NSFW`）検出

3. 軸ルール
- Main 1つ必須
- World複数は警告（運用方針で将来エラー化可）

## 5. 正規化ルール（現行）
- 一部語彙はエイリアス正規化（例: `SciFi` -> `Sci-Fi`）
- 正規化時は警告を出し、受理継続
- `safety` / `type` は値の揺れを正規化しつつ受理（警告付き）
- 行頭 `#` のコメント行は検証対象から除外

## 6. エラー時挙動
- アプリ全体は停止しない
- 不正行はスキップ
- 行番号 / id / code / detail をログ出力
- 必須列（`tag`/`en`）欠落時も `throw` せず、`SchemaError.MissingRequiredColumn` を返す
- 改行コードは `LF/CRLF/CR` を正規化して処理する
- 先頭BOM文字は `UTF-8 (FEFF)` と `UTF-16由来 (FFFE)` を除去して処理する
- 引用符セルのカンマ（`,`）とエスケープ引用符（`""`）を受理する

## 7. 推奨運用
- 追加・修正時は必ず `validate:csv` 実行
- 正常系/異常系フィクスチャで回帰確認
  - `public/data/csv/nsfw_test.csv`
  - （互換用）`public/data/csv/nsfw_test_csv`
  - `public/data/csv/nsfw_error_test.csv`
  - `public/data/csv/sfw_error_test.csv`
  - `public/data/csv/sfw&nsfw_test.csv`
- UIテスト実行時（Vitest/jsdom）は、初期CSVの自動fetchブートストラップを無効化してノイズを抑える

## 8. 関連仕様
- モデル別ベースプロンプトはCSVではなく `src/constants/modelProfiles.js` で管理する。
- 詳細は `docs/model-profiles-spec.md` を参照。
- UI表示は `safetyViewMode`（`both/sfw/nsfw`）を持ち、辞書表示だけでなく
  Main/World/Context の候補生成にも同じ安全条件を適用する。
