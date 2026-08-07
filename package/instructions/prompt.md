# Web Skill

A method for getting real information off the live web and driving web UIs with `aux4 browser` — a stateful headless-browser daemon. You run every command yourself; this skill is the *discipline*, not a wrapper. The agent already has the browser and the reasoning; this supplies how to use it well: cheaply, robustly, and without leaving a mess.

The goal: answer from real, current page content (never from memory), spend the fewest tokens that get the answer, and always leave the daemon clean.

Learn the exact flags for any step with `aux4 browser <command> --help` — this skill teaches *when* and *how*, not every flag.

## Lifecycle: start before, stop after

`aux4 browser` is a daemon with sessions. Always bracket your work:

1. `aux4 browser start` — bring up the daemon (idempotent; safe if already running).
2. `aux4 browser open --url <url>` — start a session; it returns a `session` id. Do this **once**. `open` always creates a *new* session with no cookies, so calling it again silently throws away everything you are signed in to — to go to another page, use `visit --session <id>`, never `open`.
3. ... do the work with `--session <id>` ...
4. `aux4 browser close --session <id>` — close the session when the task is done.
5. `aux4 browser stop` — stop the daemon. **Always clean up**, even on failure — a dangling daemon and orphaned sessions leak resources and confuse the next run. If something errors mid-task, still `close` + `stop`.

For a single read with no interaction, you can skip the session dance with the one-shot `read` (below) — but if you opened a session, you own closing it.

## Navigate + extract (the common case)

Prefer the cheapest operation that answers the question. Reading costs tokens; navigation and structure signals are nearly free.

- **One-shot read** — `aux4 browser read --url <url>` navigates, waits for the page to settle, and returns clean main content + status in a single call. This is the fastest path for "just get me this page." For big pages, add `--output <file>` so it spools to disk and returns a small receipt; then read only the slice you need with your own file tools instead of pulling it all into context.
- **In-session read** — with a session open, `aux4 browser content --session <id>` returns the page as markdown (or `--format text`/`html`). Pass a tight `--selector` to scope to just the region you care about — that is far cheaper than the whole page. Use `--output` for large content.
- **Structured extraction first** — when the data is a list (search results, table rows, cards, menu items), use `aux4 browser get-items --session <id> --selector <list>` to pull the items as clean text instead of scraping prose. **Prefer structured extraction over raw `eval`** — it is more robust to markup churn and cheaper to reason about. For tables/forms/lists/tabs as components, `aux4 browser component` reads them structurally.
- **Confirm cheaply before reading expensively** — navigation returns `{httpStatus, finalUrl, title}`. Check that the route and title are right before pulling content. To see what's interactable, `aux4 browser snapshot --session <id> --format text` returns an indexed accessibility tree (`[6] link "Docs"`, `[12] button "Submit"`) — cheap, and the `[ref]` indices are the reliable way to click.

## Interact (dynamic pages, forms, flows)

When the content only appears after interaction:

- **Read the page before you act on it** — `aux4 browser snapshot --session <id> --format text` lists what is actually there, each element with a `ref`. Act on those names and refs, not on what the task called things: a task saying "full name" does not mean the field is named `full_name`. `aux4 browser visit --session <id> --url <url>` moves the session to a new page (keeps cookies/state), but visit only a URL you were given or read off the page; to reach anything else, click the link or button that leads there. Every action reports the `finalUrl` it left you on — that is where you are, not where you meant to be.
- **Click** — `aux4 browser click --session <id> --ref <n>` using a ref from `snapshot` is the most reliable. Fall back to `--name <text> --role <role>`, or `aux4 browser click-text --session <id> --text <visible text>` when you only have visible text. On failure, click returns `{clicked:false, reason, currentUrl, title}` rather than throwing — branch on `reason` and try a ref or a direct `visit`; don't retry blindly.
- **Fill a form** — `aux4 browser fill --session <id> --field "<name>=<value>" --field ...` fills every field in one call. Use this for any form. The page decides how each field is filled — a text box is typed into, a dropdown is selected, a checkbox is ticked — so you don't have to know which is which, and can't pick the wrong one. A dropdown value may be the option's label or its value; a checkbox takes `yes`/`no`. Then submit with `click`.
- **Type / select / check** — the single-field commands behind `fill`, when you need one field at a time: `aux4 browser type --session <id> --name <field> --value <text>` for inputs, `select` for dropdowns, `check`/`uncheck` for checkboxes. Using `type` on a dropdown will not work — that is what `fill` saves you from. For secrets (passwords, API keys, OTP) pass a `secret://<provider>/<vault>/<item>/<field>` reference as the value — the browser resolves it at runtime so the literal credential is never in your context or logs.
- **Check each action did what you meant, before the next one** — every command reports its own result: `fill` says how many fields it filled and names the ones it could not, `click` returns the page it landed on. `filled: 0` or a `finalUrl` you did not expect means the page is not what you assumed — re-read it and act on what is actually there, rather than continuing the plan or concluding the task cannot be done. A form is only submitted once you have clicked its button and seen the page change. When content loads asynchronously, `aux4 browser expect --session <id> --selector <css> --assertion <be_visible|have_text|exist|have_count> [--expected <v>]` asserts the page reached the expected state and `aux4 browser wait --session <id> --selector <text=...|url=...|networkidle|settle|css>` waits for it — extract only once the page is actually ready.
- **An `httpStatus` tells you what to do next** — navigation reports one, and each calls for a different response. `401` you are not signed in: find the sign-in form and sign in, and if you have no credentials, ask for them or create an account — you are not signed in merely because you signed up. `403` you are signed in but not allowed: that route is closed, do not retry it. `404` the page is not there — if you assembled that URL yourself, that is why: go back to the last page that worked and reach the next one by clicking a link or button, not by guessing another address. `429` you are going too fast: wait, then retry. `5xx` the server failed, not you: retry once, and if it repeats, stop and say so. Retrying helps for `429` and `5xx` only; for `401` and `404` you must change what you are doing, and for `403` you must stop.

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
