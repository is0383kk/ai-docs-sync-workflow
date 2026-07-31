---
read_when:
    - Sie pflegen ein OpenClaw-Plugin
    - Sie sehen eine Plugin-Kompatibilitätswarnung
    - Sie planen eine Migration des Plugin-SDKs oder Manifests
summary: Plugin-Kompatibilitätsverträge, Metadaten zu veralteten Funktionen und Migrationserwartungen
title: Plugin-Kompatibilität
x-i18n:
    generated_at: "2026-07-26T18:35:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 80cf1dfce9e0538e78138ff80a6807ee36267a07d3eee6f19bd8e56e5c0c9cd3
    source_path: plugins/compatibility.md
    workflow: 16
---

OpenClaw bindet ältere Plugin-Verträge über benannte Kompatibilitätsadapter
ein, bevor diese entfernt werden. Dies schützt bestehende gebündelte und externe
Plugins, während sich die Verträge für SDK, Manifest, Einrichtung, Konfiguration und Agent-Laufzeit
weiterentwickeln.

## Kompatibilitätsregister

Plugin-Kompatibilitätsverträge werden im zentralen Register unter
`src/plugins/compat/registry.ts` nachverfolgt. Jeder Eintrag enthält:

- einen stabilen Kompatibilitätscode
- Status: `active`, `deprecated`, `removal-pending` oder `removed`
- Verantwortungsbereich: `sdk`, `config`, `setup`, `channel`, `provider`, `plugin-execution`,
  `agent-runtime` oder `core`
- Einführungs- und Einstellungsdaten, sofern zutreffend
- ein genaues Entfernungsdatum, sobald der zuständige Maintainer es genehmigt; ein fehlendes
  `removeAfter` schließt eine veraltete Oberfläche von der Entfernung aus
- Hinweise zum Ersatz
- Dokumentation, Diagnosen und Tests, die das alte und neue Verhalten abdecken

Das Register dient als Quelle für die Planung durch Maintainer und zukünftige
Prüfungen des Plugin-Inspektors. Wenn sich ein Plugin-relevantes Verhalten ändert,
muss im selben Änderungssatz, der den Adapter hinzufügt, auch der
Kompatibilitätseintrag hinzugefügt oder aktualisiert werden.

Kompatibilität für Reparaturen und Migrationen durch Doctor wird separat unter
`src/commands/doctor/shared/deprecation-compat.ts` nachverfolgt. Diese Einträge decken alte
Konfigurationsstrukturen, Layouts des Installationsverzeichnisses und Reparatur-Shims ab, die
möglicherweise auch nach Entfernung des Laufzeit-Kompatibilitätspfads verfügbar bleiben müssen.

Release-Prüfungen sollten beide Register prüfen. Löschen Sie keine Doctor-
Migration, nur weil der zugehörige Laufzeit- oder Konfigurationskompatibilitätseintrag
abgelaufen ist; prüfen Sie zuerst, ob weiterhin ein unterstützter Upgrade-Pfad
die Reparatur benötigt. Validieren Sie während der Release-Planung auch jede
Ersatzanmerkung erneut, da sich Plugin-Verantwortung und Konfigurationsumfang ändern können,
wenn Provider und Kanäle aus dem Kern ausgelagert werden.

## Einstellungsrichtlinie

OpenClaw sollte einen dokumentierten Plugin-Vertrag nicht im selben Release
entfernen, in dem dessen Ersatz eingeführt wird. Migrationsreihenfolge:

1. Den neuen Vertrag hinzufügen.
2. Das alte Verhalten über einen benannten Kompatibilitätsadapter eingebunden lassen.
3. Diagnosen oder Warnungen ausgeben, sobald Plugin-Autoren handeln können.
4. Den Ersatz und den Zeitplan dokumentieren.
5. Sowohl den alten als auch den neuen Pfad testen.
6. Das angekündigte Migrationszeitfenster abwarten.
7. Nur mit ausdrücklicher Genehmigung für ein inkompatibles Release entfernen.

