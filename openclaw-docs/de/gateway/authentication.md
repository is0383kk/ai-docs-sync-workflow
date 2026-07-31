---
read_when:
    - Fehlerbehebung bei der Modellauthentifizierung oder beim Ablauf von OAuth-Zugängen
    - Dokumentation der Authentifizierung oder Anmeldedatenspeicherung
summary: 'Modellauthentifizierung: OAuth, API-Schlüssel, Wiederverwendung der Claude-CLI und Anthropic-Einrichtungstoken'
title: Authentifizierung
x-i18n:
    generated_at: "2026-07-26T18:26:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1fd4bf1c73f41d297638811f568c1b11e920eba3bd1527206cbb760df51531f2
    source_path: gateway/authentication.md
    workflow: 16
---

<Note>
Diese Seite behandelt die Authentifizierung bei **Modell-Providern** (API-Schlüssel, OAuth, Wiederverwendung der Claude CLI, Anthropic-Setup-Token). Informationen zur Authentifizierung der **Gateway-Verbindung** (Token, Passwort, vertrauenswürdiger Proxy) finden Sie unter [Konfiguration](/de/gateway/configuration) und [Authentifizierung über einen vertrauenswürdigen Proxy](/de/gateway/trusted-proxy-auth).
</Note>

OpenClaw unterstützt OAuth und API-Schlüssel für Modell-Provider. Für einen dauerhaft aktiven Gateway-Host ist ein API-Schlüssel die berechenbarste Option; Abonnement-/OAuth-Abläufe funktionieren ebenfalls, wenn sie zum Kontomodell Ihres Providers passen.

- Vollständiger OAuth-Ablauf und Speicherstruktur: [/concepts/oauth](/de/concepts/oauth)
- SecretRef-basierte Authentifizierung (`env`/`file`/`exec`-Provider): [Secret-Verwaltung](/de/gateway/secrets)
- Von `models status --probe` verwendete Eignungs-/Ursachencodes für Anmeldedaten: [Semantik von Authentifizierungs-Anmeldedaten](/de/auth-credential-semantics)

## Empfohlene Einrichtung: API-Schlüssel (beliebiger Provider)

1. Erstellen Sie in der Konsole Ihres Providers einen API-Schlüssel.
2. Hinterlegen Sie ihn auf dem **Gateway-Host** (dem Computer, auf dem `openclaw gateway` ausgeführt wird):

```bash
export <PROVIDER>_API_KEY="..."
openclaw models status
```

3. Wenn das Gateway unter systemd/launchd ausgeführt wird, hinterlegen Sie den Schlüssel in `~/.openclaw/.env`, damit der Daemon ihn lesen kann:

```bash
cat >> ~/.openclaw/.env <<'EOF'
<PROVIDER>_API_KEY=...
EOF
```

4. Starten Sie den Gateway-Prozess (oder den Daemon) neu und prüfen Sie anschließend erneut:

```bash
openclaw models status
openclaw doctor
```

`openclaw onboard` kann API-Schlüssel auch zur Verwendung durch den Daemon speichern, wenn Sie Umgebungsvariablen nicht selbst verwalten möchten. Die vollständige Rangfolge beim Laden von Umgebungsvariablen (`env.shellEnv`, `~/.openclaw/.env`, systemd/launchd) finden Sie unter [Umgebungsvariablen](/de/help/environment).

## Anthropic: Wiederverwendung der Claude CLI

Die Authentifizierung per Anthropic-Setup-Token wird weiterhin unterstützt. Die Wiederverwendung der Claude CLI (Verwendung nach Art von `claude -p`) ist für diese Integration ebenfalls zugelassen; wenn auf dem Host eine Claude-CLI-Anmeldung verfügbar ist, ist dies der bevorzugte Weg für die lokale/Desktop-Nutzung. Für langlebige Gateway-Hosts bleibt ein Anthropic-API-Schlüssel die berechenbarste Wahl und bietet eine explizite serverseitige Abrechnungskontrolle.

Host-Einrichtung zur Wiederverwendung der Claude CLI:

```bash
# Auf dem Gateway-Host ausführen
claude auth login
claude auth status --text
openclaw models auth login --provider anthropic --method cli --set-default
```

Dies erfolgt in zwei Schritten: Melden Sie Claude Code auf dem Host bei Anthropic an und weisen Sie OpenClaw anschließend an, die Anthropic-Modellauswahl über das lokale `claude-cli`-Backend zu leiten und das passende OpenClaw-Authentifizierungsprofil zu speichern.

Der Gateway-Dienst muss `claude` über `PATH` auflösen können. Wenn eine Bereitstellung einen
nicht standardmäßigen Pfad zur ausführbaren Datei benötigt, registrieren Sie einen Wrapper über ein
[CLI-Backend-Plugin](/de/plugins/cli-backend-plugins).

## Manuelle Token-Eingabe

Funktioniert mit jedem Provider; schreibt in den agentenspezifischen SQLite-Authentifizierungsspeicher und aktualisiert die Konfiguration:

```bash
openclaw models auth paste-token --provider openrouter
```

