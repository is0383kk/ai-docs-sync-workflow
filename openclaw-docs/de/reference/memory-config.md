---
read_when:
    - Sie möchten Provider für die Speichersuche oder Embedding-Modelle konfigurieren
    - Sie möchten das QMD-Backend einrichten
    - Sie möchten die hybride Suche, MMR oder zeitlichen Verfall aktivieren
    - Sie möchten die multimodale Speicherindizierung aktivieren
sidebarTitle: Memory config
summary: Provider für Speichersuche, Abrufmodi, QMD und multimodale Indizierung
title: Referenz zur Speicherkonfiguration
x-i18n:
    generated_at: "2026-07-26T18:36:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 91f843b1516093c49e18b3d659ab24ea9cb7be32aaaac722205eca8bc3f2ca5b
    source_path: reference/memory-config.md
    workflow: 16
---

Diese Seite führt alle Konfigurationsoptionen für die OpenClaw-Speichersuche auf. Konzeptionelle Übersichten finden Sie unter:

<CardGroup cols={2}>
  <Card title="Speicherübersicht" href="/de/concepts/memory">
    Funktionsweise des Speichers.
  </Card>
  <Card title="Integrierte Engine" href="/de/concepts/memory-builtin">
    Standardmäßiges SQLite-Backend.
  </Card>
  <Card title="QMD-Engine" href="/de/concepts/memory-qmd">
    Local-First-Sidecar.
  </Card>
  <Card title="Speichersuche" href="/de/concepts/memory-search">
    Suchpipeline und Optimierung.
  </Card>
  <Card title="Active Memory" href="/de/concepts/active-memory">
    Speicher-Subagent für interaktive Sitzungen.
  </Card>
</CardGroup>

Alle gemeinsamen Speichereinstellungen befinden sich unter `memory` auf oberster Ebene in `openclaw.json`. Suchstandards verwenden `memory.search`; agentenspezifische Suchüberschreibungen verwenden `agents.entries.*.memory.search`.

<Note>
Verwenden Sie für den empfohlenen persönlichen Agenten-Workflow
`memory.search.rememberAcrossConversations`. Erweiterte Steuerelemente für Zielauswahl,
Modell, Prompt und Latenz von Active Memory befinden sich unter `plugins.entries.active-memory`.

Informationen zu beiden Aktivierungspfaden, zur Transkriptpersistenz und zu
Hinweisen für eine sichere Einführung finden Sie unter [Active Memory](/de/concepts/active-memory).
</Note>

---

## Gesprächsübergreifend erinnern

| Schlüssel                     | Typ       | Standard                                                   | Beschreibung                                                                 |
| ----------------------------- | --------- | ---------------------------------------------------------- | ---------------------------------------------------------------------------- |
| `rememberAcrossConversations` | `boolean` | Bei persönlichen Installationen aktiviert; bei konfigurierter DM-Isolierung deaktiviert | Relevanten Kontext aus anderen erkannten privaten Gesprächen dieses Agenten verwenden. |

Konfigurieren Sie dies agentenspezifisch, wenn nur ein vertrauenswürdiger
persönlicher Agent den gesprächsübergreifenden Transkriptabruf verwenden soll:

```json5
{
  agents: {
    entries: {
      personal: {
        memory: {
          search: {
            rememberAcrossConversations: true,
          },
        },
      },
    },
  },
}
```

Der Wert folgt der normalen Vererbung von `memory.search` mit einer
agentenspezifischen Überschreibung. Wenn er nicht gesetzt ist, ist er standardmäßig
nur aktiviert, wenn das globale `session.dmScope` nicht gesetzt oder `"main"` ist
und keine Bindung eine `session.dmScope`-Überschreibung besitzt. Jede konfigurierte
DM-Isolierung deaktiviert ihn standardmäßig. Ein explizites `true` oder
`false` hat immer Vorrang. Die Aktivierung impliziert die Indizierung von
Sitzungstranskripten und fügt `sessions` zu den aufgelösten Speicherquellen
des Agenten hinzu. Bei QMD aktiviert sie außerdem den Sitzungsexport dieses Agenten;
für diesen Modus ist keine separate Einstellung
`memory.qmd.sessions.enabled` erforderlich.

Der integrierte Speicher-Provider von OpenClaw unterstützt diesen geschützten Pfad
sowohl mit dem integrierten als auch mit dem QMD-Backend. Alternative Speicher-Provider
können weiterhin ihre eigenen Abruf-Hooks und erweiterten Active-Memory-Werkzeuge
verwenden, diese Einstellung wird jedoch übersprungen, sofern der aktuelle Provider
keinen geschützten Abruf privater Transkripte unterstützt.
`openclaw doctor` meldet einen nicht unterstützten Provider oder eine explizite
Active-Memory-Liste `toolsAllow`, in der `memory_search` fehlt.

Die Abrufgrenze ist enger als bei der allgemeinen Sitzungssuche:

- nur erkannte private Gespräche desselben Agenten sind zulässig
- das aktuell beantwortete Gespräch ist ausgeschlossen
- Gruppen und Kanäle sind als Quellen und Ziele ausgeschlossen
- unbekannte Gesprächsarten werden standardmäßig abgelehnt
- der Abruf in einer Sandbox kann die spezielle gesprächsübergreifende Autorisierung nicht verwenden

