---
read_when:
    - SecretRefs für Provider-Anmeldedaten und `auth-profiles.json`-Referenzen konfigurieren
    - Neuladen, Prüfen, Konfigurieren und Anwenden von Secrets im Produktionsbetrieb auf sichere Weise
    - Grundlegendes zu Fail-Fast beim Start, zur Filterung inaktiver Oberflächen und zum Verhalten mit der letzten als funktionsfähig bekannten Konfiguration
sidebarTitle: Secrets management
summary: 'Secret-Verwaltung: SecretRef-Vertrag, Verhalten von Runtime-Snapshots und sichere unumkehrbare Bereinigung'
title: Geheimnisverwaltung
x-i18n:
    generated_at: "2026-07-26T18:28:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d10989ebbce367c68d28768244d4e3649028af5ab63c9523974352c270a3c55e
    source_path: gateway/secrets.md
    workflow: 16
---

OpenClaw unterstützt additive SecretRefs, sodass unterstützte Anmeldedaten nicht im Klartext in der Konfiguration gespeichert werden müssen.

<Note>
Klartext funktioniert weiterhin. SecretRefs müssen für jede einzelne Anmeldeinformation aktiviert werden.
</Note>

<Warning>
Klartext-Anmeldedaten bleiben für den Agenten lesbar, wenn sie sich in Dateien befinden, die der Agent prüfen kann, einschließlich `openclaw.json`, `auth-profiles.json`, `.env` oder generierter `agents/*/agent/models.json`-Dateien. SecretRefs verringern diesen lokalen Schadensradius erst, wenn alle unterstützten Anmeldedaten migriert wurden und `openclaw secrets audit --check` keine Klartextreste meldet.
</Warning>

## Laufzeitmodell

- Secrets werden während der Aktivierung vorab in einen In-Memory-Laufzeit-Snapshot aufgelöst, nicht verzögert in Anfragepfaden.
- Beim Kaltstart des Gateways wird ein wiederholbarer SecretRef-Fehler auf einen bekannten Eigentümer außerhalb des Gateways beschränkt, sofern dieser Eigentümer Isolation unterstützt. Zu den zugeordneten Eigentümerklassen gehören Modell-Provider und Skills, Medien-/TTS-/Cron-Provider, geeignete Authentifizierungsprofile, agentenspezifischer Speicher, Sandbox-SSH, Kanalkonten und im Manifest deklarierte Plugin-Routen. Das Gateway startet, kennzeichnet den Eigentümer als konfiguriert, aber nicht verfügbar, und gibt eine bereinigte Warnung zur eingeschränkten Funktion aus. Gateway-Eingangsauthentifizierung, strukturell ungültige Referenzen oder aufgelöste Werte, ausfallsicher geschlossene Eigentümer und Referenzen, deren Laufzeiteigentümer nicht zugeordnet ist, verhindern weiterhin den Start.
- Beim Neuladen wird jeder zugeordnete Eigentümer unabhängig validiert und anschließend ein einzelner atomarer Snapshot veröffentlicht. Funktionierende Eigentümer werden aktualisiert. Ein geeigneter fehlgeschlagener Eigentümer behält seinen letzten als funktionierend bekannten Wert und wird nur dann veraltet, wenn seine Referenzidentitäten, Providerdefinitionen und sein vollständiger, nicht geheimer Eigentümervertrag unverändert sind; ein geänderter oder neuer fehlgeschlagener Eigentümer wird kalt. Ein strikter Fehler lehnt das Neuladen ab und bewahrt den aktiven Snapshot.
- Richtlinienverstöße (beispielsweise ein Authentifizierungsprofil im OAuth-Modus in Kombination mit einer SecretRef-Eingabe) lassen die Aktivierung vor dem Austausch der Laufzeit fehlschlagen.
- Laufzeitanfragen lesen ausschließlich den aktiven In-Memory-Snapshot. SecretRef-Anmeldedaten von Modell-Providern werden bis zur ausgehenden Übertragung als prozesslokale Sentinel-Werte durch den Authentifizierungsspeicher und die Stream-Optionen weitergegeben. Ausgehende Zustellungspfade (Discord-Antwort-/Thread-Zustellung, Telegram-Aktionssendungen) lesen ebenfalls diesen Snapshot und lösen Referenzen nicht bei jedem Sendevorgang erneut auf.

Dadurch wirken sich Ausfälle von Secret-Providern nicht auf häufig durchlaufene Anfragepfade aus.

Gateway-Eingangsschutz, strukturell ungültige Konfigurationen oder aufgelöste Werte, Richtlinienverstöße und unbekannte Eigentümerschaft werden weiterhin ausfallsicher geschlossen. Isolierte Eigentümer greifen niemals auf eine Anmeldedatenquelle mit niedrigerer Priorität zurück.

## Injektion beim ausgehenden Datenverkehr (Sentinel-Werte)

Für durch SecretRefs gesicherte Anmeldedaten von Modell-Providern erzeugt OpenClaw während der Auflösung der Modellauthentifizierung einen undurchsichtigen, prozesslokalen Sentinel-Wert. Authentifizierungsspeicher, Stream-Optionen, SDK-Konfiguration, Protokolle, Fehlerobjekte und die meisten Laufzeitprüfungen sehen daher einen Wert wie `oc-sent-v1-...` statt der Provider-Anmeldedaten. Der geschützte Modellabruf und die verwalteten Zustandsprüfungen lokaler Provider ersetzen bekannte Sentinel-Werte in URL- und Header-Werten unmittelbar bevor die jeweilige Anfrage den Prozess verlässt.

Unbekannte Werte im Sentinel-Format werden vor jeglicher Netzwerkaktivität ausfallsicher abgelehnt. OpenClaw verweigert das Senden der Anfrage, statt einen nicht aufgelösten Sentinel-Wert an einen Provider weiterzuleiten. Aufgelöste Secret-Werte werden als zusätzliche Schutzmaßnahme außerdem für die Protokollbereinigung bei exakter Wertübereinstimmung registriert.

