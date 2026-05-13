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

1. install the CLI + host SDK,
2. start a local daemon (zero-config HTTP on `127.0.0.1:7878`),
3. import the 3 dramatic demo concerns,
4. emit a few joinpoints and watch them light up activations,
5. inspect the Deep Concern Network (DCN),
6. tear down.

> Source repo: <https://github.com/HyperdustLabs/OpenCOAT>  
> Skill repo:  <https://github.com/HyperdustLabs/opencoat-skill>

---

## Quick start (≤5 min)

Copy this checklist and walk through it top-to-bottom:

```text
- [ ] Step 1: install the CLI + host SDK
- [ ] Step 2: start the daemon
- [ ] Step 3: import the 3 demo concerns
- [ ] Step 4a: `opencoat demo` — see concerns change host behavior
- [ ] Step 4b (optional): wire an OpenClaw host plugin
- [ ] Step 5: inspect the DCN
- [ ] Step 6: tear down
```

### Step 1 — install

OpenCOAT ships as three packages out of the monorepo at
<https://github.com/HyperdustLabs/OpenCOAT>:

| package | what it is |
| --- | --- |
| `opencoat-runtime-protocol` | wire envelopes + JSON Schemas (pulled in transitively) |
| `opencoat-runtime` | runtime core + daemon + `opencoat` CLI |
| `opencoat-runtime-host` | host SDK (`Client`, `JoinpointEmitter`) + OpenClaw adapter |

PyPI publication is pending; install straight from GitHub today. A
throwaway venv (Python 3.12+) keeps the install isolated from the host
agent's Python. Until PyPI lands, all three sibling packages must be
named explicitly — pip resolves transitive deps from PyPI by default,
so the protocol package has to be on disk before `opencoat-runtime`
and `opencoat-runtime-host` can find it:

```bash
python3 -m venv .opencoat/venv
source .opencoat/venv/bin/activate

REPO="git+https://github.com/HyperdustLabs/OpenCOAT.git"
pip install \
  "$REPO#subdirectory=packages/opencoat-runtime-protocol" \
  "$REPO#subdirectory=packages/opencoat-runtime" \
  "$REPO#subdirectory=packages/opencoat-runtime-host"

opencoat --version    # → 0.1.x
```

Once PyPI lands you'll be able to swap that block for
`pipx install opencoat-runtime` + `pipx inject opencoat-runtime
opencoat-runtime-host` (the protocol package comes along transitively).
This skill will be re-tagged when that happens.

### Step 2 — start the daemon

The daemon's bundled default config ships `ipc.http.enabled: true` on
`127.0.0.1:7878`, so the next line is the full setup:

```bash
mkdir -p .opencoat
opencoat runtime up --pid-file .opencoat/opencoat.pid
opencoat runtime status --pid-file .opencoat/opencoat.pid
# expect: endpoint=http://127.0.0.1:7878/rpc · pid=<NNNN> · state=running
```

`up` double-forks the daemon so it survives this shell. Logs go to
stderr until you wire `--log-file`. Pass `--port 17890` (or any free
port) if 7878 is already in use.

### Step 3 — import the 3 demo concerns

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

### Step 4a — see concerns change host behavior (one line)

```bash
opencoat demo
```

That's the whole step. The CLI subscribes a tiny in-script `FakeHost`
to the daemon via `install_hooks`, fires three events, and uses the
pickup API (`apply_to` / `guard_tool_call`) to fold the returned
advice back into the host's mutable state. Three scenes print
**BEFORE / AFTER** so concerns visibly change the host context:

```text
== OpenCOAT demo — daemon @ http://127.0.0.1:7878/rpc ==

[1/3] PROMPT FOLDING — concern: demo-prompt-prefix
  fire event : agent.started
  prompt slot: runtime_prompt.active_concerns
  BEFORE     : ""
  AFTER      : Begin every response with `[OpenCOAT demo active]`.
  → apply_to() folded demo-prompt-prefix into the prompt slot.

[2/3] TOOL GUARD — concern: demo-tool-block
  fire event : agent.before_tool (payload includes 'rm -rf')
  tool call  : shell.exec rm -rf /tmp/scratch
  outcome    : BLOCKED
  reason     : Refusing destructive shell command — `rm -rf` is blocked by demo-tool-block.
  → guard_tool_call() returned blocked=True. Host should refuse.

[3/3] MEMORY NOTE — concern: demo-memory-tag
  fire event : agent.memory_write
  memory slot: memory_write.policy_note
  BEFORE     : ""
  AFTER      : memory.policy=demo-memory-tag: write annotated by demo concern.
  → apply_to() annotated memory_write.policy_note.

✓ All three scenes produced visible host-context changes.
```

If the daemon isn't running yet (or you don't want to bother seeding
it), `opencoat demo --in-proc` builds an in-process runtime and
seeds the three demo concerns automatically — same three scenes, no
`opencoat runtime up` / `concern import` required.

Want to learn the underlying pattern? `opencoat demo --script-out
demo_host.py` writes the equivalent ~40-line Python file (the same
`install_hooks` → `apply_to` / `guard_tool_call` shape your own
host will use) to disk without running anything. That's the seed
for adapting the demo to a real host agent.

The two pickup points to remember:

- `installed.apply_to(context)` — fold every buffered advice row
  into a mutable host context (prompt slots, memory slots, …).
