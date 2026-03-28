# EGB für Libraries

So handhaben wir Branching und Versionierung bei Libraries. Libraries folgen einem anderen Rhythmus als Applikationen — sie werden nicht in Umgebungen deployt, sondern publizieren Artefakte in eine Registry. Ihre Konsumenten brauchen einen Kompatibilitätsvertrag. Deshalb nutzen wir SemVer mit bewussten menschlichen Entscheidungen für Release-Versionen, während die Entwicklung auf SNAPSHOT läuft.

---

## Branch-Modell

```
master:  0.46-SNAPSHOT ─── commits ─── 0.47-SNAPSHOT ─── commits
              │
              └── release-0.46
                  tag 0.46.0
                  commit (fix)
                  tag 0.46.1
```

Zwei Arten von Branches:

- **master** — trägt immer eine `SNAPSHOT`-Version. Aktive Entwicklung.
- **release-X.Y** — Stabilisierung und Pflege. Releases passieren per Tags auf diesem Branch.

---

## Entwicklung

Master hat immer eine `SNAPSHOT`-Version in der Build-Datei. Das ist die **einzige** Stelle, an der eine Versionsangabe in einer Datei lebt:

```xml
<version>0.46-SNAPSHOT</version>
```

Jeder Commit auf jedem Branch publiziert ein SNAPSHOT-Artefakt. **Interne** Downstream-Projekte, die aktiv gegen die Library entwickeln, bekommen so immer die neueste Version für ihre Integrationstests.

Wichtig: SNAPSHOTs sind **ausschließlich für die interne Entwicklung und Integration** gedacht. Sie sind ein bewegliches Ziel — kein stabiles Versprechen, und keine Grundlage für externe Konsumenten.

> Kein externer Konsument sollte sich auf SNAPSHOT-Artefakte verlassen.

SNAPSHOT ist für interne Teams, die eine Library aktiv ändern und validieren. Stabile Nutzung passiert immer über getaggte Releases.

---

## Releasen

Wenn eine Version stabilisiert werden soll:

```bash
git checkout -b release-0.46
git tag 0.46.0
```

CI erkennt den Tag und überschreibt die Version zur Build-Zeit:

```yaml
Deploy Release:
  script:
    - mvn versions:set -DnewVersion=$CI_COMMIT_REF_NAME
    - mvn clean deploy
  only:
    - tags
```

`$CI_COMMIT_REF_NAME` ist `0.46.0` — der Tag-Name. Die `pom.xml` in Git sagt weiterhin `SNAPSHOT`. Nur das gebaute Artefakt bekommt die Release-Version. **Die Wahrheit liegt im Tag, nicht in einer Datei.**

---

## Patchen

Bug in einer released Version? Auf dem Release-Branch fixen, dann taggen:

```bash
git checkout release-0.46
# ... Bug fixen ...
git commit -m "Fix performance test threshold"
git tag 0.46.1
```

CI publiziert `0.46.1`. Keine POM-Änderung nötig. Jeder Tag ist eine bewusste menschliche Entscheidung: "Diese Änderung ist sicher für Konsumenten."

---

## Zurückmergen und Bumpen

Nach dem Erstellen eines Release-Branches geht Master sofort weiter:

```bash
git checkout master
git merge release-0.46          # null Konflikte
# Bump für den nächsten Entwicklungszyklus
sed -i 's/0.46-SNAPSHOT/0.47-SNAPSHOT/' pom.xml
git commit -m "Switch to 0.47-SNAPSHOT"
```

Oder, wenn der nächste Schritt größer ist:

```xml
<version>2.0-SNAPSHOT</version>
```

Der SNAPSHOT-Bump ist der **einzige** versionsbezogene Commit, der jemals in der History existiert. Eine Zeile, eine Datei, einmal pro Release. Das sind die gesamten Kosten.

Das ist eine der Stärken des Modells: die gepflegte Release-Linie und die nächste Entwicklungslinie sind sauber getrennt, ab dem Moment wo der Release-Branch erstellt wird.

