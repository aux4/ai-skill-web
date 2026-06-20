#### Description

The `web` skill gives an agent a methodology for driving a headless browser with `aux4 browser` — how to navigate, extract content, interact with dynamic pages, verify state, and clean up so web work is cheap, robust, and leaves no dangling daemon. It is contributed to the shared `ai:skill` profile (`aux4 ai skill web ...`).

It is an **instruction skill**: it has no domain commands, no `run`, and no LLM dependency. It does **not** wrap `aux4 browser` — the agent already has `aux4 browser` (`start`, `open`, `visit`, `read`, `content`, `get-items`, `click`, `click-text`, `type`, `expect`, `eval`, `screenshot`, `close`, `stop`, ...) and the LLM to reason. This skill supplies the *discipline* the agent applies with its own browser calls. Its only command is `prompt`, which prints the methodology on demand.

The methodology covers:

- **Lifecycle** — `start` the daemon before, `open` one session and reuse it, `close` the session and `stop` the daemon after; always clean up, even on failure.
- **Navigate + extract** — one-shot `read` vs in-session `content`; structured extraction with `get-items`/`component` over raw `eval`; confirm cheaply (status, title, `snapshot`) before reading expensively.
- **Interact** — `click`/`click-text`/`type`/`select` for dynamic pages; `expect`/`wait` to reach the right state before extracting; `secret://` references for credentials.
- **When to use `eval` and `screenshot`** — `eval` only as an escape hatch when high-level commands can't reach the data; `screenshot` for visual confirmation.
- **Robustness** — wait for hydration on slow/SPA pages, verify before extracting, read narrowly and once, adapt to structured failure objects.

Load this skill when an agent is about to do real web work — scraping content, searching, or driving a web UI. For a single trivial page fetch the agent's direct `aux4 browser read` is enough; engage this skill when the task involves interaction, dynamic content, multiple pages, or careful token/robustness handling.

#### Usage

```bash
aux4 ai skill web prompt
```

#### Example

```bash
aux4 ai skill web prompt
```

```text
# Web Skill
...
```
