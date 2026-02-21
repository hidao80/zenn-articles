---
title: WSL2+Podmanでdeepwiki-openを使う
emoji: "📖"
type: "tech"
topics: ["podman","deepwiki-open","wsl","ollama","ai"]
published: false
---
## この記事について

:::message alert
本記事には一部、AIにより生成された文章が含まれます。
:::

:::message
- **想定対象読者**: OpenAI APIのような公開AIに頼らず、プライベートな環境でDeepWikiのようなものを利用したい開発者のうち、Windowsを利用している人
- **前提知識**: 
  - WSL[^1]、Podman[^2]、DeepWiki[^3]、Ollama[^4]、Git、GitHub、Ubuntuがどのようなものであるかと使用方法
  - Dockerの利用経験：初級程度
- **所要時間**: 初回セットアップ10分 + 設定5分 + 処理時間（グラフィックボード性能による）
- **習得内容**: Dockerを利用せずにローカルAIでDeepWiki Openを利用する
:::

## tl;dr
- DeepWikiのようなサービスを安全に運用するため、deepwiki-openをローカルホストのWSL上のPodmanとOllamaで動作させる。
- deepwiki-openを高速なOllamaとPodmanで利用するにあたり、Ollamaが実行されているホストOSであるWindowsのIPを`host.docker.internal`や`host.containers.internal`に割り当てる必要がある。
- デフォルトではWindowsで動作しているOllamaのアドレスを指定するときは`/etc/hosts`にDockerまたはPodman互換のドメイン名とWSLが持つWindowsへのIPアドレスを指定するとよい。
- ただし、WindowsのIPアドレスは起動するごとに変わる可能性がある。

ローカルリポジトリは`docker-compose.yml`を通じてボリュームとしてdeepwiki-openへ渡す。

## 1. 課題と解決のアプローチ

### 動機

