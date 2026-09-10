# ilv-modul-109

Dieses Repository stellt eine Docker-Umgebung mit Authentisierung zur Verfügung.

Es demonstriert den klassischen Aufbau eines Reverse-Proxy-Stacks mit vorgeschalteter Authentisierung: **Traefik** terminiert und routet den eingehenden Traffic, **Authelia** übernimmt Login/2FA als Forward-Auth-Middleware, und zwei einfache "whoami"-Testdienste zeigen den Unterschied zwischen geschütztem und offenem Zugriff.

## Architektur

```
Client → Traefik (Port 80/443) → [Authelia Forward-Auth] → service1.ilv.local  (geschützt)
                                └───────────────────────────→ service2.ilv.local  (offen)
```

Alle vier Compose-Projekte hängen am gemeinsamen Docker-Netzwerk `traefik_proxy`. `authelia`, `service1` und `service2` binden es als **externes** Netzwerk ein (`external: true`); `traefik` selbst legt es beim Start implizit an (Namensschema `<Ordnername>_proxy` → `traefik_proxy`, da der Ordner `traefik` heisst). **Deshalb muss traefik immer zuerst gestartet werden.**

## Komponenten

### traefik/
Reverse Proxy und Einstiegspunkt für den gesamten Traffic.
- Entrypoints: `http` (Port 80, wird automatisch auf https umgeleitet) und `https` (Port 443)
- Dashboard erreichbar über `traefik.ilv.local` – die Basic-Auth-Absicherung dafür ist im Compose-File vorbereitet, aber **auskommentiert/deaktiviert**
- Docker-Provider (liest Traefik-Labels der Container automatisch) + File-Provider (liest `traefik/data/config/`)
- TLS-Zertifikat wird **statisch** über `traefik/data/config/certificates.yaml` eingebunden: `200.marti-net.ch.crt` / `.key` (liegen unter `traefik/data/cert/`) – es handelt sich also um ein echtes Zertifikat für die eigene Domain `marti-net.ch`, nicht um ein Let's-Encrypt/ACME-Zertifikat für `*.ilv.local`
- Cloudflare-API-Variablen (`CF_API_EMAIL`, `CF_DNS_API_TOKEN`) sind im Compose-File nur als Platzhalter vorhanden und werden aktuell nicht genutzt (kein ACME-Certresolver konfiguriert)
- Die Volume-Pfade im Compose-File sind **absolut** und fest auf `/docker/projekt-ilv/traefik/...` codiert – bei einem Deployment an einem anderen Ort müssen sie angepasst werden

### authelia/
Authentisierungs-Layer, bei Traefik als Forward-Auth-Middleware eingebunden.
- Erreichbar über `auth.ilv.local`, lauscht intern auf Port 9091
- Authentisierungs-Backend: lokale `config/users_database.yml` (kein LDAP), Passwörter als Argon2id-Hash
- Ein Testbenutzer ist angelegt: **`authelia` / `authelia`** (hinterlegte E-Mail: `thomas@marti-net.ch`, Gruppen `admins`, `dev`)
- Access-Control: Default-Policy `deny`; einzige Ausnahme ist `service1.ilv.local` mit `one_factor` (Passwort genügt)
- TOTP (2FA) ist aktiviert (Issuer `ilv.local`), wird aber durch die aktuelle Access-Control-Regel für `service1` nicht erzwungen
- Storage: lokale SQLite-Datenbank (`config/db.sqlite3`); Benachrichtigungen (z.B. Passwort-Reset) landen als Datei in `config/notification.txt` statt per echtem Mailversand
- JWT-Secret, Session-Secret und der Storage-Encryption-Key stehen fest im Klartext in `config/configuration.yml` – erkennbar Test-/Beispielwerte, **nicht produktionstauglich**

### service1/
Testdienst (`containous/whoami`), **hinter Authelia geschützt** – Zugriff auf `service1.ilv.local` verlangt vorher ein Login über `auth.ilv.local`.

### service2/
Identischer Testdienst, aber **ohne Authelia-Middleware** (in der `docker-compose.yml` auskommentiert) – `service2.ilv.local` ist frei zugänglich. Dient zum direkten Vergleich mit `service1`.

## Setup / Verwendung

Auf dem Client, der die Umgebung testet, müssen folgende Einträge in die hosts-Datei eingetragen werden (Beispiel-IP aus dem Original-Setup):

```
traefik.ilv.local     10.10.10.10
auth.ilv.local        10.10.10.10
service1.ilv.local    10.10.10.10
service2.ilv.local    10.10.10.10
```

Start – Reihenfolge beachten (siehe Architektur-Hinweis oben):

```
cd traefik     && docker compose up -d
cd ../authelia && docker compose up -d
cd ../service1 && docker compose up -d
cd ../service2 && docker compose up -d
```

Login-Test: `service1.ilv.local` aufrufen → Weiterleitung zu `auth.ilv.local` → Anmeldung mit `authelia` / `authelia`. `service2.ilv.local` ist im Vergleich dazu direkt ohne Login erreichbar.

## Hinweise

- Alle Secrets in diesem Projekt (JWT-Secret, Session-Secret, Storage-Encryption-Key, Testpasswort) sind bewusst einfache Beispielwerte für Lern-/Testzwecke und sollten nicht produktiv weiterverwendet werden.
- Das Traefik-Dashboard ist ungeschützt erreichbar, sobald `traefik.ilv.local` auflösbar ist (Basic-Auth-Middleware ist vorbereitet, aber deaktiviert).
- Für die TLS-Terminierung liegt ein reales `marti-net.ch`-Zertifikat bei statt eines für `*.ilv.local` – im Browser kann das beim lokalen Testbetrieb zu einer Zertifikatswarnung wegen des abweichenden Hostnamens führen.

---
*README ergänzt am 2026-09-09 auf Basis des ursprünglichen Inhalts sowie der tatsächlichen Compose- und Config-Dateien im Projekt.*
