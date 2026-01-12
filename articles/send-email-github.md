---
title: "GitHub ActionsとPythonで、自動メール通知システムを実装してみた"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python", "github", "githubactions", "自動化", "個人開発"]
published: true
publication_name: "hibari_inc"
---

こんにちは！株式会社 HIBARI の中野と申します。

今回は、無料利用枠の範囲で使える`GitHub Actions`と `Python`を用いてサーバシステムの構築をテストしてみました。

**※2025/12/12 追記**
本記事で紹介している手法について、GitHub Actionsの利用規約や実行タイミングに関する重要な注意点があります。記事末尾の[免責事項](#免責事項)を必ずご一読ください。

# 今回作ったもの

今回作ったシステムの構成は、とてもシンプルです。

- `GitHub Actions`の`schedule`トリガーを利用して、イベントを発火させます。
- リポジトリに置いた**Python スクリプト**が実行され、メールの内容を作成します。
- Gmail アカウントなどを使って、指定したアドレスに**メールを自動送信**します。

これだけでできました。

# 実装の手順

ここからは、私が実際に手を動かした手順をまとめていきます。

## 1.リポジトリの準備

リポジトリを作りました。

ファイル構成は以下のようになりました。

```bash
.
├── .github/
│   └── workflows/
│       └── main.yml
├── main.py
└── requirements.txt
```

また、今回作成したコードは公開しています。

https://github.com/nakano0328/send_email

## 2.メールの送信処理(main.py)を書く

次にシステムの核となるメール送信部分を Python で実装しました。

このコードは「**決められた固定のメッセージを送る**」だけのコードにしています。

内容を変更したい場合は`create_email_content()`関数内を変更してください。

```py
import os
import smtplib
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
import datetime

def create_email_content():
    """
    メールの件名と本文(HTML)を生成する関数。
    ★★この関数の中身を、あなたの送りたい内容に合わせて書き換えてください★★
    """
    print("メール内容の作成を開始します...")

    # --- ▼▼▼ ここを自由にカスタマイズ ▼▼▼ ---

    # メール件名
    today_str = datetime.date.today().strftime("%Y/%m/%d")
    subject = f"{today_str}の定例通知"

    # メール本文 (HTML形式で自由に記述できます)
    html_content = """
    <html>
    <body>
        <h1>おはようございます！</h1>
        <p>今日のタスクリストです。</p>
        <ul>
            <li>タスクA：資料作成</li>
            <li>タスクB：メール返信</li>
            <li>タスクC：会議準備</li>
        </ul>
        <p>今日も一日頑張りましょう！</p>
    </body>
    </html>
    """

    # --- ▲▲▲ ここまでカスタマイズ ▲▲▲ ---

    print("メール内容の作成が完了しました。")
    return subject, html_content

def send_email(subject, html_content):
    """メールを送信する関数 (この関数は変更不要です)"""
    smtp_server = os.environ.get('SMTP_SERVER')
    smtp_port = os.environ.get('SMTP_PORT')
    smtp_username = os.environ.get('SMTP_USERNAME')
    smtp_password = os.environ.get('SMTP_PASSWORD')
    recipient_email = os.environ.get('RECIPIENT_EMAIL')

    if not all([smtp_server, smtp_port, smtp_username, smtp_password, recipient_email]):
        print("エラー: メール送信に必要な環境変数が設定されていません。")
        return

    msg = MIMEMultipart('alternative')
    msg['Subject'] = subject
    msg['From'] = smtp_username
    msg['To'] = recipient_email
    msg.attach(MIMEText(html_content, 'html', 'utf-8'))

    try:
        with smtplib.SMTP(smtp_server, int(smtp_port)) as server:
            server.starttls()
            server.login(smtp_username, smtp_password)
            server.send_message(msg)
        print("メールを正常に送信しました。")
    except Exception as e:
        print(f"メールの送信に失敗しました: {e}")

if __name__ == '__main__':
    mail_subject, mail_body = create_email_content()
    send_email(mail_subject, mail_body)
```

## 3.依存関係のファイル (requirements.txt)

一般的に、インポートすべきライブラリをこのファイル内に書きます。しかし、今回は標準ライブラリ内で完結したので**何も書きません**。やったね。

## 4.GitHub Actions の設定

今回、GitHub の機能の 1 つである GitHub Actions を使ってメールを送ります。この GitHub Actions の設定をするファイルを書きます。
`.github/workflows/main.yml` というパスに YAML ファイルを作るのがお作法のよう(?)です。

```yml
# ワークフローの名前
name: 定期メール自動送信

on:
  # スケジュール実行のトリガー
  schedule:
    # cron形式でスケジュールを指定 (時刻はUTC)
    # 以下の例は、毎日午前8時 (JST) に実行 (UTCでは前日の23:00)
    - cron: "0 23 * * *"

  # 手動で実行するためのトリガー (テストに便利)
  workflow_dispatch:

jobs:
  send-scheduled-email:
    runs-on: ubuntu-latest

    steps:
      # 1. リポジトリのコードをチェックアウト
      - name: Checkout repository
        uses: actions/checkout@v4

      # 2. Pythonの環境をセットアップ
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: "3.11"

      # 3. (今回は不要だが) 依存ライブラリをインストール
      - name: Install dependencies
        run: pip install -r requirements.txt

      # 4. Pythonスクリプトを実行してメールを送信
      - name: Run script and send email
        env:
          SMTP_SERVER: ${{ secrets.SMTP_SERVER }}
          SMTP_PORT: ${{ secrets.SMTP_PORT }}
          SMTP_USERNAME: ${{ secrets.SMTP_USERNAME }}
          SMTP_PASSWORD: ${{ secrets.SMTP_PASSWORD }}
          RECIPIENT_EMAIL: ${{ secrets.RECIPIENT_EMAIL }}

        # スクリプトを実行(ファイル名を main.py と仮定)
        run: python main.py
```

時間の指定(９行目の`cron`)が UTC なのが戸惑いました。**UTC = 日本時間(JST) - 9 時間**なので、実行したい時間から 9 時間引いた時間を指定すればいいです。

### 補足 : cron の書式について

`cron`の書式は 5 つのフィールドで構成されており、それぞれが以下の時間を表しています。

```bash
┌───────────── 分 (0 - 59)
│ ┌───────────── 時 (0 - 23)
│ │ ┌───────────── 日 (1 - 31)
│ │ │ ┌───────────── 月 (1 - 12)
│ │ │ │ ┌───────────── 曜日 (0 - 6, 日曜日から土曜日)
│ │ │ │ │
* * * * *
```

`*`(アスタリスク)は「毎（まい）」を意味し、そのフィールドが取りうる全ての値を表します。

したがって、`0 23 * * *`は以下のようになります。

- 分: `0` → 0 分
- 時: `23` → 23 時(UTC)
- 日: `*` → 毎日
- 月: `*` → 毎月
- 曜日: `*`→ 毎日

これを組み合わせると、「毎日 23:00(UTC)に実行する」つまり、日本時間(JST)を用いると「毎日 8:00(JST)に実行する」という風になります。

## 5.パスワード管理

メールのパスワードなどをコードに直書きするのは絶対に NG❌。なので今回は`GitHub Secrets`を使用してパスワード等の環境変数の設定をしました。

以下の手順で行いました。

### STEP 1: パスワードの取得(Gmail の場合)

今回は Gmail に送信しました。Gmail はセキュリティのため、**通常の Google アカウントのログインパスワードは使えない**っぽいです。代わりに、このシステム専用の「**アプリパスワード**」を生成して対応しました。

以下はアプリパスワードの生成手順です。

1. Google アカウントの管理ページにアクセスします。
2. 左のメニューからセキュリティをクリックします。
3. 「Google へのログイン」セクションにある「**2 段階認証プロセス**」がオンになっていることを確認します。もしオフの場合は、画面の指示に従って有効にしてください。（アプリパスワードの利用には 2 段階認証が必須っぽいです。）
4. 2 段階認証を有効にした後、セキュリティページに戻り、「**アプリパスワード**」を探してクリックします。（見つからない場合は検索窓で「アプリパスワード」と検索してください。）
5. アプリパスワードの作成画面が表示されます。ここで、アプリ名にわかりやすい文字を入れて作成を押してください。
6. **16 文字のパスワード**が表示されます。このパスワードは**1 度しか表示されない**ため確実にメモして下さい。

### STEP 2: GitHub Secrets の設定ページへ移動

GitHub リポジトリの `Settings > Secrets and variables > Actions` へ移動します。

![setting.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4142158/6e3ee03d-0ad0-4b52-a264-bcaec421dadb.png)

### STEP 3: 5 つの Secret を登録

`New repository secret`ボタンを押して以下の情報を１つずつ登録します。

| Secret 名         | 値（設定する内容）                                                 |
| :---------------- | :----------------------------------------------------------------- |
| `SMTP_SERVER`     | `smtp.gmail.com` （Gmail の場合）                                  |
| `SMTP_PORT`       | `587` （通常はこの値で OK）                                        |
| `SMTP_USERNAME`   | あなたの Gmail アドレス (例: `your_email@gmail.com`)               |
| `SMTP_PASSWORD`   | **【最重要】** STEP 1 で取得した 16 文字のアプリパスワード         |
| `RECIPIENT_EMAIL` | 通知メールを受け取りたいメールアドレス(`SMTP_USERNAME`と同じで OK) |

すべて完了すると`Repository secrets`のセクションに 5 つの名前がリストアップされるはずです。

これでセキュリティにも配慮しつつ環境変数を設定できました。やったね。

# 動作確認

時間まで待つのが面倒だったので、`Actions`内の`Run workflow`を押して動作確認を行いました。

![動作確認](/images/send-email-github/runworkflow.png)

**無事にメールが届きました！**

![メール結果](/images/send-email-github/result.png)

# まとめ

というわけで、GitHub Actionsを使ってPythonスクリプトを定期実行する流れを学ぶことができました！

実際に動かしてみることで、YAMLファイルの書き方や環境変数の扱いなどがよく理解できました。 今回は実験としてメールを送ってみましたが、この仕組みはCI/CD（テストの自動化やデプロイ）の基礎となる部分なので、応用が効きそうです。

# 免責事項

本記事に掲載された内容によって生じた損害等の一切の責任を負いかねますので、ご了承ください。特に、パスワード等の機密情報の管理については、ご自身の責任において厳重に行ってください。また、悪用することはしないでください。

## 1. 利用規約による制限

GitHub Actions には明確な利用規約（Terms of Service）が定められており、
特に **GitHub がホストするランナー（ubuntu-latest など）** を利用する場合、
用途に関する制約が存在いたします。

### ● ホストランナーで禁止されている利用（抜粋）

GitHub 公式の規約では、以下のような用途が禁止されております。

* 暗号資産マイニング（クリプトマイニング）
* 不正なネットワークスキャンや侵入行為
  （GitHub が認めたバグバウンティ目的を除く）
* GitHub Actions を計算リソースとして再販・転売するようなサービス提供
* GitHub のホストインフラを、CDN や汎用サーバレス基盤として恒常的に利用する行為
  （例：大量トラフィックの配信、高負荷バッチ処理の常時実行 など）

これらは禁止行為とみなされた場合、ワークフローの強制停止だけでなく、
**リポジトリの無効化** や **アカウント停止** が行われる可能性が明記されております。

### ● 「リポジトリと無関係な活動」も禁止対象

加えて、GitHub がホストするランナーでは、次の制限が設けられています。

> リポジトリに関連付けられているソフトウェアプロジェクトの
> 「開発・テスト・デプロイ・運用・公開」と無関係なその他の活動を行ってはならない。

そのため、**個人的な通知ボットや、リポジトリのコードに全く関係しない自動処理**を
GitHub Actions 上で定期運用することは、規約上のリスクがございます。

一方で、以下のような用途は CI/CD として一般的であり、通常の利用範囲と考えられます。

* リポジトリで開発しているアプリケーションのデプロイ処理
* 運用に必要なメトリクス集計・軽量なバッチ処理
* リポジトリで管理しているサービスのビルド・テスト・公開フロー

本記事で紹介しているような「個人向けの定期処理」を継続的に回す場合、
GitHub Actions が **“無料のサーバレス環境の代替として利用されている”** と判断される可能性があります。

そのため、実際の運用では **学習・検証目的に留めていただくことを強く推奨** いたします。

---

## 2. cron実行（スケジュール実行）の遅延

GitHub Actions の `schedule` トリガーは、指定した時刻に必ず実行される仕組みではございません。
GitHub のリソース状況に応じて、数分〜数十分の遅延が生じることが珍しくありません。

「毎朝 8 時ちょうどに通知が必要」といった、**厳密な時間管理を伴う処理には不向き**です。

### ● `schedule` の仕様に関する補足

正確性を求める用途に影響する可能性がありますので、以下の仕様も併せてご確認ください。

* 時刻指定は **すべて UTC** で解釈されます
  （日本時間で動かす場合は 9 時間の時差調整が必要です）
* 実行対象は **デフォルトブランチの最新コミット** のみ
* `schedule` の最短間隔は **5 分**
* リソースが高負荷の際には

  * 数十秒〜数十分程度の遅延
  * キューに積まれたジョブが **実行されずにドロップ** される可能性あり
* 公開リポジトリの場合、**60 日間アクティビティが無いと自動的に無効化**

このような性質から、`schedule` は「おおよその頻度で動けば良い処理」には適していますが、
「必ず指定時刻に動くことが前提のシステム」には適しておりません。

---

## まとめ

* 記事内のコードを **学習目的で試すこと自体は問題ありません**
* ただし、GitHub Actions のホストランナーを **個人的タスクの常時運用目的** で利用することは、
  規約上のリスクがあります
* 厳密な時刻精度が必要な用途にも、GitHub Actions は適しておりません

ご指摘いただきました方に改めて御礼申し上げますとともに、
ご利用の際は上述の点をご確認いただければ幸いです。

---
*※Qiitaにてご指摘いただいたコメントにて、規約違反のリスクや代替案について詳しく教えていただきました。ありがとうございました。*
