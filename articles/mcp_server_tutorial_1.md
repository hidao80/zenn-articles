---
title: MCP Serverチュートリアル(1) Sqlite3編
emoji: "🧠"
type: "tech"
topics: ["tutorial","sqlite","mcp","zero-config","model-context-protocol"]
published: true
---
## この記事について

:::message alert
本記事には一部、AIにより生成された文章が含まれます。
:::

:::message
- **対象読者**: MCP対応エディターでSQLite操作を行いたい中級開発者
- **前提知識**: SQLiteの基本操作、コマンドラインツールの使用経験
- **所要時間**: 約30分（セットアップ込み）
- **習得内容**: MCPサーバーの設定とSQLite操作の自動化
- **サンプルコードのリポジトリ**: <https://github.com/hidao80/mcp-tutorial-1>
:::

ローカルにあるSQLiteデータベースを操作するためのModel Context Protocol (MCP) サーバーの最初のチュートリアルです。
CursorエディターやClaudeデスクトップアプリなど、MCP対応クライアントで使用できます。

:::details Model Context Protocol (MCP)ってなに？
Claudeくん本人に聞きました：

> Model Context Protocol（MCP）は、2024年にAnthropic社が発表した、AI言語モデルと外部アプリケーションを
> 安全に接続するための標準プロトコルです。
> 
> MCPサーバーとクライアント間でJSON-RPC通信を行い、AIモデルがファイルシステム、データベース、API、
> 開発ツールなどにアクセスできます。
> 
> セキュリティを重視した設計で、モデルが実行できる操作を制限し、ユーザーの許可なしに危険な操作を防ぎます。
> ClaudeデスクトップやVS Code拡張などで実装され、開発者は独自のMCPサーバーを作成して特定の業務や
> ツールとAIを統合できます。
> 
> AI活用の新たな可能性を開く重要な技術基盤となっています。
:::

## 本記事の特徴

- **ゼロ設定**: 追加設定なしで即座に使用開始
- **完全なSQL操作サポート**: 作成、読み取り、更新、削除の全操作
- **クロスプラットフォーム**: Windows、macOS、Linux対応

:::details 利用するツールの一覧と概要
- **brew**: macOS・Linux用パッケージマネージャー。ソフトウェアを簡単にインストール・管理 [公式サイト](https://brew.sh/)
- **winget**: Microsoft公式Windows用パッケージマネージャー。コマンドラインからアプリを管理 [公式ドキュメント](https://docs.microsoft.com/ja-jp/windows/package-manager/)
- **uv**: Rust製の高速Pythonパッケージマネージャー。pipより高速で依存関係管理を統合 [公式ドキュメント](https://docs.astral.sh/uv/)
- **uvx**: uvツール実行コマンド。Pythonパッケージを一時的にインストールして実行 [公式ドキュメント](https://docs.astral.sh/uv/guides/tools/)
- **Node.js**: サーバーサイドJavaScript実行環境。npmを含む豊富なエコシステム [公式サイト](https://nodejs.org/)
- **sqlite3**: 軽量ファイルベースデータベース。サーバー不要でアプリに組み込み可能 [公式サイト](https://www.sqlite.org/)
- **Cursor**: AI支援機能搭載のコードエディター。VS Codeベースで高度なAI機能を提供 [公式サイト](https://cursor.sh/)
- **VS Code**: Microsoft製オープンソースエディター。豊富な拡張機能と統合開発環境 [公式サイト](https://code.visualstudio.com/)
- **Windsurf**: Codeium製AI駆動エディター。リアルタイムAI協調開発機能が特徴 [公式サイト](https://codeium.com/windsurf)
- **Claudeデスクトップアプリ**: Anthropic社のAIチャット「Claude」のデスクトップ用アプリ [ダウンロード](https://claude.ai/download)
- **DXT**: Desktop Extensionの略。Claudeデスクトップアプリ用の拡張機能ファイル [参考：claude-desktopでローカルmcpサーバーを始める](https://support.anthropic.com/ja/articles/10949351-claude-desktop%E3%81%A7%E3%83%AD%E3%83%BC%E3%82%AB%E3%83%ABmcp%E3%82%B5%E3%83%BC%E3%83%90%E3%83%BC%E3%82%92%E5%A7%8B%E3%82%81%E3%82%8B)
- **LM Studio**: ローカルで大規模言語モデルを実行するデスクトップアプリ。オフライン環境でLLMを利用可能 [公式サイト](https://lmstudio.ai/)
:::

## 技術選定の基準

- コードの記述が必要ないこと
- PCの環境を極力変更しないこと
- 新規インストールも無しまたは最低限に抑えること
- 応用することで実用性があること
- この記事オリジナルであること

## セットアップ

### 1. データベースの初期化

動作確認のため、データベースを作ります。SQLiteコマンドまたはDBeaverなどのDBブラウザーを使用して`init_ja.sql`を実行してください：

:::details SQLite3がインストールされている場合の実行方法
```sh
# bash、コマンドプロンプト、zshの場合
sqlite3 database.db < init_ja.sql

# powershellの場合
Get-Content .\init_ja.sql | sqlite3.exe .\database.db
```
:::

### 2. MCPサーバーの動作環境のインストール

本リポジトリでは`uvx`を利用しますので、あらかじめ`uv`のインストールが必要です。

:::details macOSの場合
```sh
brew install uv
```
:::

:::details Windows11の場合
```sh
winget install --id=astral-sh.uv
```
:::

## 3. MCP設定ファイルの配置

- **Cursorの場合**: `.cursor/mcp.json`
- **VS Codeの場合**: `.vscode/mcp.json`
- **Windsurfの場合**: `.windsurf/mcp.json`
- **Claudeデスクトップアプリの場合**: DXT
- **LM Studioの場合**: アプリから直接設定します。

このプロジェクトでは`./.cursor/mcp.json`と`./.vscode/mcp.json`と`./.windsurf/mcp.json`を
配置しているため、プロジェクトを開いた際に自動的に認識されます。

## 4. セットアップ手順

:::details Cursorの場合
1. このリポジトリをクローンまたはダウンロード
2. Cursorでこのリポジトリのディレクトリを開く
3. `Shift + Ctrl + P`（Windows/Linux）`Cmd + Shift + P`（MacBook）で
  コマンドパレットを開く
4. `openmcp`と入力し、「Tools & Integrations」タブを開く
5. 「MCP Tools」節より「sqlite」MCPサーバのスイッチをオン（緑色）にする
:::

:::details VS Codeの場合
1. このリポジトリをクローンまたはダウンロード
2. VS Codeでこのリポジトリのディレクトリを開く
3. `Shift + Ctrl + P`（Windows/Linux）`Cmd + Shift + P`（MacBook）で
  コマンドパレットを開く
4. `mcpli`と入力し、「MCP: サーバーの一覧表示」を選択する
5. コマンドパレットから「sqlite　停止中」を選択し、起動する
6. チャットペインの一番下からモードを**Agent**に変更する
:::

:::details Windsurfの場合
1. このリポジトリをクローンまたはダウンロード
2. Windsurfでこのリポジトリのディレクトリを開く
3. `Shift + Ctrl + P`（Windows/Linux）`Cmd + Shift + P`（MacBook）で
  コマンドパレットを開く
4. `mcpli`と入力し、「MCP: サーバーの一覧表示」を選択する
5. コマンドパレットから「sqlite　停止中」を選択し、起動する
:::

::::details Claudeデスクトップアプリの場合
Claude DesktopでMCPサーバーを利用するためにDXT（Desktop Extensions：デスクトップ拡張機能）パッケージをビルドしてインストールします。

Claude Desktop用のDXTパッケージをビルドする手順は以下の通り：

:::details PowerShell以外の場合
```sh
# bash、コマンドプロンプト、PowerShell、zsh共通
# dxtディレクトリに移動してビルド
cd dxt-src
npm run package
```
:::

:::details PowerShellの場合
```sh
# dxtディレクトリに移動してビルド
npm run package-ps
```
:::

ビルドが成功すると、`dist/sqlite-mcp-server.dxt`ファイルが生成されます。

:::details コラム：「DXT」ってなに？
Claudeくん本人に聞きました：

> ClaudeデスクトップのDXT（Desktop Extensions）は、2025年に導入された拡張機能システムです。
> MCP（Model Context Protocol）を基盤として、サードパーティツールやアプリケーションとClaudeを直接統合できます。
> ファイルシステムアクセス、外部APIとの連携、生産性ツールとの接続など、Webブラウザ版では制限のある機能を実現できます。
> 
> Pythonスクリプトやカスタムツールを作成し、Claudeが直接実行・操作可能で、ワークフロー自動化や専門タスクの効率化を支援します。
> 開発者やパワーユーザーにとって、Claudeをより強力なローカル作業環境に変貌させる重要な機能です。
:::

### 必要な依存関係

DXTパッケージのビルドには以下が必要です：

- Node.js 18以上
- 公式DXT CLI（npm run build実行時に自動インストール）

### Claude Desktopへのインストール

1. 上記の手順でDXTパッケージをビルド
2. 生成された`.dxt`ファイルをClaude Desktopの設定＞拡張機能画面にドラッグアンドドロップ
3. 「SQLite MCP Server」にSqliteのデータベースファイルのパスを与え、「有効」スイッチをオン（青色）にする
4. データベースファイルが指定された位置にあれば、SQLite MCPサーバーが利用可能になります
::::

:::details LM Studioの場合
※以下の説明はLM Studio 0.3.23 (Build 3)を対象にしています。
1. アプリ右上の :wrench: のような`Show Settings`から設定ペインを開きます。
2. 一番上の右のボタン`Program`をクリックします。
3. `Integrations`画面が開いたら右上の`Install`コンボボックスをクリックし、`Edit mcp.json`を開きます。
4. チャット領域にmcp.jsonタブが開きますので、その中に`.cursor/mcp.json`の内容をコピペします。
5. **Tool use**機能を持つモデルを選択し、チャットを始めます。
:::

### 使用方法

プロンプトに「データベースからもっとも多く購入したユーザーは？」などと質問することで、
データベースへのアクセスをAIが自動的に試みます。  
SQLは内部的に発行されるため、詳細に指定するとき以外は不要で、日本語などの自然言語のみでデータベース操作が可能です。

### 動作結果のスクリーンショット

:::details VS Codeの場合（🚧準備中🚧）
<!-- ![VS Codeの場合](/images/mcp_server_tutorial_1/result_vs_code.png)-->
※現在、GitHub Copilotがチャットクォータに達しておりスクリーンショットが撮影できないため、後日追記します。
:::

:::details Claudeデスクトップの場合
![Claudeデスクトップの場合](/images/mcp_server_tutorial_1/result_claude_desktop.png)
:::

:::details LM Studioの場合
![LM Studioの場合](/images/mcp_server_tutorial_1/result_lm_studio.png)
:::

## トラブルシューティング

### MCPサーバーが起動しない場合

1. `uv`がインストールされていることを確認
2. インターネット接続を確認（`uvx`でパッケージをダウンロードするため）
3. データベースファイルがプロジェクトルートに`database.db`という名前で存在することを確認

### データベースファイルが見つからない場合

1. プロジェクトディレクトリの書き込み権限を確認
2. `init_ja.sql`を`sqlite3`で実行してデータベースを作成
3. `mcp.json`や「SQLite MCP Server」DXTの設定画面にデータベースファイルへのパスが正しく設定されているか確認

### VS CodeでMCPサーバーが動作しない場合

チャットペインの一番下にあるAIのモード選択コンボボックス（`Ask`、`Edit`、`Agent`）から**Agent**を選択してからプロンプトを入力する

### LM Studioでデータベースの参照がうまくいかない場合

**Tool use**機能を持つモデルしかMCPサーバーが利用できません。

例）  
- **Gemma 3 12B**: **Vision**機能（画像読み取り）と**Tool Use**機能があるためMCPサーバーを使用可能
- **Gemma 3n E4B**: **Vision**機能はあるものの、**Tool Use**機能がないためMCPサーバーを使用できない


## ファイル構成

:::details ディレクトリツリー
```plain
mcp-tutorial-1/
├── init_ja.sql               # 初期データベーススキーマとサンプルデータ（日本語版）
├── init.sql                  # 初期データベーススキーマとサンプルデータ（英語版）
├── README.md                 # このファイル
├── database.db               # SQLiteデータベースファイル（初期化時に作成）
├── .gitignore                # Git除外設定
├── .cursor/mcp.json          # MCP設定ファイル（Cursor用）
├── .vscode/mcp.json          # MCP設定ファイル（VS Code用）
├── .windsurf/mcp.json        # MCP設定ファイル（Windsurf用）
├── docs/                     # 設計ドキュメントファイル群
│   ├── DESIGN_ja.md          # 設計ドキュメント（日本語）
│   └── DESIGN.md             # 設計ドキュメント（英語）
├── dxt-src/                  # Claude Desktop用DXTファイル群
│   ├── manifest.json         # DXTマニフェストファイル
│   ├── icon.png              # DXTアイコン画像
│   ├── index.js              # DXTメインエントリーポイント
│   ├── package.json          # DXT用パッケージ設定
│   ├── README_ja.md          # DXT固有のドキュメント（日本語）
│   └── README.md             # DXT固有のドキュメント（英語）
└── dist/                     # ビルド成果物（.dxtファイル）
    └── sqlite-mcp-server.dxt # Claude Desktop用デスクトップ拡張機能
```
:::

## ふりかえり

### 動機
「MCPをつかってみたい」と思ったとき、最初にMCPサーバー（プログラム）とMCPコネクタ
（mcp.jsonで表されるクライアントが持つ接続設定のこと。正式名称ではないが、
Claudeデスクトップアプリに記載されている）の違いがわからず戸惑ったため、
自習と実務に応用が効く形で、かつ簡単に動かしてみたい！と思いました。

### 工夫した点
この記事ではすぐに動作確認できるよう、リポジトリを作成してGitHubに公開し、
お試しであるならば可能な限りコーディングや設定編集をしないで動かせるようにしています。

また、PCローカルで完結させることでスタンドアロン環境でもMCPを利用することができるように
心がけました。

### 気づき
いまさらですが、MCPでなにかするということは**サーバーとサーバーへの接続設定の組でAIクライアントに外部サービスとの連携を可能にする**
というものであることが理解できました。

よく見かけるチュートリアル的な記事では`mcp.json`内の設定を編集したらそれだけで
機能が追加できるように見るものもありましたが、まったくそうではないということが
わかりました。
