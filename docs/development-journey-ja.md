# PromptPalette 開発経緯・実装経緯（履歴書/エントリー用）

## 1. プロジェクト概要
- 名称: PromptPalette
- 種別: 画像生成AI向けプロンプト構築/管理Webアプリ
- 目的: A1111/ComfyUI利用時の「手入力・コピペ・管理の煩雑さ」を解消する
- 開発形態: 個人開発（要件整理、設計、実装、検証、運用を一貫して実施）
- 開発期間: 実働4日

## 2. 開発背景
- 既存手段（メモ帳・スプレッドシート・断片的な拡張機能）では、以下の課題があった。
  - タグ管理が分散し、再現性が低い
  - LoRA/Embeddingの呼び出し表記が環境依存
  - 大量データで検索/操作が重くなる
  - モバイルでの操作性が悪い
- 課題を解消するため、ローカル運用前提の実務的なUIを設計した。

## 3. 実装フェーズ（要約）
1. 基盤構築
- React + Vite + TailwindでSPAを構築
- 状態永続化（localStorage）とi18n基盤を整備

2. コア機能
- タグON/OFF、重み調整、ロック、一括解除
- 複数自由入力ブロック、ライブプレビュー、コピー導線

3. 生成環境対応
- A1111/ComfyUI切替
- Dialectプロファイルによる構文差吸収
- LoRA/Embedding/Random構文の出力最適化

4. 外部データ連携
- LoRA/Embedding/Wildcardのフォルダ取り込み
- Stability Matrix/Civitaiメタデータ（JSON）解析
- Trained Words抽出、Civitai URL抽出、重複抑制

5. パフォーマンス/UX改善
- 大量タグ向け仮想スクロール
- モバイル折りたたみ、列数切替、分割リサイズ
- お気に入り機能 + ショートカット

6. データ品質管理
- CSV仕様を明文化（category 3軸、safety運用）
- バリデータ実装（構文/語彙/軸ルール）
- 正常系/異常系CSVフィクスチャで回帰検証

7. モデルプロファイル基盤の強化
- モデル別ベースプロンプトを `sfw/nsfw` モードで管理
- プロファイル定義のバリデーションとフォールバック解決を実装
- ベース文字列と手動タグを分離管理し、派生結合で出力の整合性を担保
- 未保存手動タグがある状態での上書き確認（誤操作防止）

8. 障害耐性の強化
- `AppErrorBoundary` を導入し、UI描画中の未処理例外で全画面真っ白化する事象を緩和
- エラー発生時に再読み込み導線と詳細表示を提供

9. UI確認操作の統一
- `window.confirm` 依存を廃止し、アプリ内の統一確認モーダルへ集約
- タグ削除、入力欄削除、設定JSONインポート、モデルプロファイル上書きに適用

10. App肥大化への段階対応
- モデルプロファイルの解決/フォールバック処理を `useResolvedModelProfiles` に分離
- プロファイル周りの副作用をフックへ移し、`App.jsx` の責務を縮小

11. LoRAトリガー運用の改善
- LoRA/Embeddingのキープロンプトを、その場でタグ化できるモーダルビルダーを追加
- 生成タグの名称・カテゴリを作成前に編集可能化し、誤登録時のやり直しコストを削減
- 既存カテゴリ選択 + 新規カテゴリ作成可否チェックを導入し、カテゴリ汚染を抑制
- LoRA/Embedding由来で生成したタグに `linkedSourceType` を保持し、タイプ絞り込み時の発見性を向上

12. ワークスペース即応UIの追加
- 追加先ヘッダー行とNG操作エリアの双方に、ランダム選択モードのクイックON/OFFを配置
- NSFW運用時に安全表示を `NSFW/SFW` `SFWのみ` `NSFWのみ` の3択で切替可能化
- 安全表示の選択を、タグ一覧だけでなく3軸カテゴリ候補（Main/World/Context）にも連動させ、
  表示と絞り込み候補の不整合を解消

13. フォーマッタ性能最適化（互換性維持）
- `promptFormatter` の辞書参照を `Array.find` ベースから `Map.get` ベースへ最適化
- 辞書IDの重複時は既存挙動（先勝ち）を維持するため、Map生成は手動で first-win を担保
- `generatePromptString` / `collectPromptWarnings` に null/空配列ガードを追加し、入力不正時の例外を抑止
- テストを追加し、Array入力・Map入力・重複ID互換・null安全性を回帰固定

