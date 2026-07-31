---
read_when:
    - OpenClaw hinter einem identitätsbewussten Proxy ausführen
    - Pomerium, Caddy oder nginx mit OAuth vor OpenClaw einrichten
    - Beheben von WebSocket-1008-Fehlern „unauthorized“ bei Reverse-Proxy-Konfigurationen
    - Festlegen, wo HSTS und andere HTTP-Härtungsheader gesetzt werden sollen
sidebarTitle: Trusted proxy auth
summary: Gateway-Authentifizierung an einen vertrauenswürdigen Reverse-Proxy delegieren (Pomerium, Caddy, nginx + OAuth)
title: Authentifizierung über vertrauenswürdigen Proxy
x-i18n:
    generated_at: "2026-07-26T17:49:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 39bf8f12b3ae95f53b21bfed12deb1c8ed8f767711955bbee52c74538052a89f
    source_path: gateway/trusted-proxy-auth.md
    workflow: 16
---

<Warning>
**Sicherheitskritische Funktion.** Dieser Modus delegiert die Authentifizierung vollständig an Ihren Reverse-Proxy. Eine Fehlkonfiguration kann Ihren Gateway unbefugtem Zugriff aussetzen. Lesen Sie diese Seite sorgfältig, bevor Sie die Funktion aktivieren.
</Warning>

## Verwendung

- Sie betreiben OpenClaw hinter einem **identitätsbewussten Proxy** (Pomerium, Caddy + OAuth, nginx + oauth2-proxy, Traefik + Forward Auth).
- Ihr Proxy übernimmt die gesamte Authentifizierung und übermittelt die Benutzeridentität über Header.
- Sie verwenden eine Kubernetes- oder Containerumgebung, in der der Proxy der einzige Pfad zum Gateway ist.
- WebSocket-Fehler vom Typ `1008 unauthorized` treten auf, weil Browser keine Token in WS-Nutzdaten übermitteln können.

## Nicht verwenden

- Ihr Proxy authentifiziert keine Benutzer, sondern dient lediglich als TLS-Terminator oder Lastverteiler.
- Es gibt einen Pfad zum Gateway, der den Proxy umgeht, etwa durch Firewall-Lücken oder internen Netzwerkzugriff.
- Sie sind nicht sicher, ob Ihr Proxy weitergeleitete Header ordnungsgemäß entfernt oder überschreibt.
- Sie benötigen nur persönlichen Einzelbenutzerzugriff; erwägen Sie stattdessen Tailscale Serve + Loopback.

## Funktionsweise

<Steps>
  <Step title="Proxy authentifiziert den Benutzer">
    Ihr Reverse-Proxy authentifiziert Benutzer (OAuth, OIDC, SAML usw.).
  </Step>
  <Step title="Proxy fügt einen Identitäts-Header hinzu">
    Der Proxy fügt einen Header mit der authentifizierten Benutzeridentität hinzu (z. B. `x-forwarded-user: nick@example.com`).
  </Step>
  <Step title="Gateway überprüft die vertrauenswürdige Quelle">
    OpenClaw prüft, ob die Anfrage von einer **vertrauenswürdigen Proxy-IP-Adresse** (`gateway.trustedProxies`) stammt und nicht von der eigenen Loopback- oder lokalen Schnittstellenadresse des Gateways.
  </Step>
  <Step title="Gateway extrahiert die Identität">
    OpenClaw liest die erforderlichen Header und anschließend die Benutzeridentität aus dem konfigurierten Header.
  </Step>
  <Step title="Autorisierung">
    Wenn alle Prüfungen erfolgreich sind und der Benutzer `allowUsers` erfüllt, sofern dies festgelegt ist, wird die Anfrage autorisiert.
  </Step>
</Steps>

## Konfiguration

