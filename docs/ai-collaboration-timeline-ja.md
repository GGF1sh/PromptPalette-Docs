# PromptPalette 開発タイムライン（AI協業ログ）

このドキュメントは、PromptPaletteをどのような順序で設計・実装・検証し、品質を引き上げてきたかを時系列でまとめた記録です。  
履歴書、エントリーシート、面接説明、案件提案での実績説明にそのまま利用できるよう、**意思決定・実装・検証**の流れに分けています。

---

## 1. プロジェクトの目的
- 画像生成AI（A1111 / ComfyUI）向けのプロンプト構築を、GUI中心で再現性高く運用できるようにする
- LoRA / Embedding / Wildcard / CSV辞書を1つの作業画面で扱えるようにする
- SFW/NSFWを安全に分離しつつ、実運用で必要な切替・検証・保守性を確保する
- 開発期間は実働4日（設計・実装・検証・ドキュメント整備を同サイクルで実施）

---

## 2. AI協業の役割分担

### Codex（実装担当）
- コード変更、テスト実行、回帰確認、コミット/プッシュ
- 既存挙動を壊さない最小差分の実装
- 問題発生時のデバッグと再現手順の固定化

### Gemini / ChatGPT / Grok（レビュー・データ補助）
- 設計レビュー、改善案、リスク整理、仕様文章化
- CSVデータ作成補助（多言語ラベルや初期語彙）
- バリデーション観点の洗い出し（正常系 / 異常系）

### 最終意思決定（開発者）
- 機能優先度、UX方針、安全要件の確定
- 仕様の採否判断（厳格化するか、正規化で許容するか）
- リリース判断と運用ルールの確立
- 代表判断:
  - CSVエラーは例外停止せず「行スキップ + レポート返却」に統一
  - NSFWは分離運用し、手動読込 + 年齢確認を必須化
  - 互換性を優先し、破壊的変更は最小差分で段階導入

---

## 3. 開発フェーズ（時系列）

## フェーズA: 基盤構築（プロンプト編集）
- タグON/OFF、重み調整、ポジ/ネガ分離、複数自由入力欄
- A1111 / ComfyUI の文法差を吸収する整形ロジックを実装
- 出力コピーと警告表示を追加

成果:
- 実運用可能な基本ワークフローを確立

## フェーズB: 構造化と分離（保守性）
- `lib/` にロジックを集約（フォーマッタ、辞書、選択状態）
- `hooks/` と `components/` へ責務分離
- 永続化（localStorage）とUIロジックを切り離し

成果:
- App肥大化の抑制、回収しやすい構造へ移行

## フェーズC: 外部データ連携（LoRA/Embedding/Wildcard）
- フォルダ取り込み、サブフォルダ対応、重複抑制
- Stability Matrix / Civitai メタデータ解析（JSON）
- トリガーワード抽出、キープロンプト管理、タグ化導線

成果:
- 実データ投入に強い実務系機能を追加

## フェーズD: SFW/NSFW安全制御
- SFWのみ表示、18歳確認、NSFWプリセット手動読み込み
- 安全表示モード（SFW/NSFW/両方）など運用モードを分離
- UI上で誤操作しにくい導線へ改修

成果:
- 一般利用と成人向け利用を同居させる制御基盤を確立

## フェーズE: CSV仕様の厳格化（V3系）
- `Main / World / Context` の3軸カテゴリを定義
- 検証段階を明文化: Syntax -> Vocabulary -> Axis
- エラー時クラッシュ禁止（行スキップ + レポート）を徹底

成果:
- 辞書品質を機械的に担保できる運用に移行

## フェーズF: モデルプロファイル運用
- モード別（sfw/nsfw）プロファイル解決とフォールバック
- ベース文字列とユーザー入力の分離（Derived State）
- 上書き確認や未保存保護など運用事故対策を導入

成果:
- モデル切替時の事故を抑えた堅牢な状態遷移を実現

## フェーズG: 実運用デバッグと最適化
- ライト/ダーク可読性、モバイル崩れ、誤タップ防止の改善
- NGワード・お気に入り・ソート・フィルタ導線を改修
- 広告導入はコード要因と配信要因を分離して切り分け

成果:
- 機能だけでなく運用現場の使い勝手まで改善

