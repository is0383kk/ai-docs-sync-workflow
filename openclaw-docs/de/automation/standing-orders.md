---
read_when:
    - Autonome Agenten-Workflows einrichten, die ohne Aufforderung für jede einzelne Aufgabe ausgeführt werden
    - Festlegen, was der Agent selbstständig tun kann und was eine menschliche Genehmigung erfordert
    - Multi-Programm-Agenten mit klaren Grenzen und Eskalationsregeln strukturieren
summary: Definieren Sie dauerhafte Handlungsbefugnisse für autonome Agentenprogramme
title: Daueraufträge
x-i18n:
    generated_at: "2026-07-26T18:18:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9e7ad622efe734facc9dc3716f5ee7f57ed3923499db78730bda234a5c62ad80
    source_path: automation/standing-orders.md
    workflow: 16
---

Daueraufträge gewähren Ihrem Agenten **dauerhafte Handlungsbefugnis** für definierte Programme. Statt den Agenten für jede Aufgabe einzeln anzuweisen, definieren Sie Programme mit einem klaren Umfang, Auslösern und Eskalationsregeln. Der Agent führt sie innerhalb dieser Grenzen autonom aus: „Sie sind für den wöchentlichen Bericht verantwortlich. Erstellen und versenden Sie ihn jeden Freitag und eskalieren Sie nur, wenn etwas nicht stimmt.“

## Warum Daueraufträge?

**Ohne Daueraufträge:** Sie weisen den Agenten für jede Aufgabe einzeln an, Routinearbeiten werden vergessen oder verzögert und Sie werden zum Engpass.

**Mit Daueraufträgen:** Der Agent handelt innerhalb definierter Grenzen autonom, Routinearbeiten erfolgen planmäßig und Sie werden nur bei Ausnahmen und erforderlichen Genehmigungen einbezogen.

## Funktionsweise

Daueraufträge werden in den Dateien Ihres [Agenten-Arbeitsbereichs](/de/concepts/agent-workspace) definiert. Es wird empfohlen, sie direkt in `AGENTS.md` aufzunehmen (wird in jeder Sitzung automatisch eingefügt), damit sie dem Agenten stets als Kontext zur Verfügung stehen. Bei umfangreicheren Konfigurationen können Sie sie auch in einer eigenen Datei wie `standing-orders.md` ablegen und aus `AGENTS.md` darauf verweisen.

Jedes Programm legt Folgendes fest:

1. **Umfang** – wozu der Agent berechtigt ist
2. **Auslöser** – wann die Ausführung erfolgt (Zeitplan, Ereignis oder Bedingung)
3. **Genehmigungsschranken** – was vor der Ausführung eine menschliche Freigabe erfordert
4. **Eskalationsregeln** – wann der Agent anhalten und um Hilfe bitten muss

Der Agent lädt diese Anweisungen in jeder Sitzung über die Bootstrap-Dateien des Arbeitsbereichs (die vollständige Liste der automatisch eingefügten Dateien finden Sie unter [Agenten-Arbeitsbereich](/de/concepts/agent-workspace)) und führt sie zusammen mit [Cron-Jobs](/de/automation/cron-jobs) zur zeitgesteuerten Durchsetzung aus.

<Tip>
Legen Sie Daueraufträge in `AGENTS.md` ab, damit sie garantiert in jeder Sitzung geladen werden. Der Arbeitsbereich-Bootstrap fügt `AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md`, `BOOTSTRAP.md` und `MEMORY.md` automatisch ein – jedoch keine beliebigen Dateien aus Unterverzeichnissen.
</Tip>

## Aufbau eines Dauerauftrags

```markdown
## Programm: Wöchentlicher Statusbericht

**Befugnis:** Daten zusammenstellen, Bericht erstellen und an Beteiligte übermitteln
**Auslöser:** Jeden Freitag um 16 Uhr (durch einen Cron-Job durchgesetzt)
**Genehmigungsschranke:** Keine für Standardberichte. Auffälligkeiten zur menschlichen Prüfung kennzeichnen.
**Eskalation:** Wenn eine Datenquelle nicht verfügbar ist oder Kennzahlen ungewöhnlich erscheinen (>2σ von der Norm)

### Ausführungsschritte

1. Kennzahlen aus den konfigurierten Quellen abrufen
2. Mit der Vorwoche und den Zielwerten vergleichen
3. Bericht unter Reports/weekly/YYYY-MM-DD.md erstellen
4. Zusammenfassung über den konfigurierten Kanal übermitteln
5. Abschluss unter Agent/Logs/ protokollieren

### Was NICHT zu tun ist

- Berichte nicht an externe Parteien senden
- Quelldaten nicht verändern
- Die Zustellung bei schlechten Kennzahlen nicht überspringen – korrekt berichten
```

## Daueraufträge und Cron-Jobs

