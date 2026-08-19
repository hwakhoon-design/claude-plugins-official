# Claude Plugins Official — Codebase Guide

## What This Repo Is

This is the **official curated plugin marketplace** for Claude Code, maintained by Anthropic. It is a directory-as-data repo: the primary artifact is `.claude-plugin/marketplace.json`, a catalog of plugins users can install via `/plugin install <name>@claude-plugins-official` or through the in-app Discover browser.

The repo does **not** contain most plugin source code — external plugins live in their own upstream repos and are pinned here by git SHA. Internal (Anthropic-authored) plugins live under `plugins/`.

---

## Directory Structure

```
.
├── .claude-plugin/
│   └── marketplace.json        # Canonical marketplace catalog (single source of truth)
├── .github/
│   ├── bump-tracking.json      # Controls which plugins track releases vs. HEAD
│   ├── policy/
│   │   ├── prompt.md           # Claude scan prompt for security/privacy review
│   │   └── schema.json         # JSON schema for scan verdicts
│   ├── scripts/                # Automation scripts (validate-frontmatter.ts, bump discovery, etc.)
│   └── workflows/              # GitHub Actions (see CI section)
├── plugins/                    # Internal Anthropic-authored plugins (vendored source)
│   ├── example-plugin/         # Reference implementation — read this first
│   ├── agent-sdk-dev/
│   ├── claude-code-setup/
│   ├── claude-md-management/
│   ├── claude-security/
│   ├── code-modernization/
│   ├── code-review/
│   ├── code-simplifier/
│   ├── commit-commands/
│   ├── feature-dev/
│   ├── frontend-design/
│   ├── mcp-server-dev/
│   ├── plugin-dev/
│   └── ...                     # ~39 total internal plugins
└── external_plugins/           # Small vendored external plugins maintained here
    ├── asana/
    ├── discord/
    ├── firebase/
    ├── github/
    ├── telegram/
    └── ...                     # ~15 total
```

---

## Plugin Structure

Every plugin follows this layout:

```
plugin-name/
├── .claude-plugin/
│   └── plugin.json             # Required: name, description, author
├── .mcp.json                   # Optional: MCP server configuration
├── commands/                   # Optional: slash commands (*.md)
├── agents/                     # Optional: agent definitions (*.md)
├── skills/
│   └── skill-name/
│       ├── SKILL.md            # Required per skill
│       └── references/         # Optional reference docs loaded by skill
├── hooks/
│   └── hooks.json              # Optional: lifecycle hooks
└── README.md
```

---

## Marketplace Catalog (`marketplace.json`)

Each entry in the `plugins` array represents one installable plugin. Key fields:

| Field | Purpose |
|-------|---------|
| `name` | **Immutable slug** — never change after publish; users are installed under this key |
| `displayName` | Human label shown in UI (can change) |
| `description` | Install-time description; must match actual behavior (security requirement) |
| `author` | Attribution |
| `category` | One of: `development`, `database`, `security`, `productivity`, `deployment`, `monitoring`, `design`, `automation`, `learning`, `migration` |
| `source` | Where to fetch plugin files (see Source Types below) |
| `homepage` | Link to docs or repo |
| `strict` | When `false`, plugin skips the `plugin.json` manifest check (use with `skills` array) |
| `skills` | For skill-bundle plugins without a `plugin.json`; relative paths from `source.path` |
| `lspServers` | For LSP plugins; declares language server command and file extensions |

### Source Types

**Local path** (internal/external plugins in this repo):
```json
"source": "./plugins/my-plugin"
```

**External git repo root**:
```json
"source": {
  "source": "url",
  "url": "https://github.com/org/repo.git",
  "sha": "<commit-sha>"
}
```

**Git subdirectory**:
```json
"source": {
  "source": "git-subdir",
  "url": "https://github.com/org/repo.git",
  "path": "plugins/my-plugin",
  "ref": "main",
  "sha": "<commit-sha>"
}
```

Always pin external sources to a specific commit SHA. The `ref` field is for human readability; `sha` is what the loader actually uses.

### Plugin Renames

When a plugin must be renamed, add an entry to the top-level `renames` map — never change the `name` field alone:

```json
"renames": {
  "old-name": "new-name"
}
```

---

## CI / Automation

### Required Status Checks

All PRs to `main` must pass three checks:

| Check | Workflow | Purpose |
|-------|---------|---------|
| `validate` | `validate-plugins.yml` | Validates plugin structure against marketplace schema |
| `scan` | `scan-plugins.yml` | AI-powered security/privacy review of changed external entries |
| `check` | `check-mcp-urls.yml` | Verifies MCP server URLs are reachable |

### Nightly SHA Bumps (`bump-plugin-shas.yml`)

Runs daily at 07:23 UTC. For each external entry where upstream HEAD has advanced past the pinned SHA:

1. Validates at the new SHA with `claude plugin validate`
2. Opens one PR per bumped plugin on branch `bump/<slug>`
3. Dispatches the three required checks against each bump branch (since GITHUB_TOKEN PRs don't trigger `on:pull_request`)

`bump-tracking.json` controls bump strategy per plugin:
- Default: track HEAD
- `releases-only` list: track the latest release tag instead of HEAD

### Security Scan (`scan-plugins.yml`)

- Scans changed external marketplace entries using Claude against `policy/prompt.md`
- Uses a verdict cache keyed on `(plugin, sha)` — a SHA is scanned at most once per policy version
- Cache invalidates when `policy/prompt.md` changes
- Verdicts are uploaded as artifacts and posted as sticky PR comments
- `revert-failed-bumps.yml` removes failing entries from bump PRs automatically

### External PR Policy (`close-external-prs.yml`)

PRs from non-Anthropic contributors are auto-closed with a link to the plugin submission form, **except** when the PR only adds marketplace entries pointing to a repo already backing a live plugin (i.e., a partner extending their existing plugin footprint).

---

## Key Conventions

### Immutable Plugin Names

The `name` field is a permanent slug. Changing it breaks all user installs. Use `displayName` for UI labels. Only rename via the `renames` map.

### SHA Pinning

All external sources must be pinned to a specific commit SHA. Never use a branch reference as the sole anchor.

### Skill-Bundle Plugins

When an upstream repo ships `SKILL.md` files without a `plugin.json` manifest, declare the plugin with `"strict": false` and an explicit `skills` array pointing to the directories containing each `SKILL.md`. Each registered skill surfaces as `<plugin-name>:<skill-name>`.

### LSP Plugins

Internal LSP plugins (clangd, gopls, csharp-ls, etc.) use `"strict": false` and a `lspServers` block instead of `skills`. No source code in these plugins — they're thin wrappers that configure the user's system language server.

### Adding a New Internal Plugin

1. Create `plugins/<slug>/` with at minimum `.claude-plugin/plugin.json`
2. Add an entry to `.claude-plugin/marketplace.json` pointing to `"source": "./plugins/<slug>"`
3. Use `example-plugin` as the reference implementation

### Adding a New External Plugin

1. Add an entry to `.claude-plugin/marketplace.json` with a `git-subdir` or `url` source
2. Pin to a specific commit SHA
3. CI will validate and scan the plugin automatically on the PR

---

## Development Workflow

### Making Changes

Work on a branch, open a PR to `main`. All three required checks must pass before merge.

For external plugin entries, the scan check uses Claude to review the upstream code for security/privacy violations against `policy/prompt.md`. Failures are cached (so re-scanning the same SHA is instant) and auto-posted as PR comments.

### Triggering a Manual Bump

To bump a specific plugin:
```
gh workflow run bump-plugin-shas.yml -f plugin=<plugin-name>
```

To bump with a custom cap:
```
gh workflow run bump-plugin-shas.yml -f max_bumps=50
```

### Running Validate Locally

```
claude plugin validate --marketplace-path .claude-plugin/marketplace.json
```

---

## Security / Policy

Plugins are scanned against `policy/prompt.md` which checks for:

- Malicious code or privacy violations
- Broad-scope hooks that run on every session (not gated to relevant projects)
- Undisclosed outbound telemetry
- Cross-service credential exfiltration (reading one service's credentials and sending to another)
- Description mismatches (description must reflect actual behavior)

A plugin that observes more than its stated purpose justifies will fail the scan even if not actively malicious. The bar is "handles user data responsibly."

---

## Reference

- Plugin docs: https://code.claude.com/docs/en/plugins
- Marketplace docs: https://code.claude.com/docs/en/plugin-marketplaces
- Plugin submission form: https://clau.de/plugin-directory-submission
- Example plugin: `plugins/example-plugin/`
