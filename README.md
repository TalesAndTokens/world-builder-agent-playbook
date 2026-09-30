# world-builder-agent-playbook

Tales & Tokens の Builder API が公開する MCP サーバー(`ttt-builder-mcp`)を Claude Code から利用するためのプレイブックです。world のイベント・アイテム・画像を、エージェントとの対話で構築できます。

Claude Code プラグインとして配布しているので、`/plugin` コマンドで MCP サーバー設定と専用スキルをまとめて導入できます。

## 前提: world スコープ API キー

MCP サーバーは world 単位の API キーで認証します。操作対象の world はキーから決まるため、ツール呼び出しで worldId を指定することはありません。

キーは **World Builder**(ウェブアプリケーション)から発行します。生キーは発行時に一度しか表示されないため、受け取ったらすぐに環境変数へ保存してください。

<!-- TODO: World Builder での API キー発行手順(画面への導線・操作手順・スクリーンショット)をここに追記する -->

## インストール

### 1. API キーを環境変数に設定する

MCP サーバーの認証ヘッダーは `TTT_WORLD_API_KEY` を参照します。**Claude Code を起動する前に**設定してください。

```powershell
# Windows (PowerShell): ユーザー環境変数として永続化
[Environment]::SetEnvironmentVariable('TTT_WORLD_API_KEY', '<発行したキー>', 'User')
# 反映されるのは「新しく開いたターミナル/プロセス」から。既存のウィンドウは開き直す
```

```bash
# macOS / Linux: シェルの設定ファイルに追記
echo 'export TTT_WORLD_API_KEY="<発行したキー>"' >> ~/.zshrc   # bash は ~/.bashrc
```

そのシェルだけで一時的に使う場合は `$env:TTT_WORLD_API_KEY = "<キー>"`(PowerShell)/ `export TTT_WORLD_API_KEY="<キー>"`(bash / zsh)でも構いません。

### 2. プラグインを導入する

Claude Code で次の 2 コマンドを実行します。

```
/plugin marketplace add TalesAndTokens/world-builder-agent-playbook
/plugin install world-builder@tales-and-tokens
```

インストールすると以下がまとめて有効になります。

- MCP サーバー `ttt-builder`(接続先・認証ヘッダーの設定込み)
- スキル `ttt-world-building` / `ttt-images`

### 3. 確認する

`/mcp` で `ttt-builder` が接続済みになっていることを確認し、次のように依頼して疎通を見ます。

> この world のイベントを一覧して

## リポジトリ構成

| パス | 内容 |
| --- | --- |
| `.claude-plugin/marketplace.json` | マーケットプレイス定義(`tales-and-tokens`) |
| `.claude-plugin/plugin.json` | プラグイン定義(`world-builder`)。MCP サーバーを宣言(スキルは既定の `skills/` から自動で読み込まれる) |
| `skills/ttt-world-building/` | イベント・アイテムを MCP で操作するためのスキル |
| `skills/ttt-images/` | 画像アップロード用スキル |
| `docs/tools-reference.md` | 提供される全 17 ツールのリファレンス |
| `.mcp.json.example` | プラグインを使わず手動設定する場合のひな形 |

## プラグインを使わずに設定する

このリポジトリを直接 clone して使う場合や、MCP サーバーだけを別プロジェクトで使いたい場合は、`.mcp.json` を手で用意します。

```bash
cp .mcp.json.example .mcp.json
```

`TTT_WORLD_API_KEY` を設定した状態で `claude` を起動し、プロジェクトスコープの MCP サーバー(`ttt-builder`)の使用を承認してください。

この方法ではスキルは読み込まれません。スキルも使う場合はプラグインを導入してください。

MCP Inspector から直接叩くこともできます。

```bash
npx @modelcontextprotocol/inspector
# Transport: Streamable HTTP
# URL: https://builder-api.ttt.games/mcp
# ヘッダー: Authorization: Bearer <key>
```

## スキル

- **ttt-world-building** — イベント・アイテムの作成/更新の 2 段階フロー、`update_event` のアイテム関連が「全置換」である点などの落とし穴、ページネーション規約
- **ttt-images** — 画像アップロードの形式・サイズ制限、画像には一覧/取得ツールがないため作成した id を即座に紐付ける運用

## 画像アップロードのヒント

画像は Base64 の dataUri(`data:<contentType>;base64,...`)としてツール引数に渡します(詳細は `ttt-images` スキル)。数 MB の画像はそのままだと dataUri が巨大になり、エージェント経由での受け渡しが非現実的になります。アップロード前に用途に合わせて**縮小・再圧縮**しておくと確実です(例: 最大 1024px 程度、透過が不要なら JPEG 化)。dataUri 全体で最大 5MB。

## セキュリティ

- 生の API キーは環境変数(`TTT_WORLD_API_KEY`)にのみ置き、設定ファイルに直接書かないでください
- `.mcp.json`(実ファイル)は `.gitignore` 済みです
- すでに生キーを `.mcp.json` に書いてしまった場合は、キーを環境変数へ移し、`.mcp.json` を `${TTT_WORLD_API_KEY}` 形式に戻したうえで、露出したキーは失効・再発行してください
