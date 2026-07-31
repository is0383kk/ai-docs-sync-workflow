---
read_when:
    - Erneutes Auflösen von Secret-Referenzen zur Laufzeit
    - Prüfung auf Klartextreste und nicht aufgelöste Referenzen
    - SecretRefs konfigurieren und nicht umkehrbare Bereinigungsänderungen anwenden
summary: CLI-Referenz für `openclaw secrets` (neu laden, prüfen, konfigurieren, anwenden)
title: Geheimnisse
x-i18n:
    generated_at: "2026-07-26T18:53:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 61f6f81e358ca2e6a97ac9498186b32f7a74d16052d226c398dad0030d47211e
    source_path: cli/secrets.md
    workflow: 16
---

# `openclaw secrets`

Verwalten Sie SecretRefs und halten Sie den aktiven Runtime-Snapshot funktionsfähig.

| Befehl      | Funktion                                                                                                                                                                                                      |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `reload`    | Gateway-RPC (`secrets.reload`): löst Referenzen erneut auf und veröffentlicht den eigentümerbezogenen Runtime-Snapshot atomar (ohne Konfigurationsschreibvorgänge); Fehler geeigneter Eigentümer können als Warnungen mit dem Status „kalt“ oder „veraltet“ veröffentlicht werden |
| `audit`     | Schreibgeschützter Scan von Konfigurations-, Authentifizierungs- und generierten Modellspeichern sowie Legacy-Rückständen auf Klartext, nicht aufgelöste Referenzen und Prioritätsabweichungen (Exec-Referenzen werden übersprungen, sofern nicht `--allow-exec`) |
| `configure` | Interaktiver Planer für Provider-Einrichtung, Zielzuordnung und Vorabprüfung (erfordert ein TTY)                                                                                                                |
| `apply`     | Führt einen gespeicherten Plan aus (`--dry-run` validiert nur und überspringt Exec-Prüfungen standardmäßig; der Schreibmodus lehnt Pläne mit Exec-Inhalten ab, sofern nicht `--allow-exec`) und bereinigt anschließend die ausgewählten Klartextrückstände |

Empfohlener Ablauf für den Betrieb:

```bash
openclaw secrets audit --check
openclaw secrets configure
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json
openclaw secrets audit --check
openclaw secrets reload
```

Wenn Ihr Plan `exec`-SecretRefs/-Provider enthält, übergeben Sie `--allow-exec` sowohl beim Probelauf als auch bei den schreibenden `apply`-Befehlen.

Exitcodes für CI/Gates:

- `audit --check` gibt bei Funden `1` zurück.
- Nicht aufgelöste Referenzen geben `2` zurück (unabhängig von `--check`).

Siehe auch: [Secret-Verwaltung](/de/gateway/secrets) · [SecretRef-Anmeldedatenoberfläche](/de/reference/secretref-credential-surface) · [Sicherheit](/de/gateway/security)

## Runtime-Snapshot neu laden

```bash
openclaw secrets reload
openclaw secrets reload --json
openclaw secrets reload --url ws://127.0.0.1:18789 --token <token>
```

Verwendet die Gateway-RPC-Methode `secrets.reload`. Funktionsfähige Eigentümer werden unabhängig voneinander aktualisiert. Fehlgeschlagene geeignete Eigentümer werden nur dann als veraltet eingestuft, wenn ihre Referenzidentitäten, Provider-Definitionen und ihr vollständiger, nicht geheimer Eigentümervertrag unverändert sind; neue oder geänderte Fehler werden als kalt eingestuft. Diese eingeschränkte Aktivierung ist erfolgreich und meldet `warningCount`. Strikte oder nicht zugeordnete Fehler geben einen Fehler zurück und behalten den zuvor aktiven Snapshot bei.

Optionen: `--url <url>`, `--token <token>`, `--timeout <ms>`, `--json`.

## Audit

Durchsucht den OpenClaw-Status nach:

- Speicherung von Secrets im Klartext
- nicht aufgelösten Referenzen
- Prioritätsabweichungen (`auth-profiles.json`-Anmeldedaten, die `openclaw.json`-Referenzen überlagern)
- generierten `agents/*/agent/models.json`-Rückständen (Provider-`apiKey`-Werte und sensible Provider-Header)
- Legacy-Rückständen (Einträge im alten Authentifizierungsspeicher, OAuth-Erinnerungen)

Der `.env`-Scan deckt das effektive Statusverzeichnis und das Verzeichnis ab, das die aktive Konfiguration enthält. Wenn beide Pfade dieselbe Datei bezeichnen, wird sie nur einmal gescannt.

Die Erkennung sensibler Provider-Header basiert auf Namensheuristiken: Sie kennzeichnet Header, deren Name gängige Authentifizierungs-/Anmeldedatenfragmente enthält (`authorization`, `x-api-key`, `token`, `secret`, `password`, `credential`).

```bash
openclaw secrets audit
openclaw secrets audit --check
openclaw secrets audit --json
openclaw secrets audit --allow-exec
```

Berichtsstruktur:

- `status`: `clean | findings | unresolved`
- `resolution`: `refsChecked`, `skippedExecRefs`, `resolvabilityComplete`
- `summary`: `plaintextCount`, `unresolvedRefCount`, `shadowedRefCount`, `legacyResidueCount`
- Fundcodes: `PLAINTEXT_FOUND`, `REF_UNRESOLVED`, `REF_SHADOWED`, `LEGACY_RESIDUE`

## Konfigurieren (interaktive Hilfe)

Erstellen Sie Provider- und SecretRef-Änderungen interaktiv, führen Sie eine Vorabprüfung durch und wenden Sie sie optional an:

```bash
openclaw secrets configure
openclaw secrets configure --plan-out /tmp/openclaw-secrets-plan.json
openclaw secrets configure --apply --yes
openclaw secrets configure --providers-only
openclaw secrets configure --skip-provider-setup
openclaw secrets configure --agent ops
openclaw secrets configure --json
```

Ablauf: zuerst Provider-Einrichtung (`secrets.providers`-Aliasse hinzufügen/bearbeiten/entfernen), dann Zuordnung der Anmeldedaten (Felder auswählen, `{source, provider, id}`-Referenzen zuweisen), anschließend Vorabprüfung und optionale Anwendung.

Flags:

- `--providers-only`: Nur `secrets.providers` konfigurieren, Zuordnung der Anmeldedaten überspringen
- `--skip-provider-setup`: Provider-Einrichtung überspringen, Anmeldedaten vorhandenen Providern zuordnen
- `--agent <id>`: Ermittlung und Schreibvorgänge für `auth-profiles.json`-Ziele auf einen Agentenspeicher beschränken
- `--allow-exec`: Exec-SecretRef-Prüfungen während Vorabprüfung/Anwendung zulassen (kann Provider-Befehle ausführen)

`--providers-only` und `--skip-provider-setup` können nicht kombiniert werden.

Hinweise:

- Erfordert ein interaktives TTY.
- Verarbeitet Secret-haltige Felder in `openclaw.json` sowie `auth-profiles.json` für den ausgewählten Agentenbereich; kanonisch unterstützte Oberfläche: [SecretRef-Anmeldedatenoberfläche](/de/reference/secretref-credential-surface).
- Unterstützt das direkte Erstellen neuer `auth-profiles.json`-Zuordnungen im Auswahlablauf.
- Führt vor der Anwendung eine Vorabauflösung durch.
- Bei generierten Plänen sind die Bereinigungsoptionen standardmäßig aktiviert (`scrubEnv`, `scrubAuthProfilesForProviderTargets`, `scrubLegacyAuthJson`). Die Anwendung ist für bereinigte Klartextwerte unumkehrbar.
- `--plan-out` weigert sich, einen Plan zu erstellen, dessen UTF-8-serialisierte Form 16 MiB (16,777,216 Byte) überschreitet, entsprechend dem Eingabelimit von `apply --from`.
- Ohne `--apply` fragt die CLI nach der Vorabprüfung weiterhin nach `Apply this plan now?`.
- Mit `--apply` (und ohne `--yes`) fordert die CLI eine zusätzliche Bestätigung der unumkehrbaren Migration an.
- `--json` gibt den Plan und den Vorabprüfungsbericht aus, erfordert jedoch weiterhin ein interaktives TTY.

### Sicherheit von Exec-Providern

Homebrew-Installationen stellen häufig über symbolische Links eingebundene Binärdateien unter `/opt/homebrew/bin/*` bereit. Legen Sie `allowSymlinkCommand: true` nur bei Bedarf für vertrauenswürdige Paketmanagerpfade fest, zusammen mit `trustedDirs` (zum Beispiel `["/opt/homebrew"]`). Wenn unter Windows die ACL-Prüfung für einen Provider-Pfad nicht verfügbar ist, verweigert OpenClaw aus Sicherheitsgründen den Vorgang; legen Sie nur für vertrauenswürdige Pfade `allowInsecurePath: true` für diesen Provider fest, um die Pfadsicherheitsprüfung zu umgehen.

## Gespeicherten Plan anwenden

```bash
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --json
```

`--dry-run` validiert die Vorabprüfung, ohne Dateien zu schreiben; Exec-SecretRef-Prüfungen werden beim Probelauf standardmäßig übersprungen. Der Schreibmodus lehnt Pläne ab, die Exec-SecretRefs/-Provider enthalten, sofern nicht `--allow-exec`. Verwenden Sie `--allow-exec`, um Provider-Prüfungen/-Ausführungen über Exec in beiden Modi ausdrücklich zuzulassen.

`--from` muss auf eine reguläre Datei mit höchstens 16 MiB (16,777,216 Byte) verweisen. Das Bytelimit gilt für die gesamte serialisierte Datei einschließlich Leerraum.

Was `apply` aktualisieren kann:

- `openclaw.json` (SecretRef-Ziele sowie Hinzufügen/Aktualisieren/Löschen von Providern)
- `auth-profiles.json` (Bereinigung von Provider-Zielen)
- Legacy-`auth.json`-Rückstände
- `.env`-Dateien in den effektiven Status- und aktiven Konfigurationsverzeichnissen für bekannte Secret-Schlüssel, deren Werte migriert wurden

Details zum Planvertrag (zulässige Zielpfade, Validierungsregeln, Fehlersemantik): [Vertrag für den Plan zur Anwendung von Secrets](/de/gateway/secrets-plan-contract).

### Warum es keine Rollback-Sicherungen gibt

`secrets apply` schreibt absichtlich keine Rollback-Sicherungen mit alten Klartextwerten. Die Sicherheit ergibt sich aus einer strikten Vorabprüfung und einer weitgehend atomaren Anwendung, bei der im Fehlerfall nach bestem Bemühen eine Wiederherstellung im Arbeitsspeicher erfolgt.

## Beispiel

```bash
openclaw secrets audit --check
openclaw secrets configure
openclaw secrets audit --check
```

Wenn `audit --check` weiterhin Klartextfunde meldet, aktualisieren Sie die verbleibenden gemeldeten Zielpfade und führen Sie den Audit erneut aus.

## Siehe auch

- [CLI-Referenz](/de/cli)
- [Secret-Verwaltung](/de/gateway/secrets)
- [Vault-SecretRefs](/de/plugins/vault)
