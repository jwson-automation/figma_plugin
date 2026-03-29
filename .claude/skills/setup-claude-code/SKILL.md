---
name: setup-claude-code
description: Set up Figma MCP Bridge for Claude Code. Installs dependencies and configures ~/.claude/mcp.json with absolute paths for stdio mode.
allowed-tools: Bash, Read, Write, Edit, Glob
---

Set up Figma MCP Bridge for Claude Code (stdio mode).

## Steps

1. Check if `mcp-server/node_modules` exists. If not, run `cd mcp-server && npm install`.

2. Read the existing `~/.claude/mcp.json` (create if missing).

3. Add or update the `figma-bridge` entry using **absolute paths** based on this project root. Use forward slashes even on Windows:

```json
{
  "mcpServers": {
    "figma-bridge": {
      "command": "node",
      "args": ["--import", "tsx/dist/esm", "<PROJECT_ROOT>/mcp-server/src/index.ts"],
      "cwd": "<PROJECT_ROOT>/mcp-server"
    }
  }
}
```

4. If the user passed a Figma API token as argument, add `"env": { "FIGMA_API_TOKEN": "<token>" }`. Otherwise skip and mention it's optional (for comment features only).

5. **Preserve** all other existing entries in `mcpServers`.

6. Write the merged config back to `~/.claude/mcp.json`.

7. Print result:
   - Config path written
   - Remind to restart Claude Code
   - Remind to open Figma and run MCP Bridge plugin