Die Einstellung ändert weder `tools.sessions.visibility` noch Sitzungsschlüssel,
Transkriptspeicherung, Zustellungsrouting oder die Berechtigungen von `sessions_list`,
`sessions_history` und `sessions_send`. Active Memory führt einen begrenzten,
schreibgeschützten Abrufdurchlauf aus; ein nicht verfügbarer oder wegen Zeitüberschreitung
abgebrochener Abruf blockiert die Antwort nicht.

---

## Provider-Auswahl

| Schlüssel  | Typ       | Standard         | Beschreibung                                                                                                                                                                                                                                                                                 |
| ---------- | --------- | ---------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `enabled`  | `boolean` | `true`           | Speichersuche aktivieren oder deaktivieren                                                                                                                                                                                                                                                   |
| `provider` | `string`  | `"openai"`       | ID des Embedding-Adapters, beispielsweise `bedrock`, `deepinfra`, `gemini`, `github-copilot`, `local`, `mistral`, `ollama`, `openai`, `openai-compatible` oder `voyage`; kann auch ein konfigurierter `models.providers.<id>` sein, dessen `api` auf einen Speicher-Embedding-Adapter oder eine OpenAI-kompatible Modell-API verweist |
| `model`    | `string`  | Provider-Standard | Name des Embedding-Modells                                                                                                                                                                                                                                                                   |
| `fallback` | `string`  | `"none"`         | ID des Fallback-Adapters, wenn der primäre Adapter ausfällt                                                                                                                                                                                                                                  |

Wenn `provider` nicht gesetzt ist, verwendet OpenClaw OpenAI-Embeddings.
Legen Sie `provider` explizit fest, um Bedrock, DeepInfra, Gemini,
GitHub Copilot, Mistral, Ollama, Voyage, ein lokales GGUF-Modell oder einen
OpenAI-kompatiblen `/v1/embeddings`-Endpunkt zu verwenden.
Legacy-Konfigurationen, die noch `provider: "auto"` angeben, werden als
`openai` aufgelöst.

<Warning>
Eine Änderung des Embedding-Providers, des Modells, der Provider-Einstellungen,
der Quellen, des Geltungsbereichs, der Segmentierung oder des Tokenizers kann
den vorhandenen SQLite-Vektorindex inkompatibel machen. OpenClaw pausiert die
Vektorsuche und meldet eine Warnung zur Indexidentität, statt automatisch alles
neu einzubetten. Erstellen Sie den Index mit `openclaw memory status --index --agent <id>` oder
`openclaw memory index --force --agent <id>` neu, sobald Sie bereit sind.
</Warning>

Wenn `provider` nicht gesetzt ist, das veraltete `provider: "auto"`
vorhanden ist oder `provider: "none"` absichtlich den reinen FTS-Modus auswählt,
kann der Speicherabruf weiterhin die lexikalische FTS-Rangfolge verwenden, wenn
Embeddings nicht verfügbar sind.

Explizite nicht lokale Provider werden standardmäßig abgelehnt. Wenn Sie
`memory.search.provider` auf einen konkreten, remote angebundenen Provider wie Bedrock,
DeepInfra, Gemini, GitHub Copilot, LM Studio, Mistral, Ollama, OpenAI, Voyage oder
einen OpenAI-kompatiblen benutzerdefinierten Provider setzen und dieser Provider
zur Laufzeit nicht verfügbar ist, gibt `memory_search` ein Ergebnis des Typs
„nicht verfügbar“ zurück, statt unbemerkt einen reinen FTS-Abruf zu verwenden.
Korrigieren Sie die Provider-/Authentifizierungskonfiguration, wechseln Sie zu
einem erreichbaren Provider oder setzen Sie `provider: "none"`, wenn Sie bewusst
einen reinen FTS-Abruf verwenden möchten.

### Benutzerdefinierte Provider-IDs

`memory.search.provider` kann auf einen benutzerdefinierten `models.providers.<id>`-Eintrag
für speicherspezifische Provider-Adapter wie `ollama` oder für
OpenAI-kompatible Modell-APIs wie `openai-responses` / `openai-completions`
verweisen. OpenClaw löst den `api`-Owner dieses Providers für den
Embedding-Adapter auf und behält dabei die benutzerdefinierte Provider-ID für
die Verarbeitung von Endpunkt, Authentifizierung und Modellpräfix bei. Dadurch
können Multi-GPU- oder Multi-Host-Konfigurationen Speicher-Embeddings einem
bestimmten lokalen Endpunkt zuweisen:

```json5
{
  models: {
    providers: {
      "ollama-5080": {
        api: "ollama",
        baseUrl: "http://gpu-box.local:11435",
        apiKey: "ollama-local",
        models: [{ id: "qwen3-embedding:0.6b", name: "Qwen3 Embedding 0.6B" }],
      },
    },
  },
  memory: {
    search: {
      provider: "ollama-5080",
      model: "qwen3-embedding:0.6b",
    },
  },
}
```

### Auflösung des API-Schlüssels

