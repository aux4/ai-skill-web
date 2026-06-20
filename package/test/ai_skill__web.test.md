# ai skill web

Instruction-skill tests for the migrated `agent/skill-web` package. These assume the
package is installed locally so the shared `ai:skill` profile and the
`aux4 ai skill validate`/`list` framework commands can discover it:

```bash
aux4 aux4 releaser install --dir packages/ai/ai-skill-web/package --noBuild true
```

## prompt

### should output the web skill heading

```execute
aux4 ai skill web prompt
```

```expect:partial
# Web Skill
```

### should cover the daemon lifecycle

```execute
aux4 ai skill web prompt
```

```expect:partial
## Lifecycle: start before, stop after
```

### should cover navigate and extract

```execute
aux4 ai skill web prompt
```

```expect:partial
## Navigate + extract
```

### should cover interaction

```execute
aux4 ai skill web prompt
```

```expect:partial
## Interact
```

### should cover when to use eval and screenshot

```execute
aux4 ai skill web prompt
```

```expect:partial
## When to use `eval` and `screenshot`
```

### should cover robustness

```execute
aux4 ai skill web prompt
```

```expect:partial
## Robustness
```

### should reference the current browser start command

```execute
aux4 ai skill web prompt
```

```expect:partial
aux4 browser start
```

### should reference the current browser stop command

```execute
aux4 ai skill web prompt
```

```expect:partial
aux4 browser stop
```

### should instruct the agent to run aux4 browser itself

```execute
aux4 ai skill web prompt
```

```expect:partial
aux4 browser
```

### should include the rules section

```execute
aux4 ai skill web prompt
```

```expect:partial
## Rules
```

### should NOT reference the old copilot skills web path

```execute
aux4 ai skill web prompt | grep -c "copilot skills web" || true
```

```expect
0
```

### should NOT use the old browser open/eval-only API

```execute
aux4 ai skill web prompt | grep -cE "browser (open|eval|close|execute) \"" || true
```

```expect
0
```

## native skill contract

### should be registered under the ai:skill profile

```execute
aux4 ai skill web --help
```

```expect:partial
Methodology for driving a headless browser
```

### validate should pass

```execute
aux4 ai skill validate web
```

```expect:partial
conforms to the native skill contract
```

### list should show web

```execute
aux4 ai skill list
```

```expect:partial
web
```

### should NOT expose a run command

```execute
aux4 ai skill web --help | grep -c "  run" || true
```

```expect
0
```

### should NOT delegate to ai agent ask

```execute
grep -c "ai agent ask" ../.aux4 || true
```

```expect
0
```

### should NOT expose domain commands beyond prompt

```execute
aux4 ai skill web --help | grep -cE "^  (search|fetch|open|fields|interact|download|save-pdf|close|run) " || true
```

```expect
0
```
