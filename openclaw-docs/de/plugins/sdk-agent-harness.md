---
read_when:
    - Sie ändern die eingebettete Agentenlaufzeit oder die Harness-Registry
    - Sie registrieren ein Agent-Harness aus einem gebündelten oder vertrauenswürdigen Plugin.
    - Sie müssen verstehen, wie das Codex-Plugin mit Modell-Providern zusammenhängt
sidebarTitle: Agent Harness
summary: Experimentelle SDK-Schnittstelle für Plugins, die den eingebetteten Low-Level-Agent-Ausführer ersetzen
title: Agent-Harness-Plugins
x-i18n:
    generated_at: "2026-07-26T19:10:23Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4ff4e41a46ba0074fc6c8bf46da813b58d074f5e6c5c1d236d7ab78e824bdc02
    source_path: plugins/sdk-agent-harness.md
    workflow: 16
---

Ein **Agent-Harness** ist der systemnahe Executor für einen vorbereiteten OpenClaw-Agent-
Turn. Es ist weder ein Modell-Provider noch ein Kanal oder eine Tool-Registry. Das
benutzerorientierte mentale Modell finden Sie unter [Agent-Runtimes](/de/concepts/agent-runtimes).

Verwenden Sie diese Schnittstelle nur für gebündelte oder vertrauenswürdige native Plugins. Der Vertrag ist
weiterhin experimentell, da die Parametertypen bewusst den
aktuellen eingebetteten Runner abbilden.

## Wann ein Harness verwendet werden sollte

Registrieren Sie ein Agent-Harness, wenn eine Modellfamilie über eine eigene native Session-
Runtime verfügt und der normale OpenClaw-Provider-Transport die falsche Abstraktion darstellt:

- ein nativer Coding-Agent-Server, der Threads und Compaction verwaltet
- eine lokale CLI oder ein Daemon, die bzw. der native Planungs-, Reasoning- und Tool-Ereignisse streamen muss
- eine Modell-Runtime, die zusätzlich zum OpenClaw-
  Session-Transkript eine eigene Fortsetzungs-ID benötigt

Registrieren Sie **kein** Harness, nur um eine neue LLM-API hinzuzufügen. Erstellen Sie für normale HTTP- oder
WebSocket-Modell-APIs ein [Provider-Plugin](/de/plugins/sdk-provider-plugins).

## Wofür Core weiterhin zuständig ist

Bevor ein Harness ausgewählt wird, hat OpenClaw bereits Folgendes aufgelöst:

- Provider und Modell
- Runtime-Authentifizierungsstatus, sofern das Harness nicht angibt, den Authentifizierungs-Bootstrap selbst zu verwalten
- Thinking-Level und Kontextbudget
- die OpenClaw-Transkript-/Session-Datei
- Workspace-, Sandbox- und Tool-Richtlinie
- Callbacks für Kanalantworten und Streaming
- Richtlinie für Modell-Fallback und Live-Modellwechsel

Ein Harness führt einen vorbereiteten Versuch aus; es wählt keine Provider aus, ersetzt nicht die Kanal-
Zustellung und wechselt nicht stillschweigend das Modell.

### Harness-eigener Authentifizierungs-Bootstrap

Standardmäßig löst Core die Provider-Anmeldedaten auf, bevor ein Harness aufgerufen wird. Ein
vertrauenswürdiges Harness, das sich über seine eigene native Runtime authentifizieren kann, darf
`authBootstrap: "harness"` in seiner statischen `AgentHarness`-Registrierung festlegen. Core überspringt dann
seinen generischen Bootstrap der Provider-Anmeldedaten sowie den Fehler bei fehlenden Anmeldedaten
für jeden von diesem Harness übernommenen Versuch.

Core leitet weiterhin ein kompatibles, explizit ausgewähltes oder geordnetes OpenClaw-Authentifizierungs-
profil und dessen bereichsgebundenen Store weiter, sofern eines vorhanden ist. Das Harness muss dieses
Profil oder seine nativen Anmeldedaten auflösen, bevor es Modellanfragen sendet, Secrets
auf den Versuch beschränken und aussagekräftige Authentifizierungsfehler melden. Legen Sie
diese Fähigkeit nicht für ein Harness fest, das die Authentifizierung nur gelegentlich verwaltet.

### Verifizierte Runtime-Artefakte der Einrichtung

Ein lokales Harness, das Inferenz für die Ersteinrichtung bereitstellen kann, muss die
Implementierung bestätigen, die den Probevorgang abgeschlossen hat. Wenn
`params.captureRuntimeArtifact` den Wert „true“ hat, geben Sie ein opakes
`result.runtimeArtifact` mit einer stabilen ID und einem Inhaltsfingerabdruck zurück. Registrieren Sie eine
entsprechende `runtimeArtifact.validate(...)`-Fähigkeit, die diese Bindung erneut prüft,
ohne ein anderes Harness zu laden oder nicht zugehörige Plugins zu durchsuchen.

