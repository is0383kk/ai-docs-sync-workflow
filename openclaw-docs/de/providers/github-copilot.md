---
read_when:
    - Sie möchten GitHub Copilot als Modell-Provider verwenden
    - Sie benötigen den `openclaw models auth login-github-copilot`-Ablauf
    - Sie wählen zwischen dem integrierten Copilot-Provider, dem Copilot-SDK-Harness und dem Copilot Proxy.
summary: Melden Sie sich über OpenClaw mit dem Geräteflow oder dem nicht interaktiven Tokenimport bei GitHub Copilot an
title: GitHub Copilot
x-i18n:
    generated_at: "2026-07-26T18:34:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e839e6c72e7e7cb106a2f98c62c4994b4f3d6f34a2e76b549f2f6ccfdac91fe6
    source_path: providers/github-copilot.md
    workflow: 16
---

GitHub Copilot ist der KI-Programmierassistent von GitHub. Er bietet Zugriff auf Copilot-
Modelle für Ihr GitHub-Konto und Ihren Tarif. OpenClaw kann Copilot auf drei verschiedene
Arten als Modell-Provider oder Agent-Runtime verwenden.

## Drei Möglichkeiten, Copilot in OpenClaw zu verwenden

<Tabs>
  <Tab title="Integrierter Provider (github-copilot)">
    Verwenden Sie den nativen Geräteanmeldeablauf, um ein GitHub-Token zu erhalten, und tauschen Sie es anschließend
    während der Ausführung von OpenClaw gegen Copilot-API-Token aus. Dies ist der **standardmäßige** und einfachste Weg,
    da dafür kein VS Code erforderlich ist.

    <Steps>
      <Step title="Anmeldebefehl ausführen">
        ```bash
        openclaw models auth login-github-copilot
        ```

        Sie werden aufgefordert, eine URL aufzurufen und einen Einmalcode einzugeben. Lassen Sie das
        Terminal geöffnet, bis der Vorgang abgeschlossen ist.
      </Step>
      <Step title="Standardmodell festlegen">
        ```bash
        openclaw models set github-copilot/claude-opus-4.7
        ```

        Oder in der Konfiguration:

        ```json5
        {
          agents: {
            defaults: { model: { primary: "github-copilot/claude-opus-4.7" } },
          },
        }
        ```
      </Step>
    </Steps>

  </Tab>

  <Tab title="Copilot-SDK-Harness-Plugin (copilot)">
    Installieren Sie das externe `@openclaw/copilot` Plugin, wenn die Copilot CLI
    und das SDK von GitHub die untergeordnete Agentenschleife für ausgewählte
    `github-copilot/*` Modelle steuern sollen.

    ```bash
    openclaw plugins install @openclaw/copilot
    ```

    Aktivieren Sie die Runtime anschließend für ein Modell oder einen Provider:

    ```json5
    {
      agents: {
        defaults: {
          model: "github-copilot/gpt-5.5",
          models: {
            "github-copilot/gpt-5.5": {
              agentRuntime: { id: "copilot" },
            },
          },
        },
      },
    }
    ```

    Wählen Sie diese Option, wenn Sie native Copilot-CLI-Sitzungen, einen vom SDK verwalteten
    Thread-Status und eine von Copilot gesteuerte Compaction für diese Agentendurchläufe wünschen. Ohne die
    ausdrückliche Aktivierung über `agentRuntime` verwenden `github-copilot/*` Modelle weiterhin den
    integrierten Provider. Den vollständigen Runtime-Vertrag finden Sie unter [Copilot-SDK-Harness](/de/plugins/copilot).

  </Tab>

  <Tab title="Copilot-Proxy-Plugin (copilot-proxy)">
    Verwenden Sie die VS-Code-Erweiterung **Copilot Proxy** als lokale Brücke. OpenClaw kommuniziert mit
    dem `/v1` Endpunkt des Proxys (standardmäßig `http://localhost:3000/v1`) und verwendet die von Ihnen
    konfigurierte Modellliste.

    Das `copilot-proxy` Plugin wird mit OpenClaw ausgeliefert und ist standardmäßig aktiviert.
    Konfigurieren Sie die Basis-URL und die Modell-IDs mit:

    ```bash
    openclaw models auth login --provider copilot-proxy --set-default
    ```

    <Note>
    Wählen Sie diese Option, wenn Copilot Proxy bereits in VS Code ausgeführt wird oder Sie Anfragen
    darüber weiterleiten müssen. Die VS-Code-Erweiterung muss weiterhin ausgeführt werden.
    </Note>

  </Tab>
