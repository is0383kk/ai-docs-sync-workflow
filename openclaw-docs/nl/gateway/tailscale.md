---
read_when:
    - De Gateway-beheerinterface buiten localhost beschikbaar maken
    - Toegang tot het dashboard via tailnet of openbaar netwerk automatiseren
summary: Geïntegreerde Tailscale Serve/Funnel voor het Gateway-dashboard
title: Tailscale
x-i18n:
    generated_at: "2026-07-27T05:55:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e201a64ac427994401fae1b934d94e0c5afe976b4acd34d45b059978f5f1807e
    source_path: gateway/tailscale.md
    workflow: 16
---

OpenClaw kan Tailscale **Serve** (tailnet) of **Funnel** (openbaar) automatisch configureren voor het Gateway-dashboard en de WebSocket-poort. Zo blijft de Gateway gebonden aan loopback, terwijl Tailscale HTTPS, routering en (voor Serve) identiteitsheaders biedt.

## Modi

`gateway.tailscale.mode`:

| Modus            | Gedrag                                                                    |
| --------------- | --------------------------------------------------------------------------- |
| `serve`         | Serve alleen via tailnet met `tailscale serve`. De Gateway blijft op `127.0.0.1`. |
| `funnel`        | Openbare HTTPS via `tailscale funnel`. Vereist een gedeeld wachtwoord.            |
| `off` (standaard) | Geen Tailscale-automatisering.                                                    |

Status- en audituitvoer gebruiken **Tailscale-blootstelling** voor deze OpenClaw Serve/Funnel-modus. `off` betekent dat OpenClaw Serve of Funnel niet beheert; het betekent niet dat de lokale Tailscale-daemon is gestopt of afgemeld.

## Configuratievoorbeelden

### Alleen tailnet (Serve)

```json5
{
  gateway: {
    bind: "loopback",
    tailscale: { mode: "serve" },
  },
}
```

Open: `https://<magicdns>/` (of je geconfigureerde `gateway.controlUi.basePath`)

Als je de Control UI via een benoemde Tailscale Service wilt ontsluiten in plaats van via de hostnaam van het apparaat, stel je `gateway.tailscale.serviceName` in op de naam van de Service:

```json5
{
  gateway: {
    bind: "loopback",
    tailscale: { mode: "serve", serviceName: "svc:openclaw" },
  },
}
```

Bij het opstarten wordt dan de Service-URL als `https://openclaw.<tailnet-name>.ts.net/` gemeld in plaats van de hostnaam van het apparaat. Voor Tailscale Services moet de host een goedgekeurde getagde Node in je tailnet zijn — configureer de tag en keur de Service goed in Tailscale voordat je dit inschakelt, anders mislukt `tailscale serve --service=...` tijdens het opstarten van de Gateway.

### Alleen tailnet (binden aan Tailnet-IP)

Gebruik dit om de Gateway rechtstreeks op het Tailnet-IP te laten luisteren, zonder Serve/Funnel:

```json5
{
  gateway: {
    bind: "tailnet",
    auth: { mode: "token", token: "your-token" },
  },
}
```

Maak verbinding vanaf een ander Tailnet-apparaat:

- Control UI: `http://<tailscale-ip>:18789/`
- WebSocket: `ws://<tailscale-ip>:18789`

<Note>
Wanneer een bindbaar Tailnet-IPv4-adres aanwezig is, vereist de Gateway ook `http://127.0.0.1:18789` voor geverifieerde clients op dezelfde host. Als er bij het opstarten geen Tailnet-adres beschikbaar is, wordt alleen op loopback teruggevallen; start opnieuw nadat Tailscale beschikbaar is om rechtstreekse Tailnet-toegang toe te voegen. Geen van beide paden voegt LAN- of openbare blootstelling toe.
</Note>

### Openbaar internet (Funnel + gedeeld wachtwoord)

```json5
{
  gateway: {
    bind: "loopback",
    tailscale: { mode: "funnel" },
    auth: { mode: "password", password: "replace-me" },
  },
}
```

Geef de voorkeur aan `OPENCLAW_GATEWAY_PASSWORD` boven het vastleggen van een wachtwoord op schijf.

## CLI-voorbeelden

```bash
openclaw gateway --tailscale serve
openclaw gateway --tailscale funnel --auth password
```

## Authenticatie

`gateway.auth.mode` regelt de handshake:

| Modus                                                   | Gebruiksscenario                                                                            |
| ------------------------------------------------------ | ----------------------------------------------------------------------------------- |
| `none`                                                 | Alleen privé-ingang                                                                |
| `token` (standaard wanneer `OPENCLAW_GATEWAY_TOKEN` is ingesteld) | Gedeeld token                                                                        |
| `password`                                             | Gedeeld geheim via `OPENCLAW_GATEWAY_PASSWORD` of configuratie                             |
| `trusted-proxy`                                        | Identiteitsbewuste reverse proxy; zie [Authenticatie via vertrouwde proxy](/nl/gateway/trusted-proxy-auth) |

### Tailscale-identiteitsheaders (alleen Serve)

Wanneer `tailscale.mode: "serve"` en `gateway.auth.allowTailscale` gelijk is aan `true`, kan de authenticatie van de Control UI/WebSocket Tailscale-identiteitsheaders (`tailscale-user-login`) gebruiken in plaats van een token/wachtwoord. OpenClaw verifieert de header door het `x-forwarded-for`-adres van de aanvraag via de lokale Tailscale-daemon (`tailscale whois`) om te zetten en dit te vergelijken met de aanmeldnaam in de header voordat de aanvraag wordt geaccepteerd. Een aanvraag komt alleen voor dit pad in aanmerking wanneer deze vanaf loopback binnenkomt met de headers `x-forwarded-for`, `x-forwarded-proto` en `x-forwarded-host` van Tailscale.

