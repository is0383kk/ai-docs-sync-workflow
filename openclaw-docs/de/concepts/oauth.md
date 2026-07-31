---
read_when:
    - Sie möchten OpenClaw OAuth durchgängig verstehen
    - Es treten Probleme mit der Token-Ungültigmachung oder Abmeldung auf
    - Sie möchten Claude-CLI- oder OAuth-Authentifizierungsabläufe
    - Sie möchten mehrere Konten oder Profil-Routing verwenden
summary: 'OAuth in OpenClaw: Tokenaustausch, Speicherung und Muster für mehrere Konten'
title: OAuth
x-i18n:
    generated_at: "2026-07-26T18:55:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3ef94af0601b7d57bb7e2d53c3d8231708b401251eca7dc1bb1e7e4fc09b46da
    source_path: concepts/oauth.md
    workflow: 16
---

OpenClaw unterstützt OAuth („Abonnementauthentifizierung“) für Provider, die dies anbieten,
insbesondere **OpenAI Codex (ChatGPT OAuth)** und die **Wiederverwendung der Anthropic Claude CLI**.
Für Anthropic gilt in der Praxis folgende Aufteilung:

- **Anthropic-API-Schlüssel**: normale Anthropic-API-Abrechnung.
- **Anthropic Claude CLI/Abonnementauthentifizierung innerhalb von OpenClaw**: Anthropic-Mitarbeitende
  haben uns mitgeteilt, dass diese Nutzung wieder zulässig ist. Daher betrachtet OpenClaw die Wiederverwendung der Claude CLI und
  die Nutzung von `claude -p` für diese Integration als genehmigt, sofern Anthropic
  keine neue Richtlinie veröffentlicht. Für Anthropic in Produktionsumgebungen bleibt die
  Authentifizierung per API-Schlüssel der sicherere empfohlene Weg.

OpenClaw speichert sowohl die OpenAI-Authentifizierung per API-Schlüssel als auch ChatGPT/Codex OAuth unter der
kanonischen Provider-ID `openai`. Ältere `openai-codex:*`-Profil-IDs und
`auth.order.openai-codex`-Einträge sind Legacy-Zustände, die durch
`openclaw doctor --fix` repariert werden; verwenden Sie für
neue Konfigurationen `openai:*`-Profil-IDs und `auth.order.openai`.

Diese Seite behandelt:

- wie der OAuth-**Token-Austausch** funktioniert (PKCE)
- wo Token **gespeichert** werden (und warum)
- wie **mehrere Konten** verwaltet werden (Profile und sitzungsspezifische Überschreibungen)

Provider-Plugins, die einen eigenen OAuth- oder API-Schlüssel-Ablauf bereitstellen, verwenden denselben
Einstiegspunkt:

```bash
openclaw models auth login --provider <id>
```

## Die Token-Senke (warum sie existiert)

OAuth-Provider stellen üblicherweise bei jeder Anmeldung/Aktualisierung ein neues Aktualisierungstoken aus.
Einige Provider machen das vorherige Aktualisierungstoken ungültig, wenn für denselben
Benutzer/dieselbe App ein neues ausgestellt wird. Praktisches Symptom: Sie melden sich über OpenClaw _und_
über Claude Code/Codex CLI an, und eines davon wird später zufällig abgemeldet.

Um dies zu reduzieren, behandelt OpenClaw den Speicher für Authentifizierungsprofile als **Token-Senke**:

- die Laufzeit liest Anmeldedaten pro Agent aus einer einzigen Quelle
- mehrere Profile können nebeneinander bestehen und deterministisch weitergeleitet werden
- die Wiederverwendung externer CLIs ist providerspezifisch: Sobald OpenClaw ein lokales OAuth-
  Profil für einen Provider verwaltet, ist das lokale Aktualisierungstoken kanonisch. Wird dieses lokale
  Aktualisierungstoken abgelehnt, meldet OpenClaw das Profil zur
  erneuten Authentifizierung, statt auf Token-Material der externen CLI zurückzugreifen.
  Das Bootstrap-Verfahren der Codex CLI ist noch stärker eingeschränkt: Es kann nur ein leeres
  Profil im Stil von `openai:default` initialisieren, bevor OpenClaw OAuth für diesen
  Provider verwaltet; danach bleiben von OpenClaw durchgeführte Aktualisierungen kanonisch
- Status- und Startpfade beschränken die Erkennung externer CLIs auf die bereits
  konfigurierte Provider-Menge, sodass bei einer Einrichtung mit nur einem Provider nicht der
  Anmeldespeicher einer nicht zugehörigen CLI geprüft wird

## Speicherung (wo Token abgelegt werden)

Geheimnisse werden pro Agent unter dem logischen Namen `auth-profiles.json` gespeichert (der
zugrunde liegende Speicher ist die SQLite-Datenbank des Agents; der JSON-Name wird aus
Kompatibilitätsgründen und für die Anzeige in Werkzeugen beibehalten):

- Authentifizierungsprofile (OAuth, API-Schlüssel und optionale Referenzen auf Wertebene):
  `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- Legacy-Kompatibilitätsdatei: `~/.openclaw/agents/<agentId>/agent/auth.json`
  (statische `api_key`-Einträge werden bei ihrer Erkennung bereinigt)

Legacy-Datei nur für den Import (weiterhin unterstützt, aber nicht der primäre Speicher):

- `~/.openclaw/credentials/oauth.json` (wird bei der ersten Verwendung in den Speicher für Authentifizierungsprofile importiert)

Alle vorstehenden Pfade berücksichtigen auch `$OPENCLAW_STATE_DIR` (Überschreibung des Zustandsverzeichnisses). Vollständige Referenz: [/gateway/configuration-reference#auth-storage](/de/gateway/configuration-reference#auth-storage)

Informationen zu statischen Geheimnisreferenzen und zum Aktivierungsverhalten von Laufzeit-Snapshots finden Sie unter [Geheimnisverwaltung](/de/gateway/secrets).

Wenn ein sekundärer Agent kein lokales Authentifizierungsprofil besitzt, verwendet OpenClaw eine durchgereichte
Vererbung aus dem Speicher des Standard-/Haupt-Agents; der Speicher des Haupt-Agents wird beim
Lesen nicht geklont. OAuth-Aktualisierungstoken sind besonders sensibel: Normale
Kopiervorgänge überspringen sie standardmäßig, da einige Provider Aktualisierungstoken nach der
Verwendung rotieren oder ungültig machen. Konfigurieren Sie für einen Agent eine separate OAuth-Anmeldung, wenn
er ein unabhängiges Konto benötigt.

## Wiederverwendung der Anthropic Claude CLI

OpenClaw unterstützt die Wiederverwendung der Anthropic Claude CLI und `claude -p` als genehmigten
Authentifizierungsweg. Wenn auf dem Host bereits eine lokale Claude-Anmeldung vorhanden ist,
kann diese beim Onboarding bzw. bei der Konfiguration direkt wiederverwendet werden. Das Anthropic-Einrichtungstoken bleibt
als unterstützter Weg zur Token-Authentifizierung verfügbar, OpenClaw bevorzugt jedoch die Wiederverwendung der Claude CLI,
wenn diese verfügbar ist.

<Warning>
In der öffentlichen Dokumentation von Anthropic zu Claude Code wird angegeben, dass die direkte Nutzung von Claude Code innerhalb der
Limits des Claude-Abonnements bleibt, und Anthropic-Mitarbeitende haben uns mitgeteilt, dass die Claude-
CLI-Nutzung im Stil von OpenClaw wieder zulässig ist. Daher betrachtet OpenClaw die Wiederverwendung der Claude CLI und
die Nutzung von `claude -p` für diese Integration als genehmigt, sofern Anthropic
keine neue Richtlinie veröffentlicht.

Die aktuelle Anthropic-Dokumentation zu Tarifen für die direkte Nutzung von Claude Code finden Sie unter [Claude Code
mit Ihrem Pro- oder Max-
Tarif verwenden](https://support.claude.com/en/articles/11145838-using-claude-code-with-your-pro-or-max-plan)
und [Claude Code mit Ihrem Team- oder Enterprise-
Tarif verwenden](https://support.anthropic.com/en/articles/11845131-using-claude-code-with-your-team-or-enterprise-plan/).

Weitere abonnementbasierte Optionen in OpenClaw finden Sie unter [OpenAI
Codex](/de/providers/openai), [Qwen Cloud Coding-
Tarif](/de/providers/qwen), [MiniMax Coding-Tarif](/de/providers/minimax)
und [Z.AI/GLM Coding-Tarif](/de/providers/zai).
</Warning>

## OAuth-Austausch (wie die Anmeldung funktioniert)

Die interaktiven Anmeldeabläufe von OpenClaw sind in `openclaw/plugin-sdk/llm.ts` implementiert und in die Assistenten/Befehle eingebunden.

### Anthropic-Einrichtungstoken

Ablauf:

1. erstellen Sie das Token, indem Sie `claude setup-token` auf einem beliebigen Rechner mit Claude Code ausführen, und starten Sie anschließend in OpenClaw das Anthropic-Einrichtungstoken oder das Einfügen eines Tokens
2. OpenClaw speichert die resultierenden Anthropic-Anmeldedaten in einem Authentifizierungsprofil
3. die Modellauswahl bleibt auf `anthropic/...`
4. bestehende Anthropic-Authentifizierungsprofile bleiben für die Rollback-/Reihenfolgesteuerung verfügbar

### OpenAI Codex (ChatGPT OAuth)

OpenAI Codex OAuth wird ausdrücklich für die Verwendung außerhalb der Codex CLI unterstützt, einschließlich OpenClaw-Workflows.

Der Anmeldebefehl verwendet die kanonische OpenAI-Provider-ID:

```bash
openclaw models auth login --provider openai
```

Verwenden Sie `--profile-id openai:<name>` für mehrere ChatGPT/Codex-OAuth-Konten in
einem Agent. Verwenden Sie `openai-codex:<name>` nicht für neue Profile. Doctor migriert
dieses ältere Präfix zu einer kollisionsfreien `openai:*`-Profil-ID; führen Sie
nach der Reparatur `openclaw models auth list --provider openai` aus, bevor Sie
Profil-IDs in `auth.order` oder `/model ...@<profileId>` kopieren.

Ablauf (PKCE):

1. einen PKCE-Verifizierer/eine PKCE-Challenge und einen zufälligen `state` generieren
2. `https://auth.openai.com/oauth/authorize?...` öffnen (Geltungsbereich
   `openid profile email offline_access`)
