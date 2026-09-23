# Charme du Sud – Automatischer Telefonassistent
## Claude Code Implementierungs-Spezifikation

**Projekt:** charme-phone  
**Version:** 1.0  
**Stack:** PHP 8.x · Twilio · ElevenLabs · Claude API · WA-Sende-API  
**Hosting:** DomainFactory VPS (bestehend)  
**Datum:** Mai 2026

---

## 1. Projektziel

Eingehende Anrufe auf der Festnetznummer von Charme du Sud (Weisenheim am Berg) werden automatisch von einem KI-Agenten entgegengenommen. Der Agent:

1. Begrüßt den Anrufer auf Deutsch (mit leichtem französischen Charme)
2. Erkennt das Anliegen (Reservierung, Frage, Beschwerde, Mitarbeiter)
3. Fragt ob eine Fortsetzung per WhatsApp gewünscht ist
4. Übergibt an WhatsApp-Bot ODER leitet an Mitarbeiter weiter

---

## 2. Systemarchitektur

```
Festnetz-Anruf
      │
      ▼ (Weiterleitung)
Twilio Voice Number (+49)
      │
      ▼ Webhook POST
index.php  ←→  ElevenLabs TTS
      │
      ▼ SpeechResult POST
handle-intent.php  ←→  Claude API
      │
   ┌──┴──────────────┐
   ▼                 ▼
whatsapp.php    escalate.php
(WA-Sende-API)  (Twilio Transfer)
   │
   ▼
n8n Webhook (optional: Notion-Log)
```

---

## 3. Dateistruktur

```
/var/www/charme-phone/
├── config.php              # API Keys & Konfiguration
├── index.php               # Twilio Einstiegs-Webhook (Begrüßung)
├── handle-intent.php       # Claude Intent-Klassifizierung + Antwort
├── confirm-whatsapp.php    # WA-Bestätigung einholen, Nummer prüfen
├── send-whatsapp.php       # WA-Sende-API Wrapper + Übergabe
├── escalate.php            # Weiterleitung an Mitarbeiter
├── tts.php                 # ElevenLabs TTS Helper
├── claude.php              # Anthropic API Wrapper
├── logger.php              # MariaDB Logging
└── audio/                  # Gecachte ElevenLabs MP3-Dateien
    ├── greeting.mp3
    ├── ask_intent.mp3
    └── whatsapp_offer.mp3
```

---

## 4. Konfiguration (config.php)

```php
<?php
// config.php

// Twilio
define('TWILIO_ACCOUNT_SID', 'ACxxxxxxxxxxxxxxxxxx');
define('TWILIO_AUTH_TOKEN',  'xxxxxxxxxxxxxxxxxxxxxxxx');
define('TWILIO_NUMBER',      '+49XXXXXXXXXX');

// ElevenLabs
define('ELEVENLABS_API_KEY', 'xxxxxxxxxxxxxxxxxxxxxxxx');
define('ELEVENLABS_VOICE_ID', 'xxxxxxxxxxxxxxxx'); // Dein Voice Clone

// Anthropic Claude
define('ANTHROPIC_API_KEY',  'sk-ant-xxxxxxxxxxxxxxxx');
define('CLAUDE_MODEL',       'claude-sonnet-4-20250514');

// WA-Sende-API
define('WASENDE_API_KEY',    'xxxxxxxxxxxxxxxxxxxxxxxx');
define('WASENDE_SENDER',     '+49XXXXXXXXXX'); // WhatsApp Business Nummer

// Mitarbeiter-Weiterleitung
define('STAFF_PHONE',        '+49XXXXXXXXXX');

// Eigene Domain
define('BASE_URL',           'https://charme-phone.deinvps.de');

// MariaDB (bestehende Charme-du-Sud-DB)
define('DB_HOST', 'localhost');
define('DB_NAME', 'charme_du_sud');
define('DB_USER', 'charme_phone_user');
define('DB_PASS', 'xxxxxxxx');
```

---

## 5. Kern-Module

### 5.1 index.php – Begrüßung & erste Frage

**Trigger:** Twilio ruft diese URL bei jedem eingehenden Anruf auf.

**Logik:**
- Caller-ID aus `$_POST['From']` extrahieren und loggen
- ElevenLabs-Audio abspielen (gecacht oder frisch generiert)
- Twilio `<Gather>` mit `input="speech"` öffnen
- Timeout 6 Sekunden, danach Fallback

**TwiML-Antwort:**
```xml
<Response>
  <Play>https://charme-phone.deinvps.de/audio/greeting.mp3</Play>
  <Gather input="speech" language="de-DE" 
          action="/handle-intent.php" 
          timeout="6" speechTimeout="2">
    <Play>https://charme-phone.deinvps.de/audio/ask_intent.mp3</Play>
  </Gather>
  <!-- Fallback wenn keine Spracheingabe -->
  <Redirect>/index.php</Redirect>
</Response>
```

**Gecachte Audio-Texte (ElevenLabs vorher generieren):**

- `greeting.mp3`: *„Bonjour! Guten Tag und herzlich willkommen bei Charme du Sud in Weisenheim am Berg."*
- `ask_intent.mp3`: *„Wie kann ich Ihnen helfen? Möchten Sie einen Tisch reservieren, haben Sie eine Frage zu unseren Öffnungszeiten oder der Karte, oder möchten Sie mit jemandem aus unserem Team sprechen?"*
- `whatsapp_offer.mp3`: *„Sehr gern! Darf ich mich im Anschluss kurz per WhatsApp bei Ihnen melden? So kann ich Ihnen alle Details bequem schicken."*

---

### 5.2 handle-intent.php – Claude Intent-Erkennung

**Trigger:** Twilio POST mit `SpeechResult` (transkribierter Text)

**Logik:**
1. `SpeechResult` aus POST empfangen
2. An Claude API senden zur Klassifizierung
3. Je nach Intent verzweigen

**Claude-Prompt (System):**
```
Du bist der Telefonassistent von Charme du Sud, einem charmanten 
französisch inspirierten Café-Restaurant in Weisenheim am Berg, 
Rheinland-Pfalz. Du sprichst freundlich, professionell und mit 
einem leichten französischen Flair.

Klassifiziere die Aussage des Anrufers in GENAU EINE dieser Kategorien:
- RESERVATION  (Tisch reservieren, buchen, Platz anfragen)
- QUESTION     (Öffnungszeiten, Karte, Preise, Anfahrt, allgemeine Info)
- COMPLAINT    (Beschwerde, Reklamation, Problem)
- HUMAN        (Mitarbeiter sprechen, Chef, Person)
- UNCLEAR      (unverständlich, zu kurz, Hintergrundgeräusche)

Antworte NUR mit dem Stichwort. Keine Erklärung.
```

**Intent-Routing:**

| Intent | Aktion |
|--------|--------|
| RESERVATION | → `confirm-whatsapp.php` mit Kontext "Reservierung" |
| QUESTION | → `confirm-whatsapp.php` mit Kontext "Frage" |
| COMPLAINT | → `escalate.php` (direkt Mitarbeiter) |
| HUMAN | → `escalate.php` (direkt Mitarbeiter) |
| UNCLEAR | → Nachfragen (max. 1x), dann `escalate.php` |

---

### 5.3 confirm-whatsapp.php – WhatsApp-Angebot

**Trigger:** Redirect von handle-intent.php

**Logik:**
- Caller-ID prüfen (aus Session/POST)
- WhatsApp-Angebot aussprechen
- Auf Ja/Nein warten

**TwiML:**
```xml
<Response>
  <Play>[whatsapp_offer.mp3]</Play>
  <Gather input="speech" language="de-DE" 
          action="/send-whatsapp.php?context=RESERVATION"
          timeout="5">
    <Say language="de-DE">Bitte sagen Sie Ja oder Nein.</Say>
  </Gather>
  <Redirect>/escalate.php</Redirect>
</Response>
```

**Wenn Caller-ID nicht vorhanden** (unterdrückte Nummer):
- Anrufer nach Telefonnummer fragen
- Twilio `<Gather input="dtmf">` für Nummerneingabe nutzen

---

### 5.4 send-whatsapp.php – WA-Sende-API Integration

**Trigger:** Bestätigung durch Anrufer

**WA-Sende-API Aufruf:**
```php
function sendWhatsApp(string $to, string $message): bool {
    $url = 'https://api.wa-sende.de/v1/messages'; // Endpunkt prüfen
    
    $payload = [
        'apikey'    => WASENDE_API_KEY,
        'sender'    => WASENDE_SENDER,
        'recipient' => normalizePhone($to),
        'message'   => $message,
        'type'      => 'text'
    ];
    
    $ch = curl_init($url);
    curl_setopt_array($ch, [
        CURLOPT_POST           => true,
        CURLOPT_POSTFIELDS     => json_encode($payload),
        CURLOPT_HTTPHEADER     => ['Content-Type: application/json'],
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_TIMEOUT        => 10,
    ]);
    
    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);
    
    return $httpCode === 200;
}
```

**WhatsApp-Erstnachricht je nach Kontext:**

*Reservierung:*
> Bonjour! 👋 Hier ist Charme du Sud. Sie haben gerade angerufen und möchten einen Tisch reservieren. Für wie viele Personen und wann darf ich einen Tisch für Sie buchen?

*Frage:*
> Bonjour! 👋 Hier ist Charme du Sud. Ich helfe Ihnen gerne weiter. Was möchten Sie wissen?

**Abschluss am Telefon:**
```xml
<Response>
  <Say language="de-DE">
    Wunderbar! Ich habe Ihnen gerade eine WhatsApp-Nachricht geschickt. 
    Wir melden uns dort gleich weiter. Auf Wiederhören und merci!
  </Say>
  <Hangup/>
</Response>
```

---

### 5.5 escalate.php – Weiterleitung an Mitarbeiter

**Trigger:** COMPLAINT, HUMAN, oder Anrufer möchte kein WhatsApp

**Logik:**
```xml
<Response>
  <Say language="de-DE">
    Einen Moment bitte, ich verbinde Sie mit unserem Team.
  </Say>
  <Dial timeout="20" callerId="+49XXXXXXXXXX">
    <Number>+49XXXXXXXXXX</Number>
  </Dial>
  <!-- Fallback wenn nicht abgenommen -->
  <Say language="de-DE">
    Im Moment ist leider niemand erreichbar. Hinterlassen Sie uns bitte 
    eine Nachricht, wir rufen Sie zurück.
  </Say>
  <Record maxLength="60" transcribe="false"/>
  <Hangup/>
</Response>
```

---

### 5.6 tts.php – ElevenLabs Helper

```php
function generateTTS(string $text, string $filename): string {
    $cacheFile = __DIR__ . '/audio/' . $filename . '.mp3';
    
    // Cache prüfen
    if (file_exists($cacheFile)) {
        return BASE_URL . '/audio/' . $filename . '.mp3';
    }
    
    // ElevenLabs API
    $url = 'https://api.elevenlabs.io/v1/text-to-speech/' . ELEVENLABS_VOICE_ID;
    
    $payload = [
        'text'     => $text,
        'model_id' => 'eleven_multilingual_v2',
        'voice_settings' => [
            'stability'        => 0.6,
            'similarity_boost' => 0.85,
        ]
    ];
    
    $ch = curl_init($url);
    curl_setopt_array($ch, [
        CURLOPT_POST           => true,
        CURLOPT_POSTFIELDS     => json_encode($payload),
        CURLOPT_HTTPHEADER     => [
            'xi-api-key: ' . ELEVENLABS_API_KEY,
            'Content-Type: application/json',
            'Accept: audio/mpeg',
        ],
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_TIMEOUT        => 15,
    ]);
    
    $audioData = curl_exec($ch);
    curl_close($ch);
    
    file_put_contents($cacheFile, $audioData);
    return BASE_URL . '/audio/' . $filename . '.mp3';
}
```

---

### 5.7 claude.php – Anthropic API Wrapper

```php
function callClaude(string $userMessage, string $systemPrompt = ''): string {
    $url = 'https://api.anthropic.com/v1/messages';
    
    $payload = [
        'model'      => CLAUDE_MODEL,
        'max_tokens' => 100,
        'system'     => $systemPrompt,
        'messages'   => [
            ['role' => 'user', 'content' => $userMessage]
        ]
    ];
    
    $ch = curl_init($url);
    curl_setopt_array($ch, [
        CURLOPT_POST           => true,
        CURLOPT_POSTFIELDS     => json_encode($payload),
        CURLOPT_HTTPHEADER     => [
            'x-api-key: ' . ANTHROPIC_API_KEY,
            'anthropic-version: 2023-06-01',
            'Content-Type: application/json',
        ],
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_TIMEOUT        => 10,
    ]);
    
    $response = json_decode(curl_exec($ch), true);
    curl_close($ch);
    
    return trim($response['content'][0]['text'] ?? 'UNCLEAR');
}
```

---

### 5.8 logger.php – Anruf-Logging in MariaDB

**Tabelle erstellen:**
```sql
CREATE TABLE phone_calls (
    id            INT AUTO_INCREMENT PRIMARY KEY,
    caller_number VARCHAR(20),
    call_sid      VARCHAR(64),
    intent        VARCHAR(20),
    speech_result TEXT,
    whatsapp_sent TINYINT(1) DEFAULT 0,
    escalated     TINYINT(1) DEFAULT 0,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**PHP-Funktion:**
```php
function logCall(array $data): void {
    $pdo = new PDO(
        'mysql:host=' . DB_HOST . ';dbname=' . DB_NAME . ';charset=utf8',
        DB_USER, DB_PASS
    );
    $stmt = $pdo->prepare("
        INSERT INTO phone_calls 
        (caller_number, call_sid, intent, speech_result, whatsapp_sent, escalated)
        VALUES (?, ?, ?, ?, ?, ?)
    ");
    $stmt->execute([
        $data['caller']   ?? '',
        $data['call_sid'] ?? '',
        $data['intent']   ?? '',
        $data['speech']   ?? '',
        $data['whatsapp'] ?? 0,
        $data['escalated'] ?? 0,
    ]);
}
```

---

## 6. Gesprächsskript (vollständig)

### Szenario A – Reservierung via WhatsApp

```
📞 Anruf eingehend

AGENT: „Bonjour! Guten Tag und herzlich willkommen bei Charme du Sud 
        in Weisenheim am Berg. Wie kann ich Ihnen helfen? Möchten Sie 
        einen Tisch reservieren, haben Sie eine Frage, oder möchten 
        Sie mit jemandem aus unserem Team sprechen?"

KUNDE: „Ich würde gerne einen Tisch reservieren."

AGENT: „Wie schön, dass Sie zu uns kommen möchten! 
        Darf ich mich im Anschluss kurz per WhatsApp bei Ihnen melden? 
        So kann ich Ihnen alle Details bequem schicken und wir klären 
        alles in Ruhe. Ist das für Sie in Ordnung?"

KUNDE: „Ja, gerne."

AGENT: „Wunderbar! Ich habe Ihnen gerade eine Nachricht geschickt. 
        Wir sind gleich für Sie da. Auf Wiederhören und merci!"

[WhatsApp wird gesendet]
[Telefon beendet]
```

### Szenario B – Frage per WhatsApp

```
AGENT: [Begrüßung]
KUNDE: „Was sind eure Öffnungszeiten?"
AGENT: [Intent: QUESTION]
       „Sehr gerne helfe ich Ihnen weiter! Darf ich mich kurz per 
        WhatsApp bei Ihnen melden? Da kann ich Ihnen alle Infos 
        übersichtlich schicken."
KUNDE: „Ja."
[WhatsApp mit Öffnungszeiten + Link zur Website]
```

### Szenario C – Direktweiterleitung

```
AGENT: [Begrüßung]
KUNDE: „Ich habe eine Beschwerde, ich möchte den Chef sprechen."
AGENT: [Intent: COMPLAINT / HUMAN]
       „Ich verstehe, einen Moment bitte, ich verbinde Sie sofort."
[Weiterleitung auf Mitarbeiter-Handy]
```

### Szenario D – Unterdrückte Nummer

```
AGENT: [Begrüßung + Intent erkannt]
       „Gerne helfe ich Ihnen per WhatsApp weiter. Unter welcher 
        Nummer darf ich Sie kontaktieren?"
KUNDE: [gibt Nummer ein per Tastatur oder Sprache]
[Nummer validieren → WhatsApp senden]
```

---

## 7. Twilio-Setup (Schritt für Schritt)

1. **Account anlegen:** console.twilio.com
2. **Verifizierung:** deutsche Adresse + Unternehmensnachweis für +49-Nummern
3. **Nummer kaufen:** „Phone Numbers" → Search → Germany → Local (+49)
4. **Webhook setzen:**  
   - Voice → „A call comes in" → Webhook  
   - URL: `https://charme-phone.deinvps.de/index.php`  
   - Method: HTTP POST
5. **Festnetz-Weiterleitung:**  
   - Beim Telefonanbieter die bestehende +49-Festnetznummer auf die Twilio-Nummer umleiten
   - Alternativ: Twilio-Nummer direkt als neue Hauptnummer nutzen

---

## 8. Deployment auf DomainFactory VPS

```bash
# 1. Verzeichnis anlegen
mkdir -p /var/www/charme-phone/audio
cd /var/www/charme-phone

# 2. Dateien deployen (via git oder scp)
git clone [repo] .

# 3. config.php befüllen (aus config.example.php)
cp config.example.php config.php
nano config.php

# 4. Audio-Verzeichnis schreibbar machen
chmod 755 audio/

# 5. PHP-Abhängigkeiten (keine composer nötig – reines PHP)

# 6. Audio-Dateien vorab generieren
php scripts/generate-audio.php

# 7. Nginx/Apache: Virtual Host anlegen
# Wichtig: HTTPS zwingend (Twilio sendet nur an HTTPS)
# SSL via Let's Encrypt (certbot)

# 8. MariaDB: Tabelle anlegen
mysql -u charme_phone_user -p charme_du_sud < sql/setup.sql
```

**Nginx Minimalconfig:**
```nginx
server {
    listen 443 ssl;
    server_name charme-phone.deinvps.de;
    root /var/www/charme-phone;
    index index.php;
    
    location ~ \.php$ {
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
    
    # Audio-Dateien direkt ausliefern
    location /audio/ {
        add_header Access-Control-Allow-Origin *;
    }
}
```

---

## 9. n8n-Integration (optional)

Nach jedem WhatsApp-Versand kann `send-whatsapp.php` einen n8n-Webhook triggern:

```php
// In send-whatsapp.php, nach erfolgreichem WA-Versand:
$n8nPayload = [
    'caller'    => $callerNumber,
    'intent'    => $context,
    'timestamp' => date('c'),
    'source'    => 'charme-phone'
];
// POST an n8n Webhook URL
triggerN8nWebhook($n8nPayload);
```

**n8n-Flow:**
- Eingehender Webhook
- Eintrag in Notion-Datenbank (Kontaktlog)
- Optional: Tages-Summary per WhatsApp an Inhaber

---

## 10. Kostenübersicht (monatlich, Schätzung)

| Dienst | Kosten |
|--------|--------|
| Twilio Nummer (+49) | ~1,15 $/Monat |
| Twilio Eingehende Anrufe | ~0,0085 $/Min |
| Twilio Speech-to-Text | ~0,02 $/Min |
| ElevenLabs (Starter, bereits vorhanden) | 0 € extra |
| Claude API (Sonnet, ~50 Calls/Monat) | ~0,10 $ |
| WA-Sende-API (bereits vorhanden) | 0 € extra |
| **Gesamt (50 Anrufe × 1,5 Min)** | **~5–8 €/Monat** |

---

## 11. Implementierungs-Reihenfolge für Claude Code

### Session 1 – Fundament
1. `config.php` (Beispiel-Version ohne echte Keys)
2. `claude.php` – API Wrapper
3. `tts.php` – ElevenLabs Helper
4. `logger.php` – MariaDB Logging + SQL-Setup
5. `scripts/generate-audio.php` – Audio vorab generieren

### Session 2 – Twilio-Flow
6. `index.php` – Begrüßungs-Webhook
7. `handle-intent.php` – Intent-Erkennung
8. `confirm-whatsapp.php` – WA-Angebot

### Session 3 – Ausgabe & Eskalation
9. `send-whatsapp.php` – WA-Sende-API
10. `escalate.php` – Mitarbeiter-Weiterleitung
11. End-to-End-Test mit Twilio Trial

---

## 12. Offene Punkte / Entscheidungen

- [ ] Subdomain für VPS festlegen (`charme-phone.` oder Unterverzeichnis)
- [ ] Twilio-Verifizierung für deutsche +49-Nummern starten (kann 1–2 Tage dauern)
- [ ] WA-Sende-API Endpunkt & Payload-Format verifizieren (Dokumentation prüfen)
- [ ] ElevenLabs Voice Clone ID bereithalten
- [ ] Mitarbeiter-Weiterleitung: Handy-Nummer definieren
- [ ] Festnetz-Weiterleitungs-Setup mit Telefonanbieter klären

---

*Dieses Dokument dient als alleinige Grundlage für die Claude Code Implementierungs-Session. Alle Pfade, Klassennamen und Prompts sind verbindlich.*
