# Research: Canvas-Extensions der GitHub Copilot App

Ticket: [02-canvas-extensions](../issues/02-canvas-extensions.md) · Stand: 2026-11-21

Legende: **[V]** = in Primärquelle verifiziert (Quelle verlinkt) · **[A]** = aus Primärquellen-Code/Beispielen abgeleitet, nicht explizit dokumentiert · **[?]** = nicht angegeben / offen.

## Kurzantwort

Ja, für Inbox, ToDo und Briefing auf einem Markdown-Vault-Store reicht das technisch. Ein Canvas ist ein Node-Prozess (`extension.mjs`), der einen eigenen Loopback-HTTP-Server startet. Die App bettet dessen URL als iframe ein. Was der Prozess lesen und schreiben darf, entscheidet der Code, nicht ein Canvas-Framework. Der Vault ist also direkt per `node:fs` erreichbar. Die wichtigste Grenze: **Ein Canvas lebt nur innerhalb einer offenen Session.** Es gibt keinen session-losen oder Hintergrund-Canvas. Ein Agent-*Turn* ist dagegen nicht nötig: Die UI kann über den eigenen HTTP-Server direkt Dateien schreiben. Die API ist durchgehend als `@experimental` markiert.

## 1. Aufbau / Dateistruktur / Manifest

- **[V]** Jeder Canvas liegt in einem eigenen Unterordner von `.github/extensions/` (Repo-Scope) oder `~/.copilot/extensions/` (User-Scope). Üblicher Inhalt laut Doku: `package.json` (Metadaten/Dependencies), Entry-File `extension.mjs` und optional JSON-Artefakte, z. B. unter `artifacts/`. — https://docs.github.com/en/copilot/how-tos/github-copilot-app/working-with-canvas-extensions
- **[V]** Es gibt einen dritten Scope, **Session**: `$COPILOT_HOME/session-state/<sessionId>/extensions/<name>/`, der nur für die aktuelle Session geladen wird. Die Discovery scannt nur die *direkten* Unterordner dieser drei Orte. Die `extensionId` wird als `${source}:${name}` abgeleitet (z. B. `project:my-extension`, `session:<id>:<name>`). — `create-canvas`-Skill (die Quelle des Built-in-Skills `/create-canvas`): https://github.com/github/gh-aw/blob/main/.github/skills/create-canvas/SKILL.md
- **[V]** Das Entry-File heißt `extension.mjs`, `extension.cjs` oder `extension.js` (in dieser Priorität). `.mjs` wird empfohlen, TypeScript wird nicht unterstützt. Das SDK wird automatisch aufgelöst und nicht installiert. — https://github.com/github/copilot-sdk/blob/main/nodejs/docs/extensions.md
  - Widerspruch in den Quellen: Der gh-aw-Skill sagt „kein `package.json` für das SDK anlegen“. Die Doku-Seite und viele Beispiele in `github/awesome-copilot` haben aber eins (`"type": "module"`, `"main": "extension.mjs"`, `@github/copilot-sdk: latest`). Fazit **[A]**: Ein `package.json` ist optional und schadet nicht.
  - Auch der Paketname ist uneinheitlich: SDK-Doku `@microsoft/copilot-sdk/extension`, Skill und Beispiele `@github/copilot-sdk/extension`. **[?]** Welcher Name auf dem Arbeits-PC funktioniert, muss dort geprüft werden.
- **[V]** Es gibt ein optionales Sidecar `extension.json` (Version 1, `"initialization": "module"`, Arrays `tools`/`canvases`/`commands`, max. 256 KiB). Damit startet die Extension lazy, erst bei Bedarf, statt eager. Ohne die Datei startet sie eager. — extensions.md (siehe oben)
- **[V]** Ein Manifest im engeren Sinn ist nicht nötig, denn ein Canvas wird **im Code** deklariert: `joinSession({ canvases: [createCanvas({ id, displayName, description, icon?, inputSchema?, actions[], open, onClose? })] })`. — https://github.com/github/copilot-sdk/blob/main/nodejs/src/canvas.ts
- **[A]** Die offizielle `github-security`-Extension hat zusätzlich eine `manifest.json` (`id`, `engines: { "github-app": "^0.2.0" }`, `activations[{ owner: "session", entry: { node: "./extension.mjs" }, authentication: { resources: ["github"] } }]`). Dokumentiert ist dieses Format nirgends. — https://github.com/github/awesome-copilot/tree/main/extensions/github-security

