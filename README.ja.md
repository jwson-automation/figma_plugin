# Figma MCP Bridge

> **⚠️ このリポジトリは [blueberry-team/figma-dev-mode-for-free](https://github.com/blueberry-team/figma-dev-mode-for-free) に移転しました**  
> **最新のアップデートとサポートについては、新しいリポジトリをご利用ください。**

> **Language / 언어 / 言語:** [English](./README.md) | [한국어](./README.ko.md)

FigmaのデザインデータをClaude Code や WindsurfなどのAIコードエディタから直接取得できるMCP（Model Context Protocol）ブリッジプラグインです。
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

## セットアップ

### 1. プラグインのビルド

```bash
npm install
npm run build
```

### 2. MCPサーバーのビルド

```bash
cd mcp-server
npm install
npm run build
```

### 3. Figmaへのプラグイン登録

1. Figma Desktopを起動
2. **Plugins** → **Development** → **Import plugin from manifest...**
3. `manifest.json` を選択

### 4. MCP設定

#### Claude Code

プロジェクトルートの `.mcp.json` は自動的に認識されます。
**クローン後、`.mcp.json` 内の `<パス>` プレースホルダーを実際のプロジェクトパスに変更してください**（例：`/Users/you/projects/figma-mcp-bridge`）。

グローバル登録は `~/.claude/mcp.json` に追加：

```json
{
  "mcpServers": {
    "figma-bridge": {
      "command": "node",
      "args": ["--import", "tsx/dist/esm", "<パス>/mcp-server/src/index.ts"],
      "cwd": "<パス>/mcp-server",
      "env": {
        "FIGMA_API_TOKEN": "<your-figma-token>"
      }
    }
  }
}
```

> `cwd` は必須です。ないと `tsx` モジュールが見つかりません。  
> `FIGMA_API_TOKEN` はコメントの取得・返信機能にのみ必要です。

#### Windsurf

グローバル登録方法：

1. **Windsurf** → **Preferences** → **Windsurf Settings** → **Cascade** → **Open MCP Registry**
2. 歯車アイコンをクリックして `mcp_config.json` を開く
3. 以下の設定を追加：

```json
{
  "mcpServers": {
    "figma-bridge": {
      "command": "node",
      "args": [
        "--import",
        "<パス>/mcp-server/node_modules/tsx/dist/esm/index.mjs",
        "<パス>/mcp-server/src/index.ts"
      ],
      "env": {
        "FIGMA_API_TOKEN": "<your-figma-token>"
      }
    }
  }
}
```

> Windsurfは `cwd` プロパティをサポートしていないため、`tsx` モジュールのフルパスを `args` に指定します。  
> `FIGMA_API_TOKEN` はコメントの取得・返信機能にのみ必要です。

## 使い方

1. Figma Desktopでファイルを開く
2. **Plugins** → **Development** → **MCP Bridge** を起動
3. プラグインパネルに **緑のドット + "MCP サーバー接続中"** を確認
4. AIコードエディタからツールを呼び出す

> MCPサーバーはAIコードエディタ起動時に自動起動します。

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

## ポート変更

デフォルトのWebSocketポートは `3055`。変更する場合はMCP設定の `env.MCP_BRIDGE_PORT` と `ui.ts` の `WS_URL` を同じ値に設定してから再ビルドしてください。