```json5
{
  gateway: {
    // Bei Trusted-Proxy-Authentifizierung darf die Quell-IP des Proxys standardmäßig keine Loopback-Adresse sein
    bind: "lan",

    // KRITISCH: Fügen Sie hier ausschließlich die IP-Adresse(n) Ihres Proxys hinzu
    trustedProxies: ["10.0.0.1", "172.17.0.1"],

    auth: {
      mode: "trusted-proxy",
      trustedProxy: {
        // Header mit der authentifizierten Benutzeridentität (erforderlich)
        userHeader: "x-forwarded-user",

        // Optional: Header, die vorhanden sein MÜSSEN (Proxy-Überprüfung)
        requiredHeaders: ["x-forwarded-proto", "x-forwarded-host"],

        // Optional: auf bestimmte Benutzer beschränken (leer = alle zulassen)
        allowUsers: ["nick@example.com", "admin@company.org"],

        // Optional: nach ausdrücklicher Aktivierung einen Loopback-Proxy auf demselben Host zulassen
        allowLoopback: false,

        // Optional: authentifizierten Proxy-Benutzern die Registrierung neuer Browsergeräte erlauben
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
**Laufzeitregeln in Auswertungsreihenfolge**

1. Die Quell-IP-Adresse der Anfrage muss `gateway.trustedProxies` entsprechen (CIDR-fähig), andernfalls wird sie abgelehnt (`trusted_proxy_untrusted_source`).
2. Anfragen aus Loopback-Quellen (`127.0.0.1`, `::1`) werden abgelehnt, sofern nicht `gateway.auth.trustedProxy.allowLoopback = true` gilt und die Loopback-Adresse außerdem in `trustedProxies` enthalten ist (`trusted_proxy_loopback_source`). Diese Prüfung erfolgt vor den Header-Prüfungen. Daher schlägt eine Loopback-Quelle auf diese Weise fehl, selbst wenn zusätzlich erforderliche Header fehlen.
3. Nicht-Loopback-Quellen, die mit einer lokalen Netzwerkschnittstellenadresse des Gateway-Hosts übereinstimmen, werden zum Schutz vor Spoofing abgelehnt (`trusted_proxy_local_interface_source`). Schlägt die Schnittstellenerkennung selbst fehl, wird die Anfrage ebenfalls abgelehnt (`trusted_proxy_local_interface_check_failed`).
4. `requiredHeaders` und `userHeader` müssen vorhanden und dürfen nicht leer sein.
5. `allowUsers` muss, sofern nicht leer, den extrahierten Benutzer enthalten.

**Nachweise durch weitergeleitete Header haben für den lokalen direkten Fallback Vorrang vor der Loopback-Lokalität.** Wenn eine Anfrage über Loopback eingeht, aber einen `Forwarded`-, einen beliebigen `X-Forwarded-*`- oder einen `X-Real-IP`-Header enthält, schließt dieser Nachweis sie vom lokalen direkten Passwort-Fallback und von der Geräteidentitätsprüfung aus, obwohl die Trusted-Proxy-Authentifizierung aufgrund der Loopback-Quelle weiterhin fehlschlägt.

`allowLoopback` vertraut lokalen Prozessen auf dem Gateway-Host im selben Maß wie dem Reverse-Proxy. Aktivieren Sie diese Option nur, wenn der Gateway weiterhin durch eine Firewall vor direktem Remotezugriff geschützt ist und der lokale Proxy vom Client bereitgestellte Identitäts-Header entfernt oder überschreibt.

Interne Gateway-Clients, die nicht über den Reverse-Proxy kommunizieren, sollten `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD` und keine Trusted-Proxy-Identitäts-Header verwenden. Control-UI-Bereitstellungen ohne Loopback benötigen weiterhin eine explizite Angabe von `gateway.controlUi.allowedOrigins`.
</Warning>

### Konfigurationsreferenz

<ParamField path="gateway.trustedProxies" type="string[]" required>
  Array vertrauenswürdiger Proxy-IP-Adressen oder CIDR-Bereiche. Anfragen von anderen IP-Adressen werden abgelehnt.
</ParamField>
<ParamField path="gateway.auth.mode" type="string" required>
  Muss `"trusted-proxy"` sein.
</ParamField>
<ParamField path="gateway.auth.trustedProxy.userHeader" type="string" required>
  Name des Headers, der die authentifizierte Benutzeridentität enthält.
</ParamField>
<ParamField path="gateway.auth.trustedProxy.requiredHeaders" type="string[]">
  Zusätzliche Header, die vorhanden sein müssen, damit die Anfrage als vertrauenswürdig gilt.
</ParamField>
<ParamField path="gateway.auth.trustedProxy.allowUsers" type="string[]">
  Zulassungsliste der Benutzeridentitäten. Eine leere Liste lässt alle authentifizierten Benutzer zu.
</ParamField>
<ParamField path="gateway.auth.trustedProxy.allowLoopback" type="boolean" default="false">
  Optionale Unterstützung für Loopback-Reverse-Proxys auf demselben Host.
</ParamField>
<ParamField path="gateway.auth.trustedProxy.deviceAutoApprove.enabled" type="boolean" default="false">
  Neue Geräteidentitäten der Control UI und von WebChat nach der Trusted-Proxy-Authentifizierung automatisch genehmigen.
</ParamField>
<ParamField path="gateway.auth.trustedProxy.deviceAutoApprove.scopes" type="string[]" default='["operator.read", "operator.write", "operator.approvals"]'>
  Maximale Berechtigungsbereiche, die einem automatisch genehmigten Browsergerät gewährt werden. Wird `operator.admin` ausdrücklich aufgeführt, kann jeder über den Proxy authentifizierte Benutzer automatisch vollständige Administratorrechte für ein Gerät anfordern. Anfragen ohne Berechtigungsbereiche erhalten automatisch vollständige Administratorrechte. Außerdem werden der KRITISCHE Sicherheitsprüfungsbefund `gateway.trusted_proxy_device_auto_approve_admin` sowie eine Gateway-Warnung beim Start ausgelöst.
</ParamField>

<Warning>
Aktivieren Sie `allowLoopback` nur, wenn der lokale Reverse-Proxy die vorgesehene Vertrauensgrenze darstellt. Jeder lokale Prozess, der eine Verbindung zum Gateway herstellen kann, kann versuchen, Proxy-Identitäts-Header zu senden. Beschränken Sie den direkten Gateway-Zugriff daher auf den Host und verlangen Sie vom Proxy kontrollierte Header wie `x-forwarded-proto` oder einen signierten Bestätigungs-Header, sofern Ihr Proxy dies unterstützt.
</Warning>

## Automatische Gerätegenehmigung

Die Trusted-Proxy-Authentifizierung kann optional die Proxy-Identität als Genehmigungsgrenze für neue Browsergeräte verwenden:

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

Der Standardwert ist `enabled: false`. Wenn die Option aktiviert ist, gelten alle folgenden Regeln:

1. Der WebSocket muss über die Methode `trusted-proxy` mit einer nicht leeren Benutzeridentität authentifiziert worden sein, die `allowUsers` erfüllt, wenn eine Zulassungsliste konfiguriert ist. Verbindungen über Token, Passwort, Tailscale oder ohne Authentifizierung verwenden diese Richtlinie niemals.
2. Nur ein neues Browsergerät der Control UI oder von WebChat kann automatisch genehmigt werden. Jede Anfrage für ein vorhandenes Gerät, einschließlich einer Erweiterung der Berechtigungsbereiche, bleibt zur manuellen Genehmigung mit `openclaw devices approve <requestId>` ausstehend.
3. Das Gerät wird mit der Rolle `operator` genehmigt. Wenn die Verbindungsanfrage Berechtigungsbereiche enthält, entspricht die Gewährung exakt der Schnittmenge aus den angeforderten Berechtigungsbereichen und `deviceAutoApprove.scopes`. Lässt die Anfrage die Berechtigungsbereiche aus, wird die konfigurierte Liste gewährt. Wenn diese Liste nicht angegeben ist, werden standardmäßig `operator.read`, `operator.write` und `operator.approvals` verwendet. Die resultierende Gewährung wird anschließend zusätzlich durch den Proxy-Header [`x-openclaw-scopes`](#control-ui-pairing-behavior) der Verbindung begrenzt, sofern dieser vorhanden ist. Dadurch begrenzt ein Proxy, der die Berechtigungsbereiche eines Benutzers einschränkt, auch die **dauerhafte** Gerätegewährung und nicht nur die Sitzung; ein vorhandener, aber leerer Header gewährt keine Berechtigungsbereiche. Diese Begrenzung gilt auch, wenn der Client seine eigene Liste der Berechtigungsbereiche auslässt.
4. `operator.admin` ist nur zulässig, wenn es ausdrücklich in `deviceAutoApprove.scopes` aufgeführt wird. Ist es aufgeführt, kann jeder über den Proxy authentifizierte Benutzer vollständige Administratorrechte für ein neues Browsergerät anfordern und automatisch erhalten. Anfragen ohne Berechtigungsbereiche erhalten automatisch vollständige Administratorrechte. `openclaw security audit` meldet den KRITISCHEN Befund `gateway.trusted_proxy_device_auto_approve_admin`, und der Gateway protokolliert beim Start einmalig eine Warnung. Bevorzugen Sie die manuelle Administratorgenehmigung mit `openclaw devices approve` oder `openclaw devices rotate`, bis identitätsspezifische Rollen verfügbar sind.

<Warning>
Durch Aktivieren dieser Option wird die Registrierung neuer Browsergeräte vollständig an die Reverse-Proxy-Identität delegiert. Mit einem kompromittierten Proxy-Konto kann ein dauerhaftes Gerät mit allen konfigurierten Berechtigungsbereichen registriert werden. Wird `operator.admin` aufgeführt, erhält dieses Gerät ohne manuelle Genehmigung vollständige Administratorrechte. Sorgen Sie dafür, dass der Gateway ausschließlich über den Proxy erreichbar ist, verlangen Sie eine starke Proxy-Authentifizierung, überschreiben Sie Identitäts-Header und verwenden Sie eine eng gefasste Liste für `allowUsers`.
</Warning>

## Kopplungsverhalten der Control UI

Wenn `gateway.auth.mode = "trusted-proxy"` aktiv ist und die Anfrage die Trusted-Proxy-Prüfungen besteht, können Control-UI-WebSocket-Sitzungen ohne Kopplungsidentität des Geräts eine Verbindung herstellen.

Auswirkungen auf Berechtigungsbereiche:

- Control-UI-WebSocket-Sitzungen ohne Gerät stellen eine Verbindung her, erhalten standardmäßig jedoch keine Operator-Berechtigungsbereiche. OpenClaw leert die Liste der angeforderten Berechtigungsbereiche zu `[]`, sodass eine Sitzung, die nicht an ein genehmigtes gekoppeltes Gerät oder Token gebunden ist, keine Berechtigungen selbst deklarieren kann.
- Wenn Methoden nach einer erfolgreichen WebSocket-Verbindung mit `missing scope` fehlschlagen, verwenden Sie HTTPS, damit der Browser eine Geräteidentität erzeugen und die Kopplung abschließen kann. Siehe [Unsicheres HTTP der Control UI](/de/web/control-ui#insecure-http).
- Ältere Konfigurationen, die noch den außer Betrieb genommenen Schlüssel
  `gateway.controlUi.dangerouslyDisableDeviceAuth=true` enthalten, verwenden die begrenzte
  [Upgrade-Migration der Control UI](/de/web/control-ui#device-pairing-first-connection).

Begrenzung der Berechtigungsbereiche durch den Reverse-Proxy: Wenn Ihr Proxy beim WebSocket-Upgrade der Control UI `x-openclaw-scopes` sendet, begrenzt OpenClaw die Berechtigungsbereiche der Sitzung auf die Schnittmenge aus den angeforderten und den deklarierten Berechtigungsbereichen. Dieser Header gewährt keine Berechtigungsbereiche, sondern schränkt lediglich ein, welche Berechtigungsbereiche die Sitzung besitzen kann. Wenn `deviceAutoApprove.enabled` wahr ist, gilt dieselbe Begrenzung auch für die dauerhafte Gerätegewährung, die durch die [automatische Gerätegenehmigung](#automatic-device-approval) geschrieben wird. Ein automatisch genehmigtes Gerät besitzt somit niemals mehr Berechtigungsbereiche, als der Proxy deklariert hat.

Auswirkungen:

- Die Kopplung ist nicht mehr die primäre Zugriffssperre für den gerätelosen Zugriff auf die Control UI. Wenn `deviceAutoApprove.enabled` wahr ist, wird die Proxy-Identität außerdem zur Genehmigungssperre für die Registrierung neuer Browsergeräte.
- Die Authentifizierungsrichtlinie Ihres Reverse-Proxys und `allowUsers` bilden die effektive Zugriffskontrolle.
- Beschränken Sie den Gateway-Eingang ausschließlich auf vertrauenswürdige Proxy-IP-Adressen (`gateway.trustedProxies` + Firewall).

Benutzerdefinierte WebSocket-Clients sind keine Control-UI-Sitzungen. Die außer Betrieb genommene Upgrade-Eingabe der Control UI gewährt beliebigen
`client.mode: "backend"`- oder CLI-artigen Clients keinen temporären Zugriff. Benutzerdefinierte Automatisierungen sollten
die Geräteidentität/Kopplung, den reservierten direkten lokalen Backend-Hilfspfad `client.id: "gateway-client"`
oder das [Admin-HTTP-RPC-Plugin](/de/plugins/admin-http-rpc) verwenden,
wenn eine HTTP-Anfrage-/Antwortschnittstelle besser geeignet ist.

## Header für Operator-Berechtigungsbereiche

Die Trusted-Proxy-Authentifizierung ist ein **identitätstragender** HTTP-Modus, daher können Aufrufer bei HTTP-API-Anfragen optional Operator-Berechtigungsumfänge mit `x-openclaw-scopes` deklarieren.

Hinweis: WebSocket-Berechtigungsumfänge werden durch den Gateway-Protokoll-Handshake und die Bindung der Geräteidentität bestimmt. Bei WebSocket-Upgrade-Anfragen der Control UI ist `x-openclaw-scopes` lediglich eine Obergrenze für die ausgehandelten Sitzungsberechtigungsumfänge, keine Gewährung. Siehe [Kopplungsverhalten der Control UI](#control-ui-pairing-behavior).

Beispiele:

- `x-openclaw-scopes: operator.read`
- `x-openclaw-scopes: operator.read,operator.write`
- `x-openclaw-scopes: operator.admin,operator.write`

Verhalten:

- Wenn der Header vorhanden ist, berücksichtigt OpenClaw die deklarierte Menge an Berechtigungsumfängen.
- Wenn der Header vorhanden, aber leer ist, deklariert die Anfrage **keine** Operator-Berechtigungsumfänge.
- Wenn der Header fehlt, greifen normale identitätstragende HTTP-APIs auf die standardmäßige Menge an Operator-Berechtigungsumfängen zurück (`operator.admin`, `operator.read`, `operator.write`, `operator.approvals`, `operator.pairing`, `operator.talk.secrets`).
- Durch Gateway-Authentifizierung geschützte **Plugin-HTTP-Routen** sind standardmäßig restriktiver: Wenn `x-openclaw-scopes` fehlt, greift ihr Laufzeit-Berechtigungsumfang ausschließlich auf `operator.write` zurück.
- HTTP-Anfragen mit Browser-Ursprung müssen auch nach erfolgreicher Trusted-Proxy-Authentifizierung weiterhin `gateway.controlUi.allowedOrigins` (oder den bewusst aktivierten Host-Header-Fallbackmodus) passieren.

Praxisregel: Senden Sie `x-openclaw-scopes` ausdrücklich, wenn eine Trusted-Proxy-Anfrage restriktiver als die Standardwerte sein soll oder wenn eine durch Gateway-Authentifizierung geschützte Plugin-Route einen stärkeren als den Schreibberechtigungsumfang benötigt.

## TLS-Terminierung und HSTS

Verwenden Sie einen einzigen TLS-Terminierungspunkt und wenden Sie HSTS dort an.

<Tabs>
  <Tab title="TLS-Terminierung am Proxy (empfohlen)">
    Wenn Ihr Reverse-Proxy HTTPS für `https://control.example.com` verarbeitet, setzen Sie `Strict-Transport-Security` am Proxy für diese Domain.

    - Gut für mit dem Internet verbundene Bereitstellungen geeignet.
    - Hält Zertifikat und Richtlinien zur HTTP-Absicherung an einer Stelle.
    - OpenClaw kann hinter dem Proxy weiterhin Loopback-HTTP verwenden.

    Beispiel für einen Header-Wert:

    ```text
    Strict-Transport-Security: max-age=31536000; includeSubDomains
    ```

  </Tab>
  <Tab title="TLS-Terminierung am Gateway">
    Wenn OpenClaw HTTPS selbst direkt bereitstellt (ohne TLS-terminierenden Proxy), legen Sie Folgendes fest:

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

    `strictTransportSecurity` akzeptiert einen Header-Wert als Zeichenfolge oder `false`, um die Funktion ausdrücklich zu deaktivieren.

  </Tab>