- `installed.guard_tool_call(call)` — decode `TOOL_GUARD` advice
  into a structured outcome you can branch on (`outcome.blocked` →
  refuse; `outcome.arguments` → dispatch with rewrites;
  `outcome.notes` → audit-only annotations).

### Step 4b — OpenClaw host plugin (optional)

If you're integrating OpenCOAT into a real OpenClaw-shaped host agent
(anything that exposes `subscribe(event_name, callback) -> unsubscribe`),
scaffold a plugin:

```bash
opencoat plugin install openclaw --out ./opencoat_plugin
```

Generates four lint-clean files in `./opencoat_plugin/`:

| file | role |
| --- | --- |
| `__init__.py` | makes the directory a package |
| `bootstrap_opencoat.py` | call once at host startup to register concerns + adapter |
| `host_adapter.py` | maps host events → OpenCOAT joinpoints (you only edit this) |
| `concerns.py` | three starter concerns (edit freely) |

Then, from your host's startup code:

```python
from opencoat_plugin.bootstrap_opencoat import install

installed = install(your_openclaw_host)   # default: daemon at $OPENCOAT_DAEMON_URL
try:
    while turn := your_openclaw_host.next_turn():
        # 1. events flow into the daemon automatically through the
        #    install_hooks subscriptions; concerns activate inside it.
        turn.run_until_prompt()

        # 2. fold every active advice row into the prompt context
        #    BEFORE calling the LLM. Empty buffer → identity.
        turn.prompt_ctx = installed.apply_to(turn.prompt_ctx)

        # 3. before dispatching each pending tool call, ask OpenCOAT
        #    whether any TOOL_GUARD advice applies. None → default-allow.
        for call in turn.pending_tool_calls():
            outcome = installed.guard_tool_call(call)
            if outcome is not None and outcome.blocked:
                turn.refuse(call, reason=outcome.block_reason)
            elif outcome is not None:
                turn.dispatch(call["name"], outcome.arguments, notes=outcome.notes)
            else:
                turn.dispatch(call["name"], call["arguments"])
finally:
    installed.uninstall()
```

`install()` connects to the running daemon over HTTP (the same daemon
you started in Step 2), so concerns + DCN state are shared with
`opencoat concern …` / `opencoat dcn …`. The two pickup points
(`apply_to` / `guard_tool_call`) are where OpenCOAT's advice materialises
back into your host — without them you'll see activations in the DCN
log but no visible change to the agent's prompt or tool dispatch.

For a one-process unit test where you don't want a daemon, swap
`install()` for `install_in_process()` — same signature + pickup API,
plus a bundled `OpenCOATRuntime` is returned.

For a non-OpenClaw host, swap `openclaw` for `custom` — the same four
files, with the adapter and joinpoint mapping stubbed for you to fill
in, plus a `daemon_client()` helper that returns a ready-to-use
`Client`.

### Step 5 — inspect

See [inspection.md](inspection.md) for the full surface. The two most
useful commands while activations are flowing:

```bash
opencoat concern list --lifecycle-state active
opencoat dcn activation-log --limit 20
```

Sample activation log after `demo_host.py`:

```text
2026-05-13T08:39:35  demo-memory-tag   62515cdf-…  score=0.675
2026-05-13T08:39:35  demo-prompt-prefix 7a221311-…  score=0.500
```

For a graph view:

```bash
opencoat dcn export --format dot -o dcn.dot
dot -Tsvg dcn.dot -o dcn.svg && open dcn.svg
```

### Step 6 — tear down

```bash
opencoat runtime down --pid-file .opencoat/opencoat.pid
deactivate                                # leave the venv
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

This skill tracks `opencoat-runtime` major. Today:

| component | min supported | source |
| --- | --- | --- |
| `opencoat-runtime` | `0.1.0` | `git+…#subdirectory=packages/opencoat-runtime` |
| `opencoat-runtime-host` | `0.1.0` | `git+…#subdirectory=packages/opencoat-runtime-host` |
| `opencoat-runtime-protocol` | `0.1.0` | pulled transitively |

PyPI wheels are not published yet; this skill installs from `main` on
purpose so the demo always runs against the most recent stable surface.
When PyPI publication lands, the install lines flip to
`pipx install opencoat-runtime` + `pipx inject opencoat-runtime
opencoat-runtime-host` and the skill is re-tagged.

---

## Troubleshooting (one-liners)

| symptom | likely fix |
| --- | --- |
| `opencoat runtime up` hangs | port 7878 in use → pass `--port 17890` (or another) and re-run `status` with the same flag |
| `python demo_host.py` raises `ModuleNotFoundError: opencoat_runtime_host_sdk` | `pip install` of `opencoat-runtime-host` missing — see Step 1 |
| `Client.connect(…)` raises `HostTransportConnectionError` | daemon down or bound on another port; `opencoat runtime status` is the truth |
| `concern.upsert` returns `ValidationError` | concern JSON missing `pointcut.joinpoints` or unknown `AdviceType` — see [concerns.md](concerns.md) |
| `bootstrap_opencoat.install()` does nothing visible | host loop never calls `installed.apply_to(prompt_ctx)` / `installed.guard_tool_call(call)` — see Step 4b for the canonical loop and [concerns.md](concerns.md) for the cookbook |
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