3. versuchen, den Callback unter `http://localhost:1455/auth/callback` zu erfassen (der
   Callback-Host ist standardmäßig `localhost` und akzeptiert nur Loopback-Hosts;
   überschreiben Sie ihn mit `OPENCLAW_OAUTH_CALLBACK_HOST`)
4. wenn Sie einen Code einfügen können, bevor der Callback eintrifft (oder wenn Sie
   remote/headless arbeiten und der Callback keine Bindung herstellen kann), fügen Sie stattdessen die Weiterleitungs-URL/den Code
   ein – das manuelle Einfügen konkurriert mit dem Browser-Callback, und der zuerst
   abgeschlossene Vorgang gewinnt
5. den Code unter `https://auth.openai.com/oauth/token` austauschen
6. `accountId` aus dem Zugriffstoken extrahieren und `{ access, refresh, expires, accountId }` speichern

Der Pfad im Assistenten lautet `openclaw onboard` → Authentifizierungsauswahl `openai`.

## Aktualisierung und Ablauf

Profile speichern einen `expires`-Zeitstempel. Zur Laufzeit:

- wenn `expires` in der Zukunft liegt, das gespeicherte Zugriffstoken verwenden
- wenn es abgelaufen ist, aktualisieren (unter einer Dateisperre) und die gespeicherten Anmeldedaten überschreiben
- wenn ein sekundärer Agent ein geerbtes OAuth-Profil des Haupt-Agents liest, wird die
  Aktualisierung in den Speicher des Haupt-Agents zurückgeschrieben, statt das Aktualisierungstoken
  in den Speicher des sekundären Agents zu kopieren
- extern verwaltete CLI-Anmeldedaten (Claude CLI, eingeschränktes Bootstrap-Verfahren der Codex CLI;
  siehe [Die Token-Senke](#the-token-sink-why-it-exists)) werden erneut gelesen, statt
  ein kopiertes Aktualisierungstoken zu verbrauchen. Wenn eine verwaltete Aktualisierung fehlschlägt, meldet OpenClaw
  das betroffene Profil zur erneuten Authentifizierung, statt
  Token-Material einer externen CLI zurückzugeben.

Der Aktualisierungsablauf erfolgt automatisch; normalerweise müssen Token nicht manuell verwaltet werden.

## Mehrere Konten (Profile) und Weiterleitung

Zwei Muster:

### 1) Bevorzugt: separate Agents

Wenn „persönlich“ und „Arbeit“ niemals miteinander interagieren sollen, verwenden Sie isolierte Agents (separate Sitzungen, Anmeldedaten und Arbeitsbereiche):

```bash
openclaw agents add work
openclaw agents add personal
```

Konfigurieren Sie anschließend die Authentifizierung pro Agent (Assistent) und leiten Sie Chats an den richtigen Agent weiter.

### 2) Fortgeschritten: mehrere Profile in einem Agent

Der Speicher für Authentifizierungsprofile unterstützt mehrere Profil-IDs für denselben Provider.
Wählen Sie aus, welche verwendet wird:

- global über die Konfigurationsreihenfolge (`auth.order`)
- pro Sitzung über `/model ...@<profileId>`

Beispiel (Sitzungsüberschreibung):

- `/model Opus@anthropic:work`

Vorhandene Profil-IDs auflisten mit:

```bash
openclaw models auth list --provider <id>
```

Zugehörige Dokumentation:

- [Modell-Failover](/de/concepts/model-failover) (Rotations- und Abkühlregeln)
- [Slash-Befehle](/de/tools/slash-commands) (Befehlsoberfläche)

## Verwandte Themen

- [Authentifizierung](/de/gateway/authentication) – Überblick über die Authentifizierung bei Modell-Providern
- [Geheimnisse](/de/gateway/secrets) – Speicherung von Anmeldedaten und SecretRef
- [Konfigurationsreferenz](/de/gateway/configuration-reference#auth-storage) – Konfigurationsschlüssel für die Authentifizierung
