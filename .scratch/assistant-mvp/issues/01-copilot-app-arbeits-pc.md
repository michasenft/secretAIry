# Copilot App auf dem Arbeits-PC einsatzfähig?

Type: task
Status: open
Where: work-pc
Blocked by: –

## Question

Ist die Copilot App auf dem Arbeits-PC für den Pivot tragfähig? Checkliste (HITL):

- [ ] App installierbar/erlaubt (Policy im Firmen-Tenant, ggf. Copilot-CLI-Policy)?
- [ ] WSL-Ordner (`\\wsl$\…`) als „Open local folder“ einbindbar, Sessions darin laufen?
- [ ] Sind CLI-Sessions aus der WSL in der App sichtbar / per `/chronicle` abfragbar? (Aktuelle Beobachtung: nein → siehe Ticket 03)
- [ ] Environment „WSL hosts“ (experimentell) verfügbar? Session in der WSL-Distro starten; wo landet sie, ist sie per `/chronicle` sichtbar?
- [ ] Org-Policy „Store local sessions in the Cloud“ aktiv? Erscheinen WSL-CLI-Sessions danach in App/`/chronicle`?
- [ ] Läuft der firmeninterne Jira-MCP in der App (aus CLI-Config übernommen oder neu hinzugefügt)?
- [ ] Enterprise-Setting `strictPluginOnlyCustomization` aktiv? (Dann laufen Canvases nur als Plugin.)
- [ ] Kann eine Canvas-Extension aus der Windows-App den Vault unter `\\wsl.localhost\…` lesen und schreiben?
- [ ] `/create-canvas` verfügbar? Ein Test-Canvas lässt sich anlegen?

Ergebnis: Pivot hält / Fallback nötig, plus gefundene Fakten (Pfade, Einschränkungen) — ohne Firmennamen/URLs ins Repo.
