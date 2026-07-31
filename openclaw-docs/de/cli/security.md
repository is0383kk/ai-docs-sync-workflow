---
read_when:
    - Sie möchten eine schnelle Sicherheitsprüfung der Konfiguration und des Zustands durchführen
    - Sie möchten sichere „Fix“-Vorschläge anwenden (Berechtigungen, Standardeinstellungen verschärfen)
summary: CLI-Referenz für `openclaw security` (häufige Sicherheitsfallen prüfen und beheben)
title: Sicherheit
x-i18n:
    generated_at: "2026-07-26T17:46:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6b5f9ea5cb746bfd29ff4d096062e81595abe99a883fc3b1113b45a3527d42d9
    source_path: cli/security.md
    workflow: 16
---

# `openclaw security`

Sicherheitstools: Audit plus optionale sichere Korrekturen. Siehe auch: [Sicherheit](/de/gateway/security).

```bash
openclaw security audit
openclaw security audit --deep
openclaw security audit --deep --password <password>
openclaw security audit --deep --token <token>
openclaw security audit --auth password --password <password>
openclaw security audit --fix
openclaw security audit --json
```

## Audit-Modi

Der einfache `security audit` verbleibt auf dem kalten, schreibgeschützten Konfigurations-/Dateisystempfad: Er ermittelt keine Sicherheits-Collectors der Plugin-Laufzeit, sodass routinemäßige Audits nicht jede installierte Plugin-Laufzeit laden. `--deep` ergänzt bestmögliche Live-Abfragen des Gateways und Plugin-eigene Sicherheits-Audit-Collectors (explizite interne Aufrufer können diese Collectors ebenfalls aktivieren, wenn ihnen bereits ein geeigneter Laufzeitbereich zur Verfügung steht).

Wenn die Gateway-Passwortauthentifizierung nur beim Start bereitgestellt wird, übergeben Sie denselben Wert mit `--auth password --password <password>`, damit das Audit ihn mit `hooks.token` abgleichen kann.

## Geprüfte Bereiche

**DM-/Vertrauensmodell**

- Warnt, wenn mehrere DM-Absender dieselbe Hauptsitzung verwenden, und empfiehlt den sicheren DM-Modus: `session.dmScope="per-channel-peer"` (oder `per-account-channel-peer` für Kanäle mit mehreren Konten) für gemeinsam genutzte Posteingänge. Dies dient der Absicherung kooperativer/gemeinsam genutzter Posteingänge und nicht der Isolation gegenseitig nicht vertrauenswürdiger Betreiber; trennen Sie solche Vertrauensgrenzen durch separate Gateways (oder separate Betriebssystembenutzer/Hosts).
- Gibt `security.trust_model.multi_user_heuristic` aus, wenn die Konfiguration auf einen wahrscheinlich gemeinsam genutzten Benutzerzugang hindeutet (beispielsweise offene DM-/Gruppenrichtlinien, konfigurierte Gruppenziele oder Platzhalterregeln für Absender) – OpenClaws standardmäßiges Vertrauensmodell ist ein persönlicher Assistent (ein Betreiber), keine feindselige Mandantenisolation. Bei beabsichtigten Konfigurationen mit mehreren Benutzern: Führen Sie alle Sitzungen in einer Sandbox aus, beschränken Sie den Dateisystemzugriff auf den Arbeitsbereich und halten Sie persönliche/private Identitäten oder Anmeldedaten von dieser Laufzeit fern.
- Warnt, wenn kleine Modelle (`<=300B` Parameter) ohne Sandbox und mit aktivierten Web-/Browser-Tools verwendet werden.

**Webhook/Hooks**

Beim Start wird eine nicht schwerwiegende Sicherheitswarnung protokolliert, und das Audit meldet die `hooks.token`-Wiederverwendung aktiver Shared-Secret-Authentifizierungswerte des Gateways (`gateway.auth.token` / `OPENCLAW_GATEWAY_TOKEN`, `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD`). Es warnt außerdem, wenn:

- `hooks.token` kurz ist
- `hooks.path="/"`
- `hooks.defaultSessionKey` nicht festgelegt ist
- `hooks.allowedAgentIds` uneingeschränkt ist
- Anfrageüberschreibungen für `sessionKey` aktiviert sind
- Überschreibungen ohne `hooks.allowedSessionKeyPrefixes` aktiviert sind

Führen Sie `openclaw doctor --fix` aus, um einen persistent gespeicherten, wiederverwendeten `hooks.token` zu rotieren, und aktualisieren Sie anschließend externe Hook-Absender, damit sie das neue Token verwenden.

**Sandbox/Tools**

- Warnt, wenn Docker-Einstellungen für die Sandbox konfiguriert sind, während der Sandbox-Modus deaktiviert ist.
- Warnt, wenn `gateway.nodes.commands.deny` unwirksame musterähnliche/unbekannte Einträge verwendet (der Abgleich erfolgt ausschließlich anhand des exakten Node-Befehlsnamens, nicht durch Filterung des Shell-Texts).
- Warnt, wenn `gateway.nodes.commands.allow` gefährliche Node-Befehle explizit aktiviert.
- Warnt, wenn das globale `tools.profile="minimal"` durch Agent-Toolprofile überschrieben wird.
- Warnt, wenn Schreib-/Bearbeitungs-Tools deaktiviert sind, `exec` aber weiterhin ohne eine einschränkende Dateisystemgrenze der Sandbox verfügbar ist.
- Warnt, wenn offene DMs oder Gruppen Laufzeit-/Dateisystem-Tools ohne Sandbox-/Arbeitsbereichsschutz zugänglich machen.
- Warnt, wenn installierte Plugin-Tools bei einer freizügigen Toolrichtlinie erreichbar sein könnten.

**Sandbox-Browser**

- Warnt, wenn der Sandbox-Browser das Docker-Netzwerk `bridge` ohne `sandbox.browser.cdpSourceRange` verwendet.
- Meldet gefährliche Docker-Netzwerkmodi der Sandbox, einschließlich des Beitritts zu `host`- und `container:*`-Namensräumen.
- Warnt, wenn bei vorhandenen Docker-Containern des Sandbox-Browsers Hash-Labels fehlen oder veraltet sind (beispielsweise Container vor der Migration, denen `openclaw.browserConfigEpoch` fehlt), und empfiehlt `openclaw sandbox recreate --browser --all`.

**Netzwerk/Ermittlung**

- Meldet `gateway.allowRealIpFallback=true` (Risiko gefälschter Header bei falsch konfigurierten Proxys).
- Meldet `discovery.mdns.mode="full"` (Metadatenoffenlegung über mDNS-TXT-Einträge).
- Warnt, wenn `gateway.auth.mode="none"` die HTTP-APIs des Gateways ohne Shared Secret erreichbar lässt (`/tools/invoke` sowie alle aktivierten `/v1/*`-Endpunkte).

**Plugins/Kanäle**

- Warnt, wenn npm-basierte Installationsdatensätze für Plugins/Hooks nicht auf eine Version festgelegt sind, Integritätsmetadaten fehlen oder sie von den aktuell installierten Paketversionen abweichen.
- Warnt, wenn Kanal-Zulassungslisten statt stabiler IDs veränderliche Namen/E-Mail-Adressen/Tags verwenden (gegebenenfalls für Discord-, Slack-, Google-Chat-, Microsoft-Teams-, Mattermost- und IRC-Bereiche).

Einstellungen mit dem Präfix `dangerous`/`dangerously` sind explizite Notfallüberschreibungen durch den Betreiber; ihre Aktivierung stellt für sich allein keinen Bericht über eine Sicherheitslücke dar. Das vollständige Verzeichnis gefährlicher Parameter finden Sie unter „Zusammenfassung unsicherer oder gefährlicher Flags“ in [Sicherheit](/de/gateway/security).

## Verhalten von SecretRef

