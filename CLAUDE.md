# HomeBridge / Nuki — Claude Context

> **Globale protocollen** in `~/.claude/CLAUDE.md` (verificatie, debugging, directe fixes) zijn altijd van kracht.

## Infrastructuur

- **Raspberry Pi**: `192.168.2.166`, SSH alias `pi-wp`, user `justuspak`
- **HomeBridge**: Docker container `homebridge` (image `homebridge/homebridge:latest`), host network mode
- **Config dir**: `/home/justuspak/homebridge/` (= `/homebridge/` in container)
- **UI**: http://192.168.2.166:8581
- **HAP bridge**: username `0E:73:46:80:3F:BB`, PIN `193-59-731`, port `51164`
- **Advertiser**: `bonjour-hap`

## Security

- `config.json` staat in `.gitignore` — bevat credentials (Nuki token, WiZ MACs, etc.)
- Geschiedenis is opgeschoond via `filter-branch` (2026-07-02)
- **Nooit** `config.json` committen

## Kritische patches (gaan verloren bij update)

Alle patches zitten in `startup.sh` op de Pi (`/home/justuspak/homebridge/startup.sh`) en draaien automatisch bij elke container start.

### 1. addIdentifyingMaterial fix — bridgeService.js
**Bestand**: `/home/justuspak/homebridge/node_modules/homebridge/dist/bridgeService.js`
**Regels 176 en 431**: `addIdentifyingMaterial: false`

**Waarom**: HAP-nodejs voegt bij `true` SHA512(username).slice(0,4) = `"E7AB"` toe aan de bridge displayName bij elke herstart. Elke naamswijziging bumpt c# → HomeKit sync storm.

**Controleer na homebridge update**:
```bash
ssh pi-wp "grep -n 'addIdentifyingMaterial' /home/justuspak/homebridge/node_modules/homebridge/dist/bridgeService.js"
# Moet 'false' zijn op beide regels
```

### 2. mDNS probe conflict fix — Advertiser.js
**Bestand**: `/home/justuspak/homebridge/node_modules/homebridge/node_modules/@homebridge/hap-nodejs/dist/lib/Advertiser.js`
- **Regel 79**: CiaoAdvertiser `name-change` propagatie uitgeschakeld (emitte niet meer `updated-name`)
- **Regel 174**: `probe: false` toegevoegd aan BonjourHAPAdvertiser publish call

**Waarom**: HomePod cacht HAP service en antwoordt authoritatively op mDNS probes → advertiser hernoemt → c# bumpt → volledige HomeKit resync bij elke herstart.

**Symptoom als patch weg is**: Bridge naam groeit (D1E2 → D1E2 E7AB → D1E2 E7AB E7AB), c# stijgt bij elke herstart, HomeKit toont "geen reactie" na herstart.

### 3. Dyson HF1 (635) CurrentAirPurifierState fallback — dyson-pure-cool-device.js
**Bestand**: `/home/justuspak/homebridge/node_modules/homebridge-dyson-pure-cool/src/dyson-pure-cool-device.js`
HF1 stuurt geen `fmod`/`auto` veld — fallback op `fpwr`+`fnst` toegevoegd in CURRENT-STATE en STATE-CHANGE handlers.

### 4. Dyson type 635 (HF1) — productTypeInfo.js
**Bestand**: `productTypeInfo.js` in homebridge-dyson-pure-cool
Type `635` toegevoegd. Config: `isSingleAccessoryModeEnabled:true`, `isTemperatureSensorEnabled:false`.

### 5. WiZ tunable white warm-up fix — wiz.js
**Bestand**: `homebridge-wiz-lan/dist/wiz.js`
Na elke restart start `cachedPilot` leeg. `setPilot` faalt de eerste 60s met "No cached state". AdaptiveLighting (alleen Tunable White) stuurt in die periode color-temp updates → fout → HomeKit cachet "No Response".
Fix: roept direct na `bindSocket` `getPilot()` aan voor alle geïnitialiseerde accessories.
**Let op**: warm-up mag NIET in `initAccessory` — socket is dan nog niet gebonden en `socket.send()` auto-bindt op willekeurige poort, waarna expliciete `bind(38900)` crasht.

