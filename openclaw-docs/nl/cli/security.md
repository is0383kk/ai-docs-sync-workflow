---
read_when:
    - Je wilt een snelle beveiligingsaudit uitvoeren op configuratie/status
    - Je wilt veilige suggesties voor oplossingen toepassen (machtigingen, standaardinstellingen aanscherpen)
summary: CLI-referentie voor `openclaw security` (veelvoorkomende beveiligingsvalkuilen controleren en oplossen)
title: Beveiliging
x-i18n:
    generated_at: "2026-07-27T04:55:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6b5f9ea5cb746bfd29ff4d096062e81595abe99a883fc3b1113b45a3527d42d9
    source_path: cli/security.md
    workflow: 16
---

# `openclaw security`

Beveiligingstools: audit plus optionele veilige oplossingen. Gerelateerd: [Beveiliging](/nl/gateway/security).

```bash
openclaw security audit
openclaw security audit --deep
openclaw security audit --deep --password <password>
openclaw security audit --deep --token <token>
openclaw security audit --auth password --password <password>
openclaw security audit --fix
openclaw security audit --json
```

## Auditmodi

Gewone `security audit` blijft op het koude pad voor configuratie/bestandssysteem/alleen-lezen: hiermee worden geen beveiligingscollectors van de Plugin-runtime gedetecteerd, zodat routinematige audits niet elke geïnstalleerde Plugin-runtime laden. `--deep` voegt naar beste vermogen live Gateway-controles en beveiligingsauditcollectors van Plugins toe (expliciete interne aanroepers kunnen die collectors ook gebruiken wanneer ze al een geschikt runtimebereik hebben).

Als Gateway-wachtwoordauthenticatie alleen bij het opstarten wordt opgegeven, geef je dezelfde waarde door met `--auth password --password <password>`, zodat de audit deze kan vergelijken met `hooks.token`.

## Wat wordt gecontroleerd

**DM-/vertrouwensmodel**

- Waarschuwt wanneer meerdere DM-afzenders de hoofdsessie delen en beveelt een veilige DM-modus aan: `session.dmScope="per-channel-peer"` (of `per-account-channel-peer` voor kanalen met meerdere accounts) voor gedeelde postvakken. Dit is beveiliging voor samenwerking/gedeelde postvakken, geen isolatie voor operators die elkaar niet vertrouwen; scheid vertrouwensgrenzen met afzonderlijke gateways (of afzonderlijke OS-gebruikers/hosts).
- Genereert `security.trust_model.multi_user_heuristic` wanneer de configuratie wijst op waarschijnlijke toegang door meerdere gebruikers (bijvoorbeeld een open DM-/groepsbeleid, geconfigureerde groepsdoelen of wildcardregels voor afzenders) — het standaardvertrouwensmodel van OpenClaw is een persoonlijke assistent (één operator), geen vijandige isolatie tussen meerdere tenants. Voor opzettelijke configuraties met meerdere gebruikers: voer alle sessies uit in een sandbox, beperk bestandssysteemtoegang tot de werkruimte en houd persoonlijke/privé-identiteiten of referenties buiten die runtime.
- Waarschuwt wanneer kleine modellen (`<=300B` parameters) zonder sandboxing en met ingeschakelde web-/browsertools worden gebruikt.

**Webhook/hooks**

Bij het opstarten wordt een niet-fatale beveiligingswaarschuwing gelogd en de audit markeert `hooks.token` hergebruik van actieve waarden voor Gateway-authenticatie met een gedeeld geheim (`gateway.auth.token` / `OPENCLAW_GATEWAY_TOKEN`, `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD`). Waarschuwt ook wanneer:

- `hooks.token` kort is
- `hooks.path="/"`
- `hooks.defaultSessionKey` niet is ingesteld
- `hooks.allowedAgentIds` onbeperkt is
- overschrijvingen van `sessionKey` voor aanvragen zijn ingeschakeld
- overschrijvingen zijn ingeschakeld zonder `hooks.allowedSessionKeyPrefixes`

Voer `openclaw doctor --fix` uit om een permanent opgeslagen, hergebruikte `hooks.token` te roteren en werk vervolgens externe hook-afzenders bij om het nieuwe token te gebruiken.

**Sandbox/tools**

- Waarschuwt wanneer Docker-instellingen voor de sandbox zijn geconfigureerd terwijl de sandboxmodus is uitgeschakeld.
- Waarschuwt wanneer `gateway.nodes.commands.deny` ineffectieve patroonachtige/onbekende vermeldingen gebruikt (overeenkomsten gelden uitsluitend voor de exacte naam van een Node-opdracht, niet voor het filteren van shelltekst).
- Waarschuwt wanneer `gateway.nodes.commands.allow` expliciet gevaarlijke Node-opdrachten inschakelt.
- Waarschuwt wanneer de algemene `tools.profile="minimal"` wordt overschreven door toolprofielen van agents.
- Waarschuwt wanneer schrijf-/bewerkingstools zijn uitgeschakeld, maar `exec` nog steeds beschikbaar is zonder een beperkende bestandssysteemgrens van de sandbox.
- Waarschuwt wanneer open DM's of groepen runtime-/bestandssysteemtools beschikbaar stellen zonder beveiliging door een sandbox/werkruimte.
- Waarschuwt wanneer tools van geïnstalleerde Plugins mogelijk toegankelijk zijn onder een permissief toolbeleid.

**Sandboxbrowser**

