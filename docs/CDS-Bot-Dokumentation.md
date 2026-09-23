# CDS Reservierungs-Bot — Technische Dokumentation

**Projekt:** Charme du Sud WhatsApp Reservierungs-Bot
**Datum:** 11.–12. Mai 2026
**Instanz:** https://n8n.gevio.cloud
**Workflows:** CDS-RESERVATION-BOT (`rBoD3Cu0UM9WiT3N`), CDS-VAPI_FORWARDER (`6NmOnAM7cgsRXtOh`), GetOpenings (`DgchDWuV5O2ldQQI`)

---

## 1. Systemübersicht

Das Bio-Café Charme du Sud in Weisenheim am Berg nutzt einen WhatsApp-Bot zur automatisierten Tischreservierung. Der Bot nimmt Anfragen über WhatsApp entgegen, prüft Öffnungszeiten, Belegung und Wetter, und kann Reservierungen direkt über das Webformular auf charme-du-sud.de absenden. Zusätzlich wurde ein Telefon-Assistent (VAPI) eingerichtet, der Anrufer auf den WhatsApp-Service weiterleitet.

### Technische Komponenten

- **n8n** (Workflow-Automation) auf n8n.gevio.cloud
- **WaSender** (WhatsApp Business API)
- **Notion** (Tischreservierungen DB + Team-Kontakte DB)
- **OpenRouter** (LLM API: Claude Sonnet 4.5 + Claude Haiku 4.5)
- **Open-Meteo** (Wetter-API)
- **Google Places** (Öffnungszeiten via Sub-Workflow)
- **VAPI** (Telefon-Assistent mit ElevenLabs Voice)
- **Telnyx** (Telefonnummer: +49 6221 7234134)
- **Telegram** (Team-Benachrichtigung bei Änderungswünschen)

---

## 2. Workflow-Architektur: CDS-RESERVATION-BOT

### 2.1 Gesamtstruktur (21 Nodes)

```
Webhook
  → Filter Incoming Only (IF: event = messages.received)
    → Check Team-Nummern (Notion: WhatsApp Bot Kontakte)
      → Message Gate (Code: Klassifizierung + Cooldown)
        → Route Message (Switch: Expression-Modus)
          ├── Output 0: reservation → GetOpenings → LoadReservations
          │     → FormatReservations → AI Agent → Send WA
          ├── Output 1: update → Update Reply → Send WA Update + Telegram
          └── Output 2: general → GetOpenings Light → AI Agent Light → Send WA General
```

### 2.2 Neue Nodes (11.–12. Mai 2026 hinzugefügt)

| Node | Typ | Funktion |
|------|-----|----------|
| Filter Incoming Only | IF | Filtert `message.sent`-Events (Bot-eigene Nachrichten) vor der Verarbeitung |
| Check Team-Nummern | Notion | Prüft ob Absender Team-Mitglied ist (Blockliste) |
| Message Gate | Code | Spam-Schutz, Cooldown, Nachrichtenklassifizierung |
| Route Message | Switch | Drei-Wege-Routing nach messageType (Expression-Modus) |
| Update Reply | Code | Fixe Antwort für Änderungswünsche + Telegram-Nachricht |
| Send WA Update | HTTP Request | WhatsApp-Nachricht für Update-Pfad |
| Send a text message | Telegram | Team-Benachrichtigung bei Änderungswünschen |
| GetOpenings Light | Execute Workflow | Öffnungszeiten für General-Pfad |
| AI Agent Light | Agent | Günstiges Model (Claude Haiku 4.5) für allgemeine Fragen |
| Chat Model Light | OpenRouter | Claude Haiku 4.5 via OpenRouter |
| Send WA General | HTTP Request | WhatsApp-Nachricht für General-Pfad |

---

## 3. Durchgeführte Änderungen im Detail

### 3.1 Filter Incoming Only

**Problem:** WaSender sendet zwei Event-Typen an den Webhook: `messages.received` (eingehend) und `message.sent` (ausgehend). Ausgehende Nachrichten haben eine andere Payload-Struktur ohne `cleanedSenderPn`, was den Notion-Node zum Crashen brachte (400 Error bei jeder zweiten Execution).

**Lösung:** IF-Node direkt nach dem Webhook, der nur `messages.received`-Events durchlässt.

### 3.2 Check Team-Nummern (Blockliste)

**Logik (umgekehrte Whitelist):** Nummer in DB gefunden → Team-Mitglied → STOPP. Nummer nicht gefunden → Kunde → WEITER.