</Tabs>

## GitHub Enterprise (Datenresidenz)

Wenn Ihre Organisation einen GitHub-Enterprise-Mandanten mit Datenresidenz verwendet (einen
`*.ghe.com` Host wie `your-org.ghe.com`), befindet sich Copilot auf mandantenlokalen
Endpunkten und nicht auf dem öffentlichen `github.com`. OpenClaw stellt dies als
vollwertige Authentifizierungsoption bereit, sodass Sie URLs nicht manuell bearbeiten müssen.

<Steps>
  <Step title="Enterprise-Authentifizierungsoption auswählen">
    Wählen Sie beim Onboarding oder in `openclaw models auth`
    **GitHub Copilot (Enterprise / data residency)** aus. Sie werden zur Eingabe
    Ihrer Enterprise-Domain aufgefordert (zum Beispiel `your-org.ghe.com`); anschließend wird die
    Geräteanmeldung bei diesem Mandanten ausgeführt.

    Geben Sie nur die Stammadresse des Mandanten ein (`your-org.ghe.com`). Abgeleitete Diensthosts wie
    `api.your-org.ghe.com` oder `copilot-api.your-org.ghe.com` werden nicht akzeptiert;
    OpenClaw leitet diese Endpunkte automatisch aus der Stammadresse des Mandanten ab.

    ```bash
    openclaw models auth login --provider github-copilot --method device-enterprise
    ```

  </Step>
  <Step title="Domain wird in der Konfiguration gespeichert">
    Der ausgewählte Host wird unter den Provider-Parametern gespeichert, sodass spätere Token-Aktualisierungen
    und Vervollständigungen automatisch an den Mandanten gerichtet werden:

    ```json5
    {
      models: {
        providers: {
          "github-copilot": { params: { githubDomain: "your-org.ghe.com" } },
        },
      },
    }
    ```

  </Step>
</Steps>

Der Geräteablauf, der Token-Austausch und die Vervollständigungen werden jeweils über
`https://your-org.ghe.com/login/device/code`,
`https://api.your-org.ghe.com/copilot_internal/v2/token` und
`https://copilot-api.your-org.ghe.com` abgewickelt. Datenresidenz-Token enthalten
eine Mandantenkennung und keinen Proxy-Hinweis, sodass die Basis-URL für Vervollständigungen auf den
Copilot-Host des Mandanten statt auf den öffentlichen Endpunkt zurückfällt.

<Note>
Beim Wechsel der Domain wird die Geräteanmeldung immer erneut ausgeführt. Wenn bereits ein
Copilot-Token gespeichert ist und Sie eine andere Domain auswählen (öffentliches `github.com` ↔ ein
`*.ghe.com` Mandant oder von einem Mandanten zu einem anderen), verwendet OpenClaw das vorhandene Token nicht erneut,
sondern erzwingt eine neue Anmeldung, damit das Token auf die Domain beschränkt ist, die in die
Konfiguration geschrieben wird. Bei einer erneuten Anmeldung für dieselbe Domain wird weiterhin angeboten, das aktuelle
Token wiederzuverwenden. Beim Wechsel zurück zum öffentlichen `github.com` wird das gespeicherte
`githubDomain` gelöscht, sodass die Konfiguration auf den Standard zurückgesetzt wird.
</Note>

<Note>
Die Umgebungsvariable `COPILOT_GITHUB_DOMAIN` überschreibt die aufgelöste Domain
für jeden Copilot-Pfad, der sie auflöst: die Enterprise-Geräteanmeldung
(`--method device-enterprise`), die eigenständige
`openclaw models auth login-github-copilot` Kurzform, Token-Aktualisierungen, Embeddings
und Vervollständigungen. Legen Sie sie für vollständig monitorlose oder CI-
Einrichtungen auf Ihren `*.ghe.com` Host fest. Lassen Sie sie ungesetzt (und den Konfigurationsparameter weg), um das öffentliche `github.com` zu verwenden.
Bei Anmeldungen wird die Domain gespeichert, für die das Token ausgestellt wurde (und bei der Anmeldung
über das öffentliche `github.com` gelöscht), sodass das Routing auch nach dem
Entfernen der Umgebungsvariable korrekt bleibt.
</Note>

## Optionale Flags

