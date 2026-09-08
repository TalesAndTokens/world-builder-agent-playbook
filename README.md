# world-builder-agent-playbook

Tales & Tokens の Builder API が公開する MCP サーバー(`ttt-builder-mcp`)を Claude Code から利用するためのプレイブックです。world のイベント・アイテム・画像を、エージェントとの対話で構築できます。

## リポジトリ構成

| パス | 内容 |
| --- | --- |
| `.mcp.json.example` | Claude Code 用 MCP 設定のひな形 |
| `.claude/skills/ttt-world-building/` | イベント・アイテムを MCP で操作するためのスキル |
| `.claude/skills/ttt-images/` | 画像アップロード用スキル |
| `docs/tools-reference.md` | 提供される全 17 ツールのリファレンス |

## 前提: world スコープ API キー

MCP サーバーは world 単位の API キーで認証します。操作対象の world はキーから決まるため、ツール呼び出しで worldId を指定することはありません。

キーは **World Builder**(ウェブアプリケーション)から発行します。生キーは発行時に一度しか表示されないため、受け取ったらすぐに環境変数へ保存してください。

<!-- TODO: World Builder での API キー発行手順(画面への導線・操作手順・スクリーンショット)をここに追記する -->

## セットアップ(Claude Code)

1. 設定ファイルを作成

   ```bash
   cp .mcp.json.example .mcp.json
   ```

2. API キーを環境変数に設定して Claude Code を起動

   ```powershell
   # PowerShell
   $env:TTT_WORLD_API_KEY = "<発行したキー>"
   claude
   ```

   ```bash
   # bash / zsh
   export TTT_WORLD_API_KEY="<発行したキー>"
   claude
   ```

3. 起動時にプロジェクトスコープの MCP サーバー(`ttt-builder`)の使用を承認する
4. `/mcp` コマンドで接続状態を確認する

### API キーの永続化(毎回の設定を省く)

手順 2 の `$env:` / `export` は**そのシェルセッション限り**で、ターミナルを閉じると消えます。毎回設定するのが手間な場合は、OS のユーザー環境変数として永続化できます(生キーは環境変数にのみ置き、`.mcp.json` には書かないこと)。

```powershell
# Windows (PowerShell): ユーザー環境変数として永続化
[Environment]::SetEnvironmentVariable('TTT_WORLD_API_KEY', '<発行したキー>', 'User')
# 反映されるのは「新しく開いたターミナル/プロセス」から。既存のウィンドウは開き直す
```

```bash
# macOS / Linux: シェルの設定ファイルに追記
echo 'export TTT_WORLD_API_KEY="<発行したキー>"' >> ~/.zshrc   # bash は ~/.bashrc
```

以降は環境変数が設定済みなので、手動の設定なしで `claude` を起動できます。

## 動作確認

Claude Code で次のように依頼すると疎通確認できます:

> この world のイベントを一覧して

MCP Inspector でも直接確認できます:

```bash
npx @modelcontextprotocol/inspector
# Transport: Streamable HTTP
# URL: https://builder-api.ttt.games/mcp
# ヘッダー: Authorization: Bearer <key>
```

## 画像アップロードのヒント

画像は Base64 の dataUri(`data:<contentType>;base64,...`)としてツール引数に渡します(詳細は `ttt-images` スキル)。数 MB の画像はそのままだと dataUri が巨大になり、エージェント経由での受け渡しが非現実的になります。アップロード前に用途に合わせて**縮小・再圧縮**しておくと確実です(例: 最大 1024px 程度、透過が不要なら JPEG 化)。dataUri 全体で最大 5MB。

## スキル

`.claude/skills/` 配下に、MCP ツールを効率よく正しく使うためのスキルを同梱しています。このリポジトリを Claude Code で開くと自動的に読み込まれます。

- **ttt-world-building** — イベント・アイテムの作成/更新の 2 段階フロー、`update_event` のアイテム関連が「全置換」である点などの落とし穴、ページネーション規約
- **ttt-images** — 画像アップロードの形式・サイズ制限、画像には一覧/取得ツールがないため作成した id を即座に紐付ける運用

## セキュリティ

- 生の API キーを `.mcp.json` に直接書かないでください。`${TTT_WORLD_API_KEY}` の環境変数展開を使います
- `.mcp.json`(実ファイル)は `.gitignore` 済みです
- すでに生キーを `.mcp.json` に書いてしまった場合は、キーを環境変数へ移し(上記「API キーの永続化」)、`.mcp.json` を `${TTT_WORLD_API_KEY}` 形式に戻したうえで、露出したキーは失効・再発行してください
