# opencoat-skill

The 5-minute install skill for [OpenCOAT](https://github.com/HyperdustLabs/OpenCOAT) —
the Open Concern-Oriented Agent Thinking Runtime.

Drop this skill into any agent that supports the agent-skill
convention (Cursor, Claude Code, Codex, …) and the agent will know
how to:

- install `opencoat-runtime` + `opencoat-runtime-host` (today
  straight from `main`, via three `pip install
  "git+https://github.com/HyperdustLabs/OpenCOAT.git#subdirectory=…"`
  lines for `opencoat-runtime-protocol`, `opencoat-runtime`, and
  `opencoat-runtime-host` — PyPI publication is pending, after which
  the block flips to `pipx install opencoat-runtime`),
- start a local daemon (zero-config HTTP on `127.0.0.1:7878`),
- import three dramatic demo concerns,
- emit joinpoints from any host (universal `Client` +
  `JoinpointEmitter` path) or scaffold an OpenClaw plugin
  (`opencoat plugin install openclaw`), and
- inspect the resulting Deep Concern Network.

## Layout

```text
opencoat-skill/
├── SKILL.md                       # main instructions (≤500 lines)
├── skill.json                     # manifest
├── inspection.md                  # read-only inspection commands
├── concerns.md                    # authoring patterns + recipe gallery
├── rules.md                       # safety rules the host must respect
├── LICENSE                        # Apache-2.0
└── .github/workflows/verify.yml   # CI: parse skill.json + link integrity
```

## Install (per-agent)

| Agent | Path |
| --- | --- |
| Cursor (personal) | `~/.cursor/skills/opencoat/` |
| Cursor (project) | `<repo>/.cursor/skills/opencoat/` |
| Claude Code | `~/.claude/skills/opencoat/` |
| Codex | `~/.codex/skills/opencoat/` |

Either clone this repo into the path above, or use your agent's
skill installer pointing at this repo URL.

## Versioning

This skill tracks the major version of `opencoat-runtime` (which ships
the daemon + the `opencoat` CLI in one wheel as of 0.1.0). See
[`skill.json`](skill.json) `compatible_with` for the exact minimum
versions of each upstream package.

## Issues

- Skill content / phrasing / wiring → file here.
- Upstream runtime bugs → file at
  <https://github.com/HyperdustLabs/OpenCOAT/issues>.

## License

Apache-2.0. See [LICENSE](LICENSE).