## フェーズH: App分割リファクタ（段階対応）
- Appの肥大化対策を「機能追加なし」で段階実施する方針を採用
- フェーズ1として、検索/カテゴリフィルタ/NG入力関連のstateを `useDictionaryFilterState` に抽出
- フェーズ2として、主要セクションを `React.lazy` + `Suspense` で遅延ロード化し初期バンドルを分割
- フェーズ3として、`App.jsx` の非永続 `useState` 群を `useAppTransientState` へ抽出
- フェーズ4として、辞書の派生計算（カテゴリ候補/NG正規化/フィルタ結果）を `useDictionaryDerivedData` へ抽出
- フェーズ5として、ライブプレビューの導出ロジック（候補/同期/セクション算出）を `usePreviewSections` へ抽出
- セクション単位のfallbackを追加し、遅延読込時の体感劣化を抑制
- 既存テスト・ビルドを毎段階で通しながらリグレッションを抑止

成果:
- 責務分離を進めつつ、既存挙動を維持したまま保守性を改善

## フェーズI: App分割リファクタ（ショートカット副作用の分離）
- `useGlobalShortcuts` を追加し、`App.jsx` のグローバルキーボードイベント購読をフックへ移管
- `Ctrl/⌘ + C`, `Ctrl/⌘ + Alt + N`, `Ctrl/⌘ + Alt + F`, `Ctrl/⌘ + Alt + Shift + F` の4系統を統合管理
- 編集中要素判定/テキスト選択判定などの安全条件を共通化し、重複コードを削減

成果:
- ショートカット挙動を変えずに副作用責務を縮小し、`App.jsx` の可読性とメンテナンス性を改善

## フェーズJ: App分割リファクタ（運用系ハンドラの分離）
- `useNgAndFavoriteActions` を追加し、`App.jsx` に散在していた運用系操作（NG/お気に入り/モード切替）をフックへ移管
- NGワード管理、NG/お気に入りモード、ランダムモード相互排他、World/Context切替を一箇所に集約

成果:
- 既存挙動を維持しつつ、`App.jsx` のイベントハンドラ密度を下げて追跡性を向上

## フェーズK: App分割リファクタ（モデルプロファイル操作の分離）
- `useModelProfileActions` を追加し、モデルプロファイル適用/保存とフォーマット連動ハンドラをフックへ移管
- `dirty` 判定時の上書き確認やカテゴリ適用ロジックなど、分岐の多い運用処理を集約

成果:
- モデルプロファイル運用の責務を分離し、`App.jsx` の可読性と変更容易性を向上

## フェーズL: App分割リファクタ（SFW/NSFW安全制御の分離）
- `useNsfwPresetActions` を追加し、SFW/NSFWの運用系ハンドラを `App.jsx` から分離
- SFW解除時の18歳確認、NSFWプリセット読込確認、CSV読込マージ、自動読込トリガーを集約
- NSFWプリセット取り込みの重複排除ロジックをフック内部へ移し、`App.jsx` のロジック密度を低下

成果:
- 安全制御関連の副作用責務をフックに集約し、`App.jsx` の追跡性と保守性を改善

## フェーズM: App分割リファクタ（設定バックアップ/復元/リセットの分離）
- `useSettingsBackupActions` を追加し、設定JSONのエクスポート/インポートと保存設定リセット処理を `App.jsx` から分離
- インポート時の確認導線、リセット対象バリデーション、選択リセットの通知ロジックをフック側へ集約
- 既存の挙動（クラッシュさせない・確認付き適用・選択対象のみ削除）を維持したまま責務を整理

成果:
- データ管理系の副作用が局所化され、`App.jsx` から大きな手続きブロックを削減
- 次フェーズでのさらなる分離（法務モーダル/カスタム確認管理など）に進める土台を確保

## フェーズN: App分割リファクタ（起動時副作用の分離）
- `useAppBootstrapEffects` を追加し、起動時の副作用を `App.jsx` から分離
- 分離した責務:
  - シード辞書判定に基づくデフォルトCSVブートストラップ
  - `default_tags_v1.csv` / `default_tags.csv` のフォールバック読込
  - スナップショット比較に基づく不要置換の回避
  - `overlapBehavior` 旧値マイグレーション
  - トースト参照の同期