Veraltete Einträge müssen ein Startdatum für Warnungen, einen Ersatz, einen
Dokumentationslink und ein endgültiges Entfernungsdatum enthalten, das höchstens drei Monate nach
Beginn der Warnungen liegt. Fügen Sie keinen veralteten Kompatibilitätspfad mit einem unbefristeten
Entfernungszeitfenster hinzu, es sei denn, die Maintainer entscheiden ausdrücklich, dass es sich um dauerhafte
Kompatibilität handelt, und markieren ihn stattdessen als `active`.

## Aktuelle Kompatibilitätsbereiche

Bei der Prüfung im Juli 2026 wurden die abgelaufenen Aliasse für das Stamm-SDK, Manifest, Provider, Laufzeit,
Register-Flag und die Plugin-eigene Webkonfiguration entfernt. Doctor-Migrationen werden weiterhin
separat nachverfolgt, damit unterstützte Upgrade-Pfade alte Konfigurationen weiterhin reparieren können.

Die verbleibenden zeitlich begrenzten Kompatibilitätsbereiche sind:

- die im Migrationsleitfaden aufgeführten SDK-Unterpfad-Zeitfenster für August und September
- die Hook-Aliasse `api.on("deactivate", ...)` und `api.on("subagent_spawning", ...)`
- speicherspezifische Einbettungsregistrierung und die Sitzungsspeicher-Brücke von beta.5
- die unten beschriebenen Aliasse für eingehende WhatsApp-Callbacks
- explizite Analyse von Kanalzielen und `openclaw/plugin-sdk/messaging-targets`
- eingebettete Pi-Agent-Aliasse
- die ausgelieferten SDK-Aliasse des Agent-Harness, deren Entfernung noch von einer neuen
  extern dokumentierten Migrationsentscheidung abhängt

Aktive, undatierte Registereinträge decken unterstütztes Verhalten statt
Entfernungsrückstände ab, einschließlich Aktivierungshinweisen, Plugin-Erfassung, Aktivierung gebündelter Plugins
und des generierten Fallbacks für die Kanalkonfiguration.

### Flache Aliasse für eingehende WhatsApp-Callbacks

WhatsApp-Laufzeit-Callbacks liefern `WebInboundMessage`: die kanonischen
verschachtelten Kontexte `event`, `payload`, `quote`, `group` und `platform` sowie
veraltete flache Aliasse für die ausgelieferten Callback-Felder. Neuer Callback-Code
sollte die verschachtelten Kontexte lesen. Code, der saubere verschachtelte Callback-
Nachrichten erstellt, kann `WebInboundCallbackMessage` verwenden; Kompatibilitäts-Listener, die
weiterhin alte flache Test- oder Plugin-Nachrichten einspeisen, sollten
`LegacyFlatWebInboundMessage` oder `WebInboundMessageInput` verwenden.

Die flachen Aliasse bleiben bis zum **2026-08-30** verfügbar; dieses Zeitfenster gilt
nur für den Zugriff über flache Aliasse, nicht für die verschachtelte Struktur, die den kanonischen
Laufzeitvertrag darstellt. Die TypeScript-Annotation `@deprecated` jedes flachen Alias
benennt den genauen verschachtelten Ersatz. Häufige Beispiele:

- `id`, `timestamp` und `isBatched` werden unter `event` verschoben.
- `body`, `mediaPath`, `mediaType`, `mediaFileName`, `mediaUrl`, `location`
  und `untrustedStructuredContext` werden unter `payload` verschoben.
- `to`, `chatId`, Absender-/Selbstfelder, `sendComposing`, `reply(...)` und
  `sendMedia(...)` werden unter `platform` verschoben.
- Die Felder von `replyTo*` werden unter `quote` verschoben; Felder für Gruppenbetreff, Teilnehmer und Erwähnungen
  werden unter `group` verschoben.

`payload.untrustedStructuredContext` wird aus eingehenden Provider-
Nutzdaten extrahiert. Plugins sollten `label`, `source` und `type` prüfen, bevor
sie dessen `payload` als maßgeblich behandeln.