`security audit` löst unterstützte SecretRefs für die jeweiligen Zielpfade im schreibgeschützten Modus auf. Ist eine SecretRef im aktuellen Befehlspfad nicht verfügbar, wird das Audit fortgesetzt und meldet `secretDiagnostics`, statt abzustürzen. `--token` und `--password` überschreiben nur die Authentifizierung der tiefgehenden Abfrage für diesen Befehlsaufruf; sie schreiben weder die Konfiguration noch SecretRef-Zuordnungen neu.

## Unterdrückungen

Akzeptieren Sie beabsichtigte dauerhafte Befunde mit `security.audit.suppressions`. Jede Unterdrückung gleicht eine exakte `checkId` ab und kann durch Teilzeichenfolgen in `titleIncludes` und/oder `detailIncludes` eingegrenzt werden, wobei die Groß-/Kleinschreibung nicht berücksichtigt wird:

```json
{
  "security": {
    "audit": {
      "suppressions": [
        {
          "checkId": "plugins.tools_reachable_permissive_policy",
          "detailIncludes": "Enabled extension plugins: gbrain",
          "reason": "trusted local operator plugin"
        }
      ]
    }
  }
}
```

Unterdrückte Befunde werden aus der aktiven Liste `summary` und `findings` entfernt. Die JSON-Ausgabe behält sie zur Nachvollziehbarkeit des Audits unter `suppressedFindings` bei. Wenn Unterdrückungen konfiguriert sind, enthält die aktive Ausgabe außerdem einen nicht unterdrückbaren Informationsbefund `security.audit.suppressions.active`, damit erkennbar ist, dass das Audit gefiltert wurde. Gefährliche Konfigurations-Flags werden einzeln als Befunde ausgegeben, sodass die Akzeptanz eines gefährlichen Flags andere aktivierte Flags mit derselben `config.insecure_or_dangerous_flags`-checkId nicht ausblendet.

Da Unterdrückungen dauerhafte Risiken verbergen können, erfordert ihr Hinzufügen oder Entfernen über Shell-Befehle, die von einem Agenten ausgeführt werden, eine Ausführungsgenehmigung, sofern die Ausführung nicht bereits mit `security="full"` und `ask="off"` für vertrauenswürdige lokale Automatisierung erfolgt.

## JSON-Ausgabe

```bash
openclaw security audit --json | jq '.summary'
openclaw security audit --deep --json | jq '.findings[] | select(.severity=="critical") | .checkId'
```

Mit `--fix --json` enthält die Ausgabe sowohl die Korrekturaktionen als auch den Abschlussbericht:

```bash
openclaw security audit --fix --json | jq '{fix: .fix.ok, summary: .report.summary}'
```

## Was `--fix` ändert

Wendet sichere, deterministische Abhilfemaßnahmen an:

- ändert gängige `groupPolicy="open"` in `groupPolicy="allowlist"` (einschließlich Kontovarianten in unterstützten Kanälen)
- wenn die WhatsApp-Gruppenrichtlinie auf `allowlist` geändert wird, wird `groupAllowFrom` aus der gespeicherten Datei `allowFrom` vorbelegt, sofern diese Liste vorhanden ist und die Konfiguration `allowFrom` noch nicht definiert
- ändert `logging.redactSensitive` von `"off"` in `"tools"`
- verschärft die Berechtigungen für Status-/Konfigurationsdateien und gängige vertrauliche Dateien (`credentials/*.json`, `auth-profiles.json`, `openclaw-agent.sqlite` sowie veraltete Sitzungsartefakte)
- verschärft außerdem die Berechtigungen für Konfigurations-Include-Dateien, auf die von `openclaw.json` verwiesen wird
- verwendet `chmod` auf POSIX-Hosts und `icacls`-Zurücksetzungen unter Windows

`--fix` führt Folgendes **nicht** aus:

- Token/Passwörter/API-Schlüssel rotieren
- Tools deaktivieren (`gateway`, `cron`, `exec` usw.)
- Gateway-Bindungs-, Authentifizierungs- oder Netzwerkfreigabeoptionen ändern
- Plugins/Skills entfernen oder neu schreiben

## Verwandte Themen

- [CLI-Referenz](/de/cli)
- [Sicherheitsaudit](/de/gateway/security)