### 6. Dyson PC3 type 438N — productTypeInfo.js
**Bestand**: `productTypeInfo.js` in homebridge-dyson-pure-cool
Type `438N` toegevoegd (TP14-AC, Find+Follow™). Serial: `K8T-EU-VCA0748A`, IP: `192.168.2.243`.
Config: `isSingleAccessoryModeEnabled:false`, `isTemperatureSensorEnabled:true`.

## Devices

| Device | IP | Plugin |
|--------|-----|--------|
| Dyson PC1/TP11 (438M) | 192.168.2.22 | homebridge-dyson-pure-cool |
| Dyson PC3/TP14 (438N) | 192.168.2.243 | homebridge-dyson-pure-cool |
| Dyson HF1 (635) | 192.168.2.217 | homebridge-dyson-pure-cool |
| Nuki Bridge | 192.168.2.213:8080 | homebridge-nuki |
| SwitchBot gordijnen | — | homebridge-switchbot (gepatcht) |
| Sonos | — | homebridge-zp |
| WiZ lampen (14x) | zie tabel | homebridge-wiz-lan |
| HomeConnect | — | homebridge-homeconnect |

## WiZ lampen — MAC → naam mapping

Alle 14 lampen staan met naam in `config.json` onder het WiZ platform (`devices` array).

| MAC | Naam | Type | Kamer |
|-----|------|------|-------|
| 9877d5d14386 | Gang | RGB | Gang |
| 9877d5d17ec8 | Grote Lamp Slaapkamer | RGB | Slaapkamer |
| 9877d5d2e568 | Bedlampje Slaapkamer | RGB | Slaapkamer |
| 9877d5c03d66 | Afzuigkap Links | RGB | Keuken |
| 9877d5d2e70a | Grote Lamp Kinderkamer | RGB | Kinderkamer |
| 9877d5cd3e3c | Afzuigkap Rechts | RGB | Keuken |
| 9877d5d68222 | Badkamer | RGB | Badkamer |
| 9877d5d68238 | Raam Rechts | RGB | Woonkamer |
| 9877d5b0031c | Raam Links 1 | TW | Woonkamer |
| 9877d505427c | Raam Links 2 | TW | Woonkamer |
| 9877d5b00202 | Raam Links 3 | TW | Woonkamer |
| 9877d50546ea | Raam Links 4 | TW | Woonkamer |
| 9877d5b007e8 | Raam Links 5 | TW | Woonkamer |
| 9877d5b0030c | Raam Links 6 | TW | Woonkamer |

## Nuki

- **Bridge API**: `http://192.168.2.213:8080/` (token vereist)
- **Lock**: `Nuki_30C330E4`, momenteel `paired:false` (nog te koppelen via Nuki app)
- `/list` geeft `[]` als lock niet gekoppeld is aan bridge — normaal gedrag

## HomeBridge herstarten

```bash
ssh pi-wp "sudo docker restart homebridge"
ssh pi-wp "sudo docker logs homebridge --tail 50 -f"
```

## Na elke homebridge npm update: patches controleren

```bash
ssh pi-wp "grep -n 'addIdentifyingMaterial' /home/justuspak/homebridge/node_modules/homebridge/dist/bridgeService.js"
ssh pi-wp "grep -n 'probe' /home/justuspak/homebridge/node_modules/homebridge/node_modules/@homebridge/hap-nodejs/dist/lib/Advertiser.js"
```

## mDNS / HomeKit troubleshooting checklist

1. `sf` in mDNS TXT record moet `0` zijn (paired) — check met `dns-sd -B _hap._tcp` op Mac
2. c# moet stabiel blijven na herstart (niet stijgen) — patch 1+2 aanwezig?
3. Bridge naam in UI moet exact overeenkomen met persist file `displayName`
4. Persist file: `/home/justuspak/homebridge/persist/AccessoryInfo.0E7346803FBB.json`
5. "Geen reactie" na herstart zonder c# stijging → HomeKit hub (HomePod) heeft connectie nodig (~1 min)
6. "Out of compliance" na remove+re-add → iCloud heeft gecachte staat, wacht 15 min of probeer ander netwerk
