# PromptPalette

画像生成AI向けのプロンプト構築・管理ツールです。  
タグ操作、重み調整、LoRA/Embedding/Wildcard連携、複数入力ブロック管理、CSVベース辞書運用を1画面で扱えます。

## 現在のステータス
- ベータ版
- 多言語対応: `ja / en / zh-CN / zh-TW`
- 対応フォーマット: `A1111 / ComfyUI`（Dialect切替あり）

## 主な機能

### プロンプト構築
- タグON/OFFでポジティブ/ネガティブを構築
- 重み調整（個別/一括）とロック保護
- ポジネガ重複時ルール切替（保持/移動/優先）
- 一括解除（全体/ポジ/ネガ/未ロック）

### ワークスペース
- 自由入力ブロック（ポジ/ネガ）を複数管理
- ライブプレビュー（選択中/全体/ポジ全部/ネガ全部）
- コピー操作（選択中・全体・ポジ・ネガ）
- 右カラム分割比率、プレビュー比率、高さのドラッグ調整
- モバイル折りたたみと列数切替（2/3/4）

### タグ探索・運用
- 検索（多言語ラベル/カテゴリ/タイプ/メタ情報）
- 3軸カテゴリフィルター（Main/World/Context）+ タイプ絞り込み
- ソート（既定/名前/カテゴリ）
- お気に入り機能、NGワード機能、ショートカット切替

### LoRA/Embedding/Wildcard連携
- フォルダ取り込み（サブフォルダ有無切替）
- サブフォルダ名のカテゴリ反映
- Stability Matrix / Civitaiメタデータ解析（`.cm-info.json` / `.metadata.json`）
- Trained Words抽出（キープロンプト/分割候補）
- キープロンプトをタグ化するビルダー（タグ名/カテゴリ編集対応）
- Civitai URL抽出と外部遷移

### モデルプロファイル
- モデル別ベースプロンプトを管理（`src/constants/modelProfiles.js`）
- プロファイル適用、上書き保存、dirty時の確認導線
- ベース文字列と手動タグを分離し、最終出力を派生結合

### 安全制御・データ管理
- CSVインポート/エクスポート
- SFW/NSFWプリセット分離運用（NSFWは手動読み込み + 確認付き）
- SFW表示切替 + 18歳確認ダイアログ
- Error Boundaryによるクラッシュ時フォールバック

## CSV仕様（要点）
- 基本列: `id,category,type,safety,en,ja,zh-CN,zh-TW`
- `category`: `|` 区切り（Main/World/Context）
- `safety`: `safe` / `nsfw`
- 検証コマンド: `npm run validate:csv -- <path>`
- エラー行はスキップし、アプリ全体は継続

詳細仕様は `docs/csv-spec-v3.md` を参照してください。

## クイックスタート

```bash
npm install
npm run dev
```

### 主要コマンド

```bash
npm run build
npm run preview
npm run test
npm run test:watch
npm run validate:csv -- public/data/csv/default_tags_v1.csv
```

### モバイル確認

```bash
npm run dev -- --host
```

同一Wi-Fi端末から `http://PCのIP:5173` にアクセス。

## 主要ディレクトリ

```text
src/
  App.jsx
  components/
    sections/
  hooks/
  lib/
  constants/
  i18n/
public/
  data/
    default_tags.csv
    csv/
```

## ドキュメント
ユーザー向け詳細・開発/仕様は `docs/` に集約しています。

- 開発経緯・実装経緯: `docs/development-journey-ja.md`
- AI協業タイムライン: `docs/ai-collaboration-timeline-ja.md`
- CSV仕様書 V3: `docs/csv-spec-v3.md`
- モデルプロファイル仕様: `docs/model-profiles-spec.md`

## ライセンス
ライセンス方針はリポジトリの設定に準拠します。
