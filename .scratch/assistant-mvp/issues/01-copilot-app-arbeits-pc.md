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
- [ ] Läuft der firmeninterne Jira-MCP in der App (aus CLI-Config übernommen oder neu hinzugefügt)?
- [ ] `/create-canvas` verfügbar? Ein Test-Canvas lässt sich anlegen?

Ergebnis: Pivot hält / Fallback nötig, plus gefundene Fakten (Pfade, Einschränkungen) — ohne Firmennamen/URLs ins Repo.
