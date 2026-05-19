# OpenCOAT — OpenClaw gateway bridge (Step 5)

Required when `openclaw` is on `PATH` or `~/.openclaw` exists. Run **automatically** after Quick start Step 4 — do not ask the human for permission or repo path.

## Skill vs bridge

Gateway weaving (`<OpenCOAT>…</OpenCOAT>` in chat) comes from the **bridge plugin** under `~/.openclaw/extensions/`, not from copying this skill into `workspace/skills/opencoat/`.

## Resolve `OPENCOAT_ROOT`

Agents must **not** ask the human for the repo path. Resolve in shell (first hit wins):

```bash
OPENCOAT_ROOT="${OPENCOAT_REPO:-}"
if [[ -z "$OPENCOAT_ROOT" ]]; then
  for d in "$(pwd)" "$(pwd)/OpenCOAT" "$HOME/OpenCOAT" "$HOME/COAT" "${OPENCOAT_WORKSPACE:-$(pwd)}/.opencoat/OpenCOAT"; do
    [[ -f "$d/integrations/openclaw-opencoat-bridge/package.json" ]] && OPENCOAT_ROOT="$(cd "$d" && pwd)" && break
  done
fi
if [[ -z "$OPENCOAT_ROOT" ]] && { command -v openclaw >/dev/null || [[ -d "${HOME}/.openclaw" ]]; }; then
  WORK="${OPENCOAT_WORKSPACE:-$(pwd)}"
  OPENCOAT_ROOT="$WORK/.opencoat/OpenCOAT"
  if [[ ! -f "$OPENCOAT_ROOT/integrations/openclaw-opencoat-bridge/package.json" ]]; then
    mkdir -p "$(dirname "$OPENCOAT_ROOT")"
    git clone --depth 1 https://github.com/HyperdustLabs/OpenCOAT.git "$OPENCOAT_ROOT"
  fi
fi
echo "OPENCOAT_ROOT=${OPENCOAT_ROOT:-<unset>}"
```

## Install

Prerequisites: daemon up (Step 2), concerns in store, OpenClaw **≥ 2026.3.24**.

```bash
command -v openclaw >/dev/null || [[ -d "${HOME}/.openclaw" ]] || { echo "skip Step 5: no OpenClaw"; exit 0; }
: "${OPENCOAT_ROOT:?set OPENCOAT_ROOT first}"

BRIDGE="$OPENCOAT_ROOT/integrations/openclaw-opencoat-bridge"
bash "$BRIDGE/scripts/install-local.sh"
openclaw plugins install -l "$BRIDGE"
openclaw gateway restart
openclaw plugins list   # @hyperdustlabs/opencoat-bridge → loaded
grep opencoat-bridge ~/.openclaw/logs/gateway.log | tail -5
```

In `~/.openclaw/openclaw.json`: enable `@hyperdustlabs/opencoat-bridge`, `hooks.allowPromptInjection: true`, `config.daemonUrl`: `http://127.0.0.1:7878/rpc`. Plugin id uses a **slash**; extension folder is flat (`@hyperdustlabs-opencoat-bridge`).

Hooks (summary): `message_received`→`on_user_input`, `before_prompt_build`→`before_response`, `before_tool_call`, runtime observers for queue/task/`onAgentEvent`. Details: monorepo `integrations/openclaw-opencoat-bridge/README.md`.
