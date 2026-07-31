---
read_when:
    - Anpassen der Standardeinstellungen, Zulassungslisten oder des Verhaltens von Slash-Befehlen im erhöhten Modus
    - Verstehen, wie isolierte Agenten auf den Host zugreifen können
summary: 'Erweiterter Ausführungsmodus: Befehle von einem Sandbox-Agenten außerhalb der Sandbox ausführen'
title: Erweiterter Modus
x-i18n:
    generated_at: "2026-07-26T18:49:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 40627217acf56122acfc48b689be1b9e2c61d889fe698e9c3c8fd91270d4a6cf
    source_path: tools/elevated.md
    workflow: 16
---

Wenn ein Agent innerhalb einer Sandbox ausgeführt wird, sind seine `exec`-Befehle auf die Sandbox-Umgebung beschränkt. Im **erweiterten Modus** kann der Agent stattdessen die Sandbox verlassen und Befehle außerhalb der Sandbox ausführen, wobei konfigurierbare Genehmigungsschranken gelten.

<Info>
  Der erweiterte Modus ändert das Verhalten nur, wenn der Agent in einer **Sandbox** ausgeführt wird. Bei Agenten ohne Sandbox wird exec bereits auf dem Host ausgeführt.
</Info>

## Direktiven

Steuern Sie den erweiterten Modus sitzungsbezogen mit Slash-Befehlen:

| Direktive        | Funktion                                                                                                                    |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `/elevated on`   | Außerhalb der Sandbox im konfigurierten Hostpfad ausführen und Genehmigungen beibehalten                                                             |
| `/elevated ask`  | Identisch mit `on` (Alias)                                                                                                            |
| `/elevated full` | Außerhalb der Sandbox im konfigurierten Hostpfad ausführen und Genehmigungen überspringen, wenn die Genehmigungsrichtlinie für Modus/Host bereits permissiv ist |
| `/elevated off`  | Zur auf die Sandbox beschränkten Ausführung zurückkehren                                                                                            |

Auch als `/elev on|off|ask|full` verfügbar.

Senden Sie `/elevated` ohne Argument, um die aktuelle Stufe anzuzeigen.

## Funktionsweise

<Steps>
  <Step title="Verfügbarkeit prüfen">
    Elevated muss in der Konfiguration aktiviert sein und der Absender muss auf der Zulassungsliste stehen:

    ```json5
    {
      tools: {
        elevated: {
          enabled: true,
          allowFrom: {
            discord: ["user-id-123"],
            whatsapp: ["+15555550123"],
          },
        },
      },
    }
    ```

  </Step>

  <Step title="Stufe festlegen">
    Senden Sie eine Nachricht, die ausschließlich eine Direktive enthält, um den Sitzungsstandard festzulegen:

    ```
    /elevated full
    ```

    Oder verwenden Sie sie inline (gilt nur für diese Nachricht):

    ```
    /elevated on führen Sie das Bereitstellungsskript aus
    ```

  </Step>

  <Step title="Befehle außerhalb der Sandbox ausführen">
    Wenn Elevated aktiv ist, verlassen `exec`-Aufrufe die Sandbox. Der effektive Host ist standardmäßig
    `gateway` oder `node`, wenn das konfigurierte bzw. sitzungsbezogene exec-Ziel
    `node` ist. Im Modus `full` werden exec-Genehmigungen übersprungen, wenn die aufgelöste Genehmigungsrichtlinie für exec-
    Modus/Host bereits vollständig permissiv ist (Sicherheit `full`,
    Abfrage `off`); andernfalls gilt weiterhin die normale Genehmigungsrichtlinie. Im Modus
    `on`/`ask` gelten die konfigurierten Genehmigungsregeln immer.
  </Step>
</Steps>

## Auflösungsreihenfolge

