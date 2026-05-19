# OpenCOAT — PyPI compatibility (skill 0.6.1)

This skill tracks `opencoat-runtime` **0.1.x** (same major).

| Release | Highlights |
| --- | --- |
| **0.1.5** | M6 heartbeat workers; joinpoint ADR-0011; OpenClaw bridge 26 hooks + runtime observers |
| **0.1.4** | JoinpointDiscovery, AspectJ concerns, B.AI LLM, bridge verify |

**Install (PyPI):**

```bash
pipx install 'opencoat-runtime>=0.1.4'
pipx inject opencoat-runtime 'opencoat-runtime-host>=0.1.4'
# When 0.1.5 wheels are on PyPI, prefer:
pipx upgrade opencoat-runtime && pipx inject opencoat-runtime opencoat-runtime-host
```

| component | min in `skill.json` | notes |
| --- | --- | --- |
| `opencoat-runtime` | `>=0.1.4` | [PyPI](https://pypi.org/project/opencoat-runtime/) |
| `opencoat-runtime-host` | `>=0.1.4` | [PyPI](https://pypi.org/project/opencoat-runtime-host/) |
| `opencoat-runtime-protocol` | `>=0.1.4` | transitive |

OpenClaw gateway weaving uses the TS bridge in the
[OpenCOAT monorepo](https://github.com/HyperdustLabs/OpenCOAT/tree/main/integrations/openclaw-opencoat-bridge)
(Quick start Step 5), not the runtime wheel alone.