OpenClaw liest Authentifizierungsprofile aus `openclaw-agent.sqlite` des jeweiligen Agenten. Endpunktdetails (`baseUrl`, `api`, Modell-IDs, Header, Zeitüberschreitungen) gehören unter `models.providers.<id>` in `openclaw.json` oder `models.json`, nicht in Authentifizierungsprofile.

Wenn eine ältere Installation noch `auth-profiles.json`, `auth-state.json` oder eine flache Struktur wie `{ "openrouter": { "apiKey": "..." } }` enthält, führen Sie `openclaw doctor --fix` aus, um sie in SQLite zu importieren; Doctor legt neben den ursprünglichen JSON-Dateien Sicherungen mit Zeitstempel ab.

Externe Authentifizierungsrouten wie Bedrock `auth: "aws-sdk"` sind keine Anmeldedaten. Legen Sie für eine benannte Bedrock-Route `auth.profiles.<id>.mode: "aws-sdk"` in `openclaw.json` fest – schreiben Sie `type: "aws-sdk"` nicht in den Speicher für Authentifizierungsprofile. `openclaw doctor --fix` migriert veraltete AWS-SDK-Markierungen aus dem Anmeldedatenspeicher in die Konfigurationsmetadaten.

### SecretRef-gestützte Anmeldedaten

- `api_key`-Anmeldedaten können `keyRef: { source, provider, id }` verwenden
- `token`-Anmeldedaten können `tokenRef: { source, provider, id }` verwenden
- Profile im OAuth-Modus lehnen SecretRef-Anmeldedaten ab: Wenn `auth.profiles.<id>.mode` den Wert `"oauth"` hat, wird ein SecretRef-gestütztes `keyRef`/`tokenRef` für dieses Profil abgelehnt.

## Status der Modellauthentifizierung prüfen

```bash
openclaw models status
openclaw doctor
```

Automatisierungsfreundliche Prüfung mit Rückgabecode `1` bei abgelaufenen/fehlenden und `2` bei bald ablaufenden Anmeldedaten:

```bash
openclaw models status --check
```

Live-Authentifizierungsprüfungen (fügen Sie `--probe-provider`, `--probe-profile`, `--probe-timeout`, `--probe-concurrency` oder `--probe-max-tokens` hinzu, um den Umfang einzugrenzen):

```bash
openclaw models status --probe
```

Hinweise:

- Prüfzeilen können aus Authentifizierungsprofilen, Umgebungs-Anmeldedaten oder `models.json` stammen.
- Wenn `auth.order.<provider>` ein gespeichertes Profil auslässt, meldet die Prüfung für dieses Profil `excluded_by_auth_order`, statt es auszuprobieren.
- Wenn eine Authentifizierung vorhanden ist, OpenClaw aber kein prüfbares Modell für diesen Provider auflösen kann, meldet die Prüfung `status: no_model`.
- Abklingzeiten für Ratenbegrenzungen können modellspezifisch sein: Ein Profil, das für ein Modell eine Abklingzeit durchläuft, kann weiterhin ein anderes Modell desselben Providers bedienen.