Remote-Embeddings erfordern einen API-Schlüssel. Bedrock verwendet stattdessen
die standardmäßige Anmeldedatenkette des AWS SDK (Instanzrollen, SSO,
Zugriffsschlüssel oder einen Bedrock-API-Schlüssel).

| Provider       | Umgebungsvariable                                   | Konfigurationsschlüssel              |
| -------------- | --------------------------------------------------- | ------------------------------------ |
| Bedrock        | AWS-Anmeldedatenkette oder `AWS_BEARER_TOKEN_BEDROCK` | Kein API-Schlüssel erforderlich      |
| DeepInfra      | `DEEPINFRA_API_KEY`                                 | `models.providers.deepinfra.apiKey` |
| Gemini         | `GEMINI_API_KEY`                                    | `models.providers.google.apiKey`    |
| GitHub Copilot | `COPILOT_GITHUB_TOKEN`, `GH_TOKEN`, `GITHUB_TOKEN`  | Authentifizierungsprofil über Geräteanmeldung |
| Mistral        | `MISTRAL_API_KEY`                                   | `models.providers.mistral.apiKey`   |
| Ollama         | `OLLAMA_API_KEY` (Platzhalter)                      | --                                   |
| OpenAI         | `OPENAI_API_KEY`                                    | `models.providers.openai.apiKey`    |
| Voyage         | `VOYAGE_API_KEY`                                    | `models.providers.voyage.apiKey`    |

<Note>
Codex OAuth deckt nur Chat/Vervollständigungen ab und erfüllt keine
Embedding-Anfragen.
</Note>

---

## Konfiguration des Remote-Endpunkts

Verwenden Sie `provider: "openai-compatible"` für einen generischen OpenAI-kompatiblen
`/v1/embeddings`-Server, der keine globalen OpenAI-Chat-Anmeldedaten erben soll.

<ParamField path="remote.baseUrl" type="string">
  Benutzerdefinierte API-Basis-URL.
</ParamField>
<ParamField path="remote.apiKey" type="string">
  API-Schlüssel überschreiben.
</ParamField>
<ParamField path="remote.headers" type="object">
  Zusätzliche HTTP-Header (mit den Provider-Standards zusammengeführt).
</ParamField>

```json5
{
  memory: {
    search: {
      provider: "openai-compatible",
      model: "text-embedding-3-small",
      remote: {
        baseUrl: "https://api.example.com/v1/",
        apiKey: "YOUR_KEY",
      },
    },
  },
}
```

---

## Providerspezifische Konfiguration

