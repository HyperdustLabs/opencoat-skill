---
name: opencoat
description: >-
  Install and use OpenCOAT — the Open Concern-Oriented Agent Thinking Runtime —
  to add concern-driven joinpoint / pointcut / advice / weaving on top of a host
  agent (Cursor, Claude Code, OpenClaw, LangGraph, custom). Use when the user
  wants reusable safety rails (block destructive shell calls), prompt-level
  policies, memory-write annotations, or a Deep Concern Network they can
  inspect and replay across sessions.
---

# OpenCOAT — Concern-Oriented Agent Thinking Runtime

OpenCOAT runs alongside any host agent and rewrites its joinpoints
(`runtime_start`, `before_tool_call`, `before_memory_write`, …) using
**Concerns** — first-class units that carry a pointcut, an advice, and a
weaving policy. The result is a small, observable, replayable layer that
sits between the host's prompt/tool loop and the model.

This skill is the "5-minute install" that makes the runtime visible:

1. install the CLI,
2. scaffold a host plugin,
3. start a local daemon,
4. import the 3 dramatic demo concerns,
5. observe them firing on the demo host,
6. inspect the Deep Concern Network (DCN).

> Source repo: <https://github.com/HyperdustLabs/OpenCOAT>
> Skill repo:  <https://github.com/HyperdustLabs/opencoat-skill>

---

## Quick start (≤5 min)

Copy this checklist and walk through it top-to-bottom:

```text
- [ ] Step 1: install the CLI
- [ ] Step 2: scaffold the OpenClaw host plugin
- [ ] Step 3: start the daemon
- [ ] Step 4: import the 3 demo concerns
- [ ] Step 5: trigger them from the demo host
- [ ] Step 6: inspect the DCN
- [ ] Step 7: tear down
```

### Step 1 — install

```bash
pipx install opencoat-runtime-cli
opencoat --version
```

`pipx` keeps the CLI in its own venv so it never collides with the
host agent's Python. If `pipx` is not available, fall back to
`pip install --user opencoat-runtime-cli`.

### Step 2 — scaffold a host plugin

```bash
opencoat plugin install openclaw --out ./opencoat_plugin
```

Generates four lint-clean files in `./opencoat_plugin/`:

| file | role |
| --- | --- |
| `__init__.py` | makes the directory a package |
| `bootstrap_opencoat.py` | call once at host startup to register concerns + adapter |
| `host_adapter.py` | maps host events → OpenCOAT joinpoints |
| `concerns.py` | three starter concerns (edit freely) |

For a non-OpenClaw host, swap `openclaw` for `custom` — the same four
files, with the adapter stubbed for you to fill in.

### Step 3 — start the daemon

The CLI ships a sensible default config; you only need a PID file
location. A throwaway dir works for local exploration:

```bash
mkdir -p .opencoat
opencoat runtime up --pid-file .opencoat/opencoat.pid
opencoat runtime status --pid-file .opencoat/opencoat.pid
# expect: endpoint=http://127.0.0.1:7878/rpc · pid=<NNNN> · state=running
```

`up` double-forks the daemon so it survives this shell. Logs go to
stderr until you wire `--log-file`.

### Step 4 — import the 3 demo concerns

```bash
opencoat concern import --demo
opencoat concern list
```

You should see exactly three rows:

```text
demo-prompt-prefix   active   Demo — runtime banner in system prompt
demo-tool-block      active   Demo — block destructive shell commands
demo-memory-tag      active   Demo — annotate every memory write
```

What each one does, and where it fires, is documented in
[concerns.md](concerns.md).

### Step 5 — trigger them

If you generated the OpenClaw scaffold in Step 2, run the bootstrap
once from the host process:

```python
# in the host agent's startup code
from opencoat_plugin.bootstrap_opencoat import install
install()
```

Now drive the host agent normally. The three demo concerns light up
on the joinpoints listed in [concerns.md](concerns.md) — the marker
`[OpenCOAT demo active]` should appear at the start of every reply,
`rm -rf` shell calls should be refused, and every memory write should
carry a `memory.policy=demo-memory-tag` annotation.

### Step 6 — inspect

See [inspection.md](inspection.md) for the full surface. The two most
useful commands while the demos are firing:

```bash
opencoat concern list --lifecycle-state active
opencoat dcn activation-log --limit 20
```

