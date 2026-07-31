---
read_when:
    - Sie möchten API-Schlüssel aus openclaw.json entfernen und in 1Password speichern
    - Sie führen das Gateway ohne Benutzeroberfläche aus und benötigen eine Dienstkontoauthentifizierung für op
    - Sie möchten, dass Agenten mit der op CLI Geheimnisse lesen oder einfügen.
summary: Lösen Sie Gateway-Secrets mit der 1Password CLI auf und ermöglichen Sie Agenten die Verwendung des mitgelieferten 1password-Skills
title: 1Password
x-i18n:
    generated_at: "2026-07-26T18:22:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5bb14944f0b3ce1ee3f90bf666a53e8673e7a9861e3e138a5fabe9c8e070cbd7
    source_path: gateway/1password.md
    workflow: 16
---

OpenClaw lässt sich auf drei unabhängige Arten mit **1Password** kombinieren:

- **Konfigurationsgeheimnisse:** Jedes [SecretRef](/de/gateway/secrets)-Feld in `openclaw.json` kann zur Laufzeit über die `op`-CLI aufgelöst werden, sodass API-Schlüssel nie in der Konfigurationsdatei gespeichert werden.
- **Agenten-Workflows:** Das mitgelieferte `1password`-Skill vermittelt Agenten, wie sie sich anmelden und mit `op` Geheimnisse für ihre eigenen Aufgaben lesen oder einfügen.
- **Browser-Anmeldung:** Das `claude-cli`-Backend kann die Chrome-Integration von Claude Code mit [1Password for Claude](https://support.1password.com/1password-claude/) verwenden. Dadurch kann sich der Agent bei Websites anmelden, ohne dass das Passwort jemals das Modell oder OpenClaw erreicht.

## Anforderungen

- Die [1Password CLI](https://developer.1password.com/docs/cli/get-started/) (`op`) muss auf dem Gateway-Host installiert sein (`brew install 1password-cli` unter macOS).
- Ein Authentifizierungsmodus für `op`:
  - **Dienstkonto** (für Headless-Gateways empfohlen): Exportieren Sie `OP_SERVICE_ACCOUNT_TOKEN` in der Dienstumgebung des Gateways. Keine Desktop-App und keine interaktive Anmeldung erforderlich.
  - **Desktop-App-Integration**: Die 1Password-App wird auf demselben Rechner ausgeführt und die CLI-Integration ist aktiviert. Die ersten Aufrufe können Touch ID oder eine Systemauthentifizierung auslösen.
  - **Eigenständige Anmeldung**: `op signin` fordert die Anmeldung pro Sitzung an. Für Agenten über das Skill praktikabel, aber nicht für die Auflösung von Konfigurationsgeheimnissen auf einem Headless-Gateway geeignet.

## Konfigurationsgeheimnisse mit op auflösen

Deklarieren Sie einen Exec-Provider für Geheimnisse, der `op read` mit einer `op://vault/item/field`-Referenz ausführt, und verweisen Sie anschließend mit einem beliebigen SecretRef-fähigen Feld darauf:

```json5
{
  secrets: {
    providers: {
      onepassword_openai: {
        source: "exec",
        command: "/opt/homebrew/bin/op",
        allowSymlinkCommand: true, // für über Homebrew installierte Binärdateien mit Symlink erforderlich
        trustedDirs: ["/opt/homebrew"],
        args: ["read", "op://Personal/OpenClaw QA API Key/password"],
        passEnv: ["HOME"],
        jsonOnly: false,
      },
    },
  },
  models: {
    providers: {
      openai: {
        baseUrl: "https://api.openai.com/v1",
        models: [{ id: "gpt-5", name: "gpt-5" }],
        apiKey: { source: "exec", provider: "onepassword_openai", id: "value" },
      },
    },
  },
}
```

So greifen die Komponenten ineinander:

- `command` muss ein absoluter Pfad sein; `trustedDirs` kennzeichnet das zugehörige Verzeichnis als vertrauenswürdig, und `allowSymlinkCommand` ist erforderlich, weil Homebrew `op` als Symlink installiert.
- `args` übernimmt die `op://vault/item/field`-Referenz unverändert. OpenClaw analysiert das `op://`-Schema nicht selbst; die Binärdatei `op` löst es auf.
- `passEnv` leitet die aufgeführten Variablen aus der Gateway-Umgebung weiter. Die Desktop-App-Integration benötigt `HOME`; für Dienstkonten muss außerdem `OP_SERVICE_ACCOUNT_TOKEN` in der Dienstumgebung des Gateways vorhanden sein (fügen Sie es `passEnv` hinzu oder legen Sie es nur dann über `env` fest, wenn Sie akzeptieren, dass das Token in der Konfigurationsdatei lesbar ist).
- Behalten Sie für eine Ausgabe mit einem einzelnen Wert `id: "value"` bei. Verwenden Sie bei `jsonOnly: true` und einer JSON-Nutzlast stattdessen eine JSON-Pointer-ID, um Felder zu adressieren.
- Ein Provider-Eintrag pro Geheimnis sorgt dafür, dass Referenzen nachvollziehbar bleiben; benennen Sie Provider nach ihren Verbrauchern (`onepassword_openai`, `onepassword_telegram`).

Informationen zur Auflösungsreihenfolge, Zwischenspeicherung und zum Fehlerverhalten finden Sie unter [Gateway-Geheimnisse](/de/gateway/secrets). Eine Übersicht aller Felder, die SecretRefs akzeptieren, finden Sie unter [SecretRef-Anmeldedatenoberfläche](/de/reference/secretref-credential-surface).

## Dienstkonto für Headless-Gateways einrichten

1. Erstellen Sie in Ihrem 1Password-Konto ein Dienstkonto und gewähren Sie ihm ausschließlich Lesezugriff auf die Tresorelemente, die das Gateway benötigt.
2. Stellen Sie `OP_SERVICE_ACCOUNT_TOKEN` dem Gateway-Dienst bereit (launchd-plist, systemd-Unit oder Container-Umgebungsvariable).
3. Fügen Sie `"OP_SERVICE_ACCOUNT_TOKEN"` der `passEnv`-Liste des Providers hinzu.
4. Überprüfen Sie dies in der Umgebung des Gateway-Hosts: `op whoami` sollte das Dienstkonto ausgeben, ohne zu einer Eingabe aufzufordern.

Für Lesezugriffe durch Dienstkonten muss der Tresor in der `op://`-Referenz ausdrücklich benannt werden. Begrenzen Sie den Umfang des Kontos strikt; es handelt sich um eine Bearer-Anmeldedateninformation.

## Das Skill „1password“ für Agenten

OpenClaw enthält ein `1password`-Skill, das Agenten zu kompetenten Bedienern von `op` macht: Es erkennt den verfügbaren Authentifizierungsmodus (Dienstkonto, Desktop-App-Integration oder eigenständige Anmeldung), überprüft vor jedem Lesezugriff den Zugriff mit `op whoami` und bevorzugt `op run` / `op inject`, statt geheime Werte auf den Datenträger zu schreiben. Das Skill benötigt die Binärdatei `op` und bietet eine Installation über Homebrew an, falls diese fehlt.

Agenten verwenden es für ihre eigenen Workflows, beispielsweise um während einer Aufgabe ein Bereitstellungs-Token zu lesen oder Umgebungsvariablen in einen Befehl einzufügen. Es ist von der Auflösung von Konfigurationsgeheimnissen unabhängig; das Gateway löst SecretRefs auf, ohne dass ein Skill beteiligt ist.

## Browser-Anmeldung mit 1Password for Claude

[1Password for Claude](https://support.1password.com/1password-claude/) ermöglicht Claude, eine Anmeldung anzufordern, während die Browsererweiterung von 1Password die Anmeldedaten über einen verschlüsselten Kanal direkt auf der Seite einträgt. Das Geheimnis gelangt nie in den Modellkontext, das Transkript oder OpenClaw. Wenn OpenClaw das [`claude-cli`-Backend](/de/gateway/cli-backends#claude-cli-specifics) mit aktivierter Chrome-Integration von Claude Code ausführt, können Agentenaufgaben diesen Ablauf für Websites verwenden, die eine tatsächlich angemeldete Sitzung benötigen.

Zusätzlich zum Backend selbst ist Folgendes erforderlich:

- Ein macOS-Gateway-Host mit Chrome, der verbundenen Erweiterung [Claude in Chrome](https://code.claude.com/docs/en/chrome), der 1Password-Desktop-App und der 1Password-Browsererweiterung (beide ab Version 8.12.28).
- Claude Code muss bei einem direkten Anthropic-Tarif angemeldet sein (Pro, Max, Team oder Enterprise). Die Chrome-Integration ist nicht über Amazon Bedrock, Google Cloud oder andere Drittanbieter verfügbar.
- Die einmalige Verbindung mit 1Password auf der Anthropic-Seite: 1Password for Claude wird über die Claude-Desktop-App oder den in der [Anleitung von 1Password](https://support.1password.com/1password-claude/) beschriebenen Erweiterungsablauf eingerichtet und befindet sich derzeit in einer macOS-Betaphase. Bei 1Password Business muss ein Administrator zunächst unter Policies die Option "Allow AI agents to autofill for users" aktivieren; bei Anthropic-Team-/Enterprise-Tarifen ist die Integration ebenfalls zunächst deaktiviert, bis ein Owner sie aktiviert.
- Ein [CLI-Backend-Plugin](/de/plugins/cli-backend-plugins), das `--chrome` zu den Claude-Startargumenten hinzufügt; das mitgelieferte Backend aktiviert Chrome nicht.
- Eine Person am Gateway-Host: Bei jeder Verwendung von Anmeldedaten wird dort eine 1Password-Abfrage angezeigt, die bestätigt werden muss (beispielsweise mit Touch ID). Bei einer restriktiven Exec-Richtlinie werden auch die Browser-Tool-Aufrufe selbst zunächst als OpenClaw-Genehmigungen an Ihren Kanal weitergeleitet.

Bevor Sie dies mit OpenClaw verbinden, überprüfen Sie die Komponenten in einer interaktiven Sitzung auf dem Gateway-Host: Führen Sie `claude --chrome` aus, bestätigen Sie, dass die Erweiterung eine Verbindung herstellt, und prüfen Sie, ob die `claude-in-chrome`-Tools die Anmeldedaten-Tools enthalten. Wenn sie dort nicht angezeigt werden, werden sie auch über OpenClaw nicht angezeigt.

Einmalpasswörter werden von 1Password auf derselben Seite eingetragen; leiten Sie Verifizierungscodes oder Passwörter niemals über den Chat weiter. Headless- oder Remote-Gateways können diesen Ablauf derzeit nicht verwenden, da sowohl die Genehmigung als auch der Browser auf dem Gateway-Host ausgeführt werden.

## Sicherheitshinweise

- Über Exec-Provider aufgelöste geheime Werte verbleiben im Arbeitsspeicher des Gateways; Konfigurations-Snapshots und `config.get`-Antworten schwärzen SecretRef-Felder.
- Speichern Sie geheime Werte niemals in `openclaw.json`, Protokollen oder Chats. Bewahren Sie Elementnamen in der Konfiguration und Werte in 1Password auf.
- Der Audit-Trail von 1Password zeigt jeden Lesezugriff eines Dienstkontos an, wodurch Schlüsselrotation und Vorfallprüfungen praktikabel werden.

## Fehlerbehebung

- `command not found`- oder Spawn-Fehler: Verwenden Sie den absoluten Pfad zu `op` und nehmen Sie dessen Verzeichnis in `trustedDirs` auf.
- `op` wird aufgelöst, aber Lesezugriffe schlagen mit Symlink-Fehlern fehl: Legen Sie für Homebrew-Installationen `allowSymlinkCommand: true` fest.
- `account is not signed in`: Stellen Sie bei Dienstkonten sicher, dass `OP_SERVICE_ACCOUNT_TOKEN` den Gateway-Dienst erreicht und in `passEnv` aufgeführt ist; stellen Sie bei der Desktop-Integration sicher, dass die App ausgeführt wird und entsperrt ist.
- Langsame erste Lesezugriffe: Erhöhen Sie `timeoutMs` für den Provider; Kaltstarts von `op` können auf ausgelasteten Hosts strikte Zeitüberschreitungen überschreiten.