Verifizierte OpenClaw-Fortsetzungen übergeben außerdem `params.expectedRuntimeArtifact`.
Das Harness muss diesen Wert mit dem exakt erworbenen nativen Prozess vergleichen und einen Fehler auslösen,
bevor es einen nativen Thread startet oder fortsetzt, wenn sie nicht übereinstimmen. Gewöhnliche Agent-
Turns lassen beide Felder aus, damit Inhalts-Hashing nicht in den normalen Hot Path für Anfragen gelangt.
Remote-/WebSocket-Harnesses benötigen einen Server-Bestätigungsvertrag, bevor
sie teilnehmen können; eine Versionszeichenfolge allein ist keine Artefaktidentität.

Der vorbereitete Versuch umfasst außerdem `params.runtimePlan`, ein OpenClaw-eigenes
Richtlinienpaket für Runtime-Entscheidungen, die über OpenClaw und
native Harnesses hinweg einheitlich bleiben müssen:

- `runtimePlan.tools.normalize(...)` und `runtimePlan.tools.logDiagnostics(...)`
  für Provider-spezifische Tool-Schema-Richtlinien
- `runtimePlan.transcript.resolvePolicy(...)` für Transkriptbereinigung und
  Richtlinien zur Reparatur von Tool-Aufrufen
- `runtimePlan.delivery.isSilentPayload(...)` für gemeinsame `NO_REPLY` und die Unterdrückung der Medien-
  zustellung
- `runtimePlan.outcome.classifyRunResult(...)` für die Klassifizierung von Modell-Fallbacks
- `runtimePlan.observability` für aufgelöste Provider-/Modell-/Harness-Metadaten

Harnesses dürfen den Plan für Entscheidungen verwenden, die mit dem Verhalten von OpenClaw
übereinstimmen müssen, sollten ihn jedoch als host-eigenen Versuchszustand behandeln: Ändern Sie ihn nicht und verwenden
Sie ihn nicht, um innerhalb eines Turns Provider oder Modelle zu wechseln.

### Vertrag für den Anfrage-Transport

`supports(ctx)` empfängt den aufgelösten Modelltransport in `ctx.modelProvider`.
Zwei Secret-freie, Provider-eigene Fakten beschreiben die ausgewählte Route:

- `runtimePolicy.compatibleIds` listet die Runtime-IDs auf, die der Provider
  als mit dieser konkreten Route kompatibel deklariert. Eine fehlende Richtlinie bedeutet, dass der Provider
  keine Kompatibilität auf Routenebene deklariert hat; sie ist keine Berechtigung, Unterstützung anzunehmen.
- `requestTransportOverrides: "none"` bedeutet, dass keine vom Autor festgelegte Provider-/Modellanfrage-
  Überschreibung reproduziert werden muss. `"present"` bedeutet, dass vom Autor festgelegte Header, Authentifizierungs-
  transport-, Proxy-, TLS-, lokaler Dienst-, privates Netzwerk-Verhalten oder Anfrage-
  parameter vorhanden sind. Das Faktum legt diese Werte nicht offen.

Geben Sie `{ supported: false, reason }` zurück, wenn das Harness den
vorbereiteten Transport nicht reproduzieren kann. Leiten Sie die Unterstützung nach der Auswahl nicht durch Lesen der Rohkonfiguration ab.
Wenn die Authentifizierungsvorbereitung mehrere Wiederholungsrouten ergibt, muss ein Harness
alle unterstützen, bevor die Weiterleitung erfolgt. Bei impliziter Auswahl wird OpenClaw verwendet, wenn kein Plugin
den vollständigen Satz verwalten kann; eine explizite oder persistierte Plugin-Auswahl schlägt geschlossen fehl.

## Ein Harness registrieren

**Import:** `openclaw/plugin-sdk/agent-harness`

```typescript
import type { AgentHarness } from "openclaw/plugin-sdk/agent-harness";
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

const myHarness: AgentHarness = {
  id: "my-harness",
  label: "Mein natives Agent-Harness",

  supports(ctx) {
    const routeSupportsHarness =
      ctx.modelProvider?.runtimePolicy?.compatibleIds.includes("my-harness") === true;
    const canReproduceRequest = ctx.modelProvider?.requestTransportOverrides !== "present";
    return ctx.provider === "my-provider" && routeSupportsHarness && canReproduceRequest
      ? { supported: true, priority: 100 }
      : { supported: false, reason: "Die effektive Route ist nicht Harness-kompatibel" };
  },

  async runAttempt(params) {
    // Starten Sie Ihren nativen Thread oder setzen Sie ihn fort.
    // Verwenden Sie params.prompt, params.tools, params.images, params.onPartialReply,
    // params.onAgentEvent und die anderen Felder des vorbereiteten Versuchs.
    return await runMyNativeTurn(params);
  },
};

export default definePluginEntry({
  id: "my-native-agent",
  name: "Mein nativer Agent",
  description: "Führt ausgewählte Modelle über einen nativen Agent-Daemon aus.",
  register(api) {
    api.registerAgentHarness(myHarness);
  },
});
```

