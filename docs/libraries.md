# EGB for Libraries

This is how we handle library branching and versioning. Libraries follow a different rhythm than applications — they don't deploy to environments, they publish artifacts to a registry. Their consumers need a compatibility contract. That's why we use SemVer with explicit human decisions for release versions, while keeping development on SNAPSHOT.

---

## Branch Model

```
master:  0.46-SNAPSHOT ─── commits ─── 0.47-SNAPSHOT ─── commits
              │
              └── release-0.46
                  tag 0.46.0
                  commit (fix)
                  tag 0.46.1
```

Two kinds of branches:

- **master** — always carries a `SNAPSHOT` version. Active development.
- **release-X.Y** — stabilization and maintenance. Releases happen via tags on this branch.

---

## Development

Master always has a `SNAPSHOT` version in the build file. This is the **only** place where a version string lives in a file:

```xml
<version>0.46-SNAPSHOT</version>
```

Every commit to every branch publishes a SNAPSHOT artifact. Downstream projects that depend on the SNAPSHOT always get the latest.

SNAPSHOTs are useful while a library is being adjusted and tested inside an application context. But they are a **moving target** — not a stable promise.

> No external consumer should rely on SNAPSHOT artifacts.

SNAPSHOT is for internal teams who are actively changing and validating a library. Stable consumption always happens through tagged releases.

---

## Releasing

When ready to stabilize a version:

```bash
git checkout -b release-0.46
git tag 0.46.0
```

CI detects the tag and overrides the version at build time:

```yaml
Deploy Release:
  script:
    - mvn versions:set -DnewVersion=$CI_COMMIT_REF_NAME
    - mvn clean deploy
  only:
    - tags
```

`$CI_COMMIT_REF_NAME` is `0.46.0` — the tag name. The `pom.xml` in git still says `SNAPSHOT`. Only the built artifact gets the release version. **The source of truth is the tag, not a file.**

---

## Patching

Bug on a released version? Fix it on the release branch, then tag:

```bash
git checkout release-0.46
# ... fix the bug ...
git commit -m "Fix performance test threshold"
git tag 0.46.1
```

CI publishes `0.46.1`. No pom change needed. Each tag is a conscious human decision: "this change is safe for consumers."

---

## Merge Back and Bump

After cutting a release branch, master moves on immediately:

```bash
git checkout master
git merge release-0.46          # zero conflicts
# bump for next development cycle
sed -i 's/0.46-SNAPSHOT/0.47-SNAPSHOT/' pom.xml
git commit -m "Switch to 0.47-SNAPSHOT"
```

Or, if the next step is larger:

```xml
<version>2.0-SNAPSHOT</version>
```

The SNAPSHOT bump is the **only** version-related commit that ever exists in the history. One line, one file, once per release. That's the entire cost.

This is one of the model's strengths: the maintained release line and the next development line are cleanly separated from the moment the release branch is cut.

---

## Maintained Versions

A maintained release branch like `release-0.46` can still publish regular SNAPSHOT builds — for example on a weekly schedule.

This is useful when a library is under controlled maintenance and changes need to be tested in a real application before a formal tag is created.

But the distinction must stay clear:

| Artifact type | Meaning | Who consumes it |
|---|---|---|
| `0.46-SNAPSHOT` | Work in progress | Internal teams actively testing |
| `0.46.0` | Stable release | Anyone |
| `0.46.1` | Stable patch | Anyone |

---

## Why Dots, Not Dashes

Applications use dashes (`1.96-42`) because the commit count is an automatic build number — no compatibility promise attached.

Libraries use dots (`0.46.1`) because the patch version is a **conscious human decision**: "this change is backward compatible." Consumers need to know what they're getting.

| | Library | Application |
|---|---|---|
| Version example | `0.46.1` | `1.96-42` |
| Separator | Dot (SemVer) | Dash (git describe) |
| Patch meaning | "Backward compatible fix" | "42nd commit since 1.96" |
| Who decides | Human tags `0.46.1` | Git counts automatically |
| Contract | API compatibility | "It deploys" |

In our experience, using SemVer for applications creates false precision. Using auto-counting for libraries hides important compatibility information. EGB uses each where it fits.

---

## CI Setup

### SNAPSHOT Builds

```yaml
Deploy Snapshot:
  script:
    - mvn deploy
  except:
    - tags
```

Every commit publishes a SNAPSHOT. No configuration needed beyond "not a tag."

### Release Builds

```yaml
Deploy Release:
  script:
    - mvn versions:set -DnewVersion=$CI_COMMIT_REF_NAME
    - mvn clean deploy
  only:
    - tags
```

The tag name becomes the version. The pom stays untouched in git.

---

## Trade-offs

Things that still need discipline:

- **SNAPSHOT must stay internal.** The moment external consumers depend on SNAPSHOT, you lose the ability to make breaking changes during development. Enforce this through registry access controls or policy.
- **Bump master immediately after cutting a release branch.** If you forget, two branches publish the same SNAPSHOT version, causing confusion.
- **CI must fetch tags correctly.** Shallow clones can miss them. Make sure your pipeline fetches with `--tags` or sufficient depth.
- **Clean up old release branches.** Only keep them as long as they're actually maintained. A `release-0.38` that nobody touches is just noise.

---

## Summary

In practice, our libraries need exactly one version-related commit per release cycle: the SNAPSHOT bump on master after cutting the release branch. Everything else — the release version, patch versions, artifact publishing — is driven by tags. The build file in git always says SNAPSHOT. The registry always has the real version. That clean separation has kept things simple for us over the years.
