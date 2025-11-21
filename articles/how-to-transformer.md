---
title: "Transformerとは？仕組みと注意点を今更ながら解説！"
emoji: "✨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ai","llm","transformer"]
published: true
publication_name: "hibari_inc"
---

「LLM」という言葉が一般的になり、ChatGPTやClaude、Geminiなどのサービスを日常的に触る人も増えました。
しかし、これらの根幹にある**Transformer**について、「なんとなく分かった」で済ませている方も多いのではないでしょうか。

そこで、私自身の復習も兼ねて「Transformerとは何なのか」を、アテンション機構(Attention Mechanism)から整理していきます。

# 背景：RNN・CNNの限界

まず初めに、Transformerが登場する前の話から始めます。
Transformer登場以前、自然言語処理(NLP)は主に**RNN**(Recurrent Neural Network)や**LSTM**(Long Short-Term Memory)が主流でした。
これらは時系列データに強い構造を持ち、文脈を逐次処理する点で優れていました。しかし、

  * 並列計算が難しい
  * 長文になると文脈が遠くなり、情報が失われやすい(長期依存問題)

という欠点がありました。

一方、CNNをNLPに応用する試みもありましたが、位置依存や長文理解には不向きでした。

# Attention機構の登場

そこで登場したのが**Attention**です。
Attentionとは、簡単に言えば、入力全体の中で「今注目すべき単語にどれだけ集中するか」を定量化する仕組みです。

入力文の各単語を**Query(Q)**, **Key(K)**, **Value(V)** というベクトルに変換し、「QueryとKeyの内積」に基づいて重みを求め、その重みをValueにかけ合わせて文脈情報を抽出します。

## QKVのイメージ：図書館での文献検索

このQKVの役割は、よく図書館での文献検索に例えられます。

  * **Query (Q):** あなたが知りたい情報(例：「AIに関する本」という検索クエリ)
  * **Key (K):** 図書館の蔵書データベースにある、各本の「キーワード」や「見出し」(例：「AI」「機械学習」「深層学習」)
  * **Value (V):** キーに対応する「本そのものの中身」

Attentionは、知りたい情報のQuery(Q)と各本のKey(K)の関連度を計算し(＝内積)、関連度が高い本のValue(V)を重点的に(＝加重平均)、あなたに提示する仕組みに似ています。

![Scaled Dot-Product Attention の概念図](/images/how-to-transformer/scaled_dot_product_Attention.png)
図1. Scaled Dot-Product Attention の概念図
(引用：Attention Is All You Need (Vaswani et al., 2017))

この計算により、系列全体を俯瞰しながら重要な単語同士の関係性を学習できます。
この考え方が「Self-Attention」と呼ばれるものです。

**Self-Attentionが捉える文脈の例**

例えば、「`その動物は疲れ切っていたので、道を渡らなかった。`」という文があったとします。

RNNでは「その」と「動物」の距離が近いため関係性を学習しやすいですが、「(**それ**は)疲れ切っていた」の「**それ**」が、文頭の「**動物**」を指していることを学習するのは困難でした(長期依存問題)。

Self-Attentionは、文全体を一度に見渡すため、「それ」という単語(Query)が、文中のどの単語(Key)と最も関連が深いか(この場合は「動物」)を直接計算し、その文脈を強く反映させることができます。

# Transformerの構造概要

Vaswaniらの論文「Attention is All You Need」(2017)で発表されたTransformerは、この**Self-Attentionを多層に積み重ねた構造**を持ちます。

![Transformerの全体構造イメージ](/images/how-to-transformer/transformer.png)
図2. Transformerの全体構造イメージ
(引用：Attention Is All You Need (Vaswani et al., 2017))

## Encoder

各層は以下の二つのブロックで構成されます。

1.  Multi-Head Self-Attention
2.  Position-wise Feed Forward Network

位置情報を保持するために**Positional Encoding**を付加し、系列順序を明示的に学習させます。

## Decoder

Encoderの出力を参照しながら、自己回帰的に出力を生成します。
ここでもSelf-Attentionを使いますが、「未来のトークンを見ない」ように**Masked Attention**を適用します。

# Multi-Head Attentionの意義

なぜ「Multi-Head」なのか？
一つのAttentionだけでは単語間の関係を単一の視点でしか捉えられません。
複数のヘッド(QKVのセット)を用いることで、

  * 文法的な関係(例：「主語と動詞」)
  * 意味的な関係(例：「質問と答え」や、前述の「"それ"と"動物"」)

など、**異なる観点での依存関係を同時に学習**できます。

![Multi-Head Attentionの概要](/images/how-to-transformer/multi_head_attention.png)
図3. Multi-Head Attentionの概要
(引用：Attention Is All You Need (Vaswani et al., 2017))

## LLMへの発展

Transformer構造はBERT、GPT、T5などの基礎となりました。
特にGPT系列ではDecoder部分をベースに、自己回帰型生成を採用。
BERTはEncoder部分をベースに、文脈理解に特化したモデルとして発展しました。

# まとめ

Transformerの本質は「Attentionにより系列全体を一度に見渡す」という発想です。
これにより、RNN時代の逐次処理の制約を超え、巨大な言語モデルが現実化しました。

今後もLLMやマルチモーダルモデルの基盤として、Transformerの理解は避けて通れません。
改めて基本を整理しておくことで、より高次なモデル設計や推論の理解につながるでしょう。

# 参考文献

A. Vaswani et al., "Attention Is All You Need," NIPS, 2017.