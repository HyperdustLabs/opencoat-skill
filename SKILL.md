---
name: opencoat
description: >-
  Hosts send one line (URL + must use shell / run Quick start). Full contract in SKILL.md.
  The skill defines the full agent contract (run Quick start on the user's machine,
  daemon + autostart, LLM checks without keys in chat, demo). Also: concern-driven
  joinpoints / pointcuts / advice / weaving on Cursor, Claude Code, OpenClaw, LangGraph, custom.
---

## Canonical host prompt (copy verbatim)

The human may send **only** this block to their host agent — no other
instructions are required; everything else is in this document and linked
files:

```text
Read https://www.opencoat.ai/SKILL.md to install and use OpenCOAT.
You MUST use shell and run every Quick start step; do not only print commands.
If OpenClaw is installed, auto-install the gateway bridge (Step 5) — resolve the
monorepo path yourself; do not ask me for it or wait for "go ahead".
```

(URL-only also binds via **Agent contract** below, but hosts that only read the
URL often skip execution — keep the shell line in the paste.)

### Agent contract

If that sentence is your **sole** instruction, **fetch** this URL (and pull
companion files when this SKILL tells you to), then **run** OpenCOAT on the
user's machine per **Quick start** until the daemon is healthy and the demo
path succeeds — not a passive read-through. Security and LLM rules (including
**never asking for API keys in chat**) live here and in [rules.md](rules.md);
the human does not need to paste them again.

**Execute, do not delegate back to the human.** If the skill URL is the only
input (no copied skill files on disk yet), you must still **run** OpenCOAT on
the user's machine — not summarize the install guide:

```text
Read https://www.opencoat.ai/SKILL.md, then execute every Quick start step on
the user's machine until OpenCOAT is actually running: pipx install +
pipx inject opencoat-runtime-host (or pip install in a venv), opencoat runtime
up + opencoat service install, LLM check (runtime.llm_info / opencoat configure
llm in a local terminal — never collect API keys in chat), opencoat concern
import --demo, and opencoat demo to verify. If `openclaw` is on PATH or
`~/.openclaw` exists, run **Step 5 immediately** (before claiming done): shell
`test -d ~/OpenCOAT/integrations/openclaw-opencoat-bridge` and other paths in
**Resolve OpenCOAT monorepo** — never ask the human for the repo path first.
Then `install-local.sh`, `openclaw plugins install -l`, `openclaw gateway restart`,
grep `opencoat-bridge` in gateway.log. Only then **Step 6** inspection
(`opencoat dcn activation-log --limit 20`, `opencoat concern list`).
Do not stop after skimming the checklist. Do not only paste bash blocks for the
user to run unless shell/exec is impossible — then say so explicitly.
**Forbidden:** “If you want, I can run inspection / proceed to Step 5”; “tell me where
the repo is”; “say go ahead”; reporting “Steps 1–4 done” while Step 5 is skipped on
an OpenClaw host.
```

### Definition of done

| Host | Done only when |
| --- | --- |
| Any | Steps 1–4 succeed; Step 6 inspection ran; daemon + autostart healthy |
| OpenClaw (`openclaw` or `~/.openclaw`) | **Also** Step 5: `@hyperdustlabs/opencoat-bridge` loaded + `[opencoat-bridge] registered` in gateway.log |

PyPI runtime alone is **not** “full OpenCOAT on OpenClaw”. Do not mark the install
complete until the row for your host is satisfied.

### Anti-pattern (do not repeat)

Bad (seen in real OpenClaw sessions): finish pipx + daemon + demo, offer “Step 5
if you give me the monorepo path”, then after `~/OpenCOAT` is found still say “say
go ahead and I’ll install”. **Why it’s wrong:** Step 5 is part of install on
OpenClaw; `~/OpenCOAT` (and legacy `~/COAT`) are in the resolver list — run
`ls ~/OpenCOAT/integrations/...` or `ls ~/COAT/integrations/...`
yourself before asking anyone anything.

**OpenClaw gateway note:** Weaving in chat (`<OpenCOAT>` blocks) comes from the
**bridge plugin** + daemon (Step 5). That is not the same as completing Quick
start. If `opencoat --version` or `runtime status` would fail, you have not
finished the contract — use shell and run the steps. When OpenClaw is present,
**install the bridge before declaring done**; do not wait for the human to supply
a repo path (see **Resolve OpenCOAT monorepo** below).

# OpenCOAT — Concern-Oriented Agent Thinking Runtime

