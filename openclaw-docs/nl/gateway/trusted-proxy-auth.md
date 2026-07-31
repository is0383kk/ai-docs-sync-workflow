---
read_when:
    - OpenClaw uitvoeren achter een identiteitsbewuste proxy
    - Pomerium, Caddy of nginx met OAuth instellen vóór OpenClaw
    - WebSocket 1008-fouten wegens ontbrekende autorisatie oplossen bij reverse-proxyconfiguraties
    - Bepalen waar je HSTS en andere HTTP-beveiligingsheaders instelt
sidebarTitle: Trusted proxy auth
summary: Delegeer Gateway-authenticatie aan een vertrouwde reverse proxy (Pomerium, Caddy, nginx + OAuth)
title: Authenticatie via vertrouwde proxy
x-i18n:
    generated_at: "2026-07-27T05:06:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 39bf8f12b3ae95f53b21bfed12deb1c8ed8f767711955bbee52c74538052a89f
    source_path: gateway/trusted-proxy-auth.md
    workflow: 16
---

<Warning>
**Beveiligingsgevoelige functie.** Deze modus delegeert de authenticatie volledig aan je reverse proxy. Een onjuiste configuratie kan je Gateway blootstellen aan ongeautoriseerde toegang. Lees deze pagina zorgvuldig voordat je deze functie inschakelt.
</Warning>

## Wanneer te gebruiken

- Je voert OpenClaw uit achter een **identiteitsbewuste proxy** (Pomerium, Caddy + OAuth, nginx + oauth2-proxy, Traefik + forward auth).
- Je proxy verzorgt alle authenticatie en geeft de gebruikersidentiteit door via headers.
- Je bevindt je in een Kubernetes- of containeromgeving waarin de proxy het enige pad naar de Gateway is.
- Je krijgt WebSocket-fouten met `1008 unauthorized` omdat browsers geen tokens in WS-payloads kunnen doorgeven.

## Wanneer NIET te gebruiken

- Je proxy authenticeert geen gebruikers (maar is alleen een TLS-terminator of loadbalancer).
- Er is een pad naar de Gateway dat de proxy omzeilt (gaten in de firewall, toegang via het interne netwerk).
- Je weet niet zeker of je proxy doorgestuurde headers correct verwijdert/overschrijft.
- Je hebt alleen persoonlijke toegang voor één gebruiker nodig (overweeg in plaats daarvan Tailscale Serve + loopback).

## Hoe het werkt

<Steps>
  <Step title="Proxy authenticeert de gebruiker">
    Je reverse proxy authenticeert gebruikers (OAuth, OIDC, SAML, enz.).
  </Step>
  <Step title="Proxy voegt een identiteitsheader toe">
    De proxy voegt een header toe met de geauthenticeerde gebruikersidentiteit (bijv. `x-forwarded-user: nick@example.com`).
  </Step>
  <Step title="Gateway verifieert de vertrouwde bron">
    OpenClaw controleert of het verzoek afkomstig is van een **vertrouwd proxy-IP-adres** (`gateway.trustedProxies`) en niet van het eigen loopback- of lokale-interfaceadres van de Gateway.
  </Step>
  <Step title="Gateway extraheert de identiteit">
    OpenClaw leest de vereiste headers en vervolgens de gebruikersidentiteit uit de geconfigureerde header.
  </Step>
  <Step title="Autoriseren">
    Als alle controles slagen en de gebruiker voldoet aan `allowUsers` (indien ingesteld), wordt het verzoek geautoriseerd.
  </Step>
</Steps>

## Configuratie

```json5
{
  gateway: {
    // Authenticatie via een vertrouwde proxy verwacht standaard dat het bron-IP-adres van de proxy geen loopbackadres is
    bind: "lan",

    // KRITIEK: voeg hier alleen de IP-adressen van je proxy toe
    trustedProxies: ["10.0.0.1", "172.17.0.1"],

    auth: {
      mode: "trusted-proxy",
      trustedProxy: {
        // Header met de geauthenticeerde gebruikersidentiteit (vereist)
        userHeader: "x-forwarded-user",

        // Optioneel: headers die aanwezig MOETEN zijn (proxyverificatie)
        requiredHeaders: ["x-forwarded-proto", "x-forwarded-host"],

        // Optioneel: beperken tot specifieke gebruikers (leeg = iedereen toestaan)
        allowUsers: ["nick@example.com", "admin@company.org"],

        // Optioneel: een loopbackproxy op dezelfde host toestaan na expliciete aanmelding
        allowLoopback: false,

        // Optioneel: geauthenticeerde proxygebruikers nieuwe browserapparaten laten registreren
        deviceAutoApprove: {
          enabled: false,
          scopes: ["operator.read", "operator.write", "operator.approvals"],
        },
      },
    },
  },
}
```