成果:
- 初期化処理の追跡性が向上し、`App.jsx` の責務をさらに縮小
- 機能変更なしのまま、次フェーズでの状態管理統合を進めやすい構造へ改善

## フェーズO: App分割リファクタ（同期系副作用の分離）
- `useAppSynchronizationEffects` を追加し、同期系の `useEffect` 群を `App.jsx` から分離
- 分離した責務:
  - カスタムカテゴリ/リセットカテゴリの整合処理
  - NGワード・NGグループの正規化
  - NGグループフィルタの整合
  - Dialectとフォーマットの双方向同期
  - 言語変更時のテキストブロックメタ正規化
  - Main/World/Context選択値の候補整合
  - SFW時のNSFW取り込みフラグ抑止
  - 辞書更新時の選択状態/挿入ターゲット整合

成果:
- `App.jsx` の同期副作用責務を局所化し、読みやすさと変更追跡性を改善
- 機能追加なしで、次フェーズの分割継続（状態管理整理）に進みやすい状態を確保

## フェーズP: App分割リファクタ（ワークスペース派生データの分離）
- `useWorkspaceDerivedData` を追加し、ワークスペース関連の派生計算を `App.jsx` から分離
- 分離した責務:
  - 選択中タグの表示ラベル導出（多言語）
  - inspected entry と削除可否判定
  - ライブプレビュー/バンドル出力文字列の組み立て
  - ランダムビルダーのプレビュー文字列組み立て
  - プロンプト警告統合、ブロック一致ハイライト
  - 追加先表示文字列とプレビュー対象切替ハンドラ
  - 単一選択値（Main/World/Context/Type）導出

成果:
- `App.jsx` の派生ロジック密度を下げ、表示責務への集中を促進
- 機能追加なしで、次フェーズの状態管理再編に進める土台を強化

## フェーズQ: App分割リファクタ（i18n/確認導線/プロンプト合成派生状態の分離）
- `useAppLocalization` を追加し、`App.jsx` から以下を分離
  - 言語ごとの message bundle 解決
  - 翻訳関数 `t` の生成
  - 問い合わせURL導出
  - 法務モーダル文面導出
- `useCustomConfirmActions` を追加し、カスタム確認ダイアログの操作を分離
  - `requestCustomConfirm` / `closeCustomConfirm` / `confirmCustomConfirm`
- `usePromptCompositionState` を追加し、モデルプロファイル適用時の派生状態を分離
  - `formatOptions` / `dialectProfile`
  - 選択ID集合（ポジ/ネガ/ランダム）
  - ベース+手動タグ合成（`base + user`）と dirty 判定

成果:
- 翻訳・確認導線・合成状態の責務を `App.jsx` から分離し、可読性と保守性を向上
- 挙動変更なしで既存テスト/ビルドを維持し、次フェーズの分割を継続可能な状態を維持

## フェーズR: App分割リファクタ（セクション操作コールバックの分離）
- `useSectionCallbacks` を追加し、`App.jsx` の JSX 内インラインコールバック群を分離
- 分離対象:
  - PromptOutputsSection 向けの適用/コピー/追加先切替/ロック保護削除
  - ExtraInputsSection 向けのブロック削除確認
  - WorkspaceSection 向けの解除系/検索クリア/NG操作/単一選択操作/タグ削除確認/ライブコピー/inspected操作
  - DataManagement/Legal 向けのリセット要求と法務モーダル導線
- 遅延ロードの描画タイミング揺らぎに備えて、`App.test.jsx` の ready待機条件を強化

成果:
- セクション配線責務をフックへ寄せ、`App.jsx` 側の役割を「状態オーケストレーション中心」に寄せた
- 機能仕様を変えずに回帰テストとビルドを維持

## フェーズS: App分割リファクタ（App内ユーティリティのlib分離）
- `src/lib/appStateUtils.js` を追加し、`App.jsx` に残っていたローカルユーティリティ群を分離
- 分離対象:
  - 初期折りたたみ判定
  - アプリ別デフォルトDialect解決
  - Dialect逆引きによるフォーマット解決
  - 追加先ターゲットdeserialize（旧形式後方互換含む）
  - テキストブロックdeserialize
  - NG語/グループ正規化

成果:
- `App.jsx` 内の補助関数群を削減し、表示/状態オーケストレーション責務へ集中
- 機能仕様を変えずに回帰テストとビルドを維持