14. App分割リファクタ（フェーズ1）
- `App.jsx` で肥大化していた検索/フィルター/NG入力UI系のstateを `useDictionaryFilterState` へ抽出
- 機能変更は行わず、宣言の集約と責務分離のみを実施
- 既存テスト（96件）とビルドで回帰がないことを確認

15. App分割リファクタ（フェーズ2）
- `React.lazy` + `Suspense` を導入し、重量級セクションを遅延ロード化
  - PromptOutputsSection / ExtraInputsSection / RandomBuilderSection
  - WorkspaceSection / DataManagementSection / LegalSection
- セクションごとにスケルトンfallbackを設置し、読込中の視認性を維持
- `App.test.jsx` を非同期描画前提へ更新し、遅延ロード導入後の回帰を固定

16. App分割リファクタ（フェーズ3）
- `useAppTransientState` を新設し、`App.jsx` の非永続UI stateをまとめて抽出
- 抽出対象:
  - ランダムビルダー作業state
  - 単語追加フォームstate
  - 確認ダイアログ開閉state
  - プレビューターゲット/プレビュー表示mode
  - 一時UI state（サブフォルダ読込、検査対象、法的モーダル種別）
- 方針:
  - 機能追加や仕様変更は行わず、責務分離と可読性向上のみを実施
  - 既存テスト（96件）とビルド通過を完了条件として回帰を抑止

17. App分割リファクタ（フェーズ4）
- `useDictionaryDerivedData` を新設し、辞書派生計算を `App.jsx` から分離
- 抽出対象:
  - 3軸カテゴリ候補の算出（Main/World/Context）
  - カテゴリラベル・辞書言語候補の導出
  - NG語正規化/グループ化/フィルタ/ブロック対象ID算出
  - お気に入りID集合
  - タグ一覧の検索・安全表示・ソートを含む最終フィルタ結果
- 方針:
  - 機能仕様は変更せず、責務分離と再利用性向上に限定
  - 既存テスト（96件）/ビルドの全通過を必須条件として回帰を抑止

18. App分割リファクタ（フェーズ5）
- `usePreviewSections` を新設し、ライブプレビュー算出ロジックを `App.jsx` から分離
- 抽出対象:
  - プレビュー対象候補（ポジ/ネガ）の生成
  - 入力先変更時のプレビューターゲット同期
  - フォーカス/全体/ポジ全部/ネガ全部のセクション算出
  - アクティブラベル/アクティブテキスト導出
- 方針:
  - UI仕様は変更せず、導出ロジックの責務分離に限定
  - 既存テスト（96件）とビルドを完了条件として回帰を抑止

19. App分割リファクタ（フェーズ6）
- `useGlobalShortcuts` を新設し、`App.jsx` に散在していたグローバルショートカット処理を分離
- 抽出対象:
  - `Ctrl/⌘ + C`（フォーカス中ライブプレビューのコピー）
  - `Ctrl/⌘ + Alt + N`（NG登録モード切替）
  - `Ctrl/⌘ + Alt + F`（お気に入り表示モード切替）
  - `Ctrl/⌘ + Alt + Shift + F`（最終選択タグのお気に入り切替）
- 方針:
  - 機能追加・仕様変更なし
  - 編集中要素判定/テキスト選択判定などの共通ガードをフック側へ集約
  - 既存テスト（96件）とビルド通過を完了条件として回帰を抑止

20. App分割リファクタ（フェーズ7）
- `useNgAndFavoriteActions` を新設し、`App.jsx` の運用系操作ハンドラ群を分離
- 抽出対象:
  - NGワード追加/削除/全削除/グループ更新
  - NGフィルタ切替、NG登録モード切替
  - お気に入り表示モード切替
  - ランダム選択モード切替（NGモードとの相互排他処理を含む）
  - エントリ起点のNG登録トグル
  - World/Contextカテゴリ切替
- 方針:
  - 機能追加・仕様変更なし
  - UIイベントハンドラ責務をフックへ集約し、`App.jsx` の責務を縮小
  - 既存テスト（96件）とビルド通過を完了条件として回帰を抑止

