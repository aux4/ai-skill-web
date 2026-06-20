# Web Skill

A method for getting real information off the live web and driving web UIs with `aux4 browser` — a stateful headless-browser daemon. You run every command yourself; this skill is the *discipline*, not a wrapper. The agent already has the browser and the reasoning; this supplies how to use it well: cheaply, robustly, and without leaving a mess.

The goal: answer from real, current page content (never from memory), spend the fewest tokens that get the answer, and always leave the daemon clean.

Learn the exact flags for any step with `aux4 browser <command> --help` — this skill teaches *when* and *how*, not every flag.

## Lifecycle: start before, stop after

`aux4 browser` is a daemon with sessions. Always bracket your work:

1. `aux4 browser start` — bring up the daemon (idempotent; safe if already running).
2. `aux4 browser open --url <url>` — open a session; it returns a `session` id. Reuse that **one** session for the whole task — don't open a new one per step.
3. ... do the work with `--session <id>` ...
4. `aux4 browser close --session <id>` — close the session when the task is done.
5. `aux4 browser stop` — stop the daemon. **Always clean up**, even on failure — a dangling daemon and orphaned sessions leak resources and confuse the next run. If something errors mid-task, still `close` + `stop`.

For a single read with no interaction, you can skip the session dance with the one-shot `read` (below) — but if you opened a session, you own closing it.

## Navigate + extract (the common case)

Prefer the cheapest operation that answers the question. Reading costs tokens; navigation and structure signals are nearly free.

- **One-shot read** — `aux4 browser read --url <url>` navigates, waits for the page to settle, and returns clean main content + status in a single call. This is the fastest path for "just get me this page." For big pages, add `--output <file>` so it spools to disk and returns a small receipt; then read only the slice you need with your own file tools instead of pulling it all into context.
- **In-session read** — with a session open, `aux4 browser content --session <id>` returns the page as markdown (or `--format text`/`html`). Pass a tight `--selector` to scope to just the region you care about — that is far cheaper than the whole page. Use `--output` for large content.
- **Structured extraction first** — when the data is a list (search results, table rows, cards, menu items), use `aux4 browser get-items --session <id> --selector <list>` to pull the items as clean text instead of scraping prose. **Prefer structured extraction over raw `eval`** — it is more robust to markup churn and cheaper to reason about. For tables/forms/lists/tabs as components, `aux4 browser component` reads them structurally.
- **Confirm cheaply before reading expensively** — `open`/`visit` return `{httpStatus, finalUrl, title}`. Check that the route and title are right before pulling content. To see what's interactable, `aux4 browser snapshot --session <id> --format text` returns an indexed accessibility tree (`[6] link "Docs"`, `[12] button "Submit"`) — cheap, and the `[ref]` indices are the reliable way to click.

## Interact (dynamic pages, forms, flows)

When the content only appears after interaction:

- **Navigate within a session** — `aux4 browser visit --session <id> --url <url>` to move the existing session to a new page (keeps cookies/state). If you already know the target URL, visit it directly — don't click through intermediate pages.
- **Click** — `aux4 browser click --session <id> --ref <n>` using a ref from `snapshot` is the most reliable. Fall back to `--name <text> --role <role>`, or `aux4 browser click-text --session <id> --text <visible text>` when you only have visible text. On failure, click returns `{clicked:false, reason, currentUrl, title}` rather than throwing — branch on `reason` and try a ref or a direct `visit`; don't retry blindly.
- **Type / select / check** — `aux4 browser type --session <id> --name <field> --value <text>` to fill inputs; `select`, `check`/`uncheck` for dropdowns and checkboxes. For secrets (passwords, API keys, OTP) pass a `secret://<provider>/<vault>/<item>/<field>` reference as the value — the browser resolves it at runtime so the literal credential is never in your context or logs.
- **Verify state before extracting** — after an action that loads new content, `aux4 browser expect --session <id> --selector <css> --assertion <be_visible|have_text|exist|have_count> [--expected <v>]` to assert the page reached the expected state, or `aux4 browser wait --session <id> --selector <text=...|url=...|networkidle|settle|css>` to wait for it. Extract only after the page is actually ready — reading too early returns stale or empty content.

## When to use `eval` and `screenshot`

- **`eval`** — `aux4 browser eval --session <id> --script <js>` runs custom JavaScript in the page. Use it only as an escape hatch when the high-level commands (`content`, `get-items`, `component`, `snapshot`) genuinely can't get the data — e.g. a value buried in a JS variable or computed in the DOM. Reach for structured commands first; `eval` is brittle against page changes and harder to reason about.
- **`screenshot`** — `aux4 browser screenshot --session <id> --output <file>` for visual confirmation when text extraction is ambiguous (layout, charts, "did the right thing render?"). It is a verification aid, not a substitute for reading text.

## Robustness

- **Wait for content; don't assume it's there.** SPAs and slow pages hydrate after the initial load. Use `--waitUntil networkidle` (or `settle`) on `read`/`visit`/`open` for heavy pages, or `wait`/`expect` before extracting. If a result looks empty or carries a `warning`, the page likely wasn't ready — re-read with a stronger wait, don't conclude the data is missing.
- **Verify before extracting.** Confirm the right page (title/finalUrl) and the right state (`expect`) before paying for a content read.
- **Read narrowly, once.** Scope with `--selector`, spool large pages to disk with `--output`, and never re-read content you already pulled this session.
- **Handle failures gracefully.** Commands return structured failure objects (`clicked:false`, `timedOut:true`) instead of throwing — inspect them and adapt, rather than repeating the same call.

## Rules

- `start` before, `close` the session and `stop` the daemon after — always clean up, even on error. Never leave the daemon running.
- One session per task; reuse it. Visit known URLs directly.
- Get real page content — never answer web questions from memory.
- Cheap signals first (status, title, `snapshot`), then narrow reads; use `--output` + file slicing for large pages.
- Prefer structured extraction (`get-items`, `component`, `content --selector`) over raw `eval`; use `eval` only when nothing else can reach the data.
- Wait/`expect` for dynamic content before extracting; treat empty/warning results as "not ready," not "absent."
- Pass secrets as `secret://` references, never literal credentials.
- Run `aux4 browser` yourself — this skill is the method, not a tool that calls the LLM for you.
