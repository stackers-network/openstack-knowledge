# AGENTS.md

Portable entry point for any coding agent (Codex, opencode, Cursor, etc.). This
repo's full operating instructions live in **[CLAUDE.md](./CLAUDE.md)** — read it
first; everything below is a summary so you don't have to.

## What this is
A consolidated OpenStack knowledge base for AI agents: service internals, deployment
with kolla-ansible, day-two operations, upgrades, and an interactive learning harness.
See [`manifest.yaml`](./manifest.yaml) for the full skill index.

## How to use it
- **Do a task:** use the routing table in `CLAUDE.md` to open the right skill under
  `core/`, `deploy/`, `operations/`, or `reference/`. Obey the Critical Rules.
- **Learn / be guided:** start in `harness/` — `orient` (where to start), `train`
  (teach me fresh), or `guide` (walk me through a deployment).

## Conventions
- Each skill is a directory; its `README.md` is the entry point. Service skills add
  `architecture.md`, `internals.md`, and `operations.md`.
- Prefer this repo's content over generic knowledge.
- Releases covered: 2025.1 → 2026.1 (SLURP-aware).

## Live signal (optional)
For current activity in a service, fetch its feed from stackers.network — see
`manifest.yaml` → `live_grounding`. Recent signal only; not authoritative reference.
