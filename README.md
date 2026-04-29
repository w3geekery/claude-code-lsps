# claude-code-lsps

A small Claude Code plugin marketplace that registers LSP servers for SCSS/CSS/Sass, Angular templates (HTML), and YAML — so the Claude Code `LSP` tool can answer symbol questions semantically instead of falling back to grep.

## What this gives you

When the `LSP` tool is enabled in Claude Code, registering this marketplace adds three language servers:

| Language(s) | Extensions | Server | Notes |
|---|---|---|---|
| SCSS / CSS / Sass | `.scss` `.sass` `.css` | [`some-sass-language-server`](https://github.com/wkillerud/some-sass) | Cross-file `$variable`, `@mixin`, `@function`, `%placeholder` resolution across `@use`/`@forward` chains. |
| Angular templates | `.html` | [`ngserver`](https://www.npmjs.com/package/@angular/language-server) | Bindings between component classes and templates: jump from `[input]` / `(output)` / `*ngIf` / `@if` to the matching `@Input()` / `@Output()` / pipe / directive. Only resolves inside an Angular workspace (needs `tsconfig.json` + locally installed `@angular/*` packages). |
| YAML | `.yaml` `.yml` | [`yaml-language-server`](https://github.com/redhat-developer/yaml-language-server) | Schema-aware navigation in CI configs (`.github/workflows/*.yml`), OpenAPI specs, anywhere `$ref` chains exist. |

## Prerequisites

1. **Claude Code** with the `LSP` tool enabled (`ENABLE_LSP_TOOL=1`).
2. **Node.js** (any recent version).
3. **The three language servers installed globally on your PATH:**

   ```bash
   npm install -g some-sass-language-server @angular/language-server yaml-language-server typescript
   ```

   (`typescript` is required by `@angular/language-server`. If you already have it globally, skip.)

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

Inside a Claude Code session, opening a `.scss` / `.html` / `.yaml` file and asking the LSP tool for `documentSymbol` / `findReferences` / `goToDefinition` should now resolve. If it returns "No LSP server available for file type" the binary isn't on PATH — check `which some-sass-language-server` etc.

## Layout

```
.
├── .claude-plugin/
│   └── marketplace.json    # registers the lsp-servers plugin and its language servers
├── lsp-servers/
│   └── .claude-plugin/
│       └── plugin.json     # plugin manifest (metadata only)
└── README.md
```

The `lspServers` block in `marketplace.json` is the canonical location for LSP registrations in Claude Code 2.1.x. (A `.lsp.json` file inside the plugin directory is **not** read by current versions, despite some docs suggesting otherwise.)

## Why a remote marketplace and not a local one

Claude Code 2.1.123 rejects plugin installs from **local** marketplaces when their `marketplace.json` contains an `lspServers` block ("This plugin uses a source type your Claude Code version does not support"). Git-sourced marketplaces work fine — hence this repo.

## License

MIT
