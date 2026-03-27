# EGB — Effortless Git Branching (Agent Reference)

Compact description of the EGB branching and versioning model for AI agents and automated tooling.

## Core Principle

Versions are named at the start of a development cycle via `git tag`, not at the end. All version information is derived from git history using `git describe --tags`.

## Two Modes

### Applications (deployed services)

- **Version source:** `git describe --abbrev=10 --tags` → `1.96-42-gf97ab584a2`
- **No version in files.** Ever. The version is computed, not stored.
- **Branch hierarchy:** `master` → `staging-X.Y` → `release-X.Y`
- **Merge direction:** downward only (master → staging → release). Upward only via cherry-pick.
- **Bugfix rule:** fix on the lowest branch where the bug exists, then merge downward.
- **Sprint start:** `git tag 1.97` on master. All subsequent commits are `1.97-N`.
- **Format:** dashes, not dots. Not SemVer. Commit count = build number.

### Libraries (published artifacts)

- **Version source:** `SNAPSHOT` in build file during development, tags for releases.
- **One version file:** `pom.xml` (or equivalent) contains `X.Y-SNAPSHOT` on master.
- **Release:** `git checkout -b release-X.Y && git tag X.Y.0`. CI overrides version from tag at build time.
- **Patches:** fix on release branch, `git tag X.Y.1`. No file change needed.
- **After release branch:** bump master to next SNAPSHOT immediately (e.g. `0.47-SNAPSHOT`).
- **Format:** dots, SemVer. Patch version is a human decision about compatibility.
- **Rule:** SNAPSHOT is internal only. No external consumer should depend on it.

## Key Rules

1. **Tag first.** Version is a declaration, not a conclusion.
2. **Apps: no version files.** `git describe` is the single source of truth.
3. **Libs: one SNAPSHOT bump per cycle.** The only version commit that ever exists.
4. **Merge down, cherry-pick up.** Never merge upward.
5. **Fix at the lowest level.** Don't fix on master what only exists on release.
6. **CI computes everything.** `VERSION=$(git describe --abbrev=10 --tags)` for apps. Tag name override for libs.

## Branch Naming

| Branch | Purpose |
|---|---|
| `master` | Active development |
| `staging-X.Y` | Pre-production stabilization (apps only) |
| `release-X.Y` | Production / stable release line |

## Version Examples

| Type | Development | Staging | Release | Patch |
|---|---|---|---|---|
| Application | `1.97-7-g3a8b2c1f0e` | `1.97-3-g...` (on staging branch) | `1.97-1-g...` (on release branch) | automatic via commit count |
| Library | `0.46-SNAPSHOT` | n/a | `0.46.0` (tag) | `0.46.1` (tag) |

## CI One-Liners

```bash
# Application version
VERSION=$(git describe --abbrev=10 --tags)

# Library release (tag-triggered)
mvn versions:set -DnewVersion=$CI_COMMIT_REF_NAME && mvn deploy
```

## What NOT to Do

- Don't store application versions in files (pom.xml, package.json, etc.)
- Don't merge upward (release → staging → master direction is wrong)
- Don't let external consumers use SNAPSHOT artifacts
- Don't forget to bump master after cutting a release branch (libs)
- Don't keep dead release branches around
- Don't use SemVer for applications or auto-counting for libraries
