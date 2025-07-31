---
title: "Google AI Studio (Gemini) API と LangChain で、RAG を実装してみた"
emoji: "🧠"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AI", "LLM", "gemini", "RAG"]
published: true
published_at: 2025-08-04 11:00
publication_name: "hibari_inc"
---

こんにちは！株式会社 HIBARI の中野と申します。

今回は Google AI Studio の Gemini API と LangChain を活用して、独自のテキストデータに基づいた受け答えができる RAG システムを開発しましたので、その仕組みと動かし方についてご紹介します。

このシステムを使えば、指定したディレクトリ内のテキストファイルをナレッジベースとして、AI がその内容を元に質問に回答してくれるようになります。

RAG についてはこちらの記事を参照してください。

https://zenn.dev/hibari_inc/articles/rag-overview

# システム概要

この RAG システムは、大きく分けて 2 つのプロセスで構成されています。

## 事前準備 (ベクトルデータベースの構築)

`data` ディレクトリ内のテキストファイルを読み込み、内容を分割（チャンク化）します。そして、Google の `embedding-001` モデルを使って各チャンクをベクトル化し、`ChromaDB` というベクトルデータベースに保存します。この処理は `build_vectordb.py` スクリプトが担当します。

## 質問応答 (チャットの実行)

ユーザーが質問を入力すると、その質問内容もベクトル化されます。そして、`ChromaDB` 内に保存されているデータの中から、質問ベクトルと類似度の高いチャンクを検索します。検索で得られた関連情報をコンテキスト（文脈）として、質問文と共に `gemini-pro` モデルに渡し、最終的な回答を生成させます。この処理は `chat.py` スクリプトが担当します。

## ファイル構成

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

# コード解説

主要な 2 つのスクリプト `build_vectordb.py` と `chat.py` の中身を解説します。

## `build_vectordb.py`: ベクトルデータベースの構築

このファイルは、`data`ディレクトリ内のテキストファイルからベクトルデータベースを作成するコードです。

### ドキュメントの読み込み (`load_documents_from_directory`)

`TextLoader` を使って `data` ディレクトリ内の全 `.txt` ファイルを読み込みます。

```py
def load_documents_from_directory(directory_path):
    """指定したディレクトリ内の全てのtxtファイルを読み込む"""
    documents = []

    # dataディレクトリ内の全てのtxtファイルを取得
    txt_files = glob.glob(os.path.join(directory_path, "*.txt"))

    if not txt_files:
        print(f"警告: {directory_path} ディレクトリにtxtファイルが見つかりません。")
        return documents

    print(f"{len(txt_files)}個のtxtファイルが見つかりました。")

    for file_path in txt_files:
        try:
            loader = TextLoader(file_path, encoding='utf-8')
            file_documents = loader.load()
            documents.extend(file_documents)
            print(f"読み込み完了: {os.path.basename(file_path)}")
        except Exception as e:
            print(f"エラー: {file_path} の読み込みに失敗しました - {e}")

    return documents
```

### ドキュメントの分割 (`split_documents`)

読み込んだドキュメントが長文である場合、そのままでは扱いにくいため、`RecursiveCharacterTextSplitter` を使って適切な長さのチャンク（かたまり）に分割します。これにより、後続のベクトル化や検索の精度を高めます。

```py
def split_documents(documents):
    """ドキュメントをチャンクに分割"""
    text_splitter = RecursiveCharacterTextSplitter(
        chunk_size=1000,
        chunk_overlap=200,
        separators=["\n\n", "\n", "。", "、", " ", ""]
    )

    splits = text_splitter.split_documents(documents)
    print(f"ドキュメントを{len(splits)}個のチャンクに分割しました。")

    return splits
```

### ベクトルストアの作成 (`create_vectorstore`)

分割されたチャンクを、`GoogleGenerativeAIEmbeddings` (エンベディングモデルは `embedding-001`) を使って一つずつベクトルに変換します。そして、それらのベクトルデータを `Chroma.from_documents` を使って `ChromaDB` に保存します。データベースは `./vectordb` ディレクトリに永続化されます。

```py
def create_vectorstore(documents):
    """ベクトルストアを作成"""
    # Google AI Studio APIキーの確認
    api_key = os.getenv("GOOGLE_API_KEY")
    if not api_key:
        raise ValueError("GOOGLE_API_KEYが設定されていません。.envファイルを確認してください。")

    # Google Generative AI Embeddingsを初期化
    embeddings = GoogleGenerativeAIEmbeddings(
        model="models/embedding-001",
        google_api_key=api_key
    )

    # Chromaベクトルストアを作成
    vectorstore = Chroma.from_documents(
        documents=documents,
        embedding=embeddings,
        persist_directory="./vectordb"
    )

    # Chroma 0.4.x以降では自動で永続化されるため、persist()は不要
    print("ベクトルストアが正常に作成・保存されました。")
```

