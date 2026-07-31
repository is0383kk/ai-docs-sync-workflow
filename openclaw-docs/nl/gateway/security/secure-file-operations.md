---
read_when:
    - Bestandstoegang, archiefextractie, werkruimteopslag of bestandssysteemhelpers voor plugins wijzigen
summary: Hoe OpenClaw lokale bestandstoegang veilig afhandelt en waarom de optionele Python-helper fs-safe standaard is uitgeschakeld
title: Veilige bestandsbewerkingen
x-i18n:
    generated_at: "2026-07-27T05:35:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5c8edf36ddbb8c8bc1edc52ecdf481affe5395d1779c679a40439167dfe70299
    source_path: gateway/security/secure-file-operations.md
    workflow: 16
---

OpenClaw gebruikt [`@openclaw/fs-safe`](https://github.com/openclaw/fs-safe) voor beveiligingsgevoelige lokale bestandsbewerkingen: tot een hoofdmap begrensd lezen/schrijven, atomair vervangen, archieven uitpakken, tijdelijke werkruimten, JSON-status en verwerking van geheime bestanden.

Het is een **beveiligingsmechanisme in de bibliotheek** voor vertrouwde OpenClaw-code die niet-vertrouwde padnamen ontvangt, geen sandbox. Bestandssysteemmachtigingen van de host, OS-gebruikers, containers en het agent-/toolbeleid bepalen nog steeds de werkelijke impact.

## Standaard: geen Python-helper

OpenClaw schakelt de POSIX Python-helper van fs-safe standaard **uit**:

- de Gateway mag geen permanente Python-sidecar starten, tenzij een beheerder daar expliciet voor kiest;
- de meeste installaties hebben de extra beveiliging tegen wijzigingen van bovenliggende mappen niet nodig;
- door Python uit te schakelen, blijft het runtimegedrag voorspelbaar in desktop-, Docker-, CI- en gebundelde app-omgevingen.

OpenClaw wijzigt alleen de _standaardwaarde_. Een expliciete instelling heeft altijd voorrang:

```bash
# Standaardgedrag van OpenClaw: uitsluitend op Node gebaseerde fs-safe-fallbacks.
OPENCLAW_FS_SAFE_PYTHON_MODE=off

# Gebruik de helper indien beschikbaar, met een fallback als deze niet beschikbaar is.
OPENCLAW_FS_SAFE_PYTHON_MODE=auto

# Stop veilig als de helper niet kan starten.
OPENCLAW_FS_SAFE_PYTHON_MODE=require

# Optioneel expliciet pad naar de interpreter.
OPENCLAW_FS_SAFE_PYTHON=/usr/bin/python3
```

De algemene fs-safe-omgevingsvariabelen werken ook: `FS_SAFE_PYTHON_MODE` en `FS_SAFE_PYTHON`.

Gebruik `require` (niet `auto`) wanneer de helper deel uitmaakt van je beveiligingsmodel; `auto` valt stilzwijgend terug op uitsluitend Node-gebaseerd gedrag als de helper niet kan starten.

## Wat zonder Python beschermd blijft

Als de helper is uitgeschakeld, profiteert OpenClaw nog steeds van de uitsluitend Node-gebaseerde beveiligingsmechanismen van fs-safe:

- weigert ontsnappingen via relatieve paden (`..`), absolute paden en padscheidingstekens waar alleen losse namen zijn toegestaan;
- voert bewerkingen uit via een vertrouwde hoofdmaphandle in plaats van ad-hoccontroles met `path.resolve(...).startsWith(...)`;
- weigert patronen met symbolische en harde koppelingen voor API's waarvoor dat beleid geldt;
- opent bestanden met identiteitscontroles wanneer de API bestandsinhoud retourneert of verwerkt;
- schrijft status-/configuratiebestanden via een atomair tijdelijk bestand in dezelfde map, gevolgd door hernoemen;
- handhaaft bytelimieten voor leesbewerkingen en het uitpakken van archieven;
- past privébestandsmodi toe op geheime en statusbestanden waar de API die vereist.

Dit dekt het normale dreigingsmodel van OpenClaw: vertrouwde Gateway-code die niet-vertrouwde padinvoer van modellen/plugins/kanalen verwerkt binnen één vertrouwde beheerdersgrens.

## Wat Python toevoegt

Op POSIX houdt de optionele helper één permanent Python-proces actief en gebruikt die aan bestandsdescriptoren gerelateerde bestandssysteembewerkingen voor wijzigingen van bovenliggende mappen: hernoemen, verwijderen, mappen aanmaken, status opvragen/mappen weergeven en sommige schrijfpaden.

Dit verkleint racevensters voor dezelfde UID waarin een ander proces een bovenliggende map verwisselt tussen validatie en wijziging — extra gelaagde beveiliging op hosts waar niet-vertrouwde lokale processen dezelfde mappen kunnen wijzigen waarin OpenClaw werkt.

Als je implementatie dit risico loopt en Python gegarandeerd beschikbaar is, stel je het volgende in:

```bash
OPENCLAW_FS_SAFE_PYTHON_MODE=require
```

## Richtlijnen voor plugins en kerncode

- Bestandstoegang voor plugins moet via de helpers van `openclaw/plugin-sdk/*` verlopen, niet rechtstreeks via `fs`, wanneer een pad afkomstig is uit een bericht, modeluitvoer, configuratie of plugininvoer.
- Kerncode moet de fs-safe-wrappers onder `src/infra/*` gebruiken, zodat het procesbeleid van OpenClaw consistent wordt toegepast.
- Voor het uitpakken van archieven moeten de fs-safe-archiefhelpers worden gebruikt met expliciete limieten voor grootte, aantal items, koppelingen en bestemming.
- Voor geheimen moeten de geheimhelpers van OpenClaw of de helpers van fs-safe voor geheimen/privéstatus worden gebruikt; implementeer niet zelf moduscontroles rond `fs.writeFile`.
- Vertrouw voor isolatie tegen vijandige lokale gebruikers niet alleen op fs-safe. Voer afzonderlijke Gateways uit onder afzonderlijke OS-gebruikers/hosts of gebruik sandboxing.

Zie ook: [Beveiliging](/nl/gateway/security), [Sandboxing](/nl/gateway/sandboxing), [Uitvoeringsgoedkeuringen](/nl/tools/exec-approvals), [Geheimen](/nl/gateway/secrets).