</Tabs>

### Hinweise zur Einführung

- Beginnen Sie zunächst mit einer kurzen maximalen Gültigkeitsdauer (zum Beispiel `max-age=300`), während Sie den Datenverkehr validieren.
- Erhöhen Sie sie erst dann auf langfristige Werte (zum Beispiel `max-age=31536000`), wenn eine hohe Sicherheit besteht.
- Fügen Sie `includeSubDomains` nur hinzu, wenn jede Subdomain für HTTPS bereit ist.
- Verwenden Sie Preloading nur, wenn Sie die Preload-Anforderungen für Ihre gesamte Domainmenge bewusst erfüllen.
- Eine ausschließlich über Loopback erreichbare lokale Entwicklungsumgebung profitiert nicht von HSTS.

## Beispiele für die Proxy-Einrichtung

<AccordionGroup>
  <Accordion title="Pomerium">
    Pomerium übergibt die Identität in `x-pomerium-claim-email` (oder anderen Claim-Headern) und ein JWT in `x-pomerium-jwt-assertion`.

    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["10.0.0.1"], // IP-Adresse von Pomerium
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

    Pomerium-Konfigurationsausschnitt:

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
  <Accordion title="Caddy mit OAuth">
    Caddy kann Benutzer mit dem Plugin `caddy-security` authentifizieren und Identitäts-Header übergeben.

    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["10.0.0.1"], // IP-Adresse des Caddy-/Sidecar-Proxys
        auth: {
          mode: "trusted-proxy",
          trustedProxy: {
            userHeader: "x-forwarded-user",
          },
        },
      },
    }
    ```

    Caddyfile-Ausschnitt:

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
    oauth2-proxy authentifiziert Benutzer und übergibt die Identität in `x-auth-request-email`.

    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["10.0.0.1"], // IP-Adresse von nginx/oauth2-proxy
        auth: {
          mode: "trusted-proxy",
          trustedProxy: {
            userHeader: "x-auth-request-email",
          },
        },
      },
    }
    ```

    nginx-Konfigurationsausschnitt:

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
  <Accordion title="Traefik mit vorgeschalteter Authentifizierung">
    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["172.17.0.1"], // IP-Adresse des Traefik-Containers
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

