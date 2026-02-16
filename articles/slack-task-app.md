---
title: 初心者でも簡単!! 見るだけでできる、Slackのタスク管理アプリの作り方
emoji: "🗂️"
type: tech
topics: ["slack","googlesheets", "gas", "python"]
published: true
publication_name: "hibari_inc"
---

## はじめに

日々のSlackでのやり取りの中で、

- 「これは後でやらないといけない」
- 「タスクとして管理したい」

と思いながら、そのまま流れてしまうことはないでしょうか。

本記事では、「コピペだけでできる」をモットーに、ほぼすべてのコードをコピー＆ペーストするだけで作れる仕組みを紹介します。

今回構築するのは、次のようなシンプルなタスク管理の流れです。

1. Slackでタスクを登録する  
2. Googleスプレッドシートに保存する  
3. Pythonのデスクトップアプリで管理する  

Web管理画面は使わず、**Slack + Google Sheets + Pythonデスクトップアプリ**という構成で実装します。

記事の上から順番に進めていけば、プログラムが動くように構成していますので、ぜひ一緒に作ってみてください。

---

# 1. Slackワークフローの設定

まずは、タスク入力の入口となるSlackの設定から行います。

## ワークフローの作成

Slackの左側メニューにある「その他」をクリックし、「ツール」を選択します。

「ツール」をクリックすると、「ワークフロー」が表示されます。

「ワークフローを構築」をクリックし、新規ワークフローを作成します。

![](/images/slack-task-app/1.png)

新規ワークフロー作成画面で「イベントを選択」をクリックします。

![](/images/slack-task-app/slack-workflow-create.png)

表示されるイベント一覧から「Slack内のリンクから」を選択し、「続行する」をクリックします。

![](/images/slack-task-app/2.png)

![](/images/slack-task-app/3.png)

次に「ステップを追加する」をクリックし、「情報をフォームで収集する」を選択します。

![](/images/slack-task-app/4.png)



---

## フォーム入力の設定

フォームには、以下の入力項目を追加します。

- タスク内容（短い回答 または 段落）
- タスク優先度（ドロップダウン）
  - 1（低）
  - 2
  - 3
  - 4
  - 5（高）
- タスクの期限（日時）

質問は以下のように設定しました。

![](/images/slack-task-app/6.png)

フォーム全体の設定は、次のようになります。

![](/images/slack-task-app/9.png)

設定が完了したら、Googleスプレッドシートとの連携を行います。

その後、「次へ」→「保存する」を選択してください。

![](/images/slack-task-app/10.png)

最後に、作成したワークフローを公開します。
これでSlack側の設定は完了です。

![](/images/slack-task-app/13.png)

---

## Slack側の使い方

このシステムの使い方です。

Slackのチャット入力欄の左側にある「＋」ボタンをクリックします。
「ワークフロー」を選択すると、先ほど作成したワークフローが表示されます。

![](/images/slack-task-app/15.png)

---

## Googleスプレッドシートに保存

ワークフローのアクションで「Google スプレッドシートに保存」を選択します。

保存先のスプレッドシートの列構成は、以下のようになります。



```
A: タスク内容
B: タスク優先度
C: タスクの期限はいつですか
D: 送信者
E: タイムスタンプ
F: ステータス
```

F列の「ステータス」は、後ほど使用するため手動で追加してください。

追加しなくても動作に問題はありませんが、
Pythonアプリで管理する際に使用します。


---

# 2. Googleスプレッドシートの設定

Slackから送られてきたデータを管理するため、
スプレッドシートをAPIとして利用できるようにします。

## Apps Scriptを開く

スプレッドシート上部メニューから以下を選択します。
Geminiの横にあります(2026年2月現在)。

```
拡張機能 → Apps Script
```

## スプレッドシートIDの取得

スプレッドシートのURLは以下の形式です。

```
https://docs.google.com/spreadsheets/d/XXXXXXXXXXXX/edit
```

`/d/` と `/edit` の間の文字列がスプレッドシートIDです。

## Google Apps Script の実装

### ポイント

- Webアプリとして公開する
- HTMLは使わずJSON API専用にする
- タスク取得とDONE更新を1つのAPIで扱う

### コード全文（コード.gs）

