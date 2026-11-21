# Copilot-Sessions über Windows ↔ WSL teilen?

Type: research
Status: resolved
Blocked by: –

## Question

Wo speichern Copilot CLI und Copilot App ihre Session-Historie, Memory und Config (Pfade, Env-Variablen wie ein konfigurierbares Copilot-Home), und lässt sich das so einrichten, dass die Windows-App die Sessions der CLI in der WSL sieht (`/chronicle`, Memory)? Gibt es offizielle Wege (gemeinsames Verzeichnis, Remote-/WSL-Support der App, Sync über GitHub), oder ist das nicht vorgesehen?

## Answer

Getrennte Stores: CLI und App speichern Sessions unter `$COPILOT_HOME` (Default `~/.copilot`: `session-state/<id>/` + `session-store.db`). WSL-CLI und Windows-App haben also je einen eigenen Store, ein gemeinsames Verzeichnis ist nicht vorgesehen (über 9P/DrvFs wegen SQLite riskant, Einschätzung).
Brücken: (1) Cloud-Sync der Sessions zum GitHub-Account (`remoteExport`, Business/Enterprise braucht Org-Policy „Store local sessions in the Cloud“), fließt laut Docs in `/chronicle` ein. Ob die App damit WSL-Sessions sieht, ist nicht belegt. (2) Memory liegt serverseitig und wird bei gleichem Konto automatisch geteilt. (3) **Sauberster Weg:** Die App führt Sessions experimentell direkt in einer WSL-2-Distro aus („WSL hosts“). Dann ist die App der einzige Host. Wo diese Sessions gespeichert werden, ist nicht dokumentiert.
Offen, nur auf dem Arbeits-PC klärbar: Checkliste in den Research-Findings, ergänzt in Ticket 01.

Findings: Branch `research/session-store-wsl-windows`, Datei `.scratch/assistant-mvp/research/03-session-store-wsl-windows.md`