**Notion-DB:** "WhatsApp Bot Kontakte" (`dfd1b8d327c84ff9abb0eab8ceac80d6`)
- Properties: Name (Title), Telefonnummer (Text), Aktiv (Checkbox)
- `alwaysOutputData: true` damit bei 0 Ergebnissen trotzdem ein leeres Item an Message Gate weitergegeben wird

### 3.3 Message Gate

Zentrale Steuerungslogik mit folgenden Prüfungen (in Reihenfolge):

1. **Anonyme Nummer:** Keine `cleanedSenderPn` → Abbruch
2. **fromMe-Check:** Ausgehende Nachrichten → Cooldown setzen, Abbruch
3. **Team-Blocklist:** Notion-Ergebnis mit `id` vorhanden → Abbruch
4. **Manueller Kontakt Cooldown (24h):** Café-Team hat kürzlich manuell geantwortet → Bot schweigt
5. **Spam-Schutz (5 Sek.):** Letzte Bot-Antwort < 5 Sekunden → ignorieren
6. **Klassifizierung:** Keywords bestimmen den messageType

**Keyword-Priorität:**
- Default = `reservation` (voller AI Agent mit Memory + Tools)
- `update` überschreibt bei Änderungs-Keywords (verspäten, stornieren, etc.)
- `general` nur bei expliziten Info-Keywords (öffnungszeit, adresse, parken, etc.) UND wenn keine Reservation/Update-Keywords
- **Aktive Session (seit 1.9.2026):** War die letzte Bot-Antwort ≤ 2h her (`inActiveSession`), gehen ALLE Folgenachrichten an den Reservierungs-Agenten — `update`/`general` greifen nur noch außerhalb eines laufenden Dialogs. Hintergrund: Rückfragen wie „ist früher besser?" wurden per Keyword („früher" + „Uhr") als `update` fehlklassifiziert und bekamen die statische Fallback-Antwort statt der Agenten-Antwort (vgl. §9.7).

**Entfernte Keywords:** `email` und `telefon` aus GENERAL_KEYWORDS entfernt, da sie in Kundendaten vorkommen (z.B. `googlemail.com` enthält "email") und falsches Routing zum AI Agent Light (ohne Memory) verursachten.

### 3.4 Route Message (Switch im Expression-Modus)

```javascript
{{ {'reservation': 0, 'update': 1, 'general': 2}[$json.messageType] ?? 2 }}
```

Nutzt Expression-Modus statt Rules-Modus, da der Rules-Modus bei den Condition-Auswertungen fehlerhaft routete.

### 3.5 WeatherForecast-Tool (Fix)

**Problem:** AI Agent rief das Tool nie auf (0 Items in/out), weil das Schema `jsonSchemaExample` nutzte statt `schemaType: "manual"`.

**Schema-Fix:**
- `schemaType: "manual"` mit explizitem `inputSchema` (JSON mit `date`-Property und Beschreibung)
- JS-Code liest Input über `query`-Variable: `const d = typeof query === 'string' ? JSON.parse(query) : query;`

**HTTP-Fix:**
- `$http()` existiert nicht in toolCode-Nodes
- Ersetzt durch `this.helpers.httpRequest()` mit URL-inlined Query-Parametern

### 3.6 AI Agent System-Prompt Änderungen

| Änderung | Details |
|----------|---------|
| Öffnungszeiten | Saisonal bedingt, konkrete Daten statt feste Wochentage |
| Änderungen/Stornierungen | Nur bestätigen + ans Team weiterleiten, keine Rückfragen |
| Telefonnummer | Wird automatisch aus Webhook genommen, nicht erfragt |
| WhatsApp-Ton | Keine Brieffloskeln ("Mit freundlichen Grüßen" entfernt) |
| Wohlfühl-Infos | Barrierefreiheit, Hunde, Glutenfrei, Vegan, Kinder |
| Kontaktdaten | Adresse: Hauptstraße 45, Tel: 0173-6840361 (nur auf Nachfrage) |
| Memory | Von 8 auf 20 Messages erhöht |

### 3.7 GetOpenings Sub-Workflow Erweiterung

Neue Output-Felder im Code-Node:
- `detailedHours`: Konkrete Tage mit Datum (z.B. "Donnerstag, 14.05.2026: 10:00-17:00 Uhr")
- `specialDaysInfo`: Hinweis auf Sonderöffnungen/Feiertage

