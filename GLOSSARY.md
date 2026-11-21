# SecretAIry

Persönlicher Arbeits-Assistent eines Product Owners: fängt Gedanken auf, sortiert sie in Jira, ToDos, Termine und Mails ein und erstellt ein tägliches Briefing.

## Erfassen und Einsortieren

**Notiz**:
Ein schnell erfasster, noch unsortierter Gedanke, z. B. mitten in einem Call. Rohmaterial, das später einsortiert wird.
_Avoid_: Memo, Idee, Topic

**Inbox**:
Die Menge aller Notizen, die noch nicht einsortiert sind.
_Avoid_: Eingang, Backlog

**Einsortieren**:
Eine Notiz in ihre Zielform überführen: Annotation, neues Epic/neue Story, ToDo, Termin, Mail-Entwurf oder Kontaktaufnahme. Danach verlässt die Notiz die Inbox.
_Avoid_: Triage (reserviert für Mails), Verarbeiten

## Jira

**Annotation**:
Eine lokale, private Anmerkung zu einem Jira-Epic oder einer Jira-Story, die nur im Assistenten sichtbar ist.
_Avoid_: Kommentar, Notiz

**Kommentar**:
Ein öffentlicher Kommentar direkt im Jira-Issue, für alle Kollegen sichtbar. Entsteht nur auf ausdrückliche Bestätigung, z. B. aus einer Annotation.
_Avoid_: Annotation

**Target-Date-Check**:
Prüfung der Team-Epics und -Storys darauf, ob das Target End in unter einer Woche erreicht oder schon überschritten ist und ob es dazu schon einen aktuellen Kommentar gibt.

## Aufgaben und Tagesablauf

**ToDo**:
Eine Aufgabe, die bis zu einem Zeitpunkt erledigt sein soll und für die es kein eigenes Jira-Issue gibt.
_Avoid_: Task, Aufgabe, Topic

**Briefing**:
Die tägliche Zusammenfassung aus den neuen relevanten Mails, fälligen ToDos, offenen Notizen in der Inbox und den Treffern des Target-Date-Checks, jeweils mit vorgeschlagenen Aktionen.
_Avoid_: Report, Status

**Vorschlag**:
Eine vom Assistenten vorbereitete Aktion (Kommentar, Datumsänderung, Entwurf, Termin …), die erst nach ausdrücklicher Bestätigung ausgeführt wird.

## Mail

**Mail-Triage**:
Die Einstufung jeder neuen Mail aus Posteingang und Team-Ordner als „Aktion nötig“ oder „nur Info“.
_Avoid_: Einsortieren

**Entwurf**:
Eine vom Assistenten vorbereitete Antwortmail, die als Entwurf in Outlook abgelegt und nie automatisch gesendet wird.
