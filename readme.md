# Watchtower mit Docker Compose

Diese Konfiguration ersetzt den alten `docker run`-Befehl durch eine `docker-compose.yml` und dokumentiert die wichtigsten Einstellungen für VMs bzw. Hosts, auf denen Container automatisch durch Watchtower aktualisiert werden sollen.

## Ziel

Watchtower überwacht laufende Container und aktualisiert sie automatisch, sobald ein neues Image verfügbar ist.

Wichtig: Watchtower aktualisiert **Container**, nicht die VM selbst.

Wenn mehrere VMs im Einsatz sind, muss Watchtower auf jeder VM laufen, deren Container automatisch aktualisiert werden sollen.

## Empfohlene `docker-compose.yml` für Watchtower

```yaml
services:
  watchtower:
    image: containrrr/watchtower:latest
    container_name: watchtower
    restart: always
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      TZ: Europe/Berlin
      WATCHTOWER_LABEL_ENABLE: "true"
      WATCHTOWER_CLEANUP: "true"
      WATCHTOWER_INCLUDE_STOPPED: "false"
      WATCHTOWER_REVIVE_STOPPED: "false"
      WATCHTOWER_POLL_INTERVAL: "300"
    command: --label-enable --cleanup
```

## Start

```bash
docker compose up -d
```

## Grundprinzip für Updates

Sobald `WATCHTOWER_LABEL_ENABLE=true` bzw. `--label-enable` aktiv ist, aktualisiert Watchtower **nur Container mit passendem Label**.

Das ist die sicherste Variante für VMs mit mehreren Containern.

## Container für automatische Updates markieren

Der jeweilige Container muss dieses Label bekommen:

```yaml
labels:
  - com.centurylinklabs.watchtower.enable=true
```

Beispiel für einen Service, der automatisch aktualisiert werden soll:

```yaml
services:
  app:
    image: nginx:latest
    restart: always
    labels:
      - com.centurylinklabs.watchtower.enable=true
```

## Container explizit nicht aktualisieren

Wenn Label-Only aktiviert ist, reicht es schon, **kein Label** zu setzen.

Optional kann man es auch explizit so dokumentieren:

```yaml
labels:
  - com.centurylinklabs.watchtower.enable=false
```

## Empfehlung für VMs

### Variante A: sichere Standard-Variante

Diese Variante ist empfohlen.

* Auf jeder VM läuft ein eigener Watchtower-Container
* Watchtower ist mit `WATCHTOWER_LABEL_ENABLE=true` konfiguriert
* Nur Container mit `com.centurylinklabs.watchtower.enable=true` werden aktualisiert

Vorteil:

* volle Kontrolle
* keine versehentlichen Updates
* gut für Produktivsysteme

### Variante B: alles auf einer VM automatisch aktualisieren

Wenn auf einer einzelnen VM wirklich alle Container aktualisiert werden sollen, kann man Label-Only deaktivieren.

Dann diese Einstellung **nicht** setzen:

```yaml
WATCHTOWER_LABEL_ENABLE: "true"
```

und den Command ohne `--label-enable` betreiben.

Das ist aber nur sinnvoll, wenn wirklich alle Container auf dieser VM aktualisiert werden dürfen.

## Wichtige Environment-Variablen

### `TZ`

```yaml
TZ: Europe/Berlin
```

Sorgt für korrekte Zeitangaben in Logs.

### `WATCHTOWER_LABEL_ENABLE`

```yaml
WATCHTOWER_LABEL_ENABLE: "true"
```

Nur Container mit passendem Label werden aktualisiert.

### `WATCHTOWER_CLEANUP`

```yaml
WATCHTOWER_CLEANUP: "true"
```

Alte, nicht mehr benötigte Images werden nach erfolgreichem Update entfernt.

### `WATCHTOWER_POLL_INTERVAL`

```yaml
WATCHTOWER_POLL_INTERVAL: "300"
```

Prüft alle 300 Sekunden auf neue Images.

Typische Werte:

* `300` = alle 5 Minuten
* `900` = alle 15 Minuten
* `3600` = stündlich

Für Produktivumgebungen ist oft `900` oder `3600` sinnvoll.

## Optional: geplanter Lauf statt Polling

Alternativ kann Watchtower auch nur zu festen Zeiten laufen.

Beispiel: täglich um 04:00 Uhr.

```yaml
services:
  watchtower:
    image: containrrr/watchtower:latest
    container_name: watchtower
    restart: always
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      TZ: Europe/Berlin
      WATCHTOWER_LABEL_ENABLE: "true"
      WATCHTOWER_CLEANUP: "true"
      WATCHTOWER_SCHEDULE: "0 0 4 * * *"
    command: --label-enable --cleanup
```

Hinweis: Entweder `WATCHTOWER_POLL_INTERVAL` oder `WATCHTOWER_SCHEDULE` verwenden, nicht beides gleichzeitig.

## Beispiel: App-VM mit automatischem Update

```yaml
services:
  app:
    image: ghcr.io/example/meine-app:latest
    container_name: meine-app
    restart: always
    environment:
      APP_ENV: production
    labels:
      - com.centurylinklabs.watchtower.enable=true
```

## Beispiel: Datenbank auf derselben VM ohne automatisches Update

```yaml
services:
  db:
    image: postgres:16
    container_name: postgres
    restart: always
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: geheim
```

Ohne Watchtower-Label bleibt der Container von automatischen Updates ausgeschlossen.

## Praxisregel für mehrere VMs

Auf **jeder VM**:

1. Watchtower lokal starten
2. `WATCHTOWER_LABEL_ENABLE=true` aktivieren
3. Nur die gewünschten Container mit `com.centurylinklabs.watchtower.enable=true` markieren

Damit ist klar geregelt:

* welche VM Watchtower ausführt
* welche Container auf dieser VM aktualisiert werden
* welche Container bewusst ausgeschlossen bleiben

## Empfohlener Standard

Für die meisten Setups ist das die beste Kombination:

* Watchtower je VM als eigener Container
* Label-basierte Freigabe
* Cleanup aktiv
* Poll-Intervall eher konservativ, z. B. 900 oder 3600 Sekunden
* Keine automatischen Updates für Datenbanken, Stateful Services oder kritische Einzelcontainer ohne vorherigen Test

## Minimal-Merksatz

* Watchtower aktualisiert Container, nicht VMs
* Pro VM ein eigener Watchtower
* Auto-Update nur für Container mit:

```yaml
com.centurylinklabs.watchtower.enable=true
```
