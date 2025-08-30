---
title: ローカルLLM AIでつくる無料ネット検索エージェント
emoji: "🔌"
type: "tech"
topics: ["ai","mcp","zero-config","agnet","lm-studio"]
published: false
---
## この記事について

:::message alert
本記事には一部、AIにより生成された文章が含まれます。
:::

:::message
- **想定対象読者**: ChatGPTのようなクラウドAIに頼らず、プライベートな環境で情報検索したい開発者
- **前提知識**: 
  - MCPサーバー[^1]がどのようなものであるかとLM Studioの使用方法
  - プログラミング経験：中級程度（コマンドライン操作に慣れている）
  - AI利用経験：ChatGPT等の使用経験あり
- **所要時間**: 初回セットアップ10分 + 慣熟5分
- **習得内容**: LM StudioをUIにしてAIにネット検索できるようにセットアップする
:::

## tl;dr
LM StudioのMCPコネクタ[^2]に`mcp-server-fetch`を追加して、Tool call[^3]（MCPサーバーの呼び出し）可能なAIモデルに使用許可を出してからプロンプトで調べるように指示する

## 1. 課題と解決のアプローチ

### 動機

ChatGPTなどのクラウドAIサービスは便利ですが、プライバシーの懸念やコストの問題があります。
とくに企業内での利用では、機密情報の外部送信リスクが課題でした。  
ローカルLLMに検索機能を追加することで、セキュアかつ経済的なAIアシスタントを
構築できるのではないかと考えました。

### 技術選定の基準

- 新規インストールを最低限に抑えること
- Claude CodeやClaudeデスクトップ、GitHub Copilot Chat、Cursorエディターなどに応用しやすいこと

### 選択した技術とその特徴

- **ゼロコンフィグ**: 複雑な設定ファイルの作成や環境変数の設定が不要
- **プライバシー重視**: ローカル環境で完結するため、データが外部に送信されない
- **コスト削減**: OpenAI APIなどの有料サービスを使わずに済む
- **実用性**: 最新情報の取得、ファクトチェック、リサーチ業務に即座に活用可能
- **活用シーン**: 「社内情報の機密性を保ちながら技術調査を行う」「オフライン環境での開発時のドキュメント検索」「社内Wikiなどの参照」

## 2. 前提条件と環境確認

### システム要件

- LM Studio バージョン0.2.29以上
- インターネット接続（初回セットアップ時、インターネット検索実行時）
- 対応OS：Windows 10/11、macOS、Linux

### 必要な前提知識

- MCPサーバー[^1]の基本概念
- LM Studioの基本操作
- コマンドライン操作の経験
- Function Calling[^6]の概念理解

## 3. 段階的セットアップ

### ステップ1: `uv`のインストール

Pythonパッケージマネージャーの`uv`をインストールします。WindowsならWinget[^4]、macOSならHomebrew[^5]が便利です。

:::details Windowsの場合
```bash
# Windows (Winget)
winget install astral-sh.uv

# 直接インストール (Linux/Windows/macOS)
curl -LsSf https://astral.sh/uv/install.sh | sh
```
:::

:::details macOSの場合
```bash
# macOS (Homebrew)  
brew install uv

# 直接インストール (Linux/Windows/macOS)
curl -LsSf https://astral.sh/uv/install.sh | sh
```
:::

### ステップ2: LM Studioの`mcp.json`へ設定の記述

LM StudioのGUIから`mcp.json`を編集します。

[Settings]（右上のレンチ🔧マーク）→[Program]→[Integrations]→[Install▼]→[Edit mcp.json]から`mcp.json`をLM Studio内で開き、以下のjsonを記述または追記してください。

:::details mcp.jsonを開くまでのクリック手順
![LM StudioのGUIでmcp.jsonを開く](/images/local_ai_net_search/open_mcp_json.png)
*LM StudioのGUIでmcp.jsonを開く*
:::

:::message alert
すでに`mcpServers`要素が存在するときは`mcpServers`のプロパティとして
`fetch`ツールを追加するように`mcp.json`を編集してください。
:::

```json
{
  "mcpServers": {
    "fetch": {
      "command": "uvx",
      "args": ["mcp-server-fetch"]
    }
  }
}
```

### ステップ3: ツール`fetch`の有効化

