# Outlook-Zugang auf dem Arbeits-PC verifizieren

Type: task
Status: open
Where: work-pc
Blocked by: –

## Question

Trägt der empfohlene COM-Weg auf dem Arbeits-PC? Checkliste (HITL), Details in den Findings zu Ticket 04 (Branch `research/outlook-zugriff-m365`):

- [ ] Classic Outlook (nicht New Outlook) im Einsatz, keine anstehende Zwangsumstellung bekannt?
- [ ] Minimal-Skript (PowerShell oder pywin32): Posteingang und Team-Unterordner lesen, Antwortentwurf speichern, Kalendertermine lesen. Erscheint eine Object-Model-Guard-Abfrage?
- [ ] Aufruf aus WSL über `powershell.exe` funktioniert?
- [ ] Nebenbei: Ist Work IQ bzw. der M365-MCP im Tenant schon freigegeben? (Zukunftssicherer Austausch-Kandidat)

Ergebnis: COM tragfähig ja/nein, plus Einschränkungen. Keine Mail-Inhalte ins Repo.
