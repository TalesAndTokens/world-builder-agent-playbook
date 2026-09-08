---
name: ttt-images
description: ttt-builder MCP サーバーで Tales & Tokens の world に画像をアップロード・差し替え・削除するときに使う。dataUri の形式と 5MB 制限、imageType/contentType の許容値、画像には一覧・取得ツールがないため作成した id をその場で紐付ける運用、大きい画像は Base64 をコンテキストに載せず直接アップロードする進め方を含む。
---

# T&T 画像アップロード

## ツールと制約

- `create_image { imageType, contentType, dataUri }` → `{ id }`
- `update_image { imageId, imageType, contentType, dataUri }`(内容の差し替え)
- `delete_image { imageId }`(S3 上の実体も削除される)

**画像には list / get ツールがない。** `create_image` が返す id をその場で控え、同じ会話の中で対象(アイテムやイベント)に紐付けること。id を控え忘れると MCP 経由では探せなくなる。

## dataUri

`data:<contentType>;base64,<エンコード済みデータ>` 形式。dataUri 文字列全体で最大 5MB。

contentType の許容値: `image/png` / `image/jpeg` / `image/x-icon` / `image/vnd.microsoft.icon`

5MB を超える画像は、事前に縮小・再圧縮してから dataUri 化する。

## 大きい画像はコンテキストに載せない

dataUri は Base64 のため数百 KB〜数 MB になる。これを `create_image` の引数として会話に流すと、**コンテキストを大量に消費**し、長い Base64 をモデルが正確に再現できず壊れることもある。5MB 制限内でも起きるので、**Base64 をできるだけエージェントのコンテキストに載せない**進め方が安全:

1. **縮小・再圧縮**して dataUri を小さくする。用途に対して過剰な解像度は落とす(例: 最大 1024px 程度、透過が不要なら JPEG)。
2. 生成した dataUri は**ファイルに書き出す**。会話に読み込まない(Read しない)。
3. `create_image` は、その dataUri ファイルを**実行時に読み込む小さなクライアント**から MCP サーバーへ直接送り、返り値の**画像 id だけ**を受け取る。
   - 接続先 URL・認証トークンは、この環境で MCP が使っている設定(設定ファイルや環境変数など)を**実行時に参照**する。値をスクリプトや会話にベタ書きしない。
   - クライアントは MCP の Streamable HTTP(`initialize` → `tools/call`)を話せれば何でもよい。言語・ツールは各自の環境に合わせる(既存の MCP クライアントや Inspector の CLI でもよい)。
4. 受け取った id を、通常どおり小さな引数の `update_item` / `update_event` で紐付ける(`ttt-world-building` スキル参照)。

画像を十分小さく(目安: 数十 KB)できるなら、`create_image` に dataUri を直接渡すだけでよい。問題になるのは **Base64 のサイズ**であって、小さければこの回避策は不要。直接送信でも「list/get が無いので返却 id をその場で控えて紐付ける」原則は変わらない。

## imageType と紐付け先

| imageType | 紐付け先 |
| --- | --- |
| `item_image` | `update_item` の `body.imageId` |
| `item_icon_image` | `update_item` の `body.iconId` |
| `event_image` | `update_event` の `body.eventImageId` |
| `collection_image` / `header_image` / `token_image` / `favicon` / `delivery_cover_image` | world 設定など(現状の MCP ツールからは紐付け不可) |

## 推奨フロー(例: アイテム画像)

1. `create_image { imageType: "item_image", contentType: "image/png", dataUri: "data:image/png;base64,..." }` → `{ id }`
2. 続けてすぐに `update_item { itemId, body: { imageId: <id> } }` で紐付ける
3. アップロード直後はサーバー側で非同期リサイズ中(status: `resizing`)のため、配信 URL への反映には少し時間がかかることがある

## 注意

- `delete_image` は S3 の実体ごと消える。紐付け先(アイテム・イベント)が参照していないか確認してから削除し、ユーザーの明示的な指示がある場合のみ実行する
