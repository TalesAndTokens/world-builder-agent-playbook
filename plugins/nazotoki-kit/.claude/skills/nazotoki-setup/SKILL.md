---
name: nazotoki-setup
description: flavor.json と event.json から、T&T World Builder の world に謎解きの構成（回答券・正解の印・受付イベント・回答イベント群）を一括生成する。問題数×選択肢数ぶんのイベントを管理画面で手作業するのは現実的でないため、謎解き・クイズラリー・○×クイズを world に登録するときは必ずこれを使う。書き込み前の world 照合、イベント件数の見積もり、正解表の検証を含み、既存構成への問題追加にも対応する。
---

# 謎解き構成の一括生成

`flavor.json`（世界観）と `event.json`（企画）から、world にアイテムとイベントを作る。

**先に `ttt-world-building` スキルを読むこと。** `create` → `update` の2段階、
**アイテム関連（`conditionItems` / `inputItems` / `outputItems`）が全置換であること**、
画像の扱いは、すべてそちらに書かれている。ここではその上に乗る謎解き固有の手順だけを扱う。

全体像と構成の決め方は `nazotoki` スキルにある。

## 入力

| ファイル | 中身 | 寿命 |
|---|---|---|
| `flavor.json` | 回答券と印の名前・文言 | 使い回す |
| `event.json` | 正解表・解説・期間・画像 | 開催ごとに作る |

形式の詳細は `docs/data-format.md`。**どちらにも問題文と選択肢の文言は入れない。**
それ自体がヒントになるため、必要なのは「正解がどれか」だけ。

## 書き込む前に必ず確認する

world への書き込みは取り消せない。**作り始める前に、この5つを通す。**

### 1. world の照合

`event.json` に `worldId` があれば、`list_items` を1件だけ呼んで返ってきた `worldId` と比較する。
**一致しなければ中止する。**

MCP は API キーで接続先の world が決まるため、**設定を取り違えると別の企画の world に
書き込んでしまう**。しかも取り消せない。`worldId` はそれを止めるための歯止めなので、
一致しないときに「たぶん合っているだろう」で進めない。

`worldId` が `null` のときは、接続先の world 名を読み上げて、意図した world か確認を取る。

### 2. 件数の見積もりを見せる

```
イベント数 = 1 + 問題数 × 選択肢数
```

作る前に「この構成は N 件になります」と伝える。**ベーシックプラン（無料）は10件まで**なので、
超えるなら、有料プランが必要になることを説明して確認を取る。
**課金の壁に不意に当たるのが、最悪の初回体験**になる。

### 3. 正解表の妥当性

- `questions[].correct` が `choiceLabels` のどれかと一致するか
- `questions` の長さが1以上か

### 4. 既存の構成があるか

`list_events` で、同じ名前のイベント（`問題1　答え○` など）が既にないか調べる。
あれば**追加モード**（下記）に切り替える。二重に作ると、参加者から見て同じ問題が2つ並ぶ。

### 5. 画像の有無

画像はすべて任意。無ければ画像なしで作る。あとから足せる。

## 生成の手順

### アイテム（2つ）

| flavor | itemType | 役割 |
|---|---|---|
| `ticket` | `CREDIT` | 回答券 |
| `mark` | `CREDIT` | 正解の印 |

**どちらも `CREDIT`。** 「何個持っているか」で数えるものだから。

`create_item` で枠を作り、`update_item` で `ja` / `en` と画像を入れる。

### 受付イベント（1件）

```
create_event { name: "受付" }
update_event { eventId, body: {
  ja: { name, description,
        submitPageMessage: flavor.messages.ja.registered,
        nextActionButtonLabel: flavor.messages.ja.backLabel },
  outputItems: [{ itemId: <回答券>, amount: <問題数> }],
  conditionItems: [], inputItems: [],
  checkinLimitPerUser: 1,
  endPeriod: <period.end を Unix 秒に>,
  nextActionButtonLink: <answerPageUrl（null 可）>,
  eventImageId: <images/registration.* があれば>
}}
```

**配る回答券は問題数ちょうど。** 多いと総当たりされ、少ないと解けなくなる。

### 回答イベント（問題数 × 選択肢数）

問題 `i`、選択肢 `j` について。

```
create_event { name: `問題${i}　答え${choiceLabels[j]}` }     ← 区切りは全角スペース
update_event { eventId, body: {
  ja: {
    submitLabel: flavor.messages.ja.submitLabel,
    submitPageMessage:
      正解なら   flavor.messages.ja.correct + "\n" + questions[i].note.ja
      不正解なら flavor.messages.ja.wrong,
    nextActionButtonLabel: flavor.messages.ja.backLabel
  },
  inputItems:  [{ itemId: <回答券>, amount: 1 }],
  outputItems: 正解なら [{ itemId: <印>, amount: 1 }] / 不正解なら [],
  conditionItems: [],
  checkinLimitPerUser: 1,
  endPeriod: <period.end を Unix 秒に>,
  nextActionButtonLink: <answerPageUrl（null 可）>,
  eventImageId: <images/<i>-<j>.* があれば>
}}
```

**正解のイベントだけが印を出す。** 印は1種類で個数を数える方式なので、
不正解のイベントに印を付けてしまうと、**そこを踏んだ全員がクリアできてしまう**。
生成後に `nazotoki-check` で必ず確認する。

**`endPeriod` は1件ずつ入れる。** 入れ忘れた問題だけ会期後も答えられてしまう。

> デジタルの記念品を配りたいと言われたら、**印を N 個持っていることを条件にした
> イベントを1件、`ttt-world-building` の操作で作る**。`conditionItems` に印を入れる
> （`inputItems` にすると印が消えてしまう）。1件だけなので、このスキルでは自動化していない。

## 追加モード

3問で始めた企画を9問に伸ばす、という使われ方をする。**既存を作り直さない。**

1. `list_events` で既存の `問題N　答えX` を数え、どこまで作られているかを把握する
2. **足りない問題のイベントだけ**を作る
3. **受付イベントの `outputItems` を新しい問題数に更新する**（回答券の枚数が合わなくなるため）
4. 件数が無料枠を超えるなら、先に伝えて確認を取る

3を忘れると、問題は増えたのに回答券が足りず、最後まで答えられない。**追加モードで最も起きやすい事故**。

## 途中で失敗したとき

イベントは1件ずつ作る。途中で止まったら、**作り直さずに続きから**進める。

- `list_events` で「どこまで作れたか」を数える
- 画像は登録直後に紐付ける。`create_image` が返した id をその場で使う。
  画像には一覧取得の操作がないため、**id を失うと後から探せない**
- アイテムを作り直すと id が変わり、既存イベントの参照が切れる。アイテムは最初の1回だけ作る

## 終わったら

- 作ったイベントの一覧（名前とURL）を表で出す
- **受付イベントのURLは、他と分けて示し、「これは秘密。印刷物にだけ載せ、ウェブには出さない」と添える**
- `nazotoki-check` を実行する