## フェーズT: App分割リファクタ（永続state定義の分離）
- `useAppPersistentState` を追加し、`App.jsx` の `usePersistentState` 群をフックへ集約
- 分離対象:
  - 辞書、フォーマット、言語/テーマ、選択タグ
  - 入力ブロック、追加先、UI折りたたみ状態
  - プリセット、モデルプロファイル、SFW/安全表示制御
  - NGワード、NGグループ、お気に入り、サブフォルダカテゴリ運用

成果:
- 機能仕様を変えず、`App.jsx` の責務をさらに縮小
- state宣言の見通しを改善し、次フェーズの分割/検証を継続しやすい構造へ更新

## フェーズU: App分割リファクタ（ref宣言と基礎派生stateの分離）
- `useAppFileRefs` を追加し、ファイル/ディレクトリ入力の `ref` 宣言を `App.jsx` から分離
- `useAppCoreDerivedState` を追加し、コア派生値を分離
  - `dictionaryById`
  - `hasNsfwEntries`
  - `effectiveSafetyViewMode`

成果:
- 機能仕様を変えずに、`App.jsx` の宣言密度を低下
- 入力参照と基礎派生値の責務をフックへ集約し、追跡性を改善

## フェーズV: App分割リファクタ（セクションprops配線の分離）
- `useAppSectionProps` を追加し、`App.jsx` 内で巨大化していたセクションprops配線を分離
- 分離対象:
  - PromptOutputsSection / ExtraInputsSection / RandomBuilderSection
  - WorkspaceSection / DataManagementSection / LegalSection
  - ConfirmDialogs / LegalModal
- `App.jsx` は `<Section {...sectionProps} />` 形式へ置換し、表示責務に集中できる構造へ整理

成果:
- 機能仕様を変えず、配線コードの責務をフックへ集約
- `App.jsx` の可読性と追跡性をさらに改善

## フェーズW: App分割リファクタ（遅延ロード描画スタックの分離）
- `src/components/sections/AppSectionsStack.jsx` を追加し、遅延ロードセクション描画を `App.jsx` から分離
- 分離対象:
  - PromptOutputsSection / ExtraInputsSection / RandomBuilderSection
  - WorkspaceSection / DataManagementSection / LegalSection
  - セクションごとの `Suspense` fallback定義
- 方針:
  - 機能追加・仕様変更なし
  - 画面描画配線責務を専用コンポーネントへ寄せ、`App.jsx` は状態オーケストレーションに集中
  - 既存テスト/ビルドを通して回帰を抑止

成果:
- 遅延ロード描画の責務を局所化し、`App.jsx` の可読性をさらに改善
- フェーズ2で導入した遅延ロード設計を、表示レイヤーとして独立管理できる状態へ移行

## フェーズX: App分割リファクタ（トースト/警告表示の分離）
- `src/components/sections/ToastOverlay.jsx` を追加し、トースト表示を `App.jsx` から分離
- `src/components/sections/PromptWarningsBanner.jsx` を追加し、プロンプト警告表示を `App.jsx` から分離
- 方針:
  - 機能追加・仕様変更なし
  - 表示専用UIをセクション化し、`App.jsx` の表示責務を削減
  - 既存テスト/ビルドを通して回帰を抑止

成果:
- トーストと警告バナーの表示ロジックを局所化し、`App.jsx` の可読性を向上
- 以後のUI微調整時に、表示コンポーネント単位で変更可能な構造へ改善

## フェーズY: App分割リファクタ（UI定数の分離）
- `src/constants/appUiConstants.js` を追加し、`App.jsx` 先頭のUI定数群を `constants/` へ移設
- 分離対象:
  - プレビュー関連ID/モード定数
  - タグタイプ/ソース/ソート/安全表示の選択肢
  - Ko-fi URL定数
  - 初期リセット選択定数
- 方針:
  - 機能追加・仕様変更なし
  - 定数責務を `constants/` に寄せ、`App.jsx` の宣言密度を低減
  - 既存テスト/ビルドを通して回帰を抑止

成果:
- `App.jsx` の冒頭宣言ブロックを縮小し、読み始めの把握コストを低下
- 以後の定数調整を `constants` で完結できる構造へ改善

