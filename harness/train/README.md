# Train — teach me OpenStack fresh

When the user asks to be taught, switch from task-execution to **teaching** mode. Your
source material is this repo's `core/` (and `operations/`, `reference/`) — not generic
knowledge. Teach from the skill content; where it's thin, say so rather than improvise.

> **Teaching means _explaining_, not quizzing.** Most of your output should be you
> clearly explaining concepts from the skill content — in plain language, with concrete
> examples. Asking a question is an occasional *check*, never the lesson itself. Before
> teaching a topic, actually fetch and read its skill file(s); don't teach from memory.
> **If your recent replies have been mostly questions, you are doing it wrong** — fetch
> the next skill and explain it.

## Process

1. **Assess briefly.** Ask one or two quick questions about the learner's background
   (OpenStack, Linux, virtualization, other clouds) to calibrate depth — then move on.
   This is a 30-second step, not an interview. If they already told you what they want,
   skip it and start teaching.

2. **Build a curriculum.** Read the relevant `core/` skills and sequence them
   foundation → services → operations. A sensible default arc:
   1. `core/foundation/architecture` — the mental model: control plane vs. compute,
      how services talk over the message bus and the service catalog.
   2. `core/identity` (Keystone) — everything authenticates here; teach this before any
      service that depends on it.
   3. `core/compute` (Nova + Placement) and `core/networking` (Neutron) — the heart.
   4. `core/storage/*` — block (Cinder), image (Glance), then object/shared as needed.
   5. Higher-level services by interest — `orchestration`, `load-balancing`, `dns`,
      `bare-metal`, `magnum`.
   6. `operations/` — how clouds are kept healthy and upgraded.

   Present the path and let the user reorder or trim it. Honor "I only care about X."

3. **Teach — this is the main activity.** For each topic, fetch the skill's `README.md`
   then `architecture.md` / `internals.md` / `operations.md` in that order, and *explain*
   the material: the *why* before the *how*, in plain language, with the skill's real
   commands and configs as worked examples. Spend the bulk of your output here, in full
   prose — not in questions.

4. **Check understanding — _after_ you've taught.** Once you've explained a concept, you
   may pose a single question or a "what would happen if…" to make it stick. This comes
   *after* the explanation, not instead of it. Never send a reply that is only questions.
   If the learner struggles, drop to the prerequisite skill and explain more; if they're
   flying, go deeper into `internals.md`.

6. **Connect the dots.** Explicitly link services: a booting instance touches Keystone
   (auth), Placement (scheduling), Glance (image), Neutron (port), Cinder (volume).
   Trace that path so the architecture stops being a list of names.

7. **Summarize each section.** Recap the concepts and what the learner can now *do*
   before moving on.

## Ground every service lesson in live activity (the stackers edge)

This is what makes training here different from reading docs. When you start a service
that maps to an OpenStack Gerrit project, fetch its current digest from stackers.network
(`live_grounding.project_feed` in `manifest.yaml`, slug = project with `/` → `-`):

- networking → `https://stackers.network/projects/openstack-neutron/feed.xml`
- compute → `https://stackers.network/projects/openstack-nova/feed.xml`
- identity → `https://stackers.network/projects/openstack-keystone/feed.xml`

Open the lesson with one line of *now*: "Neutron is actively worked on — this week the
biggest changes were in the OVN backend." It makes the concept concrete and shows the
project is alive. Treat the feed as current signal, **not** as authoritative reference —
the skill content is the source of truth for how things work.

If offline or a feed 404s, skip this step silently and teach from the skill content.

## Handling specific requests
- "Train me on OpenStack" — full arc from the assessment.
- "Train me on [service]" — jump to that `core/` skill and teach from there.
- "Quiz me" — test what's been covered so far.
- "What should I learn next?" — recommend based on progress and stated goals.
- "Just show me how to do X" — drop to `guide` mode for that task.

## Session continuity
Track which skills the learner has covered this session. If they stop, summarize where
they left off and what remains, so a later session (or a fresh agent) can resume.
