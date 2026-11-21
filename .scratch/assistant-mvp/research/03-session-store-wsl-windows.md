# Research: Copilot-Session-Store Windows ↔ WSL

Ticket: [issues/03-session-store-wsl-windows.md](../issues/03-session-store-wsl-windows.md)
Stand: 2026-11-21 · lokale CLI-Version: `GitHub Copilot CLI 1.0.115-5` (WSL, `~/.local/bin/copilot`)

Legende: **[V-doc]** verifiziert in Primärquelle (docs.github.com / github.blog / github/copilot-cli) · **[V-lokal]** verifiziert an der lokalen CLI (`--help`, `help config`, `help environment`, Verzeichnisnamen) · **[sekundär]** nur Drittquelle · **[nicht belegt]** in keiner Quelle gefunden · **[Einschätzung]** eigene Schlussfolgerung.

## Kurzantwort

- CLI und App teilen sich **keinen** gemeinsamen lokalen Store über die OS-Grenze. Jede Runtime schreibt in ihr eigenes `$COPILOT_HOME` (Default `~/.copilot`) – WSL-CLI also nach `/home/<user>/.copilot`, Windows-App nach dem Windows-`~/.copilot`. Ein offizieller Weg, beide auf ein Verzeichnis zu legen, ist **nicht dokumentiert**.
- Die **offizielle Brücke ist der Cloud-Sync** der Sessions zum GitHub-Account (Default an, Setting `remoteExport`). `/chronicle` arbeitet auf synchronisierten Sessions mit; ob Sessions eines *anderen Rechners/OS* im lokalen `/chronicle` der App landen, steht **nicht** explizit in den Docs.
- **Memory** liegt serverseitig auf GitHub (Repo- und User-Ebene) → automatisch geteilt zwischen CLI (WSL) und App (Windows), gleiches Konto vorausgesetzt.
- Die App kann Sessions **selbst in einer WSL-2-Distro laufen lassen** („WSL hosts“, experimentell). Das ist der naheliegende Weg, App und WSL zusammenzubringen – wo diese Sessions gespeichert werden, ist **nicht dokumentiert**.

## 1. Speicherorte Copilot CLI

| Was | Ort | Beleg |
|---|---|---|
| Config/State-Root | `$COPILOT_HOME`, Default `$HOME/.copilot` | [V-lokal] `copilot help environment`; [V-doc] [CLI config dir reference](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-config-dir-reference) |
| User-Settings | `settings.json` | [V-lokal] `copilot help config` → „Files and directories“ |
| CLI-State (Accounts, trusted folders, Plugins) | `config.json` | [V-lokal], [V-doc] |
| MCP-Server (User) | `mcp-config.json` | [V-lokal], [V-doc] |
| Persönliche Instructions | `copilot-instructions.md`, `instructions/*.instructions.md` | [V-lokal] |
| Agents/Skills/Hooks | `agents/`, `skills/`, `hooks/` | [V-lokal] |
| Session-Volldaten | `session-state/<session-id>/` mit `events.jsonl`, `workspace.yaml`, `plan.md`, `session.db`, `checkpoints/`, `files/` | [V-lokal] `help config`; Dateinamen lokal so vorgefunden |
| Session-Index (Suche/`/chronicle`) | `session-store.db` (SQLite, lokal mit `-wal`/`-shm`) | [V-lokal]; [V-doc] config dir reference: „SQLite database used by the CLI for cross-session data such as checkpoint indexing and search“ |
| Logs | `logs/` (`--log-dir` überschreibt) | [V-lokal] |
| Cache (verwerfbar) | `COPILOT_CACHE_HOME`, Default `~/.cache/copilot` (Linux) bzw. `%LOCALAPPDATA%\copilot` (Windows); **nicht** von `COPILOT_HOME` betroffen | [V-lokal] `help environment` |
| Memory | **kein** lokales Verzeichnis; Repo-Memories auf GitHub, eigene unter `https://github.com/settings/copilot/memory` | [V-lokal] `help config`, Setting `memory`; lokal kein `memories/`-Ordner vorhanden |

