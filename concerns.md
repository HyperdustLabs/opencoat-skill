# Authoring concerns

A Concern is the only first-class data unit OpenCOAT understands. This
file shows the canonical shape, walks through the three demo concerns
that ship with the runtime, and then offers four starter recipes you
can copy-paste and tweak.

## Canonical shape

```yaml
id: my-concern
name: Human-readable label
description: One sentence — why does this concern exist?
pointcut:
  joinpoints: [before_tool_call]        # one or more joinpoint ids
  match:
    any_keywords: ["rm -rf"]            # optional — narrows activation
advice:
  type: TOOL_GUARD                      # see "Advice types" below
  content: |
    Refuse the call — destructive shell command blocked by my-concern.
weaving_policy:
  mode: BLOCK                           # INSERT | BLOCK | ANNOTATE | REPLACE
  level: TOOL_LEVEL                     # PROMPT | TOOL | MEMORY | OUTPUT
  target: tool_call.arguments           # COPR target path (see below)
  priority: 0.9                         # 0.0–1.0; higher wins ties
```

Save as YAML or JSON, then:

```bash
opencoat concern import ./my-concern.yaml
opencoat concern show my-concern
```

### Joinpoint ids (the ones you'll actually use)

Run `opencoat inspect joinpoints` for the full list. The ones that
ship from the OpenClaw scaffold's `DEFAULT_EVENT_NAMES` out of the
box:

| joinpoint | fires when… |
| --- | --- |
| `runtime_start` | host calls `bootstrap_opencoat.install()` (i.e. once per session) |
| `before_memory_write` | host is about to write to its memory store |
| `before_response` | host has built a response and is about to emit it |

Add `before_tool_call` (and a matching host-side subscription to
`agent.before_tool_call`) to enable `TOOL_GUARD` advice — the
cookbook block at the bottom of this file walks through that.

### Advice types

| type | typical `level` | typical `mode` |
| --- | --- | --- |
| `PROMPT_PREFIX` | `PROMPT_LEVEL` | `INSERT` |
| `RESPONSE_REQUIREMENT` | `PROMPT_LEVEL` or `OUTPUT_LEVEL` | `INSERT` |
| `TOOL_GUARD` | `TOOL_LEVEL` | `BLOCK` |
| `MEMORY_WRITE_GUARD` | `MEMORY_LEVEL` | `ANNOTATE` |
| `CONTEXT_NOTE` | `PROMPT_LEVEL` | `INSERT` |

`opencoat inspect joinpoints` and the v0.1 design doc §13 list the
authoritative `target` paths for each level (e.g. `runtime_prompt
.active_concerns`, `tool_call.arguments`, `memory_write.policy_note`,
`response.body.prefix`).

---

## The 3 demo concerns (ship via `opencoat concern import --demo`)

### `demo-prompt-prefix` — runtime banner in system prompt

```yaml
id: demo-prompt-prefix
name: Demo — runtime banner in system prompt
description: |
  Inserts a small marker so you can confirm at a glance that
  OpenCOAT-managed concerns reached the system prompt.
pointcut:
  joinpoints: [runtime_start]
advice:
  type: RESPONSE_REQUIREMENT
  content: "Begin every response with `[OpenCOAT demo active]`."
weaving_policy:
  mode: INSERT
  level: PROMPT_LEVEL
  target: runtime_prompt.active_concerns
  priority: 0.5
```

Lights up immediately — `runtime_start` is the first joinpoint the
OpenClaw scaffold emits.

### `demo-tool-block` — refuse `rm -rf`

```yaml
id: demo-tool-block
name: Demo — block destructive shell commands
description: |
  Refuses any tool call whose arguments mention "rm -rf".
  Demonstrates the BLOCK weaving mode against
  tool_call.arguments (M5 tool_guard interpreter).
pointcut:
  joinpoints: [before_tool_call]
  match:
    any_keywords: ["rm -rf", "rm  -rf"]
advice:
  type: TOOL_GUARD
  content: |
    Refusing destructive shell command — `rm -rf` is blocked by
    demo-tool-block.
weaving_policy:
  mode: BLOCK
  level: TOOL_LEVEL
  target: tool_call.arguments
  priority: 0.9
```

