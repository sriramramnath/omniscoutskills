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

## First-time setup

Install OmniScout and its agent instructions with `omniscout install --skill`.
Add `--probe-0-mini` only when the user wants the optional local answer model
(about 700 MB). Then choose the user's Chromium browser and run
`omniscout setup --browser <browser-id>`. Setup opens the extension and remote
debugging pages and starts the operating system's Accessibility permission
flow. Ask the user to review and approve the permissions themselves. macOS may
show an Accessibility prompt; Windows and Linux may open settings instead. In
the browser, the user must turn on **Allow remote debugging for this browser
instance** and click **Allow** if prompted. Do not claim setup is complete
until the user confirms and the extension reports connected.

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
omniscout browser inspect --refs-only             # get @eN refs and a matching screenshot
omniscout browser snapshot --interactive          # use when you need the full tree
omniscout browser observe "Mac" --limit 8         # optional: rank refs
omniscout browser click @e3                       # use a ref from snapshot
omniscout browser snapshot --interactive          # after navigation / dialogs
omniscout browser fill @e12 "94103"               # when typing is needed
omniscout browser check @e18                     # checkbox or switch on
omniscout browser uncheck @e19                   # checkbox or switch off
```

Use `browser_extract_visible` (or `omniscout browser extract-visible`) when
you need named facts from the current page. A JSON schema can request common
fields such as email, pricing, or headquarters, as well as values written
under matching labels.

For a visual target with no accessible label, use MCP `browser_find_image`.
Pass `image_base64` when the reference image is only available to the agent,
or `template` with a local path when it is on the OmniScout machine. On a match,
use the returned viewport coordinates with `browser_click`.

For MCP, use `browser_action_batch` to run up to 50 ordered browser actions in
one call. Supported steps include navigation, click, fill, type, dropdown
selection, key, scroll, hover, wait, and screenshot. A batch stops at its first failure and returns the
completed results with the failing step index and an updated page image. Set
`include_screenshot` to false for text-only output. Use separate calls when you
need to inspect the page between actions.
Use `browser_find` for page text search and `browser_fill_form` to fill several
controls in one serialized call. `browser_network` and `browser_console` help
inspect page requests, messages, and errors.

The CLI also runs ordered actions in one daemon request:

```bash
omniscout browser batch '[{"type":"click","selector":"@e3"},{"type":"wait","text":"Saved"}]' --session work --no-screenshot
```

Rules for the tool:

1. **Navigate once** to start (or when changing sites). When using a connected or configured desktop browser, OmniScout opens a **new tab** and leaves the page you were using alone. Keep using the same OmniScout session for the tab it opened. Then **snapshot → click/fill**. On macOS, OmniScout routes URLs to the running browser app and does not start another Dia or Arc app process. If you need full page DOM access in your existing Dia tab, connect the OmniScout browser extension and switch to that tab; a separate managed Playwright session uses isolated Chromium.
2. Re-snapshot after the page changes or `error_kind: no_such_ref` / `stale_ref`.
   If snapshot returns `page_changed`, wait for the page to settle and retry.
   If navigation times out, inspect the response data before retrying. A
   landed URL with `navigation_complete: false` means the document is usable;
   snapshot it first and continue in the same tab. If the URL is unchanged or
   unknown, inspect the current session and wait before retrying. Also inspect
   `snapshot_generation`. Treat `data.url` as current only when
   `data.current_url_known` is true; extension timeouts return a `last_known_url`
   that may be stale. If the managed browser closes, OmniScout reopens the
   session at its last page; take a fresh snapshot before using any old refs.
   `browser tab list` also drops tabs closed outside OmniScout; if every tab was
   closed, navigate to the next URL to open a new tab.
3. Prefer `observe "label"` / `act "click Buy"` when unsure which `@eN` to use.
   Use `act "hover over Menu"` or `act --ref @eN --action hover` to reveal
   hover-only controls without clicking them. For a native dropdown, use
   `act --ref @eN --action select --value "Pro"` to choose its visible option.
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
4. **One session, one automation target.** Prefer `--new-tab` on the first
   navigate, then reuse the same session. Native computer mode opens a separate
   browser window on macOS and keeps later actions bound to it. Connected
   browser extensions open a separate tab. When Dia is configured, use
   `browser navigate`; OmniScout opens a new tab in the running Dia app and
   must not launch a second Dia process. Keep the page you were already using
   untouched. If OmniScout cannot safely reuse the running app, report the
   failure instead of starting another copy.
5. **Stop before payment.** Never enter cards or Apple ID passwords.
6. **Restart daemon** after upgrading: `omniscout daemon stop && omniscout daemon start`.
7. Snapshots default to interactive-only; re-snapshot after navigation / stale refs.
8. Login/CAPTCHA: `browser login` / `browser captcha` (headful).
9. To inspect the desktop inventory in one call, run `computer state`; it lists
   installed apps and open windows. For window actions, use `computer window list` →
   `computer window activate --id <ID>` → `computer observe` →
   `computer ui click @eN`. Use `--click-count 2` for a double-click, and
   `computer paste` for multiline plain text. On macOS, pass `--format html`
   or `--format md` to paste rich content when the target app supports it.
   Linux and Windows currently accept plain text only.
   Use `computer window bind --id <ID>` to keep observations and later actions
   attached to an exact window on macOS, Linux, and Windows. On macOS this
   scopes accessibility observations and window actions; keyboard and mouse
   input activates the app. Linux sends keyboard input without activating the
   selected window, though some apps ignore background input. Windows activates
   the selected window when sending input.
   Add `--key cmd+shift` to hold modifiers during mouse clicks, drags, moves,
   or scrolling. Use `computer ui mouse_move X Y --duration-ms 300` to move
   gradually; movement is immediate by default. Clicks also accept
   `--duration-ms 500` to hold the mouse button.
   Coordinate and accessibility clicks also accept
   `--button wheel` for a middle click, plus `--button back` and
   `--button forward` for browser navigation buttons.
10. `computer key` accepts common xdotool key names, including `F1`–`F20`
    and keypad names such as `KP_0`. Add `--duration-ms` to hold a key or
    modifier combo before it is released.
11. Use `computer screenshot --window-id <ID>` to capture one window on macOS,
    Linux, and Windows. Use `computer observe --window-id <ID>` when the
    accessibility view and screenshot should come from the same selected
    window. Windows captures the selected window's visible screen area,
    including anything covering it.
    Use `computer capabilities` (or MCP's `computer_capabilities`) to check
    which desktop actions are available on the current platform and their known
    limits.
12. When connected through MCP, use the dedicated `computer_observe` and
    `computer_ui_*` tools for desktop observation and actions. Use the
    coordinate-based pointer tools when an accessibility ref is unavailable;
    use `computer_state` to inspect apps and windows, and
    `computer_app_list` / `computer_app_launch` to choose and open an app.
    Use `computer_window_list`, `computer_window_activate`, and
    `computer_window_bind` to select the target window.
    In a bound Linux session, take coordinates from the bound window's screenshot;
    macOS and Windows use desktop screenshot coordinates. Some Linux apps ignore
    mouse input sent to an inactive window.
    Coordinate drags can include intermediate `path` points in
    `computer_ui_mouse_drag` or a batch action such as
    `{"type":"drag","x1":10,"y1":20,"x2":300,"y2":250,"path":[[80,90],[160,40]]}`;
    omit the path for a straight drag.
    Use MCP's `computer_action_batch` or `omniscout computer batch '[{"type":"click","x":10,"y":20},{"type":"type","text":"hello"}]'` for a short ordered sequence of up to 50 desktop actions. It stops at the first failure and reports which steps ran.
    `computer_paste` is useful for longer text; `computer_wait_for` waits for
    accessible text to appear or disappear.
    Use `computer ui find "Save" --role button --match exact` or MCP's
    `computer_ui_find` to retrieve fresh refs; inspect ambiguous matches before
    acting.
    Use `computer wait-for "Saved"` or MCP's `computer_wait_for` to wait for accessible text to appear or disappear.
13. For browser sessions through MCP, use `browser_tab_list` to inspect open
    tabs, then `browser_tab_switch` with a tab ID or exact URL to bind one.
    Use `browser_tab_new` to open another tab or `browser_attach_cdp` to bind
    to an existing Chromium process. Use `browser_tab_close`, `browser_back`, `browser_forward`, and
    `browser_reload` for tab cleanup and page history. Use the dedicated
    `browser_select`, `browser_check`, `browser_paste`, `browser_upload`, and
    `browser_download` tools for those page controls and file workflows.
    `browser_find` searches page text or a regular expression;
    `browser_fill_form` fills multiple named fields in one call.
    Use `browser_screenshot_image` or `computer_observe_image` when you need a
    screenshot delivered as an image the agent can inspect.

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
omniscout workflow mark-deliverable <path-or-url> [--summary TEXT]
omniscout workflow mark-handoff "what is complete and what needs attention next"
```