<AccordionGroup>
  <Accordion title="Gemini">
    | Schlüssel              | Typ      | Standard               | Beschreibung                                |
    | ---------------------- | -------- | ---------------------- | ------------------------------------------- |
    | `model`                | `string` | `gemini-embedding-001` | Unterstützt auch `gemini-embedding-2-preview` |
    | `outputDimensionality` | `number` | `3072`                 | Für Embedding 2: 768, 1536 oder 3072        |

    <Warning>
    Eine Änderung des Modells oder von `outputDimensionality` ändert die Indexidentität.
    OpenClaw pausiert die Vektorsuche, bis Sie den Speicherindex explizit neu erstellen.
    </Warning>

  </Accordion>
  <Accordion title="Eingabetypen für OpenAI-Kompatibilität">
    OpenAI-kompatible Embedding-Endpunkte können providerspezifische
    `input_type`-Anfragefelder aktivieren. Dies ist für asymmetrische
    Embedding-Modelle nützlich, die unterschiedliche Bezeichnungen für Abfrage-
    und Dokument-Embeddings erfordern.

    | Schlüssel           | Typ      | Standard    | Beschreibung                                                   |
    | ------------------- | -------- | ----------- | -------------------------------------------------------------- |
    | `inputType`         | `string` | nicht gesetzt | Gemeinsames `input_type` für Abfrage- und Dokument-Embeddings |
    | `queryInputType`    | `string` | nicht gesetzt | `input_type` zur Abfragezeit; überschreibt `inputType` |
    | `documentInputType` | `string` | nicht gesetzt | `input_type` für Index/Dokument; überschreibt `inputType` |

    ```json5
    {
      memory: {
        search: {
          provider: "openai-compatible",
          remote: {
            baseUrl: "https://embeddings.example/v1",
            apiKey: "${EMBEDDINGS_API_KEY}",
          },
          model: "asymmetric-embedder",
          queryInputType: "query",
          documentInputType: "passage",
        },
      },
    }
    ```

    Änderungen an diesen Werten wirken sich auf die Identität des Embedding-Caches für die Batch-Indexierung des Providers aus. Wenn das vorgelagerte Modell die Bezeichnungen unterschiedlich behandelt, sollte anschließend der Speicher neu indexiert werden.

  </Accordion>
  <Accordion title="Bedrock">
    ### Bedrock-Embedding-Konfiguration

    Bedrock verwendet die standardmäßige Anmeldedatenkette des AWS SDK sowie ein von OpenClaw geprüftes Bearer-Token, sodass keine API-Schlüssel in der Konfiguration gespeichert werden. Wenn OpenClaw auf EC2 mit einer für Bedrock aktivierten Instanzrolle ausgeführt wird, legen Sie lediglich Provider und Modell fest:

    ```json5
    {
      memory: {
        search: {
          provider: "bedrock",
          model: "amazon.titan-embed-text-v2:0",
        },
      },
    }
    ```

    | Schlüssel              | Typ      | Standard                        | Beschreibung                         |
    | ---------------------- | -------- | ------------------------------- | ------------------------------------ |
    | `model`     | `string` | `amazon.titan-embed-text-v2:0` | Beliebige Bedrock-Embedding-Modell-ID |
    | `outputDimensionality`     | `number` | Modellstandard                  | Für Titan V2: 256, 512 oder 1024     |

    **Unterstützte Modelle** (mit Familienerkennung und Standarddimensionen):

    | Modell-ID                                   | Provider   | Standarddimensionen | Konfigurierbare Dimensionen |
    | ------------------------------------------- | ---------- | ------------------- | --------------------------- |
    | `amazon.titan-embed-text-v2:0`             | Amazon     | 1024         | 256, 512, 1024             |
    | `amazon.titan-embed-text-v1`               | Amazon     | 1536         | --                          |
    | `amazon.titan-embed-g1-text-02`            | Amazon     | 1536         | --                          |
    | `amazon.titan-embed-image-v1`              | Amazon     | 1024         | --                          |
    | `amazon.nova-2-multimodal-embeddings-v1:0` | Amazon     | 1024         | 256, 384, 1024, 3072       |
    | `cohere.embed-english-v3`                  | Cohere     | 1024         | --                          |
    | `cohere.embed-multilingual-v3`             | Cohere     | 1024         | --                          |
    | `cohere.embed-v4:0`                        | Cohere     | 1536         | 256, 384, 512, 768, 1024, 1536 |
    | `twelvelabs.marengo-embed-3-0-v1:0`        | TwelveLabs | 512          | --                          |
    | `twelvelabs.marengo-embed-2-7-v1:0`        | TwelveLabs | 1024         | --                          |

    Varianten mit Durchsatzsuffix (z. B. `amazon.titan-embed-text-v1:2:8k`) und Inferenzprofil-IDs mit Regionspräfix (z. B. `us.amazon.titan-embed-text-v2:0`) übernehmen die Konfiguration des Basismodells.

    **Region:** wird in dieser Reihenfolge ermittelt: die Überschreibung `memory.search.remote.baseUrl`, die Konfiguration `models.providers.amazon-bedrock.baseUrl`, `AWS_REGION`, `AWS_DEFAULT_REGION` und anschließend der Standardwert `us-east-1`.

    **Authentifizierung:** OpenClaw prüft zunächst auf `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` oder `AWS_BEARER_TOKEN_BEDROCK` und greift danach auf die standardmäßige Anmeldedaten-Provider-Kette des AWS SDK zurück:

    1. Umgebungsvariablen (`AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY`), sofern nicht auch `AWS_PROFILE` festgelegt ist
    2. SSO (nur wenn SSO-Felder konfiguriert sind)
    3. Freigegebene Anmeldedaten- und Konfigurationsdateien (`fromIni`, einschließlich `AWS_PROFILE`)
    4. Anmeldedatenprozess (`credential_process` in der AWS-Konfigurationsdatei)
    5. Anmeldedaten für Webidentitäts-Token
    6. Anmeldedaten aus ECS- oder EC2-Instanzmetadaten

    **IAM-Berechtigungen:** Die IAM-Rolle oder der IAM-Benutzer benötigt:

    ```json
    {
      "Effect": "Allow",
      "Action": "bedrock:InvokeModel",
      "Resource": "*"
    }
    ```

    Beschränken Sie für das Prinzip der geringsten Rechte `InvokeModel` auf das jeweilige Modell:

    ```text
    arn:aws:bedrock:*::foundation-model/amazon.titan-embed-text-v2:0
    ```

  </Accordion>
  <Accordion title="Lokal (GGUF + llama.cpp)">
    | Schlüssel              | Typ                | Standard                | Beschreibung                                                                                                                                                                                                                                                                                                          |
    | ---------------------- | ------------------ | ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
    | `local.modelPath`     | `string` | automatisch heruntergeladen | Pfad zur GGUF-Modelldatei                                                                                                                                                                                                                                                                                         |
    | `local.modelCacheDir`     | `string` | node-llama-cpp-Standard | Cache-Verzeichnis für heruntergeladene Modelle                                                                                                                                                                                                                                                                       |
    | `local.contextSize`     | `number \| "auto"` | `4096`      | Größe des Kontextfensters für den Embedding-Kontext. 4096 deckt typische Abschnitte (128–512 Token) ab und begrenzt zugleich den nicht durch Gewichtungen belegten VRAM. Auf eingeschränkten Hosts auf 1024–2048 reduzieren. `"auto"` verwendet das trainierte Maximum des Modells – für Modelle ab 8B nicht empfohlen (Qwen3-Embedding-8B: Bis zu 40 960 Token können den VRAM-Bedarf auf ~32 GB erhöhen). |

    Installieren Sie zunächst den offiziellen llama.cpp-Provider: `openclaw plugins install @openclaw/llama-cpp-provider`.
    Standardmodell: `embeddinggemma-300m-qat-Q8_0.gguf` (~0,6 GB, wird automatisch heruntergeladen). Quellcode-Checkouts erfordern weiterhin eine Genehmigung des nativen Builds: `pnpm approve-builds` und anschließend `pnpm rebuild node-llama-cpp`.

    Verwenden Sie die eigenständige CLI, um denselben Provider-Pfad zu überprüfen, den der Gateway verwendet:

    ```bash
    openclaw memory status --deep --agent main
    openclaw memory index --force --agent main
    ```

    Numerische Werte für `local.contextSize` beeinflussen außerdem die automatische Platzierung der GPU-Schichten durch node-llama-cpp, sodass Modellgewichtungen und der angeforderte Embedding-Kontext gemeinsam eingepasst werden. `openclaw memory status --deep` meldet den zuletzt bekannten llama.cpp-Backend-, Geräte- und Auslagerungsstatus sowie den angeforderten Kontext und mit Zeitstempeln versehene Speicherinformationen, nachdem die Laufzeit geladen wurde; eine passive Statusabfrage lädt kein Modell.

    Legen Sie `provider: "local"` für lokale GGUF-Embeddings explizit fest. `hf:` und HTTP(S)-Modellreferenzen werden für explizite lokale Konfigurationen unterstützt (über die Modellauflösung von node-llama-cpp), ändern jedoch nicht den standardmäßigen Provider.

  </Accordion>