21. App分割リファクタ（フェーズ8）
- `useModelProfileActions` を新設し、モデルプロファイル関連の操作ハンドラを分離
- 抽出対象:
  - フォーマット切替時のDialect同期
  - Dialect変更時のフォーマット同期
  - モデルプロファイル適用（dirty時の確認導線を維持）
  - 現在内容のモデルプロファイル上書き保存
- 方針:
  - 機能追加・仕様変更なし
  - プロファイル運用に関する分岐ロジックをフックへ集約
  - 既存テスト（96件）とビルド通過を完了条件として回帰を抑止

22. App分割リファクタ（フェーズ9）
- `useNsfwPresetActions` を新設し、SFW/NSFW安全制御関連の操作ハンドラを分離
- 抽出対象:
  - SFWのみ表示トグル時の18歳確認導線
  - NSFWプリセット読み込み確認ダイアログ導線
  - NSFWプリセットCSVの読み込み・辞書マージ処理
  - NSFW未読込状態での自動プリセット読込トリガー
- 方針:
  - 機能追加・仕様変更なし
  - 安全制御ロジックをフックへ集約し、`App.jsx` の責務をさらに縮小
  - 既存テスト（96件）とビルド通過を完了条件として回帰を抑止

23. App分割リファクタ（フェーズ10）
- `useSettingsBackupActions` を新設し、設定バックアップ/復元/リセット関連の操作ハンドラを分離
- 抽出対象:
  - 設定JSONエクスポート
  - 設定JSONインポート（確認ダイアログ導線含む）
  - リセット対象選択の更新ロジック
  - 保存設定の一括/選択リセット実行とトースト通知
- 方針:
  - 機能追加・仕様変更なし
  - データ管理系副作用をフックへ集約し、`App.jsx` の責務をさらに縮小
  - 既存テスト（96件）とビルド通過を完了条件として回帰を抑止

24. App分割リファクタ（フェーズ11）
- `useAppBootstrapEffects` を新設し、起動時ブートストラップ副作用を `App.jsx` から分離
- 抽出対象:
  - デフォルトCSV初回読込（`default_tags_v1.csv` / `default_tags.csv` のフォールバック）
  - シード辞書判定（初期辞書一致 / 旧シード部分集合）と置換可否判定
  - 辞書スナップショット比較による不要置換回避
  - `overlapBehavior` 旧値マイグレーション
  - `showToastRef` 同期
- 方針:
  - 機能追加・仕様変更なし
  - 起動時副作用の責務をフックへ集約し、`App.jsx` の可読性と追跡性を向上
  - 既存テスト（96件）とビルド通過を完了条件として回帰を抑止

25. App分割リファクタ（フェーズ12）
- `useAppSynchronizationEffects` を新設し、同期系副作用を `App.jsx` から分離
- 抽出対象:
  - カスタムカテゴリ/リセットカテゴリの整合チェック
  - NGワードおよびNGグループの正規化
  - NGグループフィルタ値の整合
  - Dialect <-> フォーマットの同期
  - 言語変更時のテキストブロックメタ再正規化
  - Main/World/Context選択値の候補整合
  - SFW時のNSFW取り込みフラグ抑止
  - 辞書更新時の選択・挿入ターゲット整合
- 方針:
  - 機能追加・仕様変更なし
  - 同期副作用をフックへ集約し、`App.jsx` の責務をさらに縮小
  - 既存テスト（96件）とビルド通過を完了条件として回帰を抑止

26. App分割リファクタ（フェーズ13）
- `useWorkspaceDerivedData` を新設し、ワークスペース派生データ算出を `App.jsx` から分離
- 抽出対象:
  - 選択タグの多言語ラベル導出（ポジ/ネガ）
  - 検査対象タグと削除可否判定
  - ライブプレビュー文字列の組み立て
  - ポジ/ネガのコメント付きバンドル出力生成
  - ランダムビルダープレビュー（Dialect反映）
  - プロンプト警告統合とブロック一致ハイライト計算
  - 追加先表示テキストとプレビュー対象切替ハンドラ
  - Main/World/Context/Type の単一選択値導出
- 方針:
  - 機能追加・仕様変更なし
  - 派生ロジックをフックへ集約し、`App.jsx` の責務をさらに縮小
  - 既存テスト（96件）とビルド通過を完了条件として回帰を抑止

27. App分割リファクタ（フェーズ14）
- `useAppLocalization` を新設し、`App.jsx` のi18n/法務導線の派生処理を分離
  - `messageBundle` 解決、翻訳関数 `t` の生成
  - 問い合わせURL導出、法務モーダル文面導出
