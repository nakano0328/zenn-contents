---
title: "Google LabsのFlowとは？Veoで映像を作るAIフィルムメイキングツールをざっくり解説"
emoji: "🎬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["google","動画生成ai","veo"]
published: false
publication_name: "hibari_inc"
---

こんにちは！株式会社 HIBARI の中野です。

Google Labs から公開されている**Flow**は、生成モデル**Veo**を使って「クリップ → シーン → 物語」へ組み上げていくことを意識した**動画作成ツール**です。

Whisk や Opal のように “触って理解できる系” の Google Labs ですが、Flow は特に「動画生成を“単発ガチャ”で終わらせず、ストーリーとして繋げる」方向に振っているのが面白いところです。

Flow: https://labs.google/fx/tools/flow

![Flowのトップ画面](/images/what-is-flow/flow_screen.png)

# Flowとは？

Flowは、Googleが提供する**Google Labs**の機能の1つです。テキストや画像から**短い動画**を生成し、それらをタイムライン上で並べて物語っぽく繋いでいくためのツールです。
単発の生成で終わりがちな動画生成を、「クリップ → シーン → 物語」 という制作単位で扱えるように設計されている点が特徴です。

公式ヘルプでは、Flow を「AI filmmaking tool」と位置付けており、生成モデルとして**Veo**を中心に、機能によって**Imagen 4 / Gemini**も使われます。

一般的な動画生成ツールが「1回の生成＝1本の動画」で完結しがちなのに対し、Flow は 生成した後の**編集と継続生成**を前提にしています。

# Flowが「動画生成の道具」に見える理由

Flow は「動画を生成して終わり」になりがちな部分を、シーンづくりに寄せているのが特徴です。具体的には、

- クリップを繋ぐためのタイムライン(Scenebuilder)
- “続きのショット”を作るための機能(Jump to / Extend)
- 後からの修正(Insert/Remove、カメラ位置・モーションの変更)

といった、ストーリーを組むための道具立てが最初から揃っています。

# Flowでできること(機能別の概要)

## Text to Video(テキスト→動画)

文章で「何が/何を/どこで/どういう光で/どんなスタイルで」を指定して生成します。

ポイントは、**被写体・動作・環境・ライティング・画のテイスト**まで書くことです(公式も “Be specific” を推奨)。

![テキストから動画を生成した様子](/images/what-is-flow/text_to_video.png)

## Frames to Video(画像フレーム→動画)

開始/終了フレームを指定して、その間の動きや遷移を生成します。

また、プロンプトでカメラワークを書く代わりに、UI側で**Camera**を選んで指定できる導線があります(モデル/機能により対応状況が異なります)。

> スクショ差し替え：Frames to Video(start/end frame を追加できる＋`Camera` ボタンが見える)

## Ingredients to Video(参照画像を複数渡して動画)

キャラクター・物体・背景など、複数の参照画像(ingredients)を与えて、同じ見た目を維持しつつ合成するモードです。

参照画像を用意できると、キャラや小道具の一貫性が出しやすくなります。公式では、被写体やプロダクト参照は**単色背景/切り抜きに近い画像**が良いとされています。

> スクショ差し替え：Ingredients to Video(複数画像を追加する drawer/入力欄が見える)

## Scenebuilder(シーンを組む)

生成したクリップを並べて、順番を入れ替えたり、前後をトリムしたりしながら、シーンを組み立てます。

さらに、次のショットを作るための機能として:

- Jump to：直前のクリップの続きになるように次クリップを生成
- Extend：既存クリップの尺を伸ばす

が用意されています。

※公式ヘルプ時点では「Scenebuilder の状態はプロジェクトを離れるとリセットされる(クリップ自体は保存される)」という注意書きがあり、改善予定とされています。

> スクショ差し替え：Scenebuilder(複数クリップが並ぶタイムライン)

> スクショ差し替え：Add メニュー(`Jump to…` / `Extend…` が出ている)

## 生成後の編集(Insert/Remove、カメラ変更)

生成済みクリップに対して、以下のような編集が案内されています。

- Insert / Remove：指定した領域にオブジェクトを挿入/削除
- Camera Position / Camera Motion：カメラ位置や動きを変えて撮り直す

「まず出してから直す」前提のワークフローに寄っている印象です。

# 画像生成・編集もできる(Create Image / Nano Banana)

Flow は動画だけでなく、プロジェクト内で画像の生成・編集もできます。

- 画像生成:**Create Image**
- 画像編集: 公式では**Nano Banana**モデルで反復編集できる旨が記載
- モデル選択: プロンプト欄右上で選択(公式ヘルプはモデル名を列挙していないため、UI表示を参照)

生成した画像は、動画の ingredients やフレームとして再利用できます。

> スクショ差し替え：Images タブ(生成画像一覧＋`Edit`/`Add To Prompt` が分かる)

# モデル(Veo 2 / Veo 3.1)と、できることの違い