## `chat.py`: チャットの実行

このファイルは構築されたベクトルデータベースを用いて、ユーザとの対話を実現するコードです。

### ベクトルストアの読み込み (`load_vectorstore`)

`build_vectordb.py` で作成・保存された `./vectordb` ディレクトリから、`Chroma` を使ってベクトルストアを読み込みます。ここでも `GoogleGenerativeAIEmbeddings` が必要になります。

```py
def load_vectorstore():
    """保存されたベクトルストアを読み込む"""
    # Google AI Studio APIキーの確認
    api_key = os.getenv("GOOGLE_API_KEY")
    if not api_key:
        raise ValueError("GOOGLE_API_KEYが設定されていません。.envファイルを確認してください。")

    # Embeddingsを初期化
    embeddings = GoogleGenerativeAIEmbeddings(
        model="models/embedding-001",
        google_api_key=api_key
    )

    # 保存されたベクトルストアを読み込み
    vectorstore = Chroma(
        persist_directory="./vectordb",
        embedding_function=embeddings
    )

    return vectorstore
```

### QA チェーンの作成 (`create_qa_chain`)

この関数が RAG システムの心臓部です。プロンプトに基づいてベクトルデータベース内を検索し、その情報とともにプロンプトを拡張して回答を生成します。

- **LLM**
  `GoogleGenerativeAI` を使って、回答生成モデルとして `gemini-2.5-flash` を初期化します。
- **Retriever**
  読み込んだベクトルストアを `as_retriever` メソッドで検索機（Retriever）に変換します。`search_kwargs={"k": 5}` を指定することで、質問と関連性の高い上位 5 つのチャンクを検索するように設定しています。
- **PromptTemplate**
  LLM に渡すプロンプトのテンプレートを定義します。「以下の情報を基に...」という指示と共に、検索で得られた情報 (`context`) とユーザーの質問 (`question`) を埋め込めるようにしています。
- **RetrievalQA**
  これらを `RetrievalQA.from_chain_type` を使って一つにまとめ、QA チェーンを構築します。

```py
def create_qa_chain():
    """QAチェーンを作成"""
    # Google AI Studio APIキーの確認
    api_key = os.getenv("GOOGLE_API_KEY")
    if not api_key:
        raise ValueError("GOOGLE_API_KEYが設定されていません。.envファイルを確認してください。")

    # LLMを初期化（Gemini Pro）
    llm = GoogleGenerativeAI(
        model="gemini-2.5-flash",
        google_api_key=api_key,
        temperature=0.7
    )

    # ベクトルストアを読み込み
    vectorstore = load_vectorstore()

    # プロンプトテンプレートを定義
    prompt_template = """以下の情報を基に、質問に対して正確で詳細な回答を提供してください。
情報に基づいて回答できない場合は、「提供された情報では回答できません」と答えてください。

関連情報:
{context}

質問: {question}

回答:"""

    PROMPT = PromptTemplate(
        template=prompt_template,
        input_variables=["context", "question"]
    )

    # RetrievalQAチェーンを作成
    qa_chain = RetrievalQA.from_chain_type(
        llm=llm,
        chain_type="stuff",
        retriever=vectorstore.as_retriever(
            search_type="similarity",
            search_kwargs={"k": 5}  # 関連する上位5つのチャンクを取得
        ),
        chain_type_kwargs={"prompt": PROMPT},
        return_source_documents=True
    )

    return qa_chain
```

### メイン処理 (`main`)

ユーザーからの入力を受け付け、QA チェーン (`qa_chain.invoke`) を実行して回答を生成します。回答と共に、どのドキュメントチャンクを参照したかも表示する機能を実装しています。