`authBootstrap` fehlt in diesem generischen Beispiel bewusst. Fügen Sie
`authBootstrap: "harness"` nur hinzu, wenn das Harness den oben beschriebenen Vertrag erfüllt.

### Delegierte Ausführung

Ein Harness-Eigentümer darf `delegatedExecutionPluginIds` auf die IDs vertrauenswürdiger
Plugins setzen, die eine vorhandene modellgebundene Session ausführen müssen, beispielsweise ein Sprach-
transport, der eine Codex-gestützte Unterhaltung fortsetzt. Dies ist eine statische Zustimmung des Eigentümers
und keine Core-Zulassungsliste. Halten Sie sie eng begrenzt.

Delegierte erhalten nur die Arbeitsannahme und die eingebettete Ausführung. OpenClaw verlangt
den exakt gespeicherten Session-Schlüssel, Store-Pfad und die Session-ID; `modelSelectionLocked:
true`; sowie übereinstimmende Werte für `agentHarnessId` und `agentHarnessRuntimeOverride`.
Die Ausführung wird anschließend über den Harness-Eigentümer begrenzt. Session-Erstellung, Patching,
Zurücksetzen, Löschen, Archivieren und Gateway-Mutationen bleiben ausschließlich dem Eigentümer vorbehalten.

## Auswahlrichtlinie

OpenClaw wählt nach der Auflösung von Provider und Modell ein Harness aus:

1. Die modellspezifische Runtime-Richtlinie hat Vorrang.
2. Danach folgt die Provider-spezifische Runtime-Richtlinie.
3. `auto` fragt registrierte Harnesses, ob sie die aufgelöste effektive
   Route unterstützen. Provider-/Modellpräfixe allein wählen niemals ein Harness aus.
4. Wenn kein registriertes Harness übereinstimmt, verwendet OpenClaw seine eingebettete Runtime.

Fehler von Plugin-Harnesses werden als Ausführungsfehler gemeldet. Im Modus `auto`
gilt der eingebettete Fallback nur, wenn kein registriertes Plugin-Harness den aufgelösten
Provider bzw. das Modell unterstützt. Sobald ein Plugin-Harness eine Ausführung übernommen hat, spielt OpenClaw
denselben Turn nicht über eine andere Runtime erneut ab, da dies
Authentifizierungs-/Runtime-Semantik ändern oder Nebenwirkungen duplizieren kann.

Die konfigurierte Runtime-Richtlinie bleibt maßgeblich für die gewünschte Runtime. Ein
persistiertes Session-`agentHarnessId` behält die Zuständigkeit für sein natives Transkript,
während die Routen-/Authentifizierungsvorbereitung noch aussteht. Beides macht eine inkompatible
Route nicht kompatibel: Sobald vorbereitete Fakten vorliegen, muss das ausgewählte oder angeheftete Harness
sie unterstützen, andernfalls schlägt die Ausführung geschlossen fehl. `/status` zeigt die effektive Runtime,
die anhand von Richtlinie, persistierter Zuständigkeit und Routenunterstützung ausgewählt wurde.
Der Vorbereitungsstatus ist explizit: Ein fehlendes `runtimePolicy` bleibt undeklariert,
anstatt aus zufällig vorhandenen Transportfeldern abgeleitet zu werden.
Wenn eine Harness-eigene Authentifizierung mehrere physische Routen unaufgelöst lässt, ist das
vorbereitete Unterstützungsfaktum die Schnittmenge ihrer kompatiblen Runtime-IDs und
meldet Anfrageüberschreibungen, falls ein Kandidat solche besitzt. Ein nicht deklarierter Kandidat
macht die native Kompatibilität daher leer; `preparedAuth.source: "harness"`
ist ein Authentifizierungseigentümer und keine Berechtigung, Routenunterstützung abzuleiten.

Wenn das ausgewählte Harness unerwartet ist, aktivieren Sie das Debug-Logging `agents/harness`
und prüfen Sie den strukturierten `agent harness selected`-Datensatz des Gateways: Er
enthält die ID des ausgewählten Harnesses, den Auswahlgrund, die Runtime-/Fallback-Richtlinie
und im Modus `auto` das Unterstützungsergebnis jedes Plugin-Kandidaten.