## 2. Technologie / Architektur

- **[V]** Eine Extension ist ein von der CLI geforkter Node-Child-Prozess. Er spricht JSON-RPC über stdio, deshalb ist `stdout` tabu (**kein `console.log`**, stattdessen `session.log()`). — gh-aw SKILL.md, extensions.md
- **[V]** Rendering: `open()` liefert `{ url?, title?, status? }`. Die App rendert diese URL. Laut Skill bettet der Host **nur Loopback-URLs** ein. Das Muster ist ein HTTP-Server pro Instanz auf `127.0.0.1:<random port>`, Live-Updates in den iframe laufen per Server-Sent Events (`/events`). — `CanvasProviderOpenResult` in https://github.com/github/copilot-sdk/blob/main/nodejs/src/generated/rpc.ts, gh-aw SKILL.md
- **[V]** Die UI ist also normales HTML/JS/CSS im iframe. Die App spiegelt Theme-Tokens in das Dokument (`--background-color-default`, `--text-color-default`, `--font-sans`, `data-color-mode` …). — gh-aw SKILL.md
- **[V]** IDs: `canvasId` steht für den Typ, `instanceId` ist ein vom Aufrufer gewählter Handle für ein Panel (Regex `^[A-Za-z0-9][A-Za-z0-9._-]{0,127}$`). Ein erneutes `open` mit derselben `instanceId` fokussiert das Panel bzw. lädt den iframe neu. — gh-aw SKILL.md

## 3. Daten lesen/schreiben

- **[V]** Agent → Canvas: Der Agent bekommt die Tools `open_canvas`, `list_canvas_capabilities` und `invoke_canvas_action`. Die `actions[]` des Canvas (mit JSON-Schema, validiert per Ajv) sind die agent-callable Capabilities. Das Ergebnis des Handlers wird zum Tool-Result. Action-Namen dürfen nicht mit `canvas.` beginnen. — canvas.ts, gh-aw SKILL.md
- **[V]** Canvas → Agent: `session.send({ prompt, attachments })` / `session.sendAndWait()` schicken einen Prompt in die verbundene Session, `session.on(event)` lauscht auf Events. Eine Ziel-Session lässt sich dabei nicht angeben: Es geht immer an die gejointe Session. — gh-aw SKILL.md, https://github.com/github/app/issues/3030 (für `session.send` ohne Ziel-Session)
- **[V]** User → Daten ohne Agent: Die UI spricht mit dem eigenen HTTP-Server der Extension, der direkt `node:fs` nutzen kann. Ein Referenzbeispiel ist `daily-focus-board` in `github/awesome-copilot`: Board-Zustand liegt in einer JSON-Datei unter einem **beliebigen absoluten Pfad** (`input.stateFile`), UI und Agent lesen und schreiben dieselbe Datei (atomar über tmp + rename). — https://github.com/github/awesome-copilot/tree/main/extensions/daily-focus-board
- **[V]** Zusätzlich kann eine Extension über `joinSession` eigene **Tools** und **Hooks** registrieren (auch MCP-Server und Workflows, laut extensions.md). Die Tool-Namen müssen über alle geladenen Extensions eindeutig sein, sonst schlägt das Laden der zweiten Extension fehl. — extensions.md, gh-aw SKILL.md
- **[V]** Sensible Env-Variablen (z. B. `GITHUB_TOKEN`) werden aus Extension-Prozessen entfernt. Mit `requestedEnvironmentVariables` muss man sie anfragen, die Freigabe bestätigt der User per Prompt. — https://github.com/github/copilot-sdk/blob/main/nodejs/src/extension.ts
- **[V]/[?]** Dateien außerhalb von `artifacts/`: technisch kein Problem (siehe `daily-focus-board`). Zum Protected Mode sagt extensions.md wörtlich: Die automatisch hinzugefügten FS-Rechte gelten für „Agent file tools and newly launched shell commands … extension provider processes do not.“ **[?]** Ob Extension-Prozesse damit gar nicht sandboxed sind oder eigenen Regeln unterliegen, ist nicht eindeutig dokumentiert.
- **[?]** Zugriff vom Windows-Node-Prozess der App auf einen Vault im WSL-Dateisystem (`\\wsl$\…` bzw. `\\wsl.localhost\…`) ist nirgends erwähnt und muss auf dem Arbeits-PC getestet werden.

## 4. Persistenz über Sessions

