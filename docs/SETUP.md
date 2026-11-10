# Setup & Betrieb – JOAs Weltherrschaft

Stand: 10.11.2026. Diese Datei beschreibt, wie das Repo aufgebaut ist, was eingestellt wurde und was noch offen ist. Wer weitermacht (Papa oder Claude), fängt hier an.

## Ziel

Joa (13) baut mit Claude Websites. **Nichts geht live, ohne dass Papa es freigibt**, und jede Änderung wird automatisch auf Geheimnisse (API-Keys, Passwörter) und Sicherheitsprobleme geprüft. Komplett getrennt von Online United: eigene GitHub-Organisation, eigenes Cloudflare-Konto, keine Firmen-Keys, kein IONOS-Server.

## Ablauf

```
Joa + Claude           GitHub (automatisch)                 Papa (Handy, GitHub-App)    Automatisch
────────────           ────────────────────                 ────────────────────────    ───────────
"veröffentlichen" ──►  Pull Request                    ──►  Benachrichtigung      ──►  Merge auf main
                       ├ secrets (gitleaks)   Pflicht       Checks grün?                 Cloudflare Pages
                       ├ code (semgrep)       Pflicht       Claude-Review lesen          → Live-Domain
                       ├ dependencies (npm)   Pflicht       Vorschau-Link ansehen
                       ├ claude-review        Kommentar     ✅ Approve + Merge
                       └ Vorschau-URL (Cloudflare)          💬 oder Kommentar → Joa
```

## Was eingerichtet ist

### GitHub-Organisation `JOAs-Weltherrschaft`
- Gehört Papa (`matthias123OU`, persönlicher Account, nicht Online United).
- Mitglieder: Standardrecht **read**, dürfen **keine Repos anlegen** (weder public noch private), keine Pages, keine Forks privater Repos.

### Repo `JOAs-Weltherrschaft/website` (öffentlich)
Öffentlich, weil Branch-Schutz und GitHub-Secret-Blocker im Free-Plan nur für öffentliche Repos gelten. Die Website ist ohnehin öffentlich.

| Einstellung | Wert |
|---|---|
| Secret Scanning + **Push Protection** | an (GitHub blockiert bekannte Key-Typen schon beim Push) |
| Dependabot Alerts + Security Updates | an, Versions-Updates wöchentlich (`.github/dependabot.yml`) |
| Merge-Arten | nur Squash, Branch wird nach Merge gelöscht |
| Actions-Token | standardmäßig nur lesend, darf keine PRs freigeben |
| Wiki / Projects | aus |

### Ruleset `protect-main` (ID 26691437) auf `main`
- Kein Löschen, kein Force-Push, kein direkter Push.
- Pull Request Pflicht mit **1 Freigabe vom Code Owner** (`CODEOWNERS`: `* @matthias123OU`).
- Neue Pushes nach der Freigabe → Freigabe verfällt (**last push approval**). Joa kann nach deinem OK nichts nachschieben.
- Pflicht-Checks: `secrets (gitleaks)`, `code (semgrep)`, `dependencies (npm audit)`.
- Ausnahme: Org-Admin (Papa) kann eigene PRs ohne fremde Freigabe mergen. Joa wird nur **Member**, nie Admin.

### Prüfungen (`.github/workflows/`)
- **`security.yml`** (bei jedem PR und Push auf main)
  - gitleaks über die **ganze Git-Historie**
  - falls ein Build existiert (`npm run build`): bauen und **auch die fertige Website** scannen. Das fängt Keys, die über `VITE_*`/`NEXT_PUBLIC_*` in den Browser-Code wandern.
  - Semgrep (`p/default`, `p/secrets`, nur Schweregrad ERROR blockiert)
  - `npm audit --audit-level=high`, wenn `package-lock.json` existiert
- **`claude-review.yml`**: läuft als `pull_request_target`, also immer die Version aus `main`. Ein PR kann den Review nicht manipulieren. Der PR-Code wird nie ausgeführt, nur der Diff wird als Text an die Claude-API geschickt. Ergebnis: Kommentar mit Ampel 🟢/🟡/🔴 in Alltagssprache. Prompt: `.github/review-prompt.md`. **Inaktiv, solange das Secret `ANTHROPIC_API_KEY` fehlt** (überspringt sich dann).

### Regeln für Joas Claude
`CLAUDE.md` legt fest: nie auf `main`, „veröffentlichen“ = Branch + PR, vor Commit gitleaks, keine Keys im Browser-Code (APIs nur über Cloudflare Pages Functions mit Secrets in Cloudflare), nur eigene Keys mit Limit, Privatsphäre-, Datensammlungs- und Urheberrechts-Regeln, Schutzdateien nicht anfassen.

### Website
- Veröffentlichter Ordner: `public/` (statisch, kein Build). Platzhalter `index.html`, Sicherheits-Header in `public/_headers`.

## Getestet (06.11.2026)
- Harmloser Test-PR: alle Checks grün, Merge trotzdem **gesperrt** (Freigabe fehlt) ✅
- Gefälschter Key in `public/test.js`: **gitleaks und Semgrep rot**, Merge gesperrt ✅. Test-PR #2 geschlossen, Branch gelöscht.

## Offen

| # | Wer | Was |
|---|---|---|
| 1 | Papa | **Cloudflare-Konto** privat anlegen (dash.cloudflare.com, Free). |
| 2 | Claude | Cloudflare Pages mit Repo verbinden: Production-Branch `main`, Build-Befehl leer, Output `public`. Vorschau-Deployments mit **Cloudflare Access** schützen (nur Papas E-Mail). |
| 3 | Papa + Claude | Domain bei IONOS: CNAME der (Sub-)Domain auf `<projekt>.pages.dev`. Hauptdomain ohne `www` braucht ggf. Nameserver-Umzug zu Cloudflare. **Domain noch nicht genannt.** |
| 4 | Papa | Anthropic-API-Key (privat, Ausgabenlimit ~5 €/Monat) eintragen: `gh secret set ANTHROPIC_API_KEY -R JOAs-Weltherrschaft/website`. Nie in den Chat. |
| 5 | Papa | GitHub-App aufs Handy, Login `matthias123OU`, Benachrichtigungen an. |
| 6 | Joa + Claude | Joa ist seit 08.11. 13 → GitHub-Account anlegen, Papa lädt ihn als **Member** ein, Repo-Rolle **write** (Settings → Collaborators and teams). |
| 7 | Claude | Joas Rechner: Git, GitHub CLI (`gh auth login`), gitleaks, Repo klonen, Einweisung („sag Claude *veröffentlichen*“). |
| 8 | Papa | Dependabot-PR #1 (`actions/checkout` 4→7) prüfen und mergen. |
| 9 | Papa | Impressum auf der Website (Verantwortlicher: Papa), sobald Inhalte live gehen. |

## Grenzen
- Joa kann Seiten auch bei anderen Diensten veröffentlichen (Netlify, Vercel, Claude-Artifacts). Das sperrt kein Repo, das braucht eine Absprache.
- Scanner finden nicht alles. Deshalb zusätzlich Claude-Review und Papas Freigabe.
- Papas globaler Claude-Hook `release_gate.py` blockiert auch hier Pushes auf `main`. Änderungen am Repo laufen daher immer per PR.

## Lokale Pfade (Papas Rechner)
- Repo: `C:\Users\matth\joa-web` (nicht unter `online-united`)
