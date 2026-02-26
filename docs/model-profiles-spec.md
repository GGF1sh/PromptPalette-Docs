# モデルプロファイル仕様（PromptPalette）

## 1. 目的
- モデル系統ごとのベースプロンプト（Positive/Negative）を安全に切替・再利用する。
- SFW/NSFWモード切替時も、破綻しないフォールバックで選択状態を維持する。
- NSFWモード中でもSFWプロファイルを選択できるようにし、運用上の柔軟性を確保する。

## 2. データソース（SSOT）
- `src/constants/modelProfiles.js`
- オブジェクト形式で管理し、キーを `profileKey` として扱う。
- 必須項目:
  - `name` `family` `mode` `positiveBase` `negativeBase`
- 任意項目:
  - `tags` `notes` `version` `enabled` `isDefault` `sortOrder` `defaultCategories`

## 3. 解決ロジック
- 実装: `src/lib/modelProfiles.js`
- `resolveModelProfiles(rawProfiles)`:
  - 定義の妥当性検証（必須/型/mode）
  - `enabled` のみ有効化
  - `sortOrder` → `profileKey` で安定ソート
  - modeごとの default 解決（複数/未設定時は警告）
  - default未設定時の警告ログには、実際に採用したフォールバック先（name/key）を明記
  - 返却:
    - `allProfiles`, `byKey`, `profilesByMode`, `defaultByMode`, `issues`

## 4. フォールバック戦略
- 実装: `resolveProfileFallback(...)`
- 優先順位:
  1. 同じ `family` の profile（新mode内）
  2. 新modeの default
  3. 新modeの先頭（最小 `sortOrder`）
- UI上のプロファイル一覧はモードで隠さず、全プロファイルを常時表示する。
- 既に選択済み/適用済みのキーが存在する場合、モード変更では自動上書きしない。

## 5. 出力の状態設計（Derived State）
- `App.jsx` では以下を分離:
  - 適用ベース: `appliedModelProfile.positiveBase / negativeBase`
  - 手動タグ: `selectedPositive / selectedNegative` から生成
- 最終出力:
  - `mergePromptText(base, user)` で重複除去しつつ結合
  - `", "` 区切りを維持
- 初期状態:
  - profileは「選択」と「適用」を分離
  - 未適用時はベースを強制挿入しない

## 6. 適用時の上書き保護
- 条件:
  - 別profileへ切替
  - かつ手動タグが残っている（dirty）
- 挙動:
  - 確認ダイアログを表示
  - 同意時のみ適用

## 7. 保存
- 「現在内容で上書き保存」は、選択中profileに対して:
  - 現在の最終Positive/Negativeを `positiveBase / negativeBase` へ保存
  - 現在の3軸フィルター選択を `defaultCategories` へ保存

## 8. 関連i18nキー
- `modelProfile.none`
- `confirm.modelProfileOverwrite`
- `toast.modelProfileApplied / modelProfileSaved / modelProfileMissing`
