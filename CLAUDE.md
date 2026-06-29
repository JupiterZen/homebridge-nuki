# HomeBridge / Nuki — Claude Context

> **Globale protocollen** in `~/.claude/CLAUDE.md` (verificatie, debugging, directe fixes) zijn altijd van kracht.

## Infrastructuur

- **Raspberry Pi**: `192.168.2.166`, SSH alias `pi-wp`, user `justuspak`
- **HomeBridge**: Docker container `homebridge` (image `homebridge/homebridge:latest`), host network mode
- **Config dir**: `/home/justuspak/homebridge/` (= `/homebridge/` in container)
- **UI**: http://192.168.2.166:8581
- **HAP bridge**: username `0E:73:46:80:3F:BB`, PIN `193-59-731`, port `51164`
- **Advertiser**: `bonjour-hap`

## Kritische patches (gaan verloren bij update)

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

### 3. Dyson PC1/TP11 (438M) patch
**Bestand**: `/home/justuspak/homebridge/node_modules/homebridge-dyson-pure-cool/src/dyson-pure-cool-device.js`
Plugin `homebridge-dyson-pure-cool@2.9.3` — specifieke fix voor naam/services bug.

### 4. Dyson HF1 (635) patch
**Bestand**: `productTypeInfo.js` in homebridge-dyson-pure-cool
Type `635` toegevoegd. Config: `isSingleAccessoryModeEnabled:true`, `isTemperatureSensorEnabled:false`.

### 5. Dyson HF1 CurrentAirPurifierState fallback
**Bestand**: `dyson-pure-cool-device.js`
HF1 stuurt geen `fmod`/`auto` veld — fallback op `fpwr`+`fnst` toegevoegd in CURRENT-STATE en STATE-CHANGE handlers.

### 6. Deebot ServiceLabel fix
**Bestand**: `homebridge-deebot/lib/platform.js`
Plugin gebruikte `ServiceLabelIndex` zonder `ServiceLabel` service → HAP "out of compliance". Fix voegt `ServiceLabel` (namespace=1) toe vóór de eerste Switch service.

### 7. Dyson PC3 type 438N
**Bestand**: `productTypeInfo.js` in homebridge-dyson-pure-cool
Type `438N` toegevoegd (TP14-AC, Find+Follow™). Serial: `K8T-EU-VCA0748A`, IP: `192.168.2.243`.
Config: `isSingleAccessoryModeEnabled:true`, `isTemperatureSensorEnabled:true`.

### 8. WiZ tunable white warm-up fix
**Bestand**: `homebridge-wiz-lan/dist/wiz.js`
Na elke restart start `cachedPilot` leeg. `setPilot` faalt de eerste 60s met "No cached state". AdaptiveLighting (alleen Tunable White) stuurt in die periode color-temp updates → fout → HomeKit cachet "No Response".
Fix: roept direct na `bindSocket` `getPilot()` aan voor alle geïnitialiseerde accessories.
**Let op**: warm-up mag NIET in `initAccessory` — socket is dan nog niet gebonden en `socket.send()` auto-bindt op willekeurige poort, waarna expliciete `bind(38900)` crasht.

## Devices

| Device | IP | Plugin |
|--------|-----|--------|
| Dyson PC1/TP11 (438M) | 192.168.2.22 | homebridge-dyson-pure-cool |
| Dyson PC3/TP14 (438N) | 192.168.2.243 | homebridge-dyson-pure-cool |
| Dyson HF1 (635) | 192.168.2.217 | homebridge-dyson-pure-cool |
| Nuki Bridge | 192.168.2.213:8080 | homebridge-nuki |
| Deebot X11 "Marieke" | cloud | homebridge-deebot |
| SwitchBot | — | homebridge-switchbot (gepatcht) |
| Sonos | — | homebridge-zp |
| WiZ lampen | — | homebridge-wiz-lan |
| HomeConnect | — | homebridge-homeconnect |

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