Optionale Betriebsskripte (systemd/Termux): [Skripte zur Authentifizierungsüberwachung](/de/help/scripts#auth-monitoring-scripts).

## Rotation von API-Schlüsseln (Gateway)

Einige Provider wiederholen eine Anfrage mit einem anderen konfigurierten Schlüssel, wenn ein Aufruf eine Ratenbegrenzung des Providers erreicht.

Prioritätsreihenfolge der Schlüssel je Provider:

1. `OPENCLAW_LIVE_<PROVIDER>_KEY` (einzelne Überschreibung, legt einen Schlüssel fest)
2. `<PROVIDER>_API_KEYS` (durch Kommas/Leerzeichen/Semikolons getrennte Liste)
3. `<PROVIDER>_API_KEY`
4. `<PROVIDER>_API_KEY_*` (jede Umgebungsvariable mit diesem Präfix)

Google-Provider (`google`, `google-vertex`) greifen zusätzlich auf `GOOGLE_API_KEY` zurück. Vor der Verwendung werden Duplikate aus der kombinierten Liste entfernt.

OpenClaw wechselt nur dann zum nächsten Schlüssel, wenn die Fehlermeldung mit einem der folgenden Muster übereinstimmt: `rate_limit`, `rate limit`, `429`, `quota exceeded`/`quota_exceeded`, `resource exhausted`/`resource_exhausted` oder `too many requests`. Bei anderen Fehlern erfolgt kein erneuter Versuch mit alternativen Schlüsseln. Wenn alle Schlüssel fehlschlagen, wird der endgültige Fehler des letzten Versuchs zurückgegeben.

<Note>
Providerspezifische Formulierungen wie `ThrottlingException`, `concurrency limit reached` oder `workers_ai ... quota limit exceeded` bestimmen die **Failover-/Wiederholungs-Klassifizierung** (Wechsel des Modells oder Providers bei wiederholtem Fehlschlag), einen von der oben beschriebenen API-Schlüsselrotation getrennten Mechanismus.
</Note>

Durch das Entfernen gespeicherter Authentifizierungsdaten wird der Schlüssel beim Provider nicht widerrufen – rotieren oder widerrufen Sie ihn im Dashboard des Providers, wenn Sie ihn auf Providerseite ungültig machen müssen.

## Provider-Authentifizierung bei laufendem Gateway entfernen

Wenn Sie die Provider-Authentifizierung über die Gateway-Steuerungsebene entfernen, löscht OpenClaw die gespeicherten Authentifizierungsprofile für diesen Provider und bricht aktive Chat-/Agentenläufe ab, deren ausgewählter Modell-Provider dem entfernten entspricht. Abgebrochene Läufe geben die üblichen Abbruch-/Lebenszyklusereignisse mit `stopReason: "auth-revoked"` aus, sodass verbundene Clients anzeigen können, dass der Lauf aufgrund entfernter Anmeldedaten beendet wurde.

## Verwendete Anmeldedaten steuern

### OpenAI und veraltete `openai-codex`-IDs

Sowohl OpenAI-API-Schlüsselprofile als auch ChatGPT/Codex-OAuth-Profile verwenden die kanonische Provider-ID `openai`. Verwenden Sie für neue Konfigurationen `openai:*`-Profil-IDs und `auth.order.openai`.

Wenn Sie `openai-codex` in einer älteren Konfiguration, in Authentifizierungsprofil-IDs oder in `auth.order.openai-codex` sehen, behandeln Sie es als veraltete Migrationseingabe – erstellen Sie keine neuen `openai-codex`-Profile. Führen Sie Folgendes aus:

```bash
openclaw doctor --fix
openclaw models auth list --provider openai
```

Doctor schreibt veraltete `openai-codex:*`-Profil-IDs und `auth.order.openai-codex`-Einträge auf die kanonische `openai`-Route um. Informationen zur OpenAI-spezifischen Modell-/Runtime-Weiterleitung finden Sie unter [OpenAI](/de/providers/openai).

### Während der Anmeldung (CLI)

```bash
openclaw models auth login --provider openai --profile-id openai:ritsuko
openclaw models auth login --provider openai --profile-id openai:lain
```

`--profile-id` hält mehrere OAuth-Anmeldungen für denselben Provider innerhalb eines Agenten getrennt.

`--force` löscht die gespeicherten Authentifizierungsprofile für diesen Provider im ausgewählten Agentenverzeichnis und führt anschließend denselben Authentifizierungsablauf erneut aus. Verwenden Sie dies, wenn ein gespeichertes Profil festhängt, abgelaufen oder mit dem falschen Konto verknüpft ist. Dadurch werden die Anmeldedaten beim Provider nicht widerrufen.

```bash
openclaw models auth login --provider anthropic --force
```

### Pro Sitzung (Chat-Befehl)

- `/model <alias-or-id>@<profileId>` legt bestimmte Provider-Anmeldedaten für die aktuelle Sitzung fest (Beispiel-Profil-IDs: `anthropic:default`, `anthropic:work`).
- `/model` (oder `/model list`) zeigt eine kompakte Auswahl; `/model status` zeigt die vollständige Ansicht (Kandidaten + nächstes Authentifizierungsprofil sowie konfigurierte Provider-Endpunktdetails).

Wenn Sie die Authentifizierungsreihenfolge oder Profilfestlegung für einen bereits laufenden Chat ändern, senden Sie `/new` oder `/reset`, um eine neue Sitzung zu starten – bestehende Sitzungen behalten ihre aktuelle Modell-/Profilauswahl bis zum Zurücksetzen bei.

### Pro Agent (CLI-Überschreibung)

Überschreibungen der Authentifizierungsreihenfolge werden im SQLite-Authentifizierungsstatus des jeweiligen Agenten gespeichert:

```bash
openclaw models auth order get --provider anthropic
openclaw models auth order set --provider anthropic anthropic:default
openclaw models auth order clear --provider anthropic
```

Verwenden Sie `--agent <id>`, um einen bestimmten Agenten anzugeben; lassen Sie die Option weg, um den konfigurierten Standardagenten zu verwenden. `openclaw models status --probe` zeigt ausgelassene gespeicherte Profile als `excluded_by_auth_order` an, statt sie stillschweigend zu überspringen.

## Fehlerbehebung

### „Keine Anmeldedaten gefunden“

Konfigurieren Sie einen Anthropic-API-Schlüssel auf dem **Gateway-Host** oder richten Sie den Anthropic-Setup-Token-Pfad ein und prüfen Sie anschließend erneut:

```bash
openclaw models status
```

### Token läuft bald ab/ist abgelaufen

Führen Sie `openclaw models status` aus, um zu sehen, welches Profil abläuft. Wenn ein Anthropic-Token-Profil fehlt oder abgelaufen ist, aktualisieren Sie es über den Setup-Token oder migrieren Sie zu einem Anthropic-API-Schlüssel.

## Verwandte Themen

- [Secret-Verwaltung](/de/gateway/secrets)
- [Remotezugriff](/de/gateway/remote)
- [Authentifizierungsspeicher](/de/concepts/oauth)
