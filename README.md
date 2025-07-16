# Zenn Contents Repository

Zennへ投稿される記事を管理するレポジトリです。

ミスの修正も大歓迎です！


## 概要 (Overview)

このリポジトリは、[Zenn](https://zenn.dev/mtk0328) で公開している記事や本のコンテンツを管理するためのものです。
[Zenn CLI](https://zenn.dev/zenn/articles/zenn-cli-guide) を利用して、ローカル環境で執筆・プレビューを行い、このリポジトリの`main`ブランチにプッシュすることで自動的にZenn.devにコンテンツがデプロイ（公開・更新）されます。


## セットアップ (Setup)

記事を執筆する前に、ローカル環境でZenn CLIをセットアップする必要があります。

### 1\. Node.jsのインストール

Zenn CLIはnpmパッケージとして提供されているため、[Node.js](https://nodejs.org/ja/) のインストールが必要です。まだの方はインストールしてください。

### 2\. Zenn CLIのインストール

ターミナルで以下のコマンドを実行し、Zenn CLIをインストールします。

```
npm install zenn-cli
```

### 3\. リポジトリの初期化

このリポジトリをクローンした後、以下のコマンドを実行して必要なディレクトリ（`articles`, `books`）を生成します。

```
npx zenn init
```

## 執筆の流れ (Workflow)

### 1\. 新しい記事の作成

以下のコマンドを実行すると、`articles`ディレクトリに新しい記事のMarkdownファイルが作成されます。

```
npx zenn new:article --slug XXX
```

実行後、`articles/XXX.md` のファイルが生成されるので、これを編集して記事を執筆します。

### 2\. 記事のプレビュー

ローカルサーバーを立ち上げて、ブラウザで記事のプレビューを確認できます。

```
npx zenn preview
```

プレビューは `http://localhost:8000` で確認できます。ファイルを保存すると自動的にプレビューが更新されます。

### 3\. 画像の管理

記事で使用する画像は、リポジトリ内の `/images` ディレクトリに配置することを推奨します。Markdownからは以下のように相対パスで参照できます。

```
![画像の説明](/images/example.png)
```

Zennの管理画面から画像をアップロードして、そのURLを貼り付けることも可能です。

### 4\. コンテンツの公開・更新

執筆が完了した記事は、このリポジトリの `main` ブランチにプッシュ（またはマージ）することで、Zenn.dev上に自動で公開・更新されます。

**公開ステータスの管理:**
各記事のMarkdownファイルのフロントマターで公開ステータスを管理します。

  - **公開する場合:**
    ```yaml
    ---
    title: "記事のタイトル"
    emoji: "✨"
    type: "tech" # tech: 技術記事 / idea: アイデア記事
    topics: ["zenn", "github", "markdown"]
    published: true
    ---
    ```
  - **下書きにする場合:**
    ```yaml
    ---
    published: false
    ---
    ```

`published: true` に設定して`main`ブランチにプッシュすると、その記事が公開されます。

## ディレクトリ構成

```
.
├── articles/         # 記事のMarkdownファイル
│   ├── xxxxxxxxxx.md
│   └── yyyyyyyyyy.md
├── books/            # 本のMarkdownファイル
│   ├── book-slug/
│   │   ├── chapter1.md
│   │   └── config.yaml
│   └── ...
├── images/           # 画像ファイル
│   └── example.png
├── .gitignore
└── README.md
```

  - **`articles`**: 記事コンテンツを格納します。ファイル名はZenn CLIが自動で生成します。
  - **`books`**: 本のコンテンツを格納します。各本はサブディレクトリで管理されます。
  - **`images`**: 記事や本で使用する画像を格納します。

## 参考リンク (References)

  - **Zenn公式サイト:** [https://zenn.dev](https://zenn.dev)
  - **Zenn CLI 公式ガイド:** [https://zenn.dev/zenn/articles/zenn-cli-guide](https://zenn.dev/zenn/articles/zenn-cli-guide)
  - **GitHubリポジトリ連携の方法:** [https://zenn.dev/zenn/articles/connect-to-github](https://zenn.dev/zenn/articles/connect-to-github)