Lokal zusätzlich vorgefunden (nicht in der Doku-Liste): `sidebar-sessions-state/`, `open-sessions-state.json`, `command-history-state.json`, `restart/`, `servers/`, `ide/`. [V-lokal, nur Namen]

Weitere relevante Env/Settings:
- `COPILOT_HOME` ersetzt den kompletten `~/.copilot`-Pfad; `--config-dir` ist zugunsten von `COPILOT_HOME` deprecated. [V-doc] config dir reference; [V-doc] [CLI changelog](https://raw.githubusercontent.com/github/copilot-cli/main/changelog.md) (Eintrag „--config-dir is deprecated in favor of COPILOT_HOME“)
- `remoteExport`: „Export sessions to GitHub.“ [V-lokal] `help config`
- `remoteSessions: true` / `/remote on`: Remote-Steuerung über github.com und GitHub Mobile. [V-doc] [Steer remotely](https://docs.github.com/en/copilot/how-tos/copilot-cli/use-copilot-cli/steer-remotely) – Copilot App als Client dort **nicht erwähnt**.
- `sessionManagementTools.scope`: Peer-Zugriff nur auf Sessions, „sharing this configuration directory“ – also explizit an ein `COPILOT_HOME` gebunden. [V-lokal]
- `copilot sessions import <jsonl>` / `copilot memories import <jsonl>` existieren (Import „semantic session“/„semantic memories through the native memory service“). [V-lokal] Ein passender *Export*-Befehl wurde nicht gefunden. [nicht belegt]

## 2. Speicherorte Copilot App (Desktop)

- „Copilot CLI and the GitHub Copilot app store the complete record of each session under `~/.copilot/session-state/`“ + Teilmenge im lokalen SQLite-„session store“. [V-doc] [About CLI session data](https://docs.github.com/en/copilot/concepts/agents/copilot-cli/chronicle), [Session data](https://docs.github.com/en/copilot/concepts/security-governance-and-network-settings/session-data)
- Die App ist „built on GitHub Copilot CLI“. [V-doc] [About the GitHub Copilot app](https://docs.github.com/en/copilot/concepts/agents/github-copilot-app)
- Ein Windows-spezifischer Pfad wird für die App nicht genannt. Für die CLI unter Windows nennt die config dir reference Beispiele unter `C:\Users\YOUR-USER\.copilot\…`. → Windows-App vermutlich `%USERPROFILE%\.copilot\`. [Einschätzung, auf dem Arbeits-PC zu prüfen]
- Ob die App `COPILOT_HOME` respektiert: [nicht belegt].

## 3. Sync / Cross-Surface

- „By default, locally-run sessions created with Copilot CLI or the GitHub Copilot app are synced to your GitHub account.“ Abschalten über `remoteExport`. Bei Business/Enterprise muss die Policy „Store local sessions in the Cloud“ mindestens „View from cloud“ sein, sonst nur lokal. [V-doc] [About CLI session data](https://docs.github.com/en/copilot/concepts/agents/copilot-cli/chronicle)
  → **Arbeits-PC: Org-Policy entscheidet, ob Sync überhaupt geht.**
- `/chronicle`: „get insights from previous sessions, including work you started in the app“. [V-doc] [About the app](https://docs.github.com/en/copilot/concepts/agents/github-copilot-app). Löschen einer synchronisierten Kopie „removes the session from your `/chronicle` insights and query results“ → synchronisierte Sessions fließen in `/chronicle` ein. [V-doc] [Chronicle how-to](https://docs.github.com/en/copilot/how-tos/copilot-cli/use-copilot-cli/chronicle)
- `/chronicle reindex` baut den Index aus den **lokalen** `~/.copilot/session-state/`-Dateien neu und synct dabei zum Account. [V-doc] ebd.
- Session-Daten auf einen anderen Rechner bringen ist in den Docs nur als manuelles Kopieren + `reindex` beschrieben; ein „Pull synchronisierter Sessions von anderen Geräten“ ist **nicht beschrieben**. [V-doc / nicht belegt]
- CLI-Changelog 1.0.35 (2026-04-23): „Session sync prompt … explains GitHub.com cross-device sync“. [V-doc] [CLI changelog](https://raw.githubusercontent.com/github/copilot-cli/main/changelog.md)
- CLI-Changelog 1.0.112 (2026-11-13): „/chronicle cost-tips now includes Copilot App sessions“. [V-doc] ebd.
- „Sessions started in Copilot CLI now appear in your My work view“ (App) – taucht nur in Suchmaschinen-Zusammenfassungen auf; die Original-Changelog-Stelle habe ich **nicht** gefunden. [sekundär / unbestätigt]
- `copilot app` / `/app` öffnet die App im aktuellen Verzeichnis bzw. „the current session in the GitHub Copilot desktop app“ (ab App 1.1.3). [V-doc] CLI changelog 1.0.79/1.0.81; [V-lokal] `copilot app --help`. Ob das aus WSL heraus mit der Windows-App funktioniert: [nicht belegt].

## 4. Memory

- Copilot Memory: Repo-Fakten „shared with your collaborators“, nur im selben Repo; User-Präferenzen „follow you between repositories“. Unterstützt u. a. in Copilot app und Copilot CLI. [V-doc] [About Copilot Memory](https://docs.github.com/en/copilot/concepts/agents/copilot-memory)
- Speicherort serverseitig: lokale CLI-Hilfe: „Repository memories are kept on GitHub … view or delete them in its Settings > Copilot > Memory, and your own at https://github.com/settings/copilot/memory“. [V-lokal]
- CLI: interaktiv Default an (`memory`-Setting, `/memory on|off`), `-p` Default aus → `--enable-memory`. [V-lokal], [V-doc]
- App: „availability depends on whether Memory is available for your account“. [V-doc]
- → **Memory braucht kein gemeinsames Verzeichnis**; WSL-CLI und Windows-App sehen dieselben Memories, solange gleiches GitHub-Konto + (bei Repo-Memories) gleiches GitHub-Repo. [Einschätzung auf Basis V-doc/V-lokal]
- Relevanz für SecretAIry: Vault liegt evtl. nicht in einem GitHub-Repo → Repo-Memories greifen dann nicht, nur User-Präferenzen. [Einschätzung]

## 5. WSL-Support der App

- „On Windows, you can use the app to run agent sessions in a Windows Subsystem for Linux (WSL) 2 distribution.“ Voraussetzung: WSL 2, Repo/Ordner in der Distro, **separate GitHub-Anmeldung im WSL-Terminal** („Authentication inside WSL is separate from signing in to the app on Windows.“). Aktivierung: Experimental → **WSL hosts**, dann Distro unter **Environments** verbinden, pro Session im Environment-Dropdown wählen. [V-doc] [Agent sessions](https://docs.github.com/en/copilot/how-tos/github-copilot-app/agent-sessions)
- Changelog: „experimental support for Windows Subsystem for Linux (WSL)“. [V-doc] [Weekly releases Aug 24, 2026](https://github.blog/changelog/2026-08-28-github-copilot-weekly-releases-august-24)
- Technischer Unterbau vermutlich Agent Host Protocol (AHP): CLI hat `/ahp` („manage the hosts your sessions run on“), `/environments`, und laut Changelog `copilot --ahp-host`; 1.0.110: „cloud sessions and remote connections now require an AHP environment“. [V-lokal] `help commands`; [V-doc] CLI changelog. Dass die App-WSL-Hosts genau darüber laufen, ist **nicht dokumentiert**. [Einschätzung]
- Wo App-Sessions mit WSL-Host gespeichert werden (WSL-`~/.copilot` oder Windows-`~/.copilot`) und ob reine CLI-Sessions aus der WSL dann in der App-Sidebar auftauchen: **[nicht belegt]**. Separate WSL-Auth spricht dafür, dass in der Distro eine eigene CLI-Runtime mit eigenem Home läuft. [Einschätzung]

## 6. Gemeinsames Verzeichnis per `COPILOT_HOME`?

- Offiziell weder empfohlen noch verboten; config dir reference erwähnt nur Symlink für `settings.json` (Dotfiles). [V-doc]
- Drittquelle: Token-Optimizer-Release-Note – ein auf `/mnt/c/...` gesetztes `COPILOT_HOME` wurde von der native-Windows-CLI gelesen, die dann Logs auf einen nicht existierenden Pfad schrieb. [sekundär] [newreleases.io: token-optimizer v5.11.37](https://newreleases.io/project/github/alexgreensh/token-optimizer/release/v5.11.37)
- Drittquelle: Session-Store liegt in WSL unter `~/.copilot/session-store.db`, Windows-Binary sieht ihn ohne Pfadübersetzung nicht. [sekundär] [jongio/dispatch#8](https://github.com/jongio/dispatch/issues/8)
- WSL-UNC-Pfade: CLI-Changelog „Allow WSL UNC paths for local files“; Issue zu Worktree-Fehler unter `\\wsl.localhost` („dubious ownership“). [V-doc] CLI changelog; [github/copilot-cli#4383](https://github.com/github/copilot-cli/issues/4383)
- [Einschätzung] Ein über die OS-Grenze geteiltes `COPILOT_HOME` ist riskant: SQLite im WAL-Modus (`session-store.db-wal/-shm` lokal gesehen) über 9P/DrvFs mit gleichzeitigen Writern aus zwei OS ist ein klassischer Korruptions-/Lock-Kandidat; außerdem enthält `config.json` Auth-State und `mcp-config.json` OS-spezifische Pfade/Commands. Nicht empfohlen.

## Fazit für die Map

1. **Memory**: kein Problem – serverseitig, geteilt.
2. **`/chronicle` über beide Welten**: offiziell nur über **Cloud-Sync** (Default an, `remoteExport`; Org-Policy „Store local sessions in the Cloud“ muss es erlauben). Ob die Windows-App damit WSL-CLI-Sessions *lokal* in `/chronicle` einbezieht, ist nicht dokumentiert → auf dem Arbeits-PC testen.
3. **Saubere Alternative**: SecretAIry-Sessions **aus der App mit WSL-Host** starten statt parallel CLI in der WSL – dann ist die App der einzige Host. Status experimentell.
4. **Geteiltes `COPILOT_HOME`**: nicht vorgesehen, nicht empfohlen.

## Offen – nur auf dem Arbeits-PC klärbar (`Where: work-pc`)

- [ ] Org-Policy „Store local sessions in the Cloud“ – erlaubt? (Sync-Prompt / `/session info` / Mission Control-Status)
- [ ] Windows-App: Pfad des Session-Stores (`%USERPROFILE%\.copilot\session-state\` vorhanden?)
- [ ] WSL-CLI-Session starten (Sync an), danach in der Windows-App `/chronicle search <stichwort>` → taucht sie auf? Ebenso „My work“-Ansicht.
- [ ] App mit WSL host: landet `session-state/<id>` in WSL-`~/.copilot` oder Windows-`~/.copilot`? Sieht eine danach gestartete WSL-CLI (`/resume`) diese Session?
- [ ] Memory in App und WSL-CLI an? (`/memory`), Vault-Repo auf GitHub (für Repo-Memories)?

## Quellen

- https://docs.github.com/en/copilot/concepts/agents/copilot-cli/chronicle
- https://docs.github.com/en/copilot/how-tos/copilot-cli/use-copilot-cli/chronicle
- https://docs.github.com/en/copilot/concepts/security-governance-and-network-settings/session-data
- https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-config-dir-reference
- https://docs.github.com/en/copilot/concepts/agents/github-copilot-app
- https://docs.github.com/en/copilot/how-tos/github-copilot-app/agent-sessions
- https://docs.github.com/en/copilot/concepts/agents/copilot-memory
- https://docs.github.com/en/copilot/how-tos/copilot-cli/use-copilot-cli/steer-remotely
- https://github.blog/changelog/2026-08-28-github-copilot-weekly-releases-august-24
- https://raw.githubusercontent.com/github/copilot-cli/main/changelog.md
- https://github.com/github/copilot-cli/issues/4383
- Sekundär: https://github.com/jongio/dispatch/issues/8 · https://newreleases.io/project/github/alexgreensh/token-optimizer/release/v5.11.37
- Lokal: `copilot --help`, `copilot help config`, `copilot help environment`, `copilot help commands`, `copilot {app,sessions,memories} --help`, `ls ~/.copilot` (nur Namen)
