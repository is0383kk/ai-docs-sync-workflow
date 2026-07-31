---
read_when:
    - Externe Mac-bediening instellen of problemen ermee oplossen
summary: macOS-appflow voor het bedienen van een externe OpenClaw-Gateway
title: Afstandsbediening
x-i18n:
    generated_at: "2026-07-27T06:22:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7e558c39fa173a77bf11270a8961c14c6e2350dfc4f458da3633532513b98bf6
    source_path: platforms/mac/remote.md
    workflow: 16
---

Deze flow laat de macOS-app fungeren als volledige afstandsbediening voor een OpenClaw-Gateway die op een andere host (desktop/server) draait. De app maakt rechtstreeks verbinding met vertrouwde Gateway-URL's op het LAN/de Tailnet, of beheert een SSH-tunnel wanneer de externe Gateway alleen via loopback bereikbaar is. Statuscontroles, het doorsturen van Voice Wake en Web Chat gebruiken dezelfde externe configuratie uit _Settings -> General_.

## Modi

- **Lokaal (deze Mac)**: alles draait op de laptop; SSH wordt niet gebruikt.
- **Extern via SSH (standaard)**: OpenClaw-opdrachten worden op de externe host uitgevoerd. De app opent een SSH-verbinding met `-o BatchMode`, de door jou gekozen identiteit/sleutel en een lokale poortdoorsturing.
- **Rechtstreeks extern (ws/wss)**: geen SSH-tunnel; de app maakt rechtstreeks verbinding met de Gateway-URL (LAN, Tailscale, Tailscale Serve of een openbare HTTPS-reverseproxy).

## Externe transportmethoden

- **SSH-tunnel** (standaard): gebruikt `ssh -N -L ...` om de Gateway-poort door te sturen naar localhost. De Gateway ziet het IP-adres van de Node als `127.0.0.1`, omdat de tunnel via loopback loopt.
- **Rechtstreeks (ws/wss)**: maakt rechtstreeks verbinding met de Gateway-URL. De Gateway ziet het werkelijke IP-adres van de client.

De app schakelt multiplexing van SSH-verbindingen en het na authenticatie naar de achtergrond verplaatsen uit voor zijn eigen SSH-processen, zodat het exacte proces kan worden bewaakt en opnieuw gestart, zelfs als de geselecteerde alias `ControlMaster` of `ForkAfterAuthentication` inschakelt.

Verificatie van SSH-hostsleutels is standaard strikt, omdat Gateway-referenties door deze tunnel gaan. Als je het eigen vertrouwensgedrag van een beheerde SSH-alias wilt gebruiken, stel je `--ssh-host-key-policy openssh` in via `openclaw-mac configure-remote`, of stel je `gateway.remote.sshHostKeyPolicy` rechtstreeks in op `"openssh"`. Controleer de alias en eventuele overeenkomende `Host *`- of systeemconfiguratie voordat je hiervoor kiest. Als je het SSH-doel wijzigt (in de app of via `configure-remote`), wordt het beleid teruggezet naar `strict`, tenzij je dit opnieuw expliciet inschakelt voor het nieuwe doel.

In de SSH-tunnelmodus worden gevonden LAN-/Tailnet-hostnamen opgeslagen als `gateway.remote.sshTarget`. De app houdt `gateway.remote.url` op het lokale tunneleindpunt (bijvoorbeeld `ws://127.0.0.1:18789`), zodat de CLI, Web Chat en de lokale Node-hostservice allemaal hetzelfde loopbacktransport gebruiken. Wanneer de detectie zowel onbewerkte Tailnet-IP-adressen als stabiele hostnamen oplevert, geeft de app de voorkeur aan Tailscale MagicDNS- of LAN-namen, zodat verbindingen beter bestand zijn tegen adreswijzigingen. Als de lokale tunnelpoort verschilt van de externe Gateway-poort, stel je `gateway.remote.remotePort` in op de poort van de externe host.

Browserautomatisering in de externe modus wordt beheerd door de CLI-Node-host, niet door de Node van de native macOS-app. De app start waar mogelijk de geïnstalleerde Node-hostservice. Om browserbesturing vanaf die Mac in te schakelen, installeer/start je deze met `openclaw node install ...` en `openclaw node start` (of voer je `openclaw node run ...` op de voorgrond uit) en selecteer je vervolgens die browsergeschikte Node als doel.

