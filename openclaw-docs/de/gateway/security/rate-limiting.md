---
read_when:
    - Ein Client sieht `rate limit exceeded for <method>`, `AUTH_RATE_LIMITED` oder Sperrfehler
    - Sie möchten `gateway.auth.rateLimit` optimieren
    - Sie befassen sich mit dem Schutz vor Brute-Force-Angriffen auf einen öffentlich erreichbaren Gateway
    - Sie müssen wissen, welche Gateway-Schnittstellen gedrosselt werden und welche Limits gelten.
summary: 'Referenz für alle Gateway-Ratenbegrenzungen: Sperren vor der Authentifizierung, Drosselung für Browser und Webhooks, die Absicherung für Schreibvorgänge auf der Steuerungsebene, ACP-Sitzungslimits und die Abklingzeit für Neustarts'
title: Ratenbegrenzung
x-i18n:
    generated_at: "2026-07-26T17:51:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7aa37b65347610bedfb1db8f661e7ba75ef3cdfed0ba73c4ce53d80acace1e48
    source_path: gateway/security/rate-limiting.md
    workflow: 16
---

Der Gateway erzwingt mehrere unabhängige Ratenbegrenzungen. Sie schützen unterschiedliche
Grenzen, verwenden unterschiedliche Identitäten als Schlüssel und schlagen mit unterschiedlichen Fehlerstrukturen fehl.
Diese Seite dient als Referenz für alle diese Begrenzungen.

Auf einen Blick:

| Oberfläche                          | Begrenzung (Standard)             | Schlüssel                         | Konfigurierbar           |
| ----------------------------------- | --------------------------------- | --------------------------------- | ------------------------ |
| Fehlgeschlagene Authentifizierung (Token/Passwort/Gerät) | 10 Fehlschläge / 60 s, 5 Min. Sperre | IP + Anmeldedatenbereich          | `gateway.auth.rateLimit` |
| WS-Authentifizierungsfehler mit Browserursprung | identisch, Loopback **nicht** ausgenommen | IP oder Seitenursprung bei Loopback | `gateway.auth.rateLimit` |
| Authentifizierungsfehler bei Webhook (`/hooks`) | 20 Fehlschläge / 60 s, 60 s Sperre | IP                                | nein                     |
| Schreibende RPCs der Steuerungsebene | 30 Anfragen / 60 s pro Methode    | Methode + Gerät + IP              | nein                     |
| ACP-Sitzungserstellung              | 120 Sitzungen / 10 s              | Übersetzerinstanz                 | intern                   |
| Gateway-Neustartzyklen              | 30 s Abklingzeit zwischen Neustarts | Prozess                           | nein                     |

## Authentifizierungsversuche (vor der Authentifizierung)

Fehlgeschlagene Authentifizierungsversuche werden pro Client-IP gedrosselt, bevor eine
Anfrage verarbeitet wird. Dies ist der Brute-Force-Schutz für öffentlich erreichbare Gateways.

- Nur _falsche_ Anmeldedaten werden gezählt. Fehlende Anmeldedaten (ein Client, der nie
  ein Token gesendet hat) und erfolgreiche Authentifizierungen verbrauchen kein Kontingent; eine
  erfolgreiche Authentifizierung setzt den Zähler für diese IP zurück.
- Standardwerte: 10 Fehlschläge pro 60 Sekunden, danach eine 5-minütige Sperre für diese IP.
- Loopback (`127.0.0.1` / `::1`) ist standardmäßig ausgenommen, damit lokale CLI-Sitzungen
  nicht ausgesperrt werden können.
- Zähler sind nach Anmeldedatenklasse getrennt, sodass eine Flut gegen eine Oberfläche
  keine andere verdrängt. Zu den Bereichen gehören das gemeinsam genutzte Gateway-
  Token/Passwort, Gerätetoken, Node-Kopplung, erneute Genehmigung gekoppelter Nodes,
  Bootstrap-Token für Geräte und die Ausstellung von watchOS-Challenges.

Während einer Sperre schlagen Verbindungsversuche wie folgt fehl:

```json
{
  "code": "INVALID_REQUEST",
  "message": "unauthorized: too many failed authentication attempts (retry later)",
  "retryable": true,
  "retryAfterMs": 297000,
  "details": {
    "code": "AUTH_RATE_LIMITED",
    "authReason": "rate_limited",
    "recommendedNextStep": "wait_then_retry"
  }
}
```

Versuche von anderen IPs (einschließlich Loopback) sind während einer Sperre nicht betroffen.

Passen Sie die Einstellung unter `gateway.auth.rateLimit` in `openclaw.json` an:

```json
{
  "gateway": {
    "auth": {
      "rateLimit": {
        "maxAttempts": 10,
        "windowMs": 60000,
        "lockoutMs": 300000,
        "exemptLoopback": true
      }
    }
  }
}
```

Wiederholte `AUTH_RATE_LIMITED`-Einträge im Gateway-Protokoll bedeuten, dass jemand
Anmeldedaten zu erraten versucht; siehe das [Runbook zur Exposition](/de/gateway/security/exposure-runbook).

### Verbindungen mit Browserursprung

WebSocket-Verbindungen, die einen Browser-Header `Origin` enthalten, verwenden dieselben
Begrenzungen, jedoch ist die Loopback-Ausnahme **immer deaktiviert** – eine schädliche Seite in
einem lokalen Browser ist weiterhin ein nicht vertrauenswürdiger Client, daher erhält localhost
auf diesem Pfad keine Sonderbehandlung. Wenn eine solche Verbindung _von_ einer Loopback-Adresse eingeht, werden ihre
Fehlschläge anhand des normalisierten Seitenursprungs (zum Beispiel
`browser-origin:https://evil.example`) statt anhand der gemeinsam genutzten Loopback-IP erfasst,
sodass jeder Ursprung ein eigenes Kontingent erhält; bei Nicht-Loopback-Adressen bleibt
die Client-IP der Schlüssel. Dies ist nicht konfigurierbar.

