# EGB für Applikationen

So handhaben wir Branching und Versionierung bei Applikationen. Applikationen sind deployte Services — sie laufen oder sie laufen nicht. Sie haben keine API-Konsumenten, die Kompatibilitätsversprechen brauchen. Das haben wir uns zunutze gemacht: Versionen werden aus der Git-History berechnet, nie in Dateien gespeichert, und Merge-Konflikte über Versionsnummern sind bei uns einfach nicht mehr vorgekommen.

---

## Branch-Hierarchie

Applikationen nutzen ein gestuftes Branch-Modell. Die Anzahl der Stufen ist flexibel — manche Teams nutzen zwei, andere vier. Das Prinzip ist immer dasselbe:

**Merges fließen nach unten. Fixes fließen nur per Cherry-Pick nach oben.**

```
master          (Entwicklung — auto-deploy auf Dev-Umgebung)
  │
  ├── merge ↓
  │
  staging-1.96  (Vorproduktion — auto-deploy auf Staging)
  │
  ├── merge ↓
  │
  release-1.96  (Produktion — manueller Deploy-Trigger)
```

```
master:  tag 1.96 ─── commits ─── tag 1.97 ─── commits
              │                        │
              └── staging-1.96         └── staging-1.97
                    │
                    └── release-1.96
                        fix → 1.96-1
                        fix → 1.96-2
```

Ein Fix auf `release-1.96` kann in `staging-1.96` gemergt werden (nach unten). Aber ein Fix auf `staging-1.96` muss per Cherry-Pick nach `release-1.96` — oder besser: der Entwickler denkt vorher nach, von wo er brancht. Wird ein Fix auf Release gebraucht, dann Branch von Release.

Stufen hinzufügen oder entfernen je nach Bedarf. Die Richtungsregel ändert sich nicht.

---

## Versionsformat

Applikationsversionen nutzen **Bindestriche**, keine Punkte:

```
1.96-42-gf97ab584a2
 │    │  └── Commit-Hash
 │    └── Commits seit Tag (= Build-Nummer)
 └── Versionsname (Tag)
```

Das ist bewusst **kein SemVer**. Was würde eine Patch-Version bei einem deploybaren Service bedeuten? "Rückwärtskompatibel"? Wie denn — es ist eine laufende Applikation, keine Library-API. Der Commit-Count ist eine ehrliche Build-Nummer, kein Kompatibilitätsversprechen.

### Deterministisch

Die Version ist eine reine Funktion der Git-History: `Tag + Commit-Count + Hash`. Zwei Personen, die `git describe` auf demselben Commit ausführen, bekommen immer dasselbe Ergebnis. Kein Zustand außerhalb von Git. Keine Race Conditions.

### Monoton

Der Commit-Count seit dem Tag garantiert Sortierung. `1.96-7` kommt immer vor `1.96-8`. Auf einem Release-Branch bekommt jeder Fix automatisch eine höhere Nummer.

---

## Workflow

### 1. Sprint starten

Master taggen:

```bash
git tag 1.97
```

Das war's. Der Sprint hat einen Namen. Jeder Commit ist ab jetzt `1.97-N`, wobei N automatisch hochzählt.

### 2. Auf Master entwickeln

Commits landen auf Master. CI deployt jeden Commit automatisch auf die **Dev**-Umgebung. Jeder Build bekommt eine eindeutige, monotone Version gratis:

```bash
VERSION=$(git describe --abbrev=10 --tags)
# → 1.97-7-g3a8b2c1f0e
```

Keine Dateiänderungen nötig.

### 3. Staging erstellen

Wenn bereit für Vorproduktions-Tests:

```bash
git checkout -b staging-1.97
```

CI erkennt `staging-*` und deployt automatisch auf die **Staging**-Umgebung. Neue Features von Master können jederzeit nach unten in Staging gemergt werden. Staging-spezifische Fixes werden direkt auf dem Staging-Branch committed.

CI kann dabei intelligent sein: Minor-Version der Pipeline mit der aktuell deployten Staging-Version vergleichen. Gleiche Minor → **Warmfix** (In-Place-Update). Andere Minor → **neues Staging** (vollständiges Deployment).

### 4. Release erstellen

