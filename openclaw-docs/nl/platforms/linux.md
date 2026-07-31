---
read_when:
    - Op zoek naar de status van de Linux-companion-app
    - Camera, locatie of meldingen inschakelen op een Linux-Node-host
    - Platformondersteuning of bijdragen plannen
    - Linux OOM-beëindigingen of afsluitcode 137 op een VPS of in een container debuggen
summary: Status van Linux-ondersteuning en companion-app
title: Linux-app
x-i18n:
    generated_at: "2026-07-27T05:21:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fe55d3ec63fcf8291a24126c04638f005c03c3d44ff84a26a925e931066b01cc
    source_path: platforms/linux.md
    workflow: 16
---

De Gateway wordt volledig ondersteund op Linux en vereist Node. Bun kan nog steeds worden gebruikt
als installatieprogramma voor afhankelijkheden of om pakketscripts uit te voeren, maar kan OpenClaw
niet uitvoeren omdat het geen `node:sqlite` biedt.

## Desktopcompanion

De OpenClaw Linux-companion is een Tauri-desktopapp voor een lokale Gateway. Deze:

- installeert de OpenClaw CLI en beheerde Node-runtime wanneer die ontbreken; releasebuilds installeren automatisch het stabiele kanaal, terwijl ontwikkelbuilds eerst om het kanaal vragen
- maakt verbinding met een gezonde Gateway voordat wordt geprobeerd services te wijzigen
- delegeert installatie-, start-, stop- en herstartbewerkingen aan de door de CLI beheerde systemd-gebruikersservice
- ontdekt nabije Bonjour-Gateways en opent elke Control UI in een routegebonden venster, zodat meerdere
  Gateway-dashboards verbonden kunnen blijven en gelijktijdig kunnen worden gebruikt
- opent de door de Gateway aangeboden Control UI met de vastgestelde authenticatie-URL
- opent de Control UI na de eerste installatie in de onboardingmodus, waarin
  wordt aangeboden om gedetecteerde geheugens van Claude Code, Codex of Hermes in de
  agentwerkruimte te importeren (dezelfde import blijft later beschikbaar onder
  Settings → Import Memory)
- rendert door agents aangestuurde Canvas- en gebundelde A2UI-inhoud voor een CLI-nodehost op dezelfde locatie
- blijft beschikbaar vanuit het systeemvak wanneer het venster wordt gesloten

