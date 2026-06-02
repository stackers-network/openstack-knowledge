# Guide — walk me through a real task

When the user wants to *accomplish* something (most often: deploy a cloud), switch to
**guided** mode. You build a tailored plan from this repo's workflows and skills, then
walk it step by step — explaining as you go, confirming before anything destructive.

This is teaching-by-doing. The Critical Rules in `CLAUDE.md` apply to every command
you run against a real environment.

## Process

1. **Pin the goal.** Understand what they want before any steps. "Deploy an all-in-one
   for evaluation," "stand up a 3-controller HA cloud," "add a compute node," "upgrade
   2025.1 → 2025.2." Restate it back.

2. **Pick the path through the repo.** Map the goal to skills:
   - Deploy a cloud → `deploy/config-build` → `deploy/deploy` → `deploy/health-check`,
     with `reference/decision-guides` for backend choices and `reference/known-issues`
     for the target release.
   - Day-two change → `deploy/day-two`.
   - Upgrade → `operations/upgrades` (read it fully first — backups and prechecks are
     mandatory, not optional).

3. **Assess the environment.** Targeted questions:
   - How many hosts? What hardware (CPU/RAM/disk/NICs)?
   - OS and version? (Rocky Linux 10+, Ubuntu 24.04 — check `reference/compatibility`.)
   - Network topology — management, tenant, external/provider networks?
   - Starting fresh or changing something live?
   - Constraints: air-gapped, compliance, target OpenStack release, HA or not?

4. **Make the decisions.** Use `reference/decision-guides` to choose the networking
   backend (OVN / OVS / LinuxBridge) and storage backend (Ceph / LVM / NFS) with the
   tradeoffs spelled out. Record the choices — they drive every later step.

5. **Check known issues first.** Read `reference/known-issues/<release>` for the target
   version before generating any config. Surface anything relevant up front.

6. **Build and show the plan.** Produce a numbered, environment-specific plan that
   references the exact skill for each step. Show it; let the user adjust before you
   start.

7. **Walk each step.** For every step:
   - Explain what it does and why (this is the teaching part — don't just dump commands).
   - Show the exact command or config from the skill.
   - **Confirm before anything destructive or irreversible** (see Critical Rules).
   - Verify it worked. If not, go to step 8.

8. **Handle failures.** Route the symptom to `deploy/diagnose`; check
   `reference/known-issues` for a version-specific cause. Troubleshoot before moving on
   — don't paper over a failed step.

9. **Summarize.** Recap what was accomplished, the current state of the environment,
   and recommended follow-ups (verify with `deploy/health-check`, plan capacity with
   `operations/capacity-planning`).

## Ground it in current reality (the stackers edge)
Before relying on `reference/known-issues`, you can cross-check the live picture: fetch
the relevant project feed from stackers.network (`live_grounding.project_feed` in
`manifest.yaml`) to see if a fix or a regression landed *this week* that the static
known-issues file may not reflect yet. Mention it, but defer to operator judgment and
the skill content for the actual procedure.

## Handling specific requests
- "Guide me through [task]" — full process from step 1.
- "What's the next step?" — continue from where you left off.
- "Skip this step" — move on, noting what was skipped and the risk.
- "Go back" — revisit the previous step.
- "Explain this more" — you're in teaching-by-doing mode; go as deep as they want, then
  resume the walkthrough.
