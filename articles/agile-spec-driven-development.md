---
title: AIの「記憶喪失」を防ぐ技術：AIにドキュメントを書かせて開発効率を上げる工夫
emoji: "🧠"
type: "idea"
topics: ["ai","programming", "memory_doc"]
published: false
---
## はじめに

:::message alert
本記事には一部、AIにより生成された文章が含まれます。
:::

:::message
- **想定対象読者**: AIコーディングエージェントを用いた開発をするプログラマーやSE
- **前提知識**: 
  - プログラミング経験：中級程度
  - AI利用経験：Claude Code、Codex CLI、Gemini CLIなどのAIコーディングエージェントを使ったことがある
:::

## tl;dr

Claude CodeなどのAIコーディングエージェントを使った開発を効率化するために、「AI向けにもドキュメント作ろう。そのドキュメント作成もAIにやらせよう」と言う話。どのみちプログラマーは`/explain`コマンドを使うの見越して。

---

多くのプログラマーはAIに質問しながらコーディングするので、AIコーディングエージェント（以下「AI」と表記）が理解できる仕様を自然言語でドキュメント化して、AIに渡すことがプログラマーとAI両方の認知負荷を下げることにつながると考えました。  
ここではこの手法および対象のファイル群を「Memory Doc」と呼びます。

## Memory docを使った開発プロセスの基本ループ
1. 要件定義書とDesign docは人間が作成
   - AIが読みやすいようにMarkdown化
2. コード生成  
   - AIにプロンプトと要件定義書、Design docのMarkdown版を渡してソースコードを生成
3. AIによる反復的なMemory doc（仮）の更新
   - **重要**：**テスト通過後に行う**
   - プロンプトによってプロジェクト要約をAIに作らせる。Serena-mcp的な仕事。このへんは`AGENT.md`や`CLAUDE.md`に`/update-memory`というコマンドを定義しておいて、プルリクエストの前に実行すると良さそう
   - このドキュメントはMarkdown形式で作成させ、プロジェクト内でAIにコーディングさせるためのコンテキスト外部記憶として利用
   - Memory docの統一・バージョン管理は、リポジトリ直下に`spec/`ディレクトリを置き、リポジトリ内で管理して解決する
   - **重要**：**このドキュメントは人間が閲覧・編集可能**
   - スプリント単位など、開発のマイルストーン時にレビューする
5. 改善ループ
   - Memory docを参照させながらAIに追加機能や修正をプロンプトとして渡す
   - コード編集 → AIによるMemory doc更新を繰り返す
6. 仕様変更時
   - アジャイル開発的に仕様変更が発生した時は、改善ループ中に要件定義書や仕様書、Design docに人間が手を加える
   - その内容を再度AIに読ませ、Memory docを更新させて、改善ループに戻る
   - これらのドキュメントはリポジトリ内の`spec/`に持たせておくとなお良し

:::details 基本ループの図
```mermaid
flowchart LR
    A[要件定義書とDesign docを人間が作成]
    A --> B[AIに要約させMemory docを作成/更新]
    B --> C[AIに質問してコード生成]
    C --> D[テスト通過]
    D --> E[コードを元にAIがMemory doc編集]
    E --> F[マージ前の人間によるコードレビュー]
    F -- OK --> G[マージ/完了]
    F -- 仕様変更発生 --> B
```
:::

:::details `/update-memory`コマンドの例
```txt
既存のmemory docを読み直し、以下の点に注意して更新せよ：
- 既存のmemory docのファイル名に該当する内容だけ編集せよ
- 削除されたファイル/コンポーネントは完全に消せ
- 変更された型定義は正確に反映せよ
- **重要**: ファイルに出力する内容が2000トークンを超える場合、ファイルを分割しそれぞれに適切なファイル名をつけよ
- **重要**: テストコードがあればテストコードを正とし、テストコードと実装に乖離があれば備考としてメモせよ
```
:::

:::details Memory doc `components.md` の例
# コンポーネント詳細

## App.vue（ルートコンポーネント）

メインアプリケーションUI（~1500行）

### テンプレート構造
1. **ヘッダーツールバー**
   - 科目設定ボタン（cog）
   - 期間選択ボタン（calendar）
   - プレビューボタン（eye）
   - インポート/エクスポートボタン
   - 検索バー

2. **スプレッドシート**
   - jexcel統合
   - 5列: 日付、金額、借方科目、貸方科目、摘要
   - レスポンシブ列幅

3. **モーダル群**

### 主要state
```typescript
entries: Entry[]              // 全仕訳
filteredEntries: Entry[]      // 絞込み結果
searchQuery: string           // 検索文字列
creditAccounts: string[]      // 貸方科目リスト
debitAccounts: string[]       // 借方科目リスト
openingBalances: OpeningBalance[]
startDateFilter: string
endDateFilter: string
```

### 主要メソッド
| メソッド | 機能 |
|---------|------|
| `loadAccountSettings(）` | 科目設定をDBから読込 |
| `loadEntries(）` | 全仕訳を読込 |
| `applyFilters(）` | 検索+期間フィルター適用 |
| `initJexcel(）` | スプレッドシート初期化 |
| `onAccountSettingsSave(）` | 科目設定保存 |
| `onPeriodSelected(）` | 期間フィルター適用 |
| `exportPdf(）` | PDF出力（元帳形式） |
| `downloadBackup(）` | JSONバックアップ出力 |
| `onImportFile(）` | JSONインポート |

---

## AccountSettingsModal.vue

科目と開始残高の設定モーダル

### Props
```typescript
isOpen: boolean
creditAccountsData: string[]
debitAccountsData: string[]
openingBalancesData: OpeningBalance[]
```

