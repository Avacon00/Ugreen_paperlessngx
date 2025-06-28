# 🚀 Anleitung: Paperless-NGX mit Docker auf einer UGREEN NAS 🗄️

Diese Anleitung beschreibt Schritt für Schritt, wie du **Paperless-NGX** mithilfe von **Docker** und **Portainer** auf deiner **UGREEN NAS** installierst. Auch auf anderen Systemen ist sie größtenteils übertragbar.

---

## 🗺️ Inhaltsverzeichnis

- [🛠️ Teil 1: Vorbereitung](#teil-1-vorbereitung)
  - [📂 Ordnerstruktur erstellen](#1-ordnerstruktur-erstellen)
  - [🆔 Eigene Benutzer-ID (UID & GID) ermitteln](#2-eigene-benutzer-id-uid--gid-ermitteln)
- [⚙️ Teil 2: Konfiguration & Installation](#teil-2-konfiguration--installation)
  - [📝 docker-compose.yml anpassen](#3-docker-composeyml-anpassen)
  - [🔐 Berechtigungen setzen](#4-berechtigungen-setzen)
  - [▶️ Stack in Portainer starten](#5-stack-in-portainer-starten)
- [⚠️ Teil 3: Wichtige Hinweise & Fehlerbehebung](#teil-3-wichtige-hinweise--fehlerbehebung)
  - [🗑️ Paperless-NGX komplett zurücksetzen](#paperless-ngx-komplett-zurücksetzen)

---

## 🛠️ Teil 1: Vorbereitung

### 📂 1. Ordnerstruktur erstellen

1. Öffne den Dateimanager (Dateien / Files) deiner UGREEN NAS
2. Navigiere zu `docker` (Freigegebener Ordner).
3. Erstelle darin den Ordner `paperlessngx`.
4. Lege darin folgende Unterordner an:

```
/docker/paperlessngx/
├── consume
├── data
├── db
├── export
├── media
├── redis
└── trash
```

---

### 🆔 2. Eigene Benutzer-ID (UID & GID) ermitteln
(unter Systemsteuerung, Terminal, SSH Aktivieren und übernehmen drücken, nicht vergessen danach wieder zu Deaktivieren)

Öffne eine SSH-Verbindung zur NAS und führe folgenden Befehl aus:

```bash
id
```

Die Ausgabe zeigt `uid` und `gid` deines Benutzers:

```bash
uid=1026(dein_benutzer) gid=100(users) groups=100(users),101(administrators)
```

📌 Notiere dir UID und GID – du brauchst sie später in der Konfiguration.

---

## ⚙️ Teil 2: Konfiguration & Installation

### 📝 3. docker-compose.yml anpassen

Kopiere die folgende Konfiguration und passe alle `USERMAP_UID`, `USERMAP_GID` und IP-Adressen an deine Umgebung an:

```yaml
version: "3.9"

services:
  redis:
    image: redis:7
    container_name: PaperlessNGX-REDIS
    hostname: paper-redis
    command: redis-server --requirepass redispass
    volumes:
      - /volume1/docker/paperlessngx/redis:/data:rw
    user: 1026:100  # UID:GID anpassen
    restart: on-failure:5

  db:
    image: postgres:16
    container_name: PaperlessNGX-DB
    hostname: paper-db
    volumes:
      - /volume1/docker/paperlessngx/db:/var/lib/postgresql/data:rw
    environment:
      POSTGRES_DB: paperless
      POSTGRES_USER: paperlessuser
      POSTGRES_PASSWORD: paperlesspass
    restart: on-failure:5

  gotenberg:
    image: gotenberg/gotenberg:latest
    container_name: PaperlessNGX-GOTENBERG
    restart: on-failure:5

  tika:
    image: ghcr.io/paperless-ngx/tika:latest
    container_name: PaperlessNGX-TIKA
    restart: on-failure:5

  paperless:
    image: ghcr.io/paperless-ngx/paperless-ngx:latest
    container_name: PaperlessNGX
    hostname: paperless-ngx
    ports:
      - 8777:8000
    volumes:
      - /volume1/docker/paperlessngx/data:/usr/src/paperless/data:rw
      - /volume1/docker/paperlessngx/media:/usr/src/paperless/media:rw
      - /volume1/docker/paperlessngx/export:/usr/src/paperless/export:rw
      - /volume1/docker/paperlessngx/consume:/usr/src/paperless/consume:rw
      - /volume1/docker/paperlessngx/trash:/usr/src/paperless/trash:rw
    environment:
      USERMAP_UID: 1026
      USERMAP_GID: 100
      PAPERLESS_URL: http://192.168.0.100:8777
      PAPERLESS_CSRF_TRUSTED_ORIGINS: http://192.168.0.100:8777
      PAPERLESS_ADMIN_USER: demo
      PAPERLESS_ADMIN_PASSWORD: demo
      PAPERLESS_TIME_ZONE: Europe/Berlin
      PAPERLESS_OCR_LANGUAGE: deu+eng
      PAPERLESS_DBENGINE: postgresql
      PAPERLESS_REDIS: redis://:redispass@paper-redis:6379
      PAPERLESS_DBHOST: paper-db
      PAPERLESS_DBNAME: paperless
      PAPERLESS_DBUSER: paperlessuser
      PAPERLESS_DBPASS: paperlesspass
      PAPERLESS_TIKA_ENABLED: 1
      PAPERLESS_TIKA_GOTENBERG_ENDPOINT: http://gotenberg:3000
      PAPERLESS_TIKA_ENDPOINT: http://tika:9998
    restart: on-failure:5
    depends_on:
      - db
      - redis
      - tika
      - gotenberg
```

---

### 🔐 4. Berechtigungen setzen

Führe auf deiner NAS folgende Befehle aus, um die Besitzrechte korrekt zu setzen:
-Beachte  wieder deine UID & GID, diese musst du noch anpassen-

```bash
sudo chown -R 1026:100 /volume1/docker/paperlessngx/consume
sudo chown -R 1026:100 /volume1/docker/paperlessngx/data
sudo chown -R 1026:100 /volume1/docker/paperlessngx/export
sudo chown -R 1026:100 /volume1/docker/paperlessngx/media
sudo chown -R 1026:100 /volume1/docker/paperlessngx/trash
sudo chown -R 1026:100 /volume1/docker/paperlessngx/redis
```

---

### ▶️ 5. Stack in Portainer starten

1. Öffne [Portainer](http://deine-nas-ip:9000).
2. Navigiere zu **Stacks** > **+ Add Stack**.
3. Gib einen Namen ein, z. B. `paperless`.
4. Füge den Inhalt deiner `docker-compose.yml` in das Editorfeld ein.
5. Klicke auf **Deploy the stack**.

🔁 Der erste Start kann ein paar Minuten dauern. Danach erreichst du Paperless-NGX unter:

```
http://DEINE-NAS-IP:8777
```

---

## ⚠️ Teil 3: Wichtige Hinweise & Fehlerbehebung

### 🗑️ Paperless-NGX komplett zurücksetzen

Willst du die Installation **neu aufsetzen**, genügt es nicht, nur den Stack zu löschen — du musst auch das Volume mit der Datenbank entfernen:

```bash
sudo rm -rf /volume1/docker/paperlessngx/db/
```

Verifiziere, dass der Ordner leer ist:

```bash
sudo ls -l /volume1/docker/paperlessngx/db/
```

📌 Jetzt kannst du den Stack erneut über Portainer starten. Die Datenbank wird mit den Werten aus der `docker-compose.yml` neu erstellt.

---

## ✅ Fertig!

Dein Paperless-NGX ist jetzt bereit zur Nutzung! Weitere Infos findest du im [offiziellen Paperless-NGX-Repository](https://github.com/paperless-ngx/paperless-ngx).
