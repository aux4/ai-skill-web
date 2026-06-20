# agent/skill-web

A native aux4 agent skill that gives an agent a **methodology** for driving a headless browser with `aux4 browser` — how to navigate, extract content, interact with dynamic pages, verify state, and clean up so web work is cheap, robust, and leaves no dangling daemon.

It is an **instruction skill**: prompt-only, no domain commands, no `run`, and no LLM dependency. It does not wrap `aux4 browser`. The agent already has `aux4 browser` (`start`, `open`, `visit`, `read`, `content`, `get-items`, `click`, `click-text`, `type`, `expect`, `eval`, `screenshot`, `close`, `stop`, ...) and the LLM to reason — this skill supplies the discipline the agent applies with its own browser calls. It depends on `aux4/ai-skill` only (for the shared `ai:skill` profile and the skill contract).

## Installation

```bash
aux4 aux4 pkger install agent/skill-web
```

## Quick Start

```bash
aux4 ai skill web prompt
```

This prints the methodology. The agent reads it, then drives the browser directly:

```bash
aux4 browser start
session=$(aux4 browser open --url https://example.com | jq -r .session)
aux4 browser content --session "$session" --selector main
aux4 browser close --session "$session"
aux4 browser stop
```

## What the Methodology Covers

- **Lifecycle** — `start` the daemon before, `open` one session and reuse it for the whole task, `close` the session and `stop` the daemon after. Always clean up, even on failure — never leave the daemon running.
- **Navigate + extract** — one-shot `read` for "just get me this page" (spool large pages with `--output`), in-session `content` with a tight `--selector` for scoped reads, structured extraction with `get-items`/`component` over raw `eval`, and confirming cheaply (status, title, `snapshot`) before paying for a content read.
- **Interact** — `click`/`click-text`/`type`/`select` for dynamic pages, `secret://` references for credentials so literals never enter context, and `expect`/`wait` to verify the page reached the right state before extracting.
- **When to use `eval` and `screenshot`** — `eval` only as an escape hatch when high-level commands genuinely can't reach the data; `screenshot` for visual confirmation.
- **Robustness** — wait for hydration on slow/SPA pages, verify before extracting, read narrowly and once, and adapt to the structured failure objects commands return instead of retrying blindly.

Read the full guidance with `aux4 ai skill web prompt`.

## Commands

This is an instruction skill — it contributes one command to the shared `ai:skill` profile.

| Command | Description |
|---------|-------------|
| `aux4 ai skill web prompt` | Print the browser-automation methodology (lifecycle, navigate+extract, interact, robustness) |

## Native Skill Contract

This package conforms to the native aux4 skill contract:

- **scope `agent`, name `skill-web`** — depends on `aux4/ai-skill` only; never on `aux4/browser`, `aux4/ai-agent`, or `aux4/copilot`. It is pure methodology; the agent that uses it already has `aux4 browser`.
- **`help.text` everywhere** — the required discovery layer (`aux4 ai skill web --help`).
- **`man/ai_skill_web__prompt.md` and `man/ai_skill__web.md`** — the "is this a good fit?" tier.
- **`prompt`** — the methodology itself; this is an instruction skill so `prompt` is its primary surface.
- **No `run`, no domain commands** — running a skill is a runtime concern of `aux4/ai-agent`, and this skill teaches a method rather than performing a deterministic capability.

Verify conformance:

```bash
aux4 ai skill validate web
aux4 ai skill list
```

## License

This package is licensed under the Apache-2.0 License.

See [LICENSE](./LICENSE) for details.