| Befehl                                                                 | Flag            | Beschreibung                                                 |
| ---------------------------------------------------------------------- | --------------- | ------------------------------------------------------------ |
| `openclaw models auth login-github-copilot`                                                     | `--yes` | Vorhandenes Authentifizierungsprofil ohne Nachfrage überschreiben |
| `openclaw models auth login --provider github-copilot --method device`                                                     | `--set-default` | Zusätzlich das vom Provider empfohlene Standardmodell anwenden |

```bash
# Bestätigung für die erneute Anmeldung überspringen
openclaw models auth login-github-copilot --yes

# Anmelden und das Standardmodell in einem Schritt festlegen
openclaw models auth login --provider github-copilot --method device --set-default
```

## Nicht interaktives Onboarding

Der Geräteanmeldeablauf erfordert ein interaktives TTY. Importieren Sie für eine monitorlose Einrichtung
mit `openclaw onboard --non-interactive` ein vorhandenes GitHub-OAuth-Zugriffstoken:

```bash
openclaw onboard --non-interactive --accept-risk \
  --auth-choice github-copilot \
  --github-copilot-token "$COPILOT_GITHUB_TOKEN" \
  --skip-channels --skip-health
```

Sie können `--auth-choice` auch weglassen; durch die Übergabe von `--github-copilot-token` wird die
Authentifizierungsoption des GitHub-Copilot-Providers abgeleitet. Wenn das Flag weggelassen wird, greift das Onboarding
nacheinander auf `COPILOT_GITHUB_TOKEN`, `GH_TOKEN` und dann `GITHUB_TOKEN` zurück. Verwenden Sie
`--secret-input-mode ref` bei gesetztem `COPILOT_GITHUB_TOKEN`, um ein umgebungsvariablenbasiertes
`tokenRef` statt Klartext in `auth-profiles.json` zu speichern.

