---
name: scout
description: |
  OmniScout is a local-first CLI for AI agents: semantic web search (DuckDuckGo +
  local rerank), one-sentence answers, multi-step research, URL extraction to
  Markdown, browser memory, and full browser automation (navigate, click, fill,
  screenshot, login, CAPTCHA, network capture) via a daemon at 127.0.0.1:7720.
  Use this skill whenever the user wants to search the web, research a topic,
  extract page content, remember pages, or interact with a website. Prefer
  OmniScout over curl/fetch for anything that needs search ranking, clean
  extraction, or a real browser.
---

# OmniScout

Local-first CLI for AI agents. No cloud APIs, no hosted browser sessions.
Everything runs on the user's machine.

**The CLI is the interface.** Every command supports `--json` (or env
`OMNISCOUT_JSON=1`) for structured stdout. Browser commands also have a
curl-equivalent against `http://127.0.0.1:7720/command`.

## 0. When to use what

| User intent | Command | Notes |
|-------------|---------|-------|
| Quick web lookup, ranked results | `omniscout search "query"` | DDG + optional local rerank |
| One-sentence factual answer | `omniscout search "query" --answer` | Plain text; add `--data` for sources/timing |
| Deep multi-source report | `omniscout research "topic"` | Search → crawl → extract → summarize |
| Clean Markdown from a URL | `omniscout extract https://…` | No browser needed; uses on-disk cache |
| Save a page for later recall | `omniscout remember https://…` | Indexes into semantic memory |
| Search pages you've saved | `omniscout search "query" --source memory` | Or `--source hybrid` (memory + DDG) |
| Interact with a live page | `omniscout browser navigate …` | Requires daemon; use `@eN` refs |
| Open result #3 from last search | `omniscout open 3` | Workflow shorthand |
| Extract from a snapshot ref | `omniscout extract @e12` | Resolves via workflow state |

**Rule of thumb:** start with `search` or `research` for information gathering;
use `extract` for a known URL; use `browser` when the page needs clicks,
logins, JS rendering, or network inspection.

## 1. Health check (always do this first)

```bash
omniscout daemon status
```

Then act on the result:

- **`running: true`** — healthy. Proceed.
- **`running: false`** — start the daemon: `omniscout daemon start`. Idempotent.
- **command not found** — install: `pip install omniscout && omniscout install`
- **`extension_connected: false`** — fine for default Playwright backend; only
  matters if the user wants the Chrome extension backend.

Search, research, and memory commands auto-start the daemon and route embeddings
through it. Browser commands require it. Optional warm-up before a batch:

```bash
omniscout warmup   # preload embedding model (~2s, stays hot in daemon RAM)
```

Read [references/operations.md](references/operations.md) when the daemon is
misbehaving.

## 2. JSON output

Set once per shell session:

```bash
export OMNISCOUT_JSON=1
```

Every command emits structured JSON to stdout; logs go to stderr. Browser
responses include `action_id` (stable hex) for trace/replay and
`snapshot_generation` (monotonic per session — re-snapshot when it changes).

## 3. Search

```bash
omniscout search "local-first browser agents" --limit 10
omniscout search "taskgroup asyncio" --source memory
omniscout search "robotics simulators" --source hybrid --rerank
```

| Flag | Values | Default |
|------|--------|---------|
| `--source` | `ddg`, `index`, `memory`, `hybrid` | `ddg` |
| `--limit`, `-n` | int | 10 |
| `--rerank/--no-rerank` | local embedding rerank | on |
| `--answer` | one-sentence answer (plain text stdout) | off |
| `--depth` | `auto`, `fast`, `balanced`, `deep` | `auto` (with `--answer`) |
| `--data` | timing, sources, diagnostics | off |

**Sources:**

- `ddg` — live DuckDuckGo HTML results
- `index` — local Qdrant crawl corpus from past research
- `memory` — remembered visits and notes (`omniscout remember`)
- `hybrid` — memory + DDG, deduplicated by URL

**Answer mode** (`--answer`):

```bash
# Sub-second: DDG snippets only
omniscout search "who is the president" --answer

# Local index + DDG fallback (may load embedding model)
omniscout search "vector databases 2026" --answer --depth balanced

# Crawl + extract top sources (slow)
omniscout search "FedRAMP LLM hosting" --answer --depth deep

# Full diagnostics
omniscout search "…" --answer --data
```

