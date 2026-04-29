# CLAUDE.md — Setup directive for Claude Code

You are helping a developer set up the **w3geekery/claude-code-lsps** marketplace plus VSCode-MCP for use with Claude Code. By the end, the developer should have:

1. **Built-in `LSP` tool** working for TypeScript/JavaScript, SCSS/CSS/Sass, and YAML.
2. **`vscode-mcp` MCP server** working for Angular template intelligence (and richer LSP for any language VSCode handles).

Walk the developer through the steps below, **executing each step on their behalf** with the appropriate tools (`Bash`, `Edit`, etc.). Verify each step before moving on. If something fails, diagnose before continuing. Do not assume any step succeeded just because the previous one did.

---

## Prerequisites — verify before starting

Run these checks. If any are missing, tell the developer what to install before proceeding.

```bash
node --version            # any recent Node (>=18); used to run language servers
claude --version          # confirms Claude Code is installed
which code                # VSCode CLI must be on PATH for extension installs
```

If `code` is not on PATH on macOS: in VSCode, run **Cmd+Shift+P → "Shell Command: Install 'code' command in PATH"**.

---

## Part 1 — Built-in LSP tool (TypeScript, SCSS, YAML)

### 1.1 Install the language server binaries globally

```bash
npm install -g \
  typescript-language-server typescript \
  some-sass-language-server \
  yaml-language-server
```

After install, verify they're on PATH:

```bash
which typescript-language-server some-sass-language-server yaml-language-server
```

All three should print paths under the active Node install. If any return nothing, the npm global install location isn't on PATH — fix the developer's shell config before continuing.

### 1.2 Add the marketplace and install the plugin

The TypeScript LSP comes from Anthropic's official marketplace. SCSS and YAML come from this repo:

```bash
# Official Anthropic marketplace is added by default; if not, add it:
claude plugin marketplace add anthropics/claude-plugins-official

# Install the official TypeScript LSP plugin
claude plugin install typescript-lsp@claude-plugins-official

# Add this repo's marketplace
claude plugin marketplace add w3geekery/claude-code-lsps

# Install the SCSS/YAML bundle
claude plugin install lsp-servers@w3geekery-lsps
```

### 1.3 Verify

```bash
claude plugin list | grep -iE "lsp|servers"
```

Should show both `typescript-lsp@claude-plugins-official` and `lsp-servers@w3geekery-lsps` as enabled.

The `LSP` tool itself is a deferred tool — once per Claude Code session it's loaded with:

```
ToolSearch with query "select:LSP" and max_results 1
```

After that, `LSP` operations (`goToDefinition`, `findReferences`, `hover`, `documentSymbol`, etc.) are callable for `.ts`, `.tsx`, `.js`, `.jsx`, `.scss`, `.sass`, `.css`, `.yaml`, `.yml`.

---

## Part 2 — vscode-mcp (Angular templates + richer LSP for everything)

Why this exists: Claude Code's built-in `LSP` tool can't drive Angular Language Server (`ngserver`) correctly — the harness doesn't open the workspace context ngserver needs (tracking [#54814](https://github.com/anthropics/claude-code/issues/54814)). The workaround is to route through your already-running VSCode, which DOES drive ngserver properly.

vscode-mcp also exposes additional capabilities that the built-in `LSP` tool doesn't have:

- **`get_diagnostics`** — instant LSP diagnostics for any file. Replaces slow `tsc --noEmit` / `eslint .` runs.
- **`rename_symbol`** — workspace-wide rename with import updates.
- **`get_references`** — workspace references with surrounding code context.
- **`get_symbol_lsp_info`** — hover, definition, type definition, signature help, implementation in one call.
- **`execute_command`** — run arbitrary VSCode commands (format, code actions, etc.). Powerful; security warning applies.
- **`open_files`** — open files in VSCode, useful to prime an LSP's "open" set.

These work for **any language VSCode is currently handling**, not just Angular.

### 2.1 Install the VSCode bridge extension

```bash
code --install-extension YuTengjing.vscode-mcp-bridge
```