Provider-Adapter verwenden den spätesten von ihrem SDK unterstützten Injektionspunkt:

- SDKs mit einer benutzerdefinierten Abrufoption erhalten den geschützten Abruf von OpenClaw, sodass das SDK den Sentinel-Wert beibehält.
- SDKs ohne benutzerdefinierte Abrufoption entpacken den Sentinel-Wert unmittelbar vor der Client-Erstellung. Plugin-eigene Provider-Streams und Agent-Harnesses entpacken ihn bei der letzten vom Kern kontrollierten Übergabe, da diese Transporte nicht den geschützten Abruf von OpenClaw verwenden.

Sentinel-Werte verringern die Klartextexposition entlang der Modellaufrufkette, stellen jedoch keine Prozessisolation dar. Der tatsächliche Wert ist weiterhin im Speicher desselben Prozesses vorhanden und erscheint an der abschließenden Adaptergrenze. Klartext-Anmeldedaten aus der Umgebung, die nicht über SecretRefs konfiguriert sind, bleiben Klartext und fallen nicht unter diesen Mechanismus.

Setzen Sie `OPENCLAW_SECRET_SENTINELS=off` (akzeptiert außerdem `0` oder `false`, ohne Beachtung der Groß-/Kleinschreibung), um die Erzeugung von Sentinel-Werten während der Reaktion auf Sicherheitsvorfälle oder bei der Kompatibilitätsfehlerbehebung zu deaktivieren. Der Notausschalter deaktiviert nicht die Registrierung der Bereinigung bei exakter Wertübereinstimmung.

## Agentenzugriffsgrenze

SecretRefs verhindern, dass Anmeldedaten in Konfigurationsdateien und generierten Modelldateien gespeichert werden, stellen jedoch keine Prozessisolationsgrenze dar. Auf dem Datenträger verbliebene Klartext-Anmeldedaten in einem für den Agenten lesbaren Pfad können weiterhin über Datei- oder Shell-Werkzeuge gelesen werden, wodurch die Bereinigung auf API-Ebene umgangen wird.

Bei Produktionsbereitstellungen, bei denen für den Agenten zugängliche Dateien berücksichtigt werden müssen, gilt die Migration erst als abgeschlossen, wenn alle folgenden Bedingungen erfüllt sind:

- Unterstützte Anmeldedaten verwenden SecretRefs statt Klartextwerten.
- Veraltete Klartextreste wurden aus `openclaw.json`, `auth-profiles.json`, `.env` und generierten `models.json`-Dateien entfernt.
- `openclaw secrets audit --check` ist nach der Migration frei von Befunden.
- Alle verbleibenden nicht unterstützten oder rotierenden Anmeldedaten werden durch Betriebssystemisolation, Containerisolation oder einen externen Anmeldedaten-Proxy geschützt.

Aus diesem Grund ist der Arbeitsablauf für Prüfung, Konfiguration und Anwendung ein Sicherheitsmigrations-Gate und nicht lediglich ein Hilfsmittel zur Vereinfachung.

<Warning>
SecretRefs machen beliebige lesbare Dateien nicht sicher. Sicherungen, kopierte Konfigurationen, alte generierte Modellkataloge und nicht unterstützte Klassen von Anmeldedaten bleiben Produktions-Secrets, bis sie gelöscht, aus der Vertrauensgrenze des Agenten verschoben oder separat isoliert wurden.
</Warning>

## Filterung aktiver Oberflächen

SecretRefs werden nur auf tatsächlich aktiven Oberflächen validiert:

- **Aktivierte Oberflächen**: Wiederholbare Fehler bei zugeordneten, isolierbaren Eigentümern führen zu einer kalten oder veralteten Einschränkung. Strikte, ausfallsicher geschlossene, für das Gateway erforderliche oder nicht zugeordnete Fehler blockieren den Start beziehungsweise das Neuladen.
- **Inaktive Oberflächen**: Nicht aufgelöste Referenzen blockieren weder den Start noch das Neuladen; sie geben eine nicht schwerwiegende `SECRETS_REF_IGNORED_INACTIVE_SURFACE`-Diagnose aus.

<Accordion title="Beispiele für inaktive Oberflächen">
- Deaktivierte Kanal-/Kontoeinträge.
- Übergeordnete Kanalanmeldedaten, die von keinem aktivierten Konto übernommen werden.
- Deaktivierte Werkzeug-/Funktionsoberflächen.
- Provider-spezifische Schlüssel für die Websuche, die nicht durch `tools.web.search.provider` ausgewählt wurden. Im automatischen Modus (kein Provider festgelegt) werden Schlüssel gemäß ihrer Priorität zur automatischen Erkennung herangezogen, bis einer erfolgreich aufgelöst wird; nach der Auswahl sind die Schlüssel der nicht ausgewählten Provider inaktiv.
- Sandbox-SSH-Authentifizierungsmaterial (`agents.defaults.sandbox.ssh.identityData`, `certificateData`, `knownHostsData` sowie agentenspezifische Überschreibungen) ist nur aktiv, wenn das wirksame Sandbox-Backend `ssh` ist und der Sandbox-Modus für den Standardagenten oder einen aktivierten Agenten nicht `off` lautet.
- `gateway.remote.token`- / `gateway.remote.password`-SecretRefs sind aktiv, wenn eine der folgenden Bedingungen erfüllt ist:
  - `gateway.mode=remote`
  - `gateway.remote.url` ist konfiguriert
  - `gateway.tailscale.mode` ist `serve` oder `funnel`
  - Im lokalen Modus ohne diese Remote-Oberflächen: `gateway.remote.token` ist aktiv, wenn die Token-Authentifizierung Vorrang erhalten kann und kein Umgebungs-/Authentifizierungstoken konfiguriert ist; `gateway.remote.password` ist nur aktiv, wenn die Passwortauthentifizierung Vorrang erhalten kann und kein Umgebungs-/Authentifizierungspasswort konfiguriert ist.
