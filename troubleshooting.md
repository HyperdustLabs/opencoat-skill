# OpenCOAT — troubleshooting (one-liners)

| symptom | likely fix |
| --- | --- |
| `opencoat runtime up` hangs | port 7878 in use → `--port 17890` and matching `status` |
| `opencoat: command not found` | `pipx ensurepath` or Step 1b venv |
| `opencoat demo` → `ModuleNotFoundError: opencoat_runtime_host_sdk` | `pipx inject opencoat-runtime opencoat-runtime-host` |
| `concern extract` → 0 candidates; `llm: stub-fallback` | `opencoat configure llm` in local terminal (never keys in chat); restart daemon |
| `HostTransportConnectionError` | `opencoat runtime status` — daemon down or wrong port |
| `concern.upsert` → `ValidationError` | see [concerns.md](concerns.md) |
| OpenClaw: no weave / plugin missing | Step 5: [openclaw-bridge.md](openclaw-bridge.md) |
| Blocks in chat but “no skill” | Weaving = bridge plugin, not skills dir |
| stale PID file | `rm ~/.opencoat/opencoat.pid && opencoat runtime up` |
| `service install` bind error (Linux) | `opencoat runtime down` then retry install |

Else: `opencoat inspect joinpoints` / `opencoat inspect pointcuts`.