Das gebündelte Codex-Plugin registriert `codex` als seine Harness-ID. Core behandelt diese
wie eine gewöhnliche Plugin-Harness-ID; Codex-spezifische Aliasse gehören in das Plugin
oder die Betreiberkonfiguration und nicht in den gemeinsamen Runtime-Selektor.

## Kopplung von Provider und Harness

Die meisten Harnesses sollten außerdem einen Provider registrieren. Der Provider macht Modellreferenzen,
Authentifizierungsstatus, Modellmetadaten und die Auswahl `/model` für den Rest von
OpenClaw sichtbar. Das Harness übernimmt diesen Provider anschließend in `supports(...)`.

Das gebündelte Codex-Plugin folgt diesem Muster:

- bevorzugte Benutzer-Modellreferenzen: `openai/gpt-5.6-sol`
- Kompatibilitätsreferenzen: Veraltete `codex/gpt-*`-Referenzen werden weiterhin akzeptiert, neue
  Konfigurationen sollten sie jedoch nicht als normale Provider-/Modellreferenzen verwenden
- Harness-ID: `codex`
- Authentifizierung: synthetische Provider-Verfügbarkeit, da das Codex-Harness die
  native Codex-Anmeldung/-Session verwaltet
- App-Server-Anfrage: OpenClaw sendet die reine Modell-ID an Codex und lässt das
  Harness mit dem nativen App-Server-Protokoll kommunizieren

