# Map: Assistant MVP

Labels: wayfinder:map

## Destination

MVP-Spec für SecretAIry als **Copilot-App-Plugin** (MCP-Server + Skills + Canvases + Vault-Store): Notizen/Inbox, ToDos und On-Demand-Briefing inkl. Target-Date-Check. Mail-Triage ist mitgeplant, aber nicht MVP-blockierend. Fallback, falls der Pivot scheitert: eigene Browser-App (localhost).

## Notes

- Domäne: persönlicher PO-Assistent; Vokabular siehe [GLOSSARY.md](../../GLOSSARY.md). Skills: `grilling` + `domain-modeling` für Grilling-Tickets.
- **Copilot ist Host und Gehirn** (Copilot App, gebaut auf der Copilot CLI); SecretAIry liefert nur Plugin-Bestandteile. Sessions/Kontext kommen über Copilot (`/chronicle`, Memory), Primärquelle für Wissen ist der Vault.
- **Laufzeit nur auf dem Arbeits-PC** (Windows + WSL, Copilot App unter Windows, WSL-Ordner eingebunden, Outlook-Desktop mit M365, eigene Jira-Instanz per firmeninternem MCP). Dieser Rechner hier: nur Planung/Entwicklung.
- Tickets mit `Where: work-pc` sind nur auf dem Arbeits-PC klärbar; Übergabe privat ↔ Arbeit läuft über dieses Repo.
- **Öffentliches Repo:** keine Arbeitsdaten, keine Firmen-Konfiguration/-Namen, keine Credentials im Repo. Firmenspezifisches nur in lokaler Config.
- Autonomie: nur **Vorschläge**, jede Aktion einzeln bestätigt; Mails nur als Entwurf, nie senden.
- Capture-Wege MVP: Canvas „Inbox“, MCP-Tool aus jeder Copilot-Session, Standalone-CLI `secretairy note "…"` (ohne LLM).
- Store: Markdown mit Frontmatter in einem Vault-Ordner (Source of Truth); Canvas ist nur Ansicht.
- Briefing im MVP on-demand (`/briefing`).

## Decisions so far

<!-- one line per closed ticket: - [title](issues/NN-slug.md): gist -->

- [Copilot-Sessions über Windows ↔ WSL teilen?](issues/03-session-store-wsl-windows.md): getrennte Stores; Brücken sind Cloud-Sync (Policy nötig) und geteiltes Memory; am saubersten sind die „WSL hosts“ der App (experimentell)
- [Was können Canvas-Extensions der Copilot App?](issues/02-canvas-extensions.md): reicht für Inbox, ToDo und Briefing auf dem Vault (Node-Extension mit eigenem localhost-Server, fs-Zugriff); Canvas läuft nur bei offener Session, API experimentell, ggf. als Plugin wegen Enterprise-Policy
- [Wie kommt der Assistent an Outlook (M365-Firmenkonto)?](issues/04-outlook-zugriff-m365.md): Classic Outlook per COM als Windows-MCP (lesen, Entwürfe, Kalender, nie senden); Graph und Work IQ scheitern ohne Admin

## Not yet specified

- Einsortieren-Ziele **Termin** und **Kontaktaufnahme**: Kalender ist per COM erreichbar (Ticket 04). Ob Termine angelegt oder nur vorgeschlagen werden und wie „Kontakt“ aussieht (Mail-Entwurf? Teams?), ist noch offen.
- **Automatischer Morgen-Lauf** des Briefings: lokales Scheduling (Task Scheduler + `copilot -p`? `/every`?), Cloud-Automations erreichen lokale Quellen nicht.
- Globaler Windows-**Hotkey** für Capture.
- **Plugin-Packaging/Distribution** (Plugin-Struktur, Installation auf dem Arbeits-PC aus diesem Repo).
- **Bestehender Code**: was wird übernommen (Jira-Client, MCP-Gerüst, CLI), was fliegt raus, was passiert mit den Alt-Daten (Topics).
- Details der **Fallback-Browser-App**, falls der Pivot scheitert.

## Out of scope

- Jira-Darstellung offener Epics/Storys: läuft als eigene Baustelle in einer anderen Session.
- Capture übers private Handy: Firmendaten auf Privatgerät.
- Automatisches Senden von Mails.
