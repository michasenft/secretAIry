# Research: Outlook-Zugriff mit M365-Firmenkonto ohne Tenant-Admin

Zu Ticket: [04-outlook-zugriff-m365](../issues/04-outlook-zugriff-m365.md) · Stand: 2026-11-21

Legende: **[V]** = in Primärquelle verifiziert (Quelle verlinkt) · **[NV]** = nicht verifiziert / nicht in Primärquelle gefunden · **[A]** = eigene Ableitung/Einschätzung

Anforderung: Inbox + ein Unterordner lesen, Antwort**entwürfe** speichern (nie senden), Kalender lesen (später ggf. Termin anlegen). Laufzeit: Arbeits-PC, Windows + WSL, Copilot App als Host, Outlook-Desktop vorhanden.

---

## TL;DR

| Option | Ohne Admin machbar? | Kalender | Aufwand | Hauptrisiko |
|---|---|---|---|---|
| **A. Classic Outlook COM** (pywin32 / PowerShell, aus WSL via `powershell.exe`/`python.exe`) | **Ja**, wenn Classic Outlook läuft | Ja | gering–mittel | Umstellung auf New Outlook (kein COM); Object-Model-Guard-Prompts je nach AV/GPO |
| **B. Graph, eigene App-Registrierung, delegated** | **Meist nein** (Default-Consent-Policy sperrt Mail/Calendars für User-Consent) | Ja | mittel | Admin-Consent nötig; Device Code oft per CA geblockt |
| **C. Graph über Microsoft-First-Party-Client-IDs** (Graph PowerShell, Office etc.) | technisch teils, aber undokumentiert/grau | Ja | gering | Policy-/Compliance-Verstoß, Security-Alerts, jederzeit abschaltbar |
| **D. Work IQ MCP / CLI** (offizieller M365-MCP, auch als Copilot-CLI-Plugin) | **Nein** (Admin-Consent + Tenant-Enablement + Billing) | Ja | sehr gering, *falls* schon freigeschaltet | Admin-Gate; Lizenz/Billing |
| **E. Agent 365 / Copilot Studio Mail-/Calendar-MCP** | Nein (Tenant-Infrastruktur, Copilot-Studio-Umgebung) | Ja | hoch | laut MS nur noch Backward-Compat |
| **F. Microsoft MCP Server for Enterprise** | irrelevant | – | – | nur Entra-Verzeichnisdaten, read-only |
| **G. Open-Source-Outlook-MCPs** | je nach Basis: COM-basierte ja, Graph-basierte nein | meist ja | gering | Code-Qualität/Lizenz (AGPL), ungewollte `send`-Tools |

**Empfehlung [A]:** MVP auf **Option A (Classic Outlook COM)** bauen, hinter einer schmalen eigenen Schnittstelle (`list_messages(folder)`, `get_message`, `create_reply_draft`, `list_events(range)`), damit später auf Work IQ (D) oder Graph (B) umgestellt werden kann. Parallel am Arbeits-PC prüfen, ob Work IQ im Tenant schon freigegeben ist – dann wäre D der zukunftssichere Weg und passt direkt zur Copilot App.

---

## Option B: Microsoft Graph mit delegierten Rechten (eigene App-Registrierung)

### Benötigte Graph-Scopes
| Scope | AdminConsentRequired (delegated) laut Permission-Referenz |
|---|---|
| `Mail.Read` | No [V] |
| `Mail.ReadWrite` (nötig für Entwürfe) | No [V] |
| `Calendars.Read` / `Calendars.ReadWrite` | No [V] |
| `User.Read`, `offline_access` | No [V] |

