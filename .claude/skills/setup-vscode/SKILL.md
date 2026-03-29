---
name: setup-vscode
description: Set up Figma MCP Bridge for VS Code or Cursor. Creates MCP config with SSE endpoint and explains how to start the server.
allowed-tools: Bash, Read, Write, Edit, Glob
---

Set up Figma MCP Bridge for VS Code / Cursor (SSE mode).

## Steps

1. Check if `mcp-server/node_modules` exists. If not, run `cd mcp-server && npm install`.

2. Detect target based on user argument:
   - `--cursor` or user mentions Cursor → write to **both** `.vscode/mcp.json` **and** `~/.cursor/mcp.json`
   - Otherwise → write to `.vscode/mcp.json` only

3. Read the target config file(s) if they exist.

4. Add or update `figma-bridge` entry:

```json
{
  "mcpServers": {
    "figma-bridge": {
      "url": "http://localhost:3100/sse"
    }
  }
}
```

5. **Preserve** all other existing entries in `mcpServers`.

6. Write the merged config(s).

7. Print result and explain that SSE mode requires the server to be running manually:

```
cd mcp-server

# Linux / macOS
MCP_TRANSPORT=sse npm start

# Windows (PowerShell)
$env:MCP_TRANSPORT="sse"; npm start

# Windows (cmd)
set MCP_TRANSPORT=sse && npm start
```

8. Remind: start the server first, then open the editor. The MCP tools will be available once connected.
