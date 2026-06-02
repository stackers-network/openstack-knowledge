# openstack-knowledge

**Composed OpenStack expertise for AI agents** — service internals, deployment with
kolla-ansible, day-two operations, upgrades, and an interactive learning harness.

This is the durable, iterable knowledge base behind the **Learn OpenStack**
experience on [stackers.network](https://stackers.network). It consolidates what
used to be three separate repos (`openstack-core`, `openstack-kolla`, and the
learning parts of `common-skills`) into one place that can be versioned and improved
over time — and consumed three different ways.

> Independent and community-maintained. Not affiliated with or endorsed by the
> OpenInfra Foundation. "OpenStack" is used under nominative fair use to describe
> what this covers.

## Three ways to consume it

1. **Clone it (offline / CLI-native).** Point Claude Code, Codex, or opencode at this
   directory — they read `CLAUDE.md` / `AGENTS.md` and gain the whole knowledge base.
   Works with no website and survives anything happening to any hosted service.
   ```bash
   git clone https://github.com/stackers-network/openstack-knowledge
   # then, in the repo, just ask your agent: "Teach me OpenStack" / "Guide me through a deploy"
   ```

2. **Over the web (zero install).** Tell any agent:
   > Read https://stackers.network/learn.md and teach me OpenStack.

   The site renders this repo into a `/learn.md` manifest your agent can follow,
   fetching individual skills on demand.

3. **Browse it (humans).** [stackers.network/learn](https://stackers.network/learn)
   renders the knowledge as readable pages.

## Three learning modes

| Mode | Say… | What happens |
|---|---|---|
| **orient** | "Where do I start with OpenStack?" | Surveys what's here, recommends a path for your goal and level. |
| **train** | "Teach me OpenStack fresh." | Assesses your level, builds a curriculum from `core/`, teaches concepts then procedures with exercises. |
| **guide**  | "Guide me through a deployment." | Assesses your environment, builds a plan, walks you through `deploy/` step by step. |

## Layout

```
harness/        learning experience — orient / train / guide
core/           service knowledge — foundation + identity, compute, networking, storage, …
deploy/         kolla-ansible lifecycle — config-build, deploy, health-check, diagnose, day-two
operations/     cross-cutting ops — upgrades (incl. SLURP), backup-restore, capacity-planning
reference/      decision-guides, compatibility matrix, known-issues
manifest.yaml   machine-readable index of every skill + consumption + live-grounding config
CLAUDE.md       agent entry: identity, critical rules, routing table, workflows
AGENTS.md       portable entry pointer for other agents
```

Each skill is a directory whose `README.md` is the entry point; service skills add
`architecture.md`, `internals.md`, and `operations.md`.

## The live edge

The knowledge here is deliberately timeless. stackers.network adds the *now*: when
the harness teaches a service, it can pull that project's weekly code-activity digest
(e.g. `stackers.network/projects/openstack-neutron/feed.xml`) so a lesson on Neutron
opens with what actually merged this week. Frozen fundamentals + live community pulse.

## Status

Scaffold. The `harness/`, `manifest.yaml`, and agent-entry files are authored. The
knowledge skill entries are laid out with structure + migration pointers — port the
authoritative content from the prior `openstack-core` / `openstack-kolla` repos into
each skill's `README.md` (+ `architecture.md` / `internals.md` / `operations.md`).

## Contributing

Improvements to skills flow here, then ripple to every consumer (clones, the web
manifest, and the rendered pages) on the next sync. Keep skills release-aware
(2025.1 → 2026.1) and prefer concrete commands and configs over prose.
