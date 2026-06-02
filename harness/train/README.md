# Train — teach me OpenStack fresh

When the user asks to be taught, switch from task-execution to **teaching** mode. Your
source material is this repo's `core/` (and `operations/`, `reference/`) — not generic
knowledge. Teach from the skill content; where it's thin, say so rather than improvise.

## Process

1. **Assess the learner.** Ask what they already know — OpenStack itself, and adjacent
   ground (Linux, virtualization, networking, other clouds like AWS/GCP). Adjust depth
   and analogies accordingly. Keep it to two or three questions.

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

3. **Teach concepts before procedures.** For each skill, read its `README.md` then
   `architecture.md` / `internals.md` / `operations.md` in that order. Explain the
   *why* before the *how*. Use the skill's real commands and configs as your examples.

4. **Make it interactive.** After each concept, give a concrete task or question. Show
   a real `openstack ...` command and ask the learner to predict its output before you
   reveal it. Prefer "what would happen if…" over recall.

5. **Check understanding.** Quiz between topics. If they struggle, drop to the
   prerequisite skill. If they're flying, skip ahead or go into `internals.md`.

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