---

## 4. Workflow: CDS-VAPI_FORWARDER

### 4.1 Zweck

Nimmt VAPI-Tool-Calls entgegen und sendet personalisierte WhatsApp-Nachrichten an Anrufer.

### 4.2 Struktur (4 Nodes)

```
VAPI Webhook (POST /webhook/vapi-forwarder)
  → Prepare Message (Code: Nummer extrahieren, Sprache, Intent)
    → Send WA Greeting (HTTP: WaSender API)
      → Respond to VAPI (JSON-Antwort an VAPI)
```

### 4.3 Features

- **Mehrsprachig:** DE, EN, FR — basierend auf `language`-Parameter vom VAPI-Assistant
- **Personalisiert:** `caller_name` und `intent` bestimmen die Begrüßung
- **Intent-basierte Nachrichten:**
  - `reservation`: "...hier können Sie direkt Ihre Reservierung aufgeben."
  - `complaint`: "...Bitte schildern Sie uns hier Ihr Anliegen."
  - `general`: "...hier können Sie Ihr Anliegen direkt mitteilen."
- **Telefon-Referenz:** Begrüßung bezieht sich auf das gerade geführte Telefonat
- **Fehlerbehandlung:** Error-Output von WaSender wird auch an VAPI zurückgegeben

---

## 5. VAPI Telefon-Assistent

### 5.1 Konfiguration

| Setting | Wert |
|---------|------|
| Assistant-Name | CDS Receptionist |
| Model | GPT-4.1 |
| Transcriber | Deepgram (Flux General Multilingual) |
| Voice | ElevenLabs (geklonte Stimme, Multilingual v2) |
| Telefonnummer | +49 6221 7234134 (Telnyx) |

### 5.2 Tool: send_whatsapp

- **Server URL:** `https://n8n.gevio.cloud/webhook/vapi-forwarder`
- **Parameters:** `language` (required), `phone_number` (optional), `caller_name` (optional), `intent` (optional)

### 5.3 Ablauf

1. Anrufer wird begrüßt ("Café Charme, guten Tag!")
2. Assistant wartet auf Anliegen
3. Bestätigt das Anliegen kurz
4. Fragt nach WhatsApp-Nummer (aktuelle oder andere)
5. Bei anderer Nummer: wiederholt zur Bestätigung
6. Tool-Aufruf → n8n → WhatsApp-Nachricht
7. Bestätigung und Verabschiedung

---

## 6. Notion-Datenbanken

### 6.1 Tischreservierungen

- **DB-ID:** `8bc09508-d06d-4be0-8d53-d5645279e92c`
- **Verwendung:** LoadReservations Node, Belegungsprüfung

### 6.2 WhatsApp Bot Kontakte (Team-Blockliste)

- **DB-ID:** `dfd1b8d327c84ff9abb0eab8ceac80d6`
- **Properties:** Name, Telefonnummer, Aktiv, Letzter manueller Kontakt
- **Verwendung:** Check Team-Nummern Node

---

## 7. Credentials

| Name | Typ | ID | Verwendung |
|------|-----|----|------------|
| Notion account | notionApi | YCmX3TC66nkMdGlK | Notion-Abfragen |
| Bearer Auth account 2 | httpBearerAuth | SD3t1AJcwpVbkxh0 | WaSender API |
| OpenRouter account | openRouterApi | vh2opMuTaY4jQi8C | Claude Sonnet/Haiku |
| Telegram account | telegramApi | KA8jIrIBZEerUBrL | Team-Benachrichtigung |

---

## 8. Bekannte Einschränkungen und Hinweise

1. **Static Data:** Bot-Cooldowns und manuelle Kontakt-Timestamps werden in n8n Static Data gespeichert. Bei n8n-Neustart gehen diese verloren.

2. **Editor-Versionen:** Änderungen über die API können vom n8n-Editor überschrieben werden. Nach API-Änderungen immer F5 im Editor drücken.

3. **WeatherForecast:** In toolCode-Nodes muss `this.helpers.httpRequest()` statt `$http()` verwendet werden. Query-Parameter direkt in die URL inlinen.

4. **Telnyx-Nummer:** Status "Req. In Review" — Freischaltung durch Telnyx ausstehend.

5. **VAPI Talk-Test:** Im Browser-Test gibt es keine echte Anrufernummer. WhatsApp-Zustellung nur bei echten Telefonanrufen testbar.