## Gemischte Token-Konfiguration

Der Gateway-Start lehnt die Trusted-Proxy-Authentifizierung ab, wenn zusätzlich ein gemeinsam verwendetes Token konfiguriert ist (`gateway.auth.token` oder `OPENCLAW_GATEWAY_TOKEN`). Beide schließen sich gegenseitig aus, weil ein gemeinsam verwendetes Token es Aufrufern auf demselben Host ermöglichen würde, sich über einen völlig anderen Pfad als die von diesem Modus durchgesetzte, vom Proxy verifizierte Identität zu authentifizieren.

Wenn der Start mit einem Fehler wie `gateway auth mode is trusted-proxy, but a shared token is also configured` fehlschlägt:

- Entfernen Sie das gemeinsam verwendete Token, wenn Sie den Trusted-Proxy-Modus verwenden, oder
- ändern Sie `gateway.auth.mode` in `"token"`, wenn Sie eine tokenbasierte Authentifizierung verwenden möchten.

Trusted-Proxy-Identitäts-Header über Loopback schlagen weiterhin sicher fehl: Aufrufer auf demselben Host werden nicht stillschweigend als Proxy-Benutzer authentifiziert. Interne OpenClaw-Aufrufer, die den Proxy umgehen, können sich stattdessen mit `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD` authentifizieren. Ein Token-Fallback wird im Trusted-Proxy-Modus weiterhin bewusst nicht unterstützt.