## Vereisten op de externe host

1. Installeer Node + pnpm en bouw/installeer de OpenClaw-CLI (`pnpm install && pnpm build && pnpm link --global`).
2. Zorg dat `openclaw` op PATH staat voor niet-interactieve shells (maak indien nodig een symlink in `/usr/local/bin` of `/opt/homebrew/bin`).
3. Voor SSH-transport: stel SSH-authenticatie op basis van sleutels in. Tailscale-IP-adressen worden aanbevolen voor stabiele bereikbaarheid buiten het LAN.

## De macOS-app instellen

Om de app zonder de welkomstflow vooraf te configureren, via SSH:

```bash
openclaw-mac configure-remote \
  --ssh-target user@gateway-host \
  --local-port 18789 \
  --remote-port 18789 \
  --token "$OPENCLAW_GATEWAY_TOKEN"
```

Of sla SSH volledig over voor een Gateway die al bereikbaar is op een vertrouwd LAN of vertrouwde Tailnet:

```bash
openclaw-mac configure-remote \
  --direct-url ws://192.168.0.202:18789 \
  --token "$OPENCLAW_GATEWAY_TOKEN"
```

`openclaw-mac connect`, `wizard` en `configure-remote` bepalen de actieve configuratie in deze volgorde: `OPENCLAW_CONFIG_PATH`, vervolgens `$OPENCLAW_STATE_DIR/openclaw.json` en daarna `~/.openclaw/openclaw.json`. Beide configuratievormen schrijven naar dat actieve bestand, markeren de onboarding als voltooid en laten de app bij de volgende start het geselecteerde transport beheren. `--local-port`/`--remote-port` zijn standaard ingesteld op `18789`. Andere vlaggen: `--password`, `--identity <path>`, `--ssh-host-key-policy <strict|openssh>`, `--project-root <path>`, `--cli-path <path>`, `--json`. Voer `openclaw-mac configure-remote --help` uit voor de volledige referentie.

Om dit in plaats daarvan via de gebruikersinterface te configureren:

1. Open _Settings -> General_.
2. Kies onder **OpenClaw runs** de optie **Remote** en stel het volgende in:
   - **Transport**: **SSH tunnel** of **Direct (ws/wss)**.
   - **SSH target**: `user@host` (optioneel `:port`). Als de Gateway zich op hetzelfde LAN bevindt en via Bonjour wordt aangekondigd, selecteer je deze in de lijst met gevonden apparaten om dit veld automatisch in te vullen.
   - **Gateway URL** (alleen rechtstreeks): `wss://gateway.example.ts.net` (of `ws://...` voor lokaal/LAN).
   - **Identity file** (geavanceerd): pad naar je sleutel.
   - **Project root** (geavanceerd): pad naar de externe checkout dat voor opdrachten wordt gebruikt.
   - **CLI path** (geavanceerd): optioneel pad naar een uitvoerbaar `openclaw`-startpunt/binair bestand (automatisch ingevuld wanneer dit wordt aangekondigd).
3. Klik op **Test remote**. Succes betekent dat de externe `openclaw status --json` correct is uitgevoerd. Fouten wijzen meestal op problemen met PATH/de CLI; afsluitcode 127 betekent dat de CLI niet op de externe host is gevonden.
4. Statuscontroles en Web Chat worden nu automatisch via het geselecteerde transport uitgevoerd.

## Web Chat

- **SSH-tunnel**: maakt verbinding met de Gateway via de doorgestuurde WebSocket-besturingspoort (standaard 18789).
- **Rechtstreeks (ws/wss)**: maakt rechtstreeks verbinding met de geconfigureerde Gateway-URL.
- Er is geen afzonderlijke HTTP-server voor Web Chat.

## Machtigingen

- De externe host heeft dezelfde TCC-goedkeuringen nodig als lokaal (Automation, Accessibility, Screen Recording, Microphone, Speech Recognition, Notifications). Doorloop de onboarding eenmaal op die machine om deze toe te kennen.
- Nodes maken de status van hun machtigingen bekend via `node.list` / `node.describe`, zodat agents weten wat beschikbaar is.

