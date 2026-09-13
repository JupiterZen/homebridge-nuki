# Nuki — Claude Context

> **Globale protocollen** in `~/.claude/CLAUDE.md` zijn altijd van kracht.
> Voordeur-bediening liep tot 1 sep 2026 via Homebridge, is sindsdien
> **volledig verhuisd naar Home Assistant** (zie "Nachtslot-switch verhuisd
> naar HA" verderop) — HA is nu de enige HomeKit-bron voor de Nuki.
> Bredere HA-context (niet Nuki-specifiek): `~/dev/smarthome/homeassistant/CLAUDE.md`.

Deze map bevat de gekloonde broncode van de `homebridge-nuki`-plugin
(upstream-referentie, niet leidend voor onze configuratie — zie
"Infrastructuur" hieronder voor de daadwerkelijk gebruikte fork).

## Infrastructuur

- **Raspberry Pi 5**: `192.168.2.193`, SSH-alias `pi-wp` (via Tailscale),
  user `justuspak` (bijgewerkt na de Pi 5-migratie — stond origineel als
  `192.168.2.166`)
- **Nuki Bridge**: `192.168.2.213:8080` (HTTP API, token vereist)
- **Nuki-lock**: `Nuki_30C330E4` (BLE-naam), `nukiId: 818098404`, **gekoppeld**
  aan de Bridge (stond origineel als "paired:false, nog te koppelen" — dat
  is achterhaald)
- **Plugin die daadwerkelijk draait**: `homebridge-nuki-pro` (een **fork**
  van de upstream `homebridge-nuki` in deze map — heeft `nightLockSwitch`,
  wat de originele plugin niet heeft; de upstream-README in deze map is
  dus **niet leidend** voor hoe onze installatie werkt)
- **Homebridge**: Docker container `homebridge`, host network mode, config
  dir `/home/justuspak/homebridge/` (`/homebridge/` in container), UI op
  `http://192.168.2.193:8581`

⚠️ Sinds 1 sep 2026 heeft Homebridge **geen** `NukiPlatform`-blok meer in
zijn `config.json` — zie de sectie hieronder. De rest van deze
infrastructuur-info (Bridge, lock-koppeling) blijft relevant, want HA
praat met dezelfde Bridge.

## API-referentie: Nuki Bridge lockAction-waarden

Officieel gedocumenteerd (Nuki Bridge API PDF v1.13.2):

| Actie | Betekenis (smartlock) |
|---|---|
| 1 | Unlock |
| 2 | Lock |
| 3 | Unlatch |
| 4 | Lock 'n' Go |
| 5 | Lock 'n' Go with unlatch |

⚠️ **`action=6` (Night Lock / volledige 2-toeren-vergrendeling) staat NIET
in de officiële PDF**, maar is bevestigd correct en werkend via twee
onafhankelijke bronnen:
1. `homebridge-nuki-pro`'s broncode (`nuki-smart-lock-device.js`), met de
   auteur's eigen comment *"Night Lock (full lock - 2 turns)"* naast de
   `action=6`-aanroep.
2. Fysieke test 31 aug 2026 (zie verderop): deur draaide hoorbaar 2x door
   bij "aan", 2x terug bij "uit" (zonder unlatch).

Les: Nuki's PDF is niet volledig — nieuwere/geavanceerdere firmware-acties
(hier: 2-toeren-vergrendeling, een fysieke Smart Lock-functie) staan er
kennelijk niet allemaal in. Derde-partij werkende broncode kan
betrouwbaarder zijn dan de officiële documentatie, mits fysiek geverifieerd
voor je 'm vertrouwt.

## Harde bevindingen over HomeKit + sloten (geverifieerd in HA's bron)

**HomeKit-slot kan niet unlatchen.** HA's `type_locks.py`:
`STATE_TO_SERVICE = {LOCKED: "lock", UNLOCKED: "unlock"}` — geen `open`.
HomeKit's `LockMechanism`-service kent alleen secured/unsecured. Dus "van
slot" in de Woning-app = `lock.unlock` = grendel terug, klink blijft.
HA's `nuki/lock.py`: `unlock()` → Bridge `unlock()`, `open()` → Bridge
`unlatch()`.

**Een `lock` in HomeKit vereist authenticatie, een `switch` niet.** Siri
vraagt Face ID, en scènes/automatiseringen krijgen een
bevestigingsmelding in plaats van uitvoering. Daarom is elke
nachtslot-schakelaar (zowel de oude Homebridge-versie als de nieuwe
HA-versie) bewust een `Switch` en geen lock-service. **Elke oplossing die
scène-bruikbaar moet zijn, moet dus een `switch` worden, geen `lock`.**

**HomeKit kent geen accu-apparaat.** Een batterijsensor los exporteren
doet niets — HA hangt hem automatisch aan het lock-accessoire zelf
(`CONF_LINKED_BATTERY_SENSOR`). De accusensor van het slot wordt dus
automatisch meegekoppeld met de hoofdtegel, geen aparte actie nodig.

## Hoe de voordeur vroeger in Homebridge werkte (`homebridge-nuki-pro`) — historisch

Config was (`nukiId: 818098404`):

```json
"unlatchFromLockedToUnlocked": false,   // "van slot" vanuit vergrendeld = action=1 (unlock)
"unlatchFromUnlockedToUnlocked": false, // nogmaals "van slot" = niets
"unlatchLock": true,                    // 2e slot-tegel die action=3 (unlatch) doet
"nightLockSwitch": true,                // switch: aan = action=6, uit = action=1
"lockSwitch": false
```

De nachtslot-switch werd gebruikt in een **"Goedemorgen"-scène**: hij ging
*uit*, wat `action=1` stuurde. Resultaat: nachtslot eraf, deur ontgrendeld,
je liep met alleen de klink naar buiten (hond uitlaten). Het nachtslot
*erop* deed de Nuki zelf aan het eind van de avond via zijn eigen interne
"Nachtmodus"-schedule (zie verderop) — niet via een Homebridge-commando,
want er werd destijds aangenomen dat er geen software-commando voor
bestond (`action=6` was toen nog niet ontdekt).

⚠️ Het `config.schema.json` van de plugin zegt dat uitzetten `action=2`
stuurt. De **code** (`nuki-smart-lock-device.js` r325-327) stuurt
`action=1`. De code is leidend.

## Nachtslot-switch verhuisd naar HA — structureel, niet parallel (31 aug/1 sep 2026)

Het oorspronkelijke advies ("laat de voordeur voorlopig bij Homebridge,
daar staat doordachte logica die HA niet één-op-één kan overnemen") is
**herzien**: de nachtslot-switch-functionaliteit is succesvol en
structureel naar HA verhuisd (HA leidend, Homebridge's `NukiPlatform`
volledig verwijderd — niet twee kopieën naast elkaar).

### Wat er in HA is gebouwd

**13 sep 2026: verhuisd naar een eigen package-bestand.** Stond
oorspronkelijk inline in `configuration.yaml` (505 regels, meerdere
apparaten door elkaar) — leeft nu in
`/home/justuspak/homeassistant/config/packages/nuki.yaml` op de Pi zelf
(HA's `packages:`-mechanisme, zie `~/dev/smarthome/CLAUDE.md`). Het
Bridge-token staat niet meer inline maar in `secrets.yaml` als de
volledige `nuki_nightlock_url` (niet alleen het token — `!secret` kan
geen Jinja-template-interpolatie doen, dus de complete URL is de secret).
Deze YAML zelf staat **niet** in een lokale git-repo; de Pi's
configuratie-map is geen git-checkout, maar wordt wél elke nacht 23:00
automatisch gesnapshot door `~/dev/smarthome/pi-backup/` (zie
`pi-backup/LEESMIJ.md`) — dat is waar de geschiedenis van deze config
te vinden is, niet hier.

```yaml
# packages/nuki.yaml (op de Pi)
rest_command:
  nuki_nightlock_on:
    url: !secret nuki_nightlock_url
    method: GET

template:
  - switch:
      - name: "Nachtslot Voordeur"
        unique_id: nuki_nachtslot
        state: "{{ is_state('lock.de_hemelpoort', 'locked') }}"
        turn_on:
          service: rest_command.nuki_nightlock_on
        turn_off:
          service: lock.unlock
          target:
            entity_id: lock.de_hemelpoort
```

De `homekit:`-whitelist-regels (`switch.nachtslot_voordeur`,
`lock.de_hemelpoort`) blijven bewust in `configuration.yaml` zelf staan —
dat is één gedeelde whitelist voor alle apparaten, niet per package te
splitsen zonder de structuur te breken.

Twee commando's, precies zoals Homebridge's `nightLockSwitch` ooit deed:
- **Aan** → `rest_command.nuki_nightlock_on` (custom, rechtstreeks naar de
  Bridge, action=6).
- **Uit** → `lock.unlock` (HA's **ingebouwde** `nuki`-integratie-service,
  stuurt zelf action=1) op `lock.de_hemelpoort`.

Toegevoegd aan HA's HomeKit-whitelist als `switch.nuki_nachtslot`
→ HA genereert het entity_id automatisch uit de `name` als
`switch.nachtslot_voordeur` (let op: **niet** gelijk aan de `unique_id`
`nuki_nachtslot` — de `template:`-syntax in de gebruikte HA-versie leidt
het entity_id af van `name`; oudere `switch: - platform: template`-syntax,
wat aanvankelijk geprobeerd is, gaf een harde configuratiefout: *"Configuring
the template integration under the switch platform key is not supported"*).

De `state`-template volgt bewust de echte `lock.de_hemelpoort`-staat
(`is_state(..., 'locked')`) in plaats van een vaste `{{ false }}` zoals
Homebridge's origineel — geeft een correct aan/uit-beeld in zowel HA als
de Woning-app, in plaats van een tegel die na een druk altijd "uit" blijft
tonen.

### Waarom dit een functionele upgrade is, niet alleen een verhuizing

Vóór de ontdekking van `action=6` werd aangenomen dat er **geen**
software-commando bestond om het 2-toeren-nachtslot te activeren. De oude
Homebridge-opzet liet daarom het nachtslot **automatisch** door de Nuki's
eigen interne schedule aanzetten (aan het eind van de avond) — alleen
"eraf halen" (unlock, altijd action=1) was software-matig bestuurbaar.

Met `action=6` nu bevestigd bruikbaar, heeft de nachtslot-switch in HA
voor het eerst **volledige controle in beide richtingen** vanuit
software — geen afhankelijkheid meer van de Nuki's interne klok om het
nachtslot te activeren.

### Uitvoering — voltooid 31 aug/1 sep 2026

1. ✅ HA-switch gebouwd, getest (HA-service-calls én via HomeKit/Woning-app
   bevestigd — beide richtingen doen de juiste 2x-draai, fysiek gehoord).
2. ✅ "Goedemorgen"-scène in de Woning-app (leeft alleen in Apple Home, niet
   in HA of Homebridge) omgezet naar `switch.nachtslot_voordeur`.
3. ✅ **Volledige `NukiPlatform`-blok verwijderd** uit Homebridge's
   `config.json` (niet alleen `nightLockSwitch`) — bewuste keuze om HA de
   **enige** HomeKit-bron voor de Nuki te maken, geen dubbele tegels naast
   elkaar. Backup staat op de Pi als `config.json.voor-nuki-uit-hb`.
   Homebridge herstart via `docker restart homebridge` (let op: `docker
   compose restart homebridge` binnen `~/homebridge` gaf geen zichtbare
   output/effect — directe `docker restart` werkte wel).
   Bevestigd in de Homebridge-log: de plugin blijft *geladen* (staat nog
   in `package.json`-dependencies, dat verdwijnt niet), maar zonder
   config-blok initialiseert hij geen accessoires meer; `Removing orphaned
   accessory De Hemelpoort` toont dat Homebridge de oude tegel(s) zelf
   netjes opruimt uit de cache — geen crash-cyclus zoals bij WiZ's
   `ignoredDevices` (zie `homeassistant/CLAUDE.md`), dit gedraagt zich
   zoals ook bij de Dyson-luchtreinigers (nette self-cleanup bij herstart).
   Bevestigd door gebruiker: oude Homebridge-Nuki-tegel weg uit de
   Woning-app, HA's `Voordeur HA` + `Nachtslot Voordeur` werken, scène
   intact.
4. Nuki's eigen interne "Nachtmodus"-schedule (in de Nuki-app, niet
   Homebridge/HA) staat **nog aan** — bewust nog niet uitgezet, zie
   volgende sectie voor wat dat behelst.

## Wat Nuki's eigen "Nachtmodus" (in-app) precies doet

Uitgezocht voor de vraag "wat verlies ik als ik Nachtmodus in de Nuki-app
zelf uitzet, nu HA action=6 ook kan aansturen?". De Nuki-app-tekst
bevestigt zelf al dat dit exact `action=6` is: *"draait het de deur twee
keer op slot (720°)"* — Nachtmodus is dus de **in-lock-firmware-versie**
van precies wat de HA-switch nu ook kan, alleen tijdgestuurd (schedule)
i.p.v. commando-gestuurd.

Nachtmodus is een **bundel** van sub-instellingen die allemaal
verdwijnen zodra je 'm uitzet (ze bestaan niet los):
- Begintijd/eindtijd van het venster
- "Vergrendelen op het begintijdstip" (activeert de 2x-draai bij start)
- "Auto Unlock afwijzen" (blokkeert Nuki's BLE-nabijheids-auto-ontgrendeling
  specifiek tijdens dit venster)
- "Activeer Auto Lock" (dit is een *duplicaat-schakelaar* voor het
  nacht-venster van een generieke, elders-in-de-app-liggende Auto
  Lock-instelling — die generieke instelling blijft gewoon actief
  ongeacht Nachtmodus aan/uit)
- Tijdsplanning (wat er ná het venster automatisch gebeurt)

**Zonder Nachtmodus** gedraagt het slot zich 's nachts identiek aan
overdag: gewone enkele lock/unlock, geen automatische 2x-cyclus, en
belangrijk — **Auto Unlock blijft dan ook 's nachts actief** (er is niets
meer dat 'm specifiek in dat venster blokkeert), tenzij je Auto Unlock
los ergens anders in de app uitzet.

## Open punten

- **Nog geen besluit: Nachtmodus (in de Nuki-app) uitzetten of laten
  staan?** Hangt af van of je Auto Unlock 's nachts wilt blijven toestaan
  (zie hierboven).
- **Unlatch-tegel vervallen** (losse "klink openen zonder ontgrendelen",
  `action=3`) — had geen HA-equivalent toen de Nuki volledig uit
  Homebridge werd gehaald (1 sep 2026). Bewust laten vervallen ("komen we
  later op terug"), geen vervanging gebouwd. Zelfde bouwpatroon als de
  nachtslot-switch zou hiervoor werken (`rest_command` met `action=3` +
  een losse switch) als het toch nodig blijkt.
- **Accessory mode voor het slot in HA.** HA waarschuwt: een lock hoort in
  een eigen HomeKit-instantie, anders valt hij bij elke HA-herstart even
  weg. Nu bewust niet gedaan (testfase, veel HA-herstarts). Netste eindvorm
  als het slot in HA blijft.

## Overig, uit de oorspronkelijke `homebridge-nuki`-context (niet Nuki-specifiek, ter referentie)

Deze map bevatte voorheen ook generieke Homebridge-infrastructuur-notities
die niet Nuki-specifiek zijn — die info is inmiddels achterhaald of
overlapt met wat in `~/dev/smarthome/homeassistant/CLAUDE.md` staat
(WiZ-lampen, Dyson-patches, SwitchBot). Zie dat bestand voor de actuele
stand van zaken over die apparaten. De oorspronkelijke, deels verouderde
Homebridge-kernpatches (`addIdentifyingMaterial`, mDNS-probe-fix) en
mDNS/HomeKit-troubleshooting-checklist stonden hier eerder gedocumenteerd
maar zijn generiek-Homebridge, niet Nuki-specifiek — nog te migreren naar
een toepasselijkere plek (bijv. een aparte `homebridge`-CLAUDE.md) als
ze nog relevant blijken.