<Warning>
**Runtimeregels, in evaluatievolgorde**

1. Het bron-IP-adres van het verzoek moet overeenkomen met `gateway.trustedProxies` (met ondersteuning voor CIDR), anders wordt het afgewezen (`trusted_proxy_untrusted_source`).
2. Verzoeken van een loopbackbron (`127.0.0.1`, `::1`) worden afgewezen, tenzij `gateway.auth.trustedProxy.allowLoopback = true` en het loopbackadres ook in `trustedProxies` staat (`trusted_proxy_loopback_source`). Deze controle wordt vóór de headercontroles uitgevoerd. Een loopbackbron mislukt daarom op deze manier, zelfs als ook vereiste headers ontbreken.
3. Niet-loopbackbronnen die overeenkomen met een van de eigen lokale netwerkinterfaceadressen van de Gateway, worden afgewezen als bescherming tegen spoofing (`trusted_proxy_local_interface_source`). Als het detecteren van interfaces zelf mislukt, wordt het verzoek eveneens afgewezen (`trusted_proxy_local_interface_check_failed`).
4. `requiredHeaders` en `userHeader` moeten aanwezig en niet leeg zijn.
5. `allowUsers` moet, indien niet leeg, de geëxtraheerde gebruiker bevatten.

**Bewijs uit doorgestuurde headers heeft voor lokale directe terugval voorrang op loopbacklokaliteit.** Als een verzoek via loopback binnenkomt maar een `Forwarded`-, een willekeurige `X-Forwarded-*`- of een `X-Real-IP`-header bevat, sluit dat bewijs het verzoek uit van lokale directe terugval naar een wachtwoord en van gating op basis van apparaatidentiteit, ook al mislukt authenticatie via een vertrouwde proxy nog steeds omdat de bron loopback is.

`allowLoopback` vertrouwt lokale processen op de Gateway-host in dezelfde mate als de reverse proxy. Schakel dit alleen in als de Gateway nog steeds door een firewall tegen directe externe toegang is beschermd en de lokale proxy door de client aangeleverde identiteitsheaders verwijdert of overschrijft.

Interne Gateway-clients die niet via de reverse proxy communiceren, moeten `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD` gebruiken en geen identiteitsheaders van een vertrouwde proxy. Control UI-implementaties buiten loopback vereisen nog steeds expliciet `gateway.controlUi.allowedOrigins`.
</Warning>

### Configuratiereferentie

