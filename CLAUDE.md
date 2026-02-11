# ESPHome Configuration Repository

## Projektbeschreibung

Dieses Repository enthält ESPHome-Konfigurationsdateien für verschiedene Smart-Home-Geräte. Die Konfigurationen nutzen ein Package-System mit einer gemeinsamen Basis-Konfiguration.

## Dateistruktur

```
.
├── common.yaml                          # Gemeinsame Basis-Konfiguration (WiFi, API, OTA, MQTT, etc.)
├── secrets.yaml.example                 # Vorlage für Secrets (wird ins Repository committed)
├── secrets.yaml                         # Echte Secrets (wird NICHT committed, siehe .gitignore)
├── power-meter.yaml                     # Stromzähler-Konfiguration (SML)
├── gas-meter.yaml                       # Gaszähler-Konfiguration (Reed-Kontakt)
├── smartsolar.yaml                      # Victron SmartSolar MPPT Laderegler
├── smartshunt.yaml                      # Victron SmartShunt Batteriemonitor
├── esp32-bluetooth-proxy-garage.yaml    # Bluetooth Proxy (Garage)
├── esp32-bluetooth-proxy-buero.yaml     # Bluetooth Proxy (Büro)
├── ac-bedroom.yaml                      # Mitsubishi Klimaanlage Schlafzimmer (CN105)
└── .gitignore                          # Schützt secrets.yaml vor versehentlichem Commit
```

## Sicherheitskonzept

### Secrets Management

**KRITISCH:** Keine Passwörter oder sensible Daten dürfen in YAML-Dateien committed werden!

- Alle sensiblen Werte werden über `!secret` Platzhalter referenziert
- `secrets.yaml` ist in `.gitignore` und wird NIEMALS committed
- `secrets.yaml.example` dient als Vorlage mit Platzhaltern

### Benötigte Secrets

In `secrets.yaml` müssen folgende Werte definiert sein:

- `wifi_ssid` - WLAN-Name
- `wifi_password` - WLAN-Passwort
- `ap_password` - Fallback-AP Passwort
- `encryption_key` - API-Verschlüsselung (Base64, 32 Bytes)
- `ota_password` - OTA-Update Passwort
- `mqtt_broker` - MQTT-Broker IP/Hostname

## Workflow

### Neue Gerätekonfiguration hinzufügen

1. Neue YAML-Datei erstellen (z.B. `neues-geraet.yaml`)
2. Common-Package einbinden:
   ```yaml
   packages:
     common:
       url: https://github.com/legacycode-esphome/ESPHome-Config.git
       file: common.yaml
       ref: main
       refresh: 0s  # Immer aktuellste Version beim Kompilieren laden
   ```
3. Konfiguration validieren: `esphome config neues-geraet.yaml`
4. Bei erfolgreichem Test committen

**Hinweis:** `refresh: 0s` lädt die Remote-Packages bei jedem Kompilieren neu. Die ESP-Geräte aktualisieren sich **nicht automatisch** - du musst nach Änderungen manuell neu kompilieren und flashen.

**WICHTIG:** Änderungen an `common.yaml` müssen zuerst committed und gepusht werden, bevor sie beim Kompilieren anderer Geräte wirksam werden, da diese `common.yaml` als Remote-Package von GitHub laden.

### Vor jedem Commit

**PFLICHT:** Konfiguration validieren:
```bash
esphome config <datei>.yaml
```

Die Konfiguration MUSS erfolgreich validiert werden, bevor sie committed wird.

### Git Workflow

```bash
# Änderungen validieren
esphome config power-meter.yaml

# Bei Erfolg committen
git add <dateien>
git commit -m "Beschreibung"
git push
```

**WICHTIG:** Keine Co-Authorship von Claude in Commits.

**Git Remote:** Nutzt SSH-Host `github-code` statt `github.com` (konfiguriert in SSH-Config).

## Namenskonventionen

### Sensor-Namen

Sensor-Namen werden **ohne** `${friendly_name}` Prefix definiert. ESPHome stellt den Gerätenamen (`friendly_name`) automatisch voran.

