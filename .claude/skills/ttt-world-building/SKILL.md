---
name: ttt-world-building
description: ttt-builder MCP サーバーで Tales & Tokens の world のイベント・アイテムを作成・更新・削除・並び替えするときに使う。create → update の 2 段階フロー、update_event のアイテム関連が全置換であること、uiIndex によるインベントリ表示順、ページネーション規約を含む。
---

# T&T world 構築(イベント・アイテム)

ttt-builder MCP のツールで world 内のイベント・アイテムを操作するときの手順と規約。操作対象の world は API キーで固定されている(worldId の指定は不要)。

## 大原則

1. **作成は 2 段階**: `create_*` は最小限の引数しか受け付けない。作成して返ってきた id に対して `update_*` の `body` で本体を設定する。
2. **deprecated フィールドを使わない**: `update_event` の `items` / `itemIds` は使わず、`conditionItems` / `inputItems` / `outputItems` を使う。
3. **アイテム関連は全置換(最重要)**: `update_event` で `conditionItems` / `inputItems` / `outputItems` の**どれか 1 つでも**渡すと、既存のアイテム関連は**種別を問わずすべて削除されてから**、渡した配列だけが再登録される。1 種類だけ変更したい場合も、先に `get_event` で現状を取得し、**3 配列すべてを毎回送る**こと。
4. `body` は部分更新(未指定フィールドは変更されない)だが、上記 3 だけが例外。更新前は `get_event` / `get_item` で現状を確認する。

## イベント

### 作成フロー

1. `create_event { name: "<日本語名>" }` → `{ id }`
2. `update_event { eventId, body }` で残りを設定

### update_event の body

| フィールド | 型 / 制約 |
| --- | --- |
| `enabled` | boolean(公開フラグ) |
| `ja` / `en` | i18n オブジェクト(下記) |
| `externalLink` / `nextActionButtonLink` | URL または null |
| `checkinLimit` / `checkinLimitPerUser` | 0 以上の整数(0 = 無制限) |
| `startPeriod` / `endPeriod` | Unix 秒(0 = 未設定) |
| `userCheckinInterval` | 0 以上(秒) |
| `showInLog` | boolean(既定 true) |
| `numberOfNftRequired` | number(既定 1) |
| `conditionItems` / `inputItems` / `outputItems` | EventItem 配列(**全置換に注意**) |
| `eventImageId` | 画像 id(ttt-images スキル参照) |

i18n(`ja` / `en` それぞれ、すべて省略可):

- `name`(≤255)/ `description`(≤400)/ `externalLinkName`(≤255)/ `submitLabel`(≤32)/ `submitPageMessage`(≤400)/ `nextActionButtonLabel`(≤32)

EventItem 要素は `{ itemId: number, amount: number }`(amount は 1〜10,000,000)。`eventItemType` フィールドは指定不要 — どの配列に入れたかで種別が決まる。

- `conditionItems`: 参加条件として所持が必要なアイテム(消費されない)
- `inputItems`: イベント実行で消費されるアイテム
- `outputItems`: イベント実行で付与されるアイテム

### 例: クラフトイベント(素材 2 種 → 完成品)

```
update_event {
  eventId: 42,
  body: {
    inputItems:  [{ itemId: 10, amount: 3 }, { itemId: 11, amount: 1 }],
    outputItems: [{ itemId: 20, amount: 1 }],
    conditionItems: []
  }
}
```

## アイテム

### 作成フロー

1. `create_item { name?, itemType? }` → `{ id }`
   - `itemType`: `EQUIPMENT` / `CREDIT` / `COMMON_ITEM`
2. `update_item { itemId, body }`

### update_item の body

| フィールド | 内容 |
| --- | --- |
| `enabled` | boolean |
| `ja` / `en` | `name`(≤255)/ `description`(≤200)/ `credit`(≤200) |
| `equipmentSlotId` | 装備スロット id(EQUIPMENT の場合) |
| `characterIds` | 装備可能キャラクター id の配列 |
| `iconId` / `imageId` | 画像 id(ttt-images スキル参照) |
| `itemCategoryId` | カテゴリ id |

### 表示順(Play App インベントリ)

表示順は `item.uiIndex`(未設定は `null`)で決まり、専用ツールで操作する。`update_item` の `body` では変更できない。

- `set_item_order { orderedItemIds, startIndex? }`: 配列に並べた順で `uiIndex` を採番(`startIndex` 既定 1)。最大 500 件
- `reset_item_order { itemIds? }`: `uiIndex` をクリア。`itemIds` を省略すると world 内の全アイテムが対象

注意点:

- **部分指定は既存の順序と混ざる**。配列に含めなかったアイテムは現在の `uiIndex` を保持するため、並べ替えるときは対象範囲の id をすべて渡す。事前に `list_items` で現在の `uiIndex` を確認する
- `uiIndex` が `null` のアイテムは常に末尾に、従来順(新しい順)で並ぶ
- **この順序が効くのは Play App のインベントリだけ**。`list_items` の返却順は常に id 降順で、`uiIndex` の影響を受けない

```
set_item_order { orderedItemIds: [658, 637, 659, 660] }
→ uiIndex 1, 2, 3, 4 の順に採番される
```

## 一覧とページネーション

- `list_events` / `list_items` は **id 降順**。`perPage` は既定 30・最大 100
- 次ページ: 直前ページの最後の要素の id を `cursor` に渡す(その id より小さいものが返る)
- 全件が必要な場合は、返却件数が perPage 未満になるまで cursor を進めてループする
- `list_items` のフィルタ: `itemIds` を渡すと他のフィルタは無視される。`equipmentSlotIds` / `itemType` / `itemCategoryIds` で絞り込み可

## 削除

`delete_event { eventId }` / `delete_item { itemId }`。取り消せないため、ユーザーの明示的な指示がある場合のみ実行する。