```javascript
const SPREADSHEET_ID = "XXXXXXXXXXXX";

function getSheet() {
  return SpreadsheetApp
    .openById(SPREADSHEET_ID)
    .getSheets()[0]; // シート名に依存しない
}

function doGet(e) {
  const sheet = getSheet();
  const rows = sheet.getDataRange().getValues();

  // ===== DONE処理（タイムスタンプ基準）=====
  if (e.parameter.done) {
    const targetTimestamp = String(e.parameter.done);

    // ヘッダー除外して探索
    for (let i = 1; i < rows.length; i++) {
      const rowTimestamp = String(rows[i][4]); // E列（タイムスタンプ）

      if (rowTimestamp === targetTimestamp) {
        sheet.getRange(i + 1, 6).setValue("DONE"); // F列（ステータス）
        break;
      }
    }

    return ContentService
      .createTextOutput("ok")
      .setMimeType(ContentService.MimeType.TEXT);
  }

  // ===== 一覧取得 =====
  if (rows.length < 2) {
    return ContentService
      .createTextOutput(JSON.stringify([]))
      .setMimeType(ContentService.MimeType.JSON);
  }

  const tasks = rows.slice(1)
    .filter(row => row[0]) // A列（タスク内容）がある行だけ
    .map(row => {
      return {
        "タスク内容": row[0],
        "タスク優先度": row[1],
        "タスクの期限はいつですか": row[2],
        "送信者": row[3],
        "タイムスタンプ": row[4],
        "ステータス": row[5] || ""   // ← 必ずキーを作る
      };
    });

  return ContentService
    .createTextOutput(JSON.stringify(tasks))
    .setMimeType(ContentService.MimeType.JSON);
}
```

## Webアプリとしてデプロイ

Apps Scriptの右上「デプロイ」から新しいデプロイを行います。

- 種類：ウェブアプリ
- 実行ユーザー：自分
- アクセス権：全員

デプロイ後に表示されるURLを控えておきます。
![](/images/slack-task-app/19.png)

---

# 3. Pythonデスクトップアプリの設定

最後に、タスクを表示・操作するデスクトップアプリを作成します。

## 環境準備

```bash
python -m venv venv
venv\Scripts\activate
pip install requests
```

## Pythonコード全文