DeepWikiをローカルホストで利用するため、[deepwiki-open](https://github.com/AsyncFuncAI/deepwiki-open)に目をつけました。
また、Dockerは一部有料化されていますが、Red hat社が開発しているPodmanは無料で利用でき、Root権限やデーモン不要でDockerと高い互換性があります。
今回は両方の検証も兼ねてOllamaはWindows上で、PodmanはWSL2上で動作させ、Podman経由でDeepWiki Openの利用にチャレンジしました。

### 技術選定の基準

- ライセンス料金、API利用料金をかけないこと
- DeepWikiのような機能をローカルホストで実現すること
- Windows11 Home環境でも利用できること

### 選択した技術とその特徴

- **フリーソフトウェア[^5]に近いOSS**: deepwiki-openはGitHubにMITライセンスで公開されている
- **プライバシー重視**: ローカル環境で完結するため、データが外部に送信されない
- **コスト削減**: OpenAI APIなどの有料サービスを使わずに済む
- **実用性**: ドキュメントの不足している巨大なリポジトリ開発へのオンボーディングに利用できる

## 2. 前提条件と環境確認

### システム要件

- Windows11にインストールされたWSL2
- インターネットとの通信環境（初回セットアップ時）

## 3. 段階的セットアップ

:::details Ollamaのインストール
`winget`[^6]コマンドまたは[公式サイトのダウンロードページ](https://ollama.com/download)からダウンロードし、インストールします。

```sh:wingetの場合
winget update
winget install --id  Ollama.Ollama -h
```

```powershell:PowerShellの場合
irm https://ollama.com/install.ps1 | iex
```

その後、以下の2つのモデルを取得します。CUDAが使えるなどマシンパワーに余裕があるときは`gpt-oss:20b`や`gemma3:12b`などを利用しても良いでしょう。

```sh
ollama pull nomic-embed-text:latest
ollama pull qwen3:1.7b
```
:::

:::details  Git for Windowsのインストール
`winget`コマンドまたは[公式サイト](https://git-scm.com/install/windows)からダウンロードし、インストールします。

```sh
winget install --id Git.Git -h
```
:::

:::details WSL2へUbuntuのインストール
`winget`コマンドまたは[Microsoft Store](https://apps.microsoft.com/detail/9nz3klhxdjp5?hl=ja-JP&gl=JP)[^7]からUbuntu 24.04をインストールします。

```sh
winget install --id Canonical.Ubuntu.2404 -h
```
:::

:::details Podmanのインストール

<span id="Podmanのインストール"></span>

スタートメニューからUbuntuを起動し、アップデートをかけてからPodmanとpodman-composeをインストールします。

`podman-compose`はpython3製ツールであるため、python3もインストールする必要があります。
Podmanには標準で`podman compose`というオプションがありますが、これはデフォルトだとDockerバイナリに依存するので今回は利用しません。[^8]

```sh:Ubuntu起動後
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3 podman podman-compose
```
:::

:::details deepwiki-openの取得
スタートメニューからUbuntuを起動し、アップデートをかけてからcloneします。

```sh:Ubuntu起動後
sudo apt update && sudo apt upgrade -y
git clone https://github.com/AsyncFuncAI/deepwiki-open
cd deepwiki-open
```
:::

### deepwiki-openの設定

#### .envの作成

`.env`ファイルは実行に必須ですが、存在しないので作成する必要があります。
また、変数`GOOGLE_API_KEY`と`OPENAI_API_KEY`は必須で、値が空では動かないようです。今回は使用しませんが起動させるためにダミーのAPIキーを設定します。

```env:.env
OLLAMA_HOST=http://host.docker.internal:11434
DEEPWIKI_EMBEDDER_TYPE=ollama
GOOGLE_API_KEY=dummy
OPENAI_API_KEY=dummy
```

#### docker-compose.ymlの設定

WSLのホストOSであるWindowsで動作しているOllamaサーバーを利用するために`extra_hosts`プロパティを追加し、任意のローカルリポジトリをコンテナからアクセスできるようにするため、`volumes`プロパティに任意のローカルリポジトリを追加します。

```yml:docker-compose.yml
services:
  deepwiki:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "${PORT:-8001}:${PORT:-8001}"
      - "3000:3000"
    env_file:
      - .env
      # Podmanの外部にあるOllamaサーバーを利用するために追加
    extra_hosts:
      - "host.docker.internal:host-gateway"
    environment:
      - PORT=${PORT:-8001}
      - NODE_ENV=production
      - SERVER_BASE_URL=http://localhost:${PORT:-8001}
      - LOG_LEVEL=${LOG_LEVEL:-INFO}
      - LOG_FILE_PATH=${LOG_FILE_PATH:-api/logs/application.log}
    volumes:
      - ~/.adalflow:/root/.adalflow
      - ./api/logs:/app/api/logs
      # 以下の":"の左側はWSL2のUbuntuから見た、解析対象のリポジトリルートへのパス
      - /path/to/repo_root_dir:/app/repo
    mem_limit: 6g
    mem_reservation: 2g
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:${PORT:-8001}/health"]
      interval: 60s
      timeout: 10s
      retries: 3
      start_period: 30s
```

#### hostsファイルの変更

Docker Desktopが動いていれば`host.docker.internal`が利用でき、WSL内でOllamaが動いていれば`host.containers.internal`をドメインとしてOllamaサーバーへのアクセスが可能ですが、今回はどちらの条件も満たさないため、`/etc/hosts`に`host.docker.internal`項目を追加して回避します。

```sh:/etc/hostsにホストOSであるWindowsへの内部的なIPアドレスを追加する
echo "$(ip route show | grep -i default | awk '{ print $3}') host.docker.internal" | sudo tee -a /etc/hosts
```

## 4. 動作確認とトラブルシューティング

### 基本的な使用方法

1. 以下のコマンドでコンテナを起動します：

    ```sh
    podman-compose up
    ```

2. <http://localhost:3000>にアクセスします。ブラウザでDeepWiki Openが表示されます（[図1](#pic-1)）。
3. ホーム画面上部のリポジトリパス入力テキストボックスに`docker-compose.yml`で指定したボリュームの`/app/repo`を指定して「Wikiを検索」ボタンをクリックします。詳細設定ダイアログが開きます。
4. 詳細設定ダイアログでWikiの詳細度選択と作成に使うAIモデルの選択を行います。この手順書に沿っているなら利用できるのは「Ollama（ローカル）」の「qwen3:1.7b」だけです。選択して「Wikiを生成」をクリックします。
5. Wiki作成の進捗表示画面が表示されます。完了するまで待ちましょう。
6. Wikiが完成したらWiki画面が開きます（[図2](#pic-2)）。画面右下の吹き出しアイコンをクリックすると、Wikiの内容についてAIに質問できるダイアログが開きます。文章で質問を入力し「質問する」ボタンをクリックするとAIがWikiとデータベースの内容から回答を生成します。
7. 画面左上の「ホーム」リンクをクリックするとホーム画面に戻ります。一度生成したWikiはキャッシュされ、次回からはすぐに閲覧とAIへの質問が可能になります（[図3](#pic-3)）。

### 動作結果のスクリーンショット

:::details 図1

<span id="pic-1"></span>

![図1](/images/deepwiki-open_with_podman/deepwiki-open_1.png)
:::

:::details 図2

<span id="pic-2"></span>

![図2](/images/deepwiki-open_with_podman/deepwiki-open_2.png)
:::

:::details 図3

<span id="pic-3"></span>

![図3](/images/deepwiki-open_with_podman/deepwiki-open_3.png)
:::

### トラブルシューティング

#### Ollamaが呼び出せない場合

`/etc/hosts`に`host.docker.internal`項目と[hostsファイルの変更](#hostsファイルの変更)節のコマンドで取得できるIPアドレスが設定されていますか？
このIPアドレスはWindowsを再起動するごとに変わる可能性があります。

#### Ollamaは呼び出せるが正常に終了しない

OSメモリーが足りない可能性があります。PCのメモリーが16GB程度以上あると良いです。
また、Ollamaに割り当てるコンテキスト長が長すぎる場合も正常に動作しなくなります。8k（≒8000）程度にしておくと良いです。

#### podman-composeが起動しない

[Podmanのインストール](#Podmanのインストール)開閉パネルの中でインストールしている`podman-compose`（Pythonプログラム）と`Python3`の両方をインストールしてください。

## 5. 応用と拡張

### 工夫した点

- **OllamaのホストOS持ち**: ホストOSネイティブアプリを利用することによって、より簡単にグラフィックボードによる高速化を受けやすくした
- **Dockerとの互換性を活かす**: 可能な限りそのままの操作や設定でDocker Desktopでも実行できるようにした
- **ローカルリポジトリをボリュームとして取り込む**: GitLabなどのリポジトリ取得元システムの導入を不要にした

### 今後の発展性

ほぼこのままの形でOpenAI APIやGoogle API、Claude APIに対応できます。
また、macOSやLinuxでも動作確認をしていきたいです。おそらくOllamaのインストールとIPの指定方法が若干変わる程度だと思われます。

今のところ対話用のAIモデルはメモリ使用量やパフォーマンスの面から`qwen3:1.7b`を採用していますが、よりリッチな環境で`gpt-oss:20b`や`gemma3:12b`も試してみたいですね。

[^1]: **WSL**: 再起動や仮想マシンの個別設定なしで、WindowsとLinuxの一部の機能を同時に使える。[公式サイト](https://learn.microsoft.com/ja-jp/windows/wsl/)
[^2]: **Podman**: Dockerと互換性があり、かつルート権限も常駐プロセスも不要な次世代コンテナエンジン。[公式サイト](https://podman.io/)
[^3]: **DeepWiki**: GitHubリポジトリのURLを変えるだけで、コードの内容をAIが自動的に理解しやすいドキュメントにしてくれる対話型サービス。[公式サイト](https://deepwiki.com/)
[^4]: **Ollama**: ローカル環境で対話型LLM AIモデルを簡単に動かせる、無料の実行環境＆APIサーバー。[公式サイト](https://ollama.com/)
[^5]: **フリーソフトウェア**: ユーザーが使用・調査・変更・再配布を自由にできるソフトウェア。[公式サイト](https://www.fsf.org/)
[^6]: **winget**: コマンドでWindowsアプリを一括管理（検索・導入・更新・削除）できる、Microsoft公式の高速ツール。[公式サイト](https://learn.microsoft.com/ja-jp/windows/package-manager/)
[^7]: **Microsoft Store**: Windows用の安全・安心なアプリ＆ゲーム配布プラットフォームです。[公式サイト](https://apps.microsoft.com/home)
[^8]: ※設定を変更し、`podman compose`実行時に`podman-compose`を呼び出すようにすることができるようです。
