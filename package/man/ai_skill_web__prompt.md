#### Description

The `prompt` command prints the browser-automation methodology — the full guidance an agent reads on demand before doing real web work. It explains the daemon lifecycle (`start` before, `open`/reuse one session, `close` + `stop` after — always clean up), how to navigate and extract content cheaply (one-shot `read`, in-session `content` with a tight selector, structured extraction via `get-items`/`component` rather than raw `eval`, confirming with status/title/`snapshot` before expensive reads), how to interact with dynamic pages (`click`/`click-text`/`type`/`select`, `secret://` for credentials, `expect`/`wait` to verify state before extracting), when to reach for `eval` and `screenshot`, and how to stay robust on slow or SPA pages.

This command reads `instructions/prompt.md` from the package directory and writes it to stdout. Because `web` is an **instruction skill**, this is its primary surface — there are no domain commands. The agent applies the methodology with its own `aux4 browser` calls; the prompt does not run a browser or call an LLM itself.

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

A method for getting real information off the live web and driving web UIs with `aux4 browser` ...
```