```python
import requests
import tkinter as tk
from datetime import datetime

# GAS WebアプリのURLを指定してください
URL = "あなたのGAS WebアプリのURL"

BG_PARCHMENT = "#f5e6c8"
BG_PANEL     = "#eedcb3"
BG_ITEM      = "#e8d2a0"
BG_HOVER     = "#dfc48a"
GOLD         = "#8b6914"
GOLD_ACCENT  = "#b8860b"
BORDER_COLOR = "#c4a35a"
TEXT_DARK    = "#3e2c0e"
TEXT_MID     = "#6b5230"
TEXT_LIGHT   = "#9a8462"
RED_OVERDUE  = "#a03020"
RED_BTN      = "#8b3a2a"
RED_HOVER    = "#a84432"

# -------------------------
# パーサ系
# -------------------------

def parse_date(date_str):
    try:
        return datetime.strptime(date_str, "%B %dth, %Y at %I:%M %p UTC")
    except Exception:
        return datetime.max

def parse_priority(p):
    """
    "3 (優先度_中)" → 3
    """
    try:
        return int(str(p)[0])
    except Exception:
        return 0

# -------------------------
# API操作
# -------------------------

def mark_done(timestamp):
    print("DONE送信:", timestamp)
    if not timestamp:
        return
    try:
        requests.get(URL + f"?done={timestamp}", timeout=5)
    except Exception as e:
        print("通信エラー:", e)
    refresh()

# -------------------------
# UIヘルパー
# -------------------------

def hover_row(item, bg):
    item.configure(bg=bg)
    for ch in item.winfo_children():
        ch.configure(bg=bg)

# -------------------------
# メイン更新処理
# -------------------------

def refresh():
    for widget in list_frame.winfo_children():
        widget.destroy()

    try:
        tasks = requests.get(URL, timeout=10).json()
    except Exception:
        err = tk.Label(
            list_frame,
            text="  通信エラー — 再試行中...",
            bg=BG_PANEL, fg=RED_OVERDUE,
            font=("Meiryo", 10), anchor="w",
        )
        err.pack(fill="x", pady=5)
        root.after(5000, refresh)
        return

    # DONEは表示しない
    tasks = [t for t in tasks if t.get("ステータス") != "DONE"]

    # 優先度 → 期限 でソート
    tasks.sort(
        key=lambda t: (
            -parse_priority(t.get("タスク優先度")),
            parse_date(t.get("タスクの期限はいつですか", ""))
        )
    )

    if not tasks:
        empty = tk.Label(
            list_frame,
            text="  すべてのタスクが完了しました",
            bg=BG_PANEL, fg=TEXT_LIGHT,
            font=("Meiryo", 11), anchor="w",
        )
        empty.pack(fill="x", pady=20)

    for t in tasks:
        task_text = t.get("タスク内容", "不明なタスク")
        due_raw = t.get("タスクの期限はいつですか", "")
        timestamp = t.get("タイムスタンプ")

        try:
            due_dt = parse_date(due_raw)
            due_str = due_dt.strftime("%m/%d")
            is_overdue = due_dt < datetime.now()
        except Exception:
            due_str = "??/??"
            is_overdue = False

        item = tk.Frame(
            list_frame,
            bg=BG_ITEM,
            cursor="hand2",
            highlightbackground=BORDER_COLOR,
            highlightthickness=1
        )
        item.pack(fill="x", pady=(0, 3), ipady=7)

        marker = tk.Label(
            item, text="◆",
            bg=BG_ITEM, fg=GOLD_ACCENT,
            font=("Meiryo", 9),
        )
        marker.pack(side="left", padx=(12, 6))

        name_lbl = tk.Label(
            item,
            text=task_text,
            bg=BG_ITEM, fg=TEXT_DARK,
            font=("Meiryo", 11),
            anchor="w",
        )
        name_lbl.pack(side="left", fill="x", expand=True)

        due_fg = RED_OVERDUE if is_overdue else TEXT_MID
        due_lbl = tk.Label(
            item,
            text=due_str,
            bg=BG_ITEM, fg=due_fg,
            font=("Meiryo", 10),
        )
        due_lbl.pack(side="right", padx=(4, 6))

        done_btn = tk.Label(
            item,
            text="  ✓  ",
            bg=BG_ITEM, fg=TEXT_LIGHT,
            font=("Meiryo", 11, "bold"),
            cursor="hand2",
        )
        done_btn.pack(side="right", padx=(0, 4))
        done_btn.bind("<Button-1>", lambda e, ts=timestamp: mark_done(ts))
        done_btn.bind("<Enter>", lambda e, w=done_btn: w.configure(fg=GOLD))
        done_btn.bind("<Leave>", lambda e, w=done_btn: w.configure(fg=TEXT_LIGHT))

        for child in (item, marker, name_lbl, due_lbl):
            child.bind("<Enter>", lambda e, w=item: hover_row(w, BG_HOVER))
            child.bind("<Leave>", lambda e, w=item: hover_row(w, BG_ITEM))

    root.after(60000, refresh)

# -------------------------
# UI初期化
# -------------------------

root = tk.Tk()
root.title("タスク一覧")
root.geometry("420x320")
root.configure(bg=BG_PARCHMENT)
root.resizable(False, False)

header = tk.Frame(root, bg=BG_PARCHMENT)
header.pack(fill="x", padx=14, pady=(12, 4))

title_frame = tk.Frame(header, bg=BG_PARCHMENT)
title_frame.pack(side="left")

title_lbl = tk.Label(
    title_frame,
    text="タスク",
    bg=BG_PARCHMENT, fg=GOLD,
    font=("Meiryo", 15, "bold"),
)
title_lbl.pack(side="left")

sub_lbl = tk.Label(
    title_frame,
    text="  — 現在のタスク",
    bg=BG_PARCHMENT, fg=TEXT_LIGHT,
    font=("Meiryo", 9),
)
sub_lbl.pack(side="left", padx=(4, 0), pady=(4, 0))

close_btn = tk.Label(
    header,
    text="  ✕ 終了  ",
    bg=RED_BTN, fg="#f5e6c8",
    font=("Meiryo", 9, "bold"),
    cursor="hand2",
    padx=6, pady=2,
)
close_btn.pack(side="right")
close_btn.bind("<Button-1>", lambda e: root.destroy())
close_btn.bind("<Enter>", lambda e: close_btn.configure(bg=RED_HOVER))
close_btn.bind("<Leave>", lambda e: close_btn.configure(bg=RED_BTN))

sep = tk.Frame(root, bg=BORDER_COLOR, height=1)
sep.pack(fill="x", padx=14, pady=(4, 8))

list_frame = tk.Frame(root, bg=BG_PANEL)
list_frame.pack(fill="both", expand=True, padx=14, pady=(0, 14))

refresh()
root.mainloop()

```

※ `URL` には、Google Apps Script をウェブアプリとしてデプロイした際に表示されるURLを指定してください。

---

# 実行結果
実行するとディスクトップに下記のアプリが表示されます。
![](/images/slack-task-app/17.png)

---

# まとめ

Slack → Googleスプレッドシート → Python というシンプルな構成でも、
実用的なタスク管理ツールを構築できることを紹介しました。

本記事の構成のポイントは以下の通りです。

- Slackをタスク入力用のUIとして利用
- Googleスプレッドシートをデータベース代わりに活用
- Google Apps ScriptでJSON APIとして公開
- Pythonのデスクトップアプリで一覧・操作を集約

Web管理画面を作らなくても、  
普段使っているツールを組み合わせることで十分に実用的な仕組みを作れます。

この構成は、

- 個人のタスク管理
- 小規模チームでの簡易ToDo管理
- 社内ツールのプロトタイプ作成

などに特に向いています。

慣れてきたら、通知機能の追加やステータス管理の拡張など、
自分なりにカスタマイズしてみてください。


# 参考リンク
- Slack 
  https://slack.com/intl/ja-jp