- `useCustomConfirmActions` を新設し、カスタム確認ダイアログ操作を分離
  - `requestCustomConfirm` / `closeCustomConfirm` / `confirmCustomConfirm`
- `usePromptCompositionState` を新設し、モデルプロファイル合成時の派生状態を分離
  - `formatOptions` / `dialectProfile`
  - 選択ID集合（ポジ/ネガ/ランダム）
  - `base + user` 合成と dirty 判定
- 方針:
  - 機能追加・仕様変更なし
  - 表示/翻訳/確認導線/合成状態の責務をフックへ集約し、`App.jsx` の可読性と保守性を向上
  - 既存テスト（96件）とビルド通過を完了条件として回帰を抑止

28. App分割リファクタ（フェーズ15）
- `useSectionCallbacks` を新設し、`App.jsx` の JSX 内インラインコールバック群を分離
- 分離対象:
  - PromptOutputsSection 向け操作（適用/コピー/追加先切替/選択タグ削除）
  - ExtraInputsSection 向け操作（ブロック削除確認）
  - WorkspaceSection 向け操作（解除系、検索クリア、NG操作、単一選択値操作、タグ削除確認、ライブコピー、inspected操作）
  - DataManagement/Legal 向け操作（保存設定リセット要求、法務モーダル開閉）
- テスト調整:
  - `src/App.test.jsx` の `renderAppReady` に `プリセット` の待機を追加し、遅延ロード描画のタイミング揺らぎを吸収
- 方針:
  - 機能追加・仕様変更なし
  - セクション配線責務をフックへ寄せ、`App.jsx` の保守性を向上
  - 既存テスト（96件）とビルド通過を完了条件として回帰を抑止

29. App分割リファクタ（フェーズ16）
- `src/lib/appStateUtils.js` を新設し、`App.jsx` のローカルユーティリティ関数を分離
- 分離対象:
  - モバイル幅に応じた初期折りたたみ判定
  - アプリ別デフォルトDialect解決
  - Dialectからフォーマット種別解決
  - 追加先ターゲットのdeserialize（旧保存形式の後方互換を維持）
  - テキストブロックdeserialize
  - NG語/NGグループ正規化
- 方針:
  - 機能追加・仕様変更なし
  - ユーティリティ責務をApp外へ分離し、再利用性と可読性を向上
  - 既存テスト（96件）とビルド通過を完了条件として回帰を抑止

30. App分割リファクタ（フェーズ17）
- `useAppPersistentState` を新設し、`App.jsx` の永続state定義（`usePersistentState` 群）を集約
- 抽出対象:
  - 辞書/選択/フォーマット/言語テーマ
  - 入力ブロック/追加先/折りたたみ状態
  - プリセット/モデルプロファイル/SFW・安全表示
  - NGワード/お気に入り/サブフォルダカテゴリ運用
- 方針:
  - 機能追加・仕様変更なし
  - state定義責務をフックへ寄せ、`App.jsx` の可読性・保守性を向上
  - 既存テスト（96件）とビルド通過を完了条件として回帰を抑止

31. App分割リファクタ（フェーズ18）
- `useAppFileRefs` を新設し、`App.jsx` のファイル入力/ディレクトリ入力 `ref` 宣言を集約
- `useAppCoreDerivedState` を新設し、基礎派生値（辞書Map/NSFW存在判定/安全表示モード解決）を集約
- 方針:
  - 機能追加・仕様変更なし
  - refと基礎派生値の責務を分離し、`App.jsx` の追跡性を向上
  - 既存テスト（96件）とビルド通過を完了条件として回帰を抑止

32. App分割リファクタ（フェーズ19）
- `useAppSectionProps` を新設し、`App.jsx` に残っていたセクションprops配線を分離
- 分離対象:
  - PromptOutputsSection / ExtraInputsSection / RandomBuilderSection
  - WorkspaceSection / DataManagementSection / LegalSection
  - ConfirmDialogs / LegalModal
- 方針:
  - 機能追加・仕様変更なし
  - 配線責務をフックへ寄せ、`App.jsx` の可読性と追跡性を向上
  - 既存テスト（96件）とビルド通過を完了条件として回帰を抑止

33. App分割リファクタ（フェーズ20）
- `src/components/sections/AppSectionsStack.jsx` を新設し、遅延ロードセクション描画スタックを `App.jsx` から分離
- 分離対象:
  - PromptOutputsSection / ExtraInputsSection / RandomBuilderSection
  - WorkspaceSection / DataManagementSection / LegalSection
  - セクションごとの Suspense fallback（スケルトン）
- 方針:
  - 機能追加・仕様変更なし
  - 画面描画配線の責務を専用コンポーネントへ集約し、`App.jsx` の可読性を向上
  - 既存テスト（96件）とビルド通過を完了条件として回帰を抑止

34. App分割リファクタ（フェーズ21）
- `src/components/sections/ToastOverlay.jsx` を新設し、トースト表示を `App.jsx` から分離
- `src/components/sections/PromptWarningsBanner.jsx` を新設し、警告バナー表示を `App.jsx` から分離
- 方針:
  - 機能追加・仕様変更なし
  - 表示専用UIをセクションコンポーネント化し、`App.jsx` の表示責務を縮小
  - 既存テスト（96件）とビルド通過を完了条件として回帰を抑止

35. App分割リファクタ（フェーズ22）
- `src/constants/appUiConstants.js` を新設し、`App.jsx` 先頭に集約されていたUI定数群を移設
- 移設対象:
  - プレビュー関連ID/モード定数
  - タグ/ソース/ソート/安全表示の選択肢定数
  - Ko-fi URL定数
  - 初期リセット選択定数
- 方針:
  - 機能追加・仕様変更なし
  - 定数責務を `constants/` に統一し、`App.jsx` の宣言ノイズを削減
  - 既存テスト（96件）とビルド通過を完了条件として回帰を抑止

36. App分割リファクタ（フェーズ23）
- `src/constants/appStorageConfig.js` を新設し、`App.jsx` 内の保存設定キー定数を移設
- 移設対象:
  - `SETTINGS_EXPORT_STORAGE_KEYS`
  - `BACKUP_NG_STORAGE_KEYS`
  - `BACKUP_SETTINGS_STORAGE_KEYS`
- `src/lib/runtimeFlags.js` を新設し、テスト実行環境判定フラグを分離
- 方針:
  - 機能追加・仕様変更なし
  - 保存設定/実行環境判定の責務をApp外へ移し、`App.jsx` の宣言責務を縮小
  - 既存テスト（96件）とビルド通過を完了条件として回帰を抑止

37. App分割リファクタ（フェーズ24）
- `src/components/sections/AppMainContent.jsx` を新設し、`App.jsx` の上位表示レイヤーを分離
- 分離対象:
  - TopBar表示
  - PromptWarningsバナー表示
  - セクションスタック表示
  - モバイル広告表示
- 併せて `availableDialects` の `useMemo` を削除し、`PROMPT_DIALECT_OPTIONS` を直接利用
- 方針:
  - 機能追加・仕様変更なし
  - 表示責務をさらにコンポーネントへ移し、`App.jsx` をオーケストレーション中心に整理
  - 既存テスト（96件）とビルド通過を完了条件として回帰を抑止

38. App分割リファクタ（フェーズ25）
- `useAppSectionWiring` を新設し、`useSectionCallbacks` と `useAppSectionProps` の配線合成を `App.jsx` から分離
- 抽出対象:
  - セクション操作ハンドラ群の生成呼び出し
  - セクションprops群の生成呼び出し
  - ハンドラ/propsの結線順序（callbacks -> props）をフック内部で固定
- 方針:
  - 機能追加・仕様変更なし
  - `App.jsx` 側は「必要stateを渡して結果を受け取る」形へ単純化
  - 既存テスト（96件）とビルド通過を完了条件として回帰を抑止

39. App分割リファクタ（フェーズ26）
- `useWorkspaceViewState` を新設し、`usePreviewSections` と `useWorkspaceDerivedData` の合成を `App.jsx` から分離
- 抽出対象:
  - プレビュー候補/プレビューセクション導出
  - ワークスペース派生データ導出（選択ラベル、警告、コピー文字列、単一選択値）
  - プレビュー状態からワークスペース派生計算へ渡す中間配線
- 方針:
  - 機能追加・仕様変更なし
  - `App.jsx` はワークスペース表示用の状態取得を1フック呼び出しに統合
  - 既存テスト（96件）とビルド通過を完了条件として回帰を抑止

40. App分割リファクタ（フェーズ27）
- `AppRootShell` を新設し、`App.jsx` 最終レンダリング層（Toast/MainContent/Dialog）を分離
- 抽出対象:
  - ルートコンテナ描画
  - Toast表示
  - `AppMainContent` の描画とprops受け渡し
  - ConfirmDialogs / LegalModal の描画
- 方針:
  - 機能追加・仕様変更なし
  - `App.jsx` は状態オーケストレーションに集中し、表示責務をさらに縮小
  - 既存テスト（96件）とビルド通過を完了条件として回帰を抑止

41. App分割リファクタ（フェーズ28）
- `useAppRootShellProps` を新設し、`App.jsx` のルート表示props組み立てを分離
- 抽出対象:
  - `AppRootShell` 向けpropsの命名変換（`handleX` -> `onX`）
  - ルート表示に必要なprops束ね処理
- 方針:
  - 機能追加・仕様変更なし
  - `App.jsx` の return 周辺を単純化し、表示用props配線責務をフックへ移管
  - 既存テスト（96件）とビルド通過を完了条件として回帰を抑止

42. App分割リファクタ（フェーズ29）
- `useDataImportRuntime` を新設し、データ取り込み系の2フック呼び出しを統合
- 抽出対象:
  - `useImportActions`
  - `useNsfwPresetActions`
  - 取り込み/安全制御ハンドラの統合返却
- 方針:
  - 機能追加・仕様変更なし
  - `App.jsx` のデータ取り込み関連配線責務を縮小
  - 既存テスト（96件）とビルド通過を完了条件として回帰を抑止

43. App分割リファクタ（フェーズ30）
- `useAppLifecycleEffects` を新設し、副作用系フック呼び出しを `App.jsx` から集約
- 抽出対象:
  - `useAppBootstrapEffects`
  - `useThemeMode`
  - `useAppSynchronizationEffects`
  - `useGlobalShortcuts`
- 方針:
  - 機能追加・仕様変更なし
  - ライフサイクル副作用の呼び出し責務を1フックへ集約し、`App.jsx` の追跡性を改善
  - 既存テスト（96件）とビルド通過を完了条件として回帰を抑止

44. App分割リファクタ（フェーズ31）
- `usePromptActions` / `useNgAndFavoriteActions` の戻り値を分解代入せず、`promptActions` / `ngActions` オブジェクトで扱う形へ整理
- `useAppSectionWiring` とショートカット連携に渡す依存を、必要箇所だけ `promptActions.*` / `ngActions.*` 参照へ統一
- 方針:
  - 機能追加・仕様変更なし
  - `App.jsx` のローカル識別子数と分解代入行数を削減し、オーケストレーション可読性を改善
  - 既存テスト（96件）とビルド通過を完了条件として回帰を抑止

45. App分割リファクタ（フェーズ32）
- `usePromptCompositionState` と `useWorkspaceViewState` の戻り値を、分解代入ではなく `promptComposition` / `workspaceView` オブジェクト運用へ整理
- `useAppSectionWiring` / `useAppLifecycleEffects` / `useAppRootShellProps` への依存注入を、`*.property` 参照に統一
- 方針:
  - 機能追加・仕様変更なし
  - `App.jsx` の分解代入行数と再組み立てコストを削減し、状態オーケストレーションを追いやすくする
  - 既存テスト（96件）とビルド通過を完了条件として回帰を抑止

46. App分割リファクタ（フェーズ33）
- `useAppSectionWiring` への依存注入を、個別マッピング列挙から `...promptComposition` / `...workspaceView` / `...promptActions` / `...ngActions` のスプレッド中心へ再配線
- 既存の明示引数は、セクション構成に必要な固定値・UI状態のみを残して簡素化
- 方針:
  - 機能追加・仕様変更なし
  - 巨大な引数オブジェクトの重複記述を削減し、`App.jsx` のオーケストレーション可読性を向上
  - 既存テスト（96件）とビルド通過を完了条件として回帰を抑止

## 4. 技術的な設計ポイント
- 単一責務: フォーマット処理、CSV処理、取り込み処理を `src/lib` に分離
- SSOT: カテゴリ語彙を `src/constants/categories.js` 系へ集約
- SSOT: モデルプロファイル定義を `src/constants/modelProfiles.js` へ集約
- 耐障害性: CSVエラー行はスキップしアプリ全体は継続
- 実務優先: 厳格エラーと正規化警告を使い分け、運用コストを低減
- Derived State: ベースプロンプトと手動タグを分離し、最終出力を派生計算

