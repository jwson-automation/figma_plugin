# Figma MCP Bridge

> **Language / 언어 / 言語:** [English](./README.md) | [한국어](./README.ko.md)

FigmaのデザインデータをClaude Codeから直接取得できるMCP（Model Context Protocol）ブリッジプラグインです。
**Figma Desktop Plugin API** を使用して、**現在開いているファイル**のノード・スタイル・コメント・画像をリアルタイムで抽出します。

## なぜ必要か

Figma REST APIはファイル全体を取得するため、レスポンスが大きく低速です。
このプラグインは **Figma Desktop Plugin API** を活用することで：

- 特定のノードのみを選択的に取得（REST API比 **レスポンスサイズ90%以上削減**）
- デフォルト値を自動除去し、ノイズのないクリーンな出力
- `detail` レベルの指定で必要な分だけデータを受信
- 保存前の変更を含む**現在のファイルのリアルタイム状態**を反映

## アーキテクチャ

```
Claude Code ──stdio──▶ MCP Server (Node.js, port 3055)
                            │
                        WebSocket
                            │
                       Figma Plugin UI (ui.ts)
                            │
                        postMessage
                            │
                       Figma Main Thread (code.ts) ──▶ Figma Plugin API
```

## クイックスタート

```bash
# 1. インストール & ビルド
npm install && npm run build
cd mcp-server && npm install && cd ..

# 2. Figmaにプラグインを登録
#    Plugins → Development → Import plugin from manifest... → manifest.json を選択

# 3. エディターの自動設定（Claude Code内でスラッシュコマンドとして実行）
/setup-claude-code                 # Claude Code (stdio、推奨)
/setup-vscode                      # VS Code (.vscode/mcp.json、SSE)
/setup-vscode --cursor             # Cursor (~/.cursor/mcp.json、SSE)
```

### 3. エディターの設定

**方法A — スキルを使用（自動、Claude Code専用）：**

このリポジトリには `.claude/skills/` にセットアップスキルが含まれています。Claude Code内でスラッシュコマンドとして実行：

```
/setup-claude-code                 # Claude Code (stdio、推奨)
/setup-vscode                      # VS Code (.vscode/mcp.json、SSE)
/setup-vscode --cursor             # Cursor (~/.cursor/mcp.json、SSE)
```

**方法B — 手動設定：**