```py
def main():
    """メイン処理"""
    print("🤖 RAGチャットシステムを起動しています...")

    # ベクトルストアの存在確認
    if not os.path.exists("./vectordb"):
        print("エラー: ベクトルストアが見つかりません。")
        print("まず build_vectordb.py を実行してベクトルストアを作成してください。")
        return

    try:
        # QAチェーンを作成
        qa_chain = create_qa_chain()
        print("✅ RAGシステムの準備が完了しました！")
        print("質問を入力してください。終了するには 'quit' または 'exit' と入力してください。\n")

        while True:
            # ユーザーの質問を取得
            question = input("🙋 質問: ").strip()

            # 終了条件
            if question.lower() in ['quit', 'exit', '終了', 'やめる']:
                print("チャットを終了します。お疲れさまでした！")
                break

            # 空の質問をスキップ
            if not question:
                print("質問を入力してください。")
                continue

            try:
                # QAチェーンで回答を生成（新しいinvokeメソッドを使用）
                print("🤔 回答を生成中...")
                result = qa_chain.invoke({"query": question})

                # 回答を表示
                print(f"\n🤖 回答: {result['result']}")

                # ソースドキュメントがある場合は表示
                if result.get('source_documents'):
                    print(f"\n📚 参照元: {len(result['source_documents'])}個のドキュメントチャンク")
                    for i, doc in enumerate(result['source_documents'][:2], 1):  # 上位2つのソースを表示
                        source = doc.metadata.get('source', '不明')
                        filename = os.path.basename(source) if source != '不明' else '不明'
                        print(f"   {i}. {filename}")

                print("-" * 50)

            except Exception as e:
                print(f"エラーが発生しました: {e}")
                print("もう一度お試しください。")

    except Exception as e:
        print(f"初期化エラー: {e}")
```

---

# 動かし方

この RAG を実際に動かしてみましょう。

## 1. リポジトリのクローン

以下のコマンドを実行し、リポジトリをクローンし、ディレクトリを移動します。

```bash
git clone https://github.com/nakano0328/rag_google.git
cd rag_google
```

## 2. API キーの設定

1. [Google AI Studio](https://aistudio.google.com/)で API キーを取得します。
2. プロジェクトのルートディレクトリに`.env`ファイルを作成します。
3. `.env`ファイルに以下の内容を追加します。

```bash
GOOGLE_API_KEY=your_actual_api_key_here #ここを書き換える
```

ここで、`your_actual_api_key_here`の部分に取得した API キーを書いてください。

**注意**: API キーを流出しないように気をつけてください。

## 3. 依存関係のインストール

以下のコマンドを実行し、ライブラリをインストールします。

```bash
pip install -r requirements.txt
```

ここで、**venv**という仮想環境を使用することをおすすめします。**venv**とは、Python の**仮想環境**の 1 つです。プロジェクトごとに独立した環境を作成・管理できます。

以下のコマンドをプロジェクトのルートディレクトリで実行してください。

```bash
python -m venv [venvname] #[venvname]はvenvにすることをおすすめします。
```

以下のコマンドを使うことで venv を起動できます。

```bash
# Linux,Macの場合
source [venvname]/bin/activate

# Windowsの場合
.\[venvname]\Scripts\activate
```

## 4. データの準備

プロジェクト内に `data` ディレクトリを作成し、ナレッジベースとしたいテキストファイル（`.txt` 形式）をその中に配置してください。

## 5. ベクトルデータベースの構築

最初に、以下のコマンドを実行して `data` ディレクトリのファイルからベクトルデータベースを構築します。

```bash
python build_vectordb.py
```

実行が完了すると、プロジェクトルートに `./vectordb` ディレクトリが作成されます。

## 6. チャットの開始

データベースの準備ができたら、いよいよチャットを開始できます。

```bash
python chat.py
```

ターミナルに「🙋 質問:」と表示されたら、自由に質問を入力してみてください。`data` ディレクトリのテキスト情報に基づいた回答が生成されます。チャットを終了するには `quit` または `exit` と入力してください。

**注意**: Google AI Studio の API の制限は 1 分間に 15 回までです。

# まとめ

今回は、Google AI Studio (Gemini) API と LangChain を用いて、独自のデータに対応する RAG システムを構築する方法をご紹介しました。

コード自体は比較的シンプルですが、ドキュメントの読み込みからベクトル化、検索、回答生成まで、RAG の基本的な要素が詰まっています。ぜひ、皆さんのオリジナルのデータで試してみてください！

**免責事項**

- Google AI Studio API の利用には料金が発生する場合があります。API の利用料金や制限については、[Google AI Studio](https://aistudio.google.com/)の公式ドキュメントをご確認ください。
- 本記事で紹介した内容により発生した損害やトラブルについて、筆者および株式会社 HIBARI は一切の責任を負いません。
- API キーなどの認証情報は適切に管理し、公開リポジトリなどに含めないよう十分にご注意ください。
- 本記事の内容は執筆時点（2025 年 7 月）のものであり、今後のライブラリや API の更新により動作しなくなる可能性があります。