With `--answer` alone, stdout is **plain text** (just the sentence). Add
`--data` for JSON with sources, `elapsed_ms`, `cache_hit`, etc.

After search, use workflow shortcuts:

```bash
omniscout open 1          # open first hit in browser
omniscout context         # see active session, latest search set, refs
```

## 4. Research

Multi-step pipeline: search → crawl → extract → embed → rerank → summarize.

```bash
omniscout research "state of local AI agents in 2026"
omniscout research "vector database benchmarks" --results 12 --depth 2
omniscout research "topic" --no-summarize   # sources only, skip summary
```

| Flag | Default | Meaning |
|------|---------|---------|
| `--results`, `-k` | 8 | Sources to fetch |
| `--depth` | 1 | Crawl depth (1 = listing only) |
| `--summarize/--no-summarize` | on | Extractive summary |

JSON output includes `topic`, `summary`, and `sources[]` with `url`, `title`,
`score`. Use this for broad topics; use `search --answer` for quick facts.

## 5. Extract

Fetch a URL and return clean content (trafilatura + markdownify). Uses on-disk
HTML cache; no browser session required.

```bash
omniscout extract https://example.com
omniscout extract https://example.com --format text
omniscout extract https://example.com --format json
omniscout extract @e12    # resolve ref from latest snapshot
```

Formats: `markdown` (default), `text`, `json`. Pass `--no-cache` to bypass
disk cache.

## 6. Browser memory

Explicit indexing — normal navigate/extract/snapshot do **not** auto-index.

```bash
omniscout remember https://docs.python.org
omniscout remember https://docs.anthropic.com --note "API refs"
omniscout search "taskgroup" --source memory

omniscout memory list [--kind visit|note] [--limit N]
omniscout memory show <id>
omniscout memory note "standalone note text"
omniscout memory delete <id>
omniscout memory delete --url https://…
omniscout memory stats
omniscout memory clear --yes
```

Memory lives in `$OMNISCOUT_DATA_DIR/memory.sqlite` with vectors in Qdrant.
Search it with `--source memory` or `--source hybrid`.

## 7. Workflow shortcuts

Top-level aliases that tie search, browser, and extract together:

```bash
omniscout open <url-or-index> [--session NAME] [--profile NAME] [--new-tab]
omniscout snapshot [--refs-only] [--session NAME]
omniscout context    # workflow state: session, search set, snapshot generation
omniscout reset      # clear workflow state (does not close browser sessions)
omniscout workflow export [--session NAME] [--since SECONDS] [--from-context]
```

Deterministic rules:

- `omniscout open 1` = first item from the **latest search result set only**
- `omniscout extract @eN` resolves through latest snapshot lineage
- Re-snapshot if refs are stale before retrying extraction

## 8. Browser automation

All browser commands accept `--session NAME` (default `default`) and `--json`.
Prefer top-level `omniscout snapshot` / `omniscout open` when chaining with
search; use `omniscout browser …` for the full verb set.

| Tool | Args | Returns | Notes |
|------|------|---------|-------|
| `navigate` | `url`, `--new-tab`, `--profile`, `--headful` | `{url, title, tab_id, status}` | First call binds session to backend. |
| `snapshot` | `--refs-only` | `{url, title, tree, refs:[{ref,role,name,value}]}` | **Use `@eN` refs to locate elements.** |
| `click` | selector or `--coord X Y` | `{mode, tag, text}` | `@eN` preferred over CSS. |
| `fill` | selector, value | `{mode, tag}` | Works on inputs and contenteditable. |
| `scroll` | direction, `--amount`, `--ref` | `{direction, amount}` | |
| `key` | combo | `{combo}` | e.g. `cmd+a`, `Escape`, `Enter`. |
| `hover` | selector or `--coord X Y` | `{mode}` | |
| `screenshot` | `--out`, `--ref`, `--full-page` | `{format, path, size_bytes}` | **Read the returned path via your Read tool.** |
| `pdf` | `--out`, `--paper`, `--landscape` | `{path, size_bytes}` | Playwright backend only. |
| `eval` | code | `{type, value}` | Use compact `JSON.stringify`; wrap in IIFE for fresh scope. |
| `wait` | `--ref` / `--url` / `--idle` / `--ms` | `{reason}` | |
| `upload` | selector, files… | `{file_count, files}` | Playwright only. |
| `tab list/close/switch` | tab_id | tab metadata | |
| `network start/stop/list/detail` | `--filter` | request/response entries | |
| `login` | url, `--profile`, `--success-pattern` | `{logged_in, final_url}` | Headful + blocks; use `login --done` to resume. |
| `captcha` | `--solver none\|2captcha\|…`, `--detect-only` | captcha status | Default `none` = local-first, user solves. |
| `close` | `--session`, `--all` | `{closed}` | **Always call at end of task.** |