LM Studioを再起動後、チャット画面でツールアイコンをクリックし、`fetch`を有効にします。
初回実行時に`mcp-server-fetch`パッケージが自動的にダウンロードされます。

### ステップ4: ツール呼び出し可能なモデルの選定

Function Calling[^6]に対応したモデルを選択してください。推奨モデル：
- `Qwen2.5-Coder-7B-Instruct`
- `Llama-3.1-8B-Instruct`
- `Hermes-3-Llama-3.1-8B`
- `gpt-oss:20b`

### ステップ5: プロンプト実行時にツールの有効化

プロンプト送信前に、ツールのトグルスイッチがオンになっていることを確認してください。

:::details ツールのトグルスイッチがオンにしたときの画面の様子
![設定ペインのfetchツールがオンになっている](/images/local_ai_net_search/fetch_on_settings.png)
*設定ペインのfetchツールがオンになっている*

![プロンプト入力エリアのfetchツールがオンになっている](/images/local_ai_net_search/fetch_on_input_area.png)
*プロンプト入力エリアのfetchツールがオンになっている*
:::

## 4. 動作確認とトラブルシューティング

### 基本的な使用方法

基本的な使用パターンは以下の通りです：

```txt:プロンプトパターン1：最新情報の検索
「2025年のWeb技術トレンドについて最新情報を調べてください」
```

```txt:プロンプトパターン2：特定のWebページの内容確認
「https://example.com の内容を要約してください」
```

```txt:プロンプトパターン3：ファクトチェック
「○○という情報は正しいですか？最新のソースで確認してください」
```

AIが自動的にWebページを取得し、内容を解析して回答します。複数のソースを横断した調査も可能です。

:::message alert
AIツールからのアクセスを遮断する設定になっているWebサイトも存在します。
:::

### 動作結果のスクリーンショット

:::details Windowsの場合
![Windowsの場合](/images/local_ai_net_search/result_win.png)
:::

### トラブルシューティング

#### MCPサーバーが起動しない場合

1. `uv`がインストールされていることを確認
2. インターネット接続を確認（`uvx`でパッケージをダウンロードするため）
3. LM Studioのバージョンが0.2.29以上であることを確認
4. `mcp.json`の記述に誤りがないかチェック

## 5. 応用と拡張

### 工夫した点

- **シンプルな設定**: JSON1つとコマンド1つで完結する最小構成
- **汎用性**: Webアクセス、API呼び出しへの拡張が容易
- **運用性**: uvxによる依存関係の自動解決でメンテナンスフリー

### 今後の発展性

MCPの導入により、ローカルLLMの実用性が大幅に向上しました。
とくにリサーチ業務では、複数のソースを横断した情報収集が自動化できます。

今後も`uvx`や`npx`などによるローカルLLM AI機能拡張も検討したいと思います。

### 他のツールへの応用

- **Claude Code**: コマンドライン開発環境での情報検索
- **VS Code拡張**: エディター内からの直接検索
- **GitHub Copilot Chat**: 開発中のコンテキスト検索
- **Cursor**: AI駆動の統合開発環境での活用

[^1]: **MCPサーバー（Model Context Protocol Server）**: AIモデルがPythonスクリプトやWebページの取得などの外部ツールを呼び出すためのプロトコル。AnthropicによりClaude Desktop用に開発された。[公式サイト](https://modelcontextprotocol.io/)

[^2]: **MCPコネクタ**: LM StudioのMCPサーバー接続機能。v0.3.0以降で対応。設定ファイル`mcp.json`を通じてサードパーティのツールを連携できる

[^3]: **Tool call（ツール呼び出し）**: 自然言語モデルが推論中に外部ツールや関数を実行する機能。Function Calling[^6]やPlugin機能とも呼ばれる

[^4]: **Winget**: Microsoftが開発したWindows向け公式パッケージマネージャー。Windows 10/11に標準搭載。[公式サイト](https://docs.microsoft.com/ja-jp/windows/package-manager/)

[^5]: **Homebrew**: macOS/Linux向けパッケージマネージャー。開発環境構築でよく利用される。[公式サイト](https://brew.sh/ja/)

[^6]: **Function Calling**: AIモデルが事前定義された関数やツールを呼び出す機能。OpenAIが2023年6月に導入し、現在では多くのLLMが対応している標準的な機能