- Waarschuwt wanneer de sandboxbrowser het Docker-netwerk `bridge` gebruikt zonder `sandbox.browser.cdpSourceRange`.
- Markeert gevaarlijke netwerkmodi voor Docker-sandboxes, waaronder deelname aan naamruimten met `host` en `container:*`.
- Waarschuwt wanneer bestaande Docker-containers van de sandboxbrowser ontbrekende/verouderde hashlabels hebben (bijvoorbeeld containers van vóór de migratie zonder `openclaw.browserConfigEpoch`) en beveelt `openclaw sandbox recreate --browser --all` aan.

**Netwerk/detectie**

- Markeert `gateway.allowRealIpFallback=true` (risico op headervervalsing als proxy's verkeerd zijn geconfigureerd).
- Markeert `discovery.mdns.mode="full"` (lekken van metadata via mDNS TXT-records).
- Waarschuwt wanneer `gateway.auth.mode="none"` Gateway-HTTP-API's bereikbaar laat zonder gedeeld geheim (`/tools/invoke` plus elk ingeschakeld `/v1/*`-eindpunt).

**Plugins/kanalen**

- Waarschuwt wanneer op npm gebaseerde installatiegegevens van Plugins/hooks niet aan een versie zijn vastgezet, integriteitsmetadata missen of afwijken van de momenteel geïnstalleerde pakketversies.
- Waarschuwt wanneer allowlists van kanalen afhankelijk zijn van veranderlijke namen/e-mailadressen/tags in plaats van stabiele ID's (Discord, Slack, Google Chat, Microsoft Teams, Mattermost en IRC-bereiken waar van toepassing).

Instellingen met het voorvoegsel `dangerous`/`dangerously` zijn expliciete noodoverschrijvingen voor operators; het inschakelen ervan is op zichzelf geen melding van een beveiligingskwetsbaarheid. Zie voor de volledige inventaris van gevaarlijke parameters 'Overzicht van onveilige of gevaarlijke vlaggen' in [Beveiliging](/nl/gateway/security).

## SecretRef-gedrag

`security audit` verwerkt ondersteunde SecretRefs in alleen-lezenmodus voor de betreffende paden. Als een SecretRef niet beschikbaar is in het huidige opdrachtpad, gaat de audit door en meldt deze `secretDiagnostics` in plaats van te crashen. `--token` en `--password` overschrijven alleen de authenticatie voor diepgaande controles voor die specifieke opdrachtaanroep; ze herschrijven geen configuratie of SecretRef-toewijzingen.

## Onderdrukkingen

Accepteer opzettelijke, permanente bevindingen met `security.audit.suppressions`. Elke onderdrukking komt overeen met een exacte `checkId` en kan worden beperkt met hoofdletterongevoelige substrings voor `titleIncludes` en/of `detailIncludes`:

```json
{
  "security": {
    "audit": {
      "suppressions": [
        {
          "checkId": "plugins.tools_reachable_permissive_policy",
          "detailIncludes": "Enabled extension plugins: gbrain",
          "reason": "trusted local operator plugin"
        }
      ]
    }
  }
}
```

Onderdrukte bevindingen worden verwijderd uit de actieve lijsten `summary` en `findings`. JSON-uitvoer bewaart ze onder `suppressedFindings` voor controleerbaarheid. Wanneer onderdrukkingen zijn geconfigureerd, bevat de actieve uitvoer ook een niet-onderdrukbare informatieve bevinding `security.audit.suppressions.active`, zodat lezers kunnen zien dat de audit is gefilterd. Gevaarlijke configuratievlaggen worden als één bevinding per vlag gegenereerd, zodat het accepteren van één gevaarlijke vlag geen andere ingeschakelde vlaggen verbergt die dezelfde `config.insecure_or_dangerous_flags`-checkId delen.

Omdat onderdrukkingen permanente risico's kunnen verbergen, is voor het toevoegen of verwijderen ervan via door een agent uitgevoerde shellopdrachten uitvoeringsgoedkeuring vereist, tenzij de uitvoering al plaatsvindt met `security="full"` en `ask="off"` voor vertrouwde lokale automatisering.

## JSON-uitvoer

```bash
openclaw security audit --json | jq '.summary'
openclaw security audit --deep --json | jq '.findings[] | select(.severity=="critical") | .checkId'
```

Met `--fix --json` bevat de uitvoer zowel herstelacties als het eindrapport:

```bash
openclaw security audit --fix --json | jq '{fix: .fix.ok, summary: .report.summary}'
```

## Wat `--fix` wijzigt

Past veilige, deterministische herstelmaatregelen toe:

- wijzigt veelvoorkomende `groupPolicy="open"` in `groupPolicy="allowlist"` (inclusief accountvarianten in ondersteunde kanalen)
- wanneer het WhatsApp-groepsbeleid wordt gewijzigd in `allowlist`, wordt `groupAllowFrom` gevuld vanuit het opgeslagen `allowFrom`-bestand als die lijst bestaat en de configuratie nog geen `allowFrom` definieert
- wijzigt `logging.redactSensitive` van `"off"` in `"tools"`
- beperkt machtigingen voor status-/configuratiebestanden en veelvoorkomende gevoelige bestanden (`credentials/*.json`, `auth-profiles.json`, `openclaw-agent.sqlite` en verouderde sessieartefacten)
- beperkt ook de machtigingen voor configuratie-invoegbestanden waarnaar vanuit `openclaw.json` wordt verwezen
- gebruikt `chmod` op POSIX-hosts en `icacls`-resets op Windows

`--fix` doet **niet** het volgende:

- tokens/wachtwoorden/API-sleutels roteren
- tools uitschakelen (`gateway`, `cron`, `exec`, enzovoort)
- keuzes voor Gateway-binding/authenticatie/netwerkblootstelling wijzigen
- Plugins/Skills verwijderen of herschrijven

## Gerelateerd

- [CLI-referentie](/nl/cli)
- [Beveiligingsaudit](/nl/gateway/security)