### curl equivalent

```bash
curl -s -X POST http://127.0.0.1:7720/command \
  -H 'Content-Type: application/json' \
  -d '{"action":"navigate","args":{"url":"https://example.com"},"session":"demo"}'
```

Body: `{action, args, session, timeout_ms?}`. Response:
`{ok, action, data?, error?, error_kind?, elapsed_ms, action_id}`.

### Sessions

Each `--session NAME` isolates tabs, refs, and network logs. Restart-safe: the
daemon persists last URL per session and re-navigates after restart (bump in
`snapshot_generation` signals re-snapshot).

Same-session commands are serialised; different sessions run in parallel.

### Element addressing — prefer `@eN` refs

`snapshot` mints stable `@eN` refs from the accessibility tree. Compare
`snapshot_generation` on every response; if it changed, re-snapshot before
using cached refs. Stale refs return `error_kind: no_such_ref`.

### Login flow

```bash
omniscout browser login https://github.com --profile work --success-pattern '/(?!login)'
# user authenticates in headful tab, then:
omniscout browser login --done
omniscout browser navigate https://github.com --profile work   # already logged in
```

### CAPTCHA flow

```bash
omniscout browser captcha --detect-only
omniscout browser captcha                         # blocks until user solves
omniscout browser captcha --solver 2captcha       # opt-in; needs TWOCAPTCHA_API_KEY
```

### Two backends

- **Playwright** (default) — local Chrome with persistent profile
- **Extension** (opt-in) — user's real Chrome via `chrome.debugger`; set
  `args.backend = "extension"` on first `navigate`

### Known limitations

- Sites checking `event.isTrusted` reject synthetic click/fill
- Cross-origin iframes: navigate to iframe URL directly
- `pdf` and `upload` require Playwright backend

## 9. Profiles

Persistent Chrome user-data-dirs; login state survives across runs.

```bash
omniscout profile create work
omniscout profile list
omniscout profile delete work
omniscout browser navigate https://github.com --profile work
```

## 10. Observability

Every daemon action is logged to `$OMNISCOUT_DATA_DIR/daemon/actions.jsonl`:

```bash
omniscout daemon trace -n 10 --session demo
omniscout daemon trace last
omniscout daemon replay <action_id>
omniscout daemon replay --session demo --since 60
omniscout replay action-<action_id>          # top-level alias
omniscout daemon watch [--filter action.finish] [--json-lines]
omniscout workflow export --session demo --since 300
```

## 11. Typical agent workflows

**Research a topic:**

```bash
export OMNISCOUT_JSON=1
omniscout research "vector databases benchmark 2026" --results 10
```

**Search → open → extract:**

```bash
omniscout search "OmniScout browser agent" --limit 5
omniscout open 1
omniscout snapshot --refs-only
omniscout extract @e3    # or extract the page URL directly
omniscout browser close --all
```

**Logged-in scrape:**

```bash
omniscout profile create work
omniscout browser login https://github.com/login --profile work
# user finishes auth
omniscout browser login --done
omniscout browser navigate https://github.com/settings --profile work
omniscout browser snapshot --refs-only
omniscout browser close --all
```

**Build a personal knowledge base:**

```bash
omniscout remember https://docs.example.com --note "API reference"
omniscout remember https://blog.example.com/post
omniscout search "rate limiting" --source memory
omniscout memory stats
```

## 12. Cleanup

```bash
omniscout browser close --session demo
omniscout browser close --all
omniscout daemon stop    # rare; daemon normally stays up between tasks
```

Always close browser sessions at the end of a task. The daemon can stay running.

For troubleshooting, read [references/operations.md](references/operations.md).