Marks are saved in workflow context and shown by `omniscout context`. Always
share the deliverable path or URL with the user; a handoff mark records the
summary for another agent to read.

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
| `inspect` | `--refs-only`, `--interactive`/`--no-interactive`, `--screenshot-out`, `--base64` | accessibility refs and screenshot | Use to see the page and its refs in one observation. |
| `snapshot` | `--refs-only`, `--interactive`/`--no-interactive`, `--depth`, `--selector`, `--overlay` | `{url, title, tree, refs}` | Interactive-only by default. |
| `observe` | instruction, `--limit` | `{matches[], snapshot_generation}` | Rank refs by instruction. |
| `act` | instruction, `--value`, `--dry-run` | `{performed, target, result}` | One-shot observe + click/fill. |
| `click` | selector or `--coord X Y`, `--highlight` | `{mode, tag, text}` | `@eN` preferred over CSS. |
| `fill` | selector, value, `--highlight` | `{mode, tag}` | Works on inputs and contenteditable. |
| `select-text` | selector, text, `--prefix`, `--suffix`, `--selection-type` | selected range or cursor position | Editable input, textarea, or contenteditable; add prefix/suffix when a match repeats. |
| `type` | selector, value, `--delay-ms` | `{mode, tag, length}` | Appends text at the current cursor. |
| `paste` | selector, value, `--format text|md|html` | `{mode, tag, length}` | Inserts at the current cursor; HTML stays rich where the active browser session supports it; ordinary fields receive its text content. Markdown is inserted as source text. |
| `select` | selector, value, `--by label|value` | `{mode, tag, by, value}` | Choose an option from a native dropdown. Native computer mode accepts an accessible dropdown ref and one visible option label. |
| `get` | `url`, `title`, `text`, or `html` | `{property, value}` | Read a page property. |
| `is` | selector, state | `{selector, state, value}` | Check visible, hidden, enabled, disabled, checked, or editable state. |
| `mouse move/scroll` | `--coord`, `--steps` for move, and `--delta` for scroll | coordinates and deltas | Move or scroll at a page position. Movement is direct by default; add steps for intermediate pointer positions. |
| `network` / `console` | `start`, `stop`, `list`, `tail` | captured requests or messages | Captures automatically after OmniScout attaches; `start` clears the log and begins a fresh capture. |
| `history` | `--from`, `--to`, `--query`, `--limit` | recent visits | Search browser visits by time or page title/URL. The extension requests Chrome history access. |
| `download` | selector, optional `--out` | `{path, filename, url}` | Save a download from Playwright or a connected Chrome tab. Default paths keep earlier files when names repeat. |
| `clipboard` | `read`, `write`, `read-items`, `write-items` | clipboard data | Use the active browser session's clipboard; extension access requires Chrome's clipboard permissions. |
| `check` / `uncheck` | selector | requested checked state | In Computer Use, use an accessible `@eN` control whose checked state can be read. A selected radio button cannot be unchecked by itself. |
| `scroll` | direction, `--amount`, `--pixels`, `--pages` (macOS), `--ref` | `{direction, amount}` | Wheel-based pixel distances are approximate; Windows pixels require `--x` and `--y`. |
| `key` | combo | `{combo}` | e.g. `cmd+a`, `Escape`, `Enter`. |
| `hover` | selector or `--coord X Y` | `{mode}` | |
| `screenshot` | `--out`, `--ref`, `--region`, `--stitched`, `--full-page`, `--full-length`, `--delay` | `{format, path, size_bytes}` | `--delay SEC` before capture. **Read the returned path via your Read tool.** |
| `pdf` | `--out`, `--paper`, `--landscape` | `{path, size_bytes}` | Saves the current page as a PDF from a headless Playwright session or a connected Chrome tab. |
| `eval` | code | `{type, value}` | Use compact `JSON.stringify`; wrap in IIFE for fresh scope. |
| `wait` | selector state, text, URL regex/glob, idle, mutation quiet, or milliseconds | `{reason}` | Wait for the page condition before continuing. |
| `emulate-media` | `--media`, `--color-scheme`, `--reduced-motion`, `--forced-colors`, `--contrast` | applied preferences | Preview print styles or accessibility preferences; pass `reset` to restore one preference. |
| `upload` | selector, files…, `--open-chooser` | `{file_count, files, mode}` | Upload to a file field, or click a control and handle its file picker with Playwright. |
| `tab list/close/switch` | tab_id or `--url URL` | tab metadata | Use `--url` to bind the matching open tab; use a tab ID if the URL matches more than one tab. |
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