</AccordionGroup>

## Indexierungsverhalten

Speicher-Engines verwalten Synchronisierung, Batch-Verarbeitung, Überwachung und
Indexierungsheuristiken nach der Compaction. OpenClaw hält diese Verhaltensweisen mit gepflegten
Standardwerten aktiviert, anstatt installationsspezifische Zeitsteuerungsoptionen bereitzustellen.

## Konfiguration der hybriden Suche

Alle unter `memory.search.query`:

| Schlüssel    | Typ      | Standard | Beschreibung                                               |
| ------------ | -------- | -------- | ---------------------------------------------------------- |
| `maxResults` | `number` | `6` | Maximale Anzahl vor der Einfügung zurückgegebener Speichertreffer |
| `minScore` | `number` | `0.35` | Mindestrelevanzwert zum Einbeziehen eines Treffers          |

Der hybride Abruf bleibt aktiviert; MMR und zeitlicher Zerfall bleiben durch
die integrierte Engine-Richtlinie deaktiviert.

### Vollständiges Beispiel

```json5
{
  memory: {
    search: {
      query: {
        maxResults: 6,
        minScore: 0.35,
      },
    },
  },
}
```

---

## Zusätzliche Speicherpfade

| Schlüssel    | Typ        | Beschreibung                                      |
| ------------ | ---------- | ------------------------------------------------- |
| `extraPaths` | `string[]` | Zusätzliche zu indexierende Verzeichnisse oder Dateien |

```json5
{
  memory: {
    search: {
      extraPaths: ["../team-docs", "/srv/shared-notes"],
    },
  },
}
```

Pfade können absolut oder relativ zum Arbeitsbereich sein. Verzeichnisse werden rekursiv nach `.md`-Dateien durchsucht. Die Behandlung symbolischer Verknüpfungen hängt vom aktiven Backend ab: Die integrierte Engine überspringt symbolische Verknüpfungen, während QMD dem Verhalten des zugrunde liegenden QMD-Scanners folgt.

Verwenden Sie für die agentenspezifische, agentenübergreifende Transkriptsuche `agents.entries.*.memory.search.qmd.extraCollections` anstelle von `memory.qmd.paths`. Diese zusätzlichen Sammlungen folgen derselben `{ path, name, pattern? }`-Struktur, werden jedoch pro Agent zusammengeführt und können explizite gemeinsam verwendete Namen beibehalten, wenn der Pfad außerhalb des aktuellen Arbeitsbereichs liegt. Wenn derselbe aufgelöste Pfad sowohl in `memory.qmd.paths` als auch in `memory.search.qmd.extraCollections` vorkommt, behält QMD den ersten Eintrag bei und überspringt das Duplikat.

---

## Multimodaler Speicher (Gemini)

Indexieren Sie Bilder und Audiodateien zusammen mit Markdown mithilfe von Gemini Embedding 2:

| Schlüssel                 | Typ                 | Standard             | Beschreibung                              |
| ------------------------- | ------------------- | -------------------- | ----------------------------------------- |
| `multimodal.enabled`        | `boolean`  | `false`   | Multimodale Indexierung aktivieren         |
| `multimodal.modalities`        | `string[]`  | --                   | `["image"]`, `["audio"]` oder `["all"]` |
| `multimodal.maxFileBytes`        | `number`  | `10485760`   | Maximale Dateigröße für die Indexierung (10 MiB) |

<Note>
Gilt nur für Dateien in `extraPaths`. Standardspeicherstammverzeichnisse bleiben auf Markdown beschränkt. Erfordert `gemini-embedding-2-preview`. `fallback` muss `"none"` sein.
</Note>

Unterstützte Formate: `.jpg`, `.jpeg`, `.png`, `.webp`, `.gif`, `.heic`, `.heif` (Bilder); `.mp3`, `.wav`, `.ogg`, `.opus`, `.m4a`, `.aac`, `.flac` (Audio).