### Webhooks

Der HTTP-Eingang `/hooks` verfügt über eine eigene Begrenzung für Fehlschläge: 20 fehlgeschlagene
Authentifizierungen pro 60 Sekunden und Client-IP, danach eine 60-sekündige Sperre.
Loopback ist nicht ausgenommen. Eine erfolgreiche Hook-Authentifizierung setzt den Zähler zurück. Gedrosselte
Anfragen erhalten eine einfache HTTP-Antwort `429 Too Many Requests` mit einem `Retry-After`-
Header (Sekunden). Die Begrenzungen sind fest vorgegeben; wenn eine legitime Integration sie auslöst,
korrigieren Sie deren Anmeldedaten, statt die Wiederholungsversuche zu intensivieren.

## Schreibvorgänge der Steuerungsebene (nachgelagerte Absicherung nach der Authentifizierung)

Schreibende administrative RPCs (`config.apply`, `config.patch`, `plugins.install`,
`plugins.setEnabled`, `plugins.uninstall`, `update.run`, `worktrees.*`,
`gateway.restart.request`, ...) werden zusätzlich **nach**
der Autorisierung ratenbegrenzt: 30 Anfragen pro 60 Sekunden, pro Methode, pro
`deviceId+clientIp`.

Dies ist keine Sicherheitsgrenze – Aufrufer verfügen bereits über `operator.admin` –, sondern
eine Absicherung, die außer Kontrolle geratene Client- oder Agent-Schleifen begrenzt, die aufwendige
Vorgänge übermäßig häufig aufrufen. Bei interaktiver Nutzung wird die Begrenzung nie erreicht; jede Methode verfügt über ein eigenes Kontingent, sodass
das Umschalten eines Plugins nicht das Kontingent für Konfigurationsschreibvorgänge verbraucht.

Bei Überschreitung schlägt die Anfrage mit einem wiederholbaren Fehler fehl:

```json
{
  "code": "UNAVAILABLE",
  "message": "rate limit exceeded for config.patch; retry after 35s",
  "retryable": true,
  "retryAfterMs": 34539,
  "details": { "method": "config.patch", "limit": "30 per 60s" }
}
```

Clients sollten `retryAfterMs` beachten. Die Begrenzung ist fest vorgegeben (nicht konfigurierbar);
Kontingente laufen selbstständig ab und werden durch die Gateway-Wartung bereinigt.

## ACP-Sitzungserstellung

Der ACP-Übersetzer begrenzt die Sitzungserstellung auf 120 neue Sitzungen pro 10-Sekunden-
Fenster und Übersetzerinstanz. Bei Überschreitung schlägt die Anfrage mit einem Fehler fehl,
dessen Meldung die Wartezeit enthält (auf diesem Pfad gibt es kein strukturiertes Feld `retryAfterMs`):

```
Ratenbegrenzung für die ACP-Sitzungserstellung für <method> überschritten; erneuter Versuch nach <n> s.
```

Dies begrenzt außer Kontrolle geratene Clients, die Sitzungen in einer Schleife erstellen; die normale Nutzung durch IDEs und
Agents bleibt weit darunter.

## Abklingzeit für Neustarts

Gateway-Neustartanfragen werden zusammengefasst und erzwingen anschließend eine Abklingzeit von 30 Sekunden zwischen
Neustartzyklen. Ein während der Abklingzeit angeforderter Neustart wird nach deren
Ablauf eingeplant, statt abgelehnt zu werden. Dies ist unabhängig von der Begrenzung der Steuerungsebene
oben: `gateway.restart.request` verbraucht einen Kontingentplatz der Steuerungsebene _und_
der daraus resultierende Neustart unterliegt der Abklingzeit.

## Betriebshinweise

- Alle Begrenzer befinden sich im Arbeitsspeicher und gelten pro Prozess; mehrere Gateways teilen
  keinen Zustand. Durch das Ersetzen des Gateway-Prozesses werden die Gateway-eigenen
  Zähler zurückgesetzt (Authentifizierungssperren, Webhook-Drosselung, Kontingente der Steuerungsebene). Die
  Abklingzeit für Neustarts bleibt bewusst über prozessinterne Neustartzyklen hinweg bestehen – genau diese
  drosselt sie – und wird erst mit dem Prozess zurückgesetzt. Die ACP-Sitzungsbegrenzung
  gehört zu ihrer Übersetzerinstanz und wird zurückgesetzt, wenn diese Instanz
  neu erstellt wird, nicht bei einem Gateway-Neustart.
- Kontingentzuordnungen sind begrenzt (feste Obergrenzen für Einträge sowie regelmäßige Bereinigung), sodass
  eine Flut eindeutiger Schlüssel den Speicher nicht unbegrenzt anwachsen lassen kann.
- Wenn sich ein Client hinter einem Reverse-Proxy befindet, ist die effektive IP die aufgelöste
  Client-IP; unter [Authentifizierung über vertrauenswürdige Proxys](/de/gateway/trusted-proxy-auth) erfahren Sie, wie
  Proxy-Header validiert werden, bevor sie diese beeinflussen können.
- Die Signalisierung für Wiederholungsversuche variiert je nach Oberfläche: Gateway-RPC-Begrenzer geben
  `retryable: true` sowie `retryAfterMs` zurück, der Webhook-Eingang verwendet HTTP 429
  mit einem `Retry-After`-Header und ACP bettet die Wartezeit in die Fehlermeldung ein.
  Warten Sie in jedem Fall für die angegebene Dauer, statt den Vorgang
  sofort zu wiederholen.
