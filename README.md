# Figma MCP Bridge

> **⚠️ This repository has been moved to [blueberry-team/figma-dev-mode-for-free](https://github.com/blueberry-team/figma-dev-mode-for-free)**  
> **Please use the new repository for the latest updates and support.**

> **Language / 언어 / 言語:** [한국어](./README.ko.md) | [日本語](./README.ja.md)

A bridge plugin that lets you query Figma design data directly from Claude Code and Windsurf via MCP (Model Context Protocol).
Uses the **Figma Desktop Plugin API** to extract nodes, styles, comments, and images from the **currently open file** in real time.

## Why This Exists

The Figma REST API fetches the entire file, making responses large and slow.
This plugin uses the **Figma Desktop Plugin API** to:

- Query only specific nodes selectively (**90%+ smaller** responses vs REST API)
- Strip default values automatically for clean, noise-free output
- Choose a `detail` level to receive exactly the data you need
- Reflect the **live state** of the open file (including unsaved changes)

## Architecture

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

## Setup

### 1. Build the Plugin

```bash
npm install
npm run build
```

### 2. Build the MCP Server

```bash
cd mcp-server
npm install
npm run build
```

### 3. Register the Plugin in Figma

1. Open Figma Desktop
2. **Plugins** → **Development** → **Import plugin from manifest...**
3. Select `manifest.json`

### 4. Configure MCP

#### Claude Code

The `.mcp.json` at the project root is picked up automatically.
**After cloning, update the `<path>` placeholder in `.mcp.json` to your actual project path** (e.g., `/Users/you/projects/figma-mcp-bridge`).

For global registration, add to `~/.claude/mcp.json`:

```json
{
  "mcpServers": {
    "figma-bridge": {
      "command": "node",
      "args": ["--import", "tsx/dist/esm", "<path>/mcp-server/src/index.ts"],
      "cwd": "<path>/mcp-server",
      "env": {
        "FIGMA_API_TOKEN": "<your-figma-token>"
      }
    }
  }
}
```

> `cwd` is required — without it `tsx` cannot be resolved.  
> `FIGMA_API_TOKEN` is only needed for comment fetch/reply features.

#### Windsurf

For global registration:

1. **Windsurf** → **Preferences** → **Windsurf Settings** → **Cascade** → **Open MCP Registry**
2. Click the gear icon to open `mcp_config.json`
3. Add the following configuration:

```json
{
  "mcpServers": {
    "figma-bridge": {
      "command": "node",
      "args": [
        "--import",
        "<path>/mcp-server/node_modules/tsx/dist/esm/index.mjs",
        "<path>/mcp-server/src/index.ts"
      ],
      "env": {
        "FIGMA_API_TOKEN": "<your-figma-token>"
      }
    }
  }
}
```

> Windsurf does not support the `cwd` property, so specify the full path to the `tsx` module in `args`.  
> `FIGMA_API_TOKEN` is only needed for comment fetch/reply features.

## Usage

1. Open a file in Figma Desktop
2. **Plugins** → **Development** → **MCP Bridge**
3. Confirm **green dot + "MCP server connected"** in the plugin panel
4. Call tools from AI Code Editor

> The MCP server starts automatically when AI Code Editor launches.

## MCP Tools

### `figma_get_node`

Fetch the design spec for a specific node.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `nodeId` | string | **required** | Figma node ID (the `node-id` in the URL — convert `730-16041` → `730:16041`) |
| `maxDepth` | number | `5` | Child node traversal depth |
| `detail` | `minimal` \| `standard` \| `full` | `standard` | Serialization detail level |

### `figma_get_selection`

Fetch the spec of currently selected nodes in Figma. Same parameters as `figma_get_node` (minus `nodeId`).

### `figma_get_page_nodes`

Returns the top-level node list of the current page. Returns a warning if more than 30 nodes are found — reduce scope with `detail:"minimal"` or query a specific node directly via `figma_get_node`.

### `figma_get_file_info`

Returns the file name, page list, current page, and selected node count. No parameters.

### `figma_get_comments`

Fetches comments via the Figma REST API. Requires `FIGMA_API_TOKEN`.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `fileKey` | string | auto-detected | File key from the Figma URL |
| `unresolvedOnly` | boolean | `false` | Return only unresolved comments |

### `figma_reply_comment`

Post a reply to a comment (e.g., to mark a fix as complete).

| Parameter | Type | Description |
|-----------|------|-------------|
| `fileKey` | string | Figma file key |
| `commentId` | string | ID of the comment to reply to |
| `message` | string | Reply content |

### `figma_export_node`

Export a node as an image (base64).

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `nodeId` | string | **required** | Node ID |
| `format` | `PNG` \| `SVG` \| `JPG` | `PNG` | Export format |
| `scale` | number | `2` | Export scale |

### `figma_analyze_comments`

Fetches unresolved comments and returns the design spec of each commented node in one call — letting you see *where* issues are and *what* the design looks like without extra round-trips.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `fileKey` | string | auto-detected | Figma file key |
| `detail` | `minimal` \| `standard` | `minimal` | Node info detail level |

**Return fields (per comment):**

| Field | Description |
|-------|-------------|
| `commentId` | Comment ID |
| `author` | Author |
| `message` | Comment body |
| `nodeId` | Node the comment is attached to |
| `position` | Offset coordinates within the node |
| `replies` | Replies to this comment |
| `node` | Node design spec (varies by `detail` level) |

**How it works:**
1. Fetch all unresolved comments via Figma REST API
2. Group into threads (root comment + replies)
3. Collect unique node IDs from comments
4. Fetch each node's spec via Plugin API (deduped)
5. Return merged comment + node data

> A warning is included if comments exceed 20. Use `figma_get_node(nodeId, detail:"standard")` for deeper inspection of a specific node.

### `figma_get_styles`

Returns all local design system styles (colors, text, effects). No parameters.

## Detail Levels

Control response verbosity with the `detail` parameter. See [`docs/`](./docs/) for real response examples.

### `minimal`

Node identity only. Ideal for scanning large node lists.

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

**Includes:** id, name, type, width, height, childCount  
**Use for:** Understanding page structure, finding node IDs

### `standard` (default)

Core information needed to implement UI. Default values (`visible:true`, `opacity:1`, etc.) are automatically stripped.

**Includes:** minimal + fills, strokes, cornerRadius, layout (flexbox/padding/gap), typography (font/size/lineHeight/letterSpacing/color), componentId/Name, recursive children  
**Use for:** Implementing UI components, design-to-code comparison, style extraction

### `full`

All properties including design tokens, component variants, and overrides.

**Includes:** standard + absoluteBoundingBox, boundVariables (design token bindings), textTruncation/maxLines, textStyleId, componentProperties, overrides, variantGroupProperties, clipsContent, counterAxisSpacing  
**Use for:** Design token mapping, variant analysis, design system documentation

### Auto-Downgrade

Response size is automatically controlled for large node sets:

- Page nodes > 30 → warning returned (no auto-switch; user decides)
- Siblings > 50 + depth ≥ 2 → `minimal`
- Siblings > 20 + depth ≥ 3 + `full` requested → `standard`
- depth ≥ 5 → `minimal`

## Response Optimization

Comparison with the REST API response:

| Item | REST API | This Plugin (standard) |
|------|----------|------------------------|
| Default values (visible, opacity, etc.) | All included | Auto-stripped |
| Paint properties | 4 fields (type, visible, opacity, color) | 2 fields (type, color) |
| Empty arrays (`dashPattern: []`) | Included | Removed |
| boundVariables | ID only (no variable names) | Omitted in standard, included in full |
| Query scope | Entire file | Specific nodes only |

## Workflows

### Design → Code

```
1. figma_get_node(nodeId, detail: "standard")  → Extract layout, colors, typography
2. Generate component code from the spec
3. figma_export_node(nodeId)  → Screenshot the design for visual comparison
```

### Design Token Extraction

```
1. figma_get_node(nodeId, detail: "full")  → Check token IDs in boundVariables
2. figma_get_styles()  → Extract all color/text/effect styles
3. Generate design token file (JSON / CSS variables)
```

### Design QA (Code vs Design)

```
1. figma_get_selection()  → Spec of currently selected component
2. Compare 1:1 with code style values (font-size, color, padding, gap, etc.)
3. Generate diff report
```

### Comment-Driven Fix Automation

```
1. figma_analyze_comments()  → Unresolved comments + node specs in one call
2. Use each comment's node info to understand the issue
3. figma_get_node(nodeId, detail:"standard")  → Re-query for deeper spec if needed
4. Fix the code
5. figma_reply_comment(commentId, "Fixed")  → Leave a reply
```

### Page Structure Discovery

```
1. figma_get_file_info()  → File name, page list
2. figma_get_page_nodes(detail: "minimal")  → Top-level frame list
3. figma_get_node(targetNodeId, detail: "standard")  → Detailed query for a specific frame
```

## Development

```bash
# Plugin watch mode
npm run watch

# Build MCP server
cd mcp-server && npm run build
```

Reload in Figma via **Plugins** → **Development** → **Reload plugin**.

> After modifying the MCP server (`index.ts`), run `npm run build`.  
> Then use `/mcp` in Claude Code to reconnect and pick up the new tool schema.

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| Red dot in plugin panel | Restart Claude Code (MCP server starts automatically) |
| "Plugin is not connected" | Re-run the MCP Bridge plugin in Figma |
| New parameters not showing in MCP tools | Run `cd mcp-server && npm run build`, then `/mcp` to reconnect |
| Comment fetch fails | Check `FIGMA_API_TOKEN` environment variable |

## Changing the Port

The default WebSocket port is `3055`. To change it, update `env.MCP_BRIDGE_PORT` in your MCP config and `WS_URL` in `ui.ts`, then rebuild.