---

## Embedding-Cache

| Schlüssel        | Typ                 | Standard             | Beschreibung                              |
| ---------------- | ------------------- | -------------------- | ----------------------------------------- |
| `cache.enabled` | `boolean` | `true` | Embeddings von Abschnitten in SQLite zwischenspeichern |

Verhindert, dass unveränderter Text bei einer Neuindexierung oder bei Transkriptaktualisierungen erneut eingebettet wird.

---

## Batch-Indexierung

| Schlüssel                 | Typ                 | Standard             | Beschreibung                    |
| ------------------------- | ------------------- | -------------------- | ------------------------------- |
| `remote.nonBatchConcurrency`        | `number`  | `4`   | Parallele Inline-Embeddings      |
| `remote.batch.enabled`        | `boolean`  | `false`   | Batch-Embedding-API aktivieren   |

Verfügbar für `gemini`, `openai` und `voyage`. Die Batch-Verarbeitung von OpenAI ist bei umfangreichen Nachindexierungen üblicherweise am schnellsten und kostengünstigsten.

Parallelität, Abfrageintervalle und Zeitüberschreitungsverhalten werden vom Provider verwaltet.

---

## Sitzungsspeichersuche

Indexieren Sie Sitzungstranskripte und stellen Sie sie über `memory_search` bereit:

| Schlüssel                   | Typ                 | Standard             | Beschreibung                                      |
| --------------------------- | ------------------- | -------------------- | ------------------------------------------------- |
| `rememberAcrossConversations`          | `boolean`  | `false`   | Privaten Abruf über mehrere Konversationen hinweg zulassen |
| `sources`          | `string[]`  | `["memory"]`   | `"sessions"` hinzufügen, um Transkripte einzubeziehen |

<Warning>
Die Sitzungsindizierung ist optional und wird asynchron ausgeführt. Ergebnisse können geringfügig veraltet sein. Sitzungsprotokolle befinden sich auf dem Datenträger; betrachten Sie daher den Dateisystemzugriff als Vertrauensgrenze.
</Warning>