Das Codex-Plugin ist additiv. Wenn die Runtime-Richtlinie nicht festgelegt oder `auto` ist, darf OpenAI
Codex nur auswählen, wenn sein Provider-eigener Routenvertrag `codex`
als kompatibel deklariert: eine exakte offizielle HTTPS-Route für Platform Responses oder ChatGPT Responses
ohne vom Autor festgelegte Anfrageüberschreibung. Das Präfix `openai/*` allein
wählt Codex niemals aus. Benutzerdefinierte Endpunkte, Completions-Adapter und vom Autor festgelegtes Anfrage-
verhalten verbleiben bei OpenClaw. Offizielle Klartext-HTTP-Endpunkte werden abgelehnt. Ältere `codex/gpt-*`-
Referenzen bleiben Kompatibilitätseingaben. Siehe
[Implizite OpenAI-Agent-Runtime](/de/providers/openai#implicit-agent-runtime).

Informationen zur Einrichtung durch Betreiber, Beispiele für Modellpräfixe und reine Codex-Konfigurationen finden Sie unter
[Codex-Harness](/de/plugins/codex-harness).

Das Codex-Plugin erzwingt die unter
[Codex-Harness](/de/plugins/codex-harness) dokumentierte minimale App-Server-Version. Es prüft den Initialisierungs-Handshake und
blockiert ältere oder nicht versionierte Server, damit OpenClaw nur mit der getesteten
Protokollschnittstelle arbeitet.

### Tool-Ergebnis-Middleware

Gebündelte Plugins und explizit aktivierte installierte Plugins mit passenden
Manifestverträgen können über
`api.registerAgentToolResultMiddleware(...)` Runtime-neutrale Tool-Ergebnis-Middleware anhängen, wenn ihr Manifest die
anvisierten Runtime-IDs in `contracts.agentToolResultMiddleware` deklariert. Diese vertrauenswürdige
Schnittstelle ist für asynchrone Transformationen von Tool-Ergebnissen vorgesehen, die ausgeführt werden müssen, bevor OpenClaw oder
Codex Tool-Ausgaben wieder an das Modell übergibt.

Gebündelte Legacy-Plugins können weiterhin
`api.registerCodexAppServerExtensionFactory(...)` für Middleware verwenden, die ausschließlich für den Codex-App-Server
bestimmt ist, neue Ergebnistransformationen sollten jedoch die laufzeitneutrale API verwenden. Der
ausschließlich für den eingebetteten Runner bestimmte Hook `api.registerEmbeddedExtensionFactory(...)` wurde
entfernt; Transformationen eingebetteter Werkzeugergebnisse müssen laufzeitneutrale Middleware verwenden.

### Klassifizierung des terminalen Ergebnisses

Native Harnesses, die ihre eigene Protokollprojektion verwalten, können
`classifyAgentHarnessTerminalOutcome(...)` aus
`openclaw/plugin-sdk/agent-harness-runtime` verwenden, wenn ein abgeschlossener Turn keinen
sichtbaren Assistententext erzeugt hat. Der Helfer gibt `empty`, `reasoning-only` oder
`planning-only` zurück, damit die Fallback-Richtlinie von OpenClaw entscheiden kann, ob ein erneuter Versuch mit einem
anderen Modell erfolgen soll. `planning-only` erfordert das explizite Feld `planText`
des Harnesses; OpenClaw leitet es nicht aus Assistentenprosa ab. Der Helfer
lässt Prompt-Fehler, laufende Turns und absichtlich stille
Antworten wie `NO_REPLY` bewusst unklassifiziert.

### Nebeneffekte am Agentenende

Native Harnesses müssen `runAgentEndSideEffects(...)` aus
`openclaw/plugin-sdk/agent-harness-runtime` aufrufen, nachdem sie einen Versuch abgeschlossen haben. Die Funktion
löst den portablen Hook `agent_end` und die Forschungserfassung von OpenClaw aus,
ohne interaktive Antworten zu verzögern. Verwenden Sie `awaitAgentEndSideEffects(...)` für
lokale, nicht interaktive Ausführungen, bei denen der Versuch erst aufgelöst werden darf, nachdem diese
Nebeneffekte abgeschlossen sind. Beide Helfer akzeptieren dieselbe `{ event, ctx }`-Nutzlast wie
`runAgentHarnessAgentEndHook(...)`; ihre Fehler verändern das Ergebnis des abgeschlossenen
Versuchs nicht.

### Benutzereingabe- und Werkzeugoberflächen

Native Harnesses, die eine Benutzereingabeanforderung auf Laufzeitebene bereitstellen, sollten die
Benutzereingabe-Helfer aus `openclaw/plugin-sdk/agent-harness-runtime` verwenden, um
den Prompt zu formatieren, ihn über den blockierenden Antwortpfad von OpenClaw zuzustellen und
Auswahlantworten beziehungsweise Freitextantworten wieder in die native Antwortstruktur der Laufzeit zu normalisieren. Der
Helfer sorgt für eine konsistente Darstellung in Kanälen und der TUI, während jedes Harness seine
eigene Protokollanalyse und den Lebenszyklus ausstehender Anforderungen verwaltet.

Native Harnesses, die eine kompakte PI-ähnliche Werkzeugweiterleitung benötigen, sollten
`createAgentHarnessToolSurfaceRuntime(...)` aus
`openclaw/plugin-sdk/agent-harness-tool-runtime` verwenden. Die Funktion verwaltet
die Auswahl der Steuerung für Werkzeugsuche und Codemodus, schlanke Standardwerte für lokale Modelle,
laufzeitkompatible Schemafilterung, die Ausführung des verborgenen Katalogs, die
Verzeichnis-Hydratisierung und die Katalogbereinigung. Harnesses sind weiterhin für ihre SDK-spezifische
Werkzeugkonvertierung und den nativen Ausführungs-Callback verantwortlich.

### Nativer Codex-Harness-Modus

Das gebündelte Harness `codex` ist der native Codex-Modus für eingebettete OpenClaw-
Agenten-Turns. Aktivieren Sie zuerst das gebündelte Plugin `codex` und nehmen Sie `codex` in
`plugins.allow` auf, wenn Ihre Konfiguration eine restriktive Zulassungsliste verwendet. Native App-Server-
Konfigurationen sollten `openai/gpt-*` verwenden; OpenAI-Agenten-Turns wählen das Codex-Harness
nur aus, wenn die effektive Route Codex-Kompatibilität deklariert. Legacy-Codex-Modell-
Referenzen sollten mit `openclaw doctor --fix` repariert werden, und Legacy-Modellreferenzen vom Typ `codex/*`
bleiben Kompatibilitätsaliase für das native Harness.

Wenn dieser Modus ausgeführt wird, verwaltet Codex die native Thread-ID, das Fortsetzungsverhalten,
Compaction und die App-Server-Ausführung. OpenClaw verwaltet weiterhin den Chatkanal,
die sichtbare Transkriptspiegelung, die Werkzeugrichtlinie, Genehmigungen, die Medienzustellung und die Sitzungs-
auswahl. Verwenden Sie Provider/Modell `agentRuntime.id: "codex"`, wenn Sie
nachweisen müssen, dass ausschließlich der Codex-App-Server-Pfad die Ausführung übernehmen kann. Explizite Plugin-
Laufzeiten schlagen geschlossen fehl; Auswahlfehler und Laufzeitfehler des Codex-App-Servers
werden nicht über eine andere Laufzeit erneut versucht.

## Laufzeitstrenge

Standardmäßig verwendet OpenClaw die Provider/Modell-Laufzeitrichtlinie `auto`: Registrierte
Plugin-Harnesses können kompatible effektive Routen übernehmen, und die eingebettete
Laufzeit verarbeitet den Turn, wenn kein Harness übereinstimmt. Ein Provider/Modell-Präfix allein
wählt niemals ein Harness aus. Verwenden Sie eine explizite Provider/Modell-Plugin-Laufzeit wie
`agentRuntime.id: "codex"`, wenn eine fehlende Harness-Auswahl fehlschlagen soll,
anstatt über die eingebettete Laufzeit weitergeleitet zu werden. Eine explizite Auswahl macht eine
inkompatible Route nicht kompatibel. Fehler ausgewählter Plugin-Harnesses führen immer
zu einem harten Fehlschlag. Dies blockiert kein explizites Provider/Modell-
`agentRuntime.id: "openclaw"`.

Für ausschließlich Codex verwendende eingebettete Ausführungen:

```json
{
  "models": {
    "providers": {
      "openai": {
        "agentRuntime": {
          "id": "codex"
        }
      }
    }
  },
  "agents": {
    "defaults": {
      "model": "openai/gpt-5.6-sol"
    }
  }
}
```

Wenn Sie ein CLI-Backend für ein kanonisches Modell wünschen, legen Sie die Laufzeit in diesem
Modelleintrag ab:

```json
{
  "agents": {
    "defaults": {
      "model": "anthropic/claude-opus-5",
      "models": {
        "anthropic/claude-opus-5": {
          "agentRuntime": {
            "id": "claude-cli"
          }
        }
      }
    }
  }
}
```

Agentenspezifische Überschreibungen verwenden dieselbe modellbezogene Struktur:

```json
{
  "agents": {
    "list": [
      {
        "id": "codex-only",
        "model": "openai/gpt-5.6-sol",
        "models": {
          "openai/gpt-5.6-sol": {
            "agentRuntime": { "id": "codex" }
          }
        }
      }
    ]
  }
}
```

Legacy-Beispiele für eine Laufzeit auf Ebene des gesamten Agenten wie dieses werden ignoriert:

```json
{
  "agents": {
    "defaults": {
      "agentRuntime": {
        "id": "codex"
      }
    }
  }
}
```

Bei einer expliziten Plugin-Laufzeit schlägt eine Sitzung frühzeitig fehl, wenn das angeforderte
Harness nicht registriert ist, den aufgelösten Provider beziehungsweise das aufgelöste Modell nicht unterstützt oder
fehlschlägt, bevor Turn-Nebeneffekte erzeugt werden. Dies ist für reine Codex-
Bereitstellungen und für Live-Tests beabsichtigt, die nachweisen müssen, dass der Codex-App-Server-Pfad
tatsächlich verwendet wird.

Diese Einstellung steuert nur das eingebettete Agenten-Harness. Sie deaktiviert nicht
die Provider-spezifische Modellweiterleitung für Bilder, Videos, Musik, TTS, PDF oder andere Medien.

## Native Sitzungen und Transkriptspiegelung

Ein Harness kann eine native Sitzungs-ID, Thread-ID oder ein daemonseitiges Fortsetzungs-
Token beibehalten. Halten Sie diese Bindung ausdrücklich mit der OpenClaw-Sitzung verknüpft und
spiegeln Sie weiterhin für Benutzer sichtbare Assistenten- und Werkzeugausgaben in das OpenClaw-
Transkript.

Das OpenClaw-Transkript bleibt die Kompatibilitätsschicht für:

- im Kanal sichtbaren Sitzungsverlauf
- Transkriptsuche und -indizierung
- den Wechsel zurück zum integrierten OpenClaw-Harness bei einem späteren Turn
- generisches Verhalten von `/new`, `/reset` und beim Löschen von Sitzungen

Wenn Ihr Harness eine Sidecar-Bindung speichert, implementieren Sie `reset(...)`, damit OpenClaw
sie löschen kann, wenn die zugehörige OpenClaw-Sitzung zurückgesetzt wird.

## Werkzeug- und Medienergebnisse

Der Core erstellt die OpenClaw-Werkzeugliste und übergibt sie an den vorbereiteten
Versuch. Wenn ein Harness einen dynamischen Werkzeugaufruf ausführt, geben Sie das Werkzeugergebnis
über die Ergebnisstruktur des Harnesses zurück, anstatt selbst Kanalmedien
zu senden.

Dadurch verwenden Text-, Bild-, Video-, Musik-, TTS-, Genehmigungs- und Messaging-Werkzeug-
Ausgaben denselben Zustellungspfad wie von OpenClaw unterstützte Ausführungen.

Setzen Sie `AgentHarnessAttemptResult.hostOwnedToolMediaUrls` nur für native Artefakte,
die die vertrauenswürdige Harness-Laufzeit selbst erstellt und dauerhaft gespeichert hat. Jeder Eintrag muss
auch in `toolMediaUrls` enthalten sein. Nehmen Sie niemals vom Modell ausgewählte Medien dynamischer Werkzeuge oder
OpenClaw-Werkzeugmedien auf. Auf `message_tool_only`-Routen ermöglicht diese enge Herkunftszuordnung,
dass Artefakte nativer Laufzeiten die Unterdrückung der Quellantwort überstehen; die normale Senderichtlinie
und die Zulassung für Umgebungsräume gelten weiterhin.

### Terminale Werkzeugergebnisse

`AgentHarnessAttemptParams.observeToolTerminal` ist der vom Host verwaltete Akkumulator für terminale
Ergebnisse. Ein Harness, das dynamische OpenClaw-Werkzeuge oder native
Werkzeuge ausführt, muss ihn aufrufen, sobald jedes Werkzeug genau ein terminales Ergebnis erreicht, bevor das
Versuchsergebnis abgeschlossen wird. Harnesses, die keine Werkzeuge ausführen, müssen ihn nicht
aufrufen.

Melden Sie Fakten von der Ausführungsgrenze:

- Übergeben Sie die Protokollaufruf-ID, sofern vorhanden, den kanonischen Werkzeugnamen und die
  Argumente, die nach der Vorbereitung oder Umschreibung durch Hooks tatsächlich beim Werkzeug ankamen.
- Setzen Sie `executionStarted: false`, wenn Validierung, Genehmigung oder eine andere Schutzmaßnahme
  den Aufruf gestoppt hat, bevor die Werkzeugimplementierung begann. Sobald eine Weiterleitung
  stattgefunden haben könnte, melden Sie konservativ `true`.
- Melden Sie `outcome: "success"` oder `outcome: "failure"`. Nehmen Sie die strukturierten
  Fehlerfelder auf, die von der Laufzeit verfügbar sind, anstatt einen Fehler aus dem
  Anzeigetext abzuleiten.
- Verwenden Sie `nativeMutation` nur für native Werkzeuge, die keine OpenClaw-Werkzeug-
  definition verwenden. Geben Sie dort protokolleigene Mutations- und Wiedergabefakten an; kopieren Sie
  den Mutationsklassifizierer von OpenClaw nicht in das Harness.

Der Callback gibt die kanonische Auflösung für diesen Aufruf zurück. Übernehmen Sie dessen
`lastToolError` in `AgentHarnessAttemptResult` und verwenden Sie dessen Ausführungs-,
Argument- und Nebeneffektfakten in der Harness-Projektion, anstatt
parallelen Zustand abzuleiten. Der Host bewahrt einen nicht aufgelösten mutierenden Fehler über nicht zusammenhängende
erfolgreiche Werkzeuge hinweg auf und löscht ihn erst, nachdem die entsprechende Aktion erfolgreich ist.

Der Callback bleibt aus Gründen der Quellkompatibilität mit älteren experimentellen
Harnesses optional. Optional bedeutet für ein Harness, das Werkzeuge ausführt, nicht, dass er ignoriert werden darf:
Ohne terminale Meldungen kann OpenClaw die Wahrheit eines Fehlers mutierender Werkzeuge
über spätere Werkzeugaufrufe hinweg nicht bewahren, einschließlich des stillen Abschlusses eines Heartbeat.

### Finalisierung abgeschlossener Werkzeuge

OpenClaw benötigt möglicherweise eine letzte sichtbare Antwort, nachdem ein Harness jeden
Werkzeugaufruf abgeschlossen hat, sein nativer Turn jedoch ohne Assistententext endete. Ein Harness kann
diese Wiederherstellung durch Implementierung von `finalizeSettledTurn({ attempt,
settledAttempt })` aktivieren.

Der Callback ist eine separate Fähigkeit und kein weiterer gewöhnlicher Versuch. Er muss:

- entweder das exakt eingeschränkte native Transkript oder ein vollständiges Anwendungs-
  transkript verwenden, das bis einschließlich der Grenze des abgeschlossenen Werkzeugergebnisses eingefroren ist;
- keine Werkzeuge, Funktionen zum Erteilen von Berechtigungen oder für Benutzereingaben, native Ausführungs-
  Hooks, Agenten, Skills, Speicher, Zeitplanung, Erweiterungen oder Fernsteuerung bereitstellen;
- ausschließlich den vom Host bereitgestellten Finalisierungs-Prompt senden; und
- geschlossen fehlschlagen, wenn die ausgewählte Transkript-/Isolationsstrategie
  diese Einschränkungen nicht durchsetzen kann.

OpenClaw ruft den Callback einmal als terminale Unteroperation außerhalb des
gewöhnlichen Versuchs- und Wiederholungsablaufs auf. Ein Fehler beendet die Ausführung mit der
nebeneffektbewussten Warnung zu einem unvollständigen Turn; er kann nicht in gewöhnliche
Authentifizierungs-/Profilrotation, Modell-Fallback, Kontextwiederherstellung, Compaction-
Fortsetzung oder von Hooks angeforderte Überarbeitungspfade eintreten. Die Finalisierung überspringt außerdem die Plugin-
Prompt-Mutation sowie die Hooks `before_agent_run`, LLM-Eingabe/-Ausgabe, terminale Überarbeitung und
`agent_end`. Die Core-Diagnose zeichnet die Operation und ihren Fehler weiterhin auf.

Der Callback gibt `AgentHarnessSettledTurnFinalizationResult` zurück, kein
gewöhnliches Versuchsergebnis. Seine öffentlichen Felder sind auf die abgeschlossene
Assistentennachricht, die Nutzung des Finalisierungsaufrufs, Metadaten zur Transkriptzuständigkeit und
die Diagnosespur beschränkt. Werkzeug-, Zustellungs-, Medien-, Erzeugungs-, Lebenszyklus-, Wiedergabe-, Sitzungs- und
Fallback-Zustand können diese Ergebnisgrenze nicht überschreiten. Unbekannte Felder und Assistenten-
Werkzeugaufrufe schlagen geschlossen fehl.

Ein Harness, das intern seine vollständige Versuchs-Engine wiederverwendet, kann vor der Rückgabe
`projectSettledTurnFinalizationAttemptResult(...)` aufrufen. Der Helfer
weist kanonische Fehler-, Werkzeug-, Zustellungs-, Wiedergabe- und Lebenszyklusnachweise zurück und
projiziert anschließend nur das enge Ergebnis. Dies dient als gestaffelte Absicherung nach der nativen Isolation
und ersetzt nicht das Entfernen der nativen Fähigkeitsoberfläche.

Ein projektionsgestütztes Harness muss den vollständigen Kontext in
`settledAttempt.settledTurnFinalizationContext` mit
`source: "openclaw-transcript"` ablegen. Es muss den aktiven Zweig erfassen, nachdem der
abgeschlossene Turn gespiegelt wurde, nachweisen, dass der aktuelle Prompt und jeder aktuelle Werkzeug-
aufruf sowie jedes Ergebnis bis zu dieser Grenze vorhanden sind, und das resultierende Nachrichten-
Array vor der Rückgabe des Versuchs einfrieren. Der Finalisierer muss einen fehlenden,
nicht unterstützten, mehrdeutigen oder übergroßen Kontext zurückweisen. Er darf Nachrichten nicht kürzen,
frühere Verlaufsdaten nicht verwerfen und dieses Anwendungstranskript nicht als exakten nativen
Verlauf bezeichnen. Harnesses, die eine einzelne eingeschränkte native Sitzung fortsetzen, benötigen dieses
Projektionsfeld nicht.

Implementieren Sie diesen Callback nicht, indem Sie `runAttempt` mit einem Best-Effort-
Hinweis `disableTools` aufrufen. Der Harness-Eigentümer muss die vollständige native
Fähigkeitsgrenze durchsetzen. OpenClaw stellt keinen generischen Fallback bereit, da es
nicht bestätigen kann, dass eine beliebige native Laufzeit diese Einschränkungen eingehalten hat.

Der Callback bleibt für die Kompatibilität mit experimentellen Harnesses von Drittanbietern optional. Wenn der ausgewählte Harness ihn nicht bereitstellt, behält OpenClaw den bestehenden Fehler für einen unvollständigen Turn bei, anstatt wiederholte Seiteneffekte zu riskieren.

## Aktuelle Einschränkungen

- Der öffentliche Importpfad ist generisch, einige Typaliase für Versuche und Ergebnisse tragen aus Kompatibilitätsgründen jedoch weiterhin veraltete Namen.
- Die Installation von Harnesses von Drittanbietern ist experimentell. Bevorzugen Sie Provider-Plugins, bis Sie eine native Sitzungs-Runtime benötigen.
- Der Wechsel zwischen Harnesses wird über mehrere Turns hinweg unterstützt. Wechseln Sie den Harness nicht mitten in einem Turn, nachdem native Tools, Genehmigungen, Assistententext oder das Senden von Nachrichten begonnen haben.

## Verwandte Themen

- [SDK-Übersicht](/de/plugins/sdk-overview)
- [Runtime-Hilfsfunktionen](/de/plugins/sdk-runtime)
- [Provider-Plugins](/de/plugins/sdk-provider-plugins)
- [Codex-Harness](/de/plugins/codex-harness)
- [Modell-Provider](/de/concepts/model-providers)
