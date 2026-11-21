# Wie kommt der Assistent an Outlook (M365-Firmenkonto)?

Type: research
Status: resolved
Blocked by: –

## Question

Welche Wege gibt es, aus einem lokalen Tool (Python/MCP, Windows + WSL) Mails aus Posteingang und einem Unterordner zu lesen und Antwortentwürfe abzulegen, mit M365-Firmenkonto und **ohne** Tenant-Admin-Rechte? Kandidaten: Microsoft Graph mit delegierten Rechten (App-Registrierung nötig? User-Consent erlaubt?), Outlook-Desktop-Automation per COM/PowerShell (auch aus WSL aufrufbar?), Microsoft 365 Copilot / Agents / Work IQ-ähnliche MCP-Angebote von außen, fertige Outlook-MCP-Server. Je Weg: Voraussetzungen, typische Firmen-Blocker, Kalender-Zugriff gleich mit (für Termine)?

## Answer

Empfehlung: **Classic Outlook per COM** (pywin32 oder PowerShell, auf Windows bzw. aus WSL über `powershell.exe`/Windows-`python.exe`). Kommt ohne Admin aus und deckt Posteingang, Unterordner, Antwortentwürfe (`Reply().Save()`) und Kalender ab. Am einfachsten läuft es als Windows-MCP-Prozess unter der Copilot App, hinter einer schmalen eigenen Schnittstelle (list/get/create_reply_draft/list_events, **ohne Senden**).
Verworfen bzw. nur Plan B:
- **Graph mit eigener App-Registrierung:** Die Microsoft-managed Consent-Policy schließt Mail- und Kalender-Scopes vom User-Consent aus, Device Code wird oft per Conditional Access geblockt.
- **Work IQ / M365-MCP:** braucht Admin-Consent, Tenant-Enablement und einen Billing-Plan. Die Agent-365-Mail-MCPs haben nur Senden, keine Entwürfe.
- **Fremde Microsoft-Client-IDs:** Grauzone mit Compliance-Risiko.
- **Open-Source-Outlook-MCPs:** AGPL und mit Senden-Tools, taugen nur als Referenz.

Risiken: New Outlook kann kein COM (Classic wird laut Microsoft bis mindestens 2029 unterstützt). Je nach GPO oder AV kann der Object Model Guard Abfragen auslösen. Ob Work IQ im Tenant freigegeben ist, zeigt Ticket 11.

Findings: Branch `research/outlook-zugriff-m365`, Datei `.scratch/assistant-mvp/research/04-outlook-zugriff-m365.md`