公式ヘルプでは、主に以下の位置づけです。

- Veo 2：デフォルト。Flow のフル体験を支える中心モデル(推奨)
- Veo 3.1：新しいモデル。より高品質で、実験的な音声生成に対応

注意点として、**選んだ機能が Veo 3.1 で未対応の場合、Veo 2 に自動で切り替わる**旨が明記されています。

また、音声生成は Veo 3.1 の実験機能として説明されており、失敗時は返金(クレジット返却)されるケースがあるとされています。

# クレジット(料金)の考え方：生成=都度課金

Flow は**AI Credits**を消費して生成します。モデル/モードで必要クレジットが変わります。

無課金でも毎月100 credits(一部地域は180)が付与されます。ただし無料クレジットは **Veo 3.1(Fast/Quality)** の生成にのみ使え、**Veo 2** や **Ingredients to Video** は対象外です。無料クレジットは初回生成日を起点に月次更新で、未使用分は繰り越されません。

公式ヘルプの記載例(変更される可能性あり)を、要点だけ拾うとこんな感じです。

- Veo 2 - Fast：10 credits / generation
- Veo 2 - Quality：100 credits / generation
- Veo 3.1 - Fast：20 credits(Ultra は 10 credits)/ generation
- Veo 3.1 - Fast (lower priority)：20 credits(Ultra は Included)/ generation
- Veo 3.1 - Quality：100 credits / generation
- Video edits：20 credits / generation
- 1080p upscaling：Included
- 4K upscaling：50 credits

また「1リクエスト=1生成ではない」点にも注意が必要です(例：1回の操作で動画が2本生成される場合がある)。

※ Pro/Ultra の追加クレジット(Top-up)購入は対応地域のみで、日本は対象外とされています。

# ウォーターマーク(SynthID / 表示ウォーターマーク)

公式ヘルプでは、Flow(Veo/Imagen)の生成物には**不可視の SynthID ウォーターマーク**が入ることが明記されています。また追加措置として、動画については**Google One Pro ユーザーは可視ウォーターマークあり／Google One Ultra ユーザーは可視ウォーターマークなし**とされています(つまりProに入っても可視透かしが消えるわけではありません)。

利用規約や公開先のポリシーと合わせて、扱いは事前に確認するのが安全です。

# 使える条件(地域・年齢・ブラウザ)

公式ヘルプでは、利用条件として「年齢確認済みで18歳以上」「対応地域であること(※VPNでは解決しない旨が明記)」「ブラウザはデスクトップのChromium系(Chrome推奨)」が案内されています。

また、サブスクリプションとクレジット周りは複数の案内があり、無料クレジット枠/プラン別枠/Workspace枠などが存在します。最新の状態はヘルプとプロダクト内の表示で確認するのが確実です。

※公式ヘルプ内で「Google AI Pro/Ultra」「Google One Pro/Ultra」のように表記が混在している箇所があります(同一の課金体系を指している前提で読めますが、念のため最新UIでの表示名も確認してください)。

# まず何を試す？(最短のおすすめ手順)

「触って理解する」目的なら、この順が最短だと思います。

1.**Text to Video(Veo 3.1 Fast)**で短いプロンプトを3本くらい生成して“モデルの癖”を見る
2. 気に入ったクリップを**Scenebuilder**に入れて並べ、**Jump to**で繋がりを作る
3. 一貫性が欲しくなったら**Frames to Video**(first frame)→ さらに必要なら**Ingredients to Video**

この3ステップのスクショを貼るだけで、紹介記事として十分読み応えが出ます。

# まとめ

Flow は、Veo を使った動画生成を「単発の生成」から「シーン/ストーリーの組み立て」へ持っていくための、Google Labs らしい実験ツールです。

動画生成に触ったことがある人ほど、「次のショットをどう繋ぐか」「キャラをどう維持するか」で悩むと思うので、そこに機能として答えを用意している点が面白いところです。

(追記予定)実際に使ってみた所感や、プロンプト例/失敗例/コスト感は別記事でまとめます。

# 参考リンク

- Flow: https://labs.google/fx/tools/flow
- Get started with Flow: https://support.google.com/labs/answer/16353333?hl=en
- Generate videos using Flow: https://support.google.com/labs/answer/16353334?hl=en
- Generate & edit images in Flow: https://support.google.com/labs/answer/16729550?hl=en
- Learn about Flow models & supported features: https://support.google.com/labs/answer/16352836?hl=en
- Manage your AI Credits in Flow: https://support.google.com/labs/answer/16526234?hl=en
- Where you can use Flow: https://support.google.com/labs/answer/16353544?hl=en
- Stay up-to-date with changes in Flow: https://support.google.com/labs/answer/16358089?hl=en

この記事は株式会社HIBARIのテックブログからの転載です。
元記事はこちら：[自社ブログ](https://hibari-ai.com/techblog/use-claude-cowork)