# world-builder-agent-playbook

Tales & Tokens の Builder API が公開する MCP サーバー(`ttt-builder-mcp`)を Claude Code / Codex から利用するためのプレイブックです。world のイベント・アイテム・画像を、エージェントとの対話で構築できます。

Claude Code と Codex のプラグインとして配布しているので、MCP サーバー設定と専用スキルをまとめて導入できます。スキルは [Agent Skills](https://agentskills.io) 標準の形式(`skills/<name>/SKILL.md`)で、どちらのプラグインも同じ `skills/` を読み込みます。

## スキルカタログ

| スキル | 使う場面 |
| --- | --- |
| [ttt-world-building](skills/ttt-world-building/SKILL.md) | イベント・アイテムの作成・更新・削除・並び替え。create → update の 2 段階フロー、`update_event` のアイテム関連が「全置換」である点などの落とし穴、ページネーション規約 |
| [ttt-images](skills/ttt-images/SKILL.md) | 画像のアップロード・差し替え・削除。dataUri の形式・5MB 制限、画像には一覧/取得ツールがないため作成した id を即座に紐付ける運用 |
| [nazotoki](skills/nazotoki/SKILL.md) | 謎解き・クイズラリー・○×クイズ・スタンプラリーを企画するときに最初に読む。回答券と正解の印による仕組み、問題数・選択肢数と無料枠に収める件数計算、作問の注意点、人間がやる現地作業 |
| [nazotoki-setup](skills/nazotoki-setup/SKILL.md) | `flavor.json` と `event.json` から、world に謎解きの構成(回答券・正解の印・受付イベント・回答イベント群)を一括生成する。書き込み前の world 照合と件数見積もりを含む |
| [nazotoki-check](skills/nazotoki-check/SKILL.md) | 作成済みの謎解き world を公開前に検証する。正解の印の数、不正解イベントの配線、回答券の枚数、開催期間を確認する |

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
- [スキルカタログ](#スキルカタログ)のスキル一式

### 3. 確認する

`/mcp` で `ttt-builder` が接続済みになっていることを確認し、次のように依頼して疎通を見ます。

> この world のイベントを一覧して

### Codex で使う場合

API キーの環境変数(手順 1)は共通です。マーケットプレイスを追加し、Codex CLI の `/plugins` から `world-builder` をインストールします。

```bash
codex plugin marketplace add TalesAndTokens/world-builder-agent-playbook
```

### skills.sh(`npx skills`)で使う場合

[skills.sh](https://skills.sh) の CLI でも、このリポジトリのスキルを導入できます。スキル同士が連携する(例: `nazotoki-setup` は `nazotoki` のデータ形式とひな形を使う)ため、**個別に選ばず `--skill '*'` で一式まとめて入れてください**。

```bash
npx skills add TalesAndTokens/world-builder-agent-playbook --skill '*'
```

導入先のエージェントは対話で選びます。決まっている場合は `-a claude-code` / `-a codex` のように指定できます(ユーザー全体に入れる場合は `-g`)。

この方法で入るのはスキルだけで、**MCP サーバーの設定は含まれません**。API キーの環境変数(手順 1)を設定したうえで、MCP サーバーを別途登録してください。

- Claude Code: [プラグインを使わずに設定する](#プラグインを使わずに設定する) の `.mcp.json` を用意する
- Codex: 次のコマンドで登録する

```bash
codex mcp add ttt-builder --url https://builder-api.ttt.games/mcp --bearer-token-env-var TTT_WORLD_API_KEY
```

## リポジトリ構成

| パス | 内容 |
| --- | --- |
| `.claude-plugin/marketplace.json` | Claude Code 用マーケットプレイス定義(`tales-and-tokens`) |
| `.claude-plugin/plugin.json` | Claude Code 用プラグイン定義(`world-builder`)。MCP サーバーを宣言(スキルは既定の `skills/` から自動で読み込まれる) |
| `.agents/plugins/marketplace.json` | Codex 用マーケットプレイス定義(`tales-and-tokens`) |
| `.codex-plugin/plugin.json` | Codex 用プラグイン定義(`world-builder`)。MCP サーバーを宣言(スキルは既定の `skills/` から自動で読み込まれる) |
| `skills/ttt-world-building/` | イベント・アイテムを MCP で操作するためのスキル |
| `skills/ttt-images/` | 画像アップロード用スキル |
| `skills/nazotoki/` | 謎解きの企画用スキル(データ形式リファレンスとひな形を同梱) |
| `skills/nazotoki-setup/` | 謎解き構成の一括生成スキル |
| `skills/nazotoki-check/` | 謎解き world の公開前検証スキル |
| `docs/tools-reference.md` | 提供される全 17 ツールのリファレンス |
| `.mcp.json.example` | プラグインを使わず手動設定する場合のひな形 |

## プラグインを使わずに設定する

このリポジトリを直接 clone して使う場合や、MCP サーバーだけを別プロジェクトで使いたい場合は、`.mcp.json` を手で用意します。

```bash
cp .mcp.json.example .mcp.json
```

`TTT_WORLD_API_KEY` を設定した状態で `claude` を起動し、プロジェクトスコープの MCP サーバー(`ttt-builder`)の使用を承認してください。

この方法ではスキルは読み込まれません。スキルも使う場合はプラグインを導入するか、[skills.sh](#skillsshnpx-skillsで使う場合) で一式を導入してください。

MCP Inspector から直接叩くこともできます。

```bash
npx @modelcontextprotocol/inspector
# Transport: Streamable HTTP
# URL: https://builder-api.ttt.games/mcp
# ヘッダー: Authorization: Bearer <key>
```

## 画像アップロードのヒント

画像は Base64 の dataUri(`data:<contentType>;base64,...`)としてツール引数に渡します(詳細は `ttt-images` スキル)。数 MB の画像はそのままだと dataUri が巨大になり、エージェント経由での受け渡しが非現実的になります。アップロード前に用途に合わせて**縮小・再圧縮**しておくと確実です(例: 最大 1024px 程度、透過が不要なら JPEG 化)。dataUri 全体で最大 5MB。

## セキュリティ

- 生の API キーは環境変数(`TTT_WORLD_API_KEY`)にのみ置き、設定ファイルに直接書かないでください
- `.mcp.json`(実ファイル)は `.gitignore` 済みです
- すでに生キーを `.mcp.json` に書いてしまった場合は、キーを環境変数へ移し、`.mcp.json` を `${TTT_WORLD_API_KEY}` 形式に戻したうえで、露出したキーは失効・再発行してください
