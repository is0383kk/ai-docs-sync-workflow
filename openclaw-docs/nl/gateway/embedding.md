---
read_when:
    - OpenClaw integreren in een desktop- of servertoepassing
    - De Gateway als onderliggend proces beheren
    - Gateway-gereedheid, herstarten, afsluiten of ongeldige configuratie afhandelen zonder logs uit te lezen
summary: Beheer de OpenClaw Gateway als een onderliggend proces vanuit Electron of een andere host-app
title: OpenClaw insluiten
x-i18n:
    generated_at: "2026-07-27T04:59:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ca67e03994f21446bfeca58c95c2cb624dde767b9983a89982627145f80dfb90
    source_path: gateway/embedding.md
    workflow: 16
---

Een embeddinghost moet toezicht houden op het geïnstalleerde uitvoerbare bestand `openclaw`, het
Gateway-WebSocket-protocol als besturingsvlak gebruiken en het onderliggende proces als een
vervangbare runtime behandelen. Zo blijven proceseigenaarschap, gereedheid, foutherstel
en upgrades expliciet, zonder afhankelijk te zijn van de privé-indeling van de status van OpenClaw.

Lees voor clientauthenticatie en de status bij opnieuw verbinden
[Een Gateway-client bouwen](https://docs.openclaw.ai/gateway/clients).

## Start het onderliggende proces met een embeddingpreset

Gebruik een echte installatie van `node_modules` en start het uitvoerbare bestand van het pakket. Een nuttige
basis voor een host die eigenaar is van detectie, herstarts en de levenscyclus van kanalen is:

```ts
import { spawn } from "node:child_process";
import { dirname, resolve } from "node:path";
import { fileURLToPath } from "node:url";

// Geef een absoluut pad op naar een echte Node-runtime die door de hosttoepassing wordt beheerd.
declare const hostNodeExecutable: string;

const packageEntry = fileURLToPath(import.meta.resolve("openclaw"));
const openclawEntry = resolve(dirname(packageEntry), "..", "openclaw.mjs");
const gateway = spawn(hostNodeExecutable, [openclawEntry, "gateway", "--allow-unconfigured"], {
  env: {
    ...process.env,
    OPENCLAW_DISABLE_BONJOUR: "1",
    OPENCLAW_EXEC_SHELL_SNAPSHOT: "0",
    OPENCLAW_NO_RESPAWN: "1",
    OPENCLAW_SKIP_CHANNELS: "1",
  },
  stdio: ["ignore", "inherit", "inherit"],
});
```

Los OpenClaw op via het geïnstalleerde pakket, zoals weergegeven; ga er niet van uit dat een
projectlokale binaire `openclaw` beschikbaar is in de `PATH` van het hostproces. Het voorbeeld
neemt de uitvoer over, zodat het onderliggende proces niet kan blokkeren op volle stdout- of stderr-pijpen. Als de
host deze stromen in plaats daarvan vastlegt, koppel dan onmiddellijk na het starten verwerkers eraan.

| Instelling                          | Effect op embedding                                                                                                                                                                           |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `OPENCLAW_DISABLE_BONJOUR=1`     | Schakelt door de Gateway beheerde LAN-multicastadvertenties uit wanneer de host eigenaar is van de detectie.                                                                                                             |
| `OPENCLAW_NO_RESPAWN=1`          | Voorkomt in een onbeheerd onderliggend embeddingproces dat OpenClaw een herstart na een update overdraagt aan een losgekoppeld onderliggend proces. Routinematige herstarts blijven in het proces, zodat de host eigenaar blijft van de bijgehouden PID. |
| `OPENCLAW_EXEC_SHELL_SNAPSHOT=0` | Schakelt het vastleggen van een login-shellsnapshot voor uitvoeropdrachten van de host uit.                                                                                                                              |
| `OPENCLAW_SKIP_CHANNELS=1`       | Slaat het starten en herladen van kanalen over. Stel dit alleen in wanneer de embeddingapp een Gateway wil die uitsluitend als besturingsvlak of voor WebChat dient.                                                                        |

`--allow-unconfigured` omzeilt alleen de opstartbeveiliging `gateway.mode=local`. Het
schrijft geen configuratie en herstelt geen ongeldig bestand. Laat dit weg wanneer de embeddingapp
een normale lokale configuratie beschikbaar stelt via onboarding, de configuratie-CLI
of Gateway-RPC.

### Waarschuwing voor Electron-shellsnapshots

Het vastleggen van een shellsnapshot voert `process.execPath -e <script>` uit vanuit een login-shell. In
een normaal Node-proces is `process.execPath` het uitvoerbare bestand van Node. Onder Electron
is dit het binaire bestand van Electron, dat de aanroep kan interpreteren als het starten van een toepassing
en een pop-up met "Unable to find Electron app" kan weergeven. Stel
`OPENCLAW_EXEC_SHELL_SNAPSHOT=0` in de omgeving van het onderliggende Gateway-proces in, niet alleen in
het rendererproces. Om dezelfde reden moet `hostNodeExecutable` naar een
echte Node-runtime verwijzen in plaats van naar `process.execPath` van Electron.

## Verwerk ongeldige configuratie via de afsluitcode

Bij het opstarten gebruikt de Gateway afsluitcode `78` (`EX_CONFIG`) voor opstartfouten
van de configuratieklasse, waaronder een ongeldige configuratie. Maak een vertakking op basis van de afsluitcode in plaats van
voor mensen leesbare stderr te ontleden:

1. Voer `openclaw doctor --fix --yes --non-interactive` uit met dezelfde configuratie- en
   statusomgeving als het onderliggende Gateway-proces.
2. Probeer de Gateway één keer opnieuw te starten nadat doctor succesvol is afgesloten.
3. Als het onderliggende proces opnieuw wordt afgesloten met `78`, stop dan de herstellus en toon de configuratiefout
   aan de gebruiker.

Bewaar stderr voor diagnostiek, maar baseer beslissingen over de levenscyclus niet op de formulering ervan.

Na een geslaagde start heeft een ongeldige livebewerking van de configuratie minder ingrijpende gevolgen. De
configuratiewatcher registreert dat het herladen is overgeslagen en blijft de laatst geaccepteerde
configuratie in het geheugen gebruiken. Herstel het bestand en laat de watcher vervolgens de volgende geldige
snapshot accepteren.

## Wacht op protocolgereedheid

Gebruik WebSocket-signalen in plaats van een substring uit een logboek:

1. Open de Gateway-WebSocket.
2. Wacht op de gebeurtenis `connect.challenge`. Hiermee wordt aangetoond dat de listener de
   WebSocket heeft geaccepteerd en de challenge-handshake kan beginnen.
3. Verzend `connect` met de aan de challenge gekoppelde apparaathandtekening.
4. Beschouw `hello-ok` als toepassingsgereedheid voor geauthenticeerde RPC.

De challenge vindt bewust eerder plaats dan de volledige initialisatie. Als opstartende
sidecars nog in behandeling zijn, retourneert `connect` een opnieuw te proberen fout `UNAVAILABLE` met
`details.reason: "startup-sidecars"`, een begrensde `retryAfterMs`, en sluit vervolgens
met code `1013` en reden `gateway starting`. Gebruik
`resolveGatewayStartupRetryAfterMs` uit
`@openclaw/gateway-protocol/startup-unavailable` of het ingebouwde beleid van de referentieclient en maak vervolgens opnieuw verbinding.

## Interpreteer herstarts en afsluitingen

Vóór een ordelijke sluiting zendt de Gateway een gebeurtenis `shutdown` uit met `reason`
en `restartExpectedMs`. Een niet-nullwaarde voor `restartExpectedMs` betekent dat een herstart binnen het proces of
onder toezicht wordt verwacht; `null` betekent een definitieve afsluiting.

De daaropvolgende WebSocket-sluitcode is in beide gevallen `1012`. De normale
sluitreden van de client is in beide gevallen eveneens `service restart`, zodat noch de sluitcode noch
de reden onderscheid maakt tussen een herstart en een afsluiting. Bewaar de voorafgaande payload `shutdown`
wanneer deze binnenkomt en combineer deze met de eigen stopintentie van de host en de
afsluitstatus van het onderliggende proces. Als de verbinding zonder de gebeurtenis verdwijnt, gebruik dan het normale
begrensde beleid voor opnieuw verbinden en toezicht op het onderliggende proces.

## Gebruik RPC in plaats van statusbestanden

Houd de Gateway als enige eigenaar van de OpenClaw-status. Veelgebruikte embeddingbewerkingen
hebben al RPC-methoden:

| Taak                          | RPC-methoden                                          |
| ----------------------------- | ---------------------------------------------------- |
| Sessiecatalogus en levenscyclus | `sessions.list`, `sessions.patch`, `sessions.delete` |
| Transcriptweergave            | `chat.history`                                       |
| Kosten- en gebruiksrapporten        | `usage.cost`, `sessions.usage`                       |
| Status van modelreferenties       | `models.authStatus`                                  |
| Configuratie                 | `config.get`, `config.patch`                         |

`config.get` maskeert gevoelige waarden en SecretRef-identificatoren voordat
de snapshot wordt geretourneerd. Schrijfmethoden retourneren ook een gemaskeerde configuratie. Een client moet de
maskeringsmarkering als opaak behandelen en het gedocumenteerde contract voor het schrijven van configuratie gebruiken; deze
mag er nooit van uitgaan dat de Gateway geheimen als platte tekst retourneert.

Lees of wijzig geen bestanden, SQLite-tabellen, transcriptbestanden of cachemappen
onder `~/.openclaw` om appfuncties te implementeren. Deze indelingen zijn privé-implementatiedetails van de runtime
en kunnen worden verplaatst of gewijzigd zonder protocolcompatibiliteit.

## Installeer; niet afvlakken

Het hoofdpakket `openclaw` is geen doel voor vendoring als één bestand. Gebundelde runtimebestanden
onder `dist/extensions` behouden kale zelfimports, zoals
`openclaw/plugin-sdk/*`, terwijl het npm-pakket bewust
`node_modules`-structuren per extensie uitsluit.

Installeer OpenClaw via npm, pnpm of een andere normale Node-pakketinstallatie, zodat
Node de pakketexports en de afhankelijkheidsstructuur van het hoofdpakket kan oplossen. Start het geïnstalleerde
uitvoerbare bestand `openclaw`. Kopieer niet alleen `dist`, vlak het pakket niet af tot een appbundle
en neem geen geselecteerde extensiebestanden op als vendored code.

## Gerelateerd

- [Een Gateway-client bouwen](https://docs.openclaw.ai/gateway/clients)
- [Gateway-protocol](https://docs.openclaw.ai/gateway/protocol)
- [Gateway-CLI](https://docs.openclaw.ai/cli/gateway)
- [Gateway-integraties voor externe apps](https://docs.openclaw.ai/gateway/external-apps)