### 2.2 Install the official Angular Language Service extension (only if doing Angular work)

```bash
code --install-extension Angular.ng-template
```

(Skip this if the developer doesn't work in Angular projects. vscode-mcp is still useful without it for `get_diagnostics`, `rename_symbol`, etc.)

### 2.3 Register vscode-mcp with Claude Code (user scope = available in all projects)

```bash
claude mcp add vscode-mcp --scope user -- npx -y @vscode-mcp/vscode-mcp-server@latest
```

### 2.4 Enable `strictTemplates` in Angular projects

For Angular template intelligence to actually return symbols, the project's `tsconfig.json` must have `strictTemplates: true` under `angularCompilerOptions`. Without it, Angular Language Service runs in a degraded mode that returns no template type info — vscode-mcp will return empty results.

Edit the project's `tsconfig.json`:

```json
{
  "compileOnSave": false,
  "compilerOptions": { ... existing settings ... },
  "angularCompilerOptions": {
    "strictTemplates": true
  }
}
```

⚠️ Warn the developer before doing this: enabling `strictTemplates` may surface previously-hidden template type errors elsewhere in the codebase. Treat it as a small refactor, not a one-line flip. If the developer wants to evaluate the impact first, run `npm run buildCheck` (or equivalent project build command) after the edit and report the new errors before pushing.

### 2.5 Reload VSCode and restart Claude Code

- VSCode: **Cmd+Shift+P → "Developer: Reload Window"**
- Claude Code: exit and re-launch (or `claude --resume <session>`)

### 2.6 Verify

```bash
claude mcp list | grep vscode-mcp
```

Should show `✓ Connected`.

Inside Claude Code, load the health check tool and run it:

```
ToolSearch select:mcp__vscode-mcp__health_check
mcp__vscode-mcp__health_check { workspace_path: "<absolute path to the developer's repo>" }
```

Expected: `Status: ok`, version match, etc.

If it fails with `EACCES: permission denied` on a socket bind in `~/Library/Application Support/YuTengjing.vscode-mcp/...sock` (macOS only), it's the `com.apple.provenance` xattr blocking the bind. Fix:

```bash
# IMPORTANT: quit VSCode entirely first (Cmd+Q, not just close window)
xattr -dr com.apple.provenance "/Applications/Visual Studio Code.app"
xattr -dr com.apple.provenance "$HOME/Library/Application Support/YuTengjing.vscode-mcp/"
# reopen VSCode
```

For Angular template verification, ask the developer to identify a component in their workspace, then in Claude Code:

```
ToolSearch select:mcp__vscode-mcp__get_symbol_lsp_info
mcp__vscode-mcp__get_symbol_lsp_info {
  workspace_path: "<repo path>",
  filePath: "<relative path to a *.component.html>",
  symbol: "<some component property used in the template>",
  codeSnippet: "<a unique snippet around the symbol>"
}
```

Expected: hover info showing the symbol's TypeScript type, definition pointing into the corresponding `.component.ts` file. If empty results: confirm `strictTemplates: true` and that the `.ts` file is open or recently touched in VSCode.

---

## Routing — when to use which tool

| File extension | Use |
|---|---|
| `.ts` `.tsx` `.js` `.jsx` (quick targeted lookup) | Built-in `LSP` tool |
| `.ts` `.tsx` `.js` `.jsx` (workspace-wide ops, diagnostics, rename) | `mcp__vscode-mcp__*` |
| `.scss` `.sass` `.css` | Built-in `LSP` tool |
| `.yaml` `.yml` | Built-in `LSP` tool |
| `.html` (Angular component templates) | `mcp__vscode-mcp__get_symbol_lsp_info` |
| Diagnostics / "did my edit break anything" | `mcp__vscode-mcp__get_diagnostics` |
| Workspace-wide rename | `mcp__vscode-mcp__rename_symbol` |

For non-symbol queries — string literals, error messages, log lines, i18n keys, HTML class names treated as text — fall back to `grep`. LSP/MCP doesn't help with text.

### Why both TypeScript routes are useful

The built-in `LSP` tool's TypeScript plugin (`typescript-lsp@claude-plugins-official`) and vscode-mcp's TS handling both wrap the same underlying engine — TypeScript's `tsserver`. The differences are in workspace context and latency:

| Aspect | Built-in `LSP` (typescript-lsp plugin) | vscode-mcp |
|---|---|---|
| Engine | `tsserver` via `typescript-language-server` (LSP wrapper) | `tsserver` directly via VSCode |
| Workspace context | Limited by Claude Code harness (no cross-project graph init in some cases) | Full — VSCode loads the workspace, builds the project graph |
| Cold-start cost | Spawns a new tsserver per Claude session | Reuses VSCode's already-warm tsserver |
| Diagnostics speed | Has to re-analyze | Instant (project already loaded) |

**When each wins:**

- **Quick targeted lookup** in the file you're editing (`hover`, `goToDefinition`, `documentSymbol`): built-in `LSP` is fine and slightly lower latency.
- **Workspace-wide operations** — find all references across projects, rename a symbol used in many files, real-time diagnostics, anything that needs the full project graph: `vscode-mcp` wins. Especially in multi-project Angular CLI workspaces where libs are referenced by multiple apps.
- **Diagnostics specifically** (`mcp__vscode-mcp__get_diagnostics`): vscode-mcp is dramatically better. Replaces 60-second `tsc --noEmit` runs with sub-second answers because the analysis already exists in VSCode.

Don't drop either. They're complementary — fast-path versus workspace-aware path.

### Why SCSS uses the standalone route only (not vscode-mcp)

For SCSS, the comparison is different from TypeScript:

| Aspect | Built-in `LSP` (some-sass-language-server) | vscode-mcp (VSCode's bundled CSS server) |
|---|---|---|
| Engine | `some-sass-language-server` (Wkillerud) | VSCode's bundled `vscode-css-language-server` |
| Cross-file `@use`/`@forward` resolution | Full | Basic / limited |
| `$variable`, `@mixin`, `@function`, `%placeholder` symbol graph | Full | Limited |
| Single-file completion / hover | Yes | Yes |

The standalone server (`some-sass-language-server`) is more capable than the SCSS support VSCode bundles by default. So for SCSS:

- **Always use the built-in `LSP` tool**.
- vscode-mcp has no advantage for SCSS unless the developer separately installs the [Some Sass VSCode extension](https://marketplace.visualstudio.com/items?itemName=SomewhatStationery.some-sass), which would just duplicate the standalone route.

The dual-route argument that applies to TypeScript does **not** carry over to SCSS.

---

## Known gotchas

- **vscode-mcp follows whichever workspace VSCode currently has open.** Switch VSCode to a different project and vscode-mcp's queries follow.
- **Large `get_references` results truncate at 8KB** — fail with `Failed to parse response: Expected ',' or '}'`. Narrow the symbol or set `usageCodeLineRange: 0`.
- **macOS `EACCES` on socket bind** — see Part 2.6 above.
- **`Angular templates` returning empty even with strictTemplates on** — VSCode may need a few seconds to reindex after the tsconfig change. Reload window if it persists.

---

## What's NOT covered here

- Other LSP plugins (Python, Go, Rust, etc.). See the [Anthropic plugin marketplace](https://github.com/anthropics/claude-plugins-official) and the [Piebald-AI/claude-code-lsps](https://github.com/Piebald-AI/claude-code-lsps) marketplace for those.
- Configuring per-project `.lsp.json` in this repo. The current marketplace is opinionated (SCSS + YAML); Angular handled via vscode-mcp instead.

---

## Reference

- This marketplace: https://github.com/w3geekery/claude-code-lsps
- vscode-mcp: https://github.com/tjx666/vscode-mcp
- Angular Language Service extension: https://marketplace.visualstudio.com/items?itemName=Angular.ng-template
- Tracking the Claude Code LSP harness limitations: [#54814](https://github.com/anthropics/claude-code/issues/54814), [#16804](https://github.com/anthropics/claude-code/issues/16804), [#15148](https://github.com/anthropics/claude-code/issues/15148)
