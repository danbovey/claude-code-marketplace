---
name: artifacts
description: Track and copy shareable outputs (docs, review comments, PR descriptions, scripts, configs) produced during a Claude Code session. Saves to a Notion database ("Claude Code Artifacts") as the source of truth, with a local clipboard cache at ~/.claude/artifacts/ for instant pbcopy. Triggers on `/artifacts`, `/artifacts save|list|copy|share|rm|clear|refresh`, or when the user asks to "save that", "copy that as an artifact", or otherwise wants to grab Claude-produced content cleanly. Also use proactively (without being asked) right after producing a substantial shareable output — see "When to auto-save" below.
---

# /artifacts — session artifact tracker (Notion-backed)

Saves Claude-produced shareable content to **Notion** as the canonical store, with a **global local cache** at `~/.claude/artifacts/` so the user can `pbcopy` the raw text without terminal-wrap line breaks.

## Architecture

- **Source of truth: Notion database** `Claude Code Artifacts` — every artifact is a row. One row per artifact name. Re-saving the same name **updates** the existing row, never creates a duplicate.
- **Local cache: `~/.claude/artifacts/<slug>.<ext>`** — global (not per-project), used only for fast `pbcopy`. Always overwritten to match Notion.
- **Registry: `~/.claude/artifacts/.notion.json`** — maps artifact slug → `{page_id, url, repo, project_path}`. The script reads it for "copy link" actions, **scope filtering** (default `/artifacts` only shows artifacts saved from the current repo or cwd), and recovery. Saves update it. Hidden dotfile so it doesn't appear in the artifact cache listing.