<ParamField path="gateway.trustedProxies" type="string[]" required>
  Array met te vertrouwen proxy-IP-adressen (of CIDR's). Verzoeken van andere IP-adressen worden afgewezen.
</ParamField>
<ParamField path="gateway.auth.mode" type="string" required>
  Moet `"trusted-proxy"` zijn.
</ParamField>
<ParamField path="gateway.auth.trustedProxy.userHeader" type="string" required>
  Naam van de header met de geauthenticeerde gebruikersidentiteit.
</ParamField>
<ParamField path="gateway.auth.trustedProxy.requiredHeaders" type="string[]">
  Aanvullende headers die aanwezig moeten zijn om het verzoek te vertrouwen.
</ParamField>
<ParamField path="gateway.auth.trustedProxy.allowUsers" type="string[]">
  Toestaanlijst met gebruikersidentiteiten. Leeg betekent dat alle geauthenticeerde gebruikers worden toegestaan.
</ParamField>
<ParamField path="gateway.auth.trustedProxy.allowLoopback" type="boolean" default="false">
  Optionele ondersteuning voor loopback-reverse-proxy's op dezelfde host.
</ParamField>
<ParamField path="gateway.auth.trustedProxy.deviceAutoApprove.enabled" type="boolean" default="false">
  Keur nieuwe apparaatidentiteiten voor Control UI en WebChat automatisch goed na authenticatie via een vertrouwde proxy.
</ParamField>
<ParamField path="gateway.auth.trustedProxy.deviceAutoApprove.scopes" type="string[]" default='["operator.read", "operator.write", "operator.approvals"]'>
  Maximale scopes die aan een automatisch goedgekeurd browserapparaat worden toegekend. Door `operator.admin` expliciet op te nemen, kan elke via de proxy geauthenticeerde gebruiker automatisch volledige beheerderstoegang voor een apparaat aanvragen, krijgen verzoeken zonder scopes automatisch volledige beheerderstoegang en wordt de KRITIEKE beveiligingsbevinding `gateway.trusted_proxy_device_auto_approve_admin` geactiveerd, plus een waarschuwing bij het opstarten van de Gateway.
</ParamField>

<Warning>
Schakel `allowLoopback` alleen in wanneer de lokale reverse proxy de beoogde vertrouwensgrens is. Elk lokaal proces dat verbinding kan maken met de Gateway kan proberen proxy-identiteitsheaders te verzenden. Houd directe toegang tot de Gateway daarom privé voor de host en vereis headers die eigendom zijn van de proxy, zoals `x-forwarded-proto`, of een ondertekende attestatieheader als je proxy die ondersteunt.
</Warning>

## Automatische apparaatgoedkeuring

Authenticatie via een vertrouwde proxy kan optioneel de proxy-identiteit gebruiken als goedkeuringsgrens voor nieuwe browserapparaten:

```json5
{
  gateway: {
    auth: {
      mode: "trusted-proxy",
      trustedProxy: {
        userHeader: "x-forwarded-user",
        allowUsers: ["operator@example.com"],
        deviceAutoApprove: {
          enabled: true,
          scopes: ["operator.read", "operator.write", "operator.approvals"],
        },
      },
    },
  },
}
```

De standaardwaarde is `enabled: false`. Wanneer deze optie is ingeschakeld, gelden al deze regels:

1. De WebSocket moet zijn geauthenticeerd via de methode `trusted-proxy` met een niet-lege gebruikersidentiteit die voldoet aan `allowUsers` wanneer een toestaanlijst is geconfigureerd. Verbindingen via een token, wachtwoord of Tailscale en niet-geauthenticeerde verbindingen gebruiken dit beleid nooit.
2. Alleen een nieuw browserapparaat voor Control UI of WebChat kan automatisch worden goedgekeurd. Elk verzoek voor een bestaand apparaat, waaronder een scope-uitbreiding, blijft in afwachting van handmatige goedkeuring met `openclaw devices approve <requestId>`.
3. Het apparaat wordt goedgekeurd met de rol `operator`. Als het verbindingsverzoek scopes bevat, is de toekenning exact de doorsnede van de aangevraagde scopes en `deviceAutoApprove.scopes`. Als het verzoek geen scopes bevat, wordt de geconfigureerde lijst toegekend; wanneer die lijst ontbreekt, bestaat de standaard uit `operator.read`, `operator.write` en `operator.approvals`. De resulterende toekenning wordt vervolgens verder beperkt door de proxyheader [`x-openclaw-scopes`](#control-ui-pairing-behavior) van de verbinding, indien aanwezig. Een proxy die de scopes van een gebruiker beperkt, beperkt daarmee dus ook de **permanente** apparaattoekenning en niet alleen de sessie — een aanwezige maar lege header levert geen scopes op. Deze beperking geldt ook wanneer de client zijn eigen scopelijst weglaat.
4. `operator.admin` is alleen toegestaan wanneer het expliciet in `deviceAutoApprove.scopes` staat. Als het daarin staat, kan elke via de proxy geauthenticeerde gebruiker volledige beheerderstoegang voor een nieuw browserapparaat aanvragen en automatisch ontvangen; verzoeken zonder scopes krijgen automatisch volledige beheerderstoegang. `openclaw security audit` rapporteert de KRITIEKE bevinding `gateway.trusted_proxy_device_auto_approve_admin` en de Gateway registreert bij het opstarten eenmaal een waarschuwing. Geef de voorkeur aan handmatige goedkeuring voor beheerderstoegang met `openclaw devices approve` of `openclaw devices rotate` totdat rollen per identiteit beschikbaar zijn.

<Warning>
Als je deze optie inschakelt, wordt de registratie van nieuwe browserapparaten volledig aan de identiteit van de reverse proxy gedelegeerd. Met een gecompromitteerd proxyaccount kan een permanent apparaat met elke geconfigureerde scope worden geregistreerd. Door `operator.admin` op te nemen, wordt dat apparaat zonder handmatige goedkeuring een volledige beheerder. Zorg dat de Gateway alleen via de proxy bereikbaar is, vereis sterke proxyauthenticatie, overschrijf identiteitsheaders en gebruik een beperkte lijst voor `allowUsers`.
</Warning>

## Koppelingsgedrag van Control UI

Wanneer `gateway.auth.mode = "trusted-proxy"` actief is en het verzoek de controles voor een vertrouwde proxy doorstaat, kunnen Control UI-WebSocket-sessies verbinding maken zonder identiteit voor apparaatkoppeling.

Gevolgen voor scopes:

- Control UI-WebSocket-sessies zonder apparaat maken verbinding, maar ontvangen standaard geen operatorscopes. OpenClaw wist de lijst met aangevraagde scopes naar `[]`, zodat een sessie die niet aan een goedgekeurd gekoppeld apparaat/token is gebonden, niet zelf machtigingen kan declareren.
- Als methoden na een geslaagde WebSocket-verbinding mislukken met `missing scope`, gebruik dan HTTPS zodat de browser een apparaatidentiteit kan genereren en de koppeling kan voltooien. Zie [onveilige HTTP voor Control UI](/nl/web/control-ui#insecure-http).
- Oudere configuraties die nog steeds de uitgefaseerde sleutel
  `gateway.controlUi.dangerouslyDisableDeviceAuth=true` bevatten, gebruiken de begrensde
  [upgrademigratie voor Control UI](/nl/web/control-ui#device-pairing-first-connection).

Scopebeperking door de reverse proxy: als je proxy `x-openclaw-scopes` verzendt bij het upgradeverzoek voor de Control UI-WebSocket, beperkt OpenClaw de sessiescopes tot de doorsnede van de aangevraagde scopes en de gedeclareerde scopes. Deze header kent geen scopes toe, maar beperkt alleen welke scopes de sessie kan bevatten. Wanneer `deviceAutoApprove.enabled` waar is, geldt dezelfde beperking ook voor de permanente apparaattoekenning die door [automatische apparaatgoedkeuring](#automatic-device-approval) wordt geschreven, zodat een automatisch goedgekeurd apparaat nooit meer scopes bevat dan de proxy heeft gedeclareerd.

Gevolgen:

- Koppeling is niet langer de primaire toegangscontrole voor Control UI-toegang zonder apparaat. Wanneer `deviceAutoApprove.enabled` waar is, wordt de proxy-identiteit ook de goedkeuringscontrole voor de registratie van nieuwe browserapparaten.
- Het authenticatiebeleid van je reverse proxy en `allowUsers` vormen de effectieve toegangscontrole.
- Beperk inkomend Gateway-verkeer uitsluitend tot vertrouwde proxy-IP-adressen (`gateway.trustedProxies` + firewall).

Aangepaste WebSocket-clients zijn geen Control UI-sessies. De uitgefaseerde upgrade-invoer voor Control UI verleent geen tijdelijke toegang aan willekeurige
`client.mode: "backend"`-clients of clients in CLI-vorm. Aangepaste automatisering moet
apparaatidentiteit/koppeling, het gereserveerde directe lokale backend-helperpad `client.id: "gateway-client"`
of de [Plugin voor HTTP-RPC voor beheerders](/nl/plugins/admin-http-rpc)
gebruiken wanneer een HTTP-verzoek/antwoordinterface geschikter is.

## Header voor operatorscopes

Trusted-proxy-authenticatie is een **identiteitsdragende** HTTP-modus, zodat aanroepers optioneel operatorbereiken kunnen opgeven met `x-openclaw-scopes` bij HTTP API-aanvragen.

Opmerking: WebSocket-bereiken worden bepaald door de Gateway-protocolhandshake en de koppeling van de apparaatidentiteit. Bij WebSocket-upgradeaanvragen van de Control UI is `x-openclaw-scopes` alleen een bovengrens voor de onderhandelde sessiebereiken, geen toekenning. Zie [Koppelingsgedrag van de Control UI](#control-ui-pairing-behavior).

Voorbeelden:

- `x-openclaw-scopes: operator.read`
- `x-openclaw-scopes: operator.read,operator.write`
- `x-openclaw-scopes: operator.admin,operator.write`

Gedrag:

- Wanneer de header aanwezig is, respecteert OpenClaw de opgegeven verzameling bereiken.
- Wanneer de header aanwezig maar leeg is, geeft de aanvraag **geen** operatorbereiken op.
- Wanneer de header ontbreekt, vallen normale identiteitsdragende HTTP API's terug op de standaardverzameling operatorbereiken (`operator.admin`, `operator.read`, `operator.write`, `operator.approvals`, `operator.pairing`, `operator.talk.secrets`).
- Door Gateway-authenticatie beveiligde **Plugin-HTTP-routes** zijn standaard beperkter: wanneer `x-openclaw-scopes` ontbreekt, valt hun runtimebereik alleen terug op `operator.write`.
- HTTP-aanvragen vanuit een browserorigin moeten nog steeds slagen voor `gateway.controlUi.allowedOrigins` (of de bewuste terugvalmodus met de Host-header), zelfs nadat trusted-proxy-authenticatie is geslaagd.

Praktische regel: stuur `x-openclaw-scopes` expliciet wanneer je een trusted-proxy-aanvraag beperkter wilt maken dan de standaardwaarden, of wanneer een door gateway-authenticatie beveiligde Plugin-route iets sterkers dan schrijfbereik nodig heeft.

## TLS-beëindiging en HSTS

Gebruik één TLS-beëindigingspunt en pas HSTS daar toe.

<Tabs>
  <Tab title="TLS-beëindiging bij de proxy (aanbevolen)">
    Wanneer je reverse proxy HTTPS voor `https://control.example.com` afhandelt, stel je `Strict-Transport-Security` bij de proxy voor dat domein in.

    - Geschikt voor implementaties die via internet bereikbaar zijn.
    - Houdt het certificaat- en HTTP-beveiligingsbeleid op één plek.
    - OpenClaw kan achter de proxy op loopback-HTTP blijven.

    Voorbeeldwaarde voor de header:

    ```text
    Strict-Transport-Security: max-age=31536000; includeSubDomains
    ```

  </Tab>
  <Tab title="TLS-beëindiging bij de Gateway">
    Als OpenClaw zelf rechtstreeks HTTPS aanbiedt (zonder TLS-beëindigende proxy), stel je het volgende in:

    ```json5
    {
      gateway: {
        tls: { enabled: true },
        http: {
          securityHeaders: {
            strictTransportSecurity: "max-age=31536000; includeSubDomains",
          },
        },
      },
    }
    ```

    `strictTransportSecurity` accepteert een tekenreeks als headerwaarde, of `false` om dit expliciet uit te schakelen.

  </Tab>
</Tabs>

### Richtlijnen voor de uitrol

- Begin eerst met een korte maximale leeftijd (bijvoorbeeld `max-age=300`) terwijl je het verkeer valideert.
- Verhoog deze pas naar langdurige waarden (bijvoorbeeld `max-age=31536000`) wanneer je voldoende vertrouwen hebt.
- Voeg `includeSubDomains` alleen toe als elk subdomein gereed is voor HTTPS.
- Gebruik preload alleen als je bewust aan de preloadvereisten voor je volledige verzameling domeinen voldoet.
- Lokale ontwikkeling die uitsluitend loopback gebruikt, heeft geen baat bij HSTS.

## Voorbeelden van proxyconfiguraties

<AccordionGroup>
  <Accordion title="Pomerium">
    Pomerium geeft de identiteit door in `x-pomerium-claim-email` (of andere claimheaders) en een JWT in `x-pomerium-jwt-assertion`.

    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["10.0.0.1"], // IP-adres van Pomerium
        auth: {
          mode: "trusted-proxy",
          trustedProxy: {
            userHeader: "x-pomerium-claim-email",
            requiredHeaders: ["x-pomerium-jwt-assertion"],
          },
        },
      },
    }
    ```

    Pomerium-configuratiefragment:

    ```yaml
    routes:
      - from: https://openclaw.example.com
        to: http://openclaw-gateway:18789
        policy:
          - allow:
              or:
                - email:
                    is: nick@example.com
        pass_identity_headers: true
    ```

  </Accordion>
  <Accordion title="Caddy met OAuth">
    Caddy met de Plugin `caddy-security` kan gebruikers authenticeren en identiteitsheaders doorgeven.

    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["10.0.0.1"], // IP-adres van Caddy/sidecar-proxy
        auth: {
          mode: "trusted-proxy",
          trustedProxy: {
            userHeader: "x-forwarded-user",
          },
        },
      },
    }
    ```

    Caddyfile-fragment:

    ```caddy
    openclaw.example.com {
        authenticate with oauth2_provider
        authorize with policy1

        reverse_proxy openclaw:18789 {
            header_up X-Forwarded-User {http.auth.user.email}
        }
    }
    ```

  </Accordion>
  <Accordion title="nginx + oauth2-proxy">
    oauth2-proxy authenticeert gebruikers en geeft de identiteit door in `x-auth-request-email`.

    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["10.0.0.1"], // IP-adres van nginx/oauth2-proxy
        auth: {
          mode: "trusted-proxy",
          trustedProxy: {
            userHeader: "x-auth-request-email",
          },
        },
      },
    }
    ```

    nginx-configuratiefragment:

    ```nginx
    location / {
        auth_request /oauth2/auth;
        auth_request_set $user $upstream_http_x_auth_request_email;

        proxy_pass http://openclaw:18789;
        proxy_set_header X-Auth-Request-Email $user;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
    ```

  </Accordion>
  <Accordion title="Traefik met forward-authenticatie">
    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["172.17.0.1"], // IP-adres van de Traefik-container
        auth: {
          mode: "trusted-proxy",
          trustedProxy: {
            userHeader: "x-forwarded-user",
          },
        },
      },
    }
    ```
  </Accordion>
</AccordionGroup>

## Gemengde tokenconfiguratie

Bij het starten weigert de Gateway trusted-proxy-authenticatie als er ook een gedeeld token is geconfigureerd (`gateway.auth.token` of `OPENCLAW_GATEWAY_TOKEN`). Deze twee sluiten elkaar uit, omdat een gedeeld token aanroepers op dezelfde host in staat zou stellen zich te authenticeren via een volledig ander pad dan de door de proxy geverifieerde identiteit die deze modus hoort af te dwingen.

Als het starten mislukt met een fout zoals `gateway auth mode is trusted-proxy, but a shared token is also configured`:

- Verwijder het gedeelde token wanneer je de trusted-proxy-modus gebruikt, of
- Wijzig `gateway.auth.mode` in `"token"` als je tokengebaseerde authenticatie wilt gebruiken.

Identiteitsheaders van trusted-proxy via loopback blijven bij twijfel weigeren: aanroepers op dezelfde host worden niet stilzwijgend als proxygebruikers geauthenticeerd. Interne OpenClaw-aanroepers die de proxy omzeilen, kunnen zich in plaats daarvan authenticeren met `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD`. Terugvallen op een token blijft bewust niet ondersteund in de trusted-proxy-modus.

## Beveiligingschecklist

Controleer het volgende voordat je trusted-proxy-authenticatie inschakelt:

- [ ] **De proxy is het enige pad**: De Gateway-poort is voor alles behalve je proxy door een firewall afgeschermd.
- [ ] **trustedProxies is minimaal**: Alleen de daadwerkelijke IP-adressen van je proxy, niet volledige subnetten.
- [ ] **Een loopback-proxybron is een bewuste keuze**: trusted-proxy-authenticatie weigert standaard aanvragen van een loopback-bron, tenzij `gateway.auth.trustedProxy.allowLoopback` expliciet is ingeschakeld voor een proxy op dezelfde host.
- [ ] **De proxy verwijdert headers**: Je proxy overschrijft `x-forwarded-*`-headers van clients (en voegt er niet aan toe).
- [ ] **TLS-beëindiging**: Je proxy handelt TLS af; gebruikers maken verbinding via HTTPS.
- [ ] **allowedOrigins is expliciet**: De Control UI buiten loopback gebruikt expliciete `gateway.controlUi.allowedOrigins`.
- [ ] **allowUsers is ingesteld** (aanbevolen): Beperk de toegang tot bekende gebruikers in plaats van iedereen met geldige authenticatie toe te laten.
- [ ] **Geen gemengde tokenconfiguratie**: Stel niet zowel `gateway.auth.token` als `gateway.auth.mode: "trusted-proxy"` in.
- [ ] **Lokale terugval op een wachtwoord is privé**: Als je `gateway.auth.password` configureert voor rechtstreekse interne aanroepers, houd je de Gateway-poort door een firewall afgeschermd zodat externe clients buiten de proxy deze niet rechtstreeks kunnen bereiken.
- [ ] **Automatische goedkeuring van apparaten is een bewuste keuze**: Als `deviceAutoApprove.enabled` waar is, beschouw je de accountbeveiliging van de reverse proxy als de grens voor apparaatregistratie en houd je de lijst met toegekende bereiken minimaal en zonder beheerdersrechten.

## Beveiligingsaudit

`openclaw security audit` markeert trusted-proxy-authenticatie met een bevinding van **kritieke** ernst. Dit is opzettelijk; het herinnert je eraan dat je de beveiliging aan je proxyconfiguratie delegeert.

De audit controleert op:

- Algemene waarschuwing/kritieke herinnering voor `gateway.trusted_proxy_auth`.
- Ontbrekende configuratie voor `trustedProxies`.
- Ontbrekende configuratie voor `userHeader`.
- Lege `allowUsers` (staat elke geauthenticeerde gebruiker toe).
- Ingeschakelde `allowLoopback` voor proxybronnen op dezelfde host.
- Ingeschakelde automatische goedkeuring van browserapparaten (delegeert de koppeling van nieuwe apparaten aan de proxy-identiteit).

Afzonderlijke bevindingen die niet specifiek zijn voor trusted-proxy zijn ook van toepassing wanneer de Control UI beschikbaar wordt gesteld: een jokerteken of ontbrekende `gateway.controlUi.allowedOrigins`, en terugval op de origin van de Host-header.

## Problemen oplossen

<AccordionGroup>
  <Accordion title="trusted_proxy_untrusted_source">
    De aanvraag kwam niet van een IP-adres in `gateway.trustedProxies`. Controleer het volgende:

    - Klopt het IP-adres van de proxy? (IP-adressen van Docker-containers kunnen veranderen.)
    - Staat er een load balancer vóór je proxy?
    - Gebruik `docker inspect` of `kubectl get pods -o wide` om de daadwerkelijke IP-adressen te vinden.

  </Accordion>
  <Accordion title="trusted_proxy_loopback_source">
    OpenClaw heeft een trusted-proxy-aanvraag van een loopback-bron geweigerd.

    Controleer het volgende:

    - Maakt de proxy verbinding vanaf `127.0.0.1` / `::1`?
    - Probeer je trusted-proxy-authenticatie te gebruiken met een loopback-reverse-proxy op dezelfde host?

    Oplossing:

    - Gebruik bij voorkeur token-/wachtwoordauthenticatie voor interne clients op dezelfde host die niet via de proxy gaan, of
    - Leid het verkeer via een vertrouwd proxyadres dat geen loopback-adres is en houd dat IP-adres in `gateway.trustedProxies`, of
    - Stel voor een bewust gebruikte reverse proxy op dezelfde host `gateway.auth.trustedProxy.allowLoopback = true` in, houd het loopback-adres in `gateway.trustedProxies` en zorg ervoor dat de proxy identiteitsheaders verwijdert of overschrijft.

  </Accordion>
  <Accordion title="trusted_proxy_local_interface_source / trusted_proxy_local_interface_check_failed">
    Het bron-IP-adres van de aanvraag kwam overeen met een van de eigen netwerkinterfaceadressen zonder loopback van de Gateway-host (niet met de proxy), als bescherming tegen vervalst verkeer vanaf dezelfde host op tailnets of Docker-bridge-netwerken. `..._check_failed` betekent dat tijdens de detectie van interfaces zelf een fout is opgetreden, zodat OpenClaw standaard weigert.

    Controleer het volgende:

    - Verstuurt een proces op de Gateway-host zelf rechtstreeks identiteitsheaders en omzeilt het daarbij de proxy?
    - Draait de proxy in dezelfde netwerknaamruimte als de Gateway, met een IP-adres dat ook als lokale interface wordt weergegeven?

    Oplossing: leid proxyverkeer via een adres dat niet ook lokaal aan de Gateway-host is gebonden, of gebruik `allowLoopback` alleen voor een echte proxyconfiguratie op dezelfde host.

  </Accordion>
  <Accordion title="trusted_proxy_user_missing">
    De gebruikersheader was leeg of ontbrak. Controleer het volgende:

    - Is je proxy geconfigureerd om identiteitsheaders door te geven?
    - Klopt de naam van de header? (niet hoofdlettergevoelig, maar de spelling moet kloppen)
    - Is de gebruiker daadwerkelijk bij de proxy geauthenticeerd?

  </Accordion>
  <Accordion title="trusted_proxy_missing_header_*">
    Een vereiste header ontbrak. Controleer het volgende:

    - De configuratie van je proxy voor die specifieke headers.
    - Of headers ergens in de keten worden verwijderd.

  </Accordion>
  <Accordion title="trusted_proxy_user_not_allowed">
    De gebruiker is geauthenticeerd, maar staat niet in `allowUsers`. Voeg de gebruiker toe of verwijder de toelatingslijst.
  </Accordion>
  <Accordion title="trusted_proxy_no_proxies_configured / trusted_proxy_config_missing">
    `gateway.auth.mode` is `"trusted-proxy"`, maar `gateway.trustedProxies` is leeg, of `gateway.auth.trustedProxy` zelf ontbreekt. Elk verzoek wordt geweigerd totdat beide zijn ingesteld.
  </Accordion>
  <Accordion title="trusted_proxy_origin_not_allowed">
    Authenticatie via een vertrouwde proxy is geslaagd, maar de browserheader `Origin` doorstond de oorsprongscontroles van de Control UI niet.

    Controleer het volgende:

    - `gateway.controlUi.allowedOrigins` bevat de exacte browseroorsprong.
    - Je vertrouwt niet op jokertekenoorsprongen, tenzij je bewust alles wilt toestaan.
    - Als je bewust de terugvalmodus met de Host-header gebruikt, is `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback=true` doelbewust ingesteld.

  </Accordion>
  <Accordion title="Verbinding slaagt, maar methoden melden een ontbrekend bereik">
    De WebSocket maakt verbinding, maar `chat.history`, `sessions.list` of
    `models.list` mislukt met `missing scope: operator.read`.

    Veelvoorkomende oorzaken:

    - Control UI-sessie zonder apparaat: authenticatie via een vertrouwde proxy kan de WebSocket-verbinding toelaten zonder apparaatidentiteit, maar OpenClaw wist ontworpen gedrag de bereiken van sessies zonder apparaat.
    - Aangepaste backendclient: de uitgefaseerde upgrade-invoer van de Control UI verleent nooit toegang aan willekeurige backendclients of WebSocket-clients in CLI-vorm.
    - Te beperkte `x-openclaw-scopes`: als je proxy deze header injecteert in het WebSocket-upgradeverzoek van de Control UI, worden de sessiebereiken tot die verzameling beperkt. Een lege headerwaarde levert geen bereiken op.

    Oplossing:

    - Gebruik voor de Control UI HTTPS, zodat de browser een apparaatidentiteit kan genereren en de koppeling kan voltooien.
    - Gebruik voor aangepaste automatisering apparaatidentiteit/koppeling, het gereserveerde directe lokale backend-helperpad `gateway-client` of [HTTP-RPC voor beheerders](/nl/plugins/admin-http-rpc).
    - Voeg de uitgefaseerde sleutel `gateway.controlUi.dangerouslyDisableDeviceAuth` niet toe aan de huidige configuratie. Oudere installaties gebruiken automatisch de eenmalige migratie voor zelfkoppeling.

  </Accordion>
  <Accordion title="WebSocket werkt nog steeds niet">
    Zorg ervoor dat je proxy:

    - WebSocket-upgrades ondersteunt (`Upgrade: websocket`, `Connection: upgrade`).
    - De identiteitsheaders doorgeeft bij WebSocket-upgradeverzoeken (niet alleen bij HTTP).
    - Geen afzonderlijk authenticatiepad voor WebSocket-verbindingen heeft.

  </Accordion>
</AccordionGroup>

## Migratie vanaf tokenauthenticatie

<Steps>
  <Step title="De proxy configureren">
    Configureer je proxy om gebruikers te authenticeren en headers door te geven.
  </Step>
  <Step title="De proxy afzonderlijk testen">
    Test de proxyconfiguratie afzonderlijk (curl met headers).
  </Step>
  <Step title="De OpenClaw-configuratie bijwerken">
    Werk de OpenClaw-configuratie bij met authenticatie via een vertrouwde proxy.
  </Step>
  <Step title="De Gateway opnieuw starten">
    Start de Gateway opnieuw.
  </Step>
  <Step title="WebSocket testen">
    Test WebSocket-verbindingen vanuit de Control UI.
  </Step>
  <Step title="Controleren">
    Voer `openclaw security audit` uit en beoordeel de bevindingen.
  </Step>
</Steps>

## Gerelateerd

- [Configuratie](/nl/gateway/configuration) — configuratiereferentie
- [Operatorbereiken](/nl/gateway/operator-scopes) — rollen, bereiken en goedkeuringscontroles
- [Externe toegang](/nl/gateway/remote) — andere patronen voor externe toegang
- [Beveiliging](/nl/gateway/security) — volledige beveiligingshandleiding
- [Tailscale](/nl/gateway/tailscale) — eenvoudiger alternatief voor toegang uitsluitend via het tailnet