- Die SecretRef `gateway.auth.token` ist für die Auflösung der Startauthentifizierung inaktiv, wenn `OPENCLAW_GATEWAY_TOKEN` festgelegt ist, da die Token-Eingabe aus der Umgebung für diese Laufzeit Vorrang hat.

</Accordion>

## Diagnose der Gateway-Authentifizierungsoberfläche

Wenn für `gateway.auth.token`, `gateway.auth.password`, `gateway.remote.token` oder `gateway.remote.password` eine SecretRef festgelegt ist, protokolliert der Start beziehungsweise das Neuladen des Gateways den Oberflächenstatus unter dem Code `SECRETS_GATEWAY_AUTH_SURFACE`:

- `active`: Die SecretRef ist Teil der wirksamen Authentifizierungsoberfläche und muss aufgelöst werden.
- `inactive`: Eine andere Authentifizierungsoberfläche hat Vorrang oder die Remote-Authentifizierung ist deaktiviert beziehungsweise nicht aktiv.

Der Protokolleintrag enthält den von der Richtlinie für aktive Oberflächen verwendeten Grund.

## Vorabprüfung der Onboarding-Referenz

Beim interaktiven Onboarding wird nach Auswahl der SecretRef-Speicherung vor dem Speichern eine Vorabvalidierung ausgeführt:

- Umgebungsreferenzen: Validieren den Namen der Umgebungsvariable und bestätigen, dass während der Einrichtung ein nicht leerer Wert sichtbar ist.
- Provider-Referenzen (`file` oder `exec`): Validieren die Provider-Auswahl, lösen `id` auf und prüfen den Typ des aufgelösten Werts.
- Schnellstartablauf: Wenn `gateway.auth.token` bereits eine SecretRef ist, löst das Onboarding sie vor dem Start der Prüfung beziehungsweise des Dashboards (für `env`-, `file`- und `exec`-Referenzen) mit demselben sofort abbrechenden Gate auf.

Bei einem Validierungsfehler wird der Fehler angezeigt und Sie können den Vorgang erneut versuchen.

## SecretRef-Vertrag

Überall dieselbe Objektstruktur:

```json5
{ source: "env" | "file" | "exec", provider: "default", id: "..." }
```

<Tabs>
  <Tab title="env">
    ```json5
    { source: "env", provider: "default", id: "OPENAI_API_KEY" }
    ```

    Auf SecretInput-Feldern werden außerdem Kurzschreibweisen als Zeichenfolgen akzeptiert:

    ```json5
    "${OPENAI_API_KEY}"
    "$OPENAI_API_KEY"
    ```

    Validierung:

    - `provider` muss `^[a-z][a-z0-9_-]{0,63}$` entsprechen
    - `id` muss `^[A-Z][A-Z0-9_]{0,127}$` entsprechen

  </Tab>
  <Tab title="file">
    ```json5
    { source: "file", provider: "filemain", id: "/providers/openai/apiKey" }
    ```

    Validierung:

    - `provider` muss `^[a-z][a-z0-9_-]{0,63}$` entsprechen
    - `id` muss ein absoluter JSON-Zeiger (`/...`) oder für `singleValue`-Provider das Literal `value` sein
    - RFC-6901-Escaping in Segmenten: `~` wird zu `~0`, `/` wird zu `~1`

  </Tab>
  <Tab title="exec">
    ```json5
    { source: "exec", provider: "vault", id: "providers/openai/apiKey#value" }
    ```

    Validierung:

    - `provider` muss `^[a-z][a-z0-9_-]{0,63}$` entsprechen
    - `id` muss `^[A-Za-z0-9][A-Za-z0-9._:/#-]{0,255}$` entsprechen (unterstützt Selektoren wie `secret#json_key`)
    - `id` darf `.` oder `..` nicht als durch Schrägstriche begrenzte Pfadsegmente enthalten (beispielsweise wird `a/../b` abgelehnt)

  </Tab>
</Tabs>

## Provider-Konfiguration

Definieren Sie Provider unter `secrets.providers`:

```json5
{
  secrets: {
    providers: {
      default: { source: "env" },
      filemain: {
        source: "file",
        path: "~/.openclaw/secrets.json",
        mode: "json", // oder "singleValue"
      },
      vault: {
        source: "exec",
        command: "/usr/local/bin/openclaw-vault-resolver",
        args: ["--profile", "prod"],
        passEnv: ["PATH", "VAULT_ADDR"],
        jsonOnly: true,
      },
      "team-secrets": {
        source: "exec",
        pluginIntegration: {
          pluginId: "acme-secrets",
          integrationId: "secret-store",
        },
      },
    },
    defaults: {
      env: "default",
      file: "filemain",
      exec: "vault",
    },
  },
}
```

<Accordion title="Umgebungs-Provider">
- Optionale Positivliste exakter Namen über `allowlist`.
- Fehlende oder leere Umgebungswerte führen zum Fehlschlagen der Auflösung.

</Accordion>

<Accordion title="Datei-Provider">
- Liest die lokale Datei unter `path`.
- `mode: "json"` (Standard) erwartet ein JSON-Objekt als Nutzlast und löst `id` als JSON-Zeiger auf.
- `mode: "singleValue"` erwartet die Referenz-ID `"value"` und gibt den unverarbeiteten Dateiinhalt zurück (abschließender Zeilenumbruch entfernt).
- Der Pfad muss Eigentums- und Berechtigungsprüfungen bestehen; `timeoutMs` (Standardwert 5000) und `maxBytes` (Standardwert 1 MiB) begrenzen den Lesevorgang.
- Windows schließt ausfallsicher: Wenn die ACL-Prüfung für den Pfad nicht verfügbar ist, schlägt die Auflösung fehl. Legen Sie ausschließlich für vertrauenswürdige Pfade `allowInsecurePath: true` für diesen Provider fest, um die Prüfung zu umgehen.

</Accordion>

