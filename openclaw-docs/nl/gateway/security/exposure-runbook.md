---
read_when:
    - De Gateway beschikbaar maken via LAN, tailnet, Tailscale Serve, Funnel of een reverse proxy
    - Een implementatie controleren voordat echte gebruikers berichten mogen versturen
    - Een riskante configuratie voor externe toegang of privéberichten terugdraaien
sidebarTitle: Exposure runbook
summary: Preflight- en rollbackchecklist voordat een OpenClaw Gateway buiten loopback toegankelijk wordt gemaakt
title: Runbook voor blootstelling van de Gateway
x-i18n:
    generated_at: "2026-07-27T05:01:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fb8e66af57e804325afc91281122b822183337177c734efe065c5fc18b175e72
    source_path: gateway/security/exposure-runbook.md
    workflow: 16
---

<Warning>
Stel de Gateway pas bloot nadat je kunt uitleggen wie deze kan bereiken, hoe diegene
wordt geauthenticeerd, welke agents diegene kan activeren en welke tools die agents kunnen
gebruiken. Ga bij twijfel terug naar uitsluitend loopbacktoegang en voer de audit opnieuw uit.
</Warning>

Dit draaiboek zet de bredere richtlijnen voor [beveiliging](/nl/gateway/security) om in een
checklist voor beheerders voor externe toegang en blootstelling via berichtenkanalen.

## Kies het blootstellingspatroon

Geef de voorkeur aan het meest beperkte patroon dat aan de workflow voldoet.

| Patroon                    | Aanbevolen wanneer                              | Vereiste beheersmaatregelen                                                                                                               |
| -------------------------- | ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| Loopback + SSH-tunnel      | Persoonlijk gebruik, beheerderstoegang, foutopsporing           | Behoud `gateway.bind: "loopback"` en tunnel `127.0.0.1:18789`                                                                    |
| Loopback + Tailscale Serve | Persoonlijke tailnettoegang tot Control UI/WebSocket | Houd de Gateway uitsluitend op loopback; Tailscale-identiteitsheaders authenticeren alleen het WebSocket-oppervlak van de Control UI, niet andere authenticatiepaden |
| Tailnet/LAN-binding           | Speciaal privénetwerk met bekende apparaten    | Gateway-authenticatie, firewalltoelatingslijst, geen openbare poortdoorsturing                                                                        |
| Vertrouwde reverse proxy      | SSO/OIDC van de organisatie vóór de Gateway       | `trusted-proxy`-authenticatie, strikte `trustedProxies`, regels voor overschrijven/verwijderen van headers, expliciet toegestane gebruikers                             |
| Openbaar internet            | Zeldzame implementaties met hoog risico                     | Identiteitsbewuste proxy, TLS, frequentielimieten, strikte toelatingslijsten, geïsoleerde niet-hoofdsessies                                          |

Vermijd directe openbare poortdoorsturing naar de Gateway. Als openbare toegang
vereist is, plaats je er een identiteitsbewuste proxy voor en maak je de proxy het
enige netwerkpad naar de Gateway.

## Inventarisatie vooraf

Leg het volgende vast voordat je het bindings-, proxy-, Tailscale- of kanaalbeleid wijzigt:

- Gateway-host, OS-gebruiker en statusmap (standaard `~/.openclaw`).
- Gateway-URL en bindingsmodus (`gateway.bind`; standaardpoort `18789`).
- Authenticatiemodus, bron van token/wachtwoord of identiteitsbron van de vertrouwde proxy.
- Elk ingeschakeld kanaal en of het privéberichten, groepen of webhooks accepteert.
- Agents die bereikbaar zijn voor niet-lokale afzenders.
- Toolprofiel, sandboxmodus en beleid voor verhoogde tools voor elke bereikbare agent.
- Externe aanmeldgegevens die voor deze agents beschikbaar zijn.
- Back-uplocatie voor `~/.openclaw/openclaw.json` en aanmeldgegevens.

