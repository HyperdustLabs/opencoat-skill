# opencoat-skill

The 5-minute install skill for [OpenCOAT](https://github.com/HyperdustLabs/OpenCOAT) —
the Open Concern-Oriented Agent Thinking Runtime.

Drop this skill into any agent that supports the agent-skill
convention (Cursor, Claude Code, Codex, …) and the agent will know
how to:

- install `opencoat-runtime` from
  [PyPI](https://pypi.org/project/opencoat-runtime/) — one
  `pipx install opencoat-runtime` puts the `opencoat` CLI on `PATH`,
  and `pipx inject opencoat-runtime opencoat-runtime-host` wires the
  host SDK into the same env so the demo's lazy imports succeed
  (`opencoat-runtime-protocol` comes along transitively),
- start a local daemon (zero-config HTTP on `127.0.0.1:7878`),
- import three dramatic demo concerns,
- close the full loop with one command — `opencoat demo` fires three
  events through `install_hooks` and uses the pickup API
  (`apply_to` / `guard_tool_call`) to fold the resulting advice
  back into a tiny in-script host. Three scenes print BEFORE /
  AFTER so users see concerns visibly change behavior on first
  read. `opencoat demo --in-proc` runs the same tour without a
  daemon (no setup), and `opencoat demo --script-out demo_host.py`
  dumps the equivalent Python for users who want a template to
  adapt for a real host. Scaffold an OpenClaw plugin with
  `opencoat plugin install openclaw` for the same loop pre-wired,
  then
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