The `artifacts` command (installed on `$PATH` by the plugin's `bin/` directory) handles all read-side commands locally (no inference). Save and refresh need Claude in the loop because they call the Notion MCP.

## When to auto-save (proactive, no prompt)

After producing output in chat, save it as an artifact **without being asked** if it is shareable content the user is likely to paste somewhere else:

- **Documentation** — READMEs, design docs, ADRs, runbooks, explanations longer than a paragraph or two.
- **Review comments / PR feedback** — multi-bullet review notes, suggested-changes blocks, structured feedback.
- **PR descriptions / commit messages** — full PR body, multi-paragraph commit messages.
- **Scripts intended to be shared/run** — standalone shell scripts, one-off Python utilities.
- **Configs to share** — YAML/JSON/TOML blobs.
- **Slack/email/ticket drafts** — anything that reads like outbound communication.

**Do NOT** auto-save: content already written to a file via Write/Edit; short illustrative snippets (<15 lines); conversational answers / summaries / plans; tool output or logs.

Announce auto-saves in one short line at the end of your response:

> Saved as artifact `pr-description` → [Notion](<url>) · run `/artifacts` to copy.

Pick the slug yourself; don't ask. Slugs: lowercase kebab-case, no spaces, no extension (the metadata holds the extension).

## How to save (the `save` flow)

This is the only inference-heavy part. Do it in one Claude response, in this order — the writes can fire in parallel:

### 1. Resolve metadata from cwd

```bash
# All in one Bash call, captured into shell vars if useful:
CWD="$(pwd)"
GIT_ROOT="$(git rev-parse --show-toplevel 2>/dev/null || true)"
REPO="$(git remote get-url origin 2>/dev/null | sed -E 's|.*[:/]([^/]+/[^/]+)(\.git)?$|\1|')"
PROJECT_PATH="${GIT_ROOT:-$CWD}"
```

Use `(no-remote)` for `Repo` when there's no git remote.

### 2. Look up existing page in `notion.json`

```bash
jq -r --arg s "<slug>" '.pages[$s].page_id // ""' ~/.claude/artifacts/.notion.json
```

- If present → call `mcp__claude_ai_Notion__notion-update-page` with the page_id, updating properties and content.
- If absent → call `mcp__claude_ai_Notion__notion-create-pages` with `parent.data_source_id` from `notion.json` (`.data_source_id`).

### 2.5. Ensure the `Repo` select option exists

Select properties do not auto-create options. Before the create/update call, check whether the resolved `Repo` value (e.g. `oaknorthbank/acorn-2026`) is already in the schema:

```bash
# Fetch current Repo options from a recent successful response, or via:
# mcp__claude_ai_Notion__notion-fetch with id = collection://<data_source_id>
```

If the value is **not** in the existing options, add it first:

```
mcp__claude_ai_Notion__notion-update-data-source
  data_source_id: <from notion.json>
  statements: ALTER COLUMN "Repo" SET SELECT('(no-remote)':gray, '<existing-1>':blue, ..., '<new-repo>':blue)
```

**Important**: `ALTER COLUMN SET` replaces the full option list, so include all existing options too. If the create call fails with `Invalid select value for property "Repo"`, this is the recovery path.

### 3. Properties to set

| Property | Value |
|---|---|
| `Name` | `<slug>` (no extension) |
| `Type` | One of `doc` · `pr-description` · `commit-message` · `review` · `script` · `config` · `draft` · `other`. Pick based on what the content is. |
| `Repo` | `oaknorthbank/react-native-stack`-style, or `(no-remote)`. **Select options do NOT auto-create** — see step 2.5 below. |
| `Project path` | Absolute path: `$GIT_ROOT` or `$CWD`. |
| `Extension` | `md` · `sh` · `ts` · `py` · `json` · `yaml` · `toml` — based on content. Default `md`. |
| `Local path` | `file://$HOME/.claude/artifacts/<slug>.<ext>` |
| `Status` | `current` |
| `Summary` | **One sentence** of context: what was being worked on / why this artifact matters. Don't restate the content. |

### 4. Content (page body)

The raw deliverable as Notion-flavored Markdown. **No "Here's your artifact:" preamble. No surrounding code fences** (unless the artifact itself is a markdown doc that contains fenced examples). For non-markdown artifacts (scripts, configs), wrap the content in a single fenced code block with the right language tag so Notion syntax-highlights it.

### 5. Write local cache + update registry

In parallel with the MCP call, write the local cache file with `Write`:

```
~/.claude/artifacts/<slug>.<ext>
```

(Raw content, no fence wrapper, even for scripts — the cache exists for `pbcopy`, which wants the bare text.)

After the MCP call returns, update `~/.claude/artifacts/.notion.json` to record the new page mapping **and the scope metadata** (`repo`, `project_path`) so the script's scope filter can find this artifact later:

```json
{
  "pages": {
    "<slug>": {
      "page_id": "<id-from-response>",
      "url": "<url-from-response>",
      "repo": "<resolved Repo value>",
      "project_path": "<resolved Project path>"
    }
  }
}
```

(Preserve other entries — read, modify, write the JSON; don't blow it away.) These fields drive the default scope filter, so missing/wrong values mean the artifact won't appear in `/artifacts` when you `cd` back to that repo.

### 6. Confirm to the user

One short line:

> Saved `<slug>` → [Notion](<url>) · `<bytes>` bytes cached locally.

## Subcommands

### Read-side (delegate to the script — no inference)

| User runs | You run |
|---|---|
| `/artifacts` | `artifacts` |
| `/artifacts list` | `artifacts list` |
| `/artifacts copy <name>` | `artifacts copy <name>` |
| `/artifacts share [name]` | `artifacts share [name]` |
| `/artifacts rm <name>` | `artifacts rm <name>` |
| `/artifacts clear` | `artifacts clear` |
| `/artifacts url` | `artifacts url` |
| `/artifacts init [--reconfigure]` | `artifacts init [--reconfigure]` (exit 2 + `NEEDS_NOTION_SETUP` → see init flow below) |
| `/artifacts all` / `/artifacts list all` | append `--all` to drop the scope filter |

**Scope filter (default behaviour):** `list` / picker only shows artifacts saved from the current git repo (matched by `origin`), or from the current cwd when there's no remote. If nothing matches, the script falls back to showing everything (graceful — empty pickers are worse than too-many). Pass `--all` to disable filtering explicitly.

### Multi-candidate picker (how `/artifacts` and `/artifacts share` work with >1 artifact)

When more than one artifact exists, the script exits **2** and writes tab-separated candidate rows + a marker to stderr — it does **not** prompt interactively (no TTY in the harness). Markers:

- `NEEDS_PICKER` — user wants to copy text
- `NEEDS_PICKER_LINK` — user wants the Notion link

**When you see exit 2:**

1. Parse stderr rows: `<filename>\t<size>\t<age>\t<link-present>`.
2. Call `AskUserQuestion` (label = name without extension, description = `<size> · saved <age>`). Cap at the first ~10. `AskUserQuestion` requires ≥2 options, which is guaranteed here.
3. After the pick: call `artifacts copy <name>` (for `NEEDS_PICKER`) or `... share <name>` (for `NEEDS_PICKER_LINK`).

With **1 artifact** the script auto-copies directly (no marker, no extra round-trip).

### `/artifacts save <slug>` (inference required)

Follow the save flow above. If `<slug>` is omitted, propose one and confirm with `AskUserQuestion` only if ambiguous; otherwise just pick.

### `/artifacts init` (inference required when script signals)

First-time setup, and re-setup. The bash script handles dep preflight, scaffolds `notion.json` from `notion.example.json` if missing, and detects whether the Notion side already looks configured. If it does, `init` exits 0 and is done. If not (or `--reconfigure` was passed), it exits **2** with `NEEDS_NOTION_SETUP` on stderr — and you take over.

**Hard rule: this flow must be non-destructive.** Never overwrite a working `notion.json` without merging. Never delete a DB. Never modify the schema of an existing DB you've adopted. If the user already has a private `Claude Code Artifacts` database, adopt it — do not create a duplicate.

**Steps when you see `NEEDS_NOTION_SETUP`:**

1. **Search for an existing database** owned by the user:

   ```
   mcp__claude_ai_Notion__notion-search
     query: "Claude Code Artifacts"
     page_size: 10
   ```

2. **Filter to private candidates.** For each result, call `mcp__claude_ai_Notion__notion-fetch` and inspect the `<ancestor-path>` tag:
   - **Empty ancestor-path** → workspace-root / private. ✓ candidate.
   - **Has ancestors that include a teamspace, shared page, or other user's space** → SKIP. Do not adopt content the user can't fully control.
   - Also skip results that aren't databases (e.g. the OakNorth company guide page literally titled "Claude Artifacts" — type=page, not database).

3. **Decide adopt vs create:**
   - **0 private database candidates** → create one:
     ```
     mcp__claude_ai_Notion__notion-create-database
       title: "Claude Code Artifacts"
       description: "Shareable outputs from Claude Code sessions. Managed by the /artifacts skill."
       schema: <same CREATE TABLE as below>
     ```
     (no `parent` → workspace-root / private)

     **Icon (best-effort):** after creation, attempt to set the icon via `mcp__claude_ai_Notion__notion-update-page` (with the new database_id as `page_id`, `command: update_properties`, `properties: {"title": "Claude Code Artifacts"}`, `icon: ":claude-code:"`). If the resulting fetch shows the icon didn't change (the MCP appears to silently no-op the icon on databases in some cases — or the named emoji isn't in this workspace), retry once with `icon: "📒"`. Either way, don't block setup on the icon — it's purely cosmetic.
   - **1 private candidate** → adopt it. Fetch its data source (`collection://...`) to extract the `data_source_id` and full URL.
   - **2+ private candidates** → use `AskUserQuestion` to let the user pick which to use.

4. **Adopted databases: validate schema.** Compare the existing data source's schema to the expected one (below). If it's missing properties, **do NOT alter it silently** — show the user what's missing and ask whether to (a) add the missing columns via `notion-update-data-source ADD COLUMN`, or (b) abort and let them fix it manually. Never DROP, RENAME, or change types on an adopted database without explicit consent.

5. **Write `notion.json` non-destructively.** Read the existing file, merge:
   ```json
   {
     "database_id": "<new>",
     "data_source_id": "<new>",
     "data_source_url": "collection://<new>",
     "database_url": "<new>",
     "title": "Claude Code Artifacts",
     "pages": { /* preserve any existing entries — they may still be valid if reconfiguring */ }
   }
   ```
   Keep the `_comment` field. Use `Write` after reading + merging in memory; do not blindly clobber.

6. **Confirm:**

   > Setup complete — database at `<database_url>` (adopted existing | freshly created).
   > Try: ask me to save something, or `/artifacts list`.

#### Expected schema (CREATE TABLE)

```sql
CREATE TABLE (
  "Name" TITLE,
  "Type" SELECT('doc':blue, 'pr-description':green, 'commit-message':green, 'review':purple, 'script':orange, 'config':yellow, 'draft':gray, 'other':default),
  "Repo" SELECT('(no-remote)':gray),
  "Project path" RICH_TEXT,
  "Extension" SELECT('md':default, 'sh':orange, 'ts':blue, 'py':green, 'json':yellow, 'yaml':yellow, 'toml':gray),
  "Local path" URL,
  "Status" SELECT('current':green, 'archived':gray),
  "Summary" RICH_TEXT
)
```

### `/artifacts refresh` (inference required)

Pull current state from Notion to rebuild the local cache + registry. Steps:

1. Query the data source:
   ```
   mcp__claude_ai_Notion__notion-query-data-sources with
   data_source_urls: ["collection://<data_source_id from notion.json>"]
   query: SELECT url, "Name", "Extension", "Local path" FROM "collection://..." WHERE "Status" = ? ORDER BY createdTime DESC
   params: ["current"]
   ```
2. For each row, fetch the page body via `mcp__claude_ai_Notion__notion-fetch` and `Write` it to `~/.claude/artifacts/<Name>.<Extension>` (extract the body from the Markdown-formatted content, stripping the page header).
3. Rebuild `notion.json` `pages` map from the query results (slug → page_id + url).
4. Report: `Refreshed N artifacts from Notion.`

Use refresh sparingly — it's slow (one fetch per row). Useful when starting on a new machine or when the local cache has been wiped.

## Conventions

- **Always overwrite.** Same slug = update existing Notion row + overwrite local file. No version history.
- **Raw deliverable only** in both Notion body and local cache — no preamble, no closing remarks, no instructional fences around the content.
- **Slugs**: lowercase kebab-case, no spaces, no extension. The `Extension` property holds the extension; the local cache file uses it.
- **Markdown is the default.** Pick a code extension only when the content is genuinely code/config that benefits from syntax highlighting.
- **Don't `cd` into `$ART_ROOT`** in any Bash call — the harness preserves cwd, and a stray `cd` poisons future relative paths. The script uses absolute paths throughout; do the same when calling MCP tools.

## Failure modes

- **Notion MCP down / unauthenticated** → the local cache write still succeeds; tell the user the Notion mirror failed and to re-auth via `/mcp`. The artifact is recoverable from local; next save will retry.
- **Local cache miss on copy** → the script will say "not cached locally"; suggest `/artifacts refresh`.
- **Script-level errors** (empty cache, ambiguous match, missing pbcopy) → surface verbatim, don't try to recover with extra tool calls.