Als meer dan één persoon berichten naar de bot kan sturen, behandel dit dan als gedeelde,
gedelegeerde toolbevoegdheid en niet als hostisolatie per gebruiker.

## Basiscontroles

Voer het volgende uit voordat je toegang opent:

```bash
openclaw doctor
openclaw security audit
openclaw security audit --deep
openclaw health
```

Los kritieke bevindingen eerst op. Accepteer waarschuwingen alleen wanneer deze bewust
en gedocumenteerd zijn voor de implementatie. Zie [Controles van de beveiligingsaudit](/nl/gateway/security/audit-checks)
voor wat elke `checkId` betekent en wat de bijbehorende reparatiesleutel is.

Geef voor externe CLI-validatie de aanmeldgegevens expliciet door:

```bash
openclaw gateway probe --url ws://127.0.0.1:18789 --token "$OPENCLAW_GATEWAY_TOKEN"
```

Ga er niet van uit dat aanmeldgegevens uit de lokale configuratie van toepassing zijn op een expliciete externe URL.

## Minimale veilige basisconfiguratie

Gebruik deze vorm als uitgangspunt voor blootgestelde implementaties:

```json5
{
  gateway: {
    bind: "loopback",
    auth: {
      mode: "token",
      token: "replace-with-a-long-random-token",
    },
  },
  session: {
    dmScope: "per-channel-peer",
  },
  agents: {
    defaults: {
      sandbox: { mode: "non-main" },
    },
  },
  tools: {
    profile: "messaging",
    exec: { security: "deny", ask: "always" },
    elevated: { enabled: false },
  },
}
```

Verruim één beheersmaatregel tegelijk: voeg een specifieke kanaaltoelatingslijst toe voordat je
tools met schrijfrechten inschakelt, of schakel een reverse proxy in voordat je extern
Control UI-verkeer accepteert.

`tools.exec.security: "deny"` blokkeert alle exec-aanroepen, inclusief onschuldige
diagnostiek. Als diagnostiek of opdrachten met een laag risico vereist zijn, versoepel je dit pas
nadat je de specifieke afzenders, agents, opdrachten en goedkeuringsmodus hebt gekozen die
bij je dreigingsmodel passen.

## Blootstelling via privéberichten en groepen

Berichtenkanalen zijn oppervlakken voor niet-vertrouwde invoer. Voordat je privéberichten of
groepen toestaat:

- Geef de voorkeur aan `dmPolicy: "pairing"` of een strikte `allowFrom`-lijst boven `dmPolicy: "open"`.
- Combineer `"*"`-toelatingslijsten niet met brede tooltoegang.
- Vereis vermeldingen in groepen, tenzij de ruimte streng wordt beheerd.
- Stel `session.dmScope: "per-channel-peer"` in (of `"per-account-channel-peer"` voor
  kanalen met meerdere accounts) wanneer meerdere mensen privéberichten naar de bot kunnen sturen, zodat sessies voor
  privéberichten geen context delen.
- Routeer gedeelde kanalen naar agents met minimale tools en zonder persoonlijke
  aanmeldgegevens.

Koppelen geeft de afzender toestemming om de bot te activeren. Het maakt die afzender niet
tot een afzonderlijke hostbeveiligingsgrens.

## Controles voor reverse proxy's

Voor identiteitsbewuste proxy's:

- De proxy moet gebruikers authenticeren voordat aanvragen naar de Gateway worden doorgestuurd.
- De firewall of het netwerkbeleid moet directe toegang tot de Gateway-poort blokkeren.
- `gateway.trustedProxies` mag alleen de bron-IP-adressen van de proxy bevatten.
- De proxy moet door de client aangeleverde identiteits- en doorstuurheaders verwijderen of
  overschrijven.
- Stel `gateway.auth.trustedProxy.allowUsers` in wanneer de proxy meer dan
  één doelgroep bedient.
- Gebruik `gateway.auth.trustedProxy.allowLoopback` alleen voor een proxy op dezelfde host
  waarbij lokale processen worden vertrouwd en de proxy de identiteitsheaders beheert.