Stabiele releases die vanuit `main` zijn gebouwd, leveren `.deb`- en AppImage-bundels als assets bij de
[GitHub-release](https://github.com/openclaw/openclaw/releases) voor de tag,
met de namen `OpenClaw-<version>-amd64.deb` en `OpenClaw-<version>-amd64.AppImage`,
met daarnaast een `SHA256SUMS.linux-app.txt`-checksum-bestand. Download het
`.deb` en installeer het met `sudo apt install ./OpenClaw-<version>-amd64.deb`,
of markeer de AppImage als uitvoerbaar en voer deze rechtstreeks uit. De AppImage-runtime
vereist FUSE 2 (`sudo apt install libfuse2`, of `libfuse2t64` op Ubuntu 24.04+);
voer de AppImage zonder FUSE 2 uit met `APPIMAGE_EXTRACT_AND_RUN=1`.

Je kunt dezelfde bundels ook bouwen vanuit een broncheckout:

```bash
cd apps/linux/src-tauri
pnpm dlx @tauri-apps/cli@2.11.4 build --bundles deb,appimage
```

De CI-workflow `Linux App` uploadt dezelfde bundels als het
artefact `openclaw-linux-companion` voor pull requests die de app wijzigen en voor
handmatige uitvoeringen. Zie `apps/linux/README.md` in de repository voor Linux-buildafhankelijkheden
en ontwikkelopdrachten.

### Quick Chat

Open Quick Chat met `Ctrl+Shift+Space` of via het systeemvakitem **Quick Chat**. De agentchip
toont de geconfigureerde avatar, emoji of het monogram; selecteer deze om van agent te wisselen.
Berichten gebruiken de hoofdsessie van de geselecteerde agent en respecteren het globale sessiebereik.
De systeemeigen Rust-client beheert een permanente Ed25519-apparaatidentiteit. Deze gebruikt het
gedeelde token of wachtwoord uit de CLI-overdracht alleen om de koppeling te initialiseren en slaat vervolgens
het door de Gateway uitgegeven apparaattoken op, waaraan bij latere verbindingen
de voorkeur wordt gegeven. De identiteit en het apparaattoken staan in de appconfiguratiemap in een bestand met modus `0600`; de WebView van Quick
Chat ontvangt geen inloggegevens en evenmin de WebSocket.

Wanneer de systeemeigen verbinding niet beschikbaar is, toont Quick Chat **Gateway
onbereikbaar — opnieuw proberen** en wordt verzenden uitgeschakeld totdat opnieuw verbinding is gemaakt. Voor een extern apparaat
dat de koppelingsfase heeft bereikt, wordt in plaats daarvan **Keur dit apparaat goed in het dashboard
(Nodes)** weergegeven, met een korte apparaat-ID wanneer de Gateway die verstrekt. Een
Gateway die ontbrekende gedeelde inloggegevens vereist, toont **Gateway vereist
inloggegevens — open het dashboard op de Gateway-host**; in die toestand wacht geen koppelingsverzoek
op goedkeuring. Door de server verstrekte herstelrichtlijnen
vervangen deze standaardmeldingen wanneer ze specifieker zijn.
Voor TLS-Gateways geeft de CLI de SHA-256-vingerafdruk van het Gateway-certificaat
door aan de app; de systeemeigen client zet dat certificaat vast en meldt **Vertrouwen in Gateway-TLS
mislukt — controleer de certificaatvingerafdruk** afzonderlijk van uitval.
Gateways waarvan het gedeelde geheim via een SecretRef is geconfigureerd, laten dit weg uit de
CLI-overdracht. Bestaande gekoppelde installaties blijven werken via hun opgeslagen apparaattoken,
maar een nieuwe installatie kan bij authenticatie met een gedeeld geheim geen wachtend koppelingsverzoek maken
zonder die initiële inloggegevens.
Voor het inwisselen van een installatiecode en `bootstrapToken` is specifieke product-UI nodig; dit blijft
vervolgwerk. Quick Chat probeert geen van beide flows uit te voeren.

Gebruik op X11 het tandwiel in Quick Chat om een aangepaste sneltoets op te nemen of opnieuw in te stellen. De
systeemvakschakelaar **Quick Chat shortcut** schakelt deze in of uit zonder het gewone
systeemvakitem **Quick Chat** uit te schakelen. Globale sneltoetsen zijn niet beschikbaar op Wayland, dus
de sneltoetsinstellingen zijn verborgen en het systeemvakitem blijft het toegangspunt.
Na een geaccepteerd verzendverzoek blijft Quick Chat geopend en streamt het antwoord in platte tekst van de geselecteerde agent
onder het invoerveld. Druk op `Esc` om de balk en het antwoord te sluiten;
`Ctrl+Enter` opent nog steeds het dashboard.

### Canvas

Linux Canvas gebruikt twee samenwerkende processen. `openclaw node run` blijft de enige Gateway-nodeverbinding; de gebundelde Plugin `linux-canvas` stuurt `canvas.*`-aanroepen door naar de actieve desktopapp via een Unix-socket die alleen voor de gebruiker toegankelijk is. De app beheert één WebView-venster op aanvraag, inclusief de gebundelde A2UI-renderer en actiebrug terug naar de agent.

De Plugin is standaard ingeschakeld. Deze kondigt Canvas alleen aan wanneer de desktopsocket bestaat op `$XDG_RUNTIME_DIR/openclaw-canvas.sock`, of `/tmp/openclaw-canvas-$UID.sock` wanneer `XDG_RUNTIME_DIR` niet beschikbaar is. Schakel deze uit met `plugins.entries.linux-canvas.enabled: false`. Op een headless Linux-server zonder de desktopapp wordt Canvas niet aangekondigd.

Linux v1 gebruikt één Canvas-venster. HTTP- en HTTPS-pagina's kunnen worden gerenderd, maar A2UI-acties worden alleen geaccepteerd vanuit de gebundelde renderer.

## Alternatief met CLI en SSH

De CLI blijft de eenvoudigste optie voor een headless server, een VPS of een externe Gateway:

1. Installeer Node 24.15+ (aanbevolen), Node 22.22.3+ (LTS) of Node 25.9+.
2. `npm i -g openclaw@latest`
3. `openclaw onboard --install-daemon`
4. Vanaf je laptop: `ssh -N -L 18789:127.0.0.1:18789 <user>@<host>`
5. Open `http://127.0.0.1:18789/` en authenticeer met het geconfigureerde gedeelde
   geheim (standaard een token; een wachtwoord als `gateway.auth.mode` `"password"` is).

Volledige serverhandleiding: [Linux-server](/nl/vps). Stapsgewijs VPS-voorbeeld:
[exe.dev](/nl/install/exe-dev).

## Node-mogelijkheden

De gebundelde Linux Node-Plugin geeft de CLI de apparaatmogelijkheden van de `openclaw node`-service zonder dat de desktopapp vereist is. Opdrachten worden alleen aan de Gateway aangekondigd wanneer de bijbehorende mogelijkheid is ingeschakeld en het vereiste lokale hulpprogramma aanwezig is.

| Mogelijkheid                              | Standaard | Vereiste                                                           |
| --------------------------------------- | ------- | --------------------------------------------------------------------- |
| Bureaubladmeldingen (`system.notify`) | Aan      | `notify-send` van libnotify en een sessie voor bureaubladmeldingen       |
| Camerafoto's en -clips (`camera.*`)    | Uit     | FFmpeg, toegang tot een V4L2-camera en PulseAudio of PipeWire voor clipaudio |
| Locatie (`location.get`)               | Uit     | GeoClue2 en de bijbehorende `where-am-i`-demo                                    |

Configureer de Plugin in `openclaw.json`:

```json5
{
  plugins: {
    entries: {
      "linux-node": {
        config: {
          notify: { enabled: true },
          camera: { enabled: true },
          location: { enabled: true },
        },
      },
    },
  },
}
```

Herstart de nodeservice nadat je deze instellingen hebt gewijzigd. De beschikbaarheid wordt eenmaal per proces bepaald en de node-aankondiging wordt bij een herstart opnieuw opgebouwd.

De Gateway keurt het opdracht- en mogelijkhedenoppervlak van de node afzonderlijk van de apparaatkoppeling goed. Keur bij de eerste start, of nadat meer mogelijkheden zijn ingeschakeld, het wachtende oppervlak goed:

```bash
openclaw nodes pending
openclaw nodes approve <requestId>
```

Een node kan verbonden en aan een apparaat gekoppeld zijn terwijl de effectieve `caps` en `commands` leeg blijven totdat deze goedkeuring is voltooid.

Camera-apparaten moeten leesbaar zijn voor de servicegebruiker, doorgaans via de groep `video`. Cameraclips gebruiken de standaardbron van PulseAudio of PipeWire wanneer `includeAudio` waar is; microfoonaudio bestaat alleen als die audiotrack van de clip, niet als zelfstandige opdracht. Voor locatie moet de gebruiker van de nodeservice toestemming hebben volgens het GeoClue-beleid van de host.

`camera.snap` en `camera.clip` vereisen ook expliciete activering in de Gateway via `gateway.nodes.commands.allow`. Zie [Camera-opname](/nl/nodes/camera) en [Locatieopdracht](/nl/nodes/location-command) voor payloads, limieten en fouten.

## Installatie

- [Aan de slag](/nl/start/getting-started)
- [Installatie en updates](/nl/install/updating)
- Optioneel: [Bun-pakketworkflow](/nl/install/bun), [Nix](/nl/install/nix), [Docker](/nl/install/docker)

## Gateway-service (systemd)

Installeer met een van de volgende opdrachten:

```bash
openclaw onboard --install-daemon
openclaw gateway install
openclaw configure   # select "Gateway service" when prompted
```

Herstel of migreer een bestaande installatie:

```bash
openclaw doctor
```

`openclaw gateway install` genereert standaard een systemd-eenheid op **gebruikersniveau**. De volledige
servicerichtlijnen, inclusief de variant op **systeemniveau** voor gedeelde of
permanent actieve hosts, staan in het [Gateway-runbook](/nl/gateway#supervision-and-service-lifecycle).

Schrijf alleen handmatig een eenheid voor een aangepaste configuratie. Minimaal voorbeeld van een gebruikerseenheid
(`~/.config/systemd/user/openclaw-gateway[-<profile>].service`):

```ini
[Unit]
Description=OpenClaw Gateway (profile: <profile>, v<version>)
After=network-online.target
Wants=network-online.target
StartLimitBurst=5
StartLimitIntervalSec=60

[Service]
ExecStart=/usr/local/bin/openclaw gateway --port 18789
Restart=always
RestartSec=5
RestartPreventExitStatus=78
TimeoutStopSec=30
TimeoutStartSec=30
SuccessExitStatus=0 143
OOMPolicy=continue
KillMode=control-group

[Install]
WantedBy=default.target
```

Handmatig geschreven eenheden nemen de adaptieve heapgrootte die `openclaw gateway install` voor beheerde Gateway-services schrijft niet over. Geef de voorkeur aan het beheerde installatieprogramma of stel een expliciete heaplimiet in de aangepaste supervisor in nadat rekening is gehouden met ruimte voor systeemeigen geheugen.

Schakel deze in:

```bash
systemctl --user enable --now openclaw-gateway[-<profile>].service
```

## Geheugendruk en beëindiging door OOM

Op Linux kiest de kernel een OOM-slachtoffer wanneer een host, VM of container-cgroup
onvoldoende geheugen heeft. De Gateway is een ongeschikt slachtoffer omdat deze langdurige
sessies en kanaalverbindingen beheert. Daarom stuurt OpenClaw er waar mogelijk op aan dat tijdelijke onderliggende
processen eerst worden beëindigd.

Voor in aanmerking komende onderliggende Linux-processen verpakt OpenClaw de opdracht in een korte
`/bin/sh`-shim die de eigen `oom_score_adj` van het onderliggende proces verhoogt naar `1000` en vervolgens
de echte opdracht `exec`t. Hiervoor zijn geen verhoogde rechten nodig: een proces mag altijd
zijn eigen OOM-score verhogen.

Ondersteunde oppervlakken voor onderliggende processen:

- Door de supervisor beheerde onderliggende opdrachtprocessen
- Onderliggende PTY-shellprocessen
- Onderliggende MCP-stdio-serverprocessen
- Door OpenClaw gestarte browser-/Chrome-processen (via de procesruntime van de Plugin-SDK)

De wrapper is alleen voor Linux en wordt overgeslagen wanneer `/bin/sh` niet beschikbaar is, of wanneer
de omgeving van het onderliggende proces `OPENCLAW_CHILD_OOM_SCORE_ADJ` instelt op `0`, `false`, `no` of
`off`.

Controleer een onderliggend proces:

```bash
cat /proc/<child-pid>/oom_score_adj
```

De verwachte waarde voor ondersteunde onderliggende processen is `1000`; het Gateway-proces zelf
behoudt zijn normale score (doorgaans `0`).

De `OOMPolicy=continue` van de systemd-eenheid houdt de Gateway-service actief wanneer
een tijdelijk onderliggend proces door de OOM-killer wordt geselecteerd, in plaats van de hele
eenheid als mislukt te markeren en alle kanalen opnieuw te starten; het mislukte onderliggende proces of de mislukte sessie meldt
een eigen fout.

Dit vervangt de normale geheugenafstemming niet. Als een VPS of container herhaaldelijk
onderliggende processen beëindigt, verhoog dan de geheugenlimiet, verlaag de gelijktijdigheid of voeg strengere
resourcebeperkingen toe (systemd `MemoryMax=`, containergeheugenlimieten).

## Gerelateerd

- [Installatieoverzicht](/nl/install)
- [Linux-server](/nl/vps)
- [Raspberry Pi](/nl/install/raspberry-pi)
- [Gateway-draaiboek](/nl/gateway)
- [Gateway-configuratie](/nl/gateway/configuration)