## Sicherheitscheckliste

Prüfen Sie vor dem Aktivieren der Trusted-Proxy-Authentifizierung Folgendes:

- [ ] **Der Proxy ist der einzige Pfad**: Der Gateway-Port ist durch eine Firewall für alles außer Ihrem Proxy gesperrt.
- [ ] **trustedProxies ist minimal**: Nur die tatsächlichen IP-Adressen Ihres Proxys, keine ganzen Subnetze.
- [ ] **Eine Loopback-Proxyquelle ist beabsichtigt**: Die Trusted-Proxy-Authentifizierung schlägt bei Anfragen aus einer Loopback-Quelle sicher fehl, sofern `gateway.auth.trustedProxy.allowLoopback` nicht ausdrücklich für einen Proxy auf demselben Host aktiviert ist.
- [ ] **Der Proxy entfernt Header**: Ihr Proxy überschreibt von Clients stammende `x-forwarded-*`-Header, statt sie anzuhängen.
- [ ] **TLS-Terminierung**: Ihr Proxy verarbeitet TLS; Benutzer stellen die Verbindung über HTTPS her.
- [ ] **allowedOrigins ist ausdrücklich festgelegt**: Eine nicht über Loopback erreichbare Control UI verwendet ausdrücklich `gateway.controlUi.allowedOrigins`.
- [ ] **allowUsers ist festgelegt** (empfohlen): Beschränken Sie den Zugriff auf bekannte Benutzer, statt jeden authentifizierten Benutzer zuzulassen.
- [ ] **Keine gemischte Token-Konfiguration**: Legen Sie nicht sowohl `gateway.auth.token` als auch `gateway.auth.mode: "trusted-proxy"` fest.
- [ ] **Der lokale Passwort-Fallback ist privat**: Wenn Sie `gateway.auth.password` für interne direkte Aufrufer konfigurieren, schützen Sie den Gateway-Port durch eine Firewall, damit entfernte Clients außerhalb des Proxys nicht direkt darauf zugreifen können.
- [ ] **Die automatische Gerätegenehmigung ist beabsichtigt**: Wenn `deviceAutoApprove.enabled` wahr ist, behandeln Sie die Sicherheit des Reverse-Proxy-Kontos als Grenze für die Geräteregistrierung und halten Sie die Liste der gewährten Berechtigungsumfänge auf Nicht-Administratorrechte beschränkt und minimal.

