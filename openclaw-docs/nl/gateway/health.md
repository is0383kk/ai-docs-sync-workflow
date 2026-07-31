---
read_when:
    - Problemen met kanaalconnectiviteit of de status van de Gateway vaststellen
    - Inzicht in CLI-opdrachten en -opties voor statuscontroles
summary: Healthcheckopdrachten en bewaking van de Gateway-status
title: Statuscontroles
x-i18n:
    generated_at: "2026-07-27T05:33:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 59a7fbfb7fb86be7dbd3a03f96c7328c2bc8cc851230c0bdd1b1b750b3014be4
    source_path: gateway/health.md
    workflow: 16
---

Korte handleiding om kanaalconnectiviteit te verifiëren zonder te gokken.

## Snelle controles

- `openclaw status` - lokale samenvatting: bereikbaarheid/modus van de Gateway, updatehint, ouderdom van gekoppelde kanaalauthenticatie, sessies + recente activiteit.
- `openclaw status --all` - volledige lokale diagnose (alleen-lezen, met kleur, veilig om te plakken voor foutopsporing).
- `openclaw status --deep` - vraagt de actieve Gateway om een live-probe (`health` met `probe:true`), inclusief kanaalprobes per account indien ondersteund.
- `openclaw status --usage` - toont momentopnamen van gebruik/quota van modelproviders.
- `openclaw health` - vraagt de actieve Gateway om de momentopname van de status (alleen WS; geen rechtstreekse kanaalsockets vanuit de CLI).
- `openclaw health --verbose` (alias `--debug`) - dwingt een live-statusprobe af en toont verbindingsgegevens van de Gateway.
- `openclaw health --json` - machineleesbare uitvoer van de statusmomentopname.
- Stuur `/status` als een zelfstandige chatopdracht in een willekeurig kanaal om een statusantwoord te krijgen zonder de agent aan te roepen.
- Logboeken: voer `openclaw logs --follow` (of `openclaw --profile <profile> logs --follow`) uit en filter op `web-heartbeat`, `web-reconnect`, `web-auto-reply`, `web-inbound`.

Voor Discord en andere chatproviders geven sessierijen niet aan of een socket actief is.
`openclaw sessions`, Gateway `sessions.list` en de agenttool `sessions_list`
lezen opgeslagen gespreksstatus. Een provider kan opnieuw verbinding maken en een
gezonde kanaalstatus tonen voordat er een nieuwe sessierij is aangemaakt. Gebruik de
bovenstaande opdrachten voor kanaalstatus en statuscontroles voor live-connectiviteitscontroles.

## Uitgebreide diagnostiek

- Aanmeldgegevens op schijf: `ls -l ~/.openclaw/credentials/whatsapp/<accountId>/creds.json` (mtime moet recent zijn).
- Sessieopslag: `ls -l ~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`. Het aantal en recente ontvangers worden weergegeven via `status`.
- Opnieuw koppelen: `openclaw channels logout && openclaw channels login --verbose` wanneer statuscodes 409-515 of `loggedOut` in logboeken verschijnen. De QR-aanmeldingsflow start na het koppelen eenmaal automatisch opnieuw bij status 515.
- Diagnostiek is standaard ingeschakeld (`diagnostics.enabled: false` schakelt dit uit). Geheugengebeurtenissen registreren RSS-/heap-aantallen in bytes en druk door drempelwaarden/groei. Waarschuwingen over activiteit registreren event-loopvertraging/-benutting, de verhouding tot het aantal CPU-kernen en aantallen actieve/wachtende/wachtrijsessies wanneer het proces actief maar verzadigd is. Gebeurtenissen voor te grote payloads registreren wat is geweigerd/afgekapt/opgesplitst plus grootten en limieten, maar nooit berichttekst, inhoud van bijlagen, Webhook-bodies, onbewerkte request-/response-bodies, tokens, cookies of geheime waarden.
- Dezelfde Heartbeat stuurt de begrensde stabiliteitsregistratie aan: `openclaw gateway stability` (of de `diagnostics.stability` Gateway-RPC). Fatale afsluitingen van de Gateway, time-outs bij het afsluiten en opstartfouten bij opnieuw starten slaan de nieuwste momentopname op onder `~/.openclaw/logs/stability/`. Bekijk de nieuwste bundel met `openclaw gateway stability --bundle latest`.
- Voer voor bugrapporten `openclaw gateway diagnostics export` uit en voeg het gegenereerde zipbestand toe: een Markdown-samenvatting, de nieuwste stabiliteitsbundel, opgeschoonde logboekmetadata, opgeschoonde momentopnamen van Gateway-status/-gezondheid en de configuratiestructuur. Chattekst, Webhook-bodies, tooluitvoer, aanmeldgegevens, cookies, account-/bericht-ID's en geheime waarden worden weggelaten of geredigeerd. Zie [Diagnostiek exporteren](/nl/gateway/diagnostics).

