---
title: "Google AI Studio (Gemini) API と LangChain で、RAG を実装してみた"
emoji: "🧠"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AI", "LLM", "機械学習", "gemini", "RAG"]
published: false
publication_name: "hibari_inc"
---

こんにちは！株式会社 HIBARI の中野と申します。

今回はGoole AI Studioの Gemini API と LangChain を活用して、独自のテキストデータに基づいた受け答えができる RAG システムを開発しましたので、その仕組みと動かし方についてご紹介します。

このシステムを使えば、指定したディレクトリ内のテキストファイルをナレッジベースとして、AI がその内容を元に質問に回答してくれるようになります。

RAGについてはこちらの記事を参照してください。

https://zenn.dev/hibari_inc/articles/rag-overview

### 💻 システムの概要

この RAG システムは、大きく分けて2つのプロセスで構成されています。

1.  **事前準備 (ベクトルデータベースの構築):** `data` ディレクトリ内のテキストファイルを読み込み、内容を分割（チャンク化）します。そして、Google の `embedding-001` モデルを使って各チャンクをベクトル化し、`ChromaDB` というベクトルデータベースに保存します。この処理は `build_vectordb.py` スクリプトが担当します。
2.  **質問応答 (チャットの実行):** ユーザーが質問を入力すると、その質問内容もベクトル化されます。そして、`ChromaDB` 内に保存されているデータの中から、質問ベクトルと類似度の高いチャンクを検索します。検索で得られた関連情報をコンテキスト（文脈）として、質問文と共に `gemini-pro` モデルに渡し、最終的な回答を生成させます。この処理は `chat.py` スクリプトが担当します。

\<br\>

### 📄 ファイル構成

プロジェクトの主要なファイルは以下の通りです。

```
project/
├── data/              # ナレッジベースとなるtxtファイルを格納
│   ├── document1.txt
│   └── ...
├── build_vectordb.py  # ベクトルデータベースを構築するスクリプト
├── chat.py            # チャット機能を提供するメインスクリプト
├── requirements.txt   # 必要なPythonパッケージ
└── .env               # 環境変数（APIキー）を設定するファイル
```

-----

### 🛠️ コード解説

それでは、主要な2つのスクリプト `build_vectordb.py` と `chat.py` の中身を詳しく見ていきましょう。

#### 1\. `build_vectordb.py`: ベクトルデータベースの構築

このスクリプトは、`data` ディレクトリのテキストファイルからベクトルデータベースを作成する役割を担います。

  - **ドキュメントの読み込み (`load_documents_from_directory`):**
    `TextLoader` を使って `data` ディレクトリ内の全 `.txt` ファイルを読み込みます。
  - **ドキュメントの分割 (`split_documents`):**
    読み込んだドキュメントが長文である場合、そのままでは扱いにくいため、`RecursiveCharacterTextSplitter` を使って適切な長さのチャンク（かたまり）に分割します。これにより、後続のベクトル化や検索の精度を高めます。
  - **ベクトルストアの作成 (`create_vectorstore`):**
    分割されたチャンクを、`GoogleGenerativeAIEmbeddings` (models/embedding-001) を使って一つずつベクトルに変換します。そして、それらのベクトルデータを `Chroma.from_documents` を使って `ChromaDB` に保存します。データベースは `./vectordb` ディレクトリに永続化されます。

#### 2\. `chat.py`: チャットの実行

このスクリプトは、構築されたベクトルデータベースを使ってユーザーとの対話を実現します。

  - **ベクトルストアの読み込み (`load_vectorstore`):**
    `build_vectordb.py` で作成・保存された `./vectordb` ディレクトリから、`Chroma` を使してベクトルストアを読み込みます。ここでも `GoogleGenerativeAIEmbeddings` が必要になります。
  - **QAチェーンの作成 (`create_qa_chain`):**
    この関数が RAG システムの心臓部です。
      - **LLM:** `GoogleGenerativeAI` を使って、回答生成モデルとして `gemini-pro` を初期化します。
      - **Retriever:** 読み込んだベクトルストアを `as_retriever` メソッドで検索機（Retriever）に変換します。`search_kwargs={"k": 5}` を指定することで、質問と関連性の高い上位5つのチャンクを検索するように設定しています。
      - **PromptTemplate:** LLM に渡すプロンプトのテンプレートを定義します。「以下の情報を基に...」という指示と共に、検索で得られた情報 (`context`) とユーザーの質問 (`question`) を埋め込めるようにしています。
      - **RetrievalQA:** これらを `RetrievalQA.from_chain_type` を使って一つにまとめ、QAチェーンを構築します。
  - **メイン処理 (`main`):**
    ユーザーからの入力を受け付け、QAチェーン (`qa_chain.invoke`) を実行して回答を生成します。回答と共に、どのドキュメントチャンクを参照したかも表示する機能を実装しています。

-----

### 🚀 動かし方

このシステムを実際に動かしてみましょう。

#### 1\. 依存関係のインストール

まず、`requirements.txt` に記載されている必要なライブラリをインストールします。

```bash
pip install -r requirements.txt
```

#### 2\. 環境変数の設定

`.env` という名前のファイルを作成し、お使いの Google AI Studio API キーを設定してください。APIキーは [こちら](https://makersuite.google.com/app/apikey) から取得できます。

```
GOOGLE_API_KEY=your_google_ai_studio_api_key_here
```

#### 3\. データの準備

プロジェクト内に `data` ディレクトリを作成し、ナレッジベースとしたいテキストファイル（`.txt` 形式）をその中に配置してください。

#### 4\. ベクトルデータベースの構築

最初に、以下のコマンドを実行して `data` ディレクトリのファイルからベクトルデータベースを構築します。

```bash
python build_vectordb.py
```

実行が完了すると、プロジェクトルートに `./vectordb` ディレクトリが作成されます。

#### 5\. チャットの開始

データベースの準備ができたら、いよいよチャットを開始できます。

```bash
python chat.py
```

ターミナルに「🙋 質問:」と表示されたら、自由に質問を入力してみてください。`data` ディレクトリのテキスト情報に基づいた回答が生成されます。チャットを終了するには `quit` または `exit` と入力してください。

### まとめ

今回は、Google AI Studio (Gemini) API と LangChain を用いて、独自のデータに対応する RAG システムを構築する方法をご紹介しました。

コード自体は比較的シンプルですが、ドキュメントの読み込みからベクトル化、検索、回答生成まで、RAG の基本的な要素が詰まっています。ぜひ、皆さんのオリジナルのデータで試してみてください！
