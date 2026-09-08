# ttt-builder MCP ツールリファレンス

Tales & Tokens Builder API の `POST /mcp` が公開する全 17 ツールの一覧です(サーバー名: `ttt-builder-mcp`)。操作対象の world は API キーから決まるため、worldId を渡すツールはありません。

## イベント

| ツール | 入力 | 返り値 |
| --- | --- | --- |
| `list_events` | `cursor?`(この id より前を取得・exclusive), `perPage?`(1〜100、既定 30) | `{ events: [...] }`(id 降順) |
| `get_event` | `eventId` | イベント本体 + i18n + アイテム関連 |
| `create_event` | `name`(日本語名、1〜255 文字) | `{ id }` |
| `update_event` | `eventId`, `body`(全フィールド任意・部分更新) | `{ ok: true }` |
| `delete_event` | `eventId` | `{ ok: true }` |

`update_event` の `body` の詳細と注意点(アイテム関連の全置換挙動)は [ttt-world-building スキル](../.claude/skills/ttt-world-building/SKILL.md) を参照。

## アイテム

| ツール | 入力 | 返り値 |
| --- | --- | --- |
| `list_items` | `itemIds?`(指定時は他フィルタ無視), `equipmentSlotIds?`, `itemType?`, `itemCategoryIds?`, `cursor?`, `perPage?` | `{ items: [...] }`(id 降順) |
| `get_item` | `itemId` | アイテム本体 + i18n + 画像 + キャラクター関連 |
| `create_item` | `name?`(≤255), `itemType?`(`EQUIPMENT` / `CREDIT` / `COMMON_ITEM`) | `{ id }` |
| `update_item` | `itemId`, `body`(全フィールド任意・部分更新) | `{ ok: true }` |
| `delete_item` | `itemId` | `{ ok: true }` |
| `set_item_order` | `orderedItemIds`(1〜500 件), `startIndex?`(既定 1) | `{ ok: true, orders: [{ itemId, uiIndex }] }` |
| `reset_item_order` | `itemIds?`(省略時は world 内全アイテム) | `{ ok: true }` |

表示順(`uiIndex`)が効くのは Play App のインベントリのみで、`list_items` の返却順は常に id 降順です。部分指定の注意点は [ttt-world-building スキル](../.claude/skills/ttt-world-building/SKILL.md) を参照。

## 装備スロット

| ツール | 入力 | 返り値 |
| --- | --- | --- |
| `list_equipment_slots` | なし | `{ slots: [...] }`(UI 表示順、`id` / `name` / `nameJa` / `nameEn` / `enabled`) |

`update_item` の `equipmentSlotId` に渡す id をスロット名から引くときに使います。

## 画像

| ツール | 入力 | 返り値 |
| --- | --- | --- |
| `create_image` | `imageType`, `contentType`, `dataUri`(base64 data URI、≤5MB) | `{ id }` |
| `update_image` | `imageId`, `imageType`, `contentType`, `dataUri` | `{ ok: true }` |
| `delete_image` | `imageId`(S3 実体ごと削除) | `{ ok: true }` |
| `get_image` | `imageId` | 画像の実体(インライン画像ブロック) |

画像には list ツールがありません。`create_image` の返す id をその場で紐付けてください([ttt-images スキル](../.claude/skills/ttt-images/SKILL.md) 参照)。

`get_image` は画像の**ピクセルそのもの**を返すため、vision 対応クライアント(Claude Code / Claude Desktop 等)からのみ呼びます。非対応クライアントでは巨大な base64 テキストとしてコンテキストを圧迫するので、`list_items` / `get_item` / `list_events` / `get_event` が返す `imageUrl` / `iconUrl` / `eventImageUrl` を読むこと。インラインサイズ上限を超える画像は URL がテキストで返ります。

- `imageType`: `collection_image` / `event_image` / `header_image` / `token_image` / `item_image` / `item_icon_image` / `favicon` / `delivery_cover_image`
- `contentType`: `image/png` / `image/jpeg` / `image/x-icon` / `image/vnd.microsoft.icon`

## 共通仕様

- 認証: `Authorization: Bearer <world スコープ API キー>`
- Transport: Streamable HTTP(ステートレス、JSON レスポンス)。GET / DELETE は 405
- ツールの返り値はすべて JSON を整形したテキストコンテンツ
