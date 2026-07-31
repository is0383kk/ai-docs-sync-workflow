---
read_when:
    - Verhalten oder Standardeinstellungen der Tippanzeige ändern
summary: Wann OpenClaw Tippindikatoren anzeigt und wie sie angepasst werden können
title: Tippindikatoren
x-i18n:
    generated_at: "2026-07-26T18:25:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b3c66d61ea7e3e809b8e88ae2eabb9794f0886b629094753716ed02912843ffc
    source_path: concepts/typing-indicators.md
    workflow: 16
---

Tippindikatoren werden an den Chatkanal gesendet, während eine Ausführung aktiv ist. Verwenden Sie `agents.defaults.typingMode`, um zu steuern, **wann** das Tippen beginnt, und `typingIntervalSeconds`, um zu steuern, **wie oft** der Indikator aktualisiert wird (Keepalive-Intervall, standardmäßig 6 Sekunden).

## Standardwerte

Wenn `agents.defaults.typingMode` **nicht festgelegt** ist:

- **Direktchats**: Das Tippen beginnt sofort, sobald die Modellschleife startet.
- **Gruppenchats mit einer Erwähnung**: Das Tippen beginnt sofort.
- **Gruppenchats ohne Erwähnung**: Das Tippen beginnt, sobald die zugelassene Ausführung eine für Benutzer sichtbare Aktivität aufweist, etwa eine Aktivität bei der Harness-Ausführung oder Nachrichtentext.
- **Heartbeat-Ausführungen**: Das Tippen beginnt beim Start der Heartbeat-Ausführung, sofern das aufgelöste Heartbeat-Ziel ein Chat mit Tippunterstützung ist und das Tippen nicht deaktiviert wurde.

## Modi

Legen Sie `agents.defaults.typingMode` auf einen der folgenden Werte fest:

- `never` – niemals ein Tippindikator.
- `instant` – das Tippen **beginnt, sobald die Modellschleife startet**, selbst wenn die Ausführung später nur das Token für eine stille Antwort zurückgibt.
- `thinking` – das Tippen beginnt beim **ersten Reasoning-Delta** oder bei aktiver Harness-Ausführung, nachdem der Turn angenommen wurde.
- `message` – das Tippen beginnt bei der **ersten für Benutzer sichtbaren Antwortaktivität**, etwa bei aktiver Harness-Ausführung oder einem nicht stillen Text-Delta. Token für stille Antworten wie `NO_REPLY` zählen nicht als Textaktivität.

Reihenfolge danach, „wie früh der Modus ausgelöst wird“: `never` -> `message`/`thinking` -> `instant`.

## Konfiguration

Legen Sie den Standardwert auf Agent-Ebene fest:

```json5
{
  agents: {
    defaults: {
      typingMode: "thinking",
      typingIntervalSeconds: 6,
    },
  },
}
```

Überschreiben Sie die Richtlinie für einen einzelnen Agenten:

```json5
{
  agents: {
    entries: {
      support: {
        typingMode: "message",
      },
    },
  },
}
```

## Hinweise

- Der Modus `message` wird nicht durch Token für stille Antworten gestartet, eine aktive Ausführung kann jedoch weiterhin einen Tippindikator anzeigen, bevor Assistententext verfügbar ist.
- `thinking` reagiert weiterhin auf gestreamtes Reasoning (`reasoningLevel: "stream"`) und kann auch durch eine aktive Ausführung gestartet werden, bevor Reasoning-Deltas eintreffen.
- Der Heartbeat-Tippindikator ist ein Verfügbarkeitssignal für das aufgelöste Zustellungsziel. Er beginnt beim Start der Heartbeat-Ausführung, statt dem Stream-Timing von `message` oder `thinking` zu folgen. Legen Sie `typingMode: "never"` fest, um ihn zu deaktivieren.
- Heartbeats zeigen keinen Tippindikator an, wenn das Heartbeat-Ziel `"none"` ist, das Ziel nicht aufgelöst werden kann, die Chatzustellung für den Heartbeat deaktiviert ist oder der Kanal keine Tippindikatoren unterstützt.
- `agents.defaults.typingIntervalSeconds` steuert das **Aktualisierungsintervall** für jeden Agenten, nicht den Startzeitpunkt. Standardwert: 6 Sekunden.

## Verwandte Themen

<CardGroup cols={2}>
  <Card title="Präsenz" href="/de/concepts/presence" icon="signal">
    So verfolgt der Gateway verbundene Clients für die Seite „Geräte“ der Control UI und den Tab „Instanzen“ unter macOS.
  </Card>
  <Card title="Streaming und Segmentierung" href="/de/concepts/streaming" icon="bars-staggered">
    Verhalten beim ausgehenden Streaming, Segmentgrenzen und kanalspezifische Zustellung.
  </Card>
</CardGroup>