---

## 9. Nachtrag 30.8.2026 — OpenRouter-Ausfall, Error-Alarm & ehrliche Telefon-Reservierungen

Details und Beweise: `CDS-Bot-Fix-2026-08-30.md` (gleicher Ordner).

### 9.1 Störung 29.8.2026
OpenRouter-Guthaben leer (HTTP 402 am Node „OpenRouter Chat Model") → RESBOT und
ORDER-BOT antworteten ab 29.8. morgens auf keine Kundennachricht mehr. Nach
Aufladung per Live-Test verifiziert (Execution 363723, AI-Antwort „OK.").

### 9.2 Neuer Workflow „CDS Error Alarm" (`sJREPHKmhkENsUF7`, aktiv)
Error Trigger → Telegram (Credential `KA8jIrIBZEerUBrL`, Chat `1313007318`).
Als `settings.errorWorkflow` eingetragen bei RESBOT, ORDER-BOT und VAPI_FORWARDER.

### 9.3 Telefon→WhatsApp: Ehrlichkeits-Fix (30.8.2026, revidiert)
Grund: Der VAPI-Assistent durfte per Tool-Guidance „Ich habe notiert …" schreiben,
ohne dass etwas angelegt wurde (Halluzinations-Fall Chat 3, 29.8.).

**Finaler Stand (nach Nutzer-Review des Testanrufs):** VAPI bleibt bewusst schlank —
keine Datensammlung am Telefon, nur WhatsApp-Übergabe oder Rückrufbitte
(40s-Philosophie, VAPI-Kosten). Umgesetzt:

- **VAPI Tool `send_whatsapp`** (`e45275e0-…`) und **Assistant-System-Prompt**
  (`ee06a00c-…`): strenges Verbot, Reservierungen als notiert/gebucht/bestätigt
  darzustellen; Beispiel-Formulierungen entsprechend bereinigt. Sonst
  Original-Verhalten.
- **CDS-RESERVATION-BOT System-Prompt:** erst NACH erfolgreichem
  `Submit_Reservation_Form` von „Reservierung eingegangen" sprechen, vorher nur
  „Anfrage".
- **Ruhend (abwärtskompatibel, kein Verhaltensunterschied):** Forwarder
  „Prepare Message"/„Store Context", Dispatcher „Handle VAPI Context"/
  „Detect Order Intent" können ein optionales `reservierung`-Objekt
  (datum/zeit/personen/art/wunschbereich) durchreichen bzw. injizieren. Da das
  VAPI-Tool diese Felder nicht mehr enthält, bleibt der Pfad inaktiv. Bei Bedarf
  später aktivierbar.

### 9.4 Forwarder-Defekt behoben (30.8.2026)
Früher führte die Error-Branch von „Send WA Greeting" (`onError: continueErrorOutput`,
main 1) direkt zu „Respond to VAPI" mit dem statischen Text „WhatsApp-Nachricht
gesendet." — **auch bei fehlgeschlagenem WaSender-Versand**; zudem lief „Store
Context" nur auf der Erfolgs-Branch (Kontextverlust bei Fehler).

**Fix:** Neue Nodes „Send TG WA-Error" (Telegram-Alarm mit Nummer, Intent, Anliegen,
Fehlermeldung + Hinweis auf telefonischen Rückruf) und „Respond WA Failed" (ehrliche
Fehler-Antwort an VAPI, damit Pete dem Anrufer keinen Versand vortäuscht).
Verdrahtung: `Send WA Greeting [main 1] → Send TG WA-Error → Respond WA Failed`.
Verifiziert (Execution 364795): WA-Fehler → TG-Alarm gesendet, VAPI erhielt
ehrliche FEHLER-Antwort.

### 9.5 Tool-Server-URL-Verlust & Öffnungszeiten-Check im Forwarder (30.8.2026)

**Defekt 1 — Server-URL verloren:** Beim Revert des VAPI-Tools `send_whatsapp`
(9.3) ging dessen `server.url` verloren; das Tool war zudem `async: true`.
Folge: Tool-Calls gingen ins Leere, VAPI lieferte nach 4 ms ein generisches
„Success." als Fallback, in n8n kam keine Execution an und Pete behauptete
„Die Nachricht ist unterwegs". **Fix:** `server.url`
(`https://n8n.gevio.cloud/webhook/vapi-forwarder`) wieder gesetzt, `async: false`
(synchron, damit die ehrliche Fehlerantwort aus 9.4 bei Pete ankommt).
Verifiziert: Execution 364855 — Request erreicht Forwarder, Fehlerpfad antwortet
synchron.

