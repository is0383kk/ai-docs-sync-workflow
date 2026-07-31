---
read_when:
    - Je wilt uitvoeringsgoedkeuringen bewerken via de CLI
    - Je moet acceptatielijsten beheren op Gateway- of Node-hosts
    - Je moet een openstaande goedkeuring weergeven of afhandelen zonder chatinterface
summary: CLI-referentie voor `openclaw approvals` en `openclaw exec-policy`
title: Goedkeuringen
x-i18n:
    generated_at: "2026-07-27T04:58:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f8b6f198af718d7b058498dbb960a1eb68ced601e1cd9205070b7199688552d2
    source_path: cli/approvals.md
    workflow: 16
---

# `openclaw approvals`

Beheer exec-goedkeuringen voor de **lokale host**, **Gateway-host** of een **Node-host**. Zonder doelvlag lezen/schrijven opdrachten het lokale goedkeuringsbestand op schijf. Gebruik `--gateway` om de Gateway als doel te kiezen, of `--node <id|name|ip>` om een specifieke Node als doel te kiezen.

Alias: `openclaw exec-approvals`

Gerelateerd: [Exec-goedkeuringen](/nl/tools/exec-approvals), [Nodes](/nl/nodes)

## `openclaw exec-policy`

`openclaw exec-policy` is de handige opdracht die **alleen lokaal** werkt en de aangevraagde `tools.exec.*`-configuratie en het goedkeuringsbestand van de lokale host in één stap gesynchroniseerd houdt:

```bash
openclaw exec-policy show
openclaw exec-policy show --json

openclaw exec-policy preset yolo
openclaw exec-policy preset cautious --json

openclaw exec-policy set --host gateway --security full --ask off --ask-fallback full
```

Voorinstellingen (`yolo`, `cautious`, `deny-all`) passen `host`, `security`, `ask` en `askFallback` samen toe. `set` past alleen de vlaggen toe die je doorgeeft; elke geaccepteerde waarde wordt gevalideerd (`--host auto|sandbox|gateway|node`, `--security deny|allowlist|full`, `--ask off|on-miss|always`, `--ask-fallback deny|allowlist|full`).

Bereik:

- Werkt het lokale configuratiebestand en lokale goedkeuringsbestand samen bij; stuurt geen beleid naar de Gateway of een Node-host.
- `--host node` wordt geweigerd: exec-goedkeuringen voor Nodes worden tijdens runtime bij de Node opgehaald, waardoor lokale `exec-policy` deze niet kan synchroniseren. Gebruik in plaats daarvan `openclaw approvals set --node <id|name|ip>`.
- `exec-policy show` markeert `host=node`-bereiken tijdens runtime als beheerd door de Node, in plaats van een effectief beleid af te leiden uit het lokale goedkeuringsbestand.

Gebruik voor goedkeuringen op externe hosts rechtstreeks `openclaw approvals set --gateway` of `openclaw approvals set --node <id|name|ip>`.

## Veelgebruikte opdrachten

```bash
openclaw approvals get
openclaw approvals get --node <id|name|ip>
openclaw approvals get --gateway
openclaw approvals pending
openclaw approvals resolve <id> <allow-once|allow-always|deny>
```

`get` toont het effectieve exec-beleid voor het doel: het aangevraagde `tools.exec`-beleid, het beleid uit het goedkeuringsbestand van de host en het samengevoegde effectieve resultaat. Nodes met een hosteigen beleid, zoals de Windows-companion, tonen dat beleid rechtstreeks in plaats van de beleidsberekening van het OpenClaw-goedkeuringsbestand toe te passen.

Voor Nodes die bestanden als opslag gebruiken, vereist de samengevoegde weergave een door de host bepaalde momentopname van het beleid. Oudere Nodes tonen het effectieve beleid als niet beschikbaar, in plaats van aan te nemen dat het aangevraagde beleid van de Gateway ook op de host van toepassing is.

<Note>
`/exec`-overschrijvingen per sessie zijn niet inbegrepen. Voer `/exec` uit in de relevante sessie om de huidige standaardwaarden ervan te bekijken.
</Note>

Prioriteit:

- Het goedkeuringsbestand van de host is de afdwingbare bron van waarheid.
- Het aangevraagde `tools.exec`-beleid kan de intentie beperken of verruimen, maar het effectieve resultaat wordt afgeleid van de hostregels.
- `--node` combineert het goedkeuringsbestand van de Node-host met het `tools.exec`-beleid van de Gateway (beide zijn tijdens runtime van toepassing).
- Als de Gateway-configuratie niet beschikbaar is, valt de CLI terug op de momentopname van de Node-goedkeuringen en vermeldt deze dat het uiteindelijke runtimebeleid niet kon worden berekend.

## Openstaande goedkeuringen

Geef openstaande exec-, Plugin- en OpenClaw-systeemagentgoedkeuringen van de Gateway weer:

```bash
openclaw approvals pending
openclaw approvals pending --json
```

Volledige inventarisatie en de bijbehorende operatorbrede `resolve`-flow gebruiken `operator.admin`, omdat goedkeuringsrecords anders de filtering op aanvrager/beoordelaar behouden. Voor afhandeling wordt ook het specifieke `operator.approvals`-bereik aangevraagd. De standaard operatortoekenning van de CLI omvat beide bereiken; een beperkte client van derden hoort geen beheerderstoegang aan te vragen alleen om deze opdracht na te bootsen.

De voor mensen leesbare uitvoer toont het soort goedkeuring, de toewijzing aan agent/sessie, de ouderdom van de aanvraag, de tijd tot het verlopen, een ingekorte opdracht of samenvatting en een shellneutraal `id64_<base64url>`-id-token. Na de compacte tabel volgt altijd een `Full request text`-blok met elk volledige token en een verliesvrij geëscapete aanvraag, zodat inkorting vanwege de terminalbreedte geen achtervoegsel of het voor afhandeling benodigde token kan verbergen. Kopieer het volledige token naar `resolve`. Onveilige terminaltekens in andere velden worden weergegeven als zichtbare Unicode-escapes. JSON-uitvoer retourneert genormaliseerde vermeldingen onder `approvals`, waarbij de oorspronkelijke onbewerkte `id`, `summary`, `createdAtMs` en `expiresAtMs` voor scripts behouden blijven; onbewerkte id's blijven geaccepteerd door `resolve`, tenzij ze het gereserveerde `id64_`-voorvoegsel voor weergavetokens gebruiken.

Als een opgegeven `id64_`-waarde zowel overeenkomt met een letterlijke onbewerkte id als met het gedecodeerde weergavetoken van een andere goedkeuring, weigert de CLI deze als dubbelzinnig om te voorkomen dat de verkeerde aanvraag wordt afgehandeld.

Handel één goedkeuring af aan de hand van de volledige id:

```bash
openclaw approvals resolve <id> allow-once
openclaw approvals resolve <id> allow-always
openclaw approvals resolve <id> deny --reason "Niet verwacht tijdens onderhoud"
```

De CLI leest het uniforme goedkeuringsrecord om het soort te bepalen, controleert de aangevraagde beslissing aan de hand van de voor het record toegestane beslissingen en roept vervolgens de uniforme afhandelaar aan. Een eerste geslaagde beslissing sluit af met `0`. Het herhalen van de vastgelegde beslissing sluit ook af met `0` en meldt `already resolved (same decision)`. Bij een conflicterende beslissing, ontbrekende goedkeuring, verlopen goedkeuring of een beslissing die niet beschikbaar is voor dat soort goedkeuring, wordt een duidelijke fout weergegeven en sluit de CLI af met een niet-nulstatus.

`--reason` voegt een lokale notitie toe aan de bevestiging van de CLI. Het huidige goedkeuringsrecord van de Gateway heeft geen vrij tekstveld voor de reden van afhandeling, dus deze notitie wordt niet opgeslagen of naar andere goedkeuringsinterfaces verzonden.

## Goedkeuringen vervangen vanuit een bestand

```bash
openclaw approvals set --file ./exec-approvals.json
openclaw approvals set --stdin <<'EOF'
{ version: 1, defaults: { security: "full", ask: "off", askFallback: "full" } }
EOF
openclaw approvals set --node <id|name|ip> --file ./exec-approvals.json
openclaw approvals set --gateway --file ./exec-approvals.json
```

`set` accepteert JSON5, niet alleen strikte JSON. Gebruik `--file` of `--stdin`, niet beide.