1. **Inline-Direktive** in der Nachricht (gilt nur für diese Nachricht)
2. **Sitzungsüberschreibung** (wird durch Senden einer Nachricht festgelegt, die ausschließlich eine Direktive enthält)
3. **Globaler Standard** (`agents.defaults.elevatedDefault` in der Konfiguration)

## Verfügbarkeit und Zulassungslisten

- **Globale Schranke**: `tools.elevated.enabled` (muss `true` sein)
- **Absender-Zulassungsliste**: `tools.elevated.allowFrom` mit kanalspezifischen Listen
- **Agentenspezifische Schranke**: `agents.entries.*.tools.elevated.enabled` (kann nur weiter einschränken; sowohl die globale als auch die agentenspezifische Schranke müssen `true` sein)
- **Agentenspezifische Zulassungsliste**: `agents.entries.*.tools.elevated.allowFrom` (der Absender muss sowohl der globalen als auch der agentenspezifischen Liste entsprechen)
- **Vom Kanal bereitgestellte Fallback-Zulassungsliste**: Kanal-Plugins können optional über einen SDK-Adapter-Hook eine Fallback-Zulassungsliste bereitstellen, die verwendet wird, wenn `tools.elevated.allowFrom.<provider>` nicht konfiguriert ist. Derzeit implementiert kein gebündelter Kanal diesen Hook, daher benötigt heute in der Praxis jeder Provider einen expliziten `tools.elevated.allowFrom.<provider>`-Eintrag.
- **Alle Schranken müssen passiert werden**; andernfalls gilt Elevated als nicht verfügbar

Formate für Einträge in der Zulassungsliste:

| Präfix                  | Übereinstimmung                         |
| ----------------------- | ------------------------------- |
| (keines)                  | Absender-ID, E.164 oder From-Feld |
| `name:`                 | Anzeigename des Absenders             |
| `username:`             | Benutzername des Absenders                 |
| `tag:`                  | Tag des Absenders                      |
| `id:`, `from:`, `e164:` | Explizite Identitätsadressierung     |

## Was Elevated nicht steuert

- **Tool-Richtlinie**: Wenn `exec` durch die Tool-Richtlinie verweigert wird, kann Elevated dies nicht außer Kraft setzen.
- **Hostauswahlrichtlinie**: Elevated macht `auto` nicht zu einer uneingeschränkten hostübergreifenden Überschreibung. Es verwendet die konfigurierten bzw. sitzungsbezogenen Regeln für das exec-Ziel und wählt `node` nur aus, wenn das Ziel bereits `node` ist.
- **Unabhängig von `/exec`**: Die Direktive `/exec` passt für autorisierte Absender die sitzungsbezogenen exec-Standardwerte (Host, Sicherheit, Abfrage, Node) an und erfordert den erweiterten Modus nicht.

<Note>
  Der Bash-Chatbefehl (Präfix `!`; Alias `/bash`) ist eine separate Schranke, für die zusätzlich zu ihrem eigenen Flag `tools.bash.enabled` auch `tools.elevated` aktiviert sein muss. Durch das Deaktivieren von Elevated werden auch `!`-Shell-Befehle gesperrt.
</Note>

## Verwandte Themen

<CardGroup cols={2}>
  <Card title="Exec-Tool" href="/de/tools/exec" icon="terminal">
    Ausführung von Shell-Befehlen durch den Agenten.
  </Card>
  <Card title="Exec-Genehmigungen" href="/de/tools/exec-approvals" icon="shield">
    Genehmigungs- und Zulassungssystem für `exec`.
  </Card>
  <Card title="Sandboxing" href="/de/gateway/sandboxing" icon="box">
    Sandbox-Konfiguration auf Gateway-Ebene.
  </Card>
  <Card title="Sandbox, Tool-Richtlinie und Elevated im Vergleich" href="/de/gateway/sandbox-vs-tool-policy-vs-elevated" icon="scale-balanced">
    Wie die drei Schranken während eines Tool-Aufrufs zusammenwirken.
  </Card>
</CardGroup>
