# CDS Bot — Störung 29.8.2026: Diagnose, Fixes & Plan Telefon-Reservierungen

**Datum:** 30.8.2026
**Instanz:** https://n8n.gevio.cloud
**Betroffene Workflows:** CDS-RESERVATION-BOT (`rBoD3Cu0UM9WiT3N`), CDS-ORDER-BOT (`I25coJPmjjxIzZVZ`), CDS-VAPI_FORWARDER (`6NmOnAM7cgsRXtOh`), CDS-DISPATCHER (`oaxcHyyQZ5zHFNvr`)

---

## 1. Störungsbild (29.8.2026)

Drei Kunden-Chats, gleiches Muster: Der „Bot" schrieb eine erste Nachricht, auf
Kunden-Antworten kam **nichts mehr**. Ein Kunde erhielt eine englische Nachricht
(„I have noted your reservation for tomorrow at 10 AM …") — **ohne dass eine
Reservierung angelegt wurde**; die angekündigte Bestätigungsmail kam nie an.

## 2. Diagnose (per n8n-API verifiziert)

**Hauptursache: OpenRouter-Guthaben leer (HTTP 402 „Payment required").**
- Letzte erfolgreiche RESBOT-Execution: 28.8., 17:41 UTC. Ab 29.8. früh nur noch
  Errors — jede Kundennachricht (19:28, 19:40, 15:31, 10:48, 11:48, 11:49 MESZ)
  matcht 1:1 eine Error-Execution am Node `OpenRouter Chat Model`.
- **CDS-ORDER-BOT ebenfalls betroffen** (Success bis 29.8. 08:45 UTC, danach Errors).
- Fix: Guthaben aufgeladen → Test-Execution über Live-Webhook (Fake-Nummer
  `49000000000000`) am 30.8. erfolgreich: Dispatcher ✓, RESBOT ✓, OpenRouter ✓,
  AI-Antwort „OK.", WA-Versand-Node ✓.

**Zweitbefund: Die Eröffnungsnachrichten kamen nicht vom WhatsApp-Bot.**
Sie matchen exakt drei CDS-VAPI_FORWARDER-Executions (08:24, 13:30, 17:27 UTC) —
Nachrichten nach Telefonanrufen. Erklärt:
- **Englisch:** VAPI-`language`-Parameter des Anrufs.
- **Halluzinierte Reservierung:** Die `whatsapp_message`-Guidance des VAPI-Tools
  `send_whatsapp` erlaubte Formulierungen wie „Ich habe notiert …", obwohl der
  Forwarder keinerlei Schreibvorgang ausführt.

## 3. Bereits umgesetzte Änderungen (30.8.2026)

### 3.1 Neuer Workflow „CDS Error Alarm" (`sJREPHKmhkENsUF7`, aktiv)
- Nodes: **Error Trigger → Telegram Alarm**
- Telegram-Credential vom RESBOT übernommen (`KA8jIrIBZEerUBrL`), Chat `1313007318`
- Meldung enthält: Workflow-Name, fehlgeschlagener Node, Fehlermeldung, Execution-URL
- **Verknüpft als Error-Workflow** (`settings.errorWorkflow`) bei:
  - CDS-RESERVATION-BOT ✓
  - CDS-ORDER-BOT ✓
  - CDS-VAPI_FORWARDER ✓
- Hinweis: Bei ORDER-BOT und VAPI_FORWARDER ließ sich `callerPolicy` per API nicht
  explizit setzen (Default `workflowsFromSameOwner` greift — kein
  Verhaltensunterschied).
- **Arbeitsregel (bestehend):** Nach API-Änderungen F5 im n8n-Editor drücken, sonst
  kann der Editor die Änderung beim nächsten Speichern überschreiben.

## 4. Genehmigter Plan: Telefon-Reservierungen ohne Halluzinationen (Variante B)

**Grundprinzip:** Die Brücke Telefon → WhatsApp → echte Buchung **existiert bereits**
(Store Context → `vctx_<phone>` im Dispatcher → TELEFONAT-KONTEXT-Injektion →
RESBOT mit echtem Submit-Tool `Submit_Reservation_Form` → WPForms 15251 →
Bestätigungsmail → CDSTable_01 → Notion + Google Calendar). Sie scheiterte nur an
der Formulierung. Buchung bleibt beim WhatsApp-Bot; das Telefonat bleibt kurz
(40s-Philosophie) und liefert strukturierte Daten.

### B.1 VAPI: Tool `send_whatsapp` (`e45275e0-ffca-47ea-a869-43284ff43b41`) patchen
- Neue **optionale** Felder: `datum` (YYYY-MM-DD), `zeit` (HH:MM), `personen`
  (Zahl), `art`, `wunschbereich` — nur ausfüllen, was der Anrufer explizit nannte.
- `whatsapp_message`-Description umschreiben: **Nie eine abgeschlossene
  Reservierung behaupten** („notiert/gebucht/reserviert" verboten) — als Anliegen
  formulieren, nächste Frage stellen.
- System-Prompt (Assistant `ee06a00c-3fd3-4898-8d1e-33c0bd19d861`): Bei
  Reservierungswunsch Datum/Uhrzeit/Personen in einem Turn miterfragen.

### B.2 n8n Forwarder: „Prepare Message" + „Store Context"
- Neue Args aus `toolCalls[0].function.arguments` parsen und in der
  `vapi.context`-Payload an den Dispatcher mitgeben.

### B.3 n8n Dispatcher (`oaxcHyyQZ5zHFNvr`)
- „Handle VAPI Context": neue Felder im `vctx_<phone>`-JSON speichern.
- „Detect Order Intent": TELEFONAT-KONTEXT um „Genannt: Datum …, Zeit …,
  Personen …" ergänzen (nur wenn vorhanden).

### B.4 RESBOT: System-Prompt-Patch
- TELEFONAT-KONTEXT-Daten übernehmen, **nicht erneut fragen**; nur fehlende
  Pflichtangaben erfragen.
- Vor erfolgreichem `Submit_Reservation_Form` nur von „Anfrage" sprechen; danach
  ehrlich bestätigen („Anfrage abgeschickt, Bestätigungsmail folgt").

### Tests (nach Umsetzung)
1. Testanruf mit Reservierungswunsch (Datum/Zeit/Personen nennen).
2. WA-Nachricht prüfen: keine falsche Zusage.
3. Im Chat antworten → RESBOT muss Submit ausführen und ehrlich bestätigen.
4. Eintrag in Notion prüfen, Test-Eintrag danach löschen.

## 5. Verworfene Alternative (Variante A, der Vollständigkeit halber)

Echte Buchung direkt am Telefon: neuer Workflow „CDS-VAPI-RESERVATION"
(Webhook `vapi-create-reservation` → Validierung → GetOpenings-Check →
WPForms-Submit → Telegram + Antwort an VAPI) + neues VAPI-Tool
`create_reservation` (`async: false`). Verworfen wegen: längere Telefonate gegen
die 40s-Philosophie, zweiter paralleler Schreibpfad, keine Bestätigungsmail ohne
E-Mail des Anrufers. Kann später nachgeholt werden, falls gewünscht.

## 6. Revision nach Testanruf (30.8.2026)

Der Testanruf zeigte: Datensammlung am Telefon bläht die VAPI-Kosten auf und der
Kunde muss im WhatsApp-Chat ohnehin alles durchlaufen. **Entscheidung User:**
VAPI auf Originalverhalten zurücksetzen (nur WhatsApp-Übergabe oder Rückrufbitte),
nur die Ehrlichkeits-Korrektur behalten.

Umgesetzt:
- VAPI Assistant-Prompt + Tool `send_whatsapp` auf Original zurückgesetzt
  (keine datum/zeit/personen-Felder mehr, kein „RESERVIERUNG SAMMELN").
- **Behalten:** Verbot falscher Reservierungszusagen in Tool-Guidance und Prompt;
  RESBOT-Regel „erst nach Submit bestätigen"; Forwarder-Fehlerpfad-Fix.
- n8n-seitige Durchreichung (`reservierung`-Objekt in Forwarder/Dispatcher) bleibt
  ruhend und abwärtskompatibel — ohne VAPI-Felder passiert nichts.

## 7. Nachtrag: Server-URL-Verlust, WaSender-402 & Öffnungszeiten-Check (30.8.2026)

- **VAPI-Tool `send_whatsapp`:** Beim Revert (Abschnitt 6) ging `server.url`
  verloren und das Tool stand auf `async: true` → Tool-Calls liefen ins Leere,
  VAPI meldete generisch „Success.", keine n8n-Execution, Pete behauptete
  fälschlich den Versand. Fix: Server-URL wieder gesetzt, `async: false`.
- **WaSender:** `402 subscription past due` → User hat Abo verlängert.
- **Öffnungszeiten-Check (neu):** Im VAPI-Forwarder prüft der neue Knoten
  „Check Opening Day" (nach „Get Openings") bei Reservierungen, ob der genannte
  Tag ein Schließtag ist, und ersetzt die Grußnachricht ggf. durch eine
  Korrektur mit Vorschlag des nächsten Öffnungstages. Details und Tests:
  CDS-Bot-Dokumentation.md §9.5.

## 8. Offene Punkte

- [x] B.1–B.4 umgesetzt und nach Review **teil-revertiert** (VAPI-Seite zurück,
  Ehrlichkeits-Fixes behalten) — Stand: CDS-Bot-Dokumentation.md §9.3
- [ ] **End-to-End-Testanruf** durch das Team (echte WA-Zustellung ist nur mit
  echtem Anruf testbar) — jetzt möglich: Server-URL-Fix + WaSender-Abo erledigt
- [ ] Kunden vom 29.8. manuell nachfassen (keine Reservierungen im System!)
- [x] Forwarder-Defekt behoben (30.8.2026, Exec 364795): Error-Branch von
  „Send WA Greeting" → Telegram-Alarm + ehrliche FEHLER-Antwort an VAPI
  (Details: CDS-Bot-Dokumentation.md §9.4)
- [x] Tool-Server-URL wiederhergestellt + Öffnungszeiten-Check eingebaut
  (30.8.2026, Execs 364919/364921/364923) — Details: CDS-Bot-Dokumentation.md §9.5
- [x] Sanfte Zeitslot-Steuerung im RESBOT eingebaut (30.8.2026:
  FormatReservations-Ankünfte + Prompt-Abschnitt + letzter Slot korrigiert)
  — Details: CDS-Bot-Dokumentation.md §9.6
- [ ] **Livetest Zeitslot-Steuerung:** Test-Reservierungen um 10:00/10:15 in
  Notion anlegen, per WhatsApp „Samstag 10 Uhr" anfragen → Bot soll einmal
  sanft 10:15/10:30 anbieten; bei Beharren auf 10 Uhr → sofort bestätigen
- [ ] Optional später: Kapazitätsprüfung; Fehler-Monitoring auf weitere CDS-Workflows ausdehnen