Voer `openclaw security audit --deep` uit na proxywijzigingen. Bevindingen over vertrouwde proxy's
zijn sterke signalen, omdat de proxy de authenticatiegrens
wordt.

## Beoordeling van tools en sandbox

Voordat je een agent blootstelt aan externe afzenders:

- Controleer welke sessies op de host en welke in de sandbox worden uitgevoerd.
- Weiger host-exec of vereis hiervoor goedkeuring.
- Houd verhoogde tools uitgeschakeld, tenzij een specifieke, vertrouwde afzender deze nodig heeft.
- Vermijd browser-, canvas-, node-, cron-, gateway- en tools voor het starten van sessies op open
  of halfopen berichtenoppervlakken.
- Houd bind-mounts beperkt; vermijd paden voor aanmeldgegevens, de thuismap, de Docker-socket en het systeem.
- Gebruik afzonderlijke gateways, OS-gebruikers of hosts voor wezenlijk verschillende
  vertrouwensgrenzen.

Als externe gebruikers niet volledig worden vertrouwd, moet isolatie afkomstig zijn van afzonderlijke
implementaties en niet alleen van prompts of sessielabels.

## Validatie na wijzigingen

Na elke wijziging in de blootstelling:

1. Voer `openclaw security audit --deep` opnieuw uit.
2. Controleer of een geautoriseerde verbinding slaagt.
3. Controleer of een niet-geautoriseerde afzender of browsersessie wordt geweigerd.
4. Controleer of geheimen in logboeken worden geredigeerd.
5. Controleer of de routering van privéberichten/groepen alleen de bedoelde agent bereikt.
6. Controleer of tools met grote impact om goedkeuring vragen of worden geweigerd.
7. Documenteer de geaccepteerde resterende waarschuwingen.

Ga niet door naar de volgende wijziging in de blootstelling totdat de huidige wijziging
wordt begrepen.

## Terugvalplan

Als de Gateway mogelijk te veel is blootgesteld:

```json5
{
  gateway: {
    bind: "loopback",
  },
  channels: {
    whatsapp: { dmPolicy: "disabled" },
    telegram: { dmPolicy: "disabled" },
    discord: { dmPolicy: "disabled" },
    slack: { dmPolicy: "disabled" },
  },
  tools: {
    exec: { security: "deny", ask: "always" },
    elevated: { enabled: false },
  },
}
```

Vervolgens:

1. Stop openbare doorsturing, Tailscale Funnel of reverse-proxyroutes.
2. Roteer Gateway-tokens/wachtwoorden en betrokken integratie-aanmeldgegevens.
3. Verwijder `"*"` en onverwachte afzenders uit toelatingslijsten.
4. Controleer recente auditlogboeken, uitvoeringsgeschiedenis, toolaanroepen en configuratiewijzigingen.
5. Voer `openclaw security audit --deep` opnieuw uit.
6. Schakel toegang opnieuw in met het meest beperkte patroon dat aan de workflow voldoet.

## Beoordelingschecklist

- De Gateway blijft uitsluitend op loopback, tenzij er een gedocumenteerde reden is.
- Niet-loopbacktoegang heeft authenticatie en firewallbeveiliging en geen directe openbare route.
- Implementaties met een vertrouwde proxy hebben strikte proxy-IP-adressen en headercontroles.
- Privéberichten gebruiken standaard koppeling of toelatingslijsten, geen open toegang.
- Groepen vereisen vermeldingen of expliciete toelatingslijsten.
- Gedeelde kanalen hebben geen toegang tot persoonlijke aanmeldgegevens.
- Niet-hoofdsessies worden in sandboxmodus uitgevoerd.
- Host-exec en verhoogde tools worden geweigerd of vereisen goedkeuring.
- Logboeken redigeren geheimen.
- Kritieke auditbevindingen zijn opgelost.
- Terugvalstappen zijn getest en gedocumenteerd.
