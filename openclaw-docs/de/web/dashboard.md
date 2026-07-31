---
read_when:
    - Ändern der Dashboard-Authentifizierung oder der Zugriffsmodi
summary: Zugriff und Authentifizierung für das Gateway-Dashboard (Control UI)
title: Dashboard
x-i18n:
    generated_at: "2026-07-26T19:18:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ca531ad2943dfdee1cd90a4efdc1fb69c4517780e2be52237fd558b8638e7cd0
    source_path: web/dashboard.md
    workflow: 16
---

Das Gateway-Dashboard ist die browserbasierte Control UI, die standardmäßig unter `/` bereitgestellt wird (Überschreibung mit `gateway.controlUi.basePath`).

Schnellzugriff (lokales Gateway):

- [http://127.0.0.1:18789/](http://127.0.0.1:18789/) (oder [http://localhost:18789/](http://localhost:18789/))
- Mit `gateway.tls.enabled: true` verwenden Sie `https://127.0.0.1:18789/` und `wss://127.0.0.1:18789` für den WebSocket-Endpunkt.

Wichtige Referenzen:

- [Control UI](/de/web/control-ui) für Verwendung und UI-Funktionen.
- [Tailscale](/de/gateway/tailscale) für die Serve-/Funnel-Automatisierung.
- [Weboberflächen](/de/web) für Bindungsmodi und Sicherheitshinweise.

Die Authentifizierung wird beim WebSocket-Handshake über den konfigurierten Gateway-Authentifizierungspfad erzwungen:

- `connect.params.auth.token`
- `connect.params.auth.password`
- Tailscale-Serve-Identitätsheader bei `gateway.auth.allowTailscale: true`
- Identitätsheader eines vertrauenswürdigen Proxys bei `gateway.auth.mode: "trusted-proxy"`

Siehe `gateway.auth` unter [Gateway-Konfiguration](/de/gateway/configuration).

<Warning>
Die Control UI ist eine **Administrationsoberfläche** (Chat, Konfiguration, Ausführungsgenehmigungen). Stellen Sie sie nicht öffentlich bereit. Die UI speichert Dashboard-URL-Token für den aktuellen Browser-Tab und die ausgewählte Gateway-URL in sessionStorage und entfernt sie nach dem Laden aus der URL. Bevorzugen Sie localhost, Tailscale Serve oder einen SSH-Tunnel.
</Warning>

## Schnellster Weg (empfohlen)

- Nach dem Onboarding öffnet die CLI das Dashboard automatisch und gibt einen bereinigten Link (ohne Token) aus.
- Jederzeit erneut öffnen: `openclaw dashboard` (kopiert den Link, öffnet nach Möglichkeit einen Browser und gibt in einer Umgebung ohne grafische Oberfläche einen SSH-Hinweis aus).
- Wenn sowohl die Übergabe an die Zwischenablage als auch an den Browser fehlschlägt, gibt `openclaw dashboard` weiterhin die bereinigte URL aus und weist Sie an, Ihr Token (aus `OPENCLAW_GATEWAY_TOKEN` oder `gateway.auth.token`) mit dem URL-Fragment-Schlüssel `token` anzuhängen; der Token-Wert wird niemals in Protokollen ausgegeben.
- Wenn die UI zur Authentifizierung mit einem gemeinsamen Geheimnis auffordert, fügen Sie das konfigurierte Token oder Passwort in die Einstellungen der Control UI ein.

## Authentifizierungsgrundlagen (lokal und remote)

- **Localhost**: Öffnen Sie `http://127.0.0.1:18789/`.
- **Gateway-TLS**: Bei `gateway.tls.enabled: true` verwenden Dashboard-/Statuslinks `https://` und WebSocket-Links der Control UI `wss://`.
- **Quelle des Tokens für das gemeinsame Geheimnis**: `gateway.auth.token` (oder `OPENCLAW_GATEWAY_TOKEN`). `openclaw dashboard` kann es für die einmalige Ersteinrichtung über das URL-Fragment übergeben; die Control UI speichert es für den aktuellen Tab und die ausgewählte Gateway-URL in sessionStorage, nicht in localStorage.
- **Laufzeit-Token bei fehlender Konfiguration**: Wenn beim Start gemeldet wird, dass ein Laufzeit-Token generiert wurde, ist dieses Token flüchtig und nicht über `openclaw config get gateway.auth.token` verfügbar. Auch Loopback erfordert eine Authentifizierung. Führen Sie `openclaw doctor --generate-gateway-token` aus, starten Sie das Gateway neu und fügen Sie anschließend das konfigurierte Token in die Einstellungen der Control UI ein.
- Wenn `gateway.auth.token` durch SecretRef verwaltet wird, gibt `openclaw dashboard` absichtlich eine URL ohne Token aus, kopiert oder öffnet sie, damit extern verwaltete Token nicht in Shell-Protokollen, im Verlauf der Zwischenablage oder in Argumenten zum Browserstart offengelegt werden. Wenn die Referenz in Ihrer aktuellen Shell nicht aufgelöst werden kann, werden weiterhin die URL ohne Token sowie konkrete Anweisungen zur Einrichtung der Authentifizierung ausgegeben.
- **Passwort für das gemeinsame Geheimnis**: Verwenden Sie das konfigurierte `gateway.auth.password` (oder `OPENCLAW_GATEWAY_PASSWORD`). Das Dashboard speichert Passwörter nicht über ein erneutes Laden hinaus.
- **Modi mit Identitätsinformationen**: Tailscale Serve erfüllt die Authentifizierung der Control UI bzw. des WebSockets über Identitätsheader bei `gateway.auth.allowTailscale: true`; ein identitätsfähiger Nicht-Loopback-Reverse-Proxy erfüllt `gateway.auth.mode: "trusted-proxy"`. Für den WebSocket muss in keinem der beiden Fälle ein gemeinsames Geheimnis eingefügt werden.
- **Nicht localhost**: Verwenden Sie Tailscale Serve, eine Nicht-Loopback-Bindung mit gemeinsamem Geheimnis, einen identitätsfähigen Nicht-Loopback-Reverse-Proxy mit `gateway.auth.mode: "trusted-proxy"` oder einen SSH-Tunnel. HTTP-APIs verwenden weiterhin die Authentifizierung mit einem gemeinsamen Geheimnis, sofern Sie nicht bewusst privates Ingress mit `gateway.auth.mode: "none"` oder HTTP-Authentifizierung über einen vertrauenswürdigen Proxy einsetzen. Siehe [Weboberflächen](/de/web).

## In Telegram öffnen

Telegram-Bots können das Dashboard mit `/dashboard` als Telegram Mini App öffnen.

Voraussetzungen:

- `gateway.tailscale.mode: "serve"` oder `"funnel"`, damit Telegram eine HTTPS-URL für die Mini App erhält.
- Der Telegram-Absender muss der Bot-Eigentümer sein: eine numerische Telegram-Benutzer-ID in `commands.ownerAllowFrom` oder der effektive Wert von `channels.telegram.allowFrom` des ausgewählten Kontos.
- Führen Sie `/dashboard` in einer Direktnachricht an den Bot aus. Bei Aufrufen in Gruppen wird lediglich darauf hingewiesen, den Befehl in einer Direktnachricht zu öffnen; eine Schaltfläche ist nicht enthalten.
- Docker-Installationen: Serve-/Funnel-Modi erfordern, dass das Gateway neben `tailscaled` an Loopback gebunden wird; Bridge-Netzwerke mit veröffentlichten Ports können dies nicht erfüllen. Führen Sie den Gateway-Container mit `network_mode: host` aus und binden Sie den Host-Socket `tailscaled` (`/var/run/tailscale`) sowie die CLI `tailscale` in den Container ein.

Die Mini App führt eine einmalige Übergabe durch den Eigentümer aus und leitet mit einem kurzlebigen Bootstrap-Token zur Control UI weiter. Sie legt kein gemeinsames Gateway-Token in der URL offen.

Nichtziele für v1:

- Der Telegram-Web-iFrame wird nicht unterstützt.
- Tailscale Serve/Funnel ist der einzige unterstützte Pfad für eine veröffentlichte URL.

<a id="if-you-see-unauthorized-1008"></a>

## Wenn „unauthorized“ / 1008 angezeigt wird

- Prüfen Sie, ob das Gateway erreichbar ist: lokal über `openclaw status`; remote über den SSH-Tunnel `ssh -N -L 18789:127.0.0.1:18789 user@gateway-host`, öffnen Sie anschließend `http://127.0.0.1:18789/`.
- Bei `AUTH_TOKEN_MISMATCH` können Clients einen einzigen vertrauenswürdigen Wiederholungsversuch mit einem zwischengespeicherten Geräte-Token durchführen, wenn das Gateway Hinweise für einen Wiederholungsversuch zurückgibt; dabei werden die zwischengespeicherten genehmigten Geltungsbereiche des Tokens wiederverwendet (explizite Aufrufer von `deviceToken`/`scopes` behalten die von ihnen angeforderte Menge an Geltungsbereichen bei). Wenn die Authentifizierung nach diesem Wiederholungsversuch weiterhin fehlschlägt, beheben Sie die Token-Abweichung manuell.
- Bei `AUTH_SCOPE_MISMATCH` wurde das Geräte-Token erkannt, enthält jedoch nicht die angeforderten Geltungsbereiche; koppeln Sie das Gerät erneut oder genehmigen Sie die neue Menge an Geltungsbereichen, statt das gemeinsame Gateway-Token zu rotieren.
- Außerhalb dieses Wiederholungsversuchs gilt für die Verbindungsauthentifizierung folgende Priorität: explizites gemeinsames Token/Passwort, dann explizites `deviceToken`, dann gespeichertes Geräte-Token, dann Bootstrap-Token.
- Im asynchronen Tailscale-Serve-Pfad werden fehlgeschlagene Versuche für dieselbe `{scope, ip}` serialisiert, bevor sie vom Begrenzer für fehlgeschlagene Authentifizierungen erfasst werden. Daher kann bei einem zweiten gleichzeitig ausgeführten fehlerhaften Wiederholungsversuch bereits `retry later` angezeigt werden.
- Schritte zur Behebung einer Token-Abweichung finden Sie in der [Checkliste zur Wiederherstellung bei Token-Abweichungen](/de/cli/devices#token-drift-recovery-checklist).
- Rufen Sie das gemeinsame Geheimnis vom Gateway-Host ab oder stellen Sie es dort bereit:
  - Token: `openclaw config get gateway.auth.token`
  - Passwort: Lösen Sie das konfigurierte `gateway.auth.password` oder `OPENCLAW_GATEWAY_PASSWORD` auf
  - Durch SecretRef verwaltetes Token: Lösen Sie den externen Geheimnis-Provider auf oder exportieren Sie `OPENCLAW_GATEWAY_TOKEN` in dieser Shell und führen Sie `openclaw dashboard` erneut aus
  - Laufzeit-Token, das generiert wurde, weil kein gemeinsames Geheimnis konfiguriert war: Führen Sie `openclaw doctor --generate-gateway-token` aus, starten Sie das Gateway neu und verwenden Sie anschließend das konfigurierte Token
- Fügen Sie in den Dashboard-Einstellungen das Token oder Passwort in das Authentifizierungsfeld ein und stellen Sie anschließend die Verbindung her.
- Die Sprachauswahl der UI befindet sich unter **Settings -> General -> Language**, nicht unter Appearance.

## Verwandte Themen

- [Control UI](/de/web/control-ui)
- [WebChat](/de/web/webchat)