Deze tokenloze flow veronderstelt dat de Gateway-host wordt vertrouwd. Als niet-vertrouwde lokale code op dezelfde host kan worden uitgevoerd, stel je `gateway.auth.allowTailscale: false` in en vereis je in plaats daarvan authenticatie met een token/wachtwoord.

Bereik van de omzeiling:

- Geldt alleen voor het WebSocket-authenticatieoppervlak van de Control UI. HTTP-API-eindpunten (`/v1/*`, `/tools/invoke`, `/api/channels/*`, enzovoort) gebruiken nooit authenticatie via Tailscale-identiteitsheaders; ze volgen altijd de normale HTTP-authenticatiemodus van de Gateway.
- Voor operatorsessies in de Control UI die al een browserapparaatidentiteit bevatten, slaat een geverifieerde Tailscale-identiteit de retourstap voor koppelen via bootstrap-token/QR-code over.
- Dit omzeilt de apparaatidentiteit zelf niet: clients zonder apparaatidentiteit worden nog steeds geweigerd en verbindingen met een Node-rol doorlopen nog steeds de normale koppelings- en authenticatiecontroles.

## Opmerkingen

- Voor Tailscale Serve/Funnel moet de `tailscale`-CLI geïnstalleerd en aangemeld zijn.
- `tailscale.mode: "funnel"` weigert te starten tenzij de authenticatiemodus `password` is, om openbare blootstelling te voorkomen.
- `gateway.tailscale.serviceName` geldt alleen voor de Serve-modus en wordt doorgegeven aan `tailscale serve --service=<name>`. De waarde moet de `svc:<dns-label>`-indeling van Tailscale gebruiken, bijvoorbeeld `svc:openclaw`. Tailscale vereist dat Service-hosts getagde Nodes zijn en mogelijk moet de Service in de beheerconsole worden goedgekeurd voordat Serve deze kan publiceren.
- `gateway.tailscale.resetOnExit` maakt de configuratie van `tailscale serve`/`tailscale funnel` bij het afsluiten ongedaan.
- `gateway.tailscale.preserveFunnel: true` houdt een extern geconfigureerde `tailscale funnel`-route actief wanneer de Gateway opnieuw wordt gestart. Met `mode: "serve"` controleert OpenClaw `tailscale funnel status` voordat Serve opnieuw wordt toegepast en slaat dit over wanneer een Funnel-route de Gateway-poort al dekt. Het door OpenClaw beheerde beleid van Funnel met uitsluitend een wachtwoord blijft ongewijzigd.
- `gateway.bind: "tailnet"` gebruikt een rechtstreekse Tailnet-binding (geen HTTPS, geen Serve/Funnel) plus de vereiste lokale `127.0.0.1` wanneer een Tailnet-IPv4-adres beschikbaar is; anders wordt alleen op loopback teruggevallen.
- `gateway.bind: "auto"` geeft de voorkeur aan loopback; gebruik `tailnet` om de netwerkblootstelling tot het Tailnet te beperken en tegelijkertijd loopback-toegang op dezelfde host te behouden.
- Serve/Funnel ontsluiten alleen de **Gateway Control UI + WS**. Nodes maken verbinding via hetzelfde Gateway-WS-eindpunt, dus Serve werkt ook voor Node-toegang.

### Vereisten en beperkingen van Tailscale

- Voor Serve moet HTTPS voor je tailnet zijn ingeschakeld; de CLI toont een prompt als dit ontbreekt.
- Serve voegt Tailscale-identiteitsheaders toe; Funnel niet.
- Funnel vereist Tailscale v1.38.3+, MagicDNS, ingeschakelde HTTPS en een Funnel-Node-attribuut.
- Funnel ondersteunt via TLS alleen de poorten `443`, `8443` en `10000`.
- Voor Funnel op macOS is de opensourcevariant van de Tailscale-app vereist.

## Browserbesturing (externe Gateway + lokale browser)

Als je de Gateway op de ene machine wilt uitvoeren maar een browser op een andere machine wilt besturen, voer je een **Node-host** uit op de browsermachine en houd je beide binnen hetzelfde tailnet. De Gateway stuurt browseracties door naar de Node; er is geen afzonderlijke besturingsserver of Serve-URL nodig.

Vermijd Funnel voor browserbesturing; behandel het koppelen van Nodes als operatorstoegang.

## Meer informatie

- Overzicht van Tailscale Serve: [https://tailscale.com/kb/1312/serve](https://tailscale.com/kb/1312/serve)
- `tailscale serve`-opdracht: [https://tailscale.com/kb/1242/tailscale-serve](https://tailscale.com/kb/1242/tailscale-serve)
- Overzicht van Tailscale Funnel: [https://tailscale.com/kb/1223/tailscale-funnel](https://tailscale.com/kb/1223/tailscale-funnel)
- `tailscale funnel`-opdracht: [https://tailscale.com/kb/1311/tailscale-funnel](https://tailscale.com/kb/1311/tailscale-funnel)

## Gerelateerd

- [Externe toegang](/nl/gateway/remote)
- [Detectie](/nl/gateway/discovery)
- [Authenticatie](/nl/gateway/authentication)
