# ChirpStack Docker example

This repository contains a skeleton to setup the [ChirpStack](https://www.chirpstack.io)
open-source LoRaWAN Network Server (v4) using [Docker Compose](https://docs.docker.com/compose/).

**Note:** Please use this `docker-compose.yml` file as a starting point for testing
but keep in mind that for production usage it might need modifications. 

## Directory layout

* `docker-compose.yml`: the docker-compose file containing the services
* `configuration/chirpstack`: directory containing the ChirpStack configuration files
* `configuration/chirpstack-gateway-bridge`: directory containing the ChirpStack Gateway Bridge configuration
* `configuration/mosquitto`: directory containing the Mosquitto (MQTT broker) configuration
* `configuration/postgresql/initdb/`: directory containing PostgreSQL initialization scripts

## Configuration

This setup is pre-configured for all regions. You can either connect a ChirpStack Gateway Bridge
instance (v3.14.0+) to the MQTT broker (port 1883) or connect a Semtech UDP Packet Forwarder.
Please note that:

* You must prefix the MQTT topic with the region.
  Please see the region configuration files in the `configuration/chirpstack` for a list
  of topic prefixes (e.g. eu868, us915_0, au915_0, as923_2, ...).
* The protobuf marshaler is configured.

This setup also comes with two instances of the ChirpStack Gateway Bridge. One
is configured to handle the Semtech UDP Packet Forwarder data (port 1700), the
other is configured to handle the Basics Station protocol (port 3001). Both
instances are by default configured for EU868 (using the `eu868` MQTT topic
prefix).

### Reconfigure regions

ChirpStack has at least one configuration of each region enabled. You will find
the list of `enabled_regions` in `configuration/chirpstack/chirpstack.toml`.
Each entry in `enabled_regions` refers to the `id` that can be found in the
`region_XXX.toml` file. This `region_XXX.toml` also contains a `topic_prefix`
configuration which you need to configure the ChirpStack Gateway Bridge
UDP instance (see below).

#### ChirpStack Gateway Bridge (UDP)

Within the `docker-compose.yml` file, you must replace the `eu868` prefix in the
`INTEGRATION__..._TOPIC_TEMPLATE` configuration with the MQTT `topic_prefix` of
the region you would like to use (e.g. `us915_0`, `au915_0`, `in865`, ...).

#### ChirpStack Gateway Bridge (Basics Station)

Within the `docker-compose.yml` file, you must update the configuration file
that the ChirpStack Gateway Bridge instance must used. The default is
`chirpstack-gateway-bridge-basicstation-eu868.toml`. For available
configuration files, please see the `configuration/chirpstack-gateway-bridge`
directory.

# Data persistence

PostgreSQL and Redis data is persisted in Docker volumes, see the `docker-compose.yml`
`volumes` definition.

## Requirements

Before using this `docker-compose.yml` file, make sure you have [Docker](https://www.docker.com/community-edition)
installed.

## Importing device repository

To import the [lorawan-devices](https://github.com/TheThingsNetwork/lorawan-devices)
repository (optional step), run the following command:

```bash
make import-lorawan-devices
```

This will clone the `lorawan-devices` repository and execute the import command of ChirpStack.
Please note that for this step you need to have the `make` command installed.

**Note:** an older snapshot of the `lorawan-devices` repository is cloned as the
latest revision no longer contains a `LICENSE` file.

## Usage

To start the ChirpStack simply run:

```bash
$ docker-compose up
```

After all the components have been initialized and started, you should be able
to open http://localhost:8080/ in your browser.

## Quick Start
```bash
git clone https://github.com/Dxrin9/chirpstack-docker.git
cd chirpstack-docker
docker-compose up -d
```

## ChirpStack

Web UI: http://localhost:8080

Username: admin

Password: admin

## Gateway Configuration
SenseCap M2 (Basic Station)
URI: ws://YOUR_COMPUTER_IP:3001

Authentication: No Authenticatio

## Senzor 
AppKey: 9dfa5ef7f5d93323fa2f6994de1e2fa9
EUI: 0000ccbde5e22748

## Device Profiles
JavaScript Functions
```
/**
 * Decode uplink function
 * 
 * @param {object} input
 * @param {number[]} input.bytes Byte array containing the uplink payload, e.g. [255, 230, 255, 0]
 * @param {number} input.fPort Uplink fPort.
 * @param {Record<string, string>} input.variables Object containing the configured device variables.
 * 
 * @returns {{data: object, errors: string[], warnings: string[]}}
 * An object containing:
 * - data: Object representing the decoded payload.
 * - errors: An array of errors (optional).
 * - warnings: An array of warnings (optional).
 */
function decodeUplink(input) {
  var bytes = input.bytes;
  
  return {
    data: {
      temperature: bytes[0],     // °C
      humidity: bytes[1],        // %
      battery: bytes[2],         // %
      counter: bytes[3]          // contor
    }
  };
}

function encodeDownlink(input) {
  return {
    bytes: [],
    fPort: 2
  };
}

/**
 * Encode downlink function.
 * 
 * @param {object} input
 * @param {object} input.data Object representing the payload that must be encoded.
 * @param {Record<string, string>} input.variables Object containing the configured device variables.
 * 
 * @returns {{bytes: number[], fPort: number, errors: string[], warnings: string[]}}
 * An object containing:
 * - bytes: Byte array containing the downlink payload.
 * - fPort: The downlink LoRaWAN fPort.
 * - errors: An array of errors (optional).
 * - warnings: An array of warnings (optional).
 */
function encodeDownlink(input) {
  return {
    fPort: 10,
    bytes: [225, 230, 255, 0],
  };
}

```

## Arduino
Code for Wireless Stick Senzor
```
#include "LoRaWan_APP.h"
#include <Wire.h>
#include <DHT.h>

/* OTAA para - DEVICE-UL DHT11 */
uint8_t devEui[] = { 0x00, 0x00, 0xCC, 0xBD, 0xE5, 0xE2, 0x27, 0x48 };
uint8_t appEui[] = { 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00 };
uint8_t appKey[] = { 0x9D, 0xFA, 0x5E, 0xF7, 0xF5, 0xD9, 0x33, 0x23, 0xFA, 0x2F, 0x69, 0x94, 0xDE, 0x1E, 0x2F, 0xA9 };

/* CONFIGURARE DHT11 */
#define DHTPIN 20          // GPIO 13 pentru DHT11
#define DHTTYPE DHT11      // Tip senzor
DHT dht(DHTPIN, DHTTYPE);

// Variabile pentru date
float temperature, humidity;
static uint8_t counter = 0;
bool dhtError = false;

/* ABP para (ignoră - folosim OTAA) */
uint8_t nwkSKey[] = { 0x15, 0xb1, 0xd0, 0xef, 0xa4, 0x63, 0xdf, 0xbe, 0x3d, 0x11, 0x18, 0x1e, 0x1e, 0xc7, 0xda,0x85 };
uint8_t appSKey[] = { 0xd7, 0x2c, 0x78, 0x75, 0x8c, 0xdc, 0xca, 0xbf, 0x55, 0xee, 0x4a, 0x77, 0x8d, 0x16, 0xef,0x67 };
uint32_t devAddr =  ( uint32_t )0x007e6ae1;

/* LoraWan settings */
uint16_t userChannelsMask[6] = { 0x00FF,0x0000,0x0000,0x0000,0x0000,0x0000 };
LoRaMacRegion_t loraWanRegion = LORAMAC_REGION_EU868;
DeviceClass_t  loraWanClass = CLASS_A;
uint32_t appTxDutyCycle = 60000; // Transmie la fiecare 60 secunde
bool overTheAirActivation = true;
bool loraWanAdr = true;
bool isTxConfirmed = true;
uint8_t appPort = 2;
uint8_t confirmedNbTrials = 4;

/* Prepares the payload of the frame */
static void prepareTxFrame( uint8_t port )
{
  // Citește datele de la DHT11
  float tempRead = dht.readTemperature();
  float humRead = dht.readHumidity();
  
  // Verifică dacă citirea a funcționat
  if (isnan(tempRead)  isnan(humRead)) {
    Serial.println("⚠️ Eroare la citirea DHT11! Folosesc date default.");
    dhtError = true;
    
    // Date default în caz de eroare
    temperature = 25.0;
    humidity = 50.0;
  } else {
    dhtError = false;
    temperature = tempRead;
    humidity = humRead;
    
    // Filtrăm valori imposibile
    if (temperature < -10  temperature > 60) temperature = 25.0;
    if (humidity < 0 || humidity > 100) humidity = 50.0;
  }
  
  // Pregătește datele pentru transmitere
  appDataSize = 4;
  appData[0] = (uint8_t)temperature;     // Temperatură (°C)
  appData[1] = (uint8_t)humidity;        // Umiditate (%)
  appData[2] = 100;                      // Baterie 100% (simulat)
  appData[3] = counter++;                // Contor
  
  // Afișează datele în Serial Monitor
  Serial.printf("🌡 DHT11 - Temp: %.1f°C, 💧 Hum: %.1f%%, 🔋 Bat: 100%%, 🔢 Cnt: %d", 
                temperature, humidity, appData[3]);
  
  if (dhtError) {
    Serial.println(" | ⚠️ DATE DEFAULT");
  } else {
    Serial.println(" | ✅ DATE REALE");
  }
}

void setup() {
  Serial.begin(115200);
  Serial.println("🚀 Initializare Heltec Stick V3 cu DHT11...");
  
  // Initializează placa
  Mcu.begin(HELTEC_BOARD, SLOW_CLK_TPYE);
  
  // Initializează senzorul DHT11
  dht.begin();
  Serial.println("✅ DHT11 initializat pe GPIO 13");
  Serial.println("⏳ Aștept 2 secunde pentru stabilizare senzor...");
  delay(2000);
  
  // Afișează configurația
  Serial.print("📡 DevEUI: ");
  for(int i=0; i<8; i++) {
    Serial.printf("%02X", devEui[i]);
  }
  Serial.println("\n🔑 Mod OTAA activat");
}
void loop()
{
  switch( deviceState )
  {
    case DEVICE_STATE_INIT:
    {
      #if(LORAWAN_DEVEUI_AUTO)
        LoRaWAN.generateDeveuiByChipID();
      #endif
      
      Serial.println("📡 Initializare LoRaWAN...");
      LoRaWAN.init(loraWanClass, loraWanRegion);
      LoRaWAN.setDefaultDR(3);
      break;
    }
    case DEVICE_STATE_JOIN:
    {
      Serial.println("🔗 Încerc conectare la rețea...");
      LoRaWAN.join();
      break;
    }
    case DEVICE_STATE_SEND:
    {
      Serial.println("📤 Trimit date...");
      prepareTxFrame( appPort );
      LoRaWAN.send();
      deviceState = DEVICE_STATE_CYCLE;
      break;
    }
    case DEVICE_STATE_CYCLE:
    {
      // Programează următoarea transmisie
      txDutyCycleTime = appTxDutyCycle + randr( -APP_TX_DUTYCYCLE_RND, APP_TX_DUTYCYCLE_RND );
      LoRaWAN.cycle(txDutyCycleTime);
      deviceState = DEVICE_STATE_SLEEP;
      break;
    }
    case DEVICE_STATE_SLEEP:
    {
      LoRaWAN.sleep(loraWanClass);
      break;
    }
    default:
    {
      deviceState = DEVICE_STATE_INIT;
      break;
    }
  }
}
```

## Find Your Computer IP
Wi-Fi IP: Use ipconfig and look for IPv4 Address

##

The example includes the [ChirpStack REST API](https://github.com/chirpstack/chirpstack-rest-api).
You should be able to access the UI by opening http://localhost:8090 in your browser.

**Note:** It is recommended to use the [gRPC](https://www.chirpstack.io/docs/chirpstack/api/grpc.html)
interface over the [REST](https://www.chirpstack.io/docs/chirpstack/api/rest.html) interface.
