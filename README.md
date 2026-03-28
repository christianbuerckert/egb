# Effortless Git Branching (EGB)

*A lightweight branching model that worked extremely well for our team in practice.*

EGB grew out of real frustration. We kept fighting the same problems: version bumps that someone forgot, `pom.xml` conflicts on every merge back, release branches that needed ceremony, and hotfixes where nobody was sure which version they belonged to.

At some point we tried something different: **name the version at the start of a cycle, not at the end.** That one change removed most of the pain. We've been using this model for years now without a single branching incident, and we're sharing it in case it helps other teams too.

## Who This Is For

EGB works well if your team:
- ships applications **and** maintains libraries
- releases regularly (sprints, continuous delivery, or somewhere in between)
- is tired of merge conflicts in version files
- needs optional stabilization stages (staging, release) but doesn't want GitFlow's overhead

EGB is probably **not** for you if:
- you ship a single open-source library with no deployment pipeline — plain SemVer tagging is simpler
- you're a solo developer with one branch — you don't need a branching model
- your organization requires strict SemVer for deployed applications — EGB intentionally avoids that

## The Core Idea

**Set the version at the beginning, not the end.**

A version tag is not a stamp you put on a finished product. It's a name you give at the start — like naming a child at birth, not at graduation. Everything that follows inherits that name automatically.

```bash
git describe --tags
# → 1.96-42-gf97ab584a2
```

Tag `1.96`, 42 commits after it, at commit `f97ab584a2`. The version computes itself from git history. No file needs to change. No human needs to remember anything.

## Applications vs. Libraries

EGB treats them differently because they *are* different.

| | Applications | Libraries |
|---|---|---|
| What they are | Deployed services | Consumed artifacts |
| Version source | `git describe --tags` | SNAPSHOT + release tags |
| Version format | `1.96-42` (dash, auto) | `0.46.1` (SemVer, manual) |
| Version in files | Never | SNAPSHOT only |
| Merge conflicts | None | One bump per release |

## How EGB Compares

Before settling on EGB, we tried or evaluated most of the common models. Here's what we found — your mileage may vary, but these were the trade-offs as we experienced them.

### GitFlow

The most widely known model. Separate `develop` and `main` branches, plus feature, release, and hotfix branches. Strength: very explicit process with clear roles for every branch. Weakness: the ceremony was too heavy for our release cadence — version bumps in files, merges to main, merges back to develop, and `pom.xml` conflicts on almost every one. Hotfixes needed their own branch with yet another version bump.

**What EGB does differently:** no version files, so merges never conflict on metadata. The tag *is* the release. Hotfixes auto-increment by commit count instead of requiring manual version bumps.

### Trunk-Based Development

Everyone commits to a single branch. Short-lived feature branches are optional. Releases are cut by tagging trunk after the fact. Strength: maximum simplicity, minimal branching overhead, great for CI/CD. Weakness: we needed a stabilization phase before production, and trunk-based doesn't have one built in. Tagging after the fact also meant "what version is this commit?" wasn't always obvious.

**What EGB does differently:** keeps trunk-based simplicity for development but adds optional stabilization tiers (staging, release) when needed. Tags happen *before* development, so every commit already knows its version.

### GitHub Flow

One main branch, feature branches, pull requests, deploy from main. Strength: dead simple, works well for SaaS with continuous deployment. Weakness: assumes every merge to main is deployable. We needed staged rollouts and occasionally maintained older versions — GitHub Flow doesn't have a concept for that.

**What EGB does differently:** adds explicit versioning and optional environment tiers without losing the simplicity. Teams that only need main + release can use EGB as a slightly more structured GitHub Flow with built-in version identity.

### GitLab Flow

Environment branches (`staging`, `production`) that code flows through via merges. Strength: environment-aware, fits well with GitLab CI. Weakness: we found that environment branches accumulated drift when not merged regularly. Merge direction wasn't always clear, leading to "which branch should I target?" confusion.

**What EGB does differently:** similar environment-tier idea, but with a strict directional rule (merge down, cherry-pick up) and version-tagged branches (`staging-1.97` instead of just `staging`). The version is baked into the branch name, so there's no ambiguity about which release a branch belongs to.

### Release Flow (Microsoft)

Develop on main, create release branches when ready to ship, cherry-pick hotfixes into release branches. Used at scale by Microsoft for products like Azure DevOps. Strength: proven at massive scale, clean separation of release lines. Weakness: cherry-picking is the primary mechanism for hotfixes — at scale this requires tooling and discipline. Versioning is separate from the branching model.

**What EGB does differently:** similar in spirit (main + release branches, cherry-pick for hotfixes), but integrates versioning directly. The tag-first approach means versions are deterministic from git history. EGB also allows downward merges alongside cherry-picks, which reduced our cherry-pick volume in practice.

### At a Glance

| Aspect | GitFlow | Trunk-Based | GitHub Flow | GitLab Flow | Release Flow | EGB |
|--------|---------|-------------|-------------|-------------|--------------|-----|
| Version source | File | Tag (after) | None | Tag/file | Separate | Tag (before) |
| Merge conflicts | Frequent | Rare | Rare | Occasional | Rare | None (apps) |
| Release ceremony | Heavy | Light | None | Light | Medium | Minimal |
| Stabilization tiers | Fixed | None | None | Environment branches | Release branches | Flexible |
| Hotfix mechanism | Branch + bump | Re-tag | Fix on main | Merge to env branch | Cherry-pick | Fix low, merge down |
| Maintained versions | Possible | Hard | No | No | Yes | Yes |

## Deep Dives

**English:**
- **[Applications](docs/applications.md)** — branch hierarchy, versioning, deployment flow, CI setup
- **[Libraries](docs/libraries.md)** — SNAPSHOT workflow, release tagging, patch management, CI setup

**Deutsch:**
- **[Applikationen](docs/de/applications.md)** — Branch-Hierarchie, Versionierung, Deployment-Flow, CI-Setup
- **[Libraries](docs/de/libraries.md)** — SNAPSHOT-Workflow, Release-Tagging, Patch-Management, CI-Setup

**For AI Agents:**
- **[AGENT.md](AGENT.md)** — compact, structured reference for automated tooling and AI agents

## Quick Start

Tag your next sprint:

```bash
git tag 1.97
git describe --abbrev=10 --tags
# → 1.97-0-g3a8b2c1f0e
```

That's it. Every commit from now on carries a unique, traceable version. Read the deep dives for the full workflow.

## A Note on Tone

EGB is not "the right way" to do branching. It's a model that emerged from our day-to-day work and solved the specific problems we kept running into. We're sharing it because it might help teams with similar pain points — not because we think everyone should adopt it. Every team has different constraints. If your current model works well, that's great.

## Contributing

Found something unclear or have a suggestion? Open an issue or submit a pull request. Contributions that keep EGB simple are especially welcome.

## License

This project is licensed under the [MIT License](LICENSE).

---

<sub>The idea and model behind EGB are by [Christian Buerckert](https://github.com/christianbuerckert). The documentation was written with the help of [Claude](https://claude.ai).</sub>