For a graph view:

```bash
opencoat dcn export --format dot -o dcn.dot
dot -Tsvg dcn.dot -o dcn.svg && open dcn.svg
```

### Step 7 — tear down

```bash
opencoat runtime down --pid-file .opencoat/opencoat.pid
```

The PID file is unlinked on a clean exit; if the daemon was
`SIGKILL`'d, delete it manually.

---

## What "Concern" means here

A Concern is the only first-class unit OpenCOAT understands. Every
concern carries:

| field | role |
| --- | --- |
| `id` / `name` / `description` | identity + human label |
| `pointcut` | which joinpoints this concern listens on (+ optional keyword / vector match) |
| `advice` | what to inject when the pointcut fires (`PROMPT_PREFIX`, `TOOL_GUARD`, `MEMORY_WRITE_GUARD`, `RESPONSE_REQUIREMENT`, …) |
| `weaving_policy` | where in the host's COPR to weave (`PROMPT_LEVEL` / `TOOL_LEVEL` / `MEMORY_LEVEL` / `OUTPUT_LEVEL`) and how (`INSERT` / `BLOCK` / `ANNOTATE` / `REPLACE`) |
| `lifecycle_state` | `active` / `pending` / `archived` (driven by the runtime, not by hand) |

Authoring patterns and a recipe gallery live in [concerns.md](concerns.md);
safety rules around `TOOL_GUARD` and `MEMORY_WRITE_GUARD` live in
[rules.md](rules.md).

---

## When to apply this skill

Use this skill when **any** of these are true:

- The user asks to "install OpenCOAT" / "set up the OpenCOAT runtime"
  / "wire concerns into my agent".
- The user wants a quick reproducible demo of joinpoint / pointcut /
  advice / weaving on top of an existing host agent.
- The user references a `concern.upsert` failure, a missing
  `bootstrap_opencoat.install()` call, or a daemon that won't start
  on `127.0.0.1:7878`.
- The user asks how to add their own concern, edit a pointcut, or
  visualise the Deep Concern Network.

Do **not** use this skill for:

- Generic "agent design" or "prompt engineering" questions unrelated
  to OpenCOAT.
- LLM provider setup (`OPENAI_API_KEY` etc.) — that's the host
  agent's responsibility, not OpenCOAT's.
- Issues in the upstream `opencoat-runtime-*` Python packages
  themselves — file those at
  <https://github.com/HyperdustLabs/OpenCOAT/issues>.

---

## Compatibility & versions

This skill tracks `opencoat-runtime-cli` major. Today:

| component | min supported |
| --- | --- |
| `opencoat-runtime-cli` | `0.0.1` |
| `opencoat-runtime-daemon` | `0.0.1` |
| `opencoat-runtime-host-plugins[openclaw]` | `0.0.1` (M5) |

The skill does **not** install the upstream packages with pinned
versions on purpose — `pipx install opencoat-runtime-cli` always
picks the latest published wheel, and the CLI handles workspace
discovery itself.

---

## Troubleshooting (one-liners)

| symptom | likely fix |
| --- | --- |
| `opencoat runtime up` hangs | port 7878 in use → pass `--port 17890` (or another) and re-run `status` with the same flag |
| `concern.upsert` returns `ValidationError` | concern JSON missing `pointcut.joinpoints` or unknown `AdviceType` — see [concerns.md](concerns.md) |
| `bootstrap_opencoat.install()` does nothing visible | host did not subscribe to `agent.before_tool_call` — see the cookbook block at the bottom of [concerns.md](concerns.md) |
| daemon refuses to start because PID file exists | stale PID → `rm .opencoat/opencoat.pid && opencoat runtime up …` |

Anything else: `opencoat inspect joinpoints` and
`opencoat inspect pointcuts` are dependency-free and confirm the
catalogs the runtime is actually using.

---

## Related files in this skill

- [inspection.md](inspection.md) — every read-only command (`concern`,
  `dcn`, `inspect`, `replay`).
- [concerns.md](concerns.md) — authoring patterns + a recipe gallery
  (the 3 demo concerns + 4 useful starters).
- [rules.md](rules.md) — safety rules the host agent must respect
  when OpenCOAT injects `TOOL_GUARD` / `MEMORY_WRITE_GUARD` advice.