```yaml
# RICHTIG
name: "Leistungsaufnahme"        # → "Klima Schlafzimmer Leistungsaufnahme" in HA

# FALSCH (führt zu Dopplung)
name: "${friendly_name} Leistungsaufnahme"  # → "Klima Schlafzimmer Klima Schlafzimmer Leistungsaufnahme"
```

### MQTT Topics

Geräte mit Leerzeichen im `friendly_name` müssen das MQTT-Topic explizit überschreiben:

```yaml
mqtt:
  topic_prefix: esphome/${devicename}  # statt esphome/${friendly_name}
```

### Fallback-AP SSID

Die SSID `"${friendly_name} Fallback"` darf maximal 32 Zeichen haben. Bei langen Namen den `friendly_name` kürzen.

## Common Configuration

Die `common.yaml` enthält:

- ESPHome Core-Konfiguration
- Logger (Level: INFO)
- WiFi mit Fallback-AP
- API mit Verschlüsselung
- Zeit-Synchronisation (Home Assistant)
- OTA-Updates mit Passwort
- MQTT-Integration
- Web-Server (Port 80)
- Captive Portal (Fallback)
- Restart-Button (diagnostic)

### Diagnostic-Sensoren (automatisch für alle Geräte)

**Sensoren:**
- WiFi Signal Stärke (dBm, update_interval: 60s)
- Betriebszeit (Sekunden, update_interval: 60s)

**Text-Sensoren:**
- WiFi SSID, BSSID, IP Adresse, MAC Adresse
- ESPHome Version
- Neustart Grund (reset_reason)

**Binary-Sensoren:**
- Status (Verbindungsstatus)

**Wichtig:** Nach Hinzufügen neuer Sensoren zur `common.yaml` müssen Geräte in Home Assistant möglicherweise gelöscht und neu hinzugefügt werden, damit die Discovery korrekt funktioniert.

## Geräte

### power-meter.yaml

SML-Stromzähler Ausleser auf Wemos D1 Mini:

- **Board:** ESP8266 (d1_mini)
- **UART:** GPIO1 (TX), GPIO3 (RX), 9600 Baud
- **Sensoren:**
  - Bezug gesamt (OBIS 1-0:1.8.0)
  - Einspeisung gesamt (OBIS 1-0:2.8.0)
  - Aktuelle Wirkleistung (OBIS 1-0:16.7.0)
  - Stromzähler-ID (OBIS 1-0:96.1.0)

### gas-meter.yaml

Gaszähler-Ausleser mit Reed-Kontakt auf Wemos D1 Mini:

- **Board:** ESP8266 (d1_mini)
- **Reed-Kontakt:** GPIO4 (mit Pullup)
- **Remote Packages:** github://legacycode-esphome/ESPHome-Gas-Meter
- **Konfiguration:**
  - `pulses_per_cubic_meter: "100"` - Impulse pro m³
  - `initial_meter_offset: "0"` - Initialer Zählerstand-Offset
- **Sensoren:**
  - Durchflussrate (m³/h)
  - Gesamt (m³)
  - Gesamtimpulse
  - Zählerstand mit Offset (m³)
- **Funktionen:**
  - LED-Blinken bei Impuls (3s)
  - Impulszähler zurücksetzen (Button)
  - Zählerstand-Offset konfigurierbar (Number)
  - Persistente Speicherung der Impulse

### smartsolar.yaml

Victron SmartSolar MPPT Laderegler auf M5Stack AtomS3 Lite:

- **Board:** ESP32-S3 (m5stack-atoms3), Framework: arduino
- **UART:** RX=GPIO1, 19200 Baud (nur RX, VE.Direct read-only)
- **Remote On/Off:** GPIO2 steuert Victron RX-Pin
- **Externe Komponente:** github://KinDR007/VictronMPPT-ESPHOME@main
- **Sensoren:** Panel Spannung/Leistung, Batterie Spannung/Strom, Ertrag (Gesamt/Heute/Gestern), Max Leistung, Lademodus, Fehlercode
- **Switch:** Remote On/Off

### smartshunt.yaml

Victron SmartShunt Batteriemonitor auf M5Stack AtomS3 Lite:

- **Board:** ESP32-S3 (m5stack-atoms3), Framework: arduino
- **UART:** RX=GPIO1, 19200 Baud (nur RX, VE.Direct read-only)
- **Externe Komponente:** github://KinDR007/VictronMPPT-ESPHOME@main
- **Sensoren:** Batterie Spannung/Strom/Ladezustand, Momentanleistung, Energie geladen/entladen (kWh), Min/Max Spannungen
- **Energy Dashboard:** `amount_of_charged_energy` / `amount_of_discharged_energy`

### esp32-bluetooth-proxy-garage.yaml

ESP32-S3 Bluetooth Proxy (Garage):

- **Board:** ESP32-S3 (m5stack-atoms3), Framework: arduino
- **Packages:** github://esphome/bluetooth-proxies + Common
- **Device Name:** bt-proxy-garage
- **MQTT Topic:** esphome/bt-proxy-garage

### esp32-bluetooth-proxy-buero.yaml

ESP32-S3 Bluetooth Proxy (Büro):

- **Board:** ESP32-S3 (m5stack-atoms3), Framework: arduino
- **Packages:** github://esphome/bluetooth-proxies + Common
- **Device Name:** bt-proxy-buero
- **MQTT Topic:** esphome/bt-proxy-buero

### ac-bedroom.yaml

Mitsubishi MSZ-HR35VF Klimaanlage Schlafzimmer auf M5Stack AtomS3 Lite:

- **Board:** ESP32-S3 (m5stack-atoms3), Framework: esp-idf
- **UART:** TX=GPIO1, RX=GPIO2, 2400 Baud (CN105-Protokoll)
- **Externe Komponente:** github://echavet/MitsubishiCN105ESPHome
- **Device Name:** ac-bedroom
- **Friendly Name:** Klima Schlafzimmer
- **MQTT Topic:** esphome/ac-bedroom
- **Climate-Entity:**
  - Name: "Betriebsart"
  - Modi: AUTO, COOL, HEAT, DRY, FAN_ONLY
  - Lüfter: AUTO, QUIET, LOW, MEDIUM, HIGH
  - Swing: OFF, VERTICAL
  - Temperaturbereich: 16-31°C (Schrittweite 1°C)
  - Update-Intervall: 2s, Debounce: 100ms
- **Sensoren:**
  - Leistungsaufnahme (W, power, measurement)
  - Kompressor Frequenz (Hz, frequency, measurement)
  - Energieverbrauch (kWh, energy, total_increasing)
  - Betriebsstunden (h, diagnostic)
  - Verbindungszeit (s, diagnostic)
- **Text-Sensoren:**
  - Betriebsstufe (diagnostic)
  - Sub-Modus (diagnostic)
  - Auto Sub-Modus (diagnostic)
- **Steuerung:**
  - Lamellen Vertikal (select)
- **Hinweise:**
  - CN105 ist der interne Stecker der Mitsubishi-Inneneinheit (bidirektional, kein IR)
  - Horizontale Lamellen nicht vorhanden bei MSZ-HR35VF
  - Außentemperatur-Sensor liefert ungenaue Werte (40°C konstant), daher nicht eingebunden

## Hinweise für Claude

- Vor Commits IMMER `esphome config` ausführen
- **CLAUDE.md vor jedem Push aktualisieren!**
- Keine Secrets in YAML-Dateien hardcoden
- Für Validierung Test-Secrets verwenden (secrets.yaml wird nicht committed)
- Keine Co-Authorship in Git-Commits
- Bestehende Konventionen in common.yaml beachten
- Sensor-Namen ohne `${friendly_name}` Prefix (ESPHome stellt Gerätenamen automatisch voran)
- Bei Geräten mit Leerzeichen im friendly_name: MQTT-Topic auf `${devicename}` überschreiben
- Fallback-AP SSID maximal 32 Zeichen (`${friendly_name} Fallback`)
- `common.yaml` Änderungen erst pushen, bevor andere Geräte neu kompiliert werden
- Git Remote nutzt SSH-Host `github-code` (nicht `github.com`)