OpenCOAT runs alongside any host agent and rewrites its joinpoints
(`runtime_start`, `before_tool_call`, `before_memory_write`, …) using
**Concerns** — first-class units that carry a pointcut, an advice, and a
weaving policy. The result is a small, observable, replayable layer that
sits between the host's prompt/tool loop and the model.

This skill is the "5-minute install" that makes the runtime visible:

1. install the CLI + host SDK (PyPI),
2. start a local daemon + OS autostart,
3. import the 3 demo concerns,
4. run `opencoat demo`,
5. install the OpenClaw gateway bridge from the **OpenCOAT monorepo** (when OpenClaw is present),
6. inspect the DCN and leave the daemon running.

> **Two repos, two roles**
>
> | Repo | What it is |
> | --- | --- |
> | [HyperdustLabs/opencoat-skill](https://github.com/HyperdustLabs/opencoat-skill) | This skill — served at <https://www.opencoat.ai/SKILL.md> |
> | [HyperdustLabs/OpenCOAT](https://github.com/HyperdustLabs/OpenCOAT) | Runtime + **OpenClaw TS bridge** at `integrations/openclaw-opencoat-bridge/` — clone to **`~/OpenCOAT`** |
>
> The bridge does **not** need its own GitHub repo; it ships inside OpenCOAT. PyPI wheels do **not** include the TS plugin.

> Skill CDN tracks PyPI `opencoat-runtime` **0.1.5+**.

---

**Editing the OpenCOAT git repo?** See [contributing.md](contributing.md) (PR-only workflow).

---

## Self-install (for AI agents)

Drop the six skill files into whatever your host calls its skills directory
(Cursor: `~/.cursor/skills/opencoat/`, Claude Code: `~/.claude/skills/opencoat/`,
etc.) and reload:

```bash
DEST="${SKILLS_DIR:-$HOME/.cursor/skills}/opencoat"
mkdir -p "$DEST"
for f in SKILL.md concerns.md inspection.md rules.md contributing.md compatibility.md openclaw-bridge.md troubleshooting.md skill.json bootstrap_daemon.sh; do
  curl -fsSL "https://www.opencoat.ai/$f" -o "$DEST/$f"
done
chmod +x "$DEST/bootstrap_daemon.sh" 2>/dev/null || true
echo "installed opencoat skill → $DEST"
```

URLs proxy this repo's `main` ([`DEPLOY.md`](DEPLOY.md)). After files are on disk,
execute **Quick start** on the user's machine — the canonical one-line prompt does
not change that obligation.

---

## Quick start (≤5 min)

Copy this checklist and walk through it top-to-bottom:

```text
- [ ] Step 1: install the CLI + host SDK (pipx)
- [ ] Step 2: start the daemon + OS autostart (+ verify LLM — no keys in chat)
- [ ] Step 3: import the 3 demo concerns
- [ ] Step 4: opencoat demo
- [ ] Step 5 (**required** if `openclaw` or `~/.openclaw`): OpenClaw gateway bridge
- [ ] Step 6: inspect the DCN and leave the daemon running
```

**Local paths:** monorepo at `~/OpenCOAT` (`OPENCOAT_ROOT` / resolver below). Runtime
state (sqlite, PID, config) stays in `~/.opencoat/` — not inside the git tree.

**Agent: run the checklist in order.** Do not offer “I can run inspection next” or
“proceed to Step 5 if you have the repo” — execute Steps 5 and 6 yourself when OpenClaw
is installed.

### Step 1 — install

OpenCOAT ships as three packages out of the monorepo at
<https://github.com/HyperdustLabs/OpenCOAT>:

| package | what it is |
| --- | --- |
| `opencoat-runtime-protocol` | wire envelopes + JSON Schemas (pulled in transitively) |
| `opencoat-runtime` | runtime core + daemon + `opencoat` CLI |
| `opencoat-runtime-host` | host SDK (`Client`, `JoinpointEmitter`) + OpenClaw adapter |

Both packages are on PyPI. The recommended path is `pipx` — it puts
the `opencoat` CLI on `PATH` without polluting the host agent's
Python, and `pipx inject` adds the host SDK to the same isolated env
so `opencoat demo`'s lazy imports succeed:

```bash
pipx install opencoat-runtime
pipx inject  opencoat-runtime opencoat-runtime-host

opencoat --version    # → 0.1.x
```

`opencoat-runtime-protocol` comes along transitively — you don't need
to name it. `pipx inject` is what wires `opencoat-runtime-host` into
the CLI's env; if you skip it, `opencoat demo` will refuse to start
with a `ModuleNotFoundError: opencoat_runtime_host_sdk`.

### Step 1b (alternative) — install into a regular venv

If your host agent is itself a Python project that wants to `import
opencoat_runtime_host_sdk` from its own code (writing a custom host,
embedding the runtime in-process, etc.), drop the pipx layer and use
a plain venv:

```bash
python3 -m venv .opencoat/venv
source .opencoat/venv/bin/activate
pip install opencoat-runtime opencoat-runtime-host

opencoat --version    # → 0.1.x
```

The CLI works identically; the difference is just *where* the SDK
ends up on `sys.path`.

### Step 1c — install from the workspace monorepo (when present)

When the agent's **working directory** (or `OPENCOAT_REPO`) is an OpenCOAT git
checkout, prefer an **editable** install from that tree instead of only PyPI —
this is how local dev and OpenClaw sessions on an OpenCOAT workspace are meant to work:

```bash
# Resolve OPENCOAT_ROOT — see full resolver below; minimal form:
OPENCOAT_ROOT="${OPENCOAT_REPO:-}"
if [[ -z "$OPENCOAT_ROOT" ]]; then
  for d in "$(pwd)" "$(pwd)/OpenCOAT" "$HOME/OpenCOAT" "$HOME/COAT"; do
    [[ -f "$d/packages/opencoat-runtime/pyproject.toml" ]] && OPENCOAT_ROOT="$(cd "$d" && pwd)" && break
  done
fi
test -n "$OPENCOAT_ROOT" || { echo "no monorepo — use Step 1 pipx PyPI"; exit 0; }

pipx install -e "$OPENCOAT_ROOT/packages/opencoat-runtime"
pipx inject  opencoat-runtime -e "$OPENCOAT_ROOT/packages/opencoat-runtime-host"
opencoat --version
```

If there is no checkout but OpenClaw needs the bridge (Step 5), clone into the
**workspace** (not only `$HOME`):

```bash
WORK="${OPENCOAT_WORKSPACE:-$(pwd)}"
OPENCOAT_ROOT="$WORK/.opencoat/OpenCOAT"
if [[ ! -f "$OPENCOAT_ROOT/packages/opencoat-runtime/pyproject.toml" ]]; then
  mkdir -p "$(dirname "$OPENCOAT_ROOT")"
  git clone --depth 1 https://github.com/HyperdustLabs/OpenCOAT.git "$OPENCOAT_ROOT"
fi
# then Step 1c pipx -e and Step 5 bridge install from $OPENCOAT_ROOT
```

### Step 2 — start the daemon + OS autostart (long-running, persistent)

The bundled daemon defaults (current `opencoat-runtime`) enable HTTP JSON-RPC
on `127.0.0.1:7878/rpc` and **sqlite** persistence under `~/.opencoat/`. After
Step 1, bring the daemon up and register a user-level autostart unit so it
survives terminal closes **and** host-agent (Cursor / OpenClaw / …) restarts.

**Recommended one-shot (copy-paste):**

```bash
mkdir -p ~/.opencoat
opencoat runtime up
opencoat runtime status
# Free the default listen port before the OS service starts its own daemon
# (otherwise Linux `systemctl restart` can collide with the ad-hoc `runtime up`).
opencoat runtime down || true
opencoat service install    # macOS LaunchAgent · Linux systemd --user
opencoat service status
```

Or run the skill helper:

```bash
bash "${SKILLS_DIR:-$HOME/.cursor/skills}/opencoat/bootstrap_daemon.sh"
```

Default PID file is `~/.opencoat/opencoat.pid` — you only need `--pid-file` if
you override it. `runtime up` double-forks so the process is not tied to this
shell.

**PyPI / CLI drift:** `opencoat service …` (LaunchAgent / systemd autostart)
ships in **`opencoat-runtime` 0.1.2+**. If `opencoat service` is missing
(unknown subcommand / parser error), upgrade the CLI env:

```bash
pip install -U 'opencoat-runtime>=0.1.4' 'opencoat-runtime-host>=0.1.4'
# pipx:
pipx upgrade opencoat-runtime && pipx inject opencoat-runtime opencoat-runtime-host
```

Until then, keep using `opencoat runtime up` + `opencoat runtime status` and
start the daemon manually after reboot (no OS-level autostart).

**Custom sqlite paths or HTTP bind:** run `opencoat configure daemon` (and
optionally `opencoat configure llm`), then:

```bash
opencoat runtime up --config ~/.opencoat/daemon.yaml
opencoat runtime status
opencoat runtime down || true
opencoat service install --config ~/.opencoat/daemon.yaml
```

Pass `--port` / `--host` to `runtime up` / `status` if 7878 is busy.

**Hermetic CI / pytest / no autostart:** do **not** run `opencoat service
install` inside automated tests. For in-process tests that call
`load_config()`, set `OPENCOAT_TEST_MEMORY_STORES=1` so stores stay in RAM and
never touch `~/.opencoat/*.sqlite`. For a disposable daemon without OS
service registration:

```bash
mkdir -p ~/.opencoat
opencoat runtime up
opencoat runtime status
```

### LLM credentials check (**agents: never ask for keys in chat**)

After Step 2, confirm the daemon is not stuck on a **stub** LLM (no real
provider credentials). **Do not** ask the user to paste API keys into this
chat — use a local terminal and `opencoat configure llm` instead.

The wizard's **env-file** mode writes `~/.opencoat/opencoat.env`; the daemon merges **allow-listed** LLM keys on startup (`runtime up` / `opencoat service`). `source` is optional (shell-only). Details: [inspection.md — LLM credentials](inspection.md#llm-credentials-no-keys-in-chat).

**Quick probe** (default JSON-RPC URL):

```bash
curl -sS -X POST http://127.0.0.1:7878/rpc \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"runtime.llm_info","params":{}}'
```

If `"real": false`, walk the human through the recipes in
[inspection.md — LLM credentials](inspection.md#llm-credentials-no-keys-in-chat)
(wizard, non-interactive shell-only path, and `opencoat service install` caveats).

**B.AI (optional):** `export BAI_API_KEY=…` then `opencoat configure llm` → provider
**b.ai**; daemon uses OpenAI-compatible `https://api.b.ai/v1`. For OpenClaw chat on
the same key, see
[OpenCOAT `docs/config/bai-llm.md` § OpenClaw + B.AI](https://github.com/HyperdustLabs/OpenCOAT/blob/main/docs/config/bai-llm.md#openclaw--bai).

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

### Step 4 — `opencoat demo`

```bash
opencoat demo
```

That's the whole step. The CLI subscribes a tiny in-script `FakeHost`
to the daemon via `install_hooks`, fires three events, and uses the
pickup API (`apply_to` / `guard_tool_call`) to fold the returned
advice back into the host's mutable state. **Run `opencoat demo` locally**
to see three BEFORE/AFTER scenes (prompt fold → tool guard → memory annotate).

```text
(opencoat demo output is long; run the command to see the full trace.)
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

### Step 5 — OpenClaw gateway bridge (**required when OpenClaw is present**)

When `openclaw` or `~/.openclaw` exists, run Step 5 **automatically** after Step 4.
Full resolver, install commands, and config: **[openclaw-bridge.md](openclaw-bridge.md)**.

### Step 6 — inspect and keep running

Run inspection immediately after Step 4 (and Step 5 when OpenClaw is present). Do not ask
the human whether to run inspection — paste the command output in your summary.

See [inspection.md](inspection.md) for the full surface. Minimum:

```bash
opencoat concern list --lifecycle-state active
opencoat dcn activation-log --limit 20
```

Sample activation log after `demo_host.py` (run locally for full rows):

```text
2026-05-13T08:39:35  demo-memory-tag   …  score=0.675
```

`opencoat runtime up` double-forks at Step 2 — **leave the daemon running** between
sessions. Stop/uninstall: [inspection.md](inspection.md).

---

## Concerns, scope, versions, troubleshooting

- Authoring: [concerns.md](concerns.md) · Safety: [rules.md](rules.md)
- PyPI versions: [compatibility.md](compatibility.md) (skill tracks **0.1.4+**; **0.1.5** when published)
- OpenCOAT repo edits: [contributing.md](contributing.md)
- Problems: [troubleshooting.md](troubleshooting.md)

Use this skill for install/demo/OpenClaw bridge/DCN inspection — not generic prompt
engineering or secrets in chat ([rules.md](rules.md)). Runtime bugs:
<https://github.com/HyperdustLabs/OpenCOAT/issues>.

---

## Related files in this skill

- [inspection.md](inspection.md) — `concern`, `dcn`, `inspect`, `replay`
- [openclaw-bridge.md](openclaw-bridge.md) — Step 5 install + `OPENCOAT_ROOT`
- [compatibility.md](compatibility.md) · [contributing.md](contributing.md) · [troubleshooting.md](troubleshooting.md)