Quelle: [Graph permissions reference](https://learn.microsoft.com/en-us/graph/permissions-reference) (geprüft via Rohquelle [microsoft-graph-docs-contrib/concepts/permissions-reference.md](https://github.com/microsoftgraph/microsoft-graph-docs-contrib/blob/main/concepts/permissions-reference.md)).

API-Bausteine:
- Antwortentwurf: `POST /me/mailFolders/{id}/messages/{id}/createReply` → legt Draft an, Senden ist separater Call; least privileged: `Mail.ReadWrite` [V] — [message: createReply](https://learn.microsoft.com/en-us/graph/api/message-createreply?view=graph-rest-1.0)
- Unterordner: [mailFolder: list childFolders](https://learn.microsoft.com/en-us/graph/api/mailfolder-list-childfolders) (Seite existiert, Inhalt nicht einzeln gelesen [NV])
- Kalender: [List calendarView](https://learn.microsoft.com/en-us/graph/api/user-list-calendarview) (Inhalt nicht einzeln gelesen [NV])

### Der eigentliche Blocker: User-Consent-Policy
- „AdminConsentRequired: No“ heißt **nicht**, dass der User selbst zustimmen darf. Tenant-Policy entscheidet.
- Microsoft-managed Policy („Let Microsoft manage your consent settings“) ist **Default für neue Tenants** und schließt vom User-Consent u. a. aus: `Mail.Read`, `Mail.ReadWrite`, `Mail.ReadBasic`, `Calendars.Read`, `Calendars.ReadWrite`, `MailBoxFolder.*`, `MailBoxSettings.*`, plus `EWS/IMAP/POP/EAS.AccessAsUser.All` [V] — [Manage app consent policies](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/manage-app-consent-policies)
- Ausnahme: „Mail client policy“ erlaubt User-Consent für diese Scopes **nur** für eine feste Liste populärer Mail-Clients (Apple Mail, Spark, eM Client, Thunderbird, Android-Mail …) [V] — gleiche Quelle. Eine eigene App fällt nicht darunter.
- Policy wird von Microsoft automatisch nachgeschärft (zuletzt Juni/Juli 2026: Contacts/Tasks/People ergänzt; Quelle Message Center MC1304287, gesehen nur über Dritt-Mirror [itelio](https://www.itelio.com/en/microsoft-message-center/MC1304287)) [NV, Primärquelle nicht direkt erreichbar].
- Alternativen der Admins: „Do not allow user consent“ oder `microsoft-user-default-low` (nur verifizierte Publisher + als „low“ klassifizierte Rechte) [V] — [Configure user consent](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/configure-user-consent). Nur das Legacy-Setting `microsoft-user-default-legacy` („Allow user consent for apps“) würde Mail.ReadWrite für eine eigene App erlauben [V].
- **[A]** Ob der eigene Tenant noch auf „legacy“ steht, ist unbekannt → nur am Arbeits-PC testbar (Consent-Dialog zeigt dann „Need admin approval“ oder nicht). Wahrscheinlichkeit „geht ohne Admin“ in einem Konzern 2026: niedrig.

### App-Registrierung selbst
- Default: „By default in Microsoft Entra ID, all users can register applications“ — abschaltbar über **Users can register applications = No** [V] — [Delegate app management](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/delegate-app-roles). In Konzernen häufig abgeschaltet [A].
- Apps mit „Assignment required“ brauchen immer Admin-Consent [V] — [Configure user consent](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/configure-user-consent).

### Auth-Flow
- **Device code flow**: funktioniert technisch für Public Clients [V] — [Device authorization grant](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-device-code). Aber: „Microsoft recommends blocking device code flow wherever possible“, per Conditional Access sperrbar; Protocol-Tracking kann danach auch Refresh-Tokens blocken (`AADSTS530036`) [V] — [Authentication flows in CA](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-authentication-flows). **[A]** Device Code ist in Firmen-Tenants ein typischer Blocker.
- Besser auf Windows: **WAM-Broker** via MSAL Python (`enable_broker_on_windows=True`, Redirect-URI `ms-appx-web://microsoft.aad.brokerplugin/{client_id}`) [V] — [MSAL Python + WAM](https://learn.microsoft.com/en-us/entra/msal/python/advanced/wam). Erfüllt Device-Compliance-CA-Policies eher [A].
- Aus WSL: MSAL Python ≥ 1.33.0 hat `enable_broker_on_wsl` (gleiche Redirect-URI) [V] — [msal/application.py](https://github.com/AzureAD/microsoft-authentication-library-for-python/blob/dev/msal/application.py).

### Kalender
Ja, gleiche Mechanik, `Calendars.Read(Write)` — gleicher Consent-Blocker [V, s. o.].

### Aufwand
Mittel: App-Reg + MSAL + Token-Cache + Graph-Calls. Technisch sauberster Weg – aber nur mit Admin-Consent realistisch (Admin-Consent-Workflow „Request approval“ existiert, falls aktiviert [V] — [Configure user consent](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/configure-user-consent)).

---

## Option C: Graph über Microsoft-First-Party-Client-IDs

- **Microsoft Graph PowerShell** (`Connect-MgGraph -Scopes ...`, optional `-UseDeviceAuthentication`) nutzt standardmäßig die Microsoft-App „Microsoft Graph PowerShell“/„Microsoft Graph Command Line Tools“; Microsoft empfiehlt eigene App-Registrierung zur Eingrenzung [V] — [Graph PowerShell auth commands](https://learn.microsoft.com/en-us/powershell/microsoftgraph/authentication-commands?view=graph-powershell-1.0).
- Ob die Microsoft-managed Consent-Policy (Ausschluss von Mail/Calendars) auch für Microsoft-eigene Clients wie Graph PowerShell / Graph Explorer greift: **in Microsofts Doku nicht gefunden [NV]**. Praxis [A]: viele Admins blockieren „Microsoft Graph Command Line Tools“ explizit oder setzen „Assignment required“.
- Nutzung fremder First-Party-Client-IDs (z. B. Office-/Teams-Clients mit vorautorisierten Scopes) für eigene Tools: **von Microsoft nicht als unterstützter Weg dokumentiert [NV]**. **[A]** Grauzone bis Policy-Verstoß, triggert ggf. Defender/Entra-ID-Protection-Erkennungen (Token-Abuse-Muster), kann jederzeit brechen. Für ein Tool, das im öffentlichen Repo liegt und am Firmenrechner läuft: **nicht empfehlen**.
- Aufwand gering, Kalender ja – aber Risiko unverhältnismäßig.

---

## Option A: Classic Outlook Desktop per COM (Outlook Object Model)

### Funktionsweise
- Outlook Object Model über COM: Python (`pywin32`, `win32com.client.Dispatch("Outlook.Application")`) oder PowerShell (`New-Object -ComObject Outlook.Application`). Ordner: `NameSpace.GetDefaultFolder(olFolderInbox)` + `.Folders["Unterordner"]` [V für Existenz der API: [NameSpace.GetDefaultFolder](https://learn.microsoft.com/en-us/office/vba/api/outlook.namespace.getdefaultfolder)].
- Antwortentwurf: `MailItem.Reply()` → `.Body`/`.HTMLBody` setzen → `.Save()` (landet in Entwürfe) [A; Methodenseite [MailItem.Reply](https://learn.microsoft.com/en-us/office/vba/api/outlook.mailitem.reply(method)) existiert, Inhalt nicht gelesen]. `Send` wird nie aufgerufen.
- Kalender: `GetDefaultFolder(olFolderCalendar)`, `Items.Restrict` mit `IncludeRecurrences` [A, Standard-OOM].
- Kein Entra-/Graph-Kontakt, keine App-Registrierung, kein Consent – nutzt das laufende, bereits angemeldete Outlook-Profil [A].

### Aufruf aus WSL
- WSL kann Windows-Programme direkt starten (`powershell.exe`, `python.exe`, `cmd.exe /C`), sie laufen „as the active Windows user“, stdout/Pipes funktionieren [V] — [WSL: Working across file systems / Interop](https://learn.microsoft.com/en-us/windows/wsl/filesystems).
- Ein Linux-Python in WSL kann **kein** COM direkt nutzen [A] → Varianten:
  1. MCP-Server in WSL ruft pro Aktion `powershell.exe -NoProfile -File outlook.ps1 ...` bzw. Windows-`python.exe outlook_com.py ...` auf, Rückgabe als JSON auf stdout. Overhead pro Call: Prozessstart (Sekundenbereich bei PowerShell) [A].
  2. **Besser [A]:** Den Outlook-Teil gleich als Windows-Python-Prozess laufen lassen (Copilot App läuft unter Windows und kann einen Windows-MCP-Server per stdio starten). Kein WSL-Umweg.
- Interop kann pro Session deaktiviert werden; per Default aktiv [V] — gleiche Quelle. Ob die Firma WSL-Interop per Policy abschaltet: [NV].

### Typische Blocker
- **Object Model Guard**: Zugriff auf `MailItem.Body`, `HTMLBody`, `SenderEmailAddress`, `To`, `Recipients`, `AppointmentItem.Body`, `RequiredAttendees` u. a. ist geschützt [V] — [Protected properties and methods](https://learn.microsoft.com/en-us/office/vba/outlook/how-to/security/protected-properties-and-methods). Out-of-process-Caller laufen **ohne Prompt**, wenn Windows Security Center ein AV mit Status „Good“ meldet **und** keine GPO „Programmatic Access“ auf „always warn“ steht [V] — [Security behavior of the OOM](https://learn.microsoft.com/en-us/office/vba/outlook/how-to/security/security-behavior-of-the-outlook-object-model). User ohne lokale Adminrechte können die Einstellung nicht ändern [V]. **[A]** Mit Defender/Firmen-AV meist prompt-frei; harter Blocker nur bei restriktiver GPO → am Arbeits-PC testen.
- **New Outlook für Windows unterstützt kein COM und kein VBA** [V] — [Identify COM add-ins](https://learn.microsoft.com/en-us/microsoft-365-apps/outlook/get-started/state-of-com-add-ins) (COM-Add-ins „maintenance mode“, „New Outlook for Windows doesn't support VBA“). Dass auch externe COM-Automation entfällt, bestätigen OSS-Projekte („olk.exe does not support COM“, s. u.) [NV in MS-Primärquelle explizit, aber konsistent].
- **Migrationszeitplan**: Stufen Opt-in → Opt-out → Cutover; für Enterprise je mind. 12 Monate Vorankündigung; Classic Outlook (bestehende Installationen) „supported until at least 2029“ [V laut Suchauszug der Seite, Seite selbst nicht vollständig gelesen] — [Stages of migration to new Outlook](https://learn.microsoft.com/en-us/microsoft-365-apps/outlook/get-started/guide-product-availability). Konkrete Opt-out-Daten für Enterprise widersprüchlich (Q&A/Blogs nennen 2026/2027) [NV].
- Outlook muss laufen bzw. wird per COM gestartet; Elevation-Mismatch (Outlook nicht elevated, Skript elevated o. ä.) bricht COM-Zugriff [A, NV].
- Firmen-Compliance: automatisierter Mailbox-Zugriff + Weitergabe an LLM (Copilot) ist eine Policy-Frage, keine technische [A].

### Kalender
Ja, voll (lesen; Termin anlegen als `AppointmentItem.Save()` ohne Versand möglich) [A].

### Aufwand
Gering–mittel. ~200–400 Zeilen Python/PowerShell für die vier Operationen [A]. Kein Admin.

---

## Option D: Work IQ MCP / Work IQ CLI (offizielles Microsoft-Angebot)

- Work IQ MCP: ein MCP-Endpoint mit generischen Tools (`fetch /me/messages`, `create_entity /me/events`, `do_action`, `ask` …), Entra-Auth, Discovery über `/.well-known/oauth-protected-resource` [V] — [Work IQ MCP overview](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/work-iq/mcp/overview).
- Work IQ CLI/MCP lokal: `npx -y @microsoft/workiq mcp`, Plattformen inkl. „WSL with browser support“, Installation auch als **GitHub-Copilot-CLI-Plugin** (`/plugin install workiq@copilot-plugins`) [V] — [Work IQ CLI](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/work-iq/cli). Passt damit 1:1 zur Copilot App als Host [A].
- **Voraussetzungen [V]** (gleiche Quelle + [Enable Work IQ](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/work-iq/enable-work-iq) + [Work IQ permissions](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/work-iq/permissions)):
  - „Administrative consent for the Work IQ application in your Microsoft Entra tenant“; alle Scopes (`WorkIQAgent.Ask`, `WorkIQ.Act`, `WorkIQ.Read`) `AdminConsentRequired: Yes`.
  - Global Admin für einmaliges Tenant-Enablement (Service Principal `fdcc1f02-fc51-4226-8753-f668596af7f7`).
  - Usage-based Billing-Plan in Copilot Studio mit Azure-Subscription; User dem Plan zugewiesen.
  - Ob zusätzlich eine M365-Copilot-Lizenz pro User nötig ist: widersprüchliche Angaben in Sekundärquellen [NV].
- Repo [github.com/microsoft/work-iq](https://github.com/microsoft/work-iq): „If you are not an administrator, contact your tenant administrator to grant access.“ [V]
- Entwürfe: generisch über `create_entity`/`do_action` auf Graph-Pfaden denkbar; ob die CLI-Variante (`workiq mcp`) dieselben Entity-Tools exponiert oder nur `ask`: [NV]. Policy-Layer kann Pfade/Methoden einschränken [V, „Policy over scopes“].
- Kalender: ja [V, Beispiel `create_entity /me/events`].
- Aufwand: nahezu null, **falls** der Tenant es bereits freigegeben hat. Sonst Admin-Thema.

## Option E: Agent 365 / Copilot Studio MCP-Server (Mail, Calendar)

- Tenant-URL `https://agent365.svc.cloud.microsoft/agents/tenants/{tenantId}/servers/mcp_MailTools`; Tools u. a. `createMessage` (Draft), `getMessage`, `searchMessages`, `reply`/`replyAll` (senden direkt!), `sendMail`; Preview [V] — [Mail reference (Copilot Studio)](https://learn.microsoft.com/en-us/microsoft-agent-365/mcp-server-reference/mail).
- Microsoft: diese app-spezifischen M365-MCP-Tools sind „only intended for backward compatibility … shouldn't be used in … new projects“ → Work IQ nutzen [V] — gleiche Quelle.
- Kein Draft-Reply-Tool, kein Ordner-Listing in der Liste; `reply` sendet sofort [V] → widerspricht „nur Entwurf“.
- Copilot-Studio-Agenten von außen aufrufen (Direct Line / Agents SDK): erfordert Copilot-Studio-Umgebung, Lizenz, Connector-/DLP-Freigaben [NV, nicht vertieft]. **[A]** Für das MVP irrelevant.

## Option F: Microsoft MCP Server for Enterprise

`https://mcp.svc.cloud.microsoft/enterprise`, Preview, **nur Entra-Identity-/Verzeichnisdaten, read-only** [V] — [Overview](https://learn.microsoft.com/en-us/graph/mcp-server/overview). Kein Mail/Kalender → irrelevant.

## Option G: Open-Source-Outlook-MCP-Server

| Projekt | Basis | Mail/Drafts | Kalender | Hinweise |
|---|---|---|---|---|
| [Aanerud/outlook-desktop-mcp](https://github.com/Aanerud/outlook-desktop-mcp) | COM, pywin32, Python 3.12+ | `list_emails` (beliebiger Ordner), `list_folders`, `read_email`, `reply_email`; **kein** explizites Draft-Tool, dafür `send_email` | ja (list/create/update) | Classic Outlook muss laufen; „new Outlook (olk.exe) does not support COM“; AGPL-3.0 [V, README] |
| [Softeria/ms-365-mcp-server](https://github.com/Softeria/ms-365-mcp-server) | Graph, Device Code; Default eigene Softeria-Multi-Tenant-App, alternativ `MS365_MCP_CLIENT_ID` | `list-mail-messages`, `create-draft-email` (inkl. Reply-Draft), `send-mail` | ja (`get-calendar-view`) | `--org-mode` für Work-Accounts; unterliegt exakt dem Consent-Blocker aus B (Fremd-App, nicht verified-publisher-geprüft [NV]) [V, README teilweise] |
| [MagoDev/outlook-mcp](https://github.com/MagoDev/outlook-mcp) | COM (Fork marlonluo2018/outlook-mcp-server) | „draft-first guardrail“ | [NV] | nur Suchsnippet [NV] |
| outlook-classic-mcp (PyPI) | COM | [NV] | [NV] | PyPI-Seite nicht ladbar [NV] |

**[A]** Als Referenz/Spickzettel nützlich; selbst übernehmen eher nicht (AGPL, `send`-Tools vorhanden, Vertrauen in Fremdcode bei Firmen-Mailbox). Eigene schmale COM-Schicht mit hartem „kein Send“ ist kleiner und auditierbar.

---

## Prüfliste für den Arbeits-PC (Where: work-pc)

1. Outlook-Variante: Classic (`OUTLOOK.EXE`) oder New (`olk.exe`)? Gibt es schon Opt-out-Hinweise der IT?
2. COM-Smoke-Test aus WSL: `powershell.exe -NoProfile -Command "(New-Object -ComObject Outlook.Application).GetNamespace('MAPI').GetDefaultFolder(6).Items.Count"` → Zahl ohne Security-Prompt? Danach Zugriff auf `.Body` eines Items (geschützte Property) testen.
3. Trust Center → Programmatic Access: welcher Modus, grau (GPO)?
4. Work IQ: `npx -y @microsoft/workiq mcp` bzw. Copilot-CLI-Plugin – kommt Consent-Dialog „Admin approval required“ oder läuft es? (Kein Schaden, nur Login.)
5. Optional Graph: `entra.microsoft.com` → App registrations → „New registration“ erlaubt? Consent-Dialog für `Mail.ReadWrite` zeigt „Need admin approval“?
6. Interne IT-Richtlinie zu Mail-Automation/LLM-Zugriff klären.

## Offene Punkte / nicht verifiziert
- Ob Microsoft-managed Consent-Policy auch Microsoft-First-Party-Clients (Graph PowerShell/Explorer) betrifft.
- Genaue Enterprise-Opt-out-/Cutover-Termine für New Outlook.
- Ob Work-IQ-CLI-MCP Draft-Erstellung (`create_entity`/`createReply`) exponiert und ob M365-Copilot-Lizenz pro User Pflicht ist.
- COM-Verhalten bei Elevation-Mismatch und unter firmenspezifischer GPO.