<Accordion title="Exec-Provider">
- Führt den konfigurierten absoluten Binärpfad direkt und ohne Shell aus.
- Standardmäßig muss `command` eine reguläre Datei und darf kein Symlink sein. Legen Sie `allowSymlinkCommand: true` fest, um Symlink-Befehlspfade (beispielsweise Homebrew-Shims) zuzulassen, und kombinieren Sie dies mit `trustedDirs` (beispielsweise `["/opt/homebrew"]`), sodass nur Paketmanagerpfade zulässig sind.
- Unterstützt `timeoutMs` (Standardwert 5000), `noOutputTimeoutMs` (Standardwert entspricht `timeoutMs`), `maxOutputBytes` (Standardwert 1 MiB), eine Positivliste für `env`/`passEnv` sowie `trustedDirs`.
- `jsonOnly` verwendet standardmäßig `true`. Bei `jsonOnly: false` und einer einzelnen angeforderten ID wird eine einfache Nicht-JSON-Standardausgabe als Wert dieser ID akzeptiert.
- Unter Windows wird bei Fehlern sicher abgebrochen: Wenn die ACL-Prüfung für den Befehlspfad nicht verfügbar ist, schlägt die Auflösung fehl. Legen Sie ausschließlich für vertrauenswürdige Pfade bei diesem Provider `allowInsecurePath: true` fest, um die Prüfung zu umgehen.
- Von Plugins verwaltete Exec-Provider können `pluginIntegration` anstelle kopierter `command`/`args` verwenden. OpenClaw löst die aktuellen Befehlsdetails beim Starten/Neuladen anhand des Manifests des installierten Plugins auf. Wenn das Plugin deaktiviert, entfernt oder nicht vertrauenswürdig ist oder die Integration nicht mehr deklariert, schlagen aktive SecretRefs dieses Providers sicher fehl.

Anfrage-Payload (Standardeingabe):

```json
{ "protocolVersion": 1, "provider": "vault", "ids": ["providers/openai/apiKey"] }
```

Antwort-Payload (Standardausgabe):

```jsonc
{ "protocolVersion": 1, "values": { "providers/openai/apiKey": "<openai-api-key>" } } // Pragma: Geheimnis auf Positivliste zulassen
```

Optionale Fehler pro ID:

```json
{
  "protocolVersion": 1,
  "values": {},
  "errors": { "providers/openai/apiKey": { "code": "NOT_FOUND" } }
}
```

`code` ist eine optionale maschinenlesbare Diagnose. OpenClaw zeigt die erkannten
Codes `NOT_FOUND` und `AMBIGUOUS_DUPLICATE_KEY` zusammen mit dem Provider und der Referenz-ID an. Andere
Codes und frei definierbare Felder wie `message` werden aus Gründen der Kompatibilität mit Protokoll v1 akzeptiert,
aber nicht angezeigt, da die Resolver-Ausgabe Zugangsdaten enthalten kann.

</Accordion>

## Dateibasierte API-Schlüssel

Fügen Sie keine `file:...`-Zeichenfolgen in den `env`-Block der Konfiguration ein. Dieser Block ist literal und kann nicht überschrieben werden, daher wird `file:...` dort nie aufgelöst.

Verwenden Sie stattdessen eine Datei-SecretRef in einem unterstützten Zugangsdatenfeld:

```json5
{
  secrets: {
    providers: {
      xai_key_file: {
        source: "file",
        path: "~/.openclaw/secrets/xai-api-key.txt",
        mode: "singleValue",
      },
    },
  },
  models: {
    providers: {
      xai: {
        apiKey: { source: "file", provider: "xai_key_file", id: "value" },
      },
    },
  },
}
```

Für `mode: "singleValue"` lautet die SecretRef `id` `"value"`. Verwenden Sie für `mode: "json"` einen absoluten JSON-Zeiger wie `"/providers/xai/apiKey"`.

Unter [SecretRef-Zugangsdatenoberfläche](/de/reference/secretref-credential-surface) finden Sie die Felder, die SecretRefs akzeptieren.

## Beispiele für Exec-Integrationen

Eine eigene 1Password-Anleitung zu Dienstkonten, dem mitgelieferten Agent-Skill und zur Fehlerbehebung finden Sie unter [1Password](/de/gateway/1password).

