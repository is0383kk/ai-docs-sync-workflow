---
read_when:
    - Der Leitfaden zur Fehlerbehebung hat Sie zur eingehenderen Diagnose hierher verwiesen
    - Sie benötigen stabile, symptombasierte Runbook-Abschnitte mit exakten Befehlen
sidebarTitle: Troubleshooting
summary: Ausführliches Handbuch zur Fehlerbehebung für Gateway, Kanäle, Automatisierung, Nodes und Browser
title: Fehlerbehebung
x-i18n:
    generated_at: "2026-07-26T17:51:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c4bb1e061dbf2767118c24ad1ca2d2d1f7eeeff88e18ed0e6111aebe1cc99a26
    source_path: gateway/troubleshooting.md
    workflow: 16
---

Dies ist das ausführliche Runbook. Beginnen Sie zunächst mit dem schnellen Triage-Ablauf unter [/help/troubleshooting](/de/help/troubleshooting).

## Befehlsabfolge

Führen Sie die Befehle in dieser Reihenfolge aus:

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

Signale für einen fehlerfreien Zustand:

- `openclaw gateway status` zeigt `Runtime: running`, `Connectivity probe: ok` und eine `Capability: ...`-Zeile.
- `openclaw doctor` meldet keine blockierenden Konfigurations- oder Dienstprobleme.
- `openclaw channels status --probe` zeigt den aktuellen Transportstatus pro Konto und, sofern unterstützt, `works` oder `audit ok`.

## Nach einem Update

Verwenden Sie dies, wenn ein Update abgeschlossen ist, der Gateway jedoch nicht verfügbar ist, keine Kanäle angezeigt werden oder Modellaufrufe mit 401-Fehlern fehlschlagen.

```bash
openclaw status --all
openclaw update status --json
openclaw gateway status --deep
openclaw doctor --fix
openclaw gateway restart
```

Achten Sie auf Folgendes:

- `Update restart` in `openclaw status` / `openclaw status --all`. Ausstehende oder fehlgeschlagene Übergaben enthalten den als Nächstes auszuführenden Befehl.
- `plugin load failed: dependency tree corrupted; run openclaw doctor --fix` unter „Kanäle“: Die Kanalkonfiguration ist noch vorhanden, aber die Plugin-Registrierung ist fehlgeschlagen, bevor der Kanal geladen werden konnte.
- Provider-401-Fehler nach erneuter Authentifizierung: `openclaw doctor --fix` sucht nach veralteten agentenspezifischen OAuth-Authentifizierungsschatten und entfernt alte Kopien, damit alle Agenten das aktuelle gemeinsame Profil auflösen.

## Uneinheitliche Installationen und Schutz vor neuerer Konfiguration

Verwenden Sie dies, wenn ein Gateway-Dienst nach einem Update unerwartet beendet wird oder die Protokolle zeigen, dass eine `openclaw`-Binärdatei älter als die Version ist, die zuletzt `openclaw.json` geschrieben hat.

OpenClaw kennzeichnet Konfigurationsschreibvorgänge mit `meta.lastTouchedVersion`. Schreibgeschützte Befehle können eine von einer neueren OpenClaw-Version geschriebene Konfiguration prüfen, Prozess- und Dienständerungen werden von einer älteren Binärdatei jedoch verweigert. Blockierte Aktionen: Starten, Beenden, Neustarten oder Deinstallieren des Gateway-Dienstes, erzwungene Neuinstallation des Dienstes, Starten des Gateways im Dienstmodus und `gateway --force`-Portbereinigung.

```bash
which openclaw
openclaw --version
openclaw gateway status --deep
openclaw config get meta.lastTouchedVersion
```

<Steps>
  <Step title="PATH korrigieren">
    Korrigieren Sie `PATH`, sodass `openclaw` zur neueren Installation aufgelöst wird, und führen Sie die Aktion erneut aus.
  </Step>
  <Step title="Gateway-Dienst neu installieren">
    Installieren Sie den vorgesehenen Gateway-Dienst aus der neueren Installation neu:

    ```bash
    openclaw gateway install --force
    openclaw gateway restart
    ```

  </Step>
  <Step title="Veraltete Wrapper entfernen">
    Entfernen Sie veraltete Systempaket- oder alte Wrapper-Einträge, die weiterhin auf eine alte `openclaw`-Binärdatei verweisen.
  </Step>
</Steps>

<Warning>
Legen Sie `OPENCLAW_ALLOW_OLDER_BINARY_DESTRUCTIVE_ACTIONS=1` nur für ein beabsichtigtes Downgrade oder eine Notfallwiederherstellung und nur für den einzelnen Befehl fest. Lassen Sie die Variable im normalen Betrieb ungesetzt.
</Warning>

## Protokollabweichung nach einem Rollback

Verwenden Sie dies, wenn die Protokolle nach einem Downgrade oder Rollback weiterhin `protocol mismatch` ausgeben. Ein älterer Gateway wird ausgeführt, aber ein neuerer lokaler Clientprozess versucht weiterhin, mit einem Protokollbereich, den der ältere Gateway nicht unterstützt, erneut eine Verbindung herzustellen.

```bash
openclaw --version
which -a openclaw
openclaw gateway status --deep
openclaw doctor --deep
openclaw logs --follow
```

Achten Sie auf Folgendes:

- `protocol mismatch ... client=... v<version> min=<n> max=<n> expected=<n>` in den Gateway-Protokollen.
- `Established clients:` in `openclaw gateway status --deep` oder `Gateway clients` in `openclaw doctor --deep`: aktive TCP-Clients, die mit dem Gateway-Port verbunden sind, einschließlich PIDs und Befehlszeilen, sofern das Betriebssystem dies zulässt.
- Ein Clientprozess, dessen Befehlszeile auf die neuere OpenClaw-Installation oder den Wrapper verweist, von dem das Rollback durchgeführt wurde.

Behebung:

1. Beenden Sie den von `gateway status --deep` angezeigten veralteten OpenClaw-Clientprozess oder starten Sie ihn neu.
2. Starten Sie Apps oder Wrapper neu, die OpenClaw einbetten: lokale Dashboards, Editoren, App-Server-Hilfsprogramme oder langlebige `openclaw logs --follow`-Shells.
3. Führen Sie `openclaw gateway status --deep` oder `openclaw doctor --deep` erneut aus und vergewissern Sie sich, dass die PID des veralteten Clients nicht mehr vorhanden ist.

Versuchen Sie nicht, einen älteren Gateway zur Annahme eines neueren inkompatiblen Protokolls zu veranlassen. Protokollversionssprünge schützen den Übertragungsvertrag; die Wiederherstellung nach einem Rollback erfordert die Bereinigung von Prozessen und Versionen.

## Skill-Symlink wegen Pfadüberschreitung übersprungen

Verwenden Sie dies, wenn die Protokolle Folgendes enthalten:

```text
Übersprungener Skill-Pfad außerhalb seines konfigurierten Stammverzeichnisses: ... reason=symlink-escape
```

Jedes Skill-Stammverzeichnis ist eine Begrenzungsgrenze. Ein Symlink unter `~/.agents/skills`, `<workspace>/.agents/skills`, `<workspace>/skills` oder `~/.openclaw/skills` wird übersprungen, wenn sein tatsächliches Ziel außerhalb dieses Stammverzeichnisses aufgelöst wird, sofern das Ziel nicht ausdrücklich als vertrauenswürdig eingestuft ist.

Prüfen Sie den Link:

```bash
ls -l ~/.agents/skills/<name>
realpath ~/.agents/skills/<name>
openclaw config get skills.load
```

Wenn das Ziel beabsichtigt ist, konfigurieren Sie sowohl das direkte Skill-Stammverzeichnis als auch das zulässige Symlink-Ziel:

```json5
{
  skills: {
    load: {
      extraDirs: ["~/Projects/manager/skills"],
      allowSymlinkTargets: ["~/Projects/manager/skills"],
    },
  },
}
```

Starten Sie anschließend eine neue Sitzung oder warten Sie, bis der Skills-Watcher die Daten aktualisiert. Starten Sie den Gateway neu, wenn der laufende Prozess bereits vor der Konfigurationsänderung gestartet wurde.

Verwenden Sie keine weit gefassten Ziele wie `~`, `/` oder einen gesamten synchronisierten Projektordner. Beschränken Sie `allowSymlinkTargets` auf das tatsächliche Skill-Stammverzeichnis, das vertrauenswürdige `SKILL.md`-Verzeichnisse enthält.