Requires the host to subscribe to `agent.before_tool_call` (see
cookbook at the bottom).

### `demo-memory-tag` — annotate every memory write

```yaml
id: demo-memory-tag
name: Demo — annotate every memory write
description: |
  Adds a lightweight policy note to every memory write. Pairs with
  OpenClawMemoryBridge to mirror the activation into the DCN.
pointcut:
  joinpoints: [before_memory_write]
advice:
  type: MEMORY_WRITE_GUARD
  content: "memory.policy=demo-memory-tag: write annotated by demo concern."
weaving_policy:
  mode: ANNOTATE
  level: MEMORY_LEVEL
  target: memory_write.policy_note
  priority: 0.4
```

Every annotated write also shows up in
`opencoat dcn activation-log --concern-id demo-memory-tag`.

---

## Starter recipes (copy + tweak)

### Recipe 1 — refuse network egress to a denylist

```yaml
id: net-deny
pointcut:
  joinpoints: [before_tool_call]
  match:
    any_keywords: ["http://169.254.", "http://metadata.google.internal", "http://10.", "file://"]
advice:
  type: TOOL_GUARD
  content: "Blocking egress to disallowed network target."
weaving_policy: { mode: BLOCK, level: TOOL_LEVEL, target: tool_call.arguments, priority: 0.95 }
```

### Recipe 2 — require a citation in every long answer

```yaml
id: must-cite
pointcut:
  joinpoints: [before_response]
advice:
  type: RESPONSE_REQUIREMENT
  content: |
    If the response is longer than three sentences, include at least
    one inline citation in the form [source: URL or title].
weaving_policy: { mode: INSERT, level: OUTPUT_LEVEL, target: response.body.prefix, priority: 0.6 }
```

### Recipe 3 — tag every memory write with the active task

```yaml
id: memory-task-tag
pointcut:
  joinpoints: [before_memory_write]
advice:
  type: MEMORY_WRITE_GUARD
  content: "memory.task={{runtime.active_task | default('unscoped')}}"
weaving_policy: { mode: ANNOTATE, level: MEMORY_LEVEL, target: memory_write.policy_note, priority: 0.3 }
```

### Recipe 4 — soft system prompt for a tone shift

```yaml
id: tone-precise
pointcut:
  joinpoints: [runtime_start]
advice:
  type: PROMPT_PREFIX
  content: |
    Reply with precise, concrete language. Skip filler phrases like
    "It is worth noting that" or "In conclusion".
weaving_policy: { mode: INSERT, level: PROMPT_LEVEL, target: runtime_prompt.active_concerns, priority: 0.4 }
```

Save any of the above to `concerns.yaml` (one concern as a mapping,
or several as a list), then:

```bash
opencoat concern import ./concerns.yaml
opencoat concern list --tag demo
```

---

## Cookbook — wiring `before_tool_call` on the OpenClaw scaffold

The OpenClaw scaffold's `DEFAULT_EVENT_NAMES` covers `agent.started`,
`agent.user_message`, and `agent.memory_write` but **not**
`agent.before_tool_call` — pass an extended event list to `install()`
and the `TOOL_GUARD` recipes above start firing:

```python
# wherever your host agent starts up
from opencoat_plugin.bootstrap_opencoat import install

installed = install(
    your_openclaw_host,
    event_names=(
        "agent.started",
        "agent.user_message",
        "agent.before_tool_call",   # add this row
        "agent.memory_write",
    ),
    # daemon_url=...               # defaults to OPENCOAT_DAEMON_URL or http://127.0.0.1:7878
)
try:
    your_openclaw_host.run()
finally:
    installed.uninstall()
```

`install()` builds an HTTP-backed `Client` for the daemon you started
with `opencoat runtime up`, wraps it in a `RuntimeLike` proxy, and
hands it to the openclaw adapter. Activations land in the daemon's
DCN store, so `opencoat dcn activation-log --concern-id <id>` sees
them straight away.

If you'd rather run the runtime in-process (no daemon, memory stores,
state lives for the host's lifetime) — typical for unit tests — swap
`install` for `install_in_process` (returns `(runtime, installed)`
so you can introspect the local stores). The `event_names` argument
is identical.