Daueraufträge definieren, **wozu** der Agent berechtigt ist. [Cron-Jobs](/de/automation/cron-jobs) definieren, **wann** es erfolgt. Sie wirken zusammen:

```text
Dauerauftrag: „Sie sind für die tägliche Sichtung des Posteingangs verantwortlich“
    ↓
Cron-Job (täglich um 8 Uhr): „Posteingang gemäß den Daueraufträgen sichten“
    ↓
Agent: Liest Daueraufträge → führt Schritte aus → meldet Ergebnisse
```

Die Eingabeaufforderung des Cron-Jobs sollte auf den Dauerauftrag verweisen, statt ihn zu duplizieren:

```bash
openclaw cron add \
  --name daily-inbox-triage \
  --cron "0 8 * * 1-5" \
  --tz America/New_York \
  --timeout-seconds 300 \
  --announce \
  --channel imessage \
  --to "+1XXXXXXXXXX" \
  --message "Tägliche Sichtung des Posteingangs gemäß den Daueraufträgen ausführen. E-Mails auf neue Warnmeldungen prüfen. Jedes Element analysieren, kategorisieren und dauerhaft speichern. Dem Eigentümer eine Zusammenfassung melden. Unbekannte Fälle eskalieren."
```

## Beispiele

### Beispiel 1: Inhalte und soziale Medien (wöchentlicher Zyklus)

```markdown
## Programm: Inhalte und soziale Medien

**Befugnis:** Inhalte entwerfen, Beiträge planen, Interaktionsberichte zusammenstellen
**Genehmigungsschranke:** Alle Beiträge erfordern in den ersten 30 Tagen die Prüfung durch den Eigentümer, danach gilt eine dauerhafte Genehmigung
**Auslöser:** Wöchentlicher Zyklus (Prüfung am Montag → Entwürfe zur Wochenmitte → Kurzbericht am Freitag)

### Wöchentlicher Zyklus

- **Montag:** Plattformkennzahlen und Publikumsinteraktionen prüfen
- **Dienstag bis Donnerstag:** Beiträge für soziale Medien entwerfen und Blog-Inhalte erstellen
- **Freitag:** Wöchentlichen Marketing-Kurzbericht zusammenstellen → an den Eigentümer übermitteln

### Inhaltsregeln

- Der Ton muss zur Marke passen (siehe SOUL.md oder Leitfaden zur Markensprache)
- In öffentlich sichtbaren Inhalten niemals als KI ausgeben
- Kennzahlen einbeziehen, sofern verfügbar
- Auf den Mehrwert für das Publikum statt auf Eigenwerbung konzentrieren
```

### Beispiel 2: Finanzvorgänge (ereignisgesteuert)

```markdown
## Programm: Finanzverarbeitung

**Befugnis:** Transaktionsdaten verarbeiten, Berichte erstellen, Zusammenfassungen senden
**Genehmigungsschranke:** Keine für Analysen. Empfehlungen erfordern die Genehmigung des Eigentümers.
**Auslöser:** Neue Datendatei erkannt ODER geplanter monatlicher Zyklus

### Beim Eingang neuer Daten

1. Neue Datei im vorgesehenen Eingabeverzeichnis erkennen
2. Alle Transaktionen analysieren und kategorisieren
3. Mit den Budgetzielen vergleichen
4. Folgendes kennzeichnen: ungewöhnliche Posten, Schwellenwertüberschreitungen, neue wiederkehrende Belastungen
5. Bericht im vorgesehenen Ausgabeverzeichnis erstellen
6. Zusammenfassung über den konfigurierten Kanal an den Eigentümer übermitteln

### Eskalationsregeln

- Einzelposten > $500: sofortige Warnmeldung
- Kategorie liegt 20 % über dem Budget: im Bericht kennzeichnen
- Nicht erkennbare Transaktion: Eigentümer um Kategorisierung bitten
- Verarbeitung nach 2 Wiederholungsversuchen fehlgeschlagen: Fehler melden, nicht raten
```

### Beispiel 3: Überwachung und Warnmeldungen (kontinuierlich)

```markdown
## Programm: Systemüberwachung

**Befugnis:** Systemzustand prüfen, Dienste neu starten, Warnmeldungen senden
**Genehmigungsschranke:** Dienste automatisch neu starten. Eskalieren, wenn der Neustart zweimal fehlschlägt.
**Auslöser:** Bei jedem Heartbeat-Zyklus

### Prüfungen

- Zustandsendpunkte der Dienste antworten
- Verfügbarer Speicherplatz liegt über dem Schwellenwert
- Ausstehende Aufgaben sind nicht veraltet (>24 Stunden)
- Zustellungskanäle sind betriebsbereit

### Reaktionsmatrix

| Bedingung             | Aktion                              | Eskalieren?                       |
| --------------------- | ----------------------------------- | --------------------------------- |
| Dienst ausgefallen    | Automatisch neu starten             | Nur wenn der Neustart 2x fehlschlägt |
| Speicherplatz < 10 %  | Eigentümer warnen                   | Ja                                |
| Veraltete Aufgabe > 24h | Eigentümer erinnern               | Nein                              |
| Kanal offline         | Protokollieren und im nächsten Zyklus erneut versuchen | Wenn länger als 2 Stunden offline |
```

