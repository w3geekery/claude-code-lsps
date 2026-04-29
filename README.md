# claude-code-lsps

A small Claude Code plugin marketplace that registers LSP servers for SCSS/CSS/Sass and YAML — so the Claude Code `LSP` tool can answer symbol questions semantically instead of falling back to grep.

For Angular template intelligence, see [Angular templates: use vscode-mcp](#angular-templates-use-vscode-mcp) below — that path uses a different mechanism (an MCP bridge to your running VSCode), not the built-in `LSP` tool, because Claude Code's LSP harness can't currently drive Angular Language Server correctly (tracking [anthropics/claude-code#54814](https://github.com/anthropics/claude-code/issues/54814)).

## What this marketplace gives you

When the `LSP` tool is enabled in Claude Code, registering this marketplace adds two language servers:

| Language(s) | Extensions | Server | Notes |
|---|---|---|---|
| SCSS / CSS / Sass | `.scss` `.sass` `.css` | [`some-sass-language-server`](https://github.com/wkillerud/some-sass) | Cross-file `$variable`, `@mixin`, `@function`, `%placeholder` resolution across `@use`/`@forward` chains. |
| YAML | `.yaml` `.yml` | [`yaml-language-server`](https://github.com/redhat-developer/yaml-language-server) | Schema-aware navigation in CI configs (`.github/workflows/*.yml`), OpenAPI specs, anywhere `$ref` chains exist. |

## Prerequisites

1. **Claude Code** with the `LSP` tool enabled (`ENABLE_LSP_TOOL=1`).
2. **Node.js** (any recent version).
3. **The two language servers installed globally on your PATH:**

   ```bash
   npm install -g some-sass-language-server yaml-language-server
   ```

## Install

```bash
# Add this marketplace
claude plugin marketplace add w3geekery/claude-code-lsps

# Install the bundled plugin
claude plugin install lsp-servers@w3geekery-lsps
```

Then **restart Claude Code** so the language servers are picked up at session start.

## Verify

After restart:

```bash
claude plugin list | grep lsp-servers
```

Inside a Claude Code session, opening a `.scss` / `.yaml` file and asking the LSP tool for `documentSymbol` / `findReferences` / `goToDefinition` should now resolve. If it returns "No LSP server available for file type" the binary isn't on PATH — check `which some-sass-language-server` etc.

---

## Angular templates: use vscode-mcp

Angular Language Server (`ngserver` from `@angular/language-server`) needs the LSP client to provide proper workspace context (`workspaceFolders` in `initialize`, plus the right project files opened) to resolve template-to-component bindings. **Claude Code's built-in `LSP` tool doesn't do that yet** — see [issue #54814](https://github.com/anthropics/claude-code/issues/54814) and related #16804, #31364, #31365.

The working alternative today is **`vscode-mcp`** (https://github.com/tjx666/vscode-mcp). It's an MCP server that proxies queries to your running VSCode instance, where the official Angular extension drives ngserver correctly.

```
Claude Code  ->  vscode-mcp (MCP bridge, no LSPs of its own)
                         |
                         v
                 Your VSCode (running, workspace open)
                         |
                         v
                Angular.ng-template extension  ->  ngserver
```

vscode-mcp does NOT bundle its own Angular LSP. It delegates to whatever language servers your VSCode is currently running. So you need VSCode + the Angular extension installed and active.

### Setup

1. **Install the VSCode Bridge extension** (in VSCode):
   ```bash
   code --install-extension YuTengjing.vscode-mcp-bridge
   ```

2. **Install the official Angular Language Service extension** (in VSCode):
   ```bash
   code --install-extension Angular.ng-template
   ```

3. **Register the MCP server with Claude Code** (user scope so it works across all projects):
   ```bash
   claude mcp add vscode-mcp --scope user -- npx -y @vscode-mcp/vscode-mcp-server@latest
   ```

4. **Enable `strictTemplates`** in your project's `tsconfig.json`. Without this flag, Angular Language Service runs in a degraded mode that returns no template type info:
   ```json
   {
     "compilerOptions": { ... },
     "angularCompilerOptions": {
       "strictTemplates": true
     }
   }
   ```
   ⚠️ This may surface previously-hidden template type errors elsewhere in the codebase. Treat as a small refactor, not a one-line flip.

5. **Reload VSCode** (Cmd+Shift+P → "Developer: Reload Window") and **restart Claude Code**.

### Verify

In Claude Code, after the restart:

```
ToolSearch select:mcp__vscode-mcp__health_check
mcp__vscode-mcp__health_check { workspace_path: "<absolute path to repo>" }
```

Should return `Status: ok`. Then try:

```
ToolSearch select:mcp__vscode-mcp__get_symbol_lsp_info
mcp__vscode-mcp__get_symbol_lsp_info {
  workspace_path: "<repo path>",
  filePath: "<some component>.html",
  symbol: "someComponentField",
  codeSnippet: "{{someComponentField}}"
}
```

You should get hover info, definition location pointing at the `.ts` component file, etc.

### Useful tools exposed by vscode-mcp

- `mcp__vscode-mcp__get_symbol_lsp_info` — comprehensive symbol info (hover + definition + type + implementation). **Required for Angular template→component navigation.**
- `mcp__vscode-mcp__get_references` — find all references with surrounding code context
- `mcp__vscode-mcp__get_diagnostics` — real-time diagnostics, replaces slow `tsc --noEmit` / `eslint .`
- `mcp__vscode-mcp__rename_symbol` — workspace-wide rename
- `mcp__vscode-mcp__health_check`, `mcp__vscode-mcp__list_workspaces`, `mcp__vscode-mcp__open_files`, `mcp__vscode-mcp__execute_command`

### Known gotchas

- **macOS `EACCES` on socket bind** — if the bridge fails with `permission denied` on `~/Library/Application Support/YuTengjing.vscode-mcp/...sock`, that's the `com.apple.provenance` extended attribute blocking it. Fix:
  ```bash
  # quit VSCode first
  xattr -dr com.apple.provenance "/Applications/Visual Studio Code.app"
  xattr -dr com.apple.provenance "$HOME/Library/Application Support/YuTengjing.vscode-mcp/"
  # reopen VSCode
  ```
- **Large `get_references` response truncates at 8KB** — heavily-referenced symbols can fail with `Failed to parse response: Expected ',' or '}'`. Narrow the symbol or use `usageCodeLineRange: 0` to shrink results.
- **vscode-mcp talks to whichever workspace VSCode currently has open** — it's not a "all my projects" service. Switch VSCode to a different workspace and vscode-mcp's queries follow.

---

## Layout (this repo)

```
.
├── .claude-plugin/
│   └── marketplace.json    # registers the lsp-servers plugin and its language servers (sass, yaml)
├── lsp-servers/
│   ├── .claude-plugin/
│   │   └── plugin.json     # plugin manifest (metadata only)
│   └── .lsp.json           # canonical LSP configs (read by Claude Code's LSP loader)
└── README.md
```

Both `marketplace.json` `lspServers` and `lsp-servers/.lsp.json` are kept in sync — empirically the harness reads the latter at runtime, but the marketplace block is what the install validator inspects.

### Note on Angular

The previous version of this repo registered `ngserver` here too. It was removed in favor of the vscode-mcp path, because the harness can't drive `ngserver` correctly (template queries return empty regardless of `--tsProbeLocations`). When [#54814](https://github.com/anthropics/claude-code/issues/54814) is resolved, an Angular entry can be re-added here.

## Why a remote marketplace and not a local one

Claude Code 2.1.123 rejects plugin installs from **local** marketplaces when their `marketplace.json` contains an `lspServers` block ("This plugin uses a source type your Claude Code version does not support"). Git-sourced marketplaces work fine — hence this repo. (Tracking [#15148](https://github.com/anthropics/claude-code/issues/15148), [#53399](https://github.com/anthropics/claude-code/issues/53399).)

## License

MIT