## Beveiligingsopmerkingen

- Geef de voorkeur aan loopbackbindingen op de externe host en maak verbinding via SSH, Tailscale Serve of een vertrouwde rechtstreekse Tailnet-/LAN-URL.
- Voor SSH-tunneling is standaard een reeds vertrouwde hostsleutel vereist. Vertrouw eerst de hostsleutel (voeg deze toe aan het geconfigureerde known-hosts-bestand), of stel `gateway.remote.sshHostKeyPolicy: "openssh"` expliciet in voor een beheerde alias waarvan je het OpenSSH-vertrouwensbeleid accepteert.
- Als je de Gateway aan een niet-loopbackinterface bindt, moet je geldige Gateway-authenticatie vereisen: een token, wachtwoord of identiteitsbewuste reverseproxy met `gateway.auth.mode: "trusted-proxy"`.
- Rechtstreekse `wss://`-verbindingen passen één certificaatbeleid toe op zowel operator-/besturingsverkeer als de gekoppelde Mac-Node. Stel `gateway.remote.tlsFingerprint` in voor een expliciete pin. Zonder deze instelling registreert de app pas bij het eerste gebruik een pin nadat de normale vertrouwenscontrole van macOS is geslaagd.
- Zie [Beveiliging](/nl/gateway/security) en [Tailscale](/nl/gateway/tailscale).

## WhatsApp-inlogflow (extern)

- Voer `openclaw channels login --channel whatsapp --verbose` **op de externe host** uit. Scan de QR-code met WhatsApp op je telefoon.
- Voer de aanmelding opnieuw uit op die host als de authenticatie verloopt. De statuscontrole maakt koppelingsproblemen zichtbaar.

## Probleemoplossing

| Symptoom                                          | Oorzaak / oplossing                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| ------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `exit 127` / niet gevonden                           | `openclaw` staat niet in PATH voor niet-login-shells. Voeg het toe aan `/etc/paths`, het rc-bestand van je shell, of maak een symlink in `/usr/local/bin`/`/opt/homebrew/bin`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Statuscontrole mislukt                              | Controleer de SSH-bereikbaarheid, PATH en of Baileys (WhatsApp) is aangemeld (`openclaw status --json`).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| Webchat blijft hangen                                   | Controleer of de Gateway op de externe host draait en of de doorgestuurde poort overeenkomt met de WS-poort van de Gateway; de gebruikersinterface vereist een werkende WS-verbinding.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| Node-IP toont `127.0.0.1`                        | Dit is te verwachten met de SSH-tunnel. Zet **Transport** op **Direct (ws/wss)** als je wilt dat de Gateway het echte IP-adres van de client ziet.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Dashboard werkt, maar Mac-mogelijkheden zijn offline | De operator-/besturingsverbinding werkt, maar de verbinding met de begeleidende Node is niet actief of het bijbehorende commando-oppervlak ontbreekt. Open het apparaatgedeelte in de menubalk en controleer of de Mac `paired · disconnected` is. Directe `wss://`-verbindingen voor de operator en Node gebruiken hetzelfde geconfigureerde of opgeslagen certificaatbeleid. Voor vertrouwde `wss://*.ts.net` Tailscale Serve-eindpunten worden verouderde opgeslagen leaf-pins na certificaatrotatie vervangen en wordt de verbinding automatisch opnieuw geprobeerd. Geconfigureerde pins worden nooit automatisch geroteerd; werk `gateway.remote.tlsFingerprint` bij nadat je het nieuwe certificaat hebt gecontroleerd, of schakel over naar **Remote over SSH**. |
| Voice Wake                                       | Activeringszinnen worden in de externe modus automatisch doorgestuurd; er is geen afzonderlijke doorstuurservice nodig.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |

## Meldingsgeluiden

Kies per melding geluiden uit scripts met `openclaw nodes notify`, bijvoorbeeld:

```bash
openclaw nodes notify --node <id> --title "Ping" --body "Externe Gateway gereed" --sound Glass
```

De app heeft geen algemene schakelaar voor een standaardgeluid; aanroepers kiezen per verzoek een geluid (of geen geluid).

## Gerelateerd

- [macOS-app](/nl/platforms/macos)
- [Externe toegang](/nl/gateway/remote)