Wenn Staging stabil ist:

```bash
git checkout staging-1.97
git checkout -b release-1.97
```

CI erkennt `release-*` und deployt auf eine **Hotfix**-Testumgebung. Produktions-Deployment ist ein **manueller Trigger** — ein Mensch klickt den Button nach Verifikation.

Gleiche CI-Intelligenz: gleiche Minor wie Prod → **Hotfix**. Andere Minor → **Release**. Benachrichtigungen gehen automatisch an alle Teams.

### 5. Bugs fixen

Wo man brancht, bestimmt wo der Fix landet:

| Bug gefunden auf | Branch von | Fließt nach |
|---|---|---|
| Produktion | `release-X.Y` | merge → staging → master |
| Staging | `staging-X.Y` | merge → master. Cherry-Pick nach Release wenn kritisch. |
| Nur Dev | `master` | merge → staging wenn bereit |

**Fix auf der niedrigsten Stufe wo der Bug existiert, dann nach unten mergen.** Nie nach oben mergen — das würde unfertige Arbeit mitziehen.

Das ist eine der wichtigsten EGB-Gewohnheiten. Nicht den Fix zu hoch ansetzen, wenn der Bug weiter unten in der Kette existiert.

### 6. Zurückmergen

Release-Fixes fließen per Merge nach unten zurück in Staging und Master. Null Konflikte, weil keine Versionsdateien geändert wurden:

```bash
git checkout staging-1.97
git merge release-1.97        # null Konflikte

git checkout master
git merge staging-1.97         # null Konflikte
```

### 7. Nächster Sprint

Master taggen, wiederholen:

```bash
git tag 1.98
```

Wenn die nächste Version ein großer Meilenstein ist, stattdessen `2.0` taggen. Der Tag-Name ist die einzige menschliche Entscheidung.

---

## CI-Setup

### Versionsberechnung

```yaml
Setup:
  script:
    - VERSION=$(git describe --abbrev=10 --tags origin/${CI_COMMIT_REF_NAME})
    - echo "VERSION=${VERSION}" >> dot.env
  artifacts:
    reports:
      dotenv: dot.env
```

### Build und Deploy

```yaml
Build:
  script:
    - mvn versions:set -DnewVersion=${VERSION}
    - mvn clean install
    - docker build -t myapp:${VERSION} .
```

Die gesamte Versionslogik ist eine Zeile. `$VERSION` fließt in Docker-Tags, Kubernetes-Manifeste, Artefaktnamen — überall hin.

### Deployment-Klassifizierung

CI kann den Deployment-Kontext automatisch ableiten, indem die Minor-Version mit dem verglichen wird, was in Staging oder Produktion läuft:

| Aktuelle Minor vs. deployte | Klassifizierung |
|---|---|
| Unterschiedlich | **Release** (vollständiges Deployment) |
| Gleich | **Hotfix** oder **Warmfix** (In-Place-Update) |

Das steuert Benachrichtigungen, Freigabe-Gates und Rollback-Strategien — alles ohne manuellen Input.

---

## Trade-offs

Dinge, die weiterhin Disziplin brauchen:

- **CI muss Tags korrekt fetchen.** Shallow Clones können Tags verpassen. Sicherstellen, dass die Pipeline mit `--tags` oder ausreichender Tiefe fetcht.
- **Merge- und Cherry-Pick-Richtung muss allen klar sein.** Die Richtungsregel sichtbar dokumentieren. Verstöße erzeugen genau das Chaos, das EGB vermeidet.
- **Release-Branches aufräumen.** Nur behalten, solange sie tatsächlich gepflegt werden. Tote Branches sind verwirrende Branches.

---

## Warum es bei uns funktioniert hat

Die Version ist nie eine Dateiänderung, also gibt es nie Merge-Konflikte bei Versions-Metadaten. Die Version ist nie eine manuelle Entscheidung (nach dem initialen Tag), also vergisst sie niemand. Die Version ist immer auf einen spezifischen Commit zurückführbar, also ist Debugging unkompliziert.

Für unser Team war der größte Gewinn, dass der Workflow langweilig wurde — und langweilig stellte sich als genau das heraus, was wir von Branching brauchten.