Die gewöhnliche, vom Modell aufgerufene Suche in Sitzungstranskripten richtet sich nach
[`tools.sessions.visibility`](/de/gateway/config-tools#toolssessions). Die standardmäßige
Sichtbarkeit `tree` umfasst die aktuelle Sitzung, von ihr gestartete Sitzungen und
Gruppensitzungen desselben Agenten, die über die implizite Gruppenwahrnehmung beobachtet werden. Andere,
nicht zusammenhängende Sitzungen erfordern die Sichtbarkeit `agent` (oder `all` nur, wenn auch
agentenübergreifender Abruf erforderlich ist und die Agent-zu-Agent-Richtlinie dies zulässt).

`rememberAcrossConversations` erweitert diese Einstellung nicht. Es stellt eine
separate, nur zur Laufzeit gültige Autorisierung bereit, die während des begrenzten
Active-Memory-Durchlaufs auf private Transkripte desselben Agenten beschränkt ist.

Die folgenden Beispiele platzieren diese Einstellungen unter `memory.search` auf oberster Ebene. Sie können
entsprechende Einstellungen auch in einer agentenspezifischen Überschreibung `memory.search` anwenden, wenn nur ein
Agent Sitzungstranskripte indizieren und durchsuchen soll.

Für den Abruf vom Gateway in Direktnachrichten durch denselben Agenten:

<Tabs>
  <Tab title="Integriertes Backend">
    ```json5
    {
      memory: {
        search: {
          experimental: { sessionMemory: true },
          sources: ["memory", "sessions"],
        },
      },
      tools: {
        sessions: { visibility: "agent" },
      },
    }
    ```
  </Tab>
  <Tab title="QMD-Backend">
    ```json5
    {
      memory: {
        backend: "qmd",
        search: {
          experimental: { sessionMemory: true },
          sources: ["memory", "sessions"],
        },
        qmd: {
          sessions: { enabled: true },
        },
      },
      tools: {
        sessions: { visibility: "agent" },
      },
    }
    ```
  </Tab>
</Tabs>

Bei Verwendung von QMD exportiert `sources: ["sessions"]` Transkripte nicht von selbst nach QMD. Legen Sie
zusätzlich `memory.qmd.sessions.enabled: true` fest. Die übergeordnete
Einstellung `rememberAcrossConversations: true` bildet die Ausnahme: Sie impliziert den
erforderlichen QMD-Sitzungsexport für diesen Agenten. Implizite Exporte bleiben privat:
Sie verwenden immer den standardmäßigen internen Exportspeicherort (ein konfiguriertes
`sessions.exportDir` gilt nur für explizite Exporte), werden nur
beim konversationsübergreifenden Abruf dieses Agenten durchsucht und können von gewöhnlichem `memory_get`
nicht gelesen werden. Explizites
`memory.qmd.sessions.enabled: true` behält sein bestehendes Verhalten bei und macht
exportierte Transkripte zu einem Teil des gewöhnlichen Speicherkorpus.

---

## SQLite-Vektorbeschleunigung (sqlite-vec)

| Schlüssel                    | Typ       | Standardwert | Beschreibung                           |
| ---------------------------- | --------- | ------------ | -------------------------------------- |
| `store.vector.enabled`       | `boolean` | `true`  | sqlite-vec für Vektorabfragen verwenden |
| `store.vector.extensionPath` | `string`  | mitgeliefert | sqlite-vec-Pfad überschreiben          |

Wenn sqlite-vec nicht verfügbar ist, greift OpenClaw automatisch auf die prozessinterne Kosinusähnlichkeit zurück.

---

## Indexspeicherung

Integrierte Speicherindizes befinden sich in der OpenClaw-SQLite-Datenbank des jeweiligen Agenten unter
`agents/<agentId>/agent/openclaw-agent.sqlite`.

| Schlüssel              | Typ      | Standardwert | Beschreibung                                      |
| ---------------------- | -------- | ------------ | ------------------------------------------------- |
| `store.fts.tokenizer` | `string` | `unicode61` | FTS5-Tokenizer (`unicode61` oder `trigram`) |

---

## QMD-Backend-Konfiguration

Legen Sie zum Aktivieren `memory.backend = "qmd"` fest. Alle QMD-Einstellungen befinden sich unter `memory.qmd`:

| Schlüssel                 | Typ       | Standardwert | Beschreibung                                                                                                   |
| ------------------------- | --------- | ------------ | -------------------------------------------------------------------------------------------------------------- |
| `command`                | `string`  | `qmd`    | Pfad zur ausführbaren QMD-Datei; legen Sie einen absoluten Pfad fest, wenn sich der Dienst-`PATH` von Ihrer Shell unterscheidet |
| `searchMode`             | `string`  | `search` | Suchbefehl: `search`, `vsearch`, `query`                                          |
| `rerank`                 | `boolean` | --       | Mit `searchMode: "query"` und QMD 2.1+ auf `false` setzen, um das QMD-Reranking zu überspringen          |
| `includeDefaultMemory`   | `boolean` | `true`   | `MEMORY.md` + `memory/**/*.md` automatisch indizieren                                             |
| `paths[]`                | `array`   | --       | Zusätzliche Pfade: `{ name, path, pattern? }`                                               |
| `sessions.enabled`       | `boolean` | `false`  | Sitzungstranskripte nach QMD exportieren                                                   |
| `sessions.retentionDays` | `number`  | --       | Aufbewahrung von Transkripten                                                                  |
| `sessions.exportDir`     | `string`  | --       | Exportverzeichnis                                                                      |

`searchMode: "search"` arbeitet ausschließlich lexikalisch bzw. mit BM25. OpenClaw führt für diesen Modus keine semantischen Prüfungen der Vektorbereitschaft oder Wartung von QMD-Einbettungen aus, auch nicht während `memory status --deep`; `vsearch` und `query` erfordern weiterhin die QMD-Vektorbereitschaft und Einbettungen.

`rerank: false` ändert nur den QMD-Modus `query` und erfordert QMD 2.1 oder neuer. Im direkten CLI-Modus übergibt OpenClaw `--no-rerank`; im mcporter-basierten MCP-Modus übergibt es `rerank: false` an das vereinheitlichte Abfragewerkzeug von QMD. Lassen Sie die Einstellung weg, um das standardmäßige QMD-Reranking für Abfragen zu verwenden.

OpenClaw bevorzugt aktuelle QMD-Sammlungs- und MCP-Abfrageformate, unterstützt jedoch weiterhin ältere QMD-Versionen, indem es bei Bedarf kompatible Flags für Sammlungsmuster und ältere MCP-Werkzeugnamen ausprobiert. Wenn QMD die Unterstützung mehrerer Sammlungsfilter angibt, werden Sammlungen derselben Quelle mit einem einzigen QMD-Prozess durchsucht; ältere QMD-Builds behalten den Kompatibilitätspfad pro Sammlung bei. „Dieselbe Quelle“ bedeutet, dass dauerhafte Speichersammlungen (standardmäßige Speicherdateien sowie benutzerdefinierte Pfade) zusammen gruppiert werden, während Sammlungen von Sitzungstranskripten eine separate Gruppe bleiben, sodass der Diversifizierung nach Quellen weiterhin beide Eingaben zur Verfügung stehen.

<Note>
QMD-Modellüberschreibungen verbleiben auf der QMD-Seite und nicht in der OpenClaw-Konfiguration. Wenn Sie die Modelle von QMD global überschreiben müssen, legen Sie Umgebungsvariablen wie `QMD_EMBED_MODEL`, `QMD_RERANK_MODEL` und `QMD_GENERATE_MODEL` in der Laufzeitumgebung des Gateways fest.
</Note>

<AccordionGroup>
  <Accordion title="Grenzwerte">
    | Schlüssel                   | Typ      | Standardwert | Beschreibung                |
    | --------------------------- | -------- | ------------ | --------------------------- |
    | `limits.maxResults`       | `number` | `4`     | Maximale Anzahl von Suchergebnissen |
    | `limits.maxSnippetChars`  | `number` | `450`   | Ausschnittlänge begrenzen       |
    | `limits.maxInjectedChars` | `number` | `2200`  | Gesamtzahl eingefügter Zeichen begrenzen |
    | `limits.timeoutMs`        | `number` | `4000`  | Zeitüberschreitung für QMD-Befehle bei QMD-gestützter Suche, einschließlich `memory_search`; Einrichtung, Synchronisierung, integrierter Rückgriff und ergänzende Arbeiten behalten die standardmäßige Werkzeugfrist bei |
  </Accordion>
  <Accordion title="Geltungsbereich">
    Steuert, welche Sitzungen QMD-Suchergebnisse erhalten können. Dasselbe Schema wie bei [`session.sendPolicy`](/de/gateway/config-agents#session):

    ```json5
    {
      memory: {
        qmd: {
          scope: {
            default: "deny",
            rules: [{ action: "allow", match: { chatType: "direct" } }],
          },
        },
      },
    }
    ```

    Der mitgelieferte Standard ist ausschließlich auf Direktnachrichten bzw. direkte Chats beschränkt und lehnt Gruppen sowie andere Kanaltypen ab. `match.keyPrefix` entspricht dem normalisierten Sitzungsschlüssel; `match.rawKeyPrefix` entspricht dem Rohschlüssel einschließlich `agent:<id>:`.

  </Accordion>
  <Accordion title="Quellenangaben">
    `memory.citations` gilt für alle Backends:

    | Wert               | Verhalten                                            |
    | ------------------ | --------------------------------------------------- |
    | `auto` (Standardwert) | `Source: <path#line>`-Fußzeile in Ausschnitte aufnehmen |
    | `on`             | Fußzeile immer aufnehmen                               |
    | `off`            | Fußzeile weglassen (Pfad wird intern weiterhin an den Agenten übergeben) |

  </Accordion>
</AccordionGroup>

QMD wird verzögert initialisiert, wenn der Speicher erstmals verwendet wird; sein Adapter verwaltet die Zeitpläne für Aktualisierungen und Einbettungen.

### Vollständiges QMD-Beispiel

```json5
{
  memory: {
    backend: "qmd",
    citations: "auto",
    qmd: {
      includeDefaultMemory: true,
      update: { interval: "5m", debounceMs: 15000 },
      limits: { maxResults: 4, timeoutMs: 4000 },
      scope: {
        default: "deny",
        rules: [{ action: "allow", match: { chatType: "direct" } }],
      },
      paths: [{ name: "docs", path: "~/notes", pattern: "**/*.md" }],
    },
  },
}
```

---

## Dreaming

Dreaming wird unter `plugins.entries.memory-core.config.dreaming` konfiguriert, nicht unter `memory.search`.

Dreaming wird als ein geplanter Durchlauf ausgeführt und verwendet interne Leicht-/Tief-/REM-Phasen als Implementierungsdetail.

Informationen zum konzeptionellen Verhalten und zu Slash-Befehlen finden Sie unter [Dreaming](/de/concepts/dreaming).

### Benutzereinstellungen

| Schlüssel                              | Typ       | Standardwert   | Beschreibung                                                                                                                      |
| -------------------------------------- | --------- | --------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `enabled`                              | `boolean` | `false`       | Dreaming vollständig aktivieren oder deaktivieren                                                                                              |
| `frequency`                            | `string`  | `0 3 * * *`   | Optionaler Cron-Zeitplan für den vollständigen Dreaming-Durchlauf                                                                                |
| `model`                                | `string`  | Standardmodell | Optionale Modellüberschreibung für den Dream-Diary-Subagenten                                                                                     |
| `phases.deep.maxPromotedSnippetTokens` | `number`  | `160`         | Maximale Anzahl geschätzter Tokens, die aus jedem in `MEMORY.md` übernommenen Ausschnitt des Kurzzeitabrufs beibehalten werden; Herkunftsmetadaten bleiben sichtbar |

### Beispiel

```json5
{
  plugins: {
    entries: {
      "memory-core": {
        subagent: {
          allowModelOverride: true,
          allowedModels: ["anthropic/claude-sonnet-4-6"],
        },
        config: {
          dreaming: {
            enabled: true,
            frequency: "0 3 * * *",
            model: "anthropic/claude-sonnet-4-6",
          },
        },
      },
    },
  },
}
```

<Note>
- Dreaming schreibt den Maschinenzustand nach `memory/.dreams/`.
- Dreaming schreibt menschenlesbare narrative Ausgaben nach `DREAMS.md` (oder in ein vorhandenes `dreams.md`).
- `dreaming.model` verwendet die vorhandene Vertrauensprüfung des Plugins für Subagenten; legen Sie vor der Aktivierung `plugins.entries.memory-core.subagent.allowModelOverride: true` fest.
- Dream Diary versucht es einmal erneut mit dem Standardmodell der Sitzung, wenn das konfigurierte Modell nicht verfügbar ist. Fehler bei der Vertrauensprüfung oder der Zulassungsliste werden protokolliert und nicht stillschweigend erneut versucht.
- Die Richtlinie und Schwellenwerte der Leicht-/Tief-/REM-Phasen sind internes Verhalten und keine benutzerseitige Konfiguration.

</Note>

## Verwandte Themen

- [Konfigurationsreferenz](/de/gateway/configuration-reference)
- [Speicherübersicht](/de/concepts/memory)
- [Speichersuche](/de/concepts/memory-search)
