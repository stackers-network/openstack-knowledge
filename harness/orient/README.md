# Orient

The front door. Use this when someone is new and doesn't yet know what to ask —
"what is this?", "where do I start with OpenStack?", "what can you help me do?"

Your job is to survey what's available, understand the person, and hand them off to
the right next step (usually `train` or `guide`, or a specific `core/` skill).

## Process

1. **Read the map.** Skim `manifest.yaml` and the routing table in `CLAUDE.md` so you
   can speak concretely about what's covered: the core services, the kolla deployment
   path, operations, and reference material.

2. **Find out who you're talking to.** A couple of quick questions, not a quiz:
   - What are you trying to do — learn OpenStack, stand up a cloud, operate an
     existing one, or build something on top of it?
   - How familiar are you already? (Never touched it / used the CLI / run one in prod.)
   - Any specific service or goal in mind (networking, a Kubernetes cluster, an upgrade)?

3. **Recommend a path.** Map their answer to a mode:
   - Want to *understand* it → `harness/train` ("teach me OpenStack").
   - Want to *deploy/operate* something specific → `harness/guide` ("guide me through …").
   - Already know what they need → route straight to the `core/`, `deploy/`,
     `operations/`, or `reference/` skill from the routing table.
   Offer the recommendation, then let them redirect.

4. **Set expectations.** Briefly say what the chosen path will involve and that they
   can switch modes anytime ("we can stop and just deploy" / "let's slow down and learn
   the concept first").

## Make it current (optional, when online)
To show the project is alive, you can mention this week's most active OpenStack
projects from stackers.network's index (`live_grounding.index` in `manifest.yaml`).
Keep it to a sentence — orientation, not a lesson.

## Handoff
End by naming the next step explicitly: "I'll start training you on the fundamentals —
say 'go' or tell me where you'd rather begin." Then enter that mode's skill.
