---
name: omniscout
description: |
  OmniScout is a local browser + search tool for agents. For websites, write
  individual commands: browser navigate → snapshot → observe/act/click @eN /
  fill. Also search, answer, research, extract. Prefer OmniScout browser tools
  over IDE browser MCPs. Do not rely on `auto` as a black box — drive the page
  step by step with working @eN refs.
---

# OmniScout

Local-first CLI for AI agents. No cloud APIs, no hosted browser sessions.
Everything runs on the user's machine.

**Website:** [omniscout.xyz](https://omniscout.xyz) · **Docs:** [docs.omniscout.xyz](https://docs.omniscout.xyz)

**The CLI is the interface.** Set `export OMNISCOUT_JSON=1` once per session.

## Page content and exports

Use `browser snapshot --visible-text` to include rendered page text in the
snapshot. On supported pages, `browser content export-gsuite` saves Google
Workspace files and `browser content export-youtube-transcript` saves captions.
Use `browser webmcp list` to see tools offered by the page, and `browser assets
list` or `browser assets bundle` to inspect or save page resources.

## Browser tool (step-by-step — primary path)

OmniScout is a **tool**: you open a tab, read the page, click, type. You choose
each step. Snapshot returns `@eN` refs (same idea as Cursor’s `e40`).

```bash
omniscout daemon start
export OMNISCOUT_JSON=1

omniscout browser navigate "https://www.apple.com" --headful --new-tab
omniscout browser snapshot --interactive          # get @eN refs
omniscout browser observe "Mac" --limit 8         # optional: rank refs
omniscout browser click @e3                       # use a ref from snapshot
omniscout browser snapshot --interactive          # after navigation / dialogs
omniscout browser fill @e12 "94103"               # when typing is needed
omniscout browser check @e18                     # checkbox or switch on
omniscout browser uncheck @e19                   # checkbox or switch off
```

Rules for the tool:

1. **Navigate once** to start (or when changing sites). The first open uses a **new tab**; later steps **reuse that tab** by default. Then **snapshot → click/fill**.
2. Re-snapshot after the page changes or `error_kind: no_such_ref`.
3. Prefer `observe "label"` / `act "click Buy"` when unsure which `@eN` to use.
4. Same `--session` (default) for the whole task. Do not reopen the same URL.
5. Stop before payment / card / passwords.
6. Do **not** use `extract` / `answer` / IDE browser MCP to drive a shop UI.

`browser navigate` opens a **managed automation window** with a full DOM (no
Chrome extension). That is how `@eN` clicks work reliably.

## 0. When to use what

| User intent | Command | Notes |
|-------------|---------|-------|
| Quick web lookup, ranked results | `omniscout search "query"` | DDG + optional local rerank |
| One-sentence factual answer | `omniscout answer "query"` | Plain text; add `--data` for sources/timing |
| Deep multi-source report | `omniscout research "topic"` | Search → crawl → extract → summarize |
| Structured entity map (tree) | `omniscout graph "Cursor"` | 5 sources by default; evidence-first NLP tree |
| Entity map from one website | `omniscout graph "cursor.com"` or `graph "X" -w URL` | Site seed URLs; skips DDG |
| Exa-style JSON from search | `omniscout search "…" --output-schema-inline '{...}'` | Multi-source NLP + schema validation |
| Exa-style JSON from URL | `omniscout extract URL --schema-inline '{...}'` | Same envelope as structured search |
| Clean Markdown from a URL | `omniscout extract https://…` | No browser needed; uses on-disk cache |
| Structured facts from a page | `omniscout extract https://… --format structured` | Auto-extract all fields it can (NLP, no LLM) |
| Structured facts from a query | `omniscout extract -q "SpaceX founder" --format structured --fields founder` | DDG + multi-level crawl, then extract |
| Specific fields only | `omniscout extract https://… --format structured --fields company,pricing,twitter` | Only requested keys |
| Save a page for later recall | `omniscout remember https://…` | Indexes into semantic memory |
| Search pages you've saved | `omniscout search "query" --source memory` | Or `--source hybrid` (memory + DDG) |
| Interact with a live page | `omniscout browser navigate …` | Requires daemon; use `@eN` refs |
| Full-page screenshot (top to bottom) | `omniscout browser screenshot --full-length` | Viewport-only by default; `--full-page` is an alias; `--stitched` scroll-stitches instead |
| Partial screenshot (element or pixels) | `omniscout browser screenshot --ref "#hero"` or `--region 0,200,1280,800` | `--ref` takes `@eN` refs or CSS selectors |
| Open result #3 from last search | `omniscout open 3` | Workflow shorthand |
| Extract from a snapshot ref | `omniscout extract @e12` | Resolves via workflow state |

**Rule of thumb:** start with `search` or `research` for information gathering;
use `graph` when the user wants a structured overview of a product, company, or
person; use `extract` for a known URL; use `browser` when the page needs
clicks, logins, JS rendering, or network inspection.

### Agent rules (browser — read before any browser command)

1. **Drive with primitives.** `navigate` → `snapshot` → `click`/`fill`/`act`.
   You write each command; OmniScout executes it. `auto` is optional routing only.
2. **Open before you act.** `browser navigate URL --headful` (or `omniscout open`)
   first on the session, then snapshot/click on that same session.
3. **Use `@eN` from the latest snapshot.** `observe` / `act` help pick refs.
4. **One session, one tab.** Prefer `--new-tab` on the first navigate; then
   `--reuse-tab` or default session for later navigations. Do not spam open.
5. **Stop before payment.** Never enter cards or Apple ID passwords.
6. **Restart daemon** after upgrading: `omniscout daemon stop && omniscout daemon start`.
7. Snapshots default to interactive-only; re-snapshot after navigation / stale refs.
8. Login/CAPTCHA: `browser login` / `browser captcha` (headful).
9. Pure native apps (not the browser): `computer window list` →
   `computer window activate --id <ID>` → `computer observe` →
   `computer ui click @eN`. Use `--click-count 2` for a double-click, and
   `computer paste` for multiline plain text. On macOS, pass `--format html`
   or `--format md` to paste rich content when the target app supports it.
   Linux and Windows currently accept plain text only.
10. `computer key` accepts common xdotool key names, including `F1`–`F20`
    and keypad names such as `KP_0`.

### Automation settings (user-tunable)

```bash
omniscout settings set open_new_tab false          # reuse tab on open
omniscout settings set snapshot_interactive false  # full AX tree
omniscout settings set action_highlight true       # flash targets when acting
omniscout settings set browser-automation headful   # your browser + Computer Use
omniscout settings set browser-automation headless  # managed Playwright (full DOM, default)
```

## 1. Health check (always do this first)

```bash
omniscout daemon status
```

Then act on the result:

- **`running: true`** — healthy. Proceed.
- **`running: false`** — start the daemon: `omniscout daemon start`. Idempotent.
- **command not found** — install: `pip install omniscout && omniscout install`
- **Wrong browser / need Edge or Brave** — `omniscout settings browsers` then
  `omniscout settings set browser edge` (or `brave`, `vivaldi`, `arc`, `custom
  --executable /path/to/binary`). Install also prompts interactively.
- **`default_backend_for_new_session: unified`** — **extension** (if connected),
  else **headful** opens your configured browser and automates via **Computer
  Use**; **headless** uses managed Playwright (full DOM). Toggle in
  `omniscout settings` → Browser automation or `settings set browser-automation`.
- **`automation_channel`**: `extension` | `computer` | `playwright` — always
  check it before snapshot/click.

Check status fields `open_new_tab`, `snapshot_interactive`, `action_highlight`,
`default_headful`, and `default_backend_for_new_session` for runtime defaults.

Search, research, and memory commands auto-start the daemon and route embeddings
through it. Browser commands require it. Optional warm-up before a batch:

```bash
omniscout warmup   # preload embedding + answer LLM (~2s, stays hot in daemon RAM)
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
| `--data` | timing and diagnostics on search hits | off |

**Sources:**

- `ddg` — live DuckDuckGo HTML results
- `index` — local Qdrant crawl corpus from past research
- `memory` — remembered visits and notes (`omniscout remember`)
- `hybrid` — memory + DDG, deduplicated by URL

For a one-line factual answer, use [`omniscout answer "query"`](#4-answer). Search
returns ranked hits only.

After search, use workflow shortcuts:

```bash
omniscout open 1          # open first hit in browser
omniscout context         # see active session, latest search set, refs
```

## 4. Answer

Grounded one-sentence answers from web retrieval plus a small local LLM
(SmolLM2-360M, cached on disk). **Direct DuckDuckGo answers are tried first**
(HTML box, snippets, Search Assist), then extractive parsing from filtered
hits, then the local LLM, then a limited crawl for difficult who-is queries.
Falls back to extractive-only when the model is unavailable.

```bash
omniscout answer "who is the prime minister of india"
omniscout answer "who is the president of the us" --depth fast
omniscout answer "what is the capital of france"
omniscout answer "how tall is mount everest"
omniscout answer "what is 2+2"
omniscout answer "vector databases 2026" --depth balanced
omniscout answer "FedRAMP LLM hosting" --depth deep
omniscout answer "who is the pm of singapore" --data
omniscout answer "…" --no-llm   # extractive fallback only
```

| Flag | Values | Default |
|------|--------|---------|
| `--depth` | `auto`, `fast`, `balanced`, `deep` | `auto` |
| `--limit`, `-n` | int | 10 |
| `--data` | sources, timing, `llm_backend`, diagnostics | off |
| `--no-llm` | skip local LLM synthesis | off |

Without `--data`, stdout is **plain text** (just the answer). Add `--data` for
sources, `elapsed_ms`, and retrieval diagnostics (`direct+snippet`,
`extractive+ddg`, `llm+crawl`, etc.).

Prefetch on install or warm-up:

```bash
omniscout install --answer-model
omniscout warmup
```

## 5. Graph

Knowledge graph for an entity — Company, Founders, Competitors, Pricing,
Features, Reviews, and more — as a Unicode tree. Default: evidence-first NLP from
5 sources (optional `--llm` for Classic LLM overlay).
crawl (no browser fallback). Uses the same local answer LLM as `omniscout answer`.

```bash
omniscout graph "Cursor"
omniscout graph "cursor.com"                    # URL → site-only crawl
omniscout graph "Cursor" --website cursor.com   # label + pinned site
omniscout graph "Cursor" --data                 # tree + sources/timing
omniscout graph "Cursor" --llm                  # optional Classic LLM overlay
```

| Flag | Default | Meaning |
|------|---------|---------|
| `--results`, `-k` | 5 | Web hits or in-site pages |
| `--website`, `-w` | — | Same-host crawl only |
| `--data` | off | Sources and diagnostics |
| `--llm` | off | Optional Classic LLM overlay |

Prefer `graph` over `research` when the user wants a structured entity overview,
not a prose report. Pass a URL (or `-w`) when they point at a specific site.

## 6. Research

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
`score`. Use this for broad topics; use `omniscout answer` for quick facts.

## 7. Extract

Fetch a URL and return clean content (trafilatura + markdownify). Uses on-disk
HTML cache; no browser session required.

```bash
omniscout extract https://example.com
omniscout extract https://example.com --format text
omniscout extract https://example.com --format json
omniscout extract https://example.com --format structured
omniscout extract https://example.com --format structured --fields company,pricing,twitter
omniscout extract -q "Acme Inc pricing" --format structured --depth 3 --results 5
omniscout extract https://example.com --format json --fields company,pricing
omniscout extract https://example.com --format structured --data   # full ExtractResult
OMNISCOUT_JSON=1 omniscout search "Cursor pricing" --output-schema-inline '{"type":"object","properties":{"pricing":{"type":"string"}},"required":["pricing"]}'
OMNISCOUT_JSON=1 omniscout extract https://cursor.com/pricing --schema-inline '{"type":"object","properties":{"pricing":{"type":"string"}},"required":["pricing"]}'
omniscout extract @e12    # resolve ref from latest snapshot
```

Formats: `markdown` (default), `text`, `json`, `structured`. `--format structured`
auto-extracts metadata, pricing, founder, social links (Twitter/X, LinkedIn,
GitHub, YouTube, …), important page URLs (docs, blog, careers, community,
status, …), contact info, and any `Label: value` lines on the page. Pass
`--fields` to limit keys; `--data` for full metadata and stderr logs. Pass
`--no-cache` to bypass disk cache.

**Query mode** (`--query` / `-q`): omit the URL, search DuckDuckGo, crawl top
`--results` seed URLs (default 5), follow same-host links up to `--depth`
levels (default 3), merge text, then extract. Requires `--format structured` or
`--fields`.

## 8. Browser memory

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

## 9. Workflow shortcuts

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

## 10. Browser automation

All browser commands accept `--session NAME` (default `default`) and `--json`.
Prefer top-level `omniscout snapshot` / `omniscout open` when chaining with
search; use `omniscout browser …` for the full verb set.

| Tool | Args | Returns | Notes |
|------|------|---------|-------|
| `navigate` | `url`, `--new-tab`/`--reuse-tab`, `--profile`, `--headful` | `{url, title, tab_id, status}` | New tab is default from settings. |
| `snapshot` | `--refs-only`, `--interactive`/`--no-interactive`, `--depth`, `--selector`, `--overlay` | `{url, title, tree, refs}` | Interactive-only by default. |
| `observe` | instruction, `--limit` | `{matches[], snapshot_generation}` | Rank refs by instruction. |
| `act` | instruction, `--value`, `--dry-run` | `{performed, target, result}` | One-shot observe + click/fill. |
| `click` | selector or `--coord X Y`, `--highlight` | `{mode, tag, text}` | `@eN` preferred over CSS. |
| `fill` | selector, value, `--highlight` | `{mode, tag}` | Works on inputs and contenteditable. |
| `scroll` | direction, `--amount`, `--ref` | `{direction, amount}` | |
| `key` | combo | `{combo}` | e.g. `cmd+a`, `Escape`, `Enter`. |
| `hover` | selector or `--coord X Y` | `{mode}` | |
| `screenshot` | `--out`, `--ref`, `--region`, `--stitched`, `--full-page`, `--full-length`, `--delay` | `{format, path, size_bytes}` | `--delay SEC` before capture. **Read the returned path via your Read tool.** |
| `pdf` | `--out`, `--paper`, `--landscape` | `{path, size_bytes}` | Playwright backend only. |
| `eval` | code | `{type, value}` | Use compact `JSON.stringify`; wrap in IIFE for fresh scope. |
| `wait` | `--ref` / `--url` / `--idle` / `--ms` | `{reason}` | |
| `upload` | selector, files… | `{file_count, files}` | Playwright only. |
| `tab list/close/switch` | tab_id | tab metadata | |
| `network start/stop/list/detail` | `--filter` | request/response entries | |
| `login` | url, `--profile`, `--success-pattern` | `{logged_in, final_url}` | Headful + blocks; use `login --done` to resume. |
| `captcha` | `--solver none\|2captcha\|…`, `--detect-only` | captcha status | Default `none` = local-first, user solves. |
| `close` | `--session`, `--all` | `{closed}` | **Always call at end of task.** |

### Screenshots

Viewport-only by default. Pass `--full-length` or `--full-page` (aliases) to
capture the entire scrollable page from top to bottom. Pass `--stitched` to
scroll viewport-by-viewport and stitch the frames into one PNG instead (more
reliable on lazy-loaded pages; PNG only). Use `--ref @eN` — or a raw CSS
selector like `--ref "#hero"` — to capture a single element instead. Use
`--region x,y,w,h` for an exact pixel rectangle. Always pass `--out` to a path
you can read back.

```bash
omniscout browser screenshot --out /tmp/viewport.png
omniscout browser screenshot --full-length --out /tmp/full.png
omniscout browser screenshot --stitched --out /tmp/stitched.png
omniscout browser screenshot https://example.com --full-page --out /tmp/page.png
omniscout browser screenshot --ref @e3 --out /tmp/button.png
omniscout browser screenshot --ref "#hero" --out /tmp/hero.png
omniscout browser screenshot --region 0,200,1280,800 --out /tmp/mid.png
omniscout browser screenshot --delay 3 --out /tmp/after-load.png
```

Daemon JSON args: `"full_page": true`, `"delay_ms": 3000` (works with curl/`POST /command` too).

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

- **Extension** (preferred when connected) — user's browser (Dia, Chrome, …) via
  the scout extension; `agent:scout` banner on driven tabs.
- **Playwright** — managed Chromium profile when the extension is not connected;
  set `args.backend = "playwright"` on first `navigate` to force.

### Visible automation

- Extension tabs show an `agent:scout` banner.
- `snapshot --overlay` draws `@eN` labels on the page (demos / user watching).
- `browser click` / `fill` / `computer ui click` accept `--highlight` (or
  `settings.action_highlight`) for a brief visual cue.

### Token & latency

- Use `snapshot --refs-only` (interactive filtering is already on by default).
- Avoid full-page screenshots unless stuck; prefer `extract` over browsing.
- Use `browser act` to skip manual ref hunting when possible.

### Known limitations

- Sites checking `event.isTrusted` reject synthetic click/fill
- Cross-origin iframes: navigate to iframe URL directly
- `pdf` and `upload` require Playwright backend

## 11. Profiles

Persistent Chrome user-data-dirs; login state survives across runs.

```bash
omniscout profile create work
omniscout profile list
omniscout profile delete work
omniscout browser navigate https://github.com --profile work
```

## 12. Observability

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

## 13. Typical agent workflows

**Minimum browser loop (OpenCode / agents):**

```bash
export OMNISCOUT_JSON=1
omniscout daemon status
omniscout search "topic" --limit 5
omniscout open 1 --headful
omniscout snapshot --refs-only
omniscout browser click @e3
omniscout browser close --all
```

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

**Capture a full-page screenshot:**

```bash
omniscout browser navigate https://example.com
omniscout browser wait networkidle
omniscout browser screenshot --full-length --out /tmp/example-full.png
# or: omniscout browser screenshot --stitched --out /tmp/example-stitched.png
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

## 14. Cleanup

```bash
omniscout browser close --session demo
omniscout browser close --all
omniscout daemon stop    # rare; daemon normally stays up between tasks
```

Always close browser sessions at the end of a task. The daemon can stay running.

For troubleshooting, read [references/operations.md](references/operations.md).