### Zulassungsfelder für eingehende WhatsApp-Nachrichten

Akzeptierte WhatsApp-Callback-Nachrichten enthalten `admission`, eine öffentlich unbedenkliche
Hülle für die Zugriffskontrollentscheidung, durch die die Nachricht zugelassen wurde. Neuer
Callback-Code sollte Zulassungsinformationen aus `msg.admission` statt aus
den älteren Zulassungsfeldern auf oberster Ebene lesen.

Die Felder auf oberster Ebene bleiben bis zum **2026-08-30** verfügbar. Die
TypeScript-Annotation `@deprecated` jedes Feldes benennt dessen Ersatz:

- `from` und `conversationId` werden nach `admission.conversation.id` verschoben.
- `accountId` wird nach `admission.accountId` verschoben.
- `accessControlPassed` ist eine abgeleitete Kompatibilitätsansicht von
  `admission.ingress.decision === "allow"`; bei Nachrichten, die bereits
  `admission` enthalten, schreibt das Setzen des alten booleschen Werts den Eingangs-
  graphen nicht neu.
- `chatType` wird nach `admission.conversation.kind` verschoben.

## Paket des Plugin-Inspektors

Der Plugin-Inspektor sollte außerhalb des zentralen OpenClaw-Repos als
separates Paket/Repository angesiedelt sein und auf den versionierten Kompatibilitäts- und
Manifestverträgen basieren. Die CLI für den ersten Tag sollte lauten:

```sh
openclaw-plugin-inspector ./my-plugin
```

Sie sollte Manifest-/Schemavalidierung, die geprüfte Version des
Kompatibilitätsvertrags, Prüfungen von Installations-/Quellmetadaten, Importprüfungen für selten genutzte Pfade
sowie Einstellungs-/Kompatibilitätswarnungen ausgeben. Verwenden Sie `--json` für stabile
maschinenlesbare Ausgaben in CI-Anmerkungen. Der OpenClaw-Kern sollte
Verträge und Fixtures bereitstellen, die der Inspektor verwenden kann, aber die
Inspektor-Binärdatei nicht über das Hauptpaket `openclaw` veröffentlichen.

### Abnahme-Lane für Maintainer

Verwenden Sie die Crabbox-gestützte Blacksmith Testbox für die Abnahme-Lane
installierbarer Pakete, wenn der externe Inspektor mit OpenClaw-Plugin-
Paketen validiert wird. Führen Sie sie nach dem Erstellen des Pakets aus einem sauberen OpenClaw-Checkout aus:

```sh
pnpm crabbox:run -- --provider blacksmith-testbox --timing-json --shell -- "pnpm install && pnpm build && npm exec --yes @openclaw/plugin-inspector@0.1.0 -- ./extensions/telegram --json"
pnpm crabbox:run -- --provider blacksmith-testbox --timing-json --shell -- "npm exec --yes @openclaw/plugin-inspector@0.1.0 -- ./extensions/discord --json"
pnpm crabbox:run -- --provider blacksmith-testbox --timing-json --shell -- "npm exec --yes @openclaw/plugin-inspector@0.1.0 -- <clawhub-plugin-dir> --json"
```

Lassen Sie diese Lane für Maintainer optional, da sie ein externes npm-
Paket installiert und möglicherweise Plugin-Pakete untersucht, die außerhalb des Repos geklont wurden. Die lokalen
Repo-Prüfungen decken die SDK-Exportzuordnung, Metadaten des Kompatibilitätsregisters,
den Abbau veralteter SDK-Importe und Importgrenzen gebündelter Erweiterungen ab;
der Testbox-Nachweis des Inspektors deckt das Paket so ab, wie externe Plugin-Autoren
es verwenden.

## Versionshinweise

Versionshinweise sollten bevorstehende Plugin-Einstellungen mit Zieldaten
und Links zur Migrationsdokumentation enthalten, bevor ein Kompatibilitätspfad in
`removal-pending` oder `removed` übergeht.
