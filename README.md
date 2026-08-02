# Esp32-Pozzo (ESPHome, Waveshare ESP32-S3-ETH + JSN-SR04T)

Firmware ESPHome per misura del livello acqua (pozzo/cisterna) tramite
sensore ultrasonico JSN-SR04T (Mode 0, trigger/echo classico compatibile
HC-SR04), su board Waveshare ESP32-S3-ETH con connessione Ethernet PoE
(chip W5500 via SPI).

## Hardware

- Waveshare ESP32-S3-ETH (PoE)
- JSN-SR04T (modulo con pad di selezione modalità "R19"; su questo
  esemplare R19 è aperto/senza resistenza, che corrisponde a Mode 0)

> **Nota per chi arriva da versioni precedenti di questo progetto:**
> lungo lo sviluppo sono stati provati altri approcci (SR04M-2 in
> modalità UART con parser custom, poi la piattaforma ufficiale
> ESPHome `jsn_sr04t` per Mode 1/2) prima di arrivare a questa
> configurazione, che è quella effettivamente funzionante e testata.
> Il modulo usato ha pin etichettati sia "RX/Trig" che "TX/Echo" (la
> stessa coppia fisica di pin serve funzioni diverse a seconda della
> modalità impostata via il pad R19): qui sono usati come Trig/Echo
> classico, non come UART.

## Collegamenti tra le schede

### JSN-SR04T → Waveshare ESP32-S3-ETH

| JSN-SR04T | ESP32-S3-ETH | Note |
|---|---|---|
| 5V | 5V (VBUS) | Il modulo richiede 5V |
| GND | GND | Riferimento comune obbligatorio |
| RX / Trig | GPIO17 | Uscita diretta dall'ESP32, logic 3.3V, compatibile col modulo — invia l'impulso di trigger |
| TX / Echo | GPIO18 | **Tramite partitore resistivo** (vedi sotto) — riceve l'impulso di ritorno |

> **Nota:** GPIO5 (spesso usato in guide generiche per TRIG) su questa
> board Waveshare è riservato internamente all'interfaccia SD card
> (funzione `SD_MISO`) e non è esposto sul connettore a pettine:
> non utilizzabile per il sensore.

**Partitore resistivo su TX/Echo** (solo su questa linea dati; l'alimentazione
resta a 5V come da tabella sopra). Il pin Echo del modulo emette a 5V
logic, ma i GPIO dell'ESP32-S3 tollerano al massimo circa 3.3-3.6V,
quindi serve un partitore per abbassare solo il segnale di ritorno:

```
Echo (5V) ──[R1 = 1kΩ]──┬── GPIO18 (ESP32, ~3.3V)
                         │
                       [R2 = 2kΩ]
                         │
                        GND
```

Con R1=1kΩ e R2=2kΩ: Vout = 5V × (2k / (1k+2k)) ≈ 3.33V.

> **Nota:** nel mio caso specifico ho usato 2 resistenze da 1kΩ in serie.

### Alimentazione della board

| Sorgente | Collegamento |
|---|---|
| PoE (consigliato) | Cavo Ethernet dallo switch/injector PoE al connettore RJ45 della board |
| USB-C | Solo per il primo flash o alimentazione di banco, non serve se usi PoE |

### Ethernet (interno alla board, nessun cablaggio manuale)

Il chip **W5500** è collegato via SPI internamente alla board (pin già
cablati sul PCB, non richiedono collegamenti esterni). Riportati qui solo
per riferimento/debug, corrispondono a quanto dichiarato nel firmware:

| Segnale | GPIO |
|---|---|
| MOSI | GPIO11 |
| MISO | GPIO12 |
| CLK | GPIO13 |
| CS | GPIO14 |
| RESET | GPIO9 |
| INTERRUPT | GPIO10 |

## Struttura repo

- `Esp32-Pozzo.yaml` — configurazione ESPHome principale
- `secrets.yaml.example` — template da copiare in `secrets.yaml` (mai committato)
- `.github/workflows/build.yml` — validazione/compilazione su ogni push e PR
- `.github/workflows/release.yml` — build + pubblicazione firmware su GitHub Release al push di un tag `vX.Y.Z`

## Setup locale (prima configurazione / flash via USB)

Ho utilizzato EspHome Builder in Home Assistant.

### Generare la encryption key per l'API

Vedi progetto SabianaVmcToHomeAssistant per capire come generare la encryption key.

## Secrets richiesti su GitHub (per il workflow di release)

Impostali in **Settings → Secrets and variables → Actions** del repository:

| Secret | Descrizione |
|---|---|
| `API_ENCRYPTION_KEY` | Stessa chiave usata in HA per l'integrazione ESPHome |
| `OTA_PASSWORD` | Password per gli aggiornamenti OTA nativi |
| `WEB_SERVER_USERNAME` | Utente per l'accesso al web server locale del device |
| `WEB_SERVER_PASSWORD` | Password per l'accesso al web server locale del device |

Questi secrets vengono usati **solo** dal workflow `release.yml` per generare
un `secrets.yaml` temporaneo durante la build in CI; non vengono mai
committati nel repo.

## Pubblicare una release

```bash
git tag v1.0.0
git push origin v1.0.0
```

Il workflow compila il firmware e pubblica su GitHub Release il file
`.bin` e il `manifest.json`, associati al tag.

## Calibrazione a runtime

Non serve riflashare per calibrare l'installazione. Da Home Assistant
(o dal web server del device) sono esposti due parametri configurabili
e persistiti in flash:

- **Empty Distance** (cm): distanza dal sensore al fondo quando vuoto
- **Distance Offset** (cm): correzione fine della lettura grezza

Il livello acqua e la percentuale vengono ricalcolati automaticamente da
questi due valori.

## Web server locale

Disponibile su `http://<ip-device>/` (protetto da username/password),
utile per debug e per modificare i parametri sopra senza passare da HA.
Non espone l'upload firmware via browser: l'OTA è disponibile solo tramite
la piattaforma nativa (API, con password).