## Sicherheitsprüfung

`openclaw security audit` kennzeichnet die Trusted-Proxy-Authentifizierung mit einem Befund des Schweregrads **kritisch**. Dies ist beabsichtigt; es erinnert Sie daran, dass Sie die Sicherheit an Ihre Proxy-Einrichtung delegieren.

Die Prüfung kontrolliert Folgendes:

- Grundlegende `gateway.trusted_proxy_auth`-Warnung bzw. kritische Erinnerung.
- Fehlende `trustedProxies`-Konfiguration.
- Fehlende `userHeader`-Konfiguration.
- Leeres `allowUsers` (lässt jeden authentifizierten Benutzer zu).
- Aktiviertes `allowLoopback` für Proxyquellen auf demselben Host.
- Aktivierte automatische Genehmigung von Browsergeräten (delegiert die Kopplung neuer Geräte an die Proxy-Identität).

Separate, nicht speziell auf Trusted-Proxy bezogene Befunde gelten ebenfalls immer dann, wenn die Control UI verfügbar gemacht wird: Platzhalterwert oder fehlendes `gateway.controlUi.allowedOrigins` sowie der Host-Header-Ursprungs-Fallback.

## Fehlerbehebung

<AccordionGroup>
  <Accordion title="trusted_proxy_untrusted_source">
    Die Anfrage stammt nicht von einer IP-Adresse in `gateway.trustedProxies`. Prüfen Sie Folgendes:

    - Ist die Proxy-IP-Adresse korrekt? (IP-Adressen von Docker-Containern können sich ändern.)
    - Befindet sich ein Load-Balancer vor Ihrem Proxy?
    - Verwenden Sie `docker inspect` oder `kubectl get pods -o wide`, um die tatsächlichen IP-Adressen zu ermitteln.

  </Accordion>
  <Accordion title="trusted_proxy_loopback_source">
    OpenClaw hat eine Trusted-Proxy-Anfrage aus einer Loopback-Quelle abgelehnt.

    Prüfen Sie Folgendes:

    - Stellt der Proxy die Verbindung über `127.0.0.1` / `::1` her?
    - Versuchen Sie, die Trusted-Proxy-Authentifizierung mit einem Loopback-Reverse-Proxy auf demselben Host zu verwenden?

    Behebung:

    - Verwenden Sie vorzugsweise Token-/Passwortauthentifizierung für interne Clients auf demselben Host, die nicht über den Proxy kommunizieren, oder
    - leiten Sie den Datenverkehr über eine vertrauenswürdige Nicht-Loopback-Proxyadresse und belassen Sie diese IP-Adresse in `gateway.trustedProxies`, oder
    - legen Sie für einen bewusst eingesetzten Reverse-Proxy auf demselben Host `gateway.auth.trustedProxy.allowLoopback = true` fest, belassen Sie die Loopback-Adresse in `gateway.trustedProxies` und stellen Sie sicher, dass der Proxy Identitäts-Header entfernt oder überschreibt.

  </Accordion>
  <Accordion title="trusted_proxy_local_interface_source / trusted_proxy_local_interface_check_failed">
    Die Quell-IP-Adresse der Anfrage stimmte mit einer der eigenen Nicht-Loopback-Netzwerkschnittstellenadressen des Gateway-Hosts überein (nicht mit dem Proxy). Dies schützt vor gefälschtem Datenverkehr desselben Hosts in Tailnets oder Docker-Bridge-Netzwerken. `..._check_failed` bedeutet, dass bei der Ermittlung der Schnittstellen selbst ein Fehler aufgetreten ist, weshalb OpenClaw sicher fehlschlägt.

    Prüfen Sie Folgendes:

    - Sendet ein Prozess auf dem Gateway-Host selbst Identitäts-Header direkt und umgeht dabei den Proxy?
    - Wird der Proxy im selben Netzwerk-Namespace wie das Gateway mit einer IP-Adresse ausgeführt, die ebenfalls als lokale Schnittstelle angezeigt wird?

    Behebung: Leiten Sie den Proxy-Datenverkehr über eine Adresse, die nicht zugleich lokal an den Gateway-Host gebunden ist, oder verwenden Sie `allowLoopback` ausschließlich für eine echte Proxy-Einrichtung auf demselben Host.

  </Accordion>
  <Accordion title="trusted_proxy_user_missing">
    Der Benutzer-Header war leer oder fehlte. Prüfen Sie Folgendes:

    - Ist Ihr Proxy so konfiguriert, dass er Identitäts-Header übergibt?
    - Ist der Header-Name korrekt? (Groß-/Kleinschreibung wird nicht berücksichtigt, die Schreibweise muss jedoch stimmen.)
    - Ist der Benutzer tatsächlich am Proxy authentifiziert?

  </Accordion>
  <Accordion title="trusted_proxy_missing_header_*">
    Ein erforderlicher Header war nicht vorhanden. Prüfen Sie Folgendes:

    - Ihre Proxy-Konfiguration für diese spezifischen Header.
    - Ob Header an irgendeiner Stelle in der Kette entfernt werden.

  </Accordion>
  <Accordion title="trusted_proxy_user_not_allowed">
    Der Benutzer ist authentifiziert, befindet sich jedoch nicht in `allowUsers`. Fügen Sie ihn entweder hinzu oder entfernen Sie die Zulassungsliste.
  </Accordion>
  <Accordion title="trusted_proxy_no_proxies_configured / trusted_proxy_config_missing">
    `gateway.auth.mode` ist `"trusted-proxy"`, aber `gateway.trustedProxies` ist leer, oder `gateway.auth.trustedProxy` selbst fehlt. Jede Anfrage wird abgelehnt, bis beide festgelegt sind.
  </Accordion>
  <Accordion title="trusted_proxy_origin_not_allowed">
    Die Trusted-Proxy-Authentifizierung war erfolgreich, aber der Browser-Header `Origin` hat die Herkunftsprüfungen der Control UI nicht bestanden.

    Prüfen Sie Folgendes:

    - `gateway.controlUi.allowedOrigins` enthält den exakten Browser-Ursprung.
    - Sie verlassen sich nicht auf Platzhalter-Ursprünge, es sei denn, Sie möchten absichtlich ein Verhalten, das alle Ursprünge zulässt.
    - Wenn Sie absichtlich den Host-Header-Fallback-Modus verwenden, ist `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback=true` bewusst festgelegt.

  </Accordion>
  <Accordion title="Verbindung erfolgreich, aber Methoden melden fehlenden Scope">
    Die WebSocket-Verbindung wird hergestellt, aber `chat.history`, `sessions.list` oder
    `models.list` schlägt mit `missing scope: operator.read` fehl.

    Häufige Ursachen:

    - Control-UI-Sitzung ohne Gerät: Die Trusted-Proxy-Authentifizierung kann die WebSocket-Verbindung ohne Geräteidentität zulassen, aber OpenClaw entfernt bei Sitzungen ohne Gerät absichtlich die Scopes.
    - Benutzerdefinierter Backend-Client: Die außer Betrieb genommene Control-UI-Upgrade-Eingabe gewährt niemals beliebigen Backend- oder CLI-ähnlichen WebSocket-Clients Zugriff.
    - Zu eng gefasstes `x-openclaw-scopes`: Wenn Ihr Proxy diesen Header bei der WebSocket-Upgrade-Anfrage der Control UI einfügt, werden die Sitzungs-Scopes auf diese Menge begrenzt. Ein leerer Header-Wert führt dazu, dass keine Scopes vorhanden sind.

    Behebung:

    - Verwenden Sie für die Control UI HTTPS, damit der Browser eine Geräteidentität erzeugen und das Pairing abschließen kann.
    - Verwenden Sie für benutzerdefinierte Automatisierung die Geräteidentität beziehungsweise das Pairing, den reservierten direkten lokalen Backend-Hilfspfad `gateway-client` oder [Admin-HTTP-RPC](/de/plugins/admin-http-rpc).
    - Fügen Sie den außer Betrieb genommenen Schlüssel `gateway.controlUi.dangerouslyDisableDeviceAuth` nicht zur aktuellen Konfiguration hinzu. Ältere Installationen verwenden automatisch die einmalige Migration für das Selbst-Pairing.

  </Accordion>
  <Accordion title="WebSocket schlägt weiterhin fehl">
    Stellen Sie sicher, dass Ihr Proxy:

    - WebSocket-Upgrades unterstützt (`Upgrade: websocket`, `Connection: upgrade`).
    - Die Identitäts-Header bei WebSocket-Upgrade-Anfragen weiterleitet (nicht nur bei HTTP).
    - Keinen separaten Authentifizierungspfad für WebSocket-Verbindungen verwendet.

  </Accordion>
