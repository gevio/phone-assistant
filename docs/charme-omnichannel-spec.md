# Charme du Sud – Omnichannel Kommunikationssystem
## Claude Code Implementierungs-Spezifikation v2.0

**Projekt:** charme-omnichannel  
**Version:** 2.0  
**Stack:** PHP 8.x · Twilio · ElevenLabs · Claude API · WA-Sende-API · n8n · Meta WhatsApp Business API  
**Hosting:** DomainFactory VPS (bestehend)  
**Datum:** Mai 2026

---

## 1. Kanalübersicht & Verantwortlichkeiten

```
┌─────────────────────────────────────────────────────────────────┐
│                    CHARME DU SUD                                │
│              Omnichannel Kommunikation                          │
└─────────────────────────────────────────────────────────────────┘

KANAL 1          KANAL 2          KANAL 3          KANAL 4
Festnetz         Handy            WhatsApp         Website
  │                │                │                │
  │                │                │                │
  ▼                ▼                ▼                ▼
Twilio         klingelt         Meta WA           PHP
Voice Agent    beim             Business          Chat
(KI)           Mitarbeiter      API               Widget
  │                                │                │
  │                                └────────┬───────┘
  │                                         │
  └─────────────────────────────────────────┘
                                            │
                                            ▼
                                    n8n Zentrallogik
                                    (Claude API)
                                            │
                              ┌─────────────┴──────────────┐
                              ▼                            ▼
                         Reservierungs-DB             Notion Log
                         (MariaDB)                   (optional)
```

### Kanal-Übersicht auf einen Blick

| # | Kanal | Nummer/URL | KI? | Aktion bei Eingang |
|---|-------|-----------|-----|-------------------|
| 1 | Festnetz | +49 (bekannt) | ✅ Twilio Voice Agent | Ansagen, Intent, → WA |
| 2 | Handy direkt | +49 (Facebook/Instagram) | ❌ | Klingelt beim Mitarbeiter |
| 3 | WhatsApp | Handynummer (WA Client gelöscht) | ✅ n8n + Claude | Chat-Automation |
| 4 | Website | charmedusud.de | ✅ PHP Widget + Claude | Chat-Automation |

---

## 2. Kanal 1 – Festnetz → Twilio Voice Agent

### Funktionsprinzip
Die Festnetznummer wird beim Anbieter auf eine Twilio +49-Nummer umgeleitet.
Twilio nimmt den Anruf entgegen und führt den Anrufer durch den KI-Dialog.

### Gesprächsfluss

```
Anruf eingehend
      │
      ▼
index.php → ElevenLabs Begrüßung (Voice Clone)
      │
      ▼ (Spracheingabe des Anrufers)
handle-intent.php → Claude API
      │
      ├── RESERVATION ──→ confirm-whatsapp.php → send-whatsapp.php
      ├── QUESTION    ──→ confirm-whatsapp.php → send-whatsapp.php
      ├── COMPLAINT   ──→ escalate.php (→ Mitarbeiter-Handy)
      ├── HUMAN       ──→ escalate.php (→ Mitarbeiter-Handy)
      └── UNCLEAR     ──→ Nachfragen (1x) → escalate.php
```

### Gecachte ElevenLabs Audio-Texte

| Datei | Text |
|-------|------|
| greeting.mp3 | „Bonjour! Guten Tag und herzlich willkommen bei Charme du Sud in Weisenheim am Berg." |
| ask_intent.mp3 | „Wie kann ich Ihnen helfen? Möchten Sie einen Tisch reservieren, haben Sie eine Frage, oder möchten Sie mit jemandem aus unserem Team sprechen?" |
| whatsapp_offer.mp3 | „Darf ich mich im Anschluss kurz per WhatsApp bei Ihnen melden? So kann ich Ihnen alle Details bequem schicken." |
| transfer.mp3 | „Einen Moment bitte, ich verbinde Sie sofort mit unserem Team." |
| goodbye.mp3 | „Wunderbar! Sie erhalten gleich eine WhatsApp-Nachricht. Auf Wiederhören und merci!" |

### Claude Intent-Prompt

```
Du bist der Telefonassistent von Charme du Sud, einem charmanten 
französisch inspirierten Café-Restaurant in Weisenheim am Berg.
Klassifiziere die Aussage in GENAU EINE Kategorie:
RESERVATION | QUESTION | COMPLAINT | HUMAN | UNCLEAR
Antworte NUR mit dem Stichwort.
```

### Dateistruktur Kanal 1

```
/var/www/charme-phone/
├── config.php
├── index.php               # Begrüßung + Gather
├── handle-intent.php       # Claude Klassifizierung
├── confirm-whatsapp.php    # WA-Angebot
├── send-whatsapp.php       # WA-Sende-API
├── escalate.php            # Twilio Dial → Mitarbeiter
├── tts.php                 # ElevenLabs Helper
├── claude.php              # Anthropic API Wrapper
├── logger.php              # MariaDB Logging
└── audio/                  # Gecachte MP3s
```

---

## 3. Kanal 2 – Handy-Direktanruf

### Funktionsprinzip
Die Handynummer ist auf Facebook/Instagram/Google öffentlich sichtbar.
**Kein KI-Eingriff.** Das Handy klingelt direkt beim Mitarbeiter.

### Warum kein Voice Agent hier?
- Stammkunden kennen diese Nummer und erwarten persönlichen Kontakt
- Keine technische Notwendigkeit (Weiterleitung würde Erwartung brechen)
- Ressourcen-Effizienz: KI nur wo sie echten Mehrwert bringt

### Empfehlung
Eine kurze Handy-Ansage (falls nicht abgenommen) als Voicemail:
> „Bonjour! Sie haben Charme du Sud erreicht. Wir sind gerade beschäftigt –
> schreiben Sie uns gern auf WhatsApp oder rufen Sie später nochmal an. Merci!"

---

## 4. Kanal 3 – WhatsApp Business (Handynummer)

### Einmaliger Setup-Schritt: WhatsApp Client löschen

```
⚠️  WICHTIG: Datensicherung zuerst!

1. WhatsApp-Chat-Backup auf Google Drive / iCloud durchführen
2. WhatsApp-App auf dem Handy deinstallieren
3. Meta Business Manager: Nummer verifizieren (SMS-Code)
4. WhatsApp Business API aktivieren (via WA-Sende-API oder direkt Meta)
5. n8n Webhook für eingehende Nachrichten konfigurieren
```

Nach diesem Schritt ist die Handynummer eine offizielle WhatsApp Business API-Nummer.
Eingehende Nachrichten gehen nicht mehr in eine App, sondern direkt in n8n.

### n8n Flow: Eingehende WhatsApp-Nachricht

```
WhatsApp Eingehend (Webhook)
          │
          ▼
    Nachricht lesen
          │
          ▼
    Session prüfen
    (laufendes Gespräch?)
          │
    ┌─────┴──────┐
    │            │
  Neu         Läuft
    │            │
    ▼            ▼
 Begrüßung  Kontext laden
    +        aus MariaDB
  Intent
          │
          ▼
    Claude API Call
    (Charme du Sud
     System Prompt)
          │
    ┌─────┼──────────────┐
    ▼     ▼              ▼
  Info  Reservierung  Eskalation
  Antwort  Flow       → Mitarbeiter
              │       Benachrichtigung
              ▼
         MariaDB
         (Reservierung
          speichern)
```

### Claude System-Prompt für WhatsApp & Website

```
Du bist der digitale Assistent von Charme du Sud, einem charmanten 
französisch inspirierten Café-Restaurant in Weisenheim am Berg, 
Rheinland-Pfalz. Du kommunizierst freundlich, herzlich und mit 
einem leichten französischen Flair. Antworte immer auf Deutsch,
außer der Gast schreibt auf Französisch oder Englisch.

Du kannst:
- Tischreservierungen aufnehmen und bestätigen
- Fragen zu Öffnungszeiten, Karte, Anfahrt beantworten
- Bestellungen für Außer-Haus weitergeben
- Bei Beschwerden verständnisvoll reagieren und Mitarbeiter 
  benachrichtigen

Du kannst NICHT:
- Preise nennen die du nicht kennst (dann: „Schau gern in unsere
  aktuelle Karte auf der Website")
- Reservierungen für mehr als 15 Personen selbst bestätigen
  (dann: „Ich leite Ihre Anfrage direkt an unser Team weiter")

Reservierung aufnehmen: Frage nach Datum, Uhrzeit, Personenzahl, 
Name, Telefonnummer. Bestätige erst wenn alle Infos vorliegen.

Öffnungszeiten: [HIER EINTRAGEN]
Adresse: [HIER EINTRAGEN]
```

### n8n Session-Management (MariaDB)

```sql
CREATE TABLE wa_sessions (
    id            INT AUTO_INCREMENT PRIMARY KEY,
    phone_number  VARCHAR(20) NOT NULL,
    channel       ENUM('whatsapp','website') DEFAULT 'whatsapp',
    context       JSON,          -- Gesprächsverlauf als JSON-Array
    intent        VARCHAR(30),
    reservation_id INT,
    last_activity TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_phone (phone_number),
    INDEX idx_activity (last_activity)
);
```

Sessions die älter als 24h sind werden als neu behandelt (Timeout).

---

## 5. Kanal 4 – Website Chat-Widget

### Funktionsprinzip
Ein schlankes PHP-Chat-Widget wird in die bestehende Website eingebettet.
Es kommuniziert per AJAX mit einem PHP-Backend auf dem VPS,
das wiederum die Claude API aufruft – dieselbe Logik wie WhatsApp.

### Einbindung in die Website

```html
<!-- In den <body> der Website einfügen -->
<script>
  window.CHARME_CHAT_CONFIG = {
    apiUrl: 'https://charme-phone.deinvps.de/chat/api.php',
    primaryColor: '#C8A96E',   // Goldton passend zu Branding
    greeting: 'Bonjour! Wie kann ich Ihnen helfen?'
  };
</script>
<script src="https://charme-phone.deinvps.de/chat/widget.js"></script>
```

### Widget-Dateistruktur

```
/var/www/charme-phone/chat/
├── widget.js       # Chat-Widget (reines JS, kein Framework)
├── widget.css      # Styling (Farben aus Charme-du-Sud-Branding)
├── api.php         # AJAX-Endpunkt → Claude API
└── session.php     # Session-Helper (MariaDB)
```

### widget.js – Kern-Logik

```javascript
(function() {
  // Bubble-Button rechts unten
  const btn = document.createElement('button');
  btn.id = 'charme-chat-btn';
  btn.innerHTML = '💬';
  btn.style.cssText = `
    position:fixed; bottom:24px; right:24px; z-index:9999;
    width:56px; height:56px; border-radius:50%; border:none;
    background:${config.primaryColor}; color:#fff;
    font-size:24px; cursor:pointer; box-shadow:0 4px 12px rgba(0,0,0,0.2);
  `;
  
  // Chat-Fenster
  const win = document.createElement('div');
  win.id = 'charme-chat-window';
  // ... aufbauen, Messages rendern, AJAX zu api.php

  document.body.appendChild(btn);
  document.body.appendChild(win);
  
  btn.addEventListener('click', () => {
    win.style.display = win.style.display === 'none' ? 'flex' : 'none';
  });
})();
```

### api.php – AJAX-Endpunkt

```php
<?php
require_once '../config.php';
require_once '../claude.php';
require_once '../logger.php';

header('Content-Type: application/json');
header('Access-Control-Allow-Origin: *');

$input   = json_decode(file_get_contents('php://input'), true);
$message = trim($input['message'] ?? '');
$session = $input['session_id'] ?? session_id();

if (empty($message)) {
    echo json_encode(['error' => 'No message']);
    exit;
}

// Gesprächsverlauf aus DB laden
$history = loadChatHistory($session, 'website');

// Claude aufrufen (gleicher Prompt wie WhatsApp)
$reply = callClaudeWithHistory($message, $history, CHARME_SYSTEM_PROMPT);

// History speichern
saveChatMessage($session, 'website', $message, $reply);

echo json_encode(['reply' => $reply, 'session_id' => $session]);
```

---

## 6. Zentrales Datenmodell (MariaDB)

### Tabelle: reservations

```sql
CREATE TABLE reservations (
    id              INT AUTO_INCREMENT PRIMARY KEY,
    name            VARCHAR(100) NOT NULL,
    phone           VARCHAR(20),
    date            DATE NOT NULL,
    time            TIME NOT NULL,
    guests          TINYINT NOT NULL,
    notes           TEXT,
    channel         ENUM('whatsapp','website','phone','manual'),
    status          ENUM('pending','confirmed','cancelled') DEFAULT 'pending',
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Tabelle: phone_calls

```sql
CREATE TABLE phone_calls (
    id              INT AUTO_INCREMENT PRIMARY KEY,
    caller_number   VARCHAR(20),
    call_sid        VARCHAR(64),
    intent          VARCHAR(20),
    speech_result   TEXT,
    whatsapp_sent   TINYINT(1) DEFAULT 0,
    escalated       TINYINT(1) DEFAULT 0,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Tabelle: chat_sessions

```sql
CREATE TABLE chat_sessions (
    id              INT AUTO_INCREMENT PRIMARY KEY,
    session_id      VARCHAR(64) NOT NULL,
    channel         ENUM('whatsapp','website'),
    phone_number    VARCHAR(20),
    history         JSON,
    reservation_id  INT,
    last_activity   TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_session (session_id),
    INDEX idx_phone (phone_number)
);
```

---

## 7. n8n Flows – Übersicht

| Flow | Trigger | Aktionen |
|------|---------|----------|
| WA Eingehend | WhatsApp Webhook | → Claude → Antwort senden |
| Telefonanruf-Log | Webhook von PHP | → Notion-Log |
| Reservierungs-Bestätigung | DB Insert (reservations) | → WA-Bestätigung an Gast, → Mitarbeiter-Info |
| Eskalations-Alert | Webhook escalate.php | → WhatsApp an Mitarbeiter |
| Tages-Summary | Cron täglich 22:00 | → Reservierungen des nächsten Tages per WA |

---

## 8. Vollständige Kanal-Matrix

### Was passiert wo?

| Situation | Kanal 1 Festnetz | Kanal 2 Handy | Kanal 3 WhatsApp | Kanal 4 Website |
|-----------|-----------------|---------------|-----------------|----------------|
| Reservierung | KI → WA | Mitarbeiter | KI direkt | KI direkt |
| Öffnungszeiten | KI → WA | Mitarbeiter | KI direkt | KI direkt |
| Beschwerde | → Mitarbeiter | Mitarbeiter | → Mitarbeiter-Alert | → Mitarbeiter-Alert |
| Großgruppe (>15) | KI → WA | Mitarbeiter | → Mitarbeiter-Alert | → Mitarbeiter-Alert |
| Keine Antwort | Voicemail | Voicemail | n8n weiter | n8n weiter |

---

## 9. WhatsApp Client löschen – Detaillierter Plan

```
TAG 1 – Vorbereitung:
  ☐ WhatsApp: Einstellungen → Chats → Chat-Backup → Jetzt sichern
  ☐ Google Drive / iCloud: Backup verifizieren
  ☐ Wichtige Chats als Screenshots oder PDF exportieren
  ☐ Kontakte informieren: „Wir wechseln auf WhatsApp Business"

TAG 2 – Umstellung:
  ☐ WhatsApp-App deinstallieren
  ☐ Meta Business Manager: Nummer hinzufügen
  ☐ Verifizierungs-SMS empfangen (braucht SIM-Karte)
  ☐ WA-Sende-API: Nummer registrieren
  ☐ n8n Webhook: Eingehende Nachrichten testen
  ☐ Profilbild + Business-Infos setzen

TAG 3 – Test:
  ☐ Testnachricht von privater Nummer senden
  ☐ n8n Flow prüfen (Claude antwortet?)
  ☐ Reservierungs-Flow komplett durchspielen
  ☐ Eskalations-Test (Mitarbeiter-Alert)
```

---

## 10. Branding-Konsistenz über alle Kanäle

### Tonalität (für alle Claude-Prompts)

- Begrüßung immer mit „Bonjour!" oder „Bonjour! Guten Tag!"
- Abschluss immer mit „Merci!" oder „Auf Wiederhören / Wiederschreiben und merci!"
- Duzen vs. Siezen: **immer Siezen** (gehobenes Café-Restaurant)
- Emojis: sparsam aber erlaubt (🍃 ☕ 🥐 ✨)
- Fehler oder Limits: charmant kommunizieren, nie technisch/steif

### Beispiel-Antworten

*Reservierungsbestätigung WhatsApp:*
> Bonjour! Ich habe Ihre Reservierung für **3 Personen am Samstag, 24. Mai um 19:00 Uhr** notiert. Wir freuen uns auf Sie! Falls Sie Änderungen wünschen, schreiben Sie uns einfach. Merci! 🍃

*Öffnungszeiten Website-Chat:*
> Bonjour! Wir sind für Sie da: [Öffnungszeiten]. Möchten Sie gleich einen Tisch reservieren? ☕

---

## 11. Kostenübersicht Gesamt (monatlich)

| Dienst | Kanal | Kosten |
|--------|-------|--------|
| Twilio Nummer +49 | Festnetz | ~1,15 $/Monat |
| Twilio Anrufe + STT (75 Min) | Festnetz | ~3,00 $ |
| ElevenLabs (bereits vorhanden) | Festnetz | 0 € extra |
| Claude API – alle Kanäle (200 Calls) | WA + Web + Tel | ~0,40 $ |
| WA-Sende-API (bereits vorhanden) | WhatsApp | 0 € extra |
| n8n (bereits vorhanden) | Alle | 0 € extra |
| DomainFactory VPS (bereits vorhanden) | Alle | 0 € extra |
| **Gesamt** | | **~5–6 € / Monat** |

---

## 12. Implementierungs-Reihenfolge

### Phase 1 – Fundament (1 Claude Code Session)
```
☐ config.php (alle Kanäle konfigurierbar)
☐ claude.php (API Wrapper mit History-Support)
☐ logger.php + MariaDB-Setup (alle 3 Tabellen)
☐ tts.php (ElevenLabs mit Cache)
```

### Phase 2 – Kanal 1: Festnetz-Voice (1 Session)
```
☐ index.php (Twilio Begrüßung)
☐ handle-intent.php (Claude Intent)
☐ confirm-whatsapp.php
☐ send-whatsapp.php (WA-Sende-API)
☐ escalate.php
☐ scripts/generate-audio.php
```

### Phase 3 – Kanal 3: WhatsApp (1 Session)
```
☐ WA Client löschen + Meta Business API aktivieren
☐ n8n Flow: Eingehende WA → Claude → Antwort
☐ n8n Flow: Reservierungs-Bestätigung
☐ n8n Flow: Eskalations-Alert
☐ Session-Management in MariaDB
```

### Phase 4 – Kanal 4: Website-Widget (1 Session)
```
☐ chat/widget.js (Chat-Bubble)
☐ chat/widget.css (Charme-du-Sud-Branding)
☐ chat/api.php (AJAX → Claude)
☐ Einbindung in bestehende Website
☐ End-to-End-Test aller Kanäle
```

---

## 13. Offene Punkte vor Start

```
☐ Twilio-Verifizierung starten (+49 dauert 1–2 Tage)
☐ Subdomain festlegen: charme-phone.deinvps.de
☐ ElevenLabs Voice Clone ID bereithalten
☐ Mitarbeiter-Handy für Eskalation definieren
☐ Festnetz-Weiterleitung beim Anbieter klären
☐ WhatsApp-Backup durchführen (vor Client-Löschung)
☐ Öffnungszeiten + Adresse für System-Prompt bereithalten
☐ Branding-Farben der Website (für Chat-Widget CSS)
☐ WA-Sende-API Endpunkt + Payload-Format verifizieren
```

---

*Dieses Dokument ist die verbindliche Grundlage für alle Claude Code Implementierungs-Sessions.*  
*Alle vier Kanäle werden als ein kohärentes System behandelt – gleicher Ton, gleiche DB, gleiche Claude-Logik.*