## 5. 主な機能一覧（対外説明向け）
- プロンプト構築: タグ選択、重み調整、重複制御、ロック
- 出力制御: A1111/ComfyUI + Dialect
- ワークスペース: 複数入力ブロック、ライブプレビュー、コピー
- データ連携: LoRA/Embedding/Wildcard、Civitaiメタデータ
- タグ化支援: LoRAキープロンプト/候補をモーダル経由で再利用タグ化
- 検索/表示: 3軸カテゴリフィルター、タイプ絞り込み、ソート
- 安全制御: SFWのみ表示、NSFW読込確認、18歳確認
- 品質管理: CSVバリデータ、テストCSVによる検証

## 6. 検証・品質保証
- CSV検証コマンド: `npm run validate:csv -- <csv-path>`
- テスト実行時は `default_tags.csv` の自動取得を抑止し、`jsdom` 由来の不要なfetchエラーノイズを回避
- CSV必須列欠落時の例外停止を廃止し、`SchemaError.MissingRequiredColumn` で安全終了する運用へ統一
- 改行コード（LF/CRLF/CR）混在入力を受理できるよう正規化を強化
- quoted multiline（セル内改行）/ 未閉じ引用符の疑い行を検知し、`SyntaxError.UnsupportedQuotedMultiline` でスキップ
- テスト資産:
  - 正常系: `public/data/csv/nsfw_test.csv`（互換用に `nsfw_test_csv` も残置）
  - 異常系: `public/data/csv/nsfw_error_test.csv`, `public/data/csv/sfw_error_test.csv`
  - 境界系: `public/data/csv/sfw&nsfw_test.csv`
  - ロジック境界: `modelImports` のSet注入（空Set優先）と `modelProfiles` のフォールバック警告文言を固定
- 運用方針:
  - エラー: スキップ
  - 警告: 正規化しつつ可視化
- フォーマッタ回帰:
  - `src/lib/promptFormatter.test.js` で `Map` 経路と互換挙動（先勝ち）を固定

## 7. 履歴書向けサマリー（短文）
「画像生成AI向けWebアプリ（React/Vite）を個人開発。A1111/ComfyUI差異吸収、LoRAメタデータ連携、CSV厳格バリデーション、モバイル最適化まで一貫実装し、実運用可能なプロンプト管理基盤を構築。」

## 8. 職務経歴書向け箇条書き（転記用）
- React + Vite + Tailwind によるSPA設計/実装
- A1111/ComfyUIの文法差を吸収するフォーマット層を実装
- LoRA/Embedding/Wildcard取込とCivitai系JSON解析を実装
- CSVバリデータ（構文/語彙/軸ルール）とテストフィクスチャ運用を整備
- 仮想スクロールとモバイルUI改善により大量データ運用性能を向上
- 安全表示モード（both/sfw/nsfw）とカテゴリ候補連動を実装し、実運用時の探索効率を改善

## 9. エントリーシート向け説明（約300字）
私は、画像生成AIの実運用で発生する「再現性の低さ」と「管理の煩雑さ」を解消するため、PromptPaletteを個人で開発しました。React/Viteで構築したSPAに、A1111/ComfyUIの文法差吸収、LoRA/Embeddingのフォルダ取込、Civitaiメタデータ解析を実装し、実務で使える管理体験を重視しました。さらにCSV仕様を定義してバリデータを実装し、正常系/異常系のテストデータで回帰検証できる運用を整えています。単なる機能追加ではなく、運用時のエラー耐性と保守性まで含めた設計を行った点が強みです。

## 10. 今後の拡張予定
- App状態のさらなる分離（Context/Zustand導入検討）
- ユーザー設定JSONの粒度別インポート/エクスポート
- お気に入り/NGワードの管理UI拡張
- 大規模辞書向けの追加最適化

## 11. 関連ドキュメント
- AI協業タイムライン（指示/実装/検証の時系列）: `docs/ai-collaboration-timeline-ja.md`
- CSV仕様書: `docs/csv-spec-v3.md`
- モデルプロファイル仕様: `docs/model-profiles-spec.md`