---

## Gepflegte Versionen

Ein gepflegter Release-Branch wie `release-0.46` kann weiterhin regelmäßig SNAPSHOT-Builds publizieren — zum Beispiel wöchentlich.

Das ist nützlich, wenn eine Library unter kontrollierter Pflege steht und Änderungen in einer echten Applikation getestet werden müssen, bevor ein formaler Tag erstellt wird.

Aber die Unterscheidung muss klar bleiben:

| Artefakt-Typ | Bedeutung | Wer konsumiert es |
|---|---|---|
| `0.46-SNAPSHOT` | Work in Progress | Interne Teams, die aktiv testen |
| `0.46.0` | Stabiles Release | Jeder |
| `0.46.1` | Stabiler Patch | Jeder |

---

## Warum Punkte, keine Bindestriche

Applikationen nutzen Bindestriche (`1.96-42`), weil der Commit-Count eine automatische Build-Nummer ist — kein Kompatibilitätsversprechen.

Libraries nutzen Punkte (`0.46.1`), weil die Patch-Version eine **bewusste menschliche Entscheidung** ist: "Diese Änderung ist rückwärtskompatibel." Konsumenten müssen wissen, was sie bekommen.

| | Library | Applikation |
|---|---|---|
| Versionsbeispiel | `0.46.1` | `1.96-42` |
| Trennzeichen | Punkt (SemVer) | Bindestrich (git describe) |
| Patch-Bedeutung | "Rückwärtskompatibler Fix" | "42. Commit seit 1.96" |
| Wer entscheidet | Mensch taggt `0.46.1` | Git zählt automatisch |
| Vertrag | API-Kompatibilität | "Es deployt" |

Unserer Erfahrung nach erzeugt SemVer für Applikationen falsche Präzision. Auto-Counting für Libraries versteckt wichtige Kompatibilitätsinformation. EGB nutzt beides dort, wo es passt.

---

## CI-Setup

### SNAPSHOT-Builds

```yaml
Deploy Snapshot:
  script:
    - mvn deploy
  except:
    - tags
```

Jeder Commit publiziert einen SNAPSHOT. Keine Konfiguration nötig außer "kein Tag."

### Release-Builds

```yaml
Deploy Release:
  script:
    - mvn versions:set -DnewVersion=$CI_COMMIT_REF_NAME
    - mvn clean deploy
  only:
    - tags
```

Der Tag-Name wird zur Version. Die POM bleibt in Git unangetastet.

---

## Trade-offs

Dinge, die weiterhin Disziplin brauchen:

- **SNAPSHOT muss intern bleiben.** Sobald externe Konsumenten von SNAPSHOT abhängen, verliert man die Möglichkeit, Breaking Changes während der Entwicklung zu machen. Über Registry-Zugriffskontrollen oder Policy durchsetzen.
- **Master sofort bumpen nach dem Erstellen eines Release-Branches.** Wenn man es vergisst, publizieren zwei Branches dieselbe SNAPSHOT-Version — das sorgt für Verwirrung.
- **CI muss Tags korrekt fetchen.** Shallow Clones können sie verpassen. Sicherstellen, dass die Pipeline mit `--tags` oder ausreichender Tiefe fetcht.
- **Alte Release-Branches aufräumen.** Nur behalten, solange sie tatsächlich gepflegt werden. Ein `release-0.38`, den niemand anfasst, ist nur Rauschen.

---

## Zusammenfassung

In der Praxis brauchen unsere Libraries genau einen versionsbezogenen Commit pro Release-Zyklus: den SNAPSHOT-Bump auf Master nach dem Erstellen des Release-Branches. Alles andere — die Release-Version, Patch-Versionen, Artefakt-Publishing — wird durch Tags gesteuert. Die Build-Datei in Git sagt immer SNAPSHOT. Die Registry hat immer die echte Version. Diese saubere Trennung hat die Dinge bei uns über die Jahre einfach gehalten.