Wenn das Anwenden in Skill Workshop auch über diese vertrauenswürdigen, per Symlink eingebundenen Workspace-Skill-Pfade schreiben soll, aktivieren Sie `skills.workshop.allowSymlinkTargetWrites`. Lassen Sie diese Option für schreibgeschützte gemeinsame Skill-Stammverzeichnisse deaktiviert.

Verwandte Themen:

- [Skills-Konfiguration](/de/tools/skills-config#symlinked-skill-roots)
- [Konfigurationsbeispiele](/de/gateway/configuration-examples#symlinked-sibling-skill-repo)

## Für Anthropic 429 ist bei langem Kontext zusätzliche Nutzung erforderlich

Verwenden Sie dies, wenn Protokolle oder Fehler `HTTP 429: rate_limit_error: Extra usage is required for long context requests` enthalten.

```bash
openclaw logs --follow
openclaw models status
openclaw config get agents.defaults.models
```

Achten Sie auf Folgendes:

- Das ausgewählte Anthropic-Modell ist ein allgemein verfügbares Claude-4.x-Modell mit 1M-Unterstützung (Opus 4.6/4.7/4.8, Sonnet 4.6), oder die Modellkonfiguration enthält noch das veraltete `params.context1m: true`.
- Die aktuellen Anthropic-Anmeldedaten sind nicht für die Nutzung langer Kontexte berechtigt.
- Anfragen schlagen nur bei langen Sitzungen oder Modellläufen fehl, die den 1M-Kontextpfad benötigen.

Behebungsoptionen:

<Steps>
  <Step title="Standardkontextfenster verwenden">
    Wechseln Sie zu einem Modell mit Standardkontextfenster oder entfernen Sie das veraltete `context1m` aus einer älteren
    Modellkonfiguration, die nicht allgemein für einen 1M-Kontext verfügbar ist.
  </Step>
  <Step title="Berechtigte Anmeldedaten verwenden">
    Verwenden Sie Anthropic-Anmeldedaten, die für Anfragen mit langem Kontext berechtigt sind, oder wechseln Sie zu einem Anthropic-API-Schlüssel.
  </Step>
  <Step title="Fallback-Modelle konfigurieren">
    Konfigurieren Sie Fallback-Modelle, damit Läufe fortgesetzt werden, wenn Anthropic Anfragen mit langem Kontext ablehnt.
  </Step>
</Steps>

Verwandte Themen:

- [Anthropic](/de/providers/anthropic)
- [Token-Nutzung und Kosten](/de/reference/token-use)
- [Warum erhalte ich HTTP 429 von Anthropic?](/de/help/faq-first-run#why-am-i-seeing-http-429-ratelimiterror-from-anthropic)

## Blockierte 403-Antworten von Upstream-Diensten

Verwenden Sie dies, wenn ein vorgelagerter LLM-Provider einen generischen `403` wie `Your request was blocked` zurückgibt.

Gehen Sie nicht davon aus, dass dies immer ein Konfigurationsproblem von OpenClaw ist. Die Antwort kann von einer vorgelagerten Sicherheitsschicht wie einem CDN, einer WAF, einer Bot-Management-Regel oder einem Reverse Proxy vor einem OpenAI-kompatiblen Endpunkt stammen.

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
```

Achten Sie auf Folgendes:

- Mehrere Modelle desselben Providers schlagen auf dieselbe Weise fehl.
- HTML oder generischer Sicherheitstext anstelle eines normalen Provider-API-Fehlers.
- Providerseitige Sicherheitsereignisse zum selben Anfragezeitpunkt.
- Eine sehr kleine direkte `curl`-Prüfung ist erfolgreich, während normale SDK-förmige Anfragen fehlschlagen.

Beheben Sie zuerst die providerseitige Filterung, wenn die Indizien auf eine Blockierung durch WAF/CDN hindeuten. Bevorzugen Sie eine eng begrenzte Zulassungs- oder Überspringungsregel für den von OpenClaw verwendeten API-Pfad und vermeiden Sie es, den Schutz für die gesamte Website zu deaktivieren.

<Warning>
Ein erfolgreiches minimales `curl` garantiert nicht, dass echte SDK-typische Anfragen dieselbe vorgelagerte Sicherheitsschicht passieren.
</Warning>

Verwandte Themen:

- [OpenAI-kompatible Endpunkte](/de/gateway/configuration-reference#openai-compatible-endpoints)
- [Provider-Konfiguration](/de/providers)
- [Protokolle](/de/logging)

## Lokales OpenAI-kompatibles Backend besteht direkte Prüfungen, aber Agentenläufe schlagen fehl

Verwenden Sie dies, wenn:

- `curl ... /v1/models` funktioniert.
- Sehr kleine direkte `/v1/chat/completions`-Aufrufe funktionieren.
- OpenClaw-Modellläufe schlagen nur bei normalen Agentenzügen fehl.

```bash
curl http://127.0.0.1:1234/v1/models
curl http://127.0.0.1:1234/v1/chat/completions \
  -H 'content-type: application/json' \
  -d '{"model":"<id>","messages":[{"role":"user","content":"hi"}],"stream":false}'
openclaw infer model run --model <provider/model> --prompt "hi" --json
openclaw logs --follow
```

Achten Sie auf Folgendes:

- Kleine direkte Aufrufe sind erfolgreich, OpenClaw-Läufe schlagen jedoch nur bei größeren Prompts fehl.
- `model_not_found`- oder 404-Fehler, obwohl direktes `/v1/chat/completions` mit derselben unveränderten Modell-ID funktioniert.
- Backend-Fehler, laut denen `messages[].content` eine Zeichenfolge erwartet.
- Zeitweilige `incomplete turn detected ... stopReason=stop payloads=0`-Warnungen bei einem OpenAI-kompatiblen lokalen Backend.
- Backend-Abstürze, die nur bei einer größeren Anzahl von Prompt-Tokens oder vollständigen Prompts der Agentenlaufzeit auftreten.

<AccordionGroup>
  <Accordion title="Häufige Fehlermuster">
    - `model_not_found` bei einem lokalen Server im MLX-/vLLM-Stil: Vergewissern Sie sich, dass `baseUrl` `/v1` enthält, `api` bei `/v1/chat/completions`-Backends auf `"openai-completions"` gesetzt ist und `models.providers.<provider>.models[].id` die unveränderte providerlokale ID ist. Wählen Sie sie einmal mit dem Provider-Präfix aus, beispielsweise `mlx/mlx-community/Qwen3-30B-A3B-6bit`; belassen Sie den Katalogeintrag als `mlx-community/Qwen3-30B-A3B-6bit`.
    - `messages[...].content: invalid type: sequence, expected a string`: Das Backend lehnt strukturierte Inhaltsteile von Chat Completions ab. Behebung: Legen Sie `models.providers.<provider>.models[].compat.requiresStringContent: true` fest.
    - `validation.keys` oder zulässige Nachrichtenschlüssel wie `["role","content"]`: Das Backend lehnt OpenAI-typische Wiedergabemetadaten in Chat-Completions-Nachrichten ab. Behebung: Legen Sie `models.providers.<provider>.models[].compat.strictMessageKeys: true` fest.
    - `incomplete turn detected ... stopReason=stop payloads=0`: Das Backend hat die Chat-Completions-Anfrage abgeschlossen, für diesen Zug jedoch keinen für Benutzer sichtbaren Assistententext zurückgegeben. OpenClaw wiederholt wiedergabesichere leere OpenAI-kompatible Züge einmal; anhaltende Fehler bedeuten üblicherweise, dass das Backend leere oder nicht textuelle Inhalte ausgibt oder den Text der endgültigen Antwort unterdrückt.
    - Kleine direkte Anfragen sind erfolgreich, OpenClaw-Agentenläufe schlagen jedoch mit Backend- oder Modellabstürzen fehl (beispielsweise Gemma bei einigen `inferrs`-Builds): Der OpenClaw-Transport ist wahrscheinlich bereits korrekt; das Backend scheitert an der größeren Prompt-Struktur der Agentenlaufzeit.
    - Die Anzahl der Fehler nimmt nach der Deaktivierung von Werkzeugen ab, sie verschwinden jedoch nicht: Werkzeugschemas trugen zur Belastung bei, das verbleibende Problem ist jedoch weiterhin die Kapazität des vorgelagerten Modells oder Servers oder ein Backend-Fehler.

  </Accordion>
  <Accordion title="Behebungsoptionen">
    1. Legen Sie `compat.requiresStringContent: true` für Chat-Completions-Backends fest, die nur Zeichenfolgen unterstützen.
    2. Legen Sie `compat.strictMessageKeys: true` für strikte Chat-Completions-Backends fest, die für jede Nachricht nur `role` und `content` akzeptieren.
    3. Legen Sie `compat.supportsTools: false` für Modelle oder Backends fest, die die Werkzeugschema-Oberfläche von OpenClaw nicht zuverlässig verarbeiten können.
    4. Reduzieren Sie nach Möglichkeit die Prompt-Belastung: kleinerer Workspace-Bootstrap, kürzerer Sitzungsverlauf, weniger anspruchsvolles lokales Modell oder ein Backend mit besserer Unterstützung langer Kontexte.
    5. Wenn kleine direkte Anfragen weiterhin erfolgreich sind, OpenClaw-Agentenzüge jedoch nach wie vor innerhalb des Backends abstürzen, behandeln Sie dies als Einschränkung des vorgelagerten Servers oder Modells und reichen Sie dort einen reproduzierbaren Fehlerbericht mit der akzeptierten Nutzlaststruktur ein.
  </Accordion>
</AccordionGroup>

Verwandte Themen:

- [Konfiguration](/de/gateway/configuration)
- [Lokale Modelle](/de/gateway/local-models)
- [OpenAI-kompatible Endpunkte](/de/gateway/configuration-reference#openai-compatible-endpoints)

## Keine Antworten

Wenn die Kanäle aktiv sind, aber keine Antwort erfolgt, prüfen Sie Routing und Richtlinien, bevor Sie irgendetwas neu verbinden.

```bash
openclaw status
openclaw channels status --probe
openclaw pairing list --channel <channel> [--account <id>]
openclaw config get channels
openclaw logs --follow
```

Achten Sie auf:

- Ausstehendes Pairing für DM-Absender.
- Erwähnungssperre in Gruppen (`requireMention`, `mentionPatterns`).
- Nicht übereinstimmende Kanal-/Gruppen-Zulassungslisten.

Häufige Meldungen:

- `drop guild message (mention required` → Gruppennachricht wird bis zu einer Erwähnung ignoriert.
- `pairing request` → Absender benötigt eine Genehmigung.
- `blocked` / `allowlist` → Absender/Kanal wurde durch eine Richtlinie herausgefiltert.

Verwandte Themen:

- [Fehlerbehebung für Kanäle](/de/channels/troubleshooting)
- [Gruppen](/de/channels/groups)
- [Pairing](/de/channels/pairing)

## Konnektivität der Dashboard-Steuerungsoberfläche

Wenn das Dashboard bzw. die Steuerungsoberfläche keine Verbindung herstellt, prüfen Sie URL, Authentifizierungsmodus und Annahmen zum sicheren Kontext.

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
openclaw doctor
openclaw gateway status --json
```

Achten Sie auf:

- Korrekte Prüf-URL und Dashboard-URL.
- Nicht übereinstimmender Authentifizierungsmodus bzw. Token zwischen Client und Gateway.
- Verwendung von HTTP, obwohl eine Geräteidentität erforderlich ist.

Wenn ein lokaler Browser nach einer Aktualisierung keine Verbindung zu `127.0.0.1:18789` herstellen kann, stellen Sie zuerst den lokalen Gateway-Dienst wieder her und bestätigen Sie, dass er das Dashboard bereitstellt:

```bash
openclaw gateway restart
lsof -i :18789
curl http://127.0.0.1:18789
```

Wenn `curl` OpenClaw-HTML zurückgibt, funktioniert das Gateway, und das verbleibende Problem ist wahrscheinlich der Browser-Cache, ein alter Deep Link oder ein veralteter Tab-Zustand. Öffnen Sie `http://127.0.0.1:18789` direkt und navigieren Sie vom Dashboard aus. Wenn der Dienst nach dem Neustart nicht weiterläuft, führen Sie `openclaw gateway start` aus und prüfen Sie `openclaw gateway status` erneut.

<AccordionGroup>
  <Accordion title="Verbindungs-/Authentifizierungsmeldungen">
    - `device identity required` → unsicherer Kontext oder fehlende Geräteauthentifizierung.
    - `origin not allowed` → Browser-`Origin` ist nicht in `gateway.controlUi.allowedOrigins` enthalten (oder Sie stellen die Verbindung von einem Browser-Ursprung außerhalb der Loopback-Schnittstelle ohne explizite Zulassungsliste her).
    - `device nonce required` / `device nonce mismatch` → der Client schließt den Challenge-basierten Ablauf zur Geräteauthentifizierung nicht ab (`connect.challenge` + `device.nonce`).
    - `device signature invalid` / `device signature expired` → der Client hat für den aktuellen Handshake die falsche Nutzlast (oder einen veralteten Zeitstempel) signiert.
    - `AUTH_TOKEN_MISMATCH` mit `canRetryWithDeviceToken=true` → der Client kann einen einzelnen vertrauenswürdigen Wiederholungsversuch mit dem zwischengespeicherten Geräte-Token durchführen.
    - Bei diesem Wiederholungsversuch mit zwischengespeichertem Token wird der mit dem gekoppelten Geräte-Token gespeicherte Scope-Satz wiederverwendet. Aufrufer mit explizitem `deviceToken` / explizitem `scopes` behalten stattdessen ihren angeforderten Scope-Satz bei.
    - `AUTH_SCOPE_MISMATCH` → das Geräte-Token wurde erkannt, aber seine genehmigten Scopes decken diese Verbindungsanfrage nicht ab; koppeln Sie das Gerät erneut oder genehmigen Sie den angeforderten Scope-Vertrag, statt ein gemeinsam verwendetes Gateway-Token zu rotieren.
    - Außerhalb dieses Wiederholungspfads gilt für die Verbindungsauthentifizierung folgende Rangfolge: zuerst explizites gemeinsam verwendetes Token/Passwort, dann explizites `deviceToken`, dann gespeichertes Geräte-Token und schließlich Bootstrap-Token.
    - Im asynchronen Tailscale-Serve-Pfad der Steuerungsoberfläche werden fehlgeschlagene Versuche für dasselbe `{scope, ip}` serialisiert, bevor der Begrenzer den Fehler erfasst. Zwei gleichzeitig ausgeführte fehlerhafte Wiederholungsversuche desselben Clients können daher beim zweiten Versuch `retry later` statt zweier einfacher Nichtübereinstimmungen ausgeben.
    - `too many failed authentication attempts (retry later)` von einem Loopback-Client mit Browser-Ursprung → wiederholte Fehler von demselben normalisierten `Origin` werden vorübergehend gesperrt; ein anderer localhost-Ursprung verwendet einen separaten Bucket.
    - Wiederholtes `unauthorized` nach diesem Wiederholungsversuch → Abweichung zwischen gemeinsam verwendetem Token und Geräte-Token; aktualisieren Sie die Token-Konfiguration und genehmigen oder rotieren Sie das Geräte-Token bei Bedarf erneut.
    - `gateway connect failed:` → falsches Host-/Port-/URL-Ziel.

  </Accordion>
</AccordionGroup>

### Schnellübersicht der Authentifizierungsdetailcodes

Verwenden Sie `error.details.code` aus der fehlgeschlagenen `connect`-Antwort, um die nächste Aktion auszuwählen:

| Detailcode                  | Bedeutung                                                                                                                                                                                      | Empfohlene Aktion                                                                                                                                                                                                                                                                       |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `AUTH_TOKEN_MISSING`         | Der Client hat kein erforderliches gemeinsam verwendetes Token gesendet.                                                                                                                                                 | Fügen Sie das Token im Client ein bzw. legen Sie es dort fest und versuchen Sie es erneut. Für Dashboard-Pfade: `openclaw config get gateway.auth.token`, anschließend in die Einstellungen der Steuerungsoberfläche einfügen.                                                                                                                                              |
| `AUTH_TOKEN_MISMATCH`        | Das gemeinsam verwendete Token stimmte nicht mit dem Gateway-Authentifizierungs-Token überein.                                                                                                                                               | Falls `canRetryWithDeviceToken=true`, erlauben Sie einen einzelnen vertrauenswürdigen Wiederholungsversuch. Wiederholungsversuche mit zwischengespeicherten Token verwenden gespeicherte genehmigte Scopes erneut; Aufrufer mit explizitem `deviceToken` / `scopes` behalten die angeforderten Scopes bei. Falls der Fehler weiterhin auftritt, führen Sie die [Checkliste zur Wiederherstellung bei Token-Abweichungen](/de/cli/devices#token-drift-recovery-checklist) aus. |
| `AUTH_DEVICE_TOKEN_MISMATCH` | Das zwischengespeicherte gerätespezifische Token ist veraltet oder wurde widerrufen.                                                                                                                                                 | Rotieren bzw. genehmigen Sie das Geräte-Token mithilfe der [Geräte-CLI](/de/cli/devices) erneut und stellen Sie danach die Verbindung wieder her.                                                                                                                                                                                                        |
| `AUTH_SCOPE_MISMATCH`        | Das Geräte-Token ist gültig, aber seine genehmigte Rolle bzw. seine genehmigten Scopes decken diese Verbindungsanfrage nicht ab.                                                                                                       | Koppeln Sie das Gerät erneut oder genehmigen Sie den angeforderten Scope-Vertrag; behandeln Sie dies nicht als Abweichung des gemeinsam verwendeten Tokens.                                                                                                                                                                                     |
| `PAIRING_REQUIRED`           | Die Geräteidentität muss genehmigt werden. Prüfen Sie `error.details.reason` auf `not-paired`, `scope-upgrade`, `role-upgrade` oder `metadata-upgrade` und verwenden Sie `requestId` / `remediationHint`, sofern vorhanden. | Genehmigen Sie die ausstehende Anfrage: `openclaw devices list`, anschließend `openclaw devices approve <requestId>`. Für Scope-/Rollen-Upgrades gilt derselbe Ablauf, nachdem Sie den angeforderten Zugriff geprüft haben.                                                                                                               |

<Note>
Direkte Loopback-Backend-RPCs, die mit dem gemeinsam verwendeten Gateway-Token/Passwort authentifiziert werden, sollten nicht von der Scope-Basis der gekoppelten Geräte der CLI abhängen. Wenn Subagenten oder andere interne Aufrufe weiterhin mit `scope-upgrade` fehlschlagen, überprüfen Sie, ob der Aufrufer `client.id: "gateway-client"` und `client.mode: "backend"` verwendet und nicht explizit ein `deviceIdentity` oder Geräte-Token erzwingt.
</Note>

Prüfung der Migration auf Geräteauthentifizierung v2:

```bash
openclaw --version
openclaw doctor
openclaw gateway status
```

Wenn die Protokolle Nonce-/Signaturfehler anzeigen, aktualisieren Sie den verbindenden Client und überprüfen Sie ihn:

<Steps>
  <Step title="Auf connect.challenge warten">
    Der Client wartet auf das vom Gateway ausgegebene `connect.challenge`.
  </Step>
  <Step title="Nutzlast signieren">
    Der Client signiert die an die Challenge gebundene Nutzlast.
  </Step>
  <Step title="Geräte-Nonce senden">
    Der Client sendet `connect.params.device.nonce` mit derselben Challenge-Nonce.
  </Step>
</Steps>

Wenn `openclaw devices rotate` / `revoke` / `remove` unerwartet abgelehnt wird:

- Sitzungen mit Token gekoppelter Geräte können nur **ihr eigenes** Gerät verwalten, sofern der Aufrufer nicht zusätzlich über `operator.admin` verfügt.
- `openclaw devices rotate --scope ...` kann nur Operator-Scopes anfordern, über die die Aufrufersitzung bereits verfügt.

Verwandte Themen:

- [Konfiguration](/de/gateway/configuration) (Gateway-Authentifizierungsmodi)
- [Steuerungsoberfläche](/de/web/control-ui)
- [Geräte](/de/cli/devices)
- [Remotezugriff](/de/gateway/remote)
- [Authentifizierung über vertrauenswürdigen Proxy](/de/gateway/trusted-proxy-auth)

## Gateway-Dienst wird nicht ausgeführt

Verwenden Sie diesen Abschnitt, wenn der Dienst installiert ist, der Prozess jedoch nicht aktiv bleibt.

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
openclaw doctor
openclaw gateway status --deep   # auch Dienste auf Systemebene prüfen
```

Achten Sie auf:

- `Runtime: stopped` mit Hinweisen zum Beenden.
- Nicht übereinstimmende Dienstkonfiguration (`Config (cli)` gegenüber `Config (service)`).
- Port-/Listener-Konflikte.
- Zusätzliche launchd-/systemd-/schtasks-Installationen bei Verwendung von `--deep`.
- `Other gateway-like services detected (best effort)`-Bereinigungshinweise.

<AccordionGroup>
  <Accordion title="Häufige Meldungen">
    - `Gateway start blocked: set gateway.mode=local` oder `existing config is missing gateway.mode` → der lokale Gateway-Modus ist nicht aktiviert oder die Konfigurationsdatei wurde überschrieben und `gateway.mode` ging verloren. Abhilfe: Legen Sie `gateway.mode="local"` in Ihrer Konfiguration fest oder führen Sie `openclaw onboard --mode local` / `openclaw setup` erneut aus, um die erwartete Konfiguration für den lokalen Modus wiederherzustellen. Wenn Sie OpenClaw über Podman ausführen, lautet der standardmäßige Konfigurationspfad `~/.openclaw/openclaw.json`.
    - `refusing to bind gateway ... without auth` → Bindung außerhalb der Loopback-Schnittstelle ohne gültigen Gateway-Authentifizierungspfad (Token/Passwort oder, sofern konfiguriert, vertrauenswürdiger Proxy).
    - `another gateway instance is already listening` / `EADDRINUSE` → Portkonflikt.
    - `Other gateway-like services detected (best effort)` → veraltete oder parallele launchd-/systemd-/schtasks-Einheiten sind vorhanden. In den meisten Setups sollte pro Computer nur ein Gateway verwendet werden. Falls Sie mehrere benötigen, isolieren Sie Ports sowie Konfiguration, Zustand und Workspace. Siehe [/gateway#multiple-gateways-same-host](/de/gateway#multiple-gateways-same-host).
    - `System-level OpenClaw gateway service detected` von Doctor → eine systemweite systemd-Einheit ist vorhanden, während der Dienst auf Benutzerebene fehlt. Entfernen oder deaktivieren Sie das Duplikat, bevor Sie Doctor die Installation eines Benutzerdienstes erlauben, oder legen Sie `OPENCLAW_SERVICE_REPAIR_POLICY=external` fest, wenn die Systemeinheit als Supervisor vorgesehen ist.
    - `Gateway service port does not match current gateway config` → der installierte Supervisor ist weiterhin auf das alte `--port` festgelegt. Führen Sie `openclaw doctor --fix` oder `openclaw gateway install --force` aus und starten Sie anschließend den Gateway-Dienst neu.

  </Accordion>
</AccordionGroup>

Verwandte Themen:

- [Hintergrundausführung und Prozesswerkzeug](/de/gateway/background-process)
- [Konfiguration](/de/gateway/configuration)
- [Doctor](/de/gateway/doctor)

## Das macOS-Gateway reagiert ohne Meldung nicht mehr und setzt den Betrieb fort, sobald Sie das Dashboard verwenden

Verwenden Sie dies, wenn Kanäle (Telegram, WhatsApp usw.) auf einem macOS-Host minuten- bis stundenlang verstummen und der Gateway scheinbar genau dann wieder aktiv wird, wenn Sie die Control UI öffnen, sich per SSH anmelden oder anderweitig mit dem Host interagieren. In `openclaw status` ist normalerweise kein offensichtliches Symptom zu sehen, da der Gateway bereits wieder aktiv ist, sobald Sie nachsehen.

```bash
ls ~/.openclaw/logs/stability/ | tail -5
openclaw gateway stability --bundle latest
pmset -g log | grep -iE "sleep|wake|maintenance" | tail -50
launchctl print gui/$UID/ai.openclaw.gateway | grep -E "state|last exit|runs"
```

Achten Sie auf Folgendes:

- Ein oder mehrere `*-uncaught_exception.json`-Bundles in `~/.openclaw/logs/stability/`, bei denen `error.code` auf einen vorübergehenden Netzwerkcode wie `ENETDOWN`, `ENETUNREACH`, `EHOSTUNREACH` oder `ECONNREFUSED` gesetzt ist.
- `pmset -g log`-Zeilen wie `Entering Sleep state due to 'Maintenance Sleep'` oder `en0 driver is slow (msg: WillChangeState to 0)`, die zeitlich mit den Abstürzen übereinstimmen. Power Nap / Maintenance Sleep versetzt den WLAN-Treiber kurzzeitig in Zustand 0; jeder ausgehende `connect()`, der in dieses Zeitfenster fällt, kann mit `ENETDOWN` fehlschlagen, selbst wenn der Host ansonsten über vollständige Netzwerkkonnektivität verfügt.
- `launchctl print`-Ausgabe, die `state = not running` mit mehreren kürzlichen `runs` und einem Exit-Code zeigt, insbesondere wenn zwischen dem Absturz und dem nächsten Start etwa eine Stunde statt nur wenige Sekunden liegt. macOS launchd wendet nach einer Serie von Abstürzen eine undokumentierte Schutzsperre für Neustarts an, durch die `KeepAlive=true` möglicherweise nicht mehr berücksichtigt wird, bis ein externer Auslöser wie eine interaktive Anmeldung, eine Dashboard-Verbindung oder `launchctl kickstart` die Sperre wieder aktiviert.

Häufige Merkmale:

- Ein Stabilitäts-Bundle, dessen `error.code` den Wert `ENETDOWN` oder einen verwandten Code enthält und dessen Aufrufstapel auf Node `net` `lookupAndConnect` / `Socket.connect` verweist. OpenClaw `2026.5.26` und neuere Versionen klassifizieren diese als harmlose vorübergehende Netzwerkfehler, sodass sie nicht mehr bis zum obersten Handler für nicht abgefangene Fehler weitergegeben werden. Wenn Sie eine ältere Version verwenden, führen Sie zuerst ein Upgrade durch.
- Lange Ruhephasen, die in dem Moment enden, in dem Sie eine Verbindung zur Control UI herstellen oder sich per SSH am Host anmelden: Die für Benutzer sichtbare Aktivität reaktiviert die Neustartsperre von launchd, nicht eine Aktion des Dashboards am Gateway.
- Der Zähler `runs` steigt im Laufe des Tages, ohne dass eine entsprechende `received SIG*; shutting down`-Zeile in `~/Library/Logs/openclaw/gateway.log` vorhanden ist: Bei ordnungsgemäßem Herunterfahren wird ein Signal protokolliert, bei vorübergehenden Abstürzen nicht.

Vorgehensweise:

1. **Führen Sie ein Upgrade des Gateways durch**, wenn Sie eine Version vor `2026.5.26` verwenden. Nach dem Upgrade werden zukünftige `ENETDOWN`-Fehler als Warnungen protokolliert, statt den Prozess zu beenden.
2. **Reduzieren Sie die Aktivität des Wartungsruhezustands** auf Mac-mini-/Desktop-Hosts, die als dauerhaft verfügbare Server betrieben werden sollen:

   ```bash
   sudo pmset -a sleep 0 disksleep 0 standby 0 powernap 0
   ```

   Dadurch wird die zugrunde liegende Treiberunterbrechung deutlich reduziert, jedoch nicht vollständig beseitigt. Unabhängig von diesen Flags kann das System weiterhin einige Wartungsruhezustände für TCP-Keepalive und die mDNS-Wartung ausführen.

3. **Fügen Sie einen Verfügbarkeits-Watchdog hinzu**, damit eine zukünftige Absturzserie, die von launchd angehalten wird, schnell erkannt wird:

   ```bash
   # Beispiel für eine launchd-fähige Verfügbarkeitsprüfung, geeignet für einen 5-minütigen Cron oder LaunchAgent
   state=$(launchctl print gui/$UID/ai.openclaw.gateway 2>/dev/null | awk -F'= ' '/state =/ {print $2; exit}')
   if [ "$state" != "running" ]; then
     launchctl kickstart -k gui/$UID/ai.openclaw.gateway
   fi
   ```

   Ziel ist es, die Neustartsperre extern zu reaktivieren. `KeepAlive=true` allein reicht unter macOS nach einer Absturzserie nicht aus.

Verwandte Themen:

- [Hinweise zur macOS-Plattform](/de/platforms/macos)
- [Protokollierung](/de/logging)
- [Doctor](/de/gateway/doctor)

## macOS-launchd-Supervisorschleife mit doppelten Gateway-/Node-LaunchAgents

Verwenden Sie dies, wenn eine macOS-Installation alle paar Sekunden neu startet, `openclaw`
Integritätsprüfungen zwischen „fehlerfrei“ und „nicht verfügbar“ wechseln und die Kanalauslieferung stockt,
obwohl der Dienst scheinbar ausgeführt wird.

Dies wurde bei älteren Installationen beobachtet, bei denen sowohl `ai.openclaw.gateway` als auch
`ai.openclaw.node` als LaunchAgents aktiv waren und jeweils
`OPENCLAW_LAUNCHD_LABEL` einschleusten. In diesem Zustand kann OpenClaw die
Überwachung durch launchd erkennen, versuchen, den Neustart wieder an launchd zu übergeben, und statt eines
stabilen Gateway-Prozesses in eine schnelle `EADDRINUSE`-/Neustartschleife geraten.

```bash
for i in 1 2 3 4; do
  ps aux | grep 'openclaw.*index.js' | grep -v grep | awk '{print $2}'
  sleep 10
done

openclaw gateway status --deep
openclaw node status
launchctl print gui/$UID/ai.openclaw.gateway | grep -E 'state|last exit|runs'
tail -n 80 ~/Library/Logs/openclaw/gateway.log
```

Achten Sie auf Folgendes:

- Mehr als eine Gateway-PID während der 30-sekündigen Stichprobe statt eines stabilen
  Prozesses.
- `EADDRINUSE`, `another gateway instance is already listening` oder wiederholte
  Neustart-/Übergabezeilen in `gateway.log`.
- Sowohl `~/Library/LaunchAgents/ai.openclaw.gateway.plist` als auch
  `~/Library/LaunchAgents/ai.openclaw.node.plist` sind gleichzeitig auf einem
  Host geladen, auf dem nur ein verwalteter Gateway-Dienst ausgeführt werden sollte.

Vorgehensweise:

1. Wenn auf diesem Host nur der Gateway-Dienst ausgeführt werden soll, entfernen Sie den verwalteten Node-
   Dienst über OpenClaw. **Überspringen Sie diesen Schritt**, wenn Sie den Node-
   Dienst aktiv für Remote-Node-Funktionen verwenden. Durch seine Deinstallation werden diese Funktionen auf
   diesem Host beendet:

   ```bash
   openclaw node uninstall
   ```

2. Installieren Sie einen persistenten Gateway-Wrapper, der die geerbten launchd-
   Markierungen löscht, bevor OpenClaw gestartet wird. Verwenden Sie die unterstützte Option `--wrapper`;
   bearbeiten Sie nicht die generierte Datei unter `~/.openclaw/service-env/`, da diese Datei bei der
   Neuinstallation des Dienstes, bei Updates und bei Reparaturen durch Doctor neu generiert wird:

   ```bash
   mkdir -p ~/.local/bin
   cat >~/.local/bin/openclaw-launchd-workaround <<'EOF'
   #!/bin/sh
   set -eu
   unset OPENCLAW_LAUNCHD_LABEL LAUNCH_JOB_LABEL LAUNCH_JOB_NAME XPC_SERVICE_NAME || true
   exec openclaw "$@"
   EOF
   chmod 700 ~/.local/bin/openclaw-launchd-workaround

   openclaw gateway install \
     --wrapper ~/.local/bin/openclaw-launchd-workaround \
     --force
   ```

   `gateway install` behält den Wrapper-Pfad über erzwungene Neuinstallationen,
   Updates und Reparaturen durch Doctor hinweg bei.

3. Überprüfen Sie, ob der Gateway stabil ist und RPC bereitstellt, statt lediglich auf Verbindungen zu warten:

   ```bash
   openclaw gateway status --deep --require-rpc

   for i in 1 2 3 4; do
     ps aux | grep 'openclaw.*index.js' | grep -v grep | awk '{print $2}'
     sleep 10
   done
   ```

   Die PID-Stichprobe sollte einen einzelnen stabilen Prozess statt einer wechselnden Gruppe von
   PIDs zeigen, und die eingehende Kanalauslieferung sollte fortgesetzt werden.

4. Entfernen Sie nach dem Upgrade auf eine Version, in der die zugrunde liegende Schleife aus zwei LaunchAgents
   behoben ist, die Problemumgehung und installieren Sie den normalen verwalteten Dienst erneut:

   ```bash
   OPENCLAW_WRAPPER= openclaw gateway install --force
   rm ~/.local/bin/openclaw-launchd-workaround
   ```

Verwandte Themen:

- [Hinweise zur macOS-Plattform](/de/platforms/mac/bundled-gateway)
- [Doctor](/de/gateway/doctor)
- [Gateway-CLI](/de/cli/gateway)

## Gateway wird bei hoher Speicherauslastung beendet

Verwenden Sie dies, wenn der Gateway unter Last verschwindet, der Supervisor einen Neustart nach Art eines OOM meldet oder die Protokolle `critical memory pressure bundle written` erwähnen.

```bash
openclaw gateway status --deep
openclaw logs --follow
openclaw gateway stability --bundle latest
openclaw gateway diagnostics export
```

Achten Sie auf Folgendes:

- `Reason: diagnostic.memory.pressure.critical` im neuesten Stabilitäts-Bundle.
- `Memory pressure:` mit `critical/rss_threshold`, `critical/heap_threshold` oder `critical/rss_growth`.
- `V8 heap:`-Werte nahe am Heap-Limit.
- `Largest session files:`-Einträge wie `agents/<agent>/sessions/<session>.jsonl` oder `sessions/<session>.jsonl`.
- Linux-cgroup-Speicherzähler, wenn der Gateway in einem Container oder einem Dienst mit Speicherbegrenzung ausgeführt wird.

Häufige Merkmale:

- `critical memory pressure bundle written` erscheint kurz vor dem Neustart → OpenClaw hat vor dem OOM ein Stabilitäts-Bundle erfasst. Untersuchen Sie es mit `openclaw gateway stability --bundle latest`.
- `memory pressure: level=critical` erscheint in den Gateway-Protokollen → OpenClaw hat kritischen Speicherdruck erkannt und die verfügbaren prozessinternen Speicherdaten aufgezeichnet.
- `Largest session files:` verweist auf einen sehr großen, redigierten Transkriptpfad → Reduzieren Sie den gespeicherten Sitzungsverlauf, untersuchen Sie das Sitzungswachstum oder verschieben Sie alte Transkripte aus dem aktiven Speicher, bevor Sie neu starten.
- Die von `V8 heap:` verwendeten Bytes liegen nahe am Heap-Limit → Reduzieren Sie zuerst den Prompt-/Sitzungsdruck oder die Anzahl gleichzeitiger Aufgaben. Prüfen Sie bei einem verwalteten Dienst `Gateway heap:` in `openclaw gateway status`. Wenn dort `not set` angegeben ist, generieren Sie alte Dienstmetadaten mit `openclaw gateway install --force` neu. `NODE_OPTIONS` aus der Shell-Umgebung wird absichtlich ignoriert. Verwenden Sie eine explizite Heap-Überschreibung auf Supervisor-Ebene erst, nachdem Sie die dauerhafte Arbeitslast bestätigt und ausreichend Spielraum für nativen Speicher eingeplant haben.
- `Memory pressure: critical/rss_growth` → Der Speicher ist innerhalb eines einzelnen Abtastintervalls schnell angewachsen. Prüfen Sie die neuesten Protokolle auf einen großen Import, unkontrollierte Tool-Ausgaben, wiederholte Wiederholungsversuche oder eine Reihe in die Warteschlange gestellter Agentenaufgaben.
- In den Protokollen erscheint kritischer Speicherdruck, aber es ist kein Bundle vorhanden → Erfassen Sie nach dem Ereignis `openclaw gateway diagnostics export`, um die verfügbaren Betriebsnachweise zu sichern.

Das Stabilitäts-Bundle enthält keine Nutzdaten. Es enthält betriebliche Speichernachweise und redigierte relative Dateipfade, jedoch keine Nachrichtentexte, Webhook-Inhalte, Anmeldedaten, Token, Cookies oder unverarbeiteten Sitzungs-IDs. Hängen Sie den Diagnoseexport an Fehlerberichte an, statt unverarbeitete Protokolle zu kopieren.

Verwandte Themen:

- [Gateway-Integrität](/de/gateway/health)
- [Diagnoseexport](/de/gateway/diagnostics)
- [Sitzungen](/de/cli/sessions)

## Gateway hat eine ungültige Konfiguration abgelehnt

Verwenden Sie dies, wenn der Start des Gateways mit `Invalid config` fehlschlägt oder die Protokolle des Hot Reload angeben, dass eine ungültige Änderung übersprungen wurde.

```bash
openclaw logs --follow
openclaw config file
openclaw config validate
openclaw doctor
```

Achten Sie auf Folgendes:

- `Invalid config at ...`
- `config reload skipped (invalid config): ...`
- `Config write rejected: ...`
- Eine mit einem Zeitstempel versehene `openclaw.json.rejected.*`-Datei neben der aktiven Konfiguration.
- Eine mit einem Zeitstempel versehene `openclaw.json.clobbered.*`-Datei, wenn `doctor --fix` eine fehlerhafte direkte Bearbeitung repariert hat.
- OpenClaw behält für jeden Konfigurationspfad die neuesten 32 `.clobbered.*`-Dateien bei und rotiert ältere Dateien.

<AccordionGroup>
  <Accordion title="Was ist passiert?">
    - Die Konfiguration konnte während des Starts, des Hot Reload oder eines von OpenClaw ausgeführten Schreibvorgangs nicht validiert werden.
    - Der Start des Gateways schlägt sicher geschlossen fehl, statt `openclaw.json` neu zu schreiben.
    - Der Hot Reload überspringt ungültige externe Änderungen und lässt die aktuelle Laufzeitkonfiguration aktiv.
    - Von OpenClaw ausgeführte Schreibvorgänge lehnen ungültige oder destruktive Nutzdaten vor dem Commit ab und speichern `.rejected.*`.
    - `openclaw doctor --fix` ist für die Reparatur zuständig. Es kann Präfixe entfernen, die nicht zu JSON gehören, oder die letzte als fehlerfrei bekannte Kopie wiederherstellen, während die abgelehnten Nutzdaten als `.clobbered.*` erhalten bleiben.
    - Wenn für einen Konfigurationspfad viele Reparaturen erfolgen, rotiert OpenClaw ältere `.clobbered.*`-Dateien, sodass die neuesten reparierten Nutzdaten weiterhin verfügbar sind.

  </Accordion>
  <Accordion title="Prüfen und reparieren">
    ```bash
    CONFIG="$(openclaw config file)"
    ls -lt "$CONFIG".clobbered.* "$CONFIG".rejected.* 2>/dev/null | head
    diff -u "$CONFIG" "$(ls -t "$CONFIG".clobbered.* 2>/dev/null | head -n 1)"
    openclaw config validate
    openclaw doctor
    ```
  </Accordion>
  <Accordion title="Häufige Anzeichen">
    - `.clobbered.*` ist vorhanden → Doctor hat eine fehlerhafte externe Bearbeitung beim Reparieren der aktiven Konfiguration beibehalten.
    - `.rejected.*` ist vorhanden → Ein OpenClaw-eigener Konfigurationsschreibvorgang hat vor dem Commit die Schema- oder Überschreibprüfungen nicht bestanden.
    - `Config write rejected:` → Der Schreibvorgang versuchte, eine erforderliche Struktur zu entfernen, die Datei stark zu verkleinern oder eine ungültige Konfiguration zu speichern.
    - `config reload skipped (invalid config):` → Eine direkte Bearbeitung hat die Validierung nicht bestanden und wurde vom laufenden Gateway ignoriert.
    - `Invalid config at ...` → Der Start schlug fehl, bevor die Gateway-Dienste gestartet wurden.
    - `missing-meta-vs-last-good`, `gateway-mode-missing-vs-last-good` oder `size-drop-vs-last-good:*` → Ein OpenClaw-eigener Schreibvorgang wurde abgelehnt, weil dabei im Vergleich zur letzten als fehlerfrei bekannten Sicherung Felder verloren gingen oder die Dateigröße abnahm.
    - `Config last-known-good promotion skipped` → Der Kandidat enthielt Platzhalter für geschwärzte Geheimnisse wie `***`.

  </Accordion>
  <Accordion title="Behebungsoptionen">
    1. Führen Sie `openclaw doctor --fix` aus, damit Doctor Konfigurationen mit Präfix oder Überschreibungen repariert oder den letzten als fehlerfrei bekannten Stand wiederherstellt.
    2. Kopieren Sie nur die vorgesehenen Schlüssel aus `.clobbered.*` oder `.rejected.*` und wenden Sie sie anschließend mit `openclaw config set` oder `config.patch` an.
    3. Führen Sie vor dem Neustart `openclaw config validate` aus.
    4. Wenn Sie die Datei manuell bearbeiten, behalten Sie die vollständige JSON5-Konfiguration bei, nicht nur das Teilobjekt, das Sie ändern wollten.
  </Accordion>
</AccordionGroup>

Verwandte Themen:

- [Konfiguration](/de/cli/config)
- [Konfiguration: Hot Reload](/de/gateway/configuration#config-hot-reload)
- [Konfiguration: strikte Validierung](/de/gateway/configuration#strict-validation)
- [Doctor](/de/gateway/doctor)

## Warnungen bei Gateway-Prüfungen

Verwenden Sie dies, wenn `openclaw gateway probe` etwas erreicht, aber weiterhin einen Warnungsblock ausgibt.

```bash
openclaw gateway probe
openclaw gateway probe --json
openclaw gateway probe --ssh user@gateway-host
```

Achten Sie auf:

- `warnings[].code` und `primaryTargetId` in der JSON-Ausgabe.
- Ob sich die Warnung auf den SSH-Fallback, mehrere Gateways, fehlende Scopes oder nicht aufgelöste Authentifizierungsreferenzen bezieht.

Häufige Anzeichen:

- `SSH tunnel failed to start; falling back to direct probes.` → Die SSH-Einrichtung ist fehlgeschlagen, der Befehl hat jedoch weiterhin direkte konfigurierte bzw. Loopback-Ziele versucht.
- `multiple reachable gateway identities detected` → Verschiedene Gateways haben geantwortet oder OpenClaw konnte nicht nachweisen, dass es sich bei den erreichbaren Zielen um dasselbe Gateway handelt. Ein SSH-Tunnel, eine Proxy-URL oder eine konfigurierte Remote-URL zum selben Gateway wird als ein Gateway mit mehreren Transportwegen behandelt, selbst wenn sich die Transport-Ports unterscheiden.
- `Read-probe diagnostics are limited by gateway scopes (missing operator.read)` → Die Verbindung wurde hergestellt, aber der Detail-RPC ist durch Scopes eingeschränkt; koppeln Sie die Geräteidentität oder verwenden Sie Zugangsdaten mit `operator.read`.
- `Gateway accepted the WebSocket connection, but follow-up read diagnostics failed` → Die Verbindung wurde hergestellt, aber für den vollständigen Satz diagnostischer RPCs trat eine Zeitüberschreitung oder ein Fehler auf. Behandeln Sie dies als erreichbares Gateway mit eingeschränkter Diagnosefunktion; vergleichen Sie `connect.ok` und `connect.rpcOk` in der Ausgabe von `--json`.
- `Capability: pairing-pending` oder `gateway closed (1008): pairing required` → Das Gateway hat geantwortet, aber dieser Client muss weiterhin gekoppelt bzw. genehmigt werden, bevor ein normaler Bedienerzugriff möglich ist.
- Nicht aufgelöster `gateway.auth.*`- / `gateway.remote.*`-SecretRef-Warntext → Das Authentifizierungsmaterial war in diesem Befehlspfad für das fehlgeschlagene Ziel nicht verfügbar.

Verwandte Themen:

- [Gateway](/de/cli/gateway)
- [Mehrere Gateways auf demselben Host](/de/gateway#multiple-gateways-same-host)
- [Remote-Zugriff](/de/gateway/remote)

## Kanal verbunden, aber Nachrichten werden nicht übertragen

Wenn der Kanalstatus „verbunden“ lautet, aber keine Nachrichten übertragen werden, konzentrieren Sie sich auf Richtlinien, Berechtigungen und kanalspezifische Zustellungsregeln.

```bash
openclaw channels status --probe
openclaw pairing list --channel <channel> [--account <id>]
openclaw status --deep
openclaw logs --follow
openclaw config get channels
```

Achten Sie auf:

- DM-Richtlinie (`pairing`, `allowlist`, `open`, `disabled`).
- Gruppen-Zulassungsliste und Anforderungen an Erwähnungen.
- Fehlende API-Berechtigungen/Scopes des Kanals.

Häufige Anzeichen:

- `mention required` → Die Nachricht wurde aufgrund der Richtlinie für Gruppenerwähnungen ignoriert.
- `pairing` / Spuren ausstehender Genehmigungen → Der Absender ist nicht genehmigt.
- `missing_scope`, `not_in_channel`, `Forbidden`, `401/403` → Problem mit der Kanalauthentifizierung oder den Kanalberechtigungen.

Verwandte Themen:

- [Fehlerbehebung für Kanäle](/de/channels/troubleshooting)
- [Discord](/de/channels/discord)
- [Telegram](/de/channels/telegram)
- [WhatsApp](/de/channels/whatsapp)

## Cron- und Heartbeat-Zustellung

Wenn Cron oder Heartbeat nicht ausgeführt oder nicht zugestellt wurde, überprüfen Sie zuerst den Scheduler-Status und anschließend das Zustellungsziel.

```bash
openclaw cron status
openclaw cron list
openclaw cron runs --id <jobId> --limit 20
openclaw system heartbeat last
openclaw logs --follow
```

Achten Sie auf:

- Cron ist aktiviert und der nächste Aktivierungszeitpunkt ist vorhanden.
- Status im Verlauf der Auftragsausführungen (`ok`, `skipped`, `error`).
- Gründe für das Überspringen des Heartbeats (`quiet-hours`, `requests-in-flight`, `cron-in-progress`, `lanes-busy`, `alerts-disabled`, `empty-heartbeat-file`).

<AccordionGroup>
  <Accordion title="Häufige Anzeichen">
    - `cron: scheduler disabled; jobs will not run automatically` → Cron ist deaktiviert.
    - `cron: timer tick failed` → Der Scheduler-Takt ist fehlgeschlagen; prüfen Sie auf Datei-, Protokoll- oder Laufzeitfehler.
    - `heartbeat skipped` mit `reason=quiet-hours` → Außerhalb des Zeitfensters der aktiven Stunden.
    - `heartbeat skipped` mit `reason=empty-heartbeat-file` → Der Entwurf der Heartbeat-Überwachung enthält nur leere Inhalte, Kommentare, Überschriften, Codeblöcke oder ein Gerüst aus einer leeren Checkliste, daher überspringt OpenClaw den Modellaufruf.
    - `heartbeat: unknown accountId` → Ungültige Konto-ID für das Heartbeat-Zustellungsziel.
    - `heartbeat skipped` mit `reason=dm-blocked` → Das Heartbeat-Ziel wurde als DM-artiges Ziel aufgelöst, während `agents.defaults.heartbeat.directPolicy` (oder die agentenspezifische Überschreibung) auf `block` gesetzt ist.

  </Accordion>
</AccordionGroup>

Verwandte Themen:

- [Heartbeat](/de/gateway/heartbeat)
- [Geplante Aufgaben](/de/automation/cron-jobs)
- [Geplante Aufgaben: Fehlerbehebung](/de/automation/cron-jobs#troubleshooting)

## Node gekoppelt, Tool schlägt fehl

Wenn eine Node gekoppelt ist, aber Tools fehlschlagen, grenzen Sie den Vordergrund-, Berechtigungs- und Genehmigungsstatus ein.

```bash
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
openclaw approvals get --node <idOrNameOrIp>
openclaw logs --follow
openclaw status
```

Achten Sie auf:

- Node ist mit den erwarteten Funktionen online.
- Betriebssystemberechtigungen für Kamera, Mikrofon, Standort und Bildschirm.
- Status der Ausführungsgenehmigungen und der Zulassungsliste.

Häufige Anzeichen:

- `NODE_BACKGROUND_UNAVAILABLE` → Die Node-App muss sich im Vordergrund befinden.
- `*_PERMISSION_REQUIRED` / `LOCATION_PERMISSION_REQUIRED` → Fehlende Betriebssystemberechtigung.
- `SYSTEM_RUN_DENIED: approval required` → Die Ausführungsgenehmigung steht aus.
- `SYSTEM_RUN_DENIED: allowlist miss` → Der Befehl wurde durch die Zulassungsliste blockiert.

Verwandte Themen:

- [Ausführungsgenehmigungen](/de/tools/exec-approvals)
- [Fehlerbehebung für Nodes](/de/nodes/troubleshooting)
- [Nodes](/de/nodes/index)

## Browser-Tool schlägt fehl

Verwenden Sie dies, wenn Aktionen des Browser-Tools fehlschlagen, obwohl das Gateway selbst fehlerfrei funktioniert.

```bash
openclaw browser status
openclaw browser start --browser-profile openclaw
openclaw browser profiles
openclaw logs --follow
openclaw doctor
```

Achten Sie auf:

- Ob `plugins.allow` festgelegt ist und `browser` enthält.
- Gültiger Pfad zur ausführbaren Browserdatei.
- Erreichbarkeit des CDP-Profils.
- Lokale Chrome-Verfügbarkeit für `existing-session`- / `user`-Profile.

<AccordionGroup>
  <Accordion title="Plugin- / Programmdatei-Anzeichen">
    - `unknown command "browser"` oder `unknown command 'browser'` → Das gebündelte Browser-Plugin wird durch `plugins.allow` ausgeschlossen.
    - Browser-Tool fehlt / ist nicht verfügbar, während `browser.enabled=true` → `plugins.allow` schließt `browser` aus, sodass das Plugin nie geladen wurde.
    - `Failed to start Chrome CDP on port` → Der Browserprozess konnte nicht gestartet werden.
    - `browser.executablePath not found` → Der konfigurierte Pfad ist ungültig.
    - `browser.cdpUrl must be http(s) or ws(s)` → Die konfigurierte CDP-URL verwendet ein nicht unterstütztes Schema wie `file:` oder `ftp:`.
    - `browser.cdpUrl has invalid port` → Die konfigurierte CDP-URL enthält einen ungültigen Port oder einen Port außerhalb des zulässigen Bereichs.
    - `Playwright is not available in this gateway build; '<feature>' is unsupported.` → Der aktuellen Gateway-Installation fehlt die Kernlaufzeit-Abhängigkeit für den Browser; installieren oder aktualisieren Sie OpenClaw erneut und starten Sie anschließend das Gateway neu. ARIA-Snapshots und einfache Seiten-Screenshots können weiterhin funktionieren, aber Navigation, KI-Snapshots, Element-Screenshots mit CSS-Selektoren und der PDF-Export bleiben nicht verfügbar.

  </Accordion>
  <Accordion title="Anzeichen für Chrome MCP / bestehende Sitzungen">
    - `Could not find DevToolsActivePort for chrome` → Die bestehende Chrome-MCP-Sitzung konnte noch keine Verbindung zum ausgewählten Browser-Datenverzeichnis herstellen. Öffnen Sie die Browser-Prüfseite, aktivieren Sie das Remote-Debugging, lassen Sie den Browser geöffnet, genehmigen Sie die erste Verbindungsanfrage und versuchen Sie es anschließend erneut. Wenn kein angemeldeter Zustand erforderlich ist, verwenden Sie vorzugsweise das verwaltete Profil `openclaw`.
    - `No browser tabs found for profile="user"` → Im Verbindungsprofil für Chrome MCP sind keine lokalen Chrome-Tabs geöffnet.
    - `Remote CDP for profile "<name>" is not reachable` → Der konfigurierte Remote-CDP-Endpunkt ist vom Gateway-Host aus nicht erreichbar.
    - `Browser attachOnly is enabled ... not reachable` oder `Browser attachOnly is enabled and CDP websocket ... is not reachable` → Das reine Verbindungsprofil hat kein erreichbares Ziel oder der HTTP-Endpunkt hat geantwortet, aber der CDP-WebSocket konnte dennoch nicht geöffnet werden.

  </Accordion>
  <Accordion title="Anzeichen für Elemente / Screenshots / Uploads">
    - `fullPage is not supported for element screenshots` → Die Screenshot-Anfrage kombinierte `--full-page` mit `--ref` oder `--element`.
    - `element screenshots are not supported for existing-session profiles; use ref from snapshot.` → Screenshot-Aufrufe von Chrome MCP / `existing-session` müssen die Seitenerfassung oder eine Snapshot-Referenz `--ref` verwenden, nicht den CSS-Selektor `--element`.
    - `existing-session file uploads do not support element selectors; use ref/inputRef.` → Chrome-MCP-Upload-Hooks benötigen Snapshot-Referenzen, keine CSS-Selektoren.
    - `existing-session file uploads currently support one file at a time.` → Senden Sie bei Chrome-MCP-Profilen einen Upload pro Aufruf.
    - `existing-session dialog handling does not support timeoutMs.` → Dialog-Hooks bei Chrome-MCP-Profilen unterstützen keine Überschreibungen des Zeitlimits.
    - `existing-session type does not support timeoutMs overrides.` → Lassen Sie `timeoutMs` für `act:type` bei `profile="user"`- / bestehenden Chrome-MCP-Sitzungsprofilen weg oder verwenden Sie ein verwaltetes/CDP-Browserprofil, wenn ein benutzerdefiniertes Zeitlimit erforderlich ist.
    - `response body is not supported for existing-session profiles yet.` → `responsebody` erfordert weiterhin ein verwaltetes Browserprofil oder ein unverarbeitetes CDP-Profil.
    - Veraltete Überschreibungen für Ansichtsbereich / Dunkelmodus / Gebietsschema / Offlinemodus bei reinen Verbindungs- oder Remote-CDP-Profilen → Führen Sie `openclaw browser stop --browser-profile <name>` aus, um die aktive Steuerungssitzung zu schließen und den Playwright-/CDP-Emulationsstatus freizugeben, ohne das gesamte Gateway neu zu starten.

  </Accordion>
</AccordionGroup>

Verwandte Themen:

- [Browser (von OpenClaw verwaltet)](/de/tools/browser)
- [Fehlerbehebung für Browser](/de/tools/browser-linux-troubleshooting)

## Wenn nach einem Upgrade plötzlich etwas nicht mehr funktioniert

Die meisten Probleme nach einem Upgrade entstehen durch Konfigurationsabweichungen oder dadurch, dass strengere Standardwerte nun durchgesetzt werden.

<AccordionGroup>
  <Accordion title="1. Verhalten der Authentifizierungs- und URL-Überschreibungen wurde geändert">
    ```bash
    openclaw gateway status
    openclaw config get gateway.mode
    openclaw config get gateway.remote.url
    openclaw config get gateway.auth.mode
    ```

    Zu prüfen:

    - Wenn `gateway.mode=remote`, richten sich CLI-Aufrufe möglicherweise an eine Remote-Instanz, während Ihr lokaler Dienst ordnungsgemäß funktioniert.
    - Explizite `--url`-Aufrufe greifen nicht ersatzweise auf gespeicherte Anmeldedaten zurück.

    Häufige Anzeichen:

    - `gateway connect failed:` → falsche Ziel-URL.
    - `unauthorized` → Endpunkt erreichbar, aber falsche Authentifizierung.

  </Accordion>
  <Accordion title="2. Schutzmechanismen für Bindung und Authentifizierung sind strenger">
    ```bash
    openclaw config get gateway.bind
    openclaw config get gateway.auth.mode
    openclaw config get gateway.auth.token
    openclaw gateway status
    openclaw logs --follow
    ```

    Zu prüfen:

    - Nicht an Loopback gebundene Adressen (`lan`, `tailnet`, `custom`) benötigen einen gültigen Gateway-Authentifizierungspfad: Authentifizierung mit gemeinsamem Token/Passwort oder eine korrekt konfigurierte Nicht-Loopback-`trusted-proxy`-Bereitstellung.
    - Alte Schlüssel wie `gateway.token` ersetzen `gateway.auth.token` nicht.

    Häufige Anzeichen:

    - `refusing to bind gateway ... without auth` → Nicht-Loopback-Bindung ohne gültigen Gateway-Authentifizierungspfad.
    - `Connectivity probe: failed`, während die Laufzeit ausgeführt wird → Gateway aktiv, aber mit der aktuellen Authentifizierung/URL nicht erreichbar.

  </Accordion>
  <Accordion title="3. Kopplungs- und Geräteidentitätsstatus haben sich geändert">
    ```bash
    openclaw devices list
    openclaw pairing list --channel <channel> [--account <id>]
    openclaw logs --follow
    openclaw doctor
    ```

    Zu prüfen:

    - Ausstehende Gerätefreigaben für Dashboard/Nodes.
    - Ausstehende Freigaben für die DM-Kopplung nach Richtlinien- oder Identitätsänderungen.

    Häufige Anzeichen:

    - `device identity required` → Geräteauthentifizierung nicht erfüllt.
    - `pairing required` → Absender/Gerät muss freigegeben werden.

  </Accordion>
</AccordionGroup>

Wenn Dienstkonfiguration und Laufzeit nach diesen Prüfungen weiterhin voneinander abweichen, installieren Sie die Dienstmetadaten aus demselben Profil-/Statusverzeichnis neu:

```bash
openclaw gateway install --force
openclaw gateway restart
```

Verwandte Themen:

- [Authentifizierung](/de/gateway/authentication)
- [Hintergrundausführung und Prozesswerkzeug](/de/gateway/background-process)
- [Node-Kopplung](/de/gateway/pairing)

## Verwandte Themen

- [Doctor](/de/gateway/doctor)
- [Häufig gestellte Fragen](/de/help/faq)
- [Gateway-Betriebshandbuch](/de/gateway)