## Configuratie van de statusmonitor

- `channels.<provider>.healthMonitor.enabled`: schakelt herstarts door de statusmonitor uit voor een specifiek kanaal terwijl globale monitoring ingeschakeld blijft.
- `channels.<provider>.accounts.<accountId>.healthMonitor.enabled`: overschrijving voor meerdere accounts die voorrang heeft op de instelling op kanaalniveau.
- Deze overschrijvingen per kanaal gelden voor de ingebouwde kanalen die ze momenteel aanbieden: Discord, Google Chat, iMessage, IRC, Microsoft Teams, Signal, Slack, Telegram en WhatsApp.

## Uptimebewaking

Externe diensten voor uptimebewaking moeten het speciale `/health`-eindpunt gebruiken, niet `/v1/chat/completions`.

- **WEL gebruiken:** `GET /health` - onmiddellijk antwoord, geen sessie aangemaakt, geen LLM-aanroep, retourneert `{"ok":true,"status":"live"}`
- **NIET gebruiken:** `/v1/chat/completions` voor statuscontroles - elk verzoek maakt een volledige agentsessie met een Skills-momentopname, contextopbouw en LLM-aanroepen

Wanneer geen `x-openclaw-session-key`-header of `user`-veld is opgegeven, genereert `/v1/chat/completions` voor elk verzoek een nieuwe willekeurige sessie. Monitoringdiensten die elke 15 minuten pingen, maken ~96 sessies/dag aan, die elk 4-22KB verbruiken. Na verloop van tijd veroorzaakt dit een opgeblazen sessieopslag en kan het tot overschrijding van het contextvenster leiden.

### Voorbeelden voor het instellen van monitoringdiensten

- **BetterStack:** Stel de URL voor de statuscontrole in op `https://<your-gateway-host>:<port>/health`
- **UptimeRobot:** Voeg een nieuwe HTTP-monitor toe met URL `https://<your-gateway-host>:<port>/health`
- **Algemeen:** Elke HTTP GET naar `/health` retourneert 200 met `{"ok":true}` wanneer de Gateway gezond is

## Wanneer iets mislukt

- `logged out` of status 409-515 -> koppel opnieuw met `openclaw channels logout` en daarna `openclaw channels login`.
- Gateway onbereikbaar -> start deze: `openclaw gateway --port 18789` (gebruik `--force` als de poort bezet is).
- Geen inkomende berichten -> bevestig dat de gekoppelde telefoon online is en de afzender is toegestaan (`channels.whatsapp.allowFrom`); zorg er bij groepschats voor dat de toelatingslijst + vermeldingsregels overeenkomen (`channels.whatsapp.groups`, `agents.entries.*.groupChat.mentionPatterns`).

## Speciale opdracht "health"

`openclaw health` vraagt de actieve Gateway om de momentopname van de status (geen rechtstreekse
kanaalsockets vanuit de CLI). Standaard retourneert deze een recente, in de cache opgeslagen
Gateway-momentopname en vernieuwt de Gateway die cache op de achtergrond; `--verbose` dwingt
in plaats daarvan een live-probe af. De opdracht rapporteert indien beschikbaar de ouderdom van
gekoppelde aanmeldgegevens/authenticatie, probesamenvattingen per kanaal, een samenvatting van de
sessieopslag en de duur van de probe. De opdracht sluit af met een niet-nulcode als de Gateway
onbereikbaar is of de probe mislukt/een time-out bereikt.

Opties:

- `--json`: machineleesbare JSON-uitvoer
- `--timeout <ms>`: overschrijft de standaardtime-out van 10s voor de probe
- `--verbose`: dwingt een live-probe af en toont verbindingsgegevens van de Gateway
- `--debug`: alias voor `--verbose`

De statusmomentopname bevat: `ok` (booleaans), `ts` (tijdstempel), `durationMs` (probetijd), status per kanaal, beschikbaarheid van de agent en een samenvatting van de sessieopslag.

## Gerelateerd

- [Gateway-draaiboek](/nl/gateway)
- [Diagnostiek exporteren](/nl/gateway/diagnostics)
- [Problemen met de Gateway oplossen](/nl/gateway/troubleshooting)