<AccordionGroup>
  <Accordion title="1Password-CLI">
    ```json5
    {
      secrets: {
        providers: {
          onepassword_openai: {
            source: "exec",
            command: "/opt/homebrew/bin/op",
            allowSymlinkCommand: true, // erforderlich für über Symlinks verknüpfte Homebrew-Binärdateien
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
  </Accordion>
  <Accordion title="Bitwarden Secrets Manager (`bws`)">
    Verwenden Sie einen Resolver-Wrapper, um SecretRef-IDs den Elementschlüsseln von Bitwarden Secrets Manager zuzuordnen. Das Repository enthält `scripts/secrets/openclaw-bws-resolver.mjs`. Installieren oder kopieren Sie ihn auf dem Host, auf dem der Gateway ausgeführt wird, an einen absoluten vertrauenswürdigen Pfad.

    Anforderungen:

    - Die Bitwarden Secrets Manager-CLI (`bws`) ist auf dem Gateway-Host installiert.
    - `BWS_ACCESS_TOKEN` ist für den Gateway-Dienst verfügbar.
    - `PATH` wird an den Resolver übergeben oder `BWS_BIN` wird auf den absoluten Pfad der Binärdatei `bws` festgelegt.
    - `BWS_SERVER_URL` ist bei Verwendung einer selbst gehosteten Bitwarden-Instanz in der Umgebung festgelegt.

    ```json5
    {
      secrets: {
        providers: {
          bws: {
            source: "exec",
            command: "/usr/local/bin/openclaw-bws-resolver.mjs",
            passEnv: ["BWS_ACCESS_TOKEN", "BWS_SERVER_URL", "PATH", "BWS_BIN"],
            jsonOnly: true,
          },
        },
      },
      models: {
        providers: {
          openai: {
            baseUrl: "https://api.openai.com/v1",
            models: [{ id: "gpt-5", name: "gpt-5" }],
            apiKey: {
              source: "exec",
              provider: "bws",
              id: "openclaw/providers/openai/apiKey",
            },
          },
        },
      },
    }
    ```

    Der Resolver verarbeitet angeforderte IDs gebündelt, führt `bws secret list` aus und gibt Werte für übereinstimmende geheime `key`-Felder zurück. Verwenden Sie Schlüssel, die den ID-Vertrag der Exec-SecretRef erfüllen, beispielsweise `openclaw/providers/openai/apiKey`. Schlüssel im Stil von Umgebungsvariablen mit Unterstrichen werden abgelehnt, bevor der Resolver ausgeführt wird. Wenn mehrere sichtbare Bitwarden-Geheimnisse denselben angeforderten Schlüssel besitzen, lässt der Resolver diese ID wegen Mehrdeutigkeit fehlschlagen, anstatt zu raten. Überprüfen Sie nach dem Aktualisieren der Konfiguration den Resolver-Pfad:

    ```bash
    openclaw secrets audit --allow-exec
    ```

  </Accordion>
  <Accordion title="HashiCorp Vault-CLI">
    ```json5
    {
      secrets: {
        providers: {
          vault_openai: {
            source: "exec",
            command: "/opt/homebrew/bin/vault",
            allowSymlinkCommand: true, // erforderlich für über Symlinks verknüpfte Homebrew-Binärdateien
            trustedDirs: ["/opt/homebrew"],
            args: ["kv", "get", "-field=OPENAI_API_KEY", "secret/openclaw"],
            passEnv: ["VAULT_ADDR", "VAULT_TOKEN"],
            jsonOnly: false,
          },
        },
      },
      models: {
        providers: {
          openai: {
            baseUrl: "https://api.openai.com/v1",
            models: [{ id: "gpt-5", name: "gpt-5" }],
            apiKey: { source: "exec", provider: "vault_openai", id: "value" },
          },
        },
      },
    }
    ```
  </Accordion>
  <Accordion title="password-store (`pass`)">
    Verwenden Sie einen kleinen Resolver-Wrapper, um SecretRef-IDs direkt `pass`-Einträgen zuzuordnen. Speichern Sie ihn als ausführbare Datei unter einem absoluten Pfad, der die Pfadprüfungen Ihres Exec-Providers besteht, beispielsweise `/usr/local/bin/openclaw-pass-resolver`. Der `#!/usr/bin/env node`-Shebang löst `node` aus dem `PATH` des Resolver-Prozesses auf. Nehmen Sie daher `PATH` in `passEnv` auf. Wenn `pass` in diesem `PATH` nicht enthalten ist, legen Sie `PASS_BIN` in der übergeordneten Umgebung fest und nehmen Sie ihn ebenfalls in `passEnv` auf:

    ```js
    #!/usr/bin/env node
    const { spawnSync } = require("node:child_process");

    let stdin = "";
    process.stdin.setEncoding("utf8");
    process.stdin.on("data", (chunk) => {
      stdin += chunk;
    });
    process.stdin.on("error", (err) => {
      process.stderr.write(`${err.message}\n`);
      process.exit(1);
    });
    process.stdin.on("end", () => {
      let request;
      try {
        request = JSON.parse(stdin || "{}");
      } catch (err) {
        process.stderr.write(`Anfrage konnte nicht analysiert werden: ${err.message}\n`);
        process.exit(1);
      }

      const passBin = process.env.PASS_BIN || "pass";
      const values = {};
      const errors = {};

      for (const id of request.ids ?? []) {
        const result = spawnSync(passBin, ["show", id], { encoding: "utf8" });
        if (result.status === 0) {
          values[id] = result.stdout.split(/\r?\n/, 1)[0] ?? "";
        } else {
          errors[id] = { message: (result.stderr || `pass wurde mit ${result.status} beendet`).trim() };
        }
      }

      process.stdout.write(JSON.stringify({ protocolVersion: 1, values, errors }));
    });
    ```

    Konfigurieren Sie anschließend den Exec-Provider und verweisen Sie mit `apiKey` auf den Pfad des `pass`-Eintrags:

    ```json5
    {
      secrets: {
        providers: {
          pass_store: {
            source: "exec",
            command: "/usr/local/bin/openclaw-pass-resolver",
            passEnv: ["PATH", "HOME", "GNUPGHOME", "GPG_TTY", "PASSWORD_STORE_DIR", "PASS_BIN"],
            jsonOnly: true,
          },
        },
      },
      models: {
        providers: {
          openai: {
            baseUrl: "https://api.openai.com/v1",
            models: [{ id: "gpt-5", name: "gpt-5" }],
            apiKey: {
              source: "exec",
              provider: "pass_store",
              id: "openclaw/providers/openai/apiKey",
            },
          },
        },
      },
    }
    ```

    Bewahren Sie das Geheimnis in der ersten Zeile des `pass`-Eintrags auf oder passen Sie den Wrapper so an, dass stattdessen die vollständige `pass show`-Ausgabe zurückgegeben wird. Überprüfen Sie nach dem Aktualisieren der Konfiguration sowohl das statische Audit als auch den Pfad des Exec-Resolvers:

    ```bash
    openclaw secrets audit --check
    openclaw secrets audit --allow-exec
    ```

  </Accordion>
  <Accordion title="sops">
    ```json5
    {
      secrets: {
        providers: {
          sops_openai: {
            source: "exec",
            command: "/opt/homebrew/bin/sops",
            allowSymlinkCommand: true, // erforderlich für über Symlinks verknüpfte Homebrew-Binärdateien
            trustedDirs: ["/opt/homebrew"],
            args: ["-d", "--extract", '["providers"]["openai"]["apiKey"]', "/path/to/secrets.enc.json"],
            passEnv: ["SOPS_AGE_KEY_FILE"],
            jsonOnly: false,
          },
        },
      },
      models: {
        providers: {
          openai: {
            baseUrl: "https://api.openai.com/v1",
            models: [{ id: "gpt-5", name: "gpt-5" }],
            apiKey: { source: "exec", provider: "sops_openai", id: "value" },
          },
        },
      },
    }
    ```
  </Accordion>
</AccordionGroup>

## Umgebungsvariablen des MCP-Servers

