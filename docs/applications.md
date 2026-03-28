# EGB for Applications

This is how we handle application branching and versioning. Applications are deployed services — they either run or they don't. They don't have API consumers who need compatibility promises. We leaned into that: versions are computed from git history, never stored in files, and merge conflicts over version strings simply stopped happening.

---

## Branch Hierarchy

Applications use a tiered branch model. The number of tiers is flexible — some teams use two, others four. The principle is always the same:

**There are two directions, and each has its own mechanism.**

```
                    toward stabilization →
master ──────────── staging-1.96 ──────────── release-1.96
  (development)       (pre-production)          (production)
                    ← toward development
```

- **Toward stabilization** (master → staging → release): new features and code move this way, always via **merge**.
- **Toward development** (release → staging → master): fixes flow back this way, also via **merge**.
- **Cherry-pick**: only needed when a fix from a branch closer to development must reach a branch closer to production — e.g., a staging fix that's also critical on release. Better yet: if a fix is needed on release, branch off release in the first place.

```
master:  tag 1.96 ─── commits ─── tag 1.97 ─── commits
              │                        │
              └── staging-1.96         └── staging-1.97
                    │
                    └── release-1.96
                        fix → 1.96-1
                        fix → 1.96-2
```

Add or remove tiers based on how many stabilization gates your process needs. The direction rules don't change.

---

## Version Format

Application versions use **dashes**, not dots:

```
1.96-42-gf97ab584a2
 │    │  └── commit hash
 │    └── commits since tag (= build number)
 └── version name (tag)
```

This is intentionally **not SemVer**. What would a patch version mean for a deployed service? "Backward compatible"? How — it's a running application, not a library API. The dash-separated commit count is an honest build number, not a compatibility promise.

### Deterministic

The version is a pure function of git history: `tag + commit count + hash`. Two people running `git describe` on the same commit always get the same result. No state outside of git. No race conditions.

### Monotonic

Commit count since the tag guarantees ordering. `1.96-7` always comes before `1.96-8`. On a release branch, every fix gets a higher number automatically.

---

## Workflow

### 1. Start a Sprint

Tag master:

```bash
git tag 1.97
```

That's it. The sprint has a name. Every commit from now on is `1.97-N` where N counts up automatically.

### 2. Develop on Master

Commits land on master. CI auto-deploys every commit to the **dev** environment. Each build gets a unique, monotonic version for free:

```bash
VERSION=$(git describe --abbrev=10 --tags)
# → 1.97-7-g3a8b2c1f0e
```

No file changes needed.

### 3. Create Staging

When ready for pre-production testing:

```bash
git checkout -b staging-1.97
```

CI detects `staging-*` and auto-deploys to the **staging** environment. New features from master can be merged down into staging at any time. Staging-specific fixes are committed directly on the staging branch.

CI can be smart about it: compare the pipeline's minor version against the currently deployed staging version. Same minor → **Warmfix** (in-place update). Different minor → **new Staging** (full deployment).

### 4. Create Release

When staging is stable:

```bash
git checkout staging-1.97
git checkout -b release-1.97
```

CI detects `release-*` and deploys to a **hotfix** test environment automatically. Production deployment is a **manual trigger** — a human clicks the button after verification.

Same CI intelligence: same minor as prod → **Hotfix**. Different minor → **Release**. Notifications go out to all teams automatically.

### 5. Fix Bugs

Where you branch from determines where the fix lands:

| Bug found on | Branch from | Flows to |
|---|---|---|
| Production | `release-X.Y` | merge back toward development (→ staging → master) |
| Staging | `staging-X.Y` | merge back toward development (→ master). Cherry-pick toward stabilization (→ release) if critical. |
| Dev only | `master` | merge toward stabilization (→ staging) when ready |

**Fix at the lowest level where the bug exists.** Fixes naturally merge back toward development. They only need a cherry-pick if they must also reach a more stable branch.

This is one of the most important EGB habits. Do not start the fix too high if the bug already exists lower in the chain.

### 6. Merge Back

Release fixes flow back toward development (release → staging → master) via merge. Zero conflicts because no version files were changed:

```bash
git checkout staging-1.97
git merge release-1.97        # zero conflicts

git checkout master
git merge staging-1.97         # zero conflicts
```

### 7. Next Sprint

Tag master, repeat:

```bash
git tag 1.98
```

If the next version is a major milestone, tag `2.0` instead. The tag name is the only human decision.

---

## CI Setup

### Version Computation

```yaml
# CI setup stage
Setup:
  script:
    - VERSION=$(git describe --abbrev=10 --tags origin/${CI_COMMIT_REF_NAME})
    - echo "VERSION=${VERSION}" >> dot.env
  artifacts:
    reports:
      dotenv: dot.env
```

### Build and Deploy

```yaml
Build:
  script:
    - mvn versions:set -DnewVersion=${VERSION}
    - mvn clean install
    - docker build -t myapp:${VERSION} .
```

The entire version logic is one line. `$VERSION` flows into Docker tags, Kubernetes manifests, artifact names — everywhere.

### Deployment Classification

CI can derive deployment context automatically by comparing the current version's minor number against what's running in staging or production:

| Current minor vs. deployed | Classification |
|---|---|
| Different | **Release** (full deployment) |
| Same | **Hotfix** or **Warmfix** (in-place update) |

This drives notifications, approval gates, and rollback strategies — all without manual input.

---

## Trade-offs

Things that still need discipline:

- **CI must fetch tags correctly.** Shallow clones can miss tags. Make sure your CI fetches with `--tags` or sufficient depth.
- **Merge and cherry-pick direction must be clear to everyone.** Document the direction rule visibly. Violations create the exact mess EGB avoids.
- **Release branches should be cleaned up.** Only keep them as long as they're actually maintained. Dead branches are confusing branches.

---

## Why It Worked for Us

The version is never a file change, so merges never conflict on version metadata. The version is never a manual decision (after the initial tag), so nobody forgets. The version is always traceable to a specific commit, so debugging is straightforward.

For our team, the biggest win was that the workflow became boring — and boring turned out to be exactly what we needed from branching.