In Computer Use mode, an element screenshot needs a recent `computer observe`
snapshot with bounds for its `@eN` ref. Computer Use captures the visible desktop;
full-page and stitched screenshots need a DOM browser session.

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

Same-session commands are serialised. Sessions on separate tabs can run in
parallel; actions on the same tab wait for one another. Sessions sharing a
profile keep the same sign-in state but start on separate tabs. Close sibling
sessions before changing browser visibility. On the extension backend, each
tab can belong to one session at a time; close or switch the owning session
before selecting that tab elsewhere. A session cannot close another session's
active tab.

### Element addressing — prefer `@eN` refs

`snapshot` mints stable `@eN` refs from the accessibility tree. Compare
`snapshot_generation` on every response; if it changed, re-snapshot before
using cached refs. Stale refs return `error_kind: no_such_ref` or `stale_ref`, depending on the backend.

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
- Extension sessions support same-origin frame CSS selectors as `frame=<iframe CSS> >> <element CSS>`; cross-origin frames may require the Playwright backend.
- `pdf` requires the Playwright backend

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

With the browser extension connected, you can list and use an already open
browser tab without opening a second one:

```bash
omniscout browser tab list --session work
omniscout browser tab switch 42 --session work
omniscout browser tab switch --url https://example.com --session work
omniscout browser snapshot --interactive --session work
```

Use the `tab_id` returned by `tab list` in the switch command.
Closing the session leaves an existing tab open; tabs created by OmniScout are
closed with their session.

To wait for an element to reach a specific state, use a CSS selector or a
current `@eN` ref. The default state is `visible`:

```bash
omniscout browser wait selector "#results" --state visible
omniscout browser wait selector "#loading" --state hidden
omniscout browser wait selector "#dialog" --state detached
```

The other supported states are `attached` and `detached`. Re-snapshot after
page changes before using a ref.

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