### Emits
- `update:isOpen` - モーダル表示制御
- `save` - 設定データ保存

### タブ構成
1. 貸方科目（textarea、改行区切り）
2. 借方科目（textarea、改行区切り）
3. 開始残高（`account,amount` またはJSON）

---

## PeriodSelectorDialog.vue

期間選択ダイアログ

### Props
- `isOpen: boolean`

### Emits
- `update:isOpen`
- `select: { startDate、endDate }`

### クイック選択
- 今年
- 今年度（4月〜3月）
- 今月
- 先月
- 直近30日
- カスタム日付入力

---

## FullscreenModal.vue

汎用フルスクリーンモーダル

### Props
```typescript
isOpen: boolean
title: string
```

### Emits
- `close`

### 特徴
- スロットによるコンテンツ挿入
- スクロール可能なコンテンツ領域
- z-index: 1000

---

## SpeechInput.vue（Beta）

Web Speech API音声入力

### 特徴
- 日本語（ja-JP）
- 録音トグルボタン
- トランスクリプト表示
- エラーハンドリング

### 状態
開発済みだが未統合
:::

---

## ドキュメント保守戦略

### 更新タイミング
- **Memory doc**: マージ前レビューの直前
- **要件定義書**: スプリント開始時に人間が更新
- **Design doc**: 大きなアーキテクチャ変更時のみ人間が更新

### 乖離防止策
- スプリント単位やリリース時にMemory doc軍を再生成しコミット

### 運用ルール
- あらかじめリポジトリの役割や機能単位でMemory docを分割しておく。以下は例
  - `spec/overview.md`
  - `spec/components.md`
  - `spec/utilities.md`
  - `spec/configuration.md`
  - `spec/database.md`
  - `spec/todo.md`
  - `spec/known_bugs.md`
- 1ファイルの上限を日本語で書かれたMarkdownファイルで3000～4000文字程度（2000トークン）を目安にする

---

## この手法が向いているケース

### 推奨
- 新規プロジェクト（1〜3ヶ月程度）
- チーム規模: 1〜3名
- 仕様が比較的明確な開発
- APIサーバー、CLIツール、Webアプリなど

### 要検討
- レガシーコード改修（既存ドキュメントとの整合性）
- 探索的プロトタイピング（仕様が流動的）
- 大規模チーム（ドキュメント保守負荷）
- リアルタイム性が求められる開発

### 非推奨
- セキュリティクリティカルな部分（AI生成コードの精査が必須）
- パフォーマンスチューニングが必要な箇所（人間の最適化が必要）

---

## 実践例：個人開発プロジェクトquokkabookでの運用

### プロジェクト概要
- 個人事業主向けモバイル簡易会計PWA
- 開発期間：5時間（現在進行中）
- 技術スタック：Vue.js、Dexie.js、PWA
- Memory doc構成：`spec/components.md`, `spec/database.md`など5ファイル

### 実際に起きたこと

**Good**
- `GitHub Copilot Chat`と`Claude Code`を単一プロジェクトで利用しても矛盾が発生しなかった。
- コンテキストのオーバーフローによる的はずれな回答の減少
- 要件定義書とDesign docの作成2時間、Memory docを更新しつつ**コードを1行も書かずに60%程度のMVPが動作確認可能になった**

**Bad**
- Memory docが古いまま開発を続けたら、AIが削除済みの関数を使ったコードを生成

### 暫定的な運用ルール
- 機能単位（例：仕訳入力画面完成時）で`/update-memory`を実行

---

## メリット
- **コンテキスト短縮**：長大な履歴やプロジェクト全体のソースを渡さなくても、要約済みのMemory docでAIがアタリを付けられる
- **人間によるコンテキストのレビュー・編集が可能**：AIが生成したコンテキスト要約を人間がレビュー・編集できるので、透明性と柔軟性を確保
- **AI向けだけど人間も読める**：Markdown形式なので、AIの外部記憶でありながら人間のレビュー資料にもなる

## デメリット
- **ドキュメントレビューコストの大量発生**：高頻度で大量のドキュメント更新が発生し、それをレビューする可能性がでてくる
- **Memory docのコンフリクトの発生**：複数のプログラマーが同一リポジトリで開発するとき、ほぼ必ずMemory docにコンフリクトが発生することが予想される
  - ただし、Memory docはAIによる生成物であるため、マージで苦戦するよりも『最新のコードをもとに再生成』して上書きする運用が良いかもしれない

**読者の皆さんへ**：良い解決策をご存知の方、コメント欄で教えてください！

---

## まとめ
この手法は「**AIがコードを理解するための自然言語ドキュメント**」を中心に回す開発手法です。開発環境としてはMCPや外部サービスを使わなくても、AIだけで運用可能です。
まだ試せていませんが、Cursorエディターなどのマルチエージェントに刺さるかもしれません。

運用方法として、以下の割り切りが重要です：  
- AIはコーディングが得意な相棒とみなして活用すると考える
- 細部の修正はやはり人の手の方が今のところ早い事は確か
- ドキュメントはAIが理解できれば十分。分からない時はAIに質問してから該当するソースコードを読めばいい
- 人間が読むのはおまけだが、レビューや共有には役立つ
- 要件定義 → Design doc → ソース生成 → Memory doc更新 → ソース生成、というループを回す
- プロジェクトでの仕様粒度も`spec/`内でできればなお良し

[^1]: GitHub Blog. (2025). "Spec-driven development with AI: Get started with a new open source toolkit" https://github.blog/ai-and-ml/generative-ai/spec-driven-development-with-ai-get-started-with-a-new-open-source-toolkit/
