---
read_when:
    - Dashboardverificatie of blootstellingsmodi wijzigen
summary: Toegang tot en authenticatie voor het Gateway-dashboard (Control UI)
title: Dashboard
x-i18n:
    generated_at: "2026-07-27T06:37:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ca531ad2943dfdee1cd90a4efdc1fb69c4517780e2be52237fd558b8638e7cd0
    source_path: web/dashboard.md
    workflow: 16
---

De Gateway-dashboard is de browsergebaseerde Control UI die standaard wordt aangeboden op `/` (overschrijf dit met `gateway.controlUi.basePath`).

Snel openen (lokale Gateway):

- [http://127.0.0.1:18789/](http://127.0.0.1:18789/) (of [http://localhost:18789/](http://localhost:18789/))
- Met `gateway.tls.enabled: true` gebruik je `https://127.0.0.1:18789/` en `wss://127.0.0.1:18789` voor het WebSocket-eindpunt.

Belangrijke verwijzingen:

- [Control UI](/nl/web/control-ui) voor gebruik en UI-mogelijkheden.
- [Tailscale](/nl/gateway/tailscale) voor Serve/Funnel-automatisering.
- [Weboppervlakken](/nl/web) voor bindmodi en beveiligingsopmerkingen.

Authenticatie wordt tijdens de WebSocket-handshake afgedwongen via het geconfigureerde authenticatiepad van de Gateway:

- `connect.params.auth.token`
- `connect.params.auth.password`
- Tailscale Serve-identiteitsheaders wanneer `gateway.auth.allowTailscale: true`
- Identiteitsheaders van een vertrouwde proxy wanneer `gateway.auth.mode: "trusted-proxy"`

Zie `gateway.auth` in [Gateway-configuratie](/nl/gateway/configuration).

<Warning>
De Control UI is een **beheerdersinterface** (chat, configuratie, uitvoeringsgoedkeuringen). Stel deze niet openbaar beschikbaar. De UI bewaart tokens uit dashboard-URL's in sessionStorage voor het huidige browsertabblad en de geselecteerde Gateway-URL en verwijdert ze na het laden uit de URL. Gebruik bij voorkeur localhost, Tailscale Serve of een SSH-tunnel.
</Warning>

## Snelste methode (aanbevolen)

- Na de onboarding opent de CLI automatisch het dashboard en toont deze een nette link (zonder token).
- Op elk moment opnieuw openen: `openclaw dashboard` (kopieert de link, opent indien mogelijk een browser en toont een SSH-hint in een headless-omgeving).
- Als zowel levering via het klembord als via de browser mislukt, toont `openclaw dashboard` nog steeds de nette URL en wordt aangegeven dat je jouw token (uit `OPENCLAW_GATEWAY_TOKEN` of `gateway.auth.token`) als URL-fragmentsleutel `token` moet toevoegen; de tokenwaarde wordt nooit in logboeken weergegeven.
- Als de UI om authenticatie met een gedeeld geheim vraagt, plak je het geconfigureerde token of wachtwoord in de instellingen van de Control UI.

## Basisprincipes van authenticatie (lokaal versus extern)

- **Localhost**: open `http://127.0.0.1:18789/`.
- **Gateway-TLS**: wanneer `gateway.tls.enabled: true`, gebruiken dashboard-/statuslinks `https://` en gebruiken WebSocket-links van de Control UI `wss://`.
- **Bron van token voor gedeeld geheim**: `gateway.auth.token` (of `OPENCLAW_GATEWAY_TOKEN`). `openclaw dashboard` kan dit via het URL-fragment doorgeven voor eenmalige initialisatie; de Control UI bewaart het in sessionStorage voor het huidige tabblad en de geselecteerde Gateway-URL, niet in localStorage.
- **Runtime-token bij ontbrekende configuratie**: als bij het opstarten wordt gemeld dat een runtime-token is gegenereerd, is dat token tijdelijk en niet beschikbaar via `openclaw config get gateway.auth.token`. Ook voor loopback is authenticatie vereist. Voer `openclaw doctor --generate-gateway-token` uit, start de Gateway opnieuw en plak daarna het geconfigureerde token in de instellingen van de Control UI.
- Als `gateway.auth.token` door SecretRef wordt beheerd, toont/kopieert/opent `openclaw dashboard` bewust een URL zonder token, om te voorkomen dat extern beheerde tokens zichtbaar worden in shelllogboeken, klembordgeschiedenis of argumenten voor het starten van de browser. Als de verwijzing in je huidige shell niet kan worden opgelost, wordt nog steeds de URL zonder token weergegeven, samen met uitvoerbare instructies voor het instellen van authenticatie.
- **Wachtwoord voor gedeeld geheim**: gebruik de geconfigureerde `gateway.auth.password` (of `OPENCLAW_GATEWAY_PASSWORD`). Het dashboard bewaart wachtwoorden niet tussen herlaadbeurten.
- **Modi met identiteit**: Tailscale Serve voldoet via identiteitsheaders aan de authenticatievereisten voor de Control UI/WebSocket wanneer `gateway.auth.allowTailscale: true`; een identiteitsbewuste reverse proxy zonder loopback voldoet aan `gateway.auth.mode: "trusted-proxy"`. Voor geen van beide hoeft een gedeeld geheim voor de WebSocket te worden geplakt.
- **Niet localhost**: gebruik Tailscale Serve, een bind zonder loopback met een gedeeld geheim, een identiteitsbewuste reverse proxy zonder loopback met `gateway.auth.mode: "trusted-proxy"`, of een SSH-tunnel. HTTP-API's gebruiken nog steeds authenticatie met een gedeeld geheim, tenzij je bewust `gateway.auth.mode: "none"` met privé-ingang of HTTP-authenticatie via een vertrouwde proxy uitvoert. Zie [Weboppervlakken](/nl/web).

## Openen in Telegram

Telegram-bots kunnen het dashboard met `/dashboard` openen als een Telegram Mini App.

Vereisten:

- `gateway.tailscale.mode: "serve"` of `"funnel"`, zodat Telegram een HTTPS-URL voor de Mini App ontvangt.
- De Telegram-afzender moet de eigenaar van de bot zijn: een numerieke Telegram-gebruikers-ID in `commands.ownerAllowFrom` of de effectieve `channels.telegram.allowFrom` van het geselecteerde account.
- Voer `/dashboard` uit in een privébericht met de bot. Bij aanroepen in groepen wordt alleen aangegeven dat je de opdracht in een privébericht moet openen en wordt geen knop weergegeven.
- Docker-installaties: voor Serve/Funnel-modi moet de Gateway naast `tailscaled` aan loopback worden gebonden, waaraan bridgenetwerken met gepubliceerde poorten niet kunnen voldoen. Voer de Gateway-container uit met `network_mode: host` en koppel de `tailscaled`-socket van de host (`/var/run/tailscale`) plus de `tailscale`-CLI in de container.

De Mini App voert een eenmalige eigendomsoverdracht uit en leidt met een kortlevend bootstrap-token door naar de Control UI. Er wordt geen gedeeld Gateway-token in de URL weergegeven.

Niet-doelen voor v1:

- Telegram Web-iframe wordt niet ondersteund.
- Tailscale Serve/Funnel is het enige ondersteunde pad voor een gepubliceerde URL.

<a id="if-you-see-unauthorized-1008"></a>

## Als je "unauthorized" / 1008 ziet

- Controleer of de Gateway bereikbaar is: lokaal via `openclaw status`; extern via de SSH-tunnel `ssh -N -L 18789:127.0.0.1:18789 user@gateway-host`, waarna je `http://127.0.0.1:18789/` opent.
- Voor `AUTH_TOKEN_MISMATCH` mogen clients één vertrouwde nieuwe poging doen met een gecachet apparaattoken wanneer de Gateway aanwijzingen voor een nieuwe poging retourneert; die nieuwe poging gebruikt opnieuw de gecachete goedgekeurde bereiken van het token (aanroepers met expliciete `deviceToken`/`scopes` behouden hun aangevraagde verzameling bereiken). Als de authenticatie na die nieuwe poging nog steeds mislukt, los je tokenafwijking handmatig op.
- Voor `AUTH_SCOPE_MISMATCH` is het apparaattoken herkend, maar bevat het niet de aangevraagde bereiken; koppel opnieuw of keur de nieuwe verzameling bereiken goed in plaats van het gedeelde Gateway-token te roteren.
- Buiten dat pad voor een nieuwe poging is de prioriteitsvolgorde voor verbindingsauthenticatie: expliciet gedeeld token/wachtwoord, vervolgens expliciete `deviceToken`, daarna het opgeslagen apparaattoken en ten slotte het bootstrap-token.
- Op het asynchrone Tailscale Serve-pad worden mislukte pogingen voor dezelfde `{scope, ip}` geserialiseerd voordat de begrenzer voor mislukte authenticatie ze registreert, zodat een tweede gelijktijdige mislukte nieuwe poging al `retry later` kan tonen.
- Zie [Controlelijst voor herstel van tokenafwijking](/nl/cli/devices#token-drift-recovery-checklist) voor stappen om tokenafwijking te herstellen.
- Haal het gedeelde geheim op van of verstrek het vanaf de Gateway-host:
  - Token: `openclaw config get gateway.auth.token`
  - Wachtwoord: los de geconfigureerde `gateway.auth.password` of `OPENCLAW_GATEWAY_PASSWORD` op
  - Door SecretRef beheerd token: los de externe geheimenprovider op, of exporteer `OPENCLAW_GATEWAY_TOKEN` in deze shell en voer `openclaw dashboard` opnieuw uit
  - Runtime-token gegenereerd omdat geen gedeeld geheim was geconfigureerd: voer `openclaw doctor --generate-gateway-token` uit, start de Gateway opnieuw en gebruik daarna het geconfigureerde token
- Plak in de dashboardinstellingen het token of wachtwoord in het authenticatieveld en maak vervolgens verbinding.
- De taalkiezer van de UI staat onder **Settings -> General -> Language**, niet onder Appearance.

## Gerelateerd

- [Control UI](/nl/web/control-ui)
- [WebChat](/nl/web/webchat)
