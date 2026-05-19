# OpenCOAT monorepo — git workflow (for agents)

When you change **[HyperdustLabs/OpenCOAT](https://github.com/HyperdustLabs/OpenCOAT)** (runtime, bridge, `docs/`) — not when you only run Quick start on a user's machine:

- **Never** `git push origin main`. Every change uses a **feature branch + PR** (CI + paper trail).
- Flow: `git switch -c feat/…` → edit → `./scripts/verify.sh` → commit →
  `git push -u origin HEAD` → `gh pr create` → squash-merge after checks green.
- Rules: [CONTRIBUTING.md](https://github.com/HyperdustLabs/OpenCOAT/blob/main/CONTRIBUTING.md)
  (§1–§3, §9), [AGENTS.md](https://github.com/HyperdustLabs/OpenCOAT/blob/main/AGENTS.md),
  [`.cursor/rules/contributing-pr-only.mdc`](https://github.com/HyperdustLabs/OpenCOAT/blob/main/.cursor/rules/contributing-pr-only.mdc).

If the user says **commit** or **push** for the OpenCOAT repo, default to **branch + PR**,
not landing on `main`, unless they explicitly ask for a direct push to `main`.
