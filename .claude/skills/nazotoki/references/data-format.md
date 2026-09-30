# nazotoki-kit データ形式 v0.1

2026-09-29 ドラフト。構想は [スキル化提案_20260929.md](スキル化提案_20260929.md)。
実データの裏付けは [茨城町編 実装記録](../../20_町/茨城町/40_WorldBuilder/実装記録.md)（world 10064）。

## §1 2つのファイルに分ける

| ファイル | 中身 | 寿命 |
|---|---|---|
| `flavor.json` | 世界観。回答券・印・記念品の名前と文言 | **使い回す**。一度書けば何回でも |
| `event.json` | 企画。正解表・解説・期間・画像 | **開催ごとに作る** |

分ける理由は、フォークダイバーの副担当者が30〜40案件を回すとき、
flavor をそのままに event だけ差し替えられるようにするため。

**どちらにも問題文と選択肢の文言は入れない。** それ自体がヒントになる。
必要なのは「正解がどれか」だけ。

## §2 flavor.json

```json
{
  "name": "plain",
  "ticket": {
    "ja": { "name": "回答券", "description": "1問に答えるたびに1枚使う。受付で受け取れる。" },
    "en": { "name": "Answer Ticket", "description": "Spend one each time you answer." },
    "image": null
  },
  "mark": {
    "ja": { "name": "正解の印", "description": "正解すると1つもらえる。" },
    "en": { "name": "Mark", "description": "You get one for each correct answer." },
    "image": null
  },
  "reward": null,
  "messages": {
    "ja": {
      "registered": "回答券を受け取りました。はじめましょう！",
      "correct": "正解です！ 正解の印を手に入れました。",
      "wrong": "不正解です。回答券が残っていれば、ほかの選択肢で再挑戦できます。",
      "cleared": "クリアです！",
      "submitLabel": "回答する",
      "backLabel": "解答ページに戻る"
    },
    "en": { "...": "同じキー" }
  }
}
```

| フィールド | 必須 | 備考 |
|---|---|---|
| `ticket` | ○ | 回答券。World Builder の `CREDIT` |
| `mark` | ○ | 正解の印。`CREDIT`（**個数を数える**ので CREDIT） |
| `reward` | — | 記念品。`COMMON_ITEM`。**`null` が既定**（→ 構想 §2.1） |
| `image` | — | **すべて任意。** 画像なしで登録でき、あとから足せる |
| `en` | — | 省略可。省略したら英語版を作らない |

`messages.correct` は**書き出しだけ**。この後ろに event.json の解説が改行で連結される。

## §3 event.json

4問×○×（無料枠ぴったり・クリアあり）の例。

```json
{
  "title": "○○町編 2026",
  "worldId": 10099,
  "flavor": "plain",

  "choiceLabels": ["○", "×"],
  "clearAt": 3,

  "period": { "start": null, "end": "2027-03-31" },
  "answerPageUrl": null,
  "deliveryId": null,

  "questions": [
    { "correct": "○", "note": { "ja": "涸沼のシジミは…", "en": "The clams of…" } },
    { "correct": "×", "note": { "ja": "…", "en": "…" } },
    { "correct": "○", "note": { "ja": "…" } },
    { "correct": "○", "note": { "ja": "…" } }
  ]
}
```

| フィールド | 必須 | 備考 |
|---|---|---|
| `worldId` | — | **安全装置。** 書き込む前に接続先の world と一致するか検証する（→ §6） |
| `flavor` | ○ | 使う flavor の名前 |
| `choiceLabels` | — | 既定は問題数で決まる。2なら `["○","×"]`、3なら `["A","B","C"]` |
| `clearAt` | — | クリア条件。**`null` ならクリアイベントを作らない**（既定） |
| `period.end` | — | ISO 日付。UNIX 秒への変換は setup がやる |
| `answerPageUrl` | — | 回答ページのURL。**サイトなし構成では `null`** |
| `deliveryId` | — | 記念品の配布設定。`reward` がある場合のみ |
| `questions[].correct` | ○ | `choiceLabels` のどれか1つ |
| `questions[].note` | — | 正解時に出す解説。**企画の中身はここ** |

**問題数は `questions` の長さで決まる。** 別のフィールドで二重に持たない。
**選択肢数は `choiceLabels` の長さで決まる。**

## §4 画像

`event.json` と同じ階層の `images/` に置く。**すべて任意。**

```
images/
├── registration.jpg     受付イベント
├── clear.jpg            クリアイベント
├── 1-1.jpg  1-2.jpg     問題1の選択肢1・2
├── 2-1.jpg  2-2.jpg
├── 3-1.jpg  3-2.jpg
└── 4-1.jpg  4-2.jpg
```

`<問題番号>-<選択肢番号>.<拡張子>`。無いものは画像なしで登録する。

## §5 World Builder への変換規則