- **[V]** Laut Skill ist `instanceId` das Panel, nicht die Daten, also darf persistenter State nie an `instanceId` hängen. Die empfohlenen Speicherorte:
  - pro Session: `session.workspacePath` (wird mit der Session aufgeräumt)
  - pro User/global: `$COPILOT_HOME/extensions/<extension-name>/artifacts/`
  - pro Dokument: stabiler `documentId`/Pfad über `input`
  - im Repo committet: sinnvoller Repo-Pfad
  — gh-aw SKILL.md
- **[V]** `open` kann nach einem Provider-Reconnect erneut kommen. Es muss deshalb idempotent sein und den State aus dauerhaftem Speicher rehydrieren. — gh-aw SKILL.md
- **[A]** Für SecretAIry heißt das: Der Vault-Pfad kommt per Config/`input`, der Canvas ist nur Ansicht und Editor. Das passt zur Map-Entscheidung „Store = Markdown im Vault, Canvas nur Ansicht“.

## 5. User-Scope vs. Repo-Scope

- **[V]** Repo: `.github/extensions/<name>/` (committet, fürs Team). User: `~/.copilot/extensions/<name>/` (persönlich, `COPILOT_HOME` überschreibt `~/.copilot`). Session: siehe oben. — Doku-Seite, gh-aw SKILL.md, https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-config-dir-reference
- **[V]** **Enterprise-Risiko:** Das Managed Setting `strictPluginOnlyCustomization` kann Standalone-Customizations aus User- und Projekt-Quellen sperren, ausdrücklich auch „executable canvas providers“. Erlaubt bleiben dann nur Plugins und Managed Settings, eingrenzbar über `strictKnownMarketplaces`. — https://github.blog/changelog/2026-11-16-control-where-github-copilot-customizations-come-from/
- **[A]** Konsequenz: Auf dem Arbeits-PC könnte ein User-Scope-Canvas blockiert sein, als Plugin ausgeliefert aber funktionieren. **[?]** Wie die Firmen-Policy aussieht, ist nur dort klärbar.

## 6. Nutzbar ohne laufende Agent-Session?

- **[V]** Nein, ohne Session geht es nicht. `joinSession()` wirft einen Fehler, wenn `SESSION_ID` fehlt („intended for extensions running as child processes of the Copilot CLI“), und hängt sich an die *foreground session*. Extensions werden bei `/clear` oder einem Wechsel der Foreground-Session neu geladen und beim CLI-Exit gestoppt (unter Windows sofort, ohne Grace Period). — extension.ts, extensions.md
- **[V]** Auch der Einstieg in der UI läuft über Sessions: Customize → Canvas → **New session** öffnet „a session with the canvas“, `/create-canvas` läuft „in a session“. — Doku-Seite
- **[V]** Ein Agent-*Turn* ist nicht nötig. Laut GitHub Blog können Interaktionen „sent back to the agents *or* processed locally by the canvas“ werden. — https://github.blog/ai-and-ml/github-copilot/how-to-build-interactive-experiences-with-canvases/
- **[A]** Für die Inbox heißt das: Eine Session muss offen sein (z. B. eine dauerhaft gepinnte „SecretAIry“-Session), danach geht die Eingabe ohne LLM-Aufruf direkt in den Vault. Ein globaler Hotkey oder Capture ganz ohne offene App ist mit Canvas nicht abbildbar. Dafür bleibt die geplante CLI `secretairy note`.
- **[V]** Ein Canvas kann keine *andere* lokale Session öffnen oder fokussieren. Der Feature-Request dazu wurde als „Not planned“ (stale) geschlossen. — https://github.com/github/app/issues/3030

## 7. Paketierung als Plugin

