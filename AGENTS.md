# Phone Assistant (VAPI-Telefonbot + Omnichannel)

**Docs-only Repo — kein Code, kein VPS-Deployment.**
Der Bot läuft als VAPI-Agent; Webhooks/Weiterleitung über n8n
(`https://n8n.gevio.cloud/webhook/vapi-forwarder`), WhatsApp-Versand als Tool.

## Pflichtlektüre vor Arbeiten

- `docs/charme-phone-spec.md` — Telefonbot-Spezifikation (PDF-Version vorhanden)
- `docs/charme-omnichannel-spec.md` — Omnichannel-Erweiterung (PDF vorhanden)
- `docs/CDS-Bot-Dokumentation.md` — Gesamtdokumentation inkl. Changelog
- `docs/CDS-Bot-Fix-2026-08-30.md` — letzter größerer Fix (Tool-Server-URL,
  Öffnungszeiten-Check)

## Regeln

- Änderungen am Bot selbst passieren in der VAPI-/n8n-Oberfläche —
  hier dokumentieren, was sich geändert hat (neue Fix-Datei oder Changelog-Eintrag)
- Specs ändern sich selten; bei Änderungen auch die PDF neu erzeugen (`make_pdf.py`-Prinzip)
- Secrets (VAPI-Keys, Webhook-Tokens) gehören NIEMALS in diese Docs