</AccordionGroup>

## Migration von Token-Authentifizierung

<Steps>
  <Step title="Proxy konfigurieren">
    Konfigurieren Sie Ihren Proxy so, dass er Benutzer authentifiziert und Header weiterleitet.
  </Step>
  <Step title="Proxy unabhängig testen">
    Testen Sie die Proxy-Einrichtung unabhängig (curl mit Headern).
  </Step>
  <Step title="OpenClaw-Konfiguration aktualisieren">
    Aktualisieren Sie die OpenClaw-Konfiguration mit Trusted-Proxy-Authentifizierung.
  </Step>
  <Step title="Gateway neu starten">
    Starten Sie das Gateway neu.
  </Step>
  <Step title="WebSocket testen">
    Testen Sie WebSocket-Verbindungen über die Control UI.
  </Step>
  <Step title="Überprüfen">
    Führen Sie `openclaw security audit` aus und prüfen Sie die Ergebnisse.
  </Step>
</Steps>

## Verwandte Themen

- [Konfiguration](/de/gateway/configuration) — Konfigurationsreferenz
- [Operator-Scopes](/de/gateway/operator-scopes) — Rollen, Scopes und Genehmigungsprüfungen
- [Remotezugriff](/de/gateway/remote) — weitere Muster für den Remotezugriff
- [Sicherheit](/de/gateway/security) — vollständiger Sicherheitsleitfaden
- [Tailscale](/de/gateway/tailscale) — einfachere Alternative für den Zugriff ausschließlich über das Tailnet
