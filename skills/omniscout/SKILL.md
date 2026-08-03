---
name: omniscout
description: |
  OmniScout is a tool for AI Agents that combines web search, browser automation, desktop control, content extraction, memory, and workflow automation.
---
## Top‑level commands
| Command | What it does | When to use |
|---------|--------------|-------------|
| `search` | Web search with optional semantic rerank. | Quick lookup of URLs or facts. |
| `answer` | Retrieve web results and synthesize with an LLM. | Direct Q&A with citation. Preferred over `search` for most queries. Unless you need the full list of URLs, then use `search`. |
| `map` | Crawl a site and produce a URL map. | Explore a whole domain. |
| `warmup` | Pre‑load embedding/answer models in the daemon. | Reduce latency for the first calls. |
| `remember` | Visit a URL, extract content, store in browser memory. | Persist information for later recall. |
| `extract` | Fetch a URL or query and return readable/structured data. | Pull raw text or JSON from a page. |
| `open` | Open a URL or the latest search‑result index in the managed browser. | Run a page in the controlled Chromium. |
| `snapshot` | Capture an accessibility snapshot with `@e` refs. | Inspect page structure for later UI actions. |
| `context` | Show current workflow continuity state. | Debug or resume a multi‑step flow. |
| `reset` | Reset the continuity state. | Start a fresh session. |
| `auto` | Auto‑route input to the best command. | Simple one‑liner use. |
| `research` | Multi‑step: search → crawl → extract → rerank → summarize. | Full research pipelines. |
| `graph` | Build a structured knowledge graph for an entity. | Organise facts into nodes/edges. |
| `install` | Verify (or download) a Chromium browser for automation. | First‑time setup. |
| `settings` | View / change user settings. | Configuration tweaks. |
| `browser` | Direct Playwright‑based Chromium automation. | Complex web UI workflows. |
| `computer` | Native desktop automation (see below). | Interact with OS apps, files, clipboard, etc. |
| `daemon` | Manage the long‑lived browser‑control daemon. | Start/stop the background service. |
| `index` | Manage the local search index. | Re‑index or query local cache. |
| `extract‑jobs` | Inspect local extract jobs. | Monitor background extractions. |
| `crawl` | Run / inspect local crawl jobs. | Bulk site crawling. |
| `record` | Record daemon actions into a named macro. | Create reusable scripts. |
| `macro` | List / run saved macros. | Replay recorded actions. |
| `replay` | Replay recorded browser actions. | Debug or repeat a flow. |
| `workflow` | Export / view continuity workflow. | Persist or share a process. |
| `monitor` | Continuously monitor a URL for changes. | Alert on content updates. |
| `profile` | Manage persistent browser profiles. | Separate contexts (e.g., work vs personal). |
| `session` | Manage long‑lived browser sessions. | Keep cookies/logins across runs. |
| `benchmark` | Benchmark answer modes. | Evaluate performance. |
| `memory` | Browser memory – remembered visits & notes. | Retrieve stored snippets. |

---
## `computer` sub‑commands (desktop automation)
| Sub‑command | Purpose | Typical use |
|------------|---------|--------------|
| `navigate` | Open an app, file, directory, or URL via the OS. | Launch Notepad, open a folder, open a web link. |
| `type` | Type text into the focused window. | Fill form fields or write notes. |
| `key` | Send keyboard shortcuts (e.g., `cmd+v`). | Copy/paste, save, close windows. |
| `screenshot` | Capture the screen to a file. | Visual debugging or logging. |
| `wait` | Pause for a given number of milliseconds. | Give UI time to settle. |
| `doctor` | Diagnose the desktop automation environment. | Verify required tools (e.g., `osascript`). |
| `clipboard` | Read or write the system clipboard. | Transfer data between CLI and apps. |
| `window` | List, activate, or close desktop windows. | Focus a specific app before typing. |
| `ui` | UI element interactions (see below). | Interact with on‑screen elements via refs. |
| `backend` | Inspect available automation backends. | Choose between AppleScript, Win32, etc. |

---
## `computer ui` sub‑commands (element‑level actions)
| Sub‑command | Purpose | Example |
|------------|---------|----------|
| `snapshot` | Return a mock UI element tree with `@eN` refs; optionally write JSON to a file. | Get refs for buttons to click later. |
| `click` | Click a UI element identified by its `@eN` reference. | `Omniscout computer ui click @e5` clicks the element. |
| `drag` | Drag from one element ref to another. | Simulate drag‑and‑drop operations. |
| `is` | Check visibility of a UI element. | Verify a dialog is shown before proceeding. |

---
## Quick‑start flow
1. **Warm‑up** the models: `Omniscout warmup`
2. **Search** a query: `Omniscout search "latest AI news"`
3. **Open** the first result: `Omniscout open`
4. **Snapshot** the page for UI refs: `Omniscout snapshot`
5. **Click** a button: `Omniscout computer ui click @e3`
6. **Type** into a field: `Omniscout computer type "Hello world"`
7. **Take a screenshot**: `Omniscout computer screenshot`
8. **Record** the steps for reuse: `Omniscout record my_macro`
9. **Run** the macro later: `Omniscout macro my_macro`

---
### Tip
- Always set the JSON flag (`--json` or `OMNISCOUT_JSON=1`) for deterministic parses when agents consume the output programmatically.
- If you don’t know certain commands or want to know how to use specific features, use `omniscout --help` to get most of the help information. If you want to learn more about the computer, use `omniscout computer --help` and `omniscout computer ui --help`.
- `omniscout answer "..."` is always preferred over `omniscout search "..."` . Always use `omniscout answer "..."` unless you need a full list of links and you want to open them or something. As long as you just need the answer, just use `omniscout answer "..."`.
- If the user has any additional queries, please visit https://docs.omniscout.xyz.