## フェーズZ: App分割リファクタ（保存設定定数・実行環境フラグの分離）
- `src/constants/appStorageConfig.js` を追加し、保存設定キー群の導出定数を `App.jsx` から分離
- `src/lib/runtimeFlags.js` を追加し、`IS_TEST_RUNTIME` 判定を `App.jsx` から分離
- 方針:
  - 機能追加・仕様変更なし
  - 保存設定/実行環境判定の責務をApp外へ移し、`App.jsx` の宣言責務をさらに縮小
  - 既存テスト/ビルドを通して回帰を抑止

成果:
- `App.jsx` の冒頭定数をさらに削減し、依存責務を `constants` / `lib` へ整理
- テスト実行環境判定を再利用可能なユーティリティとして独立化

## フェーズAA: App分割リファクタ（上位表示レイヤーの分離）
- `src/components/sections/AppMainContent.jsx` を追加し、`App.jsx` から上位表示レイヤーを分離
- 分離対象:
  - TopBar表示
  - PromptWarningsバナー表示
  - セクションスタック表示
  - モバイル広告表示
- `availableDialects` の `useMemo` を廃止し、`PROMPT_DIALECT_OPTIONS` を直接利用
- 方針:
  - 機能追加・仕様変更なし
  - 画面構成責務をコンポーネントへ移し、`App.jsx` を状態オーケストレーション中心に整理
  - 既存テスト/ビルドを通して回帰を抑止

成果:
- `App.jsx` のJSX密度を削減し、表示構造の見通しを改善
- 以後のTopBar/警告/広告レイアウト調整を `AppMainContent` 単位で変更可能化

## フェーズAB: App分割リファクタ（セクション配線合成の分離）
- `useAppSectionWiring` を追加し、`App.jsx` で直列実行していた
  - `useSectionCallbacks(...)`
  - `useAppSectionProps(...)`
  の2段配線をフック内部へ統合
- `callbacks -> sectionProps` の依存順を `useAppSectionWiring` に固定し、`App.jsx` 側は結果受け取りに専念
- 既存挙動を変えず、配線責務だけを段階的に外出し

成果:
- `App.jsx` の配線密度をさらに低減し、次フェーズのstate分割を進めやすい構造へ更新

## フェーズAC: App分割リファクタ（ワークスペース配線合成の分離）
- `useWorkspaceViewState` を追加し、`App.jsx` で分離済みだった
  - `usePreviewSections(...)`
  - `useWorkspaceDerivedData(...)`
  の2段配線を1フックへ統合
- プレビュー導出結果をワークスペース派生計算へ受け渡す責務をフック内へ移管
- `App.jsx` 側はワークスペース関連の取得を単一呼び出しで完結

成果:
- プレビュー関連配線の追跡コストを削減し、`App.jsx` の可読性をさらに改善

## フェーズAD: App分割リファクタ（ルート表示層の分離）
- `AppRootShell` を追加し、`App.jsx` に残っていた最終表示責務（Toast/MainContent/Dialog）を分離
- `App.jsx` の return を `AppRootShell` 1コンポーネント呼び出しへ置換
- 表示責務を `components/sections` 側に寄せ、`App.jsx` を状態オーケストレーション中心へ寄せる

成果:
- `App.jsx` のJSX密度をさらに削減し、今後の状態分離フェーズでの変更影響範囲を縮小

## フェーズAE: App分割リファクタ（ルートprops組み立ての分離）
- `useAppRootShellProps` を追加し、`App.jsx` の return 手前に残っていたprops組み立てをフックへ移管
- `handleX` 系ハンドラの `onX` 変換をフック側へ集約し、`App.jsx` 側は `AppRootShell` 呼び出しを単純化

成果:
- ルート表示配線の見通しを改善し、`App.jsx` 末尾の責務をさらに縮小

## フェーズAF: App分割リファクタ（データ取り込みランタイム配線の分離）
- `useDataImportRuntime` を追加し、`useImportActions` と `useNsfwPresetActions` の呼び出し配線を統合
- 取り込み系ハンドラとNSFWプリセット導線を1つのフック返却へ集約

成果:
- `App.jsx` のデータ取り込み関連の責務を縮小し、取り込み運用ロジックの追跡性を改善

