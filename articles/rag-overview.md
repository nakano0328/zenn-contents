---
title: "RAGについて（未定）"
emoji: "🔍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AI", "LLM", "機械学習", "自然言語処理", "RAG"]
published: false
---

初めまして！株式会社HIBARIの中野と申します。これからの取り組みとしてテックブログを書き始めることにしました。よろしくお願いします！

# RAGとは？
RAG(Retrieval-Augmented Generation)は日本語では「検索拡張生成」などと言われています。大規模言語モデル(LLM)の持つ文章生成能力と、外部の知識を検索する能力を組み合わせた技術です。

## なぜRAGが必要なのか
今までのLLMでは特定のドメインに特化した情報を答えることができません。例えば社内会議の情報や会社独自の知識などを答えることができません。LLM自体が学んでいないことを解答しようとするともっともらしい嘘「ハルシネーション」をしてしまいます。RAGはこれらを解決することができます。

# RAGの仕組み
RAGの処理のフローは大きく分けて「検索(Retriever)」と「生成(Generator)」の２つに分かれます。
まずはユーザーからの質問をベクトルに変換します。このベクトルを検索します。次に検索した情報と元の質問を合体してLLMへ新しいプロンプトを入力します。

## 検索(Retriever)

## 生成(Generator)

# ファインチューニングとの違い

# RAGの活用例

# RAGのメリット・デメリット

# まとめ

# 参考文献
P. Lewis et al. "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks." In Advances in Neural Information Processing Systems, vol. 33, pp. 9459-9474. Curran Associates, Inc., 2020.