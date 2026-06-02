# Known Issues — Format and Usage

This directory documents known bugs, regressions, and workarounds for specific OpenStack releases. Check the file for your target release before deploying or upgrading.

## When to Read This

- Before deploying a new release for the first time
- Before upgrading from one release to another (read the **target** release file)
- When investigating an unexpected behavior that appeared after an upgrade
- When a deployment step fails in a way that is not covered by standard documentation

## File Index

| File | Release | Code Name | Release Date |
|---|---|---|---|
| [2025.1.md](2025.1.md) | 2025.1 | Epoxy | April 2025 |
| [2025.2.md](2025.2.md) | 2025.2 | Flamingo | October 2025 |
| [2026.1.md](2026.1.md) | 2026.1 | (codename pending) | April 2026 |

## Entry Format

Each issue follows this template:

```
### [Short Description — the symptom in 8 words or fewer]

**Symptom:** What the operator sees — error messages, behavior, affected operations.

**Cause:** Why it happens — the underlying bug or design change.

**Workaround:** Exact steps to mitigate or resolve the issue. Include full commands.

**Affected versions:** Release range where the bug is present (e.g., "2025.1.0 through 2025.1.2").

**Status:** One of:
  - Open — no fix merged yet
  - Fixed in <version> — patch released; upgrade to resolve
  - Upstream fix pending — patch proposed, not yet merged
  - Won't fix — accepted limitation
```

## Issue Categories

Issues are grouped within each release file under these headings:

- **Upgrade Blockers** — must be addressed before upgrading to this release
- **Configuration Changes** — options renamed, removed, or changed in behavior that break existing configs
- **Deprecations Enforced** — items deprecated in a prior release that are now removed
- **Known Bugs** — bugs present in the release that operators may encounter
- **Security Advisories** — OSSAs (OpenStack Security Advisories) relevant to this release
- **Performance Notes** — regressions or new tuning requirements

## Sources

- Official OpenStack release notes: https://releases.openstack.org
- Launchpad bug tracker: https://bugs.launchpad.net/openstack
- OpenStack Security Advisories: https://security.openstack.org/ossalist.html
- Individual project release notes (e.g., https://docs.openstack.org/nova/latest/release-notes.html)

## Contributing New Issues

When you encounter a bug not documented here:

1. Reproduce it reliably on a clean deployment.
2. Identify the affected component and version range.
3. Write a workaround with exact commands.
4. Add the entry under the correct release file following the format above.
5. Link the upstream Launchpad bug if one exists.
