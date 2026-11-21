# Was können Canvas-Extensions der Copilot App?

Type: research
Status: resolved
Blocked by: –

## Question

Wie sind Canvas-Extensions der Copilot App aufgebaut und wo sind ihre Grenzen? Konkret: Dateistruktur/Manifest, Technologie (HTML/JS?), wie ein Canvas Daten liest/schreibt (eigene Agent-Tools, MCP-Server, lokale Dateien außerhalb von `artifacts/`?), Persistenz über Sessions hinweg, User-Scope (`~/.copilot/extensions`) vs. Repo-Scope, ob ein Canvas ohne laufende Agent-Session nutzbar ist (z. B. Inbox-Eingabe), Paketierung als Plugin. Reicht das für Inbox-, ToDo- und Briefing-Ansichten auf einem Markdown-Store im Vault?

## Answer

Technisch reicht es für Inbox-, ToDo- und Briefing-Ansichten auf einem Vault-Store. Ein Canvas ist ein Node-Prozess (`extension.mjs`, Copilot SDK `joinSession({ canvases: [createCanvas(...)] })`), der einen eigenen localhost-Server startet. Die App bettet ihn als iframe ein. Die UI kann per `node:fs` beliebige Pfade lesen und schreiben (Referenz: `daily-focus-board` in awesome-copilot). Der Agent ruft Canvas-`actions[]` über `invoke_canvas_action` auf, die Gegenrichtung läuft über `session.send()`.
Grenzen: **Ohne offene Session läuft kein Canvas** (er braucht `SESSION_ID` und wird bei Session-Wechsel oder Exit beendet). Capture ohne App bleibt deshalb Sache von CLI bzw. MCP, Capture aus beliebigen Sessions braucht ein MCP- oder Extension-Tool. Die API ist `@experimental`. Die Enterprise-Setting `strictPluginOnlyCustomization` kann User- und Repo-Canvases sperren, ein Plugin-Paket wäre der Ausweg. Wo Extensions im neuen Plugin-Format (Agent Plugins 1.0) liegen, ist nicht offiziell dokumentiert.
Offen, nur auf dem Arbeits-PC klärbar: Policy und Zugriff auf `\\wsl.localhost` aus der Windows-App (ergänzt in Ticket 01).

Findings: Branch `research/canvas-extensions`, Datei `.scratch/assistant-mvp/research/02-canvas-extensions.md`