- **[V]** Plugins können Skills, Hooks, Custom Agents, MCP-Server und Canvas-Extensions bündeln. — https://docs.github.com/en/copilot/how-tos/github-copilot-app/customize-github-copilot-app
- **[V]** Im Legacy-Manifest ist `extensions` ein Component-Path-Feld („Path(s) to extension directories“; String, Array oder Objekt mit `paths`/`exclusive`). Manifest-Orte: `.plugin/plugin.json`, `plugin.json`, `.github/plugin/plugin.json`, `.claude-plugin/plugin.json`. Im Agent-Plugins-1.0-Manifest bedeutet `extensions` dagegen etwas anderes (Client-Daten nach Reverse-Domain-Namespace). — https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-plugin-reference
- **[A]** Für Agent Plugins 1.0 dokumentiert docs.github.com keinen Pfad für Extensions. Das Build-Script von `github/awesome-copilot` legt sie im ausgelieferten Plugin unter **`com.github.copilot/extensions/<name>/`** ab (Kommentar: „Copilot-specific content is discovered from com.github.copilot/“). Skills liegen unter `skills/`, MCP in der Root-`mcp.json`. — https://github.com/github/awesome-copilot/blob/main/.github/scripts/catalog/materialize-plugins.mjs
- **[V]** Installationsort: `~/.copilot/installed-plugins/<marketplace>/<plugin>/` bzw. `_direct/`. Ein Marketplace ist ein GitHub-Repo, eine andere Git-URL oder ein lokales Verzeichnis (`marketplace.json`). — cli-plugin-reference, https://docs.github.com/en/copilot/concepts/customizations/plugins
- **[V]** Alternativ lässt sich ein Canvas per Gist teilen (`share_extension` / `install_extension`, Manifest `copilot-extension.json` `{ name, version: 1 }`, ~1 MB pro Datei, ~5 MB gesamt). — gh-aw SKILL.md

## 8. Bewertung für Inbox / ToDo / Briefing

| Ansicht | Machbar? | Wie / Einschränkung |
|---|---|---|
| Inbox (Capture) | ja **[A]** | Formular im iframe → POST an den Loopback-Server → Markdown-Datei mit Frontmatter in den Vault. Setzt eine offene Session voraus. |
| ToDo | ja **[A]** | Liste aus dem Vault lesen, Status ändern per UI (direkt im FS) oder per Agent (`invoke_canvas_action`). Live-Update per SSE bzw. File-Watch. |
| Briefing | ja **[A]** | Ein Button ruft `session.send({ prompt: "/briefing" })` auf, oder der Agent schreibt das Ergebnis in den Vault und der Canvas rendert es. |
| Capture aus *jeder* Session | nicht per Canvas | Canvas-Actions gibt es nur auf einer offenen Instanz. Dafür braucht es ein MCP- oder Extension-Tool (gleiches Plugin). |

Risiken: experimentelle API (Breaking Changes möglich), Enterprise-Policy (`strictPluginOnlyCustomization`), WSL-Pfadzugriff ungetestet, uneinheitlicher SDK-Paketname.

## Offene Punkte (nur auf dem Arbeits-PC klärbar)

1. Kann der Extension-Prozess der Windows-App den Vault unter `\\wsl.localhost\…` lesen und schreiben?
2. Welche Managed Settings gelten (`strictPluginOnlyCustomization`, `strictKnownMarketplaces`)?
3. Welcher Importpfad funktioniert (`@github/copilot-sdk/extension` vs. `@microsoft/copilot-sdk/extension`)?
4. Wird ein Plugin mit `com.github.copilot/extensions/<name>/` aus einem lokalen Marketplace (dieses Repo) geladen?

## Quellen

- https://docs.github.com/en/copilot/how-tos/github-copilot-app/working-with-canvas-extensions
- https://docs.github.com/en/copilot/how-tos/github-copilot-app/customize-github-copilot-app
- https://docs.github.com/en/copilot/reference/github-copilot-app-reference/built-in-skills
- https://docs.github.com/en/copilot/concepts/customizations/plugins
- https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-plugin-reference
- https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-config-dir-reference
- https://github.com/github/gh-aw/blob/main/.github/skills/create-canvas/SKILL.md
- https://github.com/github/copilot-sdk/blob/main/nodejs/src/canvas.ts
- https://github.com/github/copilot-sdk/blob/main/nodejs/src/extension.ts
- https://github.com/github/copilot-sdk/blob/main/nodejs/src/generated/rpc.ts
- https://github.com/github/copilot-sdk/blob/main/nodejs/docs/extensions.md
- https://github.com/github/awesome-copilot/tree/main/extensions/daily-focus-board
- https://github.com/github/awesome-copilot/tree/main/extensions/github-security
- https://github.com/github/awesome-copilot/blob/main/.github/scripts/catalog/materialize-plugins.mjs
- https://github.com/github/app/issues/3030
- https://github.blog/ai-and-ml/github-copilot/how-to-build-interactive-experiences-with-canvases/
- https://github.blog/changelog/2026-11-16-control-where-github-copilot-customizations-come-from/
- Community-Beispiel (nicht primär, nur zur Bestätigung des Musters): https://github.com/microsoft/Build26-LAB502-make-github-copilot-work-your-way-custom-tools-context-and-workflows/blob/main/.github/extensions/lab502-community-canvas/extension.mjs