Über `plugins.entries.acpx.config.mcpServers` konfigurierte Umgebungsvariablen des MCP-Servers akzeptieren SecretInput, sodass API-Schlüssel und Token nicht im Klartext in der Konfiguration stehen:

```json5
{
  plugins: {
    entries: {
      acpx: {
        enabled: true,
        config: {
          mcpServers: {
            github: {
              command: "npx",
              args: ["-y", "@modelcontextprotocol/server-github"],
              env: {
                GITHUB_PERSONAL_ACCESS_TOKEN: {
                  source: "env",
                  provider: "default",
                  id: "MCP_GITHUB_PAT",
                },
              },
            },
          },
        },
      },
    },
  },
}
```

Klartext-Zeichenfolgenwerte funktionieren weiterhin. Umgebungsvariablen-Vorlagenreferenzen wie `${MCP_SERVER_API_KEY}` und SecretRef-Objekte werden während der Gateway-Aktivierung aufgelöst, bevor der MCP-Serverprozess gestartet wird. Wie bei anderen SecretRef-Oberflächen blockieren nicht aufgelöste Referenzen die Aktivierung nur, wenn das Plugin `acpx` tatsächlich aktiv ist.

## SSH-Authentifizierungsmaterial für die Sandbox

Das zentrale `ssh`-Sandbox-Backend unterstützt außerdem SecretRefs für SSH-Authentifizierungsmaterial:

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "ssh",
        ssh: {
          target: "user@gateway-host:22",
          identityData: { source: "env", provider: "default", id: "SSH_IDENTITY" },
          certificateData: { source: "env", provider: "default", id: "SSH_CERTIFICATE" },
          knownHostsData: { source: "env", provider: "default", id: "SSH_KNOWN_HOSTS" },
        },
      },
    },
  },
}
```

Laufzeitverhalten:

- OpenClaw löst diese Referenzen während der Sandbox-Aktivierung auf, nicht verzögert bei jedem SSH-Aufruf.
- Aufgelöste Werte werden mit restriktiven Dateiberechtigungen (`0o600`) in ein temporäres Verzeichnis geschrieben und in der generierten SSH-Konfiguration verwendet.
- Wenn das effektive Sandbox-Backend nicht `ssh` ist (oder der Sandbox-Modus `off` ist), bleiben diese Referenzen inaktiv und blockieren den Start nicht.

## Unterstützte Anmeldedatenoberfläche

Die kanonisch unterstützten und nicht unterstützten Anmeldedaten sind unter [SecretRef-Anmeldedatenoberfläche](/de/reference/secretref-credential-surface) aufgeführt.

<Note>
Zur Laufzeit ausgestellte oder rotierende Anmeldedaten und OAuth-Aktualisierungsmaterial sind absichtlich von der schreibgeschützten SecretRef-Auflösung ausgeschlossen.
</Note>

## Erforderliches Verhalten und Rangfolge

- Feld ohne Referenz: unverändert.
- Feld mit Referenz: während der Aktivierung auf aktiven Oberflächen erforderlich.
- Wenn sowohl Klartext als auch eine Referenz vorhanden sind, hat die Referenz auf unterstützten Rangfolgepfaden Vorrang.
- Der Schwärzungs-Sentinel `__OPENCLAW_REDACTED__` ist für die interne Konfigurationsschwärzung/-wiederherstellung reserviert und wird als wörtlich übermittelte Konfigurationsangabe abgelehnt.

Warn- und Auditsignale:

- `SECRETS_REF_OVERRIDES_PLAINTEXT` (Laufzeitwarnung)
- `REF_SHADOWED` (Auditfund, wenn `auth-profiles.json`-Anmeldedaten Vorrang vor `openclaw.json`-Referenzen haben)

Google Chat `serviceAccount` akzeptiert Inline-JSON oder eine SecretRef. Doctor verschiebt das außer Betrieb genommene benachbarte Feld `serviceAccountRef` in dieses kanonische Feld, wenn es nicht gesetzt ist.

## Aktivierungsauslöser

Die Geheimnisaktivierung wird ausgeführt bei:

- Start (Vorprüfung plus abschließende Aktivierung)
- Hot-Apply-Pfad beim Neuladen der Konfiguration
- Neustartprüfungspfad beim Neuladen der Konfiguration
- Manuellem Neuladen über `secrets.reload`
- Vorprüfung des Gateway-RPC zum Schreiben der Konfiguration (`config.set` / `config.apply` / `config.patch`), wobei SecretRefs aktiver Oberflächen innerhalb der übermittelten Konfigurationsnutzlast validiert werden, bevor Änderungen dauerhaft gespeichert werden

Aktivierungsvertrag:

- Bei Erfolg wird der Snapshot atomar ausgetauscht.
- Ein strikter Startfehler bricht den Start des Gateway ab.
- Während eines Kaltstarts kann ein wiederholbarer Auflösungsfehler für einen zugeordneten, isolierbaren Nicht-Gateway-Eigentümer den Snapshot veröffentlichen, wobei genau dieser Eigentümer als konfiguriert-nicht-verfügbar markiert ist. Anfragen an den Eigentümer schlagen mit `SECRET_SURFACE_UNAVAILABLE` fehl; Eigentümer von Modell-Providern greifen nach dem Fehlschlagen einer expliziten Referenz nicht auf Anmeldedaten aus der Umgebung oder aus Authentifizierungsprofilen zurück.
- Neuladen und Neustartprüfung isolieren geeignete zugeordnete Eigentümer. Unveränderte Referenzidentitäten mit unveränderten Provider-Definitionen und einem unveränderten vollständigen, nicht geheimen Eigentümervertrag behalten ihre exakten letzten als funktionsfähig bekannten Werte als veraltet bei; geänderte oder neu konfigurierte, nicht aufgelöste Referenzen werden nur für diesen Eigentümer kalt veröffentlicht. Ein strikter Fehler beim Neuladen bewahrt den zuvor aktiven Snapshot.
- `config.set`, `config.apply` und `config.patch` akzeptieren syntaktisch gültige, nicht aufgelöste Referenzen für isolierbare Eigentümer und geben einen geschwärzten `degradedSecretOwners`-Bericht zurück. Gateway-Eingangsauthentifizierung, strukturell ungültige Konfigurationen oder aufgelöste Werte, Richtlinienverstöße und unbekannte Eigentümer werden weiterhin vor einer Änderung auf dem Datenträger abgelehnt.
- Funktionsfähige benachbarte Eigentümer werden normal aufgelöst und veröffentlicht, selbst wenn ein anderer Eigentümer kalt oder veraltet ist.
- Die Bereitstellung eines expliziten kanalspezifischen Tokens pro Aufruf für einen ausgehenden Hilfs-/Tool-Aufruf löst keine SecretRef-Aktivierung aus; Aktivierungspunkte bleiben Start, Neuladen und explizites `secrets.reload`.

## Signale für beeinträchtigten und wiederhergestellten Zustand

Wenn die Aktivierung beim Neuladen nach einem funktionsfähigen Zustand fehlschlägt, wechselt OpenClaw in einen beeinträchtigten Geheimniszustand und gibt einmalige Systemereignisse und Protokollcodes aus:

- `SECRETS_RELOADER_DEGRADED`
- `SECRETS_RELOADER_RECOVERED`

Verhalten:

- Beeinträchtigt: Funktionsfähige Eigentümer werden aktualisiert, veraltete Eigentümer behalten den letzten als funktionsfähig bekannten Wert und kalte Eigentümer bleiben nicht verfügbar.
- Wiederhergestellt: Wird nach der nächsten erfolgreichen Aktivierung einmal ausgegeben.
- Wiederholte Fehler im bereits beeinträchtigten Zustand protokollieren Warnungen, geben das Ereignis jedoch nicht erneut aus.
- Ein strikter Startfehler gibt niemals ein Beeinträchtigungsereignis aus, da die Laufzeit nie aktiv wurde. Ein erfolgreicher Start mit kalten Eigentümern protokolliert die Beeinträchtigung des Eigentümers, gibt jedoch kein Ereignis des Neuladers aus.
- Referenzbezogene Start- und Neuladefehler geben für jeden betroffenen Eigentümer eine strukturierte `SECRETS_DEGRADED`-Warnung aus. Provider-bezogene Ausfälle geben stattdessen eine `SECRETS_PROVIDER_DEGRADED`-Warnung mit dem Provider und der vollständigen Liste betroffener Eigentümer aus, anstatt den Provider-Fehler für jeden Eigentümer zu wiederholen. Warnungen enthalten einen geschwärzten Grund, den Eigentümerzustand `cold` oder `stale` und den Wiederholungshinweis `openclaw secrets reload`. Sie enthalten niemals aufgelöste Werte oder SecretRef-IDs.
- `openclaw doctor` führt kalte und veraltete Eigentümer mit ihren betroffenen Konfigurationspfaden, dem geschwärzten Grund und Hinweisen zur Wiederholung auf.

## Auflösung in Befehlspfaden

Befehlspfade können über einen Gateway-Snapshot-RPC die unterstützte SecretRef-Auflösung aktivieren. Es gelten zwei allgemeine Verhaltensweisen:

<Tabs>
  <Tab title="Strikte Befehlspfade">
    Beispielsweise `openclaw memory`-Remote-Speicherpfade und `openclaw qr --remote`, wenn Remote-Referenzen auf gemeinsame Geheimnisse benötigt werden. Sie lesen aus dem aktiven Snapshot und schlagen sofort fehl, wenn eine erforderliche SecretRef nicht verfügbar ist.
  </Tab>
  <Tab title="Schreibgeschützte Befehlspfade">
    Beispielsweise `openclaw status`, `openclaw status --all`, `openclaw channels status`, `openclaw channels resolve`, `openclaw security audit` und schreibgeschützte Doctor-/Konfigurationsreparaturabläufe. Sie bevorzugen ebenfalls den aktiven Snapshot, wechseln jedoch in einen beeinträchtigten Zustand, statt abzubrechen, wenn eine gezielt benötigte SecretRef nicht verfügbar ist.

    Schreibgeschütztes Verhalten:

    - Wenn das Gateway ausgeführt wird, lesen diese Befehle zuerst aus dem aktiven Snapshot.
    - Wenn die Gateway-Auflösung unvollständig oder das Gateway nicht verfügbar ist, versuchen sie einen gezielten lokalen Rückgriff für diese Befehlsoberfläche.
    - Wenn eine gezielt benötigte SecretRef weiterhin nicht verfügbar ist, wird der Befehl mit beeinträchtigter schreibgeschützter Ausgabe und einer ausdrücklichen Diagnose fortgesetzt, dass die Referenz konfiguriert, aber in diesem Befehlspfad nicht verfügbar ist.
    - Dieses beeinträchtigte Verhalten gilt nur lokal für den Befehl; es schwächt weder den Laufzeitstart noch Neulade-, Sende- oder Authentifizierungspfade.

  </Tab>
</Tabs>

Weitere Hinweise:

- Die Snapshot-Aktualisierung nach einer Geheimnisrotation im Backend wird von `openclaw secrets reload` verarbeitet.
- Von diesen Befehlspfaden verwendete Gateway-RPC-Methode: `secrets.resolve`.

## Audit- und Konfigurationsablauf

Standardablauf für Betreiber:

<Steps>
  <Step title="Aktuellen Zustand auditieren">
    ```bash
    openclaw secrets audit --check
    ```
  </Step>
  <Step title="SecretRefs konfigurieren und anwenden">
    ```bash
    openclaw secrets configure --apply
    ```
  </Step>
  <Step title="Erneut auditieren">
    ```bash
    openclaw secrets audit --check
    ```
  </Step>
</Steps>

Betrachten Sie die Migration erst als abgeschlossen, wenn das erneute Audit keine Befunde mehr ergibt. Wenn das Audit weiterhin dauerhaft gespeicherte Klartextwerte meldet, besteht das Risiko eines Agentenzugriffs fort, selbst wenn Laufzeit-APIs geschwärzte Werte zurückgeben.

Wenn Sie während `configure` einen Plan speichern, statt ihn anzuwenden, wenden Sie diesen gespeicherten Plan vor dem erneuten Audit mit `openclaw secrets apply --from <plan-path>` an.

<AccordionGroup>
  <Accordion title="secrets audit">
    Zu den Befunden gehören:

    - Dauerhaft gespeicherte Klartextwerte (`openclaw.json`, `auth-profiles.json`, `.env` und generiertes `agents/*/agent/models.json`).
    - Verbliebene sensible Klartext-Provider-Header in generierten `models.json`-Einträgen.
    - Nicht aufgelöste Referenzen.
    - Überschattung durch Rangfolge (`auth-profiles.json` hat Vorrang vor `openclaw.json`-Referenzen).
    - Altlasten (`auth.json`, OAuth-Erinnerungen).

    Hinweis zu Exec: Standardmäßig überspringt das Audit die Auflösbarkeitsprüfungen für Exec-SecretRefs, um Nebeneffekte von Befehlen zu vermeiden. Verwenden Sie `openclaw secrets audit --allow-exec`, um Exec-Provider während des Audits auszuführen.

    Hinweis zu Header-Altlasten: Die Erkennung sensibler Provider-Header basiert auf Namensheuristiken (gängige Namen und Fragmente von Authentifizierungs-/Anmeldedaten-Headern wie `authorization`, `x-api-key`, `token`, `secret`, `password` und `credential`).

  </Accordion>
  <Accordion title="secrets configure">
    Interaktives Hilfsprogramm, das:

    - Zuerst `secrets.providers` konfiguriert (`env`/`file`/`exec`, hinzufügen/bearbeiten/entfernen).
    - Sie unterstützte geheimnistragende Felder in `openclaw.json` sowie `auth-profiles.json` für einen Agentenbereich auswählen lässt.
    - Direkt in der Zielauswahl eine neue `auth-profiles.json`-Zuordnung erstellen kann.
    - SecretRef-Details erfasst (`source`, `provider`, `id`).
    - Eine Vorabauflösung ausführt und die Änderungen sofort anwenden kann.

    Hinweis zu Exec: Die Vorprüfung überspringt Exec-SecretRef-Prüfungen, sofern `--allow-exec` nicht gesetzt ist. Wenn Sie direkt aus `configure --apply` anwenden und der Plan Exec-Referenzen/-Provider enthält, lassen Sie `--allow-exec` auch für den Anwendungsschritt gesetzt.

    Hilfreiche Modi:

    - `openclaw secrets configure --providers-only`
    - `openclaw secrets configure --skip-provider-setup`
    - `openclaw secrets configure --agent <id>`

    Standardwerte für die Anwendung von `configure`:

    - Übereinstimmende statische Anmeldedaten für die ausgewählten Provider aus `auth-profiles.json` entfernen.
    - Veraltete statische `api_key`-Einträge aus `auth.json` entfernen.
    - Übereinstimmende bekannte Geheimniszeilen aus den effektiven Zustandsdateien und den `.env`-Dateien der aktiven Konfiguration entfernen (dedupliziert, wenn beide Pfade übereinstimmen).

  </Accordion>
  <Accordion title="secrets apply">
    Einen gespeicherten Plan anwenden:

    ```bash
    openclaw secrets apply --from /tmp/openclaw-secrets-plan.json
    openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --allow-exec
    openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
    openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run --allow-exec
    ```

    Hinweis zu Exec: Der Probelauf überspringt Exec-Prüfungen, sofern `--allow-exec` nicht gesetzt ist; der Schreibmodus lehnt Pläne mit Exec-SecretRefs/-Providern ab, sofern `--allow-exec` nicht gesetzt ist.

    Einzelheiten zum strikten Ziel-/Pfadvertrag und die genauen Ablehnungsregeln finden Sie unter [Vertrag für den Plan zur Anwendung von Geheimnissen](/de/gateway/secrets-plan-contract).

  </Accordion>
</AccordionGroup>

## Einseitige Sicherheitsrichtlinie

<Warning>
OpenClaw schreibt absichtlich keine Rollback-Sicherungen, die historische Klartextwerte von Geheimnissen enthalten.
</Warning>

Sicherheitsmodell:

- Die Vorprüfung muss vor dem Schreibmodus erfolgreich sein.
- Die Laufzeitaktivierung wird vor dem Commit validiert.
- Die Anwendung aktualisiert Dateien durch atomaren Dateiaustausch und versucht bei einem Fehler nach bestem Bemühen eine Wiederherstellung.

## Hinweise zur Kompatibilität mit veralteter Authentifizierung

Bei statischen Anmeldedaten hängt die Laufzeit nicht mehr von einer veralteten Klartextspeicherung für die Authentifizierung ab.

- Die Quelle der Laufzeitanmeldedaten ist der aufgelöste speicherinterne Snapshot.
- Veraltete statische `api_key`-Einträge werden entfernt, wenn sie erkannt werden.
- OAuth-bezogenes Kompatibilitätsverhalten bleibt davon getrennt.

## Hinweis zur Web-Benutzeroberfläche

Einige SecretInput-Unions lassen sich im Rohdateneditormodus einfacher konfigurieren als im Formularmodus.

## Verwandte Themen

- [Authentifizierung](/de/gateway/authentication) - Authentifizierung einrichten
- [CLI: Secrets](/de/cli/secrets) - CLI-Befehle
- [Vault SecretRefs](/de/plugins/vault) - HashiCorp-Vault-Provider einrichten
- [Umgebungsvariablen](/de/help/environment) - Priorität von Umgebungsvariablen
- [SecretRef-Anmeldedatenoberfläche](/de/reference/secretref-credential-surface) - Anmeldedatenoberfläche
- [Vertrag für den Plan zur Anwendung von Secrets](/de/gateway/secrets-plan-contract) - Details zum Planvertrag
- [Sicherheit](/de/gateway/security) - Sicherheitskonzept