<AccordionGroup>
  <Accordion title="Interaktives TTY erforderlich">
    Der Geräteanmeldeablauf erfordert ein interaktives TTY. Führen Sie ihn direkt in einem
    Terminal aus, nicht in einem nicht interaktiven Skript oder einer CI-Pipeline.
  </Accordion>

  <Accordion title="Modellverfügbarkeit hängt von Ihrem Tarif ab">
    Die Verfügbarkeit von Copilot-Modellen hängt von Ihrem GitHub-Tarif ab. Wenn ein Modell
    abgelehnt wird, versuchen Sie eine andere ID (zum Beispiel `github-copilot/gpt-5.5`). Die aktuelle Modellliste
    finden Sie in der GitHub-Dokumentation zu den [unterstützten Modellen je Copilot-Tarif](https://docs.github.com/en/copilot/reference/ai-models/supported-models#supported-ai-models-per-copilot-plan).
  </Accordion>

  <Accordion title="Live-Aktualisierung des Katalogs über die Copilot-API">
    Sobald über den Geräteanmelde- oder Umgebungsvariablen-Authentifizierungspfad ein GitHub-Token aufgelöst wurde,
    aktualisiert OpenClaw den Modellkatalog bei Bedarf über `${baseUrl}/models`
    (denselben Endpunkt, den VS Code Copilot verwendet), sodass die Runtime
    kontospezifische Berechtigungen und korrekte Kontextfenster ohne Änderungen am Manifest
    berücksichtigt. Neu veröffentlichte Copilot-Modelle werden ohne OpenClaw-
    Upgrade sichtbar, und die Kontextfenster entsprechen den tatsächlichen modellspezifischen Grenzwerten
    (z. B. 400k für die gpt-5.x-Reihe, 1M für die internen
    `claude-opus-*-1m` Varianten).

    Der mitgelieferte statische Katalog bleibt als sichtbare Ausweichoption bestehen, wenn die Erkennung
    deaktiviert ist, kein GitHub-Authentifizierungsprofil vorhanden ist, der Token-Austausch
    fehlschlägt oder beim HTTPS-Aufruf von `/models` ein Fehler auftritt. Um dies zu deaktivieren und sich vollständig
    auf den statischen Manifestkatalog zu verlassen (Offline-/Air-Gap-Szenarien):

    ```json5
    {
      plugins: {
        entries: {
          "github-copilot": {
            config: { discovery: { enabled: false } },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Transportauswahl">
    Claude-Modell-IDs verwenden automatisch den Anthropic-Messages-Transport.
    Gemini-Modelle verwenden den OpenAI-Chat-Completions-Transport; GPT- und o-Reihen-
    Modelle verwenden weiterhin den OpenAI-Responses-Transport. OpenClaw wählt den richtigen
    Transport anhand der Modellreferenz aus.
  </Accordion>

  <Accordion title="Anfragekompatibilität">
    OpenClaw sendet auf Copilot-Transporten Anfrage-Header im Stil der Copilot-IDE
    (Versionen des VS-Code-Editors/Plugins und die Integrations-ID `vscode-chat`),
    kennzeichnet auf Werkzeugergebnisse folgende Durchläufe als vom Agenten initiiert und setzt den Copilot-
    Vision-Header, wenn ein Durchlauf eine Bildeingabe enthält.
  </Accordion>

  <Accordion title="Auflösungsreihenfolge der Umgebungsvariablen">
    OpenClaw löst die Copilot-Authentifizierung anhand von Umgebungsvariablen in der folgenden
    Prioritätsreihenfolge auf:

    | Priorität | Variable              | Hinweise                               |
    | --------- | --------------------- | -------------------------------------- |
    | 1         | `COPILOT_GITHUB_TOKEN`    | Höchste Priorität, Copilot-spezifisch  |
    | 2         | `GH_TOKEN`    | GitHub-CLI-Token (Ausweichoption)      |
    | 3         | `GITHUB_TOKEN`    | Standard-GitHub-Token (niedrigste)     |

    Wenn mehrere Variablen gesetzt sind, verwendet OpenClaw diejenige mit der höchsten Priorität.
    Der Geräteanmeldeablauf (`openclaw models auth login-github-copilot`) speichert
    sein Token im Authentifizierungsprofilspeicher und hat Vorrang vor allen Umgebungsvariablen.

  </Accordion>

  <Accordion title="Token-Speicherung">
    Bei der Anmeldung wird ein GitHub-Token im Authentifizierungsprofilspeicher gespeichert (Profil-ID
    `github-copilot:github`) und während der Ausführung von OpenClaw gegen ein kurzlebiges Copilot-API-
    Token ausgetauscht. Sie müssen das Token nicht manuell verwalten.
  </Accordion>
</AccordionGroup>

## Embeddings für die Speichersuche

GitHub Copilot kann auch als Embedding-Provider für die
[Speichersuche](/de/concepts/memory-search) dienen. Wenn Sie über ein Copilot-Abonnement verfügen und
angemeldet sind, kann OpenClaw Copilot ohne separaten API-Schlüssel für Embeddings verwenden.

### Konfiguration

Legen Sie `memory.search.provider` ausdrücklich fest, um GitHub-Copilot-Embeddings zu verwenden. Wenn ein
GitHub-Token verfügbar ist, ermittelt OpenClaw verfügbare Embedding-Modelle über
die Copilot-API und wählt automatisch das beste aus.

```json5
{
  memory: {
    search: {
      provider: "github-copilot",
      // Optional: automatisch ermitteltes Modell überschreiben
      model: "text-embedding-3-small",
    },
  },
}
```

### Funktionsweise

1. OpenClaw löst Ihr GitHub-Token auf (aus Umgebungsvariablen oder dem Authentifizierungsprofil).
2. Es wird gegen ein kurzlebiges Copilot-API-Token ausgetauscht.
3. Der Copilot-Endpunkt `/models` wird abgefragt, um verfügbare Embedding-Modelle zu ermitteln.
4. Das beste Modell wird ausgewählt (Präferenzreihenfolge: `text-embedding-3-small`,
   `text-embedding-3-large`, `text-embedding-ada-002`).
5. Embedding-Anfragen werden an den Copilot-Endpunkt `/embeddings` gesendet.

Die Modellverfügbarkeit hängt von Ihrem GitHub-Tarif ab. Wenn keine Embedding-Modelle
verfügbar sind, überspringt OpenClaw Copilot und versucht den nächsten Provider.

## Verwandte Themen

<CardGroup cols={2}>
  <Card title="Modellauswahl" href="/de/concepts/model-providers" icon="layers">
    Auswahl von Providern, Modellreferenzen und Failover-Verhalten.
  </Card>
  <Card title="OAuth und Authentifizierung" href="/de/gateway/authentication" icon="key">
    Details zur Authentifizierung und Regeln für die Wiederverwendung von Anmeldedaten.
  </Card>
</CardGroup>
