# Predictive Vehicle Care

## Projektbeschreibung
DSGVO-konformes Predictive-Maintenance-System fuer KFZ-Werkstaetten.
Laeuft auf NVIDIA DGX Spark (ARM64, Ubuntu 24.04).

## Architektur
- **N8N**: Zentrale Workflow-Engine — gesamte Geschaeftslogik (TUeV-Erinnerungen, saisonale Checks, Gesundheitspass, Terminbuchung, SMS-Versand, Health-Score-Berechnung)
- **PostgreSQL 16**: Consent-Datenbank (getrennt von Werkstattdaten, pgcrypto-verschluesselt)
- **MS SQL Server**: Werkstatt-DB (Read-Only ueber Views, kein Schreibzugriff)
- **Caddy**: Reverse Proxy mit automatischem TLS + statische HTML-Seiten (Buchungsformular, Gesundheitspass)
- **Ollama**: Lokales LLM (nemotron-3-super) auf DGX Spark GPU fuer KI-gestuetzte Analysen
  120B Parameter MoE-Architektur, 12B aktive Parameter, ~87GB VRAM,
  optimiert fuer DGX Spark Blackwell GPU. Kein Cloud-API-Call, vollstaendig lokal.
- **AnythingLLM**: RAG-Interface fuer Werkstattbetreiber (Chat mit eigenen Daten)

## Docker-Netzwerke
- `backend_net`: Caddy <-> N8N <-> PostgreSQL <-> Ollama <-> AnythingLLM
- `db_net`: N8N <-> MS SQL Server (nur Read-Only)

## Services und Ports
| Service       | Container         | Port (intern) | Zugang                      |
|---------------|-------------------|---------------|-----------------------------|
| Caddy         | pvc-caddy         | 80, 443       | werkstatt.example.de        |
| N8N           | pvc-n8n           | 5678          | n8n.werkstatt.example.de    |
| AnythingLLM   | pvc-anythingllm   | 3001          | chat.werkstatt.example.de   |
| Ollama        | pvc-ollama        | 11434         | nur intern (backend_net)    |
| PostgreSQL    | pvc-postgres      | 5432          | nur intern (backend_net)    |

## Statische Seiten (Caddy)
- `caddy/static/buchen.html` — Buchungsformular (Tailwind CSS, Token aus URL, POST an N8N Webhook)
- `caddy/static/gesundheitspass.html` — Fahrzeug-Gesundheitspass (Tailwind CSS, Ampelsystem, GET von N8N Webhook)

## N8N Webhooks
- `POST /webhook/booking` — Terminbuchung empfangen (aus buchen.html)
- `GET  /webhook/booking?token=...` — Buchungsdaten fuer Token abrufen
- `GET  /webhook/health-score?vehicleId=...` — Gesundheitspass-Daten abrufen

## Wichtige Konventionen
- Sprache im Code: Englisch. Kommentare/Docs: Deutsch erlaubt.
- Secrets IMMER in `.env`, NIE im Code oder Git.
- `.env` ist in `.gitignore` — nur `.env.example` wird committed.
- SQL-Migrationen in `db/migrations/` mit Prefix `NNNN_beschreibung.sql`.
- N8N-Workflows als JSON in `n8n/workflows/`.

## Health-Score-Engine
Gewichtung der Komponenten:
- TUeV-Status: 30%
- Wartungsintervall: 25%
- Fahrzeugalter: 15%
- Reifenzustand: 15%
- Offene Positionen: 15%

Ampelsystem: Gruen (80-100), Gelb (50-79), Rot (0-49)

## SMS-Integration (brevis.one)
- Primaerer Kanal: SMS ueber brevis.one HTTP-API
- Fallback: Automatischer E-Mail-Versand bei SMS-Fehler (SMTP)
- Alle Zustellversuche werden in `reminder_log` protokolliert
- SMS-Versand laeuft komplett ueber N8N-Workflows

## N8N <-> Ollama Integration
N8N spricht Ollama ueber HTTP-Request Node an:
- URL: `http://ollama:11434/api/chat`
- Methode: POST
- Modell: nemotron-3-super (konfigurierbar via OLLAMA_MODEL)

## Befehle
- `docker compose up -d` — System starten
- `docker compose down` — System stoppen
- `docker compose logs -f [service]` — Logs verfolgen
- `./scripts/init-db.sh` — Consent-DB initialisieren
- `./scripts/backup-consent-db.sh` — Consent-DB Backup
- `./scripts/pull-ollama-model.sh` — Ollama Modell herunterladen

## Sicherheitsregeln
- Kein Schreibzugriff auf MS SQL (Werkstatt-DB).
- Consent-DB: Personenbezogene Daten mit pgcrypto verschluesseln.
- Booking-Tokens: 14 Tage Gueltigkeit, einmalige Nutzung, kryptografisch sicher.
- Docker Volumes mit Verschluesselung.
- Netzwerkisolation strikt einhalten.
- Ollama und AnythingLLM nur intern erreichbar (kein direkter Port-Export in Produktion).

## MVP Features
1. TUeV/HU-Erinnerung (gestaffelt: 8/4/1 Woche vorher)
2. One-Click Terminbuchung (tokenbasierte Links, 14 Tage gueltig)
3. Saisonale Checks (Reifenwechsel, Klima, Batterie, Lichttest)
4. Digitaler Fahrzeug-Gesundheitspass (Score 0-100, Ampelsystem)
5. KI-Chat fuer Werkstattbetreiber (AnythingLLM + Ollama RAG)
6. SMS-Benachrichtigungen mit E-Mail-Fallback (brevis.one)

## Skills
- Lese und befolge die Anweisungen in .claude/skills/frontend-design/SKILL.md bei jeder Frontend-Arbeit.