Hosteigen Windows-Nodes gebruiken hun eigen beleidsvorm:

```bash
openclaw approvals set --node <id|name|ip> --stdin <<'EOF'
{
  defaultAction: "deny",
  rules: [{ pattern: "hostname", action: "allow" }]
}
EOF
```

De CLI leest eerst de huidige hash van de Node en verzendt deze met de update, zodat gelijktijdige lokale bewerkingen worden geweigerd in plaats van overschreven. `rules` is vereist omdat deze bewerking de volledige regellijst van de Node vervangt; `defaultAction` is optioneel. Een Node die meldt dat het eigen beleid is uitgeschakeld, kan niet extern worden geconfigureerd; schakel het beleid eerst op die host in of configureer het daar. Hosteigen beleid ondersteunt de `allowlist add|remove`-hulpfuncties niet.

## Voorbeeld voor 'Nooit vragen' / YOLO

Stel voor een host die nooit moet stoppen voor exec-goedkeuringen de standaardwaarden voor hostgoedkeuringen in op `full` + `off`:

```bash
openclaw approvals set --stdin <<'EOF'
{
  version: 1,
  defaults: {
    security: "full",
    ask: "off",
    askFallback: "full"
  }
}
EOF
```

Gebruik voor Nodes die een OpenClaw-goedkeuringsbestand beschikbaar stellen dezelfde inhoud met `openclaw approvals set --node <id|name|ip> --stdin`. Hosteigen Nodes vereisen de hierboven getoonde eigenaarspecifieke vorm.

Dit wijzigt alleen het **goedkeuringsbestand van de host**. Stel ook het volgende in om het aangevraagde OpenClaw-beleid daarmee in overeenstemming te houden:

```bash
openclaw config set tools.exec.host gateway
openclaw config set tools.exec.mode full
```

`tools.exec.host=gateway` wordt hier expliciet vermeld omdat `host=auto` nog steeds 'sandbox indien beschikbaar, anders Gateway' betekent: YOLO gaat over goedkeuringen, niet over routering. Gebruik `gateway` (of `/exec host=gateway`) wanneer je exec op de host wilt, zelfs als een sandbox is geconfigureerd.

Een weggelaten `askFallback` wordt standaard ingesteld op `deny`. Stel `askFallback: "full"` expliciet in bij het upgraden van een host zonder UI die het gedrag zonder vragen moet behouden.

Lokale snelkoppeling voor dezelfde intentie, alleen op de lokale machine:

```bash
openclaw exec-policy preset yolo
```

## Hulpfuncties voor de toelatingslijst

```bash
openclaw approvals allowlist add "~/Projects/**/bin/rg"
openclaw approvals allowlist add --agent main --node <id|name|ip> "/usr/bin/uptime"
openclaw approvals allowlist add --agent "*" "/usr/bin/uname"

openclaw approvals allowlist remove "~/Projects/**/bin/rg"
```

## Algemene opties

`get`, `set` en `allowlist add|remove` ondersteunen allemaal:

- `--node <id|name|ip>` (herleidt id, naam, IP of id-voorvoegsel; dezelfde herleider als `openclaw nodes`)
- `--gateway`
- gedeelde opties voor Node-RPC: `--url`, `--token`, `--timeout`, `--json`

Geen doelvlag betekent het lokale goedkeuringsbestand op schijf.

`allowlist add|remove` ondersteunt ook `--agent <id>` (standaard `"*"`, toegepast op alle agents).

`pending` en `resolve` gebruiken altijd de Gateway, omdat openstaande aanvragen live Gateway-status zijn. Ze ondersteunen de gedeelde Gateway-verbindingsopties `--url`, `--token` en `--timeout`; `pending` ondersteunt ook `--json`.

## Opmerkingen

- De Node-host moet `system.execApprovals.get/set` adverteren (macOS-app, headless Node-host of Windows-companion).
- Goedkeuringsbestanden worden per host opgeslagen in de OpenClaw-statusmap: `$OPENCLAW_STATE_DIR/exec-approvals.json`, of `~/.openclaw/exec-approvals.json` wanneer de variabele niet is ingesteld.

## Gerelateerd

- [CLI-referentie](/nl/cli)
- [Exec-goedkeuringen](/nl/tools/exec-approvals)