## Muster „Ausführen–Überprüfen–Berichten“

Daueraufträge funktionieren am besten in Verbindung mit strenger Ausführungsdisziplin. Jede Aufgabe in einem Dauerauftrag sollte dieser Schleife folgen:

1. **Ausführen** – Die eigentliche Arbeit erledigen (die Anweisung nicht nur bestätigen)
2. **Überprüfen** – Bestätigen, dass das Ergebnis korrekt ist (Datei vorhanden, Nachricht zugestellt, Daten analysiert)
3. **Berichten** – Dem Eigentümer mitteilen, was erledigt und was überprüft wurde

```markdown
### Ausführungsregeln

- Jede Aufgabe folgt dem Muster Ausführen–Überprüfen–Berichten. Keine Ausnahmen.
- „Ich werde das erledigen“ ist keine Ausführung. Führen Sie es aus und berichten Sie anschließend.
- „Erledigt“ ohne Überprüfung ist nicht akzeptabel. Weisen Sie es nach.
- Wenn die Ausführung fehlschlägt: einmal mit einer angepassten Vorgehensweise erneut versuchen.
- Wenn sie weiterhin fehlschlägt: Fehler mit Diagnose melden. Fehler niemals verschweigen.
- Niemals unbegrenzt wiederholen – maximal 3 Versuche, dann eskalieren.
```

Dieses Muster verhindert den häufigsten Fehlermodus eines Agenten: eine Aufgabe zu bestätigen, ohne sie abzuschließen.

## Architektur mit mehreren Programmen

Für Agenten, die mehrere Bereiche verwalten, sollten Daueraufträge als separate Programme mit klaren Grenzen organisiert werden:

```markdown
## Programm 1: [Bereich A] (Wöchentlich)

...

## Programm 2: [Bereich B] (Monatlich + bei Bedarf)

...

## Programm 3: [Bereich C] (Nach Bedarf)

...

## Eskalationsregeln (Alle Programme)

- [Gemeinsame Eskalationskriterien]
- [Programmübergreifend geltende Genehmigungsschranken]
```

Jedes Programm sollte Folgendes besitzen:

- Einen eigenen **Ausführungsrhythmus** (wöchentlich, monatlich, ereignisgesteuert, kontinuierlich)
- Eigene **Genehmigungsschranken** (einige Programme benötigen mehr Aufsicht als andere)
- Klare **Grenzen** (der Agent sollte wissen, wo ein Programm endet und ein anderes beginnt)

## Bewährte Vorgehensweisen

### Empfohlen

- Mit eng begrenzten Befugnissen beginnen und diese mit wachsendem Vertrauen erweitern
- Explizite Genehmigungsschranken für risikoreiche Aktionen definieren
- Abschnitte zu „Was NICHT zu tun ist“ aufnehmen – Grenzen sind ebenso wichtig wie Berechtigungen
- Mit Cron-Jobs kombinieren, um eine zuverlässige zeitgesteuerte Ausführung zu gewährleisten
- Agentenprotokolle wöchentlich prüfen, um die Einhaltung der Daueraufträge zu bestätigen
- Daueraufträge an veränderte Anforderungen anpassen – sie sind lebende Dokumente

### Vermeiden

- Am ersten Tag weitreichende Befugnisse erteilen („Tun Sie, was Sie für richtig halten“)
- Eskalationsregeln auslassen – jedes Programm benötigt eine Klausel dazu, wann der Agent anhalten und nachfragen muss
- Davon ausgehen, dass sich der Agent an mündliche Anweisungen erinnert – alles in der Datei festhalten
- Mehrere Bereiche in einem einzigen Programm vermischen – separate Programme für separate Bereiche verwenden
- Die Durchsetzung durch Cron-Jobs vergessen – Daueraufträge ohne Auslöser werden zu Vorschlägen

## Verwandte Themen

- [Automatisierung](/de/automation): alle Automatisierungsmechanismen im Überblick.
- [Cron-Jobs](/de/automation/cron-jobs): zeitgesteuerte Durchsetzung von Daueraufträgen.
- [Hooks](/de/automation/hooks): ereignisgesteuerte Skripte für Ereignisse im Lebenszyklus des Agenten.
- [Webhooks](/de/automation/cron-jobs#webhooks): Auslöser für eingehende HTTP-Ereignisse.
- [Agenten-Arbeitsbereich](/de/concepts/agent-workspace): der Speicherort von Daueraufträgen einschließlich der vollständigen Liste automatisch eingefügter Bootstrap-Dateien (`AGENTS.md`, `SOUL.md` usw.).
