---
title: "Geminiを使ってずんだもんと会話してみた"
emoji: "🫛"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python","AI","gemini","voicevox"]
published: true
published_at: 2025-08-02 13:00 
publication_name: "hibari_inc"
---

こんにちは！株式会社 HIBARI の中野と申します。

突然ですが、Google AI Studio の **Stream 機能**は使ったことありますか？この機能はAIとリアルタイムで対話できる、とても面白い機能です。ただ、AIの音声がgoogle側が用意したもの数種類しかなく、自由に変更できないのが少し残念な点でした。

「それなら、自分の好きなキャラクターと話せるようにすればもっと面白いのでは？」

なので今回は、ずんだもんとリアルタイムで会話できる簡単なシステムを開発してみました！

▼ GitHubはこちら ▼
https://github.com/nakano0328/realtime-zunda


# このシステムでできること

今回作ったシステムは、マイクに向かって話しかけるだけで、ずんだもんがリアルタイムで応答してくれます。

# 主な機能
- マイクからの声をテキストに変換します。
- Google AI StudioのGemini APIが、ずんだもんのキャラクターになりきって返事を考えます。
- VOICEVOXが、生成されたテキストをずんだもんの声で読み上げます。

以上の処理を連続的に行うことで、途切れることのない会話体験を実現します。
「さようなら」や「バイバイ」と言うと、会話を終了してくれる機能もつけました。

# 技術概要

このアプリケーションは、いくつかの技術を連携させて動いています。全体の流れは以下の通りです。

## 1.音声入力(`speech_recognition`,`pyaudio`)

まず、マイクから入力された音声をpyaudioで受け取ります。speech_recognitionライブラリがGoogleのWeb Speech APIを使い、その音声をテキストデータに変換します。

## 2.応答生成(`google-generativeai`)

テキスト化された音声は、**Gemini**に送られます。この時にテキストと同時に、ずんだもんの**キャラクター設定**を送っています。これにより、Geminiの生成した文章がずんだもんっぽくなります。

```py
# main.pyより抜粋
ZUNDAMON_PROMPT = """
あなたは「ずんだもん」というキャラクターです。
以下のルールに従って、ユーザーと会話してください。

# ルール
- 東北訛りの元気な女の子として振る舞ってください。
- 一人称は「ボク」です。
- 口癖は「〜のだ」「〜なのだ」です。
- ユーザーからの質問には、元気いっぱいに、少しお茶目に答えてください。
"""
```

## 3.音声合成(`requets`,VOICEVOX)

Geminiが生成したずんだもんの返答を、ローカルで起動しているVOICEVOXを用いてテキストを音声にします。

## 4.音声生成(`pydub`)

受け取った音声データを`pydub`ライブラリを用いて再生します。これでずんだもんの声がスピーカーから聞こえてきます。

これらの処理をループさせることでリアルタイムな会話が成立します。

# 使ったもの

実行には以下のものを使いました。

## ソフトウェア

- Python (3.9以上 推奨)
- VOICEVOX

## ライブラリ

- `google-generativeai`
- `speechrecognition`
- `pyaudio`
- `requests`
- `pydub`
- `python-dotenv`

## APIキー

- Google AI Studio の API キー

## ファイル構成

```bash
realtime-zunda/
├── main.py             # メインプログラム
├── requirements.txt    # Python依存関係
├── README.md           # このファイル
├── .gitignore          # Git除外ファイル
└── .env                # 環境変数（APIキー等）
```

# セットアップ方法

このシステムは**簡単**で**無料**なので、皆さんのPCでも動かせます。ぜひずんだもんと対話してみてください。

## 1. リポジトリのクローン

以下のコマンドを実行し、リポジトリをクローンし、ディレクトリを移動します。

```bash
git clone https://github.com/your-username/realtime-zunda.git
cd realtime-zunda
```

## 2.APIキーの設定

1. [Google AI Studio](https://aistudio.google.com/)でAPIキーを取得します。
2. プロジェクトのルートディレクトリに`.env`ファイルを作成します。
3. `.env`ファイルに以下の内容を追加します。

```bash
GOOGLE_API_KEY=your_actual_api_key_here #ここを書き換える
```

ここで、`your_actual_api_key_here`の部分に`1.`で取得したAPIキーを書いてください。

**注意**: APIキーを流出しないように気をつけてください。

## 3.依存関係のインストール

以下のコマンドを実行し、ライブラリをインストールします。
```bash
pip install -r requirements.txt
```

## 4. VOICEVOXのセットアップ

VOICEVOXを起動してください。

VOICEVOXを持っていない場合は、[VOICEVOX公式サイト](https://voicevox.hiroshiba.jp/)からソフトをインストールしてください。

## 5. プログラムの実行

VOICEVOXを起動した状態で、ターミナルで以下のコマンドを実行します。

```bash
python main.py
```

「どうぞ、話しかけてくださいのだ。」と表示されたら成功です！マイクに向かって話しかけてみてください。

# まとめ

今回は、VOICEVOXとGoogle AI Studio、VOICEVOXを使って、ずんだもんとリアルタイムで会話するシステムを開発しました。音声認識、生成AI、音声合成という3つの技術を組み合わせることで、キャラクターとの対話を実現できました。

ソースコードは[GitHub](https://github.com/nakano0328/realtime-zunda?tab=readme-ov-file)で公開しているので、興味のある方はぜひ改造して遊んでみてください。
皆さんも、好きなキャラクターと話せる自分だけのアプリを作ってみてはいかがでしょうか？

最後まで読んでいただき、ありがとうございましたのだ！

**免責事項**

本システムの利用は、利用者の自己責任において行うものとします。
本システムの利用により生じたいかなる損害やトラブルについて、開発者は一切の責任を負いません。