**Defekt 2 — WaSender-Abo überfällig:** WaSender lieferte `402 „subscription
past due"`; Versand war bis zur Verlängerung durch den User komplett gestoppt.

**Neues Feature — Öffnungszeiten-Check für Reservierungs-Grußnachricht:**
Pete kennt bewusst keine Öffnungszeiten; die erste WA-Nachricht fragte daher
auch bei Wunschterminen an Schließtagen (z. B. Freitag) ahnungslos nach. Neue
Kette im WA-Zweig des Forwarders:
`Intent Switch [main 0] → Get Openings → Check Opening Day → Send WA Greeting`.
- **„Get Openings"** (`executeWorkflow` → Sub-Workflow `DgchDWuV5O2ldQQI`) holt
  `weeklyClosedDays`, `closedDates`, `holidayDates`, `nextOpen`.
- **„Check Opening Day"** (Code) prüft nur bei `intent = reservation`: Erkennt in
  `summary`/`text` Wochentage (de/en/fr), „heute/morgen" (today/tomorrow,
  aujourd'hui/demain; „am Morgen" wird nicht als „morgen" gewertet) und explizite
  Daten `dd.mm.[yyyy]`; Datumswertung in Zeitzone Europe/Berlin. Geschlossen, wenn
  Datum in `closedDates` oder Wochentag in `weeklyClosedDays` (außer Feiertag mit
  Sonderöffnung). Dann wird die Grußnachricht durch eine Korrektur ersetzt
  („Am Freitag, 04.09. haben wir leider geschlossen — Samstag, 05.09.2026,
  10:00 Uhr sind wir wieder für Sie da. Passt Ihnen das? …", Templates de/en/fr).
  Kein Tag erkennbar oder Tag geöffnet → Petes Nachricht bleibt unverändert.
- Verifiziert (Executions 364919/364921/364923): Freitag → Korrektur mit
  Samstags-Vorschlag; Samstag und Summary ohne Tagesbezug → unverändert.

Grenze: Nur explizit genannte Tage werden geprüft; bei komplexen Formulierungen
korrigiert der RESBOT weiterhin nach der ersten Kunden-Antwort.

### 9.6 Sanfte Zeitslot-Steuerung im RESBOT (30.8.2026)

**Ziel:** Stoßzeiten (10–12 Uhr, aber auch nachmittags) etwas verteilen, ohne
Gäste zu verprellen — der Kunde bekommt immer seine Wunschzeit.

**Änderung 1 — Node „FormatReservations" (RESBOT):** Die Kalkulation
(`<INTERNE_SYSTEM_KALKULATION>`) enthält jetzt zusätzlich zur bisherigen
BELEGUNG (120-Min-Overlap) eine ANKUENFTE-Sektion: pro Tag alle Start-Slots mit
Anzahl neu ankommender Tische/Personen, z. B.
`Sa, 5.9.: 10:00 → 2 Tische / 6 Pers. | 10:15 → 1 Tische / 4 Pers.`.
Damit sieht das LLM den Ansturm gleichzeitiger Ankünfte, nicht nur die
Überlapp-Belegung. Bestehende Ausgabe unverändert (abwärtskompatibel); lokal
mit Node.js gegen Beispieldaten getestet (inkl. Storniert-Filter und
Leereingabe).

**Änderung 2 — System-Prompt „AI Agent":** Neuer Abschnitt
„SANFTE ZEIT-STEUERUNG (FINGERSPITZENGEFÜHL)" (vor SERVICE_LOGIK):
- Auslöser: Wunschslot frei, aber im Fenster ±30 Min kommen laut ANKUENFTE
  ≥ 2 Tische oder ≥ 8 Personen neu an.
- Dann genau EINMAL sanft vorfühlen (ehrlicher Hinweis „wird ziemlich voll,
  kleine Wartezeiten an der Kaffeemaschine möglich"), nur ±15/±30 Min als
  Alternative, bei Wunsch 10:00 (Öffnung) nur spätere Alternativen.
- Besteht der Gast auf seiner Zeit: sofort akzeptieren, kein zweites
  Nachfragen („wir bekommen das hin").
- Keine Zahlen gegenüber Gästen (Datenschutz-Regeln gelten weiter).
- Nicht anwenden bei VOLL — das bleibt der „ausgebucht"-Pfad.

**Änderung 3 — Korrektur letzter Slot:** Prompt sagte „letzter Slot 16:00",
korrigiert auf **16:30 an Sa/So/Feiertagen, 15:30 an Wochentagen**
(Wochentage derzeit ohnehin geschlossen; Regel für die Zukunft).

**Betriebsferien/Urlaub (keine Änderung nötig, verifiziert):** Die in der
Tischbelegungs-App gepflegten Schließtage/Betriebsferien (z. B. 05.10.–22.10.2026)
fließen bereits über `Get Schliesstage` → GetOpenings-Workflow als
`closedDates`/`closedDaysInfo` in den RESBOT-Prompt (`<geschlossene_tage>`,
Submit-Verbot an Schließtagen) und in den Forwarder-Check „Check Opening Day".

**Test (durch User):** Test-Reservierungen um 10:00/10:15 in Notion anlegen,
per WhatsApp „Samstag 10 Uhr" anfragen → Bot bietet einmal sanft 10:15/10:30
an; bei „nein, 10 Uhr bitte" → sofort normale Bestätigung. Zusatz: Anfrage für
Oktober-Betriebsferien → geschlossen-Antwort mit nächstem Öffnungstag.

### 9.7 Fix: Session-Routing im Message Gate + Steuerungsfenster (1.9.2026)

**Anlass (Execution 368914/368922):** Kundin fragte mitten im
Reservierungsdialog „Wie ist es denn früher als um 11 Uhr? Wäre das besser
noch?" — das Gate klassifizierte per Keyword („früher" in UPDATE_KEYWORDS +
„Uhr" im Reservierungs-Kontext) als `update` und schickte die statische
Fallback-Antwort („Ich habe Ihren Hinweis notiert und leite ihn an unser Team
weiter"), statt den AI Agent mit Memory und Belegungsdaten antworten zu lassen.

**Änderung 1 — Node „Message Gate":** Session-aware Routing. Neue Variable
`inActiveSession = lastBotInteraction > 0 && hoursSinceBot <= SESSION_TIMEOUT_H`.
Bei aktiver Session (laufender Dialog ≤ 2h) ist der messageType immer
`reservation`; die Keyword-Klassifizierung (`update`/`general`) greift nur
noch bei Session-Start bzw. nach Timeout. Nebenwirkung: Echte Änderungswünsche
mitten im Dialog landen jetzt ebenfalls beim Agenten — der kann sie ohnehin
nur entgegennehmen (Team-Benachrichtigung läuft dann nicht mehr über den
Update-Pfad, sondern über den normalen Flow).

**Änderung 2 — Steuerungsfenster (System-Prompt „SANFTE ZEIT-STEUERUNG"):**
Zunächst am 1.9. mittags auf „60 Min VOR bis 30 Min NACH" erweitert — das
erwies sich im Livetest als falsch: Der Bot steuerte eine 11:00-Anfrage nur
wegen des 10:00-Ansturms (25 Personen), obwohl 11:00 selbst frei war.
**Betriebliche Grundregel (Klarstellung durch Inhaber):** Es zählen NUR
gleichzeitige Ankünfte — früher angekommene, noch sitzende Gäste sind egal,
neue Gäste können immer dazugenommen werden, solange nicht zu viele
GLEICHZEITIG ankommen. Auslöser daher final: Ankünfte im engen Fenster
**±15 Minuten um den Wunschslot** (≥ 2 Tische ODER ≥ 8 Personen). Dazu zwei
Formulierungsregeln: (a) Nur von gleichzeitigen Ankünften sprechen („da
kommen viele Gäste auf einmal an"), niemals von allgemeiner Auslastung oder
sitzenden Gästen; (b) bei aktiver Rückfrage des Gastes („ist früher/später
besser?") ehrlich anhand der ANKUENFTE-Daten antworten und den Slot mit den
wenigsten gleichzeitigen Ankünften empfehlen.

**Verifiziert:** PUT via n8n-API (Status 200), Re-Fetch bestätigt beide
Änderungen im aktiven Workflow. Keine Halluzination festgestellt: „Brunch bis
12:00" und „Wasserschale für Hunde" stehen beide bereits als gesicherte Fakten
im Prompt (§ SERVICE_LOGIK / WOHLFUEHL-INFOS).

---

*Dokument erstellt am 12. Mai 2026 — Nachtrag 30.8.2026*