| エディター | 方法 |
|------------|------|
| **Claude Code** | プロジェクトルートの `.mcp.json` が自動認識されます。このプロジェクトをClaude Codeで開いて再起動するだけで使用可能。他のプロジェクトからグローバルに使用するには[ステップ4](#4-claude-code接続stdioモード--推奨)を参照。 |
| **VS Code / Cursor** | SSEモードでサーバーを起動し、エディターで `http://localhost:3100/sse` に接続。[ステップ4-1](#4-1-sseモード他のmcpクライアント用)を参照。 |

## セットアップ

### 1. プラグインのビルド

```bash
npm install
npm run build
```

### 2. MCPサーバーの依存関係インストール

```bash
cd mcp-server
npm install
```

### 3. Figmaへのプラグイン登録

1. Figma Desktopを起動
2. **Plugins** → **Development** → **Import plugin from manifest...**
3. `manifest.json` を選択

### 4. Claude Code接続（stdioモード — 推奨）

> **仕組み:** Claude Codeは起動中のサーバーに接続するのではありません。Claude Codeが**MCPサーバープロセスを自ら起動**し、stdin/stdout（stdio）で通信します。サーバーを手動で起動する必要はありません — 設定を追加してClaude Codeを再起動するだけです。

**方法A — プロジェクトレベル（クローン後すぐ使用可能）：**

このリポジトリには、プロジェクトルートに相対パスの `.mcp.json` が含まれています。Claude Codeでこのプロジェクトディレクトリを開くと、MCPサーバーが自動認識されます。**追加設定は不要です。**

```
your-project/
├── .mcp.json          ← Claude Codeが自動認識
├── mcp-server/
│   └── src/index.ts
```

事前に `cd mcp-server && npm install` を実行しておいてください。

**方法B — グローバル登録（他のプロジェクトからも使用）：**

`~/.claude/mcp.json` に**絶対パス**で追加：

```json
{
  "mcpServers": {
    "figma-bridge": {
      "command": "node",
      "args": ["--import", "tsx/dist/esm", "<絶対パス>/mcp-server/src/index.ts"],
      "cwd": "<絶対パス>/mcp-server",
      "env": {
        "FIGMA_API_TOKEN": "<your-figma-token>"
      }
    }
  }
}
```

`<絶対パス>` を実際のパスに置き換えてください（例：`/Users/you/figma-mcp-bridge` または `C:/Users/you/figma-mcp-bridge`）。

> `cwd` は必須です。ないと `tsx` モジュールが見つかりません。
> `FIGMA_API_TOKEN` はコメントの取得・返信機能にのみ必要です。

### 4-1. SSEモード（他のMCPクライアント用）

一部のMCPクライアント（Cursor、特定のIDE拡張機能など）はstdioをサポートせず、HTTPサーバーへの接続が必要です。その場合は**SSE（Server-Sent Events）モード**を使用してください。

> **重要な違い:** stdioモードではClaude Codeがサーバーを自動起動します。SSEモードでは**サーバーを手動で起動**してからクライアントを接続する必要があります。

**Step 1 — サーバーを起動：**

```bash
cd mcp-server

# Linux / macOS
MCP_TRANSPORT=sse npm start

# Windows (PowerShell)
$env:MCP_TRANSPORT="sse"; npm start

# Windows (cmd)
set MCP_TRANSPORT=sse && npm start
```

サーバーは `http://localhost:3100/sse` で待機します。ポート変更は `MCP_SSE_PORT` 環境変数。

**Step 2 — MCPクライアントを設定：**

```json
{
  "mcpServers": {
    "figma-bridge": {
      "url": "http://localhost:3100/sse"
    }
  }
}
```

> Figmaプラグイン用WebSocketブリッジ（ポート3055）は両モードで動作します。

## 使い方

1. Figma Desktopでファイルを開く
2. **Plugins** → **Development** → **MCP Bridge** を起動
3. プラグインパネルに **緑のドット + "MCPサーバー接続中"** を確認
4. Claude Code（またはMCPクライアント）からツールを呼び出す

> stdioモード：Claude Code起動時にMCPサーバーが自動起動します。
> SSEモード：サーバーが起動済みであることを確認してください。

## MCPツール一覧

### `figma_get_node`

特定ノードのデザインスペックを取得します。

| パラメータ | 型 | デフォルト | 説明 |
|------------|-----|-----------|------|
| `nodeId` | string | **必須** | FigmaノードID（URLの`node-id`の値。`730-16041` → `730:16041`に変換） |
| `maxDepth` | number | `5` | 子ノード探索の深さ |
| `detail` | `minimal` \| `standard` \| `full` | `standard` | シリアライズの詳細レベル |

### `figma_get_selection`

現在Figmaで選択中のノードのスペックを取得します。パラメータは `figma_get_node` と同様（`nodeId` を除く）。

### `figma_get_page_nodes`

現在のページのトップレベルノード一覧を返します。30件を超える場合は警告メッセージを返すため、`detail:"minimal"` に変更するか、`figma_get_node` で特定ノードを直接取得して範囲を絞ってください。

### `figma_get_file_info`

ファイル名・ページ一覧・現在のページ・選択ノード数を返します。パラメータなし。

### `figma_get_comments`

Figma REST APIを通じてコメントを取得します。`FIGMA_API_TOKEN` 環境変数が必要です。

| パラメータ | 型 | デフォルト | 説明 |
|------------|-----|-----------|------|
| `fileKey` | string | 自動検出 | FigmaのURLに含まれるファイルキー |
| `unresolvedOnly` | boolean | `false` | 未解決コメントのみ返す |

### `figma_reply_comment`

コメントに返信します（修正完了の記録などに使用）。

| パラメータ | 型 | 説明 |
|------------|-----|------|
| `fileKey` | string | Figmaファイルキー |
| `commentId` | string | 返信対象のコメントID |
| `message` | string | 返信内容 |

### `figma_export_node`

ノードを画像（base64）としてエクスポートします。

| パラメータ | 型 | デフォルト | 説明 |
|------------|-----|-----------|------|
| `nodeId` | string | **必須** | ノードID |
| `format` | `PNG` \| `SVG` \| `JPG` | `PNG` | エクスポート形式 |
| `scale` | number | `2` | スケール倍率 |

### `figma_analyze_comments`

未解決コメントを取得し、各コメントが位置するノードのデザインスペックを1回の呼び出しで同時に返します。「どこに・どんな問題があるか」を一括把握できます。

| パラメータ | 型 | デフォルト | 説明 |
|------------|-----|-----------|------|
| `fileKey` | string | 自動検出 | Figmaファイルキー |
| `detail` | `minimal` \| `standard` | `minimal` | ノード情報の詳細レベル |

**返却値（各コメント）：**

| フィールド | 説明 |
|------------|------|
| `commentId` | コメントID |
| `author` | 投稿者 |
| `message` | コメント内容 |
| `nodeId` | コメントが位置するノードID |
| `position` | ノード内のオフセット座標 |
| `replies` | このコメントへの返信一覧 |
| `node` | ノードのデザインスペック（detailレベルに応じた内容） |

**動作フロー：**
1. Figma REST APIで未解決コメントをすべて取得
2. スレッド単位でグループ化（元コメント + 返信）
3. コメントが位置する一意のノードIDを収集
4. Plugin APIで各ノードのスペックを取得（重複ノードは1回のみ）
5. コメント + ノード情報をマージして返却

> コメントが20件を超える場合は警告が含まれます。特定ノードを詳細確認するには `figma_get_node(nodeId, detail:"standard")` で再取得してください。

### `figma_get_styles`

ローカルのデザインシステムスタイル（色・テキスト・エフェクト）を返します。パラメータなし。

## Detailレベル

`detail` パラメータでレスポンスの詳細度を調整します。実際のレスポンス例は [`docs/`](./docs/) を参照。

### `minimal`

ノードの識別情報のみを返します。大量ノードのリスト把握に最適。

```json
{
  "id": "673:2483",
  "name": "type=job, context=featured",
  "type": "COMPONENT",
  "width": 320,
  "height": 121,
  "childCount": 2
}
```

**含む内容：** id, name, type, width, height, childCount  
**用途：** ページ構造の把握、特定ノードIDの探索

### `standard`（デフォルト）

UI実装に必要なコア情報。デフォルト値（`visible:true`、`opacity:1` など）は自動除去されます。

**含む内容：** minimal + fills, strokes, cornerRadius, レイアウト（flexbox/padding/gap）, タイポグラフィ（font/size/lineHeight/letterSpacing/color）, componentId/Name, 子ノード再帰  
**用途：** UIコンポーネント実装、デザイン→コード比較、スタイル抽出

### `full`

デザイントークン・コンポーネントvariant・オーバーライドなど全プロパティ。

**含む内容：** standard + absoluteBoundingBox, boundVariables（デザイントークンバインディング）, textTruncation/maxLines, textStyleId, componentProperties, overrides, variantGroupProperties, clipsContent, counterAxisSpacing  
**用途：** デザイントークンマッピング、variant分析、デザインシステムドキュメント化

### 自動ダウングレード

大量ノード時のレスポンスサイズ制御：

- ページノード30件超 → 警告メッセージを返す（自動切替なし、ユーザーが判断）
- 深い子ノードで兄弟ノード50件超 + depth ≥ 2 → `minimal`
- 深い子ノードで兄弟ノード20件超 + depth ≥ 3 + `full` 指定 → `standard`
- depth ≥ 5 → `minimal`

## レスポンス最適化

REST APIレスポンスとの比較：

| 項目 | REST API | このプラグイン（standard） |
|------|----------|---------------------------|
| デフォルト値（visible、opacity等） | すべて含む | 自動除去 |
| Paintプロパティ | 4フィールド（type, visible, opacity, color） | 2フィールド（type, color） |
| 空配列（`dashPattern: []`） | 含む | 除去 |
| boundVariables | IDのみ（変数名なし） | standardでは省略、fullで含む |
| 取得スコープ | ファイル全体 | 特定ノードのみ |

## 活用ワークフロー

### デザイン → コード自動実装

```
1. figma_get_node(nodeId, detail: "standard")  → レイアウト・色・タイポ抽出
2. スペックからコンポーネントコード生成
3. figma_export_node(nodeId)  → デザインスクリーンショットで結果比較
```

### デザイントークン抽出

```
1. figma_get_node(nodeId, detail: "full")  → boundVariablesでトークンID確認
2. figma_get_styles()  → 色・テキスト・エフェクトスタイルを全量抽出
3. デザイントークンファイル（JSON/CSS Variables）生成
```

### デザインQA（コード vs デザイン比較）

```
1. figma_get_selection()  → 現在選択中のコンポーネントスペック
2. コードのスタイル値と1:1比較（font-size, color, padding, gap等）
3. 差分レポート生成
```

### コメントベースの修正自動化

```
1. figma_analyze_comments()  → 未解決コメント + 各位置ノードスペックを一括取得
2. 各コメントのnode情報で問題を把握
3. figma_get_node(nodeId, detail:"standard")  → 詳細スペックが必要な場合は再取得
4. コードを修正
5. figma_reply_comment(commentId, "修正完了")  → 返信を残す
```

### ページ構造の把握

```
1. figma_get_file_info()  → ファイル名、ページ一覧
2. figma_get_page_nodes(detail: "minimal")  → トップレベルフレーム一覧
3. figma_get_node(targetNodeId, detail: "standard")  → 特定フレームの詳細取得
```

## 開発

```bash
# プラグインのwatchモード
npm run watch

# MCPサーバービルド
cd mcp-server && npm run build
```

Figmaでの反映は **Plugins** → **Development** → **Reload plugin**。

> MCPサーバーのコード（`index.ts`）を修正した後は必ず `npm run build` を実行してください。  
> その後、Claude Codeで `/mcp` コマンドを使って再接続すると、新しいツールスキーマが反映されます。

## トラブルシューティング

| 症状 | 対処法 |
|------|--------|
| プラグインパネルに赤いドット | Claude Codeを再起動（MCPサーバーが自動起動） |
| 「プラグインが接続されていません」 | FigmaでMCP Bridgeプラグインを再起動 |
| MCPツールに新しいパラメータが反映されない | `cd mcp-server && npm run build` の後、`/mcp` で再接続 |
| コメント取得が失敗する | `FIGMA_API_TOKEN` 環境変数を確認 |
| 「MCPサーバーに接続できない」 | サーバーを手動で起動しないでください。`mcp.json`に設定を追加してClaude Codeを再起動すれば、stdioで自動起動されます。stdioをサポートしないクライアントはSSEモードを使用してください（4-1参照）。 |

## ポート変更

デフォルトのWebSocketポートは `3055`。変更する場合はMCP設定の `env.MCP_BRIDGE_PORT` と `ui.ts` の `WS_URL` を同じ値に設定してから再ビルドしてください。