**この対応が書けることが、形式が足りている証明。**

### アイテム

| flavor | API |
|---|---|
| `ticket` | `create_item({itemType:"CREDIT", name})` → `update_item({ja, en, iconId?, imageId?})` |
| `mark` | 同上（`CREDIT`） |
| `reward`（あれば） | `create_item({itemType:"COMMON_ITEM"})` → `update_item(...)` |

### 受付イベント（1件）

```
create_event({ name: "受付" })
update_event({
  ja: { name, description, submitPageMessage: flavor.messages.ja.registered,
        nextActionButtonLabel: flavor.messages.ja.backLabel },
  outputItems: [{ itemId: ticket, amount: questions.length }],   ← 問題数ぶん配る
  checkinLimitPerUser: 1,
  endPeriod: period.end を UNIX 秒に,
  nextActionButtonLink: answerPageUrl,
  eventImageId: images/registration.jpg
})
```

### 回答イベント（問題数 × 選択肢数）

問題 i、選択肢 j について。

```
create_event({ name: `問題${i}　答え${choiceLabels[j]}` })    ← 区切りは全角スペース
update_event({
  ja: {
    submitLabel: flavor.messages.ja.submitLabel,
    submitPageMessage:
      j が正解        → flavor.messages.ja.correct + "\n" + questions[i].note.ja
      j が正解でない  → flavor.messages.ja.wrong,
    nextActionButtonLabel: flavor.messages.ja.backLabel
  },
  inputItems:  [{ itemId: ticket, amount: 1 }],               ← 常に回答券を1枚消費
  outputItems: j が正解 ? [{ itemId: mark, amount: 1 }] : [],  ← 不正解は空
  checkinLimitPerUser: 1,
  endPeriod: period.end を UNIX 秒に,
  nextActionButtonLink: answerPageUrl,
  eventImageId: images/<i>-<j>.jpg
})
```

### クリアイベント（`clearAt` があるときだけ・1件）

```
create_event({ name: "クリア" })
update_event({
  ja: { submitPageMessage: flavor.messages.ja.cleared, ... },
  conditionItems: [{ itemId: mark, amount: clearAt }],   ← 所持条件。消費しない
  inputItems: [], outputItems: [],
  deliveryId: deliveryId,                               ← 記念品はここで配る
  checkinLimitPerUser: 1,
  endPeriod: period.end を UNIX 秒に
})
```

## §6 setup が書き込む前にやること

| # | 検査 | 落ちたら |
|---|---|---|
| 1 | **`worldId` が接続先と一致するか**（`list_items` の `worldId` で確認） | **中止。** 別の world に書き込む事故を防ぐ |
| 2 | イベント件数の計算と表示 | 無料枠を超えるなら警告して確認を取る |
| 3 | `questions[].correct` が `choiceLabels` に含まれるか | 中止 |
| 4 | `clearAt` ≦ 問題数 か。満点になっていないか | 警告 |
| 5 | 既存イベントがあるか（追記か新規か） | 追記モードに切り替え |

### イベント件数

```
1（受付） + 問題数 × 選択肢数 + （clearAt があれば 1）
```

無料枠は**イベント10件**（受付も1件に数える。**アイテムは無制限**）。

| 構成 | 件数 | |
|---|---|---|
| 4問 × ○× ＋ クリア | 10 | **無料枠ぴったり。初めての人に勧める既定** |
| 3問 × 3択 | 10 | 無料枠ぴったり（クリアは作れない） |
| 9問 × 3択 ＋ クリア | 29 | 有料。茨城町編と同じ規模 |

## §7 check が見るもの

setup と同じ計算を、**実際に作られた world の中身に対して**行う。

- 回答イベントの数 ＝ 問題数 × 選択肢数
- 受付が配る回答券の枚数 ＝ 問題数
- **印を出すイベントの数 ＝ 問題数**（1問につきちょうど1つ）
  ← **印が1種類で個数を数える方式なので、1件の配線ミスで全員クリアになる。最重要**
- 不正解イベントの `outputItems` が空か
- 全イベントに `endPeriod` が入っているか（**漏れた問題だけ会期後も答えられる**）
- クリアが `conditionItems` か（`inputItems` だと印が消える）
- `answerPageUrl` を設定した場合、受付イベントのURLがサイト側に漏れていないか

## §8 未確定

1. **`create_item` の `name` と `update_item` の `ja.name` の関係** — `create_item` にも name があるが、
   i18n は `update_item` で入れる。二重になるので、実装時に実データで確認する
2. **`deliveryId` の作り方** — 茨城町編は `119` が入っていたが、これをどう用意するかは未調査。
   `reward` を使う構成を実装するときに調べる
3. **画像アップロードの順序** — `create_image` が返す id を `update_item` / `update_event` に渡す。
   件数が多いので、失敗時の再開方法を決める必要がある
