# FFW IoT Docker Stack

IoT-Infrastruktur der Freiwilligen Feuerwehr Wörnitzthal auf Basis von Docker Compose.

## Übersicht

Der Stack betreibt eine vollständige Smart-Home- und Gebäudeautomation mit zentralem Reverse Proxy, Authentifizierung und MQTT-Messaging.

**Domain**: `*.iot.vpn.ffw-it.de`
**Auth**: Authentik SSO unter `auth.ffw-it.de`
**Netzwerk**: Bridge `172.18.0.0/24`

---

## Schnellstart

```bash
# .env anlegen (aus Vorlage)
cp .env.example .env
# HETZNER_API_TOKEN und WATCHTOWER_NOTIFICATION_URL eintragen

# Authentik-Token setzen
cp authentik.env.example authentik.env
# AUTHENTIK_TOKEN eintragen

# MQTT-Passwortdatei anlegen (siehe Abschnitt mqtt)

# Stack starten
docker compose up -d
```

---

## Umgebungsvariablen

| Datei                | Variable                      | Beschreibung                                      |
|----------------------|-------------------------------|---------------------------------------------------|
| `.env`               | `COMPOSE_PROJECT_NAME`        | Projektname für Docker Compose                    |
| `.env`               | `HETZNER_API_TOKEN`           | API-Token für Hetzner DNS-Challenge (Let's Encrypt) |
| `.env`               | `WATCHTOWER_NOTIFICATION_URL` | Benachrichtigungs-URL für Watchtower (z. B. Shoutrrr) |
| `authentik.env`      | `AUTHENTIK_TOKEN`             | Outpost-Token aus Authentik-Admin                 |

---

## Services

### traefik

**Image**: `traefik:latest`
**URL**: `https://traefik.iot.vpn.ffw-it.de` (SSO-geschützt)

Reverse Proxy für alle HTTP/HTTPS-Dienste sowie TCP-Routing für MQTT.

- HTTP (Port 80) leitet dauerhaft auf HTTPS (Port 443) um
- TLS-Zertifikate via Let's Encrypt mit Hetzner DNS-Challenge (Wildcard `*.iot.vpn.ffw-it.de`)
- Zertifikate werden im Volume `traefik-certs_data` gespeichert
- Docker-Provider erkennt Container automatisch anhand von Labels
- Dynamische Konfiguration aus `./traefik/dynamic/` wird live neu geladen

**Ports**:

| Port | Protokoll | Zweck                    |
|------|-----------|--------------------------|
| 80   | HTTP      | Redirect → HTTPS         |
| 443  | HTTPS     | Alle Webdienste          |
| 8883 | TCP/TLS   | MQTT (privat, TLS)       |
| 8884 | TCP/TLS   | MQTT über WebSocket (TLS)|

**Konfiguration**:
- `./traefik/config.yml` – Hauptkonfiguration (Entrypoints, Provider, ACME)
- `./traefik/dynamic/ssl.yml` – Standard-TLS-Store mit Wildcard-Zertifikat
- `./traefik/dynamic/auth.yml` – `sso-auth` ForwardAuth-Middleware via Authentik
- `./traefik/dynamic/dashboard.yml` – Traefik-Dashboard-Route

---

### authentik_proxy

**Image**: `ghcr.io/goauthentik/proxy:2026.8.5`

Eigenständiger Authentik-Proxy-Outpost. Übernimmt SSO-Authentifizierung für alle mit `sso-auth@file` geschützten Dienste.

- Verbindet sich mit dem Authentik-Server unter `https://auth.ffw-it.de`
- Token kommt aus `authentik.env`
- Traefik leitet Authentifizierungsanfragen per ForwardAuth an Port 9000 weiter
- Gibt Benutzerinformationen (Username, Gruppen, E-Mail etc.) als HTTP-Header weiter

---

### watchtower

**Image**: `nickfedor/watchtower`

Überwacht laufende Container auf neue Image-Versionen und sendet Benachrichtigungen.

- Läuft im **Monitor-Only-Modus** (`WATCHTOWER_MONITOR_ONLY=true`) – aktualisiert nicht automatisch
- Benachrichtigungen via `WATCHTOWER_NOTIFICATION_URL` (Shoutrrr-Format)

---

### mqtt

**Image**: `eclipse-mosquitto`
**Listener**: 1883 (MQTT, auch direkt auf dem Host veröffentlicht), 8883 (MQTT, nur intern), 8884 (WebSocket, nur intern)
**TLS-Ports via Traefik**: 8883 → 8883 (MQTT), 8884 → 8884 (MQTT over WebSocket)

Eclipse Mosquitto MQTT-Broker. Zentraler Message-Bus für alle IoT-Geräte und Dienste.

- Authentifizierung erforderlich (`allow_anonymous false`)
- Passwortdatei unter `./mqtt/passwd` (nicht im Repository)
- Daten persistent in Volume `mosquitto-data`
- Konfiguration unter `./mqtt/mosquitto.conf`

**Passwortdatei anlegen**:
```bash
docker run --rm -it eclipse-mosquitto mosquitto_passwd -c /tmp/passwd <username>
# Datei nach ./mqtt/passwd kopieren
```

---

### hass

**Image**: `homeassistant/home-assistant:stable`
**URL**: `https://hass.iot.vpn.ffw-it.de`
**Konfiguration**: `./hass/`

Home Assistant – zentrale Automatisierungsplattform.

Die HTTP-/Proxy-Einstellungen speichert Home Assistant in seiner Datenbank, nicht in `configuration.yaml`.

**Konfiguration** (`hass/configuration.yaml`):
- Standort: FFW Wörnitzthal (Breitengrad 49.25, Längengrad 10.24, 460 m)
- Lokaler Login: `homeassistant`-Provider sowie `trusted_networks` für `10.2.1.34/32`
- Recorder: 31 Tage Verlauf, viele Sensor-Entitäten ausgeschlossen
- OpenID-Login via Authentik (`auth.ffw-it.de`), `block_login: true`
- Trusted IPs (kein Login-Block): `172.18.0.0/24`, `10.2.1.0/24`
- MQTT-Integration für Tore, Schalter und Sensoren
- `browser_mod`, `mobile_app`, `timer`, `media_source` aktiviert

---

### node-red

**Image**: `nodered/node-red`
**URL**: `https://flow.iot.vpn.ffw-it.de` (SSO-geschützt)

Visuelle Ablaufprogrammierung. Verbindet Home Assistant, MQTT und andere Dienste.

- Projekte-Feature aktiviert (`NODE_RED_ENABLE_PROJECTS=true`)
- Läuft als Benutzer `node-red`
- Daten persistent in Volume `node-red-data`
- Zugriff auf den Docker-Host via `docker-host:host-gateway`
- Startet nach `hass`

---

### zigbee2mqtt

**Image**: `koenkk/zigbee2mqtt`
**URL**: `https://z2m.iot.vpn.ffw-it.de` (SSO-geschützt)
**Konfiguration**: `./zigbee2mqtt/`

Erste Zigbee-Instanz. Verbindet Zigbee-Geräte mit dem MQTT-Broker.

| Einstellung     | Wert                    |
|-----------------|-------------------------|
| Koordinator     | `tcp://10.2.1.32:6638` (ZStack, 115200 Baud) |
| Kanal           | 25                      |
| MQTT-Topic      | `zigbee2mqtt`           |
| MQTT-Server     | `mqtt://mqtt`           |
| HA-Discovery    | aktiviert               |

---

### zigbee2mqtt_2

**Image**: `koenkk/zigbee2mqtt`
**URL**: `https://z2m-2.iot.vpn.ffw-it.de` (SSO-geschützt)
**Konfiguration**: `./zigbee2mqtt_2/`

Zweite Zigbee-Instanz für ein separates Zigbee-Netz.

| Einstellung     | Wert                    |
|-----------------|-------------------------|
| Koordinator     | `tcp://10.2.1.41:6638` (ZStack, 115200 Baud) |
| Kanal           | 24                      |
| MQTT-Topic      | `zigbee2mqtt_2`         |
| PAN ID          | `0x1a63`                |
| MQTT-Server     | `mqtt://mqtt`           |
| HA-Discovery    | aktiviert               |

---

### esphome

**Image**: `ghcr.io/esphome/esphome`
**URL**: `https://esphome.iot.vpn.ffw-it.de` (SSO-geschützt)
**Konfiguration**: `./esphome/`

ESPHome Dashboard zum Verwalten und Flashen von ESP8266/ESP32-Geräten (hauptsächlich Shelly).

- Ping-basierte Erreichbarkeitsüberwachung aktiviert (`ESPHOME_DASHBOARD_USE_PING=true`)
- Details zur ESPHome-Konfigurationsstruktur: [esphome/README.md](esphome/README.md)

---

### vlmcsd

**Image**: `ghcr.io/tobsa13/vlmcsd:master`
**Port**: 1688/TCP (direkt erreichbar)

KMS-Emulator zur Aktivierung von Microsoft-Produkten (Windows, Office) im lokalen Netz.

---

## Volumes

| Volume              | Verwendet von | Inhalt                        |
|---------------------|---------------|-------------------------------|
| `mosquitto-data`    | mqtt          | MQTT-Persistenz-Datenbank     |
| `node-red-data`     | node-red      | Flows, Projekte, Credentials  |
| `traefik-certs_data`| traefik       | Let's Encrypt Zertifikate     |

---

## Netzwerk

Alle Container laufen im Bridge-Netz `172.18.0.0/24` mit Gateway `172.18.0.1`.

**Interne Service-URLs** (Container-zu-Container):

| Service         | Interne URL                            |
|-----------------|----------------------------------------|
| mqtt            | `mqtt://mqtt:1883`                     |
| hass            | `http://hass:8123`                     |
| node-red        | `http://node-red:1880`                 |
| authentik_proxy | `http://authentik_proxy:9000`          |

---

## Service-Abhängigkeiten

```
mqtt
 └── hass
      └── node-red
```

---

## Update

Das Skript `./update` schreibt den aktuellen Stand (Versionen, Container, Images, Compose-Konfiguration) nach `backups/`, zieht neue Images und startet die Container nach Rückfrage neu.

---

## Sicherheitshinweise

- Alle Webdienste außer `hass` sind per Traefik-Middleware mit Authentik SSO geschützt; `hass` nutzt eigenes OpenID-Login über Authentik
- MQTT ist ohne TLS auf Port 1883 des Hosts erreichbar; TLS gibt es auf 8883/8884 über Traefik
- MQTT-Passwort und Zigbee-Netzwerkschlüssel liegen in Secrets-Dateien, nicht im Repository
- Hetzner API-Token nur für ACME DNS-Challenge benötigt
