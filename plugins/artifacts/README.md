# `/artifacts` — a Claude Code skill for shareable outputs

Stop fighting terminal-wrap line breaks when copying Claude Code output into Slack, GitHub, or Notion.

`/artifacts` saves Claude-produced content (docs, PR descriptions, review comments, scripts, configs) to a **private Notion database**, with a **local clipboard cache** for instant `pbcopy`. The raw markdown lands intact in your clipboard — no reflowing, no hand-editing.

```
me:   write a PR description for the auth refactor
cc:   <writes it inline>
      Saved as artifact `auth-refactor-pr` → Notion · /artifacts to copy.

me:   /artifacts
cc:   Copied auth-refactor-pr.md (1.4 KB) to clipboard.

me:   <paste into GitHub — markdown renders perfectly>
```

## What you get

- **One Notion database** (`Claude Code Artifacts`) — your private source of truth. Rows have Type · Repo · Project path · Extension · Status · Summary properties for sorting/filtering.
- **A global local cache** at `~/.claude/artifacts/` — purely so `pbcopy` is instant. Always overwritten to match Notion.
- **A small bash helper** for read-side ops — `list`, `copy`, `share` (link), `rm`, `clear`, `url`. No inference needed for any of those.
- **Repo-scoped picker** by default — when you run `/artifacts` from inside a repo, only artifacts saved from that repo appear. `--all` to disable.
- **Auto-save** — Claude proactively saves shareable outputs (configurable rules in `SKILL.md`).

## Requirements

- **macOS** (uses `pbcopy`; cross-platform PRs welcome)
- **Claude Code** with the **Notion MCP** connector authenticated (`/mcp` → "claude.ai Notion")
- **`jq`** (`brew install jq`)
- **`git`** (used to detect current repo for scoping)

## Install

Inside Claude Code:

```
# 1. Add the marketplace (one-time — shared across all of Dan's plugins)
/plugin marketplace add danbovey/claude-code-marketplace

# 2. Install the plugin
/plugin install artifacts@danbovey

# 3. Authenticate the Notion MCP (one-time, if you haven't already)
/mcp                       # then select "claude.ai Notion"

# 4. Run setup
/artifacts init
```

`/artifacts init` checks deps, scaffolds the registry, searches your **private** Notion for an existing `Claude Code Artifacts` database (adopting it if found), and only creates a new one if nothing private exists. Non-destructive — never deletes or silently mutates an adopted database. Pass `--reconfigure` to re-detect later.

### (Recommended) Skip the permission prompt

Add an allow-rule so the helper doesn't prompt on every call. In `~/.claude/settings.json`:

```json
{
  "permissions": {
    "allow": [
      "Bash(artifacts:*)"
    ]
  }
}
```

(The bare command name works because the plugin's `bin/` directory is added to `$PATH` while the plugin is enabled.)

### Database schema

If you (or Claude) need to create it by hand, the schema is:

| Property | Type | Notes |
|---|---|---|
| Name | Title | Artifact slug (kebab-case, no extension) |
| Type | Select | `doc` · `pr-description` · `commit-message` · `review` · `script` · `config` · `draft` · `other` |
| Repo | Select | `oaknorthbank/react-native-stack`-style, or `(no-remote)` |
| Project path | Text | Absolute path to git root or cwd |
| Extension | Select | `md` · `sh` · `ts` · `py` · `json` · `yaml` · `toml` |
| Local path | URL | `file://...` |
| Status | Select | `current` · `archived` |
| Summary | Text | One-sentence context |
| Created / Last edited | Auto | Notion built-ins |

**Important**: create the database under a **private** Notion page, not a shared team page — your artifacts will live there.

## Usage

```bash
/artifacts              # default: pick + copy raw text (scoped to current repo)
/artifacts list         # table of artifacts in current scope
/artifacts list --all   # show every artifact across repos
/artifacts copy <name>  # copy named artifact (fuzzy matched)
/artifacts share        # copy Notion link instead of text
/artifacts share <name>
/artifacts rm <name>    # delete local cache (Notion row stays)
/artifacts clear        # wipe local cache
/artifacts url          # print Notion database URL
/artifacts save <slug>  # explicit save of most recent shareable output (Claude inference)
/artifacts refresh      # rebuild local cache from Notion (Claude inference)
```

### When Claude auto-saves

Auto-save fires for content that looks like a deliverable you'll paste elsewhere:

- Documentation, READMEs, ADRs, runbooks
- Review comments / PR feedback
- PR descriptions and multi-paragraph commit messages
- Scripts and configs meant to be shared/run
- Slack/email/ticket drafts

It does **not** auto-save: short illustrative snippets, conversational answers, plans, summaries, tool output, or content already written to a file via `Write`/`Edit`.

See `SKILL.md` for the full ruleset Claude reads.

## How it works (architecture)

- **Notion = source of truth.** Every save is an upsert against the database (matched by slug).
- **Local files are write-through cache.** `~/.claude/artifacts/<slug>.<ext>` — exists only for fast `pbcopy`.
- **Registry**: `notion.json` (per-user, gitignored) maps slug → `{page_id, url, repo, project_path}`. Drives the scope filter and link-copy.
- **The bash script** handles list/copy/share/rm/clear locally — single tool call, no Claude in the loop. The model only steps in for `save` and `refresh`.
- **The picker**: when called via `/artifacts` (no TTY in the harness), the script exits 2 with candidate rows + a marker on stderr; the skill prompts via `AskUserQuestion` and re-invokes with `copy <name>`.

## File layout

Source layout (in the [`claude-code-marketplace`](https://github.com/danbovey/claude-code-marketplace) monorepo under `plugins/artifacts/`):

```
plugins/artifacts/
├── .claude-plugin/
│   └── plugin.json           plugin manifest
├── skills/artifacts/
│   └── SKILL.md              instructions the model reads
├── bin/
│   └── artifacts             bash helper (added to $PATH by the plugin loader)
├── notion.example.json       registry template
├── README.md                 this file
└── LICENSE
```

User-data (mutable, lives outside the plugin install so updates don't clobber it):

```
~/.claude/artifacts/
├── .notion.json              your registry (slug → page_id, URL, repo, project_path)
└── <slug>.<ext>              one file per cached artifact
```

## Status

Personal-use prototype, recently shared. Known rough edges before wider rollout:

- **Cross-machine `refresh`** is implemented but lightly tested; expect quirks with `>50` artifacts.
- **`Repo` select option auto-creation** uses `ALTER COLUMN SET` which replaces the full option list; concurrent saves from different machines into new repos could clobber each other's options (transient — re-saving fixes it).
- **No `pbcopy` fallback** for Linux/WSL (PRs welcome — `wl-copy` / `xclip` / `clip.exe`).
- **`jq` is a hard dep** with no preflight check.

## License

[MIT](./LICENSE)