## フェーズAG: App分割リファクタ（ライフサイクル副作用呼び出しの集約）
- `useAppLifecycleEffects` を追加し、`App.jsx` に散在していた副作用系フック呼び出しを集約
  - `useAppBootstrapEffects`
  - `useThemeMode`
  - `useAppSynchronizationEffects`
  - `useGlobalShortcuts`
- 副作用の実行順（bootstrap -> theme -> synchronization -> shortcuts）を専用フックで固定

成果:
- `App.jsx` の副作用呼び出し責務を縮小し、状態オーケストレーションの見通しを改善

## フェーズAH: App分割リファクタ（アクション戻り値のオブジェクト運用化）
- `usePromptActions` と `useNgAndFavoriteActions` の戻り値を、分解代入ではなく `promptActions` / `ngActions` として保持
- `useAppSectionWiring` とショートカット連携の依存注入を `promptActions.*` / `ngActions.*` 参照に統一

成果:
- `App.jsx` のローカル変数展開を圧縮し、呼び出し配線の追跡性を改善

## フェーズAI: App分割リファクタ（合成/表示状態のオブジェクト運用化）
- `usePromptCompositionState` と `useWorkspaceViewState` の戻り値を `promptComposition` / `workspaceView` オブジェクトとして保持
- セクション配線とルートprops組み立てで、個別変数ではなくオブジェクト参照で依存を注入

成果:
- `App.jsx` の分解代入を削減し、表示/合成状態の利用箇所を追跡しやすい構造へ整理

## フェーズAJ: App分割リファクタ（セクション配線引数の圧縮）
- `useAppSectionWiring` 呼び出しの依存注入を、個別列挙中心から `promptComposition` / `workspaceView` / `promptActions` / `ngActions` スプレッド中心へ移行
- 固定UI状態と可変オブジェクト依存を分離し、配線コードの重複を削減

成果:
- `App.jsx` の巨大引数ブロックを縮小し、改修時の変更差分とレビュー負荷を低減

---

## 4. テスト戦略の進化（時系列）

## 4.1 単体テスト
- `src/lib/*.test.js` を中心に、変換ロジックと分類ロジックを固定
- フォーマット差、重複処理、辞書整合を回帰検証

## 4.2 CSV検証（fixture運用）
- 正常系: `public/data/csv/nsfw_test.csv`（互換用に `nsfw_test_csv` も残置）
- 異常系: `public/data/csv/nsfw_error_test.csv`, `public/data/csv/sfw_error_test.csv`
- 境界系: `public/data/csv/sfw&nsfw_test.csv`

運用:
- `npm run validate:csv -- <path>` で事前検証
- 問題行はクラッシュさせずスキップ、レポートで追跡

## 4.3 統合テスト
- `src/App.test.jsx` で主要導線を固定
- クリア系、重複時ルール、上書き確認など事故ポイントを検証

---

## 5. 直近の品質強化（抜粋）
- CSV必須列欠落時の `throw` を廃止し、レポート返却へ統一
- CR-only 改行を受理できるよう改行正規化を拡張
- セル内改行（quoted multiline）と未閉じ引用符の疑い行を `SyntaxError.UnsupportedQuotedMultiline` で安全にスキップ
- model import の重複判定に外部Set注入を追加（上位キャッシュ前提）
- model profile フォールバックログを具体化（どのキーへ落ちたかを明示）
- `promptFormatter` の辞書探索を Map化し、`find` の反復コストを削減
- Map化時も重複IDは「先勝ち」を維持し、既存出力との後方互換を保持
- フォーマッタに null/空配列ガードを追加し、異常入力時のクラッシュ耐性を強化

---

## 6. なぜこの記録が武器になるか
- ただ「作った」ではなく、**設計判断の理由**を説明できる
- ただ「動く」ではなく、**回帰を防ぐ運用**を示せる
- AI利用を丸投げではなく、**役割分担して品質管理した実績**として示せる

面接・提案時に伝える要点:
- 仕様の厳格化と柔軟性（正規化/厳格チェック）のトレードオフを管理した
- エラーを例外で落とさず、報告可能な形式へ統一した
- UI/UX、データ品質、実装保守性を並行して改善した

---

## 7. 次の改善候補
- 大規模辞書表示の仮想スクロール強化（必要なら）
- テストfixture拡張（BOM/CRLF/LF混在、引用符境界）
- 運用ログの可視化（画面側サマリー表示）
