---
read_when:
    - Sie möchten, dass OpenClaw-Agenten im Codex-Modus native Codex-Plugins verwenden
    - Sie migrieren aus dem Quellcode installierte, von OpenAI kuratierte Codex-Plugins
    - Sie konfigurieren ein vorhandenes Codex-Plugin im Workspace-Verzeichnis
    - Sie beheben Probleme mit codexPlugins, dem App-Inventar, destruktiven Aktionen oder der Diagnose von Plugin-Apps.
summary: Native Codex-Plugins für OpenClaw-Agenten im Codex-Modus konfigurieren
title: Native Codex-Plugins
x-i18n:
    generated_at: "2026-07-26T17:58:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0b1cfa39838d4dbd1f33a1e5b7f52faec4b033f9fa98ef5c029003177c2e27e5
    source_path: plugins/codex-native-plugins.md
    workflow: 16
---

Native Codex-Plugin-Unterstützung ermöglicht es einem OpenClaw-Agenten im Codex-Modus, die eigenen App- und Plugin-Funktionen des Codex
app-server innerhalb desselben Codex-Threads zu verwenden, der
den OpenClaw-Turn verarbeitet. Plugin-Aufrufe verbleiben im nativen Codex-Transkript;
Codex app-server übernimmt die App-gestützte MCP-Ausführung. OpenClaw übersetzt
Codex-Plugins nicht in synthetische dynamische `codex_plugin_*`-OpenClaw-Tools.

Verwenden Sie diese Seite, nachdem das grundlegende [Codex-Harness](/de/plugins/codex-harness)
funktioniert.

## Anforderungen

- Die Agent-Runtime muss das native Codex-Harness sein.
- `plugins.entries.codex.enabled` ist `true`.
- `plugins.entries.codex.config.codexPlugins.enabled` ist `true`.
- Der Ziel-Codex-app-server kann auf den erwarteten Marketplace sowie den Plugin- und
  App-Bestand zugreifen.
- Die Migration unterstützt nur `openai-curated`-Plugins, die sie im Quell-Codex-Home als
  aus dem Quellcode installiert erkannt hat.
- Manuell konfigurierte `workspace-directory`-Plugins erfordern einen Codex-app-server,
  dessen `plugin/list` `marketplaceKinds` akzeptiert und dessen pfadlose Workspace-
  Zusammenfassungen `remotePluginId` enthalten. Das Plugin muss bereits installiert und
  aktiviert sein, und die zugehörigen Apps müssen in `app/list` zugänglich sein.

`codexPlugins` hat keine Auswirkungen auf Ausführungen mit dem OpenClaw-Provider, ACP-Konversations-
bindungen oder andere Harnesses, da diese Pfade niemals Codex-
app-server-Threads mit nativer `apps`-Konfiguration erstellen.

Das OpenAI-seitige Codex-Konto, die App-Verfügbarkeit sowie die App-/Plugin-Steuerung des Workspace
stammen aus dem angemeldeten Codex-Konto. Informationen zum OpenAI-Konto und
Administrationsmodell finden Sie unter
[Codex mit Ihrem ChatGPT-Tarif verwenden](https://help.openai.com/en/articles/11369540-using-codex-with-your-chatgpt-plan).

## Schnellstart

Zeigen Sie eine Vorschau der Migration aus dem Quell-Codex-Home an:

```bash
openclaw migrate codex --dry-run
```

Fügen Sie `--verify-plugin-apps` hinzu, damit die Migration das Quell-`app/list` aufruft und
verlangt, dass jede zugehörige App vorhanden, aktiviert und zugänglich ist, bevor
die native Aktivierung geplant wird:

```bash
openclaw migrate codex --dry-run --verify-plugin-apps
```

Wenden Sie die Migration an, wenn der Plan korrekt aussieht:

```bash
openclaw migrate apply codex --yes
```

Die Migration schreibt explizite `codexPlugins`-Einträge für geeignete Plugins und
ruft für ausgewählte Plugins `plugin/install` des Codex-app-server auf. Eine migrierte
Konfiguration sieht wie folgt aus:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          codexPlugins: {
            enabled: true,
            allow_destructive_actions: true,
            plugins: {
              "google-calendar": {
                enabled: true,
                marketplaceName: "openai-curated",
                pluginName: "google-calendar",
              },
            },
          },
        },
      },
    },
  },
}
```

Die Migration bleibt auf `openai-curated` beschränkt. Um ein vorhandenes
`workspace-directory`-Plugin zu verwenden, fügen Sie es manuell mit dem exakten
Marketplace-qualifizierten `summary.id` hinzu, das von `plugin/list` zurückgegeben wird. Wenn
Codex beispielsweise `example-plugin@workspace-directory` zurückgibt, konfigurieren Sie diesen vollständigen
Wert anstelle seines Anzeigenamens:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          codexPlugins: {
            enabled: true,
            plugins: {
              "example-plugin": {
                enabled: true,
                marketplaceName: "workspace-directory",
                pluginName: "example-plugin@workspace-directory",
              },
            },
          },
        },
      },
    },
  },
}
```

OpenClaw ruft für ein `workspace-directory`-Plugin weder `plugin/install` auf noch startet es
eine Authentifizierung. Installieren, aktivieren und authentifizieren Sie es in Codex,
bevor Sie die OpenClaw-Richtlinie hinzufügen oder aktivieren. OpenClaw hält Apps verborgen, wenn
die Antwort den exakten Marketplace, die Plugin-ID, die Detail-ID oder Nachweise
zur App-Bereitschaft auslässt. Wenn Codex die explizite Workspace-Anfrage `plugin/list` ablehnt,
meldet OpenClaw für jedes aktivierte Workspace-Plugin `marketplace_missing` und
hält unabhängig erkannte kuratierte Plugins weiterhin verfügbar.

Nach einer Änderung an `codexPlugins` übernehmen neue Codex-Konversationen den aktualisierten
App-Satz automatisch. Führen Sie `/new` oder `/reset` aus, um die aktuelle
Konversation zu aktualisieren. Für Änderungen beim Aktivieren oder Deaktivieren von Plugins
ist kein Neustart des Gateway erforderlich.

## Plugins über den Chat verwalten

`/codex plugins` prüft oder ändert konfigurierte native Codex-Plugins in demselben
Chat, in dem Sie das Codex-Harness bedienen:

```text
/codex plugins
/codex plugins list
/codex plugins disable google-calendar
/codex plugins enable google-calendar
```

`/codex plugins` ist ein Alias für `/codex plugins list`. Die Liste zeigt den Schlüssel,
den Ein-/Aus-Zustand, den Codex-Plugin-Namen und den Marketplace jedes
konfigurierten Plugins aus `plugins.entries.codex.config.codexPlugins.plugins`.

`enable`/`disable` schreiben ausschließlich in `~/.openclaw/openclaw.json`; sie bearbeiten niemals
`~/.codex/config.toml` und installieren keine neuen Codex-Plugins. Nur der Eigentümer oder ein
Gateway-Client mit dem Geltungsbereich `operator.admin` kann sie ausführen.

Durch das Aktivieren eines konfigurierten Plugins wird auch der globale Schalter `codexPlugins.enabled`
aktiviert. Wenn ein kuratiertes Plugin deaktiviert geschrieben wurde, weil die Migration
`auth_required` zurückgegeben hat, autorisieren Sie die App in Codex erneut, bevor Sie sie in OpenClaw aktivieren.
Bei einem `workspace-directory`-Eintrag ändert das Aktivieren an dieser Stelle nur die OpenClaw-
Richtlinie; das Plugin und die App müssen bereits in Codex aktiv sein.

## Funktionsweise der nativen Plugin-Einrichtung

Die Integration verfolgt drei Zustände:

| Zustand     | Bedeutung                                                                                                                            |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| Installiert | Codex verfügt über das Plugin-Bundle in der Runtime des Ziel-app-server.                                                              |
| Aktiviert   | Codex meldet das Plugin als aktiviert, und die OpenClaw-Konfiguration lässt es für Codex-Harness-Turns zu.                            |
| Zugänglich  | Codex app-server bestätigt, dass die App-Einträge des Plugins für das aktive Konto verfügbar sind und der konfigurierten Plugin-Identität entsprechen. |

Für `openai-curated`-Plugins ist die Migration der dauerhafte Installations-/Eignungs-
schritt:

- Während der Planung liest OpenClaw die `plugin/read`-Details des Quell-Codex und
  prüft, ob es sich beim Konto des Quell-Codex-app-server um ein ChatGPT-Abonnementkonto
  handelt. Bei einer Antwort mit einem Nicht-ChatGPT-Konto oder einem fehlenden Konto werden App-gestützte
  Plugins mit `codex_subscription_required` übersprungen.
- Standardmäßig überspringt die Migration den Quellaufruf `app/list`: App-gestützte Quell-
  Plugins, die die Kontoprüfung bestehen, werden ohne Überprüfung der App-
  Zugänglichkeit in der Quelle geplant, und Transportfehler bei der Kontoabfrage führen dazu,
  dass sie mit `codex_account_unavailable` übersprungen werden.
- Mit `--verify-plugin-apps` erstellt die Migration eine neue Quell-`app/list`-
  Momentaufnahme und verlangt, dass jede zugehörige App vorhanden, aktiviert und
  zugänglich ist, bevor die native Aktivierung geplant wird. Transportfehler bei der Kontoabfrage
  gehen dann in die Prüfung des App-Bestands der Quelle über, anstatt
  unmittelbar zum Überspringen zu führen.

Für `workspace-directory`-Plugins erfolgt die Einrichtung außerhalb von OpenClaw. OpenClaw
fragt diesen Marketplace nur ab, wenn mindestens ein aktivierter Workspace-Eintrag
konfiguriert ist, löst jedes Plugin anhand des exakten `summary.id` auf und verwendet die vorhandenen
Prüfungen für `plugin/read`-Eigentümerschaft und `app/list`-Bereitschaft erneut. Ein nicht installiertes,
deaktiviertes, unzugängliches oder nicht authentifiziertes Plugin stellt keine Apps bereit; OpenClaw
versucht weder eine Installation noch eine Authentifizierung.

Der App-Bestand der Runtime dient sowohl für migrierte kuratierte Plugins als auch für
manuell konfigurierte Workspace-Plugins als Zugänglichkeitsprüfung der Zielsitzung. Bei der Einrichtung einer
Codex-Harness-Sitzung wird aus den aktivierten und zugänglichen Plugin-Apps eine restriktive
Thread-App-Konfiguration berechnet; sie wird nicht bei jedem Turn neu berechnet, daher wirken sich
`/codex plugins enable`/`disable` nur auf
neue Codex-Konversationen aus. Verwenden Sie `/new` oder `/reset`, um die Änderung in der
aktuellen Konversation zu übernehmen.

## V1-Unterstützungsgrenze

- Nur `openai-curated`-Plugins, die bereits im app-server-Bestand des Quell-Codex
  installiert sind, sind für die Migration geeignet.
- Die Runtime unterstützt außerdem explizite `workspace-directory`-Einträge auf app-server-
  Builds, deren `plugin/list` `marketplaceKinds` implementiert und
  `remotePluginId` für pfadlose Workspace-Zusammenfassungen zurückgibt. Diese Einträge müssen
  ihr exaktes Marketplace-qualifiziertes `summary.id` verwenden und bereits installiert,
  aktiviert sowie für Apps zugänglich sein. Eine abgelehnte Workspace-Listenanfrage erzeugt die
  vorhandene pluginspezifische Diagnose `marketplace_missing`; fehlende Marketplace-,
  Plugin-, Detail- oder App-Nachweise stellen keine Workspace-App bereit. Der kuratierte Bestand
  aus der standardmäßigen Listenanfrage bleibt nutzbar.
- App-gestützte Quell-Plugins müssen die Abonnementprüfung zum Migrationszeitpunkt bestehen.
  `--verify-plugin-apps` fügt die Prüfung des Quell-App-Bestands hinzu. Konten, die an der Abonnementprüfung
  scheitern, sowie im Überprüfungsmodus unzugängliche, deaktivierte oder fehlende Quell-
  Apps oder Fehler bei der Aktualisierung des App-Bestands werden als übersprungene manuelle
  Elemente statt als aktivierte Konfigurationseinträge gemeldet. Nicht lesbare Plugin-Details werden
  vor der Prüfung des App-Bestands übersprungen.
- Die Migration schreibt explizite Plugin-Identitäten (`marketplaceName` und
  `pluginName`); sie schreibt keine lokalen `marketplacePath`-Cache-Pfade.
- `codexPlugins.enabled` ist der einzige globale Aktivierungsschalter; es gibt weder
  einen `plugins["*"]`-Platzhalter noch einen Konfigurationsschlüssel, der beliebige Installations-
  berechtigungen gewährt.
- Nicht kuratierte Marketplaces, zwischengespeicherte Plugin-Bundles, Hooks und Codex-Konfigurations-
  dateien bleiben im Migrationsbericht zur manuellen Prüfung erhalten und werden nicht
  automatisch aktiviert. Die Runtime akzeptiert manuell konfigurierte `workspace-directory`-
  Einträge; andere Marketplaces werden weiterhin nicht unterstützt.

## App-Bestand und Eigentümerschaft

OpenClaw liest den Codex-App-Bestand über `app/list` des app-server, speichert ihn
eine Stunde lang im Arbeitsspeicher zwischen und aktualisiert veraltete oder fehlende Einträge
asynchron. Der Cache ist prozesslokal; ein Neustart der CLI oder des Gateway
verwirft ihn, und OpenClaw erstellt ihn beim nächsten Lesen von `app/list` neu.

Migration und Runtime verwenden separate Cache-Schlüssel:

- Die Überprüfung der Quellmigration verwendet das Quell-Codex-Home und die Start-
  optionen. Sie wird nur mit `--verify-plugin-apps` ausgeführt und erzwingt für diesen Planungslauf
  eine neue Traversierung des Quell-`app/list`.
- Die Einrichtung der Ziel-Runtime verwendet die Codex-app-server-Identität des Ziel-Agenten,
  wenn die Thread-App-Konfiguration erstellt wird. Die Aktivierung eines kuratierten Plugins invalidiert diesen
  Ziel-Cache-Schlüssel und erzwingt anschließend nach `plugin/install` dessen Aktualisierung.
  Bei der Einrichtung von `workspace-directory` wird dieser Aktivierungspfad nie ausgeführt.

Eine Plugin-App wird nur bereitgestellt, wenn OpenClaw sie über eine stabile
Eigentümerschaft dem konfigurierten Plugin zuordnen kann: eine exakte App-ID aus den Plugin-Details, einen bekannten
MCP-Servernamen oder eindeutige stabile Metadaten. Eine ausschließlich auf dem Anzeigenamen basierende oder mehrdeutige
Eigentümerschaft wird ausgeschlossen, bis die nächste Bestandsaktualisierung die Eigentümerschaft bestätigt.

## Apps verbundener Konten

Von Eigentümern betriebene Agenten können alle Apps einbeziehen, die bereits mit ihrem Codex-
Konto verbunden sind, ohne dass ein entsprechendes Plugin-Paket erforderlich ist:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          codexPlugins: {
            enabled: true,
            allow_all_plugins: true,
            allow_destructive_actions: "auto",
          },
        },
      },
    },
  },
}
```

`allow_all_plugins: true` erstellt eine vollständige `app/list`-Momentaufnahme, wenn ein neuer nativer
Codex-Thread eingerichtet wird, und lässt nur Apps zu, die für dieses
Konto als zugänglich gekennzeichnet sind. Es installiert, authentifiziert oder aktiviert Apps nicht global. Bestehende
Threads behalten ihren gespeicherten App-Satz; verwenden Sie `/new`, `/reset` oder starten Sie das
Gateway neu, um neu verbundene oder widerrufene Apps zu übernehmen.

Konto-Apps übernehmen den globalen Wert `codexPlugins.allow_destructive_actions`,
der `true`, `false`, `"auto"` oder `"ask"` akzeptiert. Eine explizite Richtlinie pro Plugin
überschreibt die globale Richtlinie für sich überschneidende App-IDs. Inventarisierungsfehler führen
zu einer restriktiven Ablehnung, statt auf eine uneingeschränkte Standardeinstellung zurückzufallen.

## App-Konfiguration für Threads

OpenClaw fügt einen restriktiven `config.apps`-Patch für den Codex-Thread ein:
`_default` ist deaktiviert, und nur Apps, die aktivierten konfigurierten Plugins gehören, oder
zugängliche Konto-Apps, die durch `allow_all_plugins` zugelassen sind, werden aktiviert.

`destructive_enabled` für jede App stammt aus der wirksamen globalen oder
Plugin-spezifischen `allow_destructive_actions`-Richtlinie; `true`, `"auto"` und `"ask"`
setzen alle `destructive_enabled: true`, und `false` setzt es auf `false`. Codex
erzwingt weiterhin Metadaten für destruktive Tools aus seinen nativen App-Tool-Annotationen.
`_default` ist mit `open_world_enabled: false` deaktiviert; aktivierte Plugin-Apps
erhalten `open_world_enabled: true`. OpenClaw stellt keinen separaten
Plugin-weiten Richtlinienparameter für eine offene Welt bereit und verwaltet keine
Plugin-spezifischen Sperrlisten für Namen destruktiver Tools.

Der Tool-Genehmigungsmodus ist für zugelassene Apps standardmäßig automatisch, sodass nicht destruktive
Lese-Tools ohne Genehmigungsaufforderung im selben Thread ausgeführt werden. Destruktive Tools bleiben
durch die `destructive_enabled`-Richtlinie der jeweiligen App kontrolliert.

## Richtlinie für destruktive Aktionen

Destruktive Plugin-Abfragen sind für konfigurierte Codex-
Plugins standardmäßig zulässig, während unsichere Schemas und eine mehrdeutige Eigentümerschaft restriktiv abgelehnt werden:

- Der globale Wert `allow_destructive_actions` ist standardmäßig `true`.
- Der Plugin-spezifische Wert `allow_destructive_actions` überschreibt die globale Richtlinie für
  dieses Plugin.
- `false`: OpenClaw gibt eine deterministische Ablehnung zurück.
- `true`: OpenClaw akzeptiert nur sichere Schemas automatisch, die es einer Genehmigungsantwort
  zuordnen kann, beispielsweise einem booleschen Genehmigungsfeld.
- `"auto"`: OpenClaw macht destruktive Plugin-Aktionen für Codex verfügbar und
  wandelt anschließend MCP-Genehmigungsabfragen mit nachgewiesener Eigentümerschaft in OpenClaw-Plugin-
  Genehmigungen um, bevor es die Codex-Genehmigungsantwort zurückgibt.
- `"ask"`: OpenClaw verwendet dieselbe Codex-Sperrlogik für Schreibvorgänge und destruktive Aktionen wie
  `"auto"`, löscht vor dem Start des Threads dauerhafte Codex-Genehmigungsüberschreibungen pro Tool für die App
  und bietet nur eine einmalige Genehmigung oder Ablehnung an, damit
  dauerhafte Genehmigungen spätere Aufforderungen für Schreibaktionen nicht unterdrücken können. Für jede
  zugelassene App, die `"ask"` verwendet, wählt OpenClaw den Prüfer für menschliche Genehmigungen von Codex
  für diese App aus, sodass Codex seine Genehmigungsabfragen an
  OpenClaw sendet; andere Apps und Thread-Genehmigungen, die sich nicht auf Apps beziehen, behalten ihren konfigurierten
  Prüfer und ihre Richtlinie.
- Eine fehlende Plugin-Identität, eine mehrdeutige Eigentümerschaft, eine fehlende oder nicht übereinstimmende
  Turn-ID oder ein unsicheres Abfrageschema führt zur Ablehnung, statt eine Aufforderung anzuzeigen.

## Fehlerbehebung

| Code                                              | Bedeutung                                                                                                                              | Lösung                                                                                                                    |
| ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------- |
| `auth_required`                                   | Bei der Migration wurde das Plugin installiert, aber eine seiner Apps muss noch authentifiziert werden. Der Eintrag wird deaktiviert geschrieben, bis Sie ihn erneut autorisieren. | Autorisieren Sie die App in Codex erneut und aktivieren Sie anschließend das Plugin in OpenClaw.                                                      |
| `app_inaccessible`, `app_disabled`, `app_missing` | Mit `--verify-plugin-apps` zeigte die Inventarisierung der Codex-Quell-Apps nicht alle zugehörigen Apps als vorhanden, aktiviert und zugänglich an.         | Autorisieren oder aktivieren Sie die App in Codex erneut und führen Sie anschließend die Migration mit `--verify-plugin-apps` erneut aus.                              |
| `app_inventory_unavailable`                       | Eine strikte Überprüfung der Quell-App wurde angefordert, aber die Aktualisierung der Inventarisierung der Codex-Quell-Apps ist fehlgeschlagen.                                      | Beheben Sie den Zugriff auf den App-Server der Codex-Quell-App oder wiederholen Sie den Vorgang ohne `--verify-plugin-apps`, um den schnelleren kontobeschränkten Plan zu akzeptieren.   |
| `codex_subscription_required`                     | Das Konto des App-Servers der Codex-Quell-App war kein ChatGPT-Abonnementkonto.                                                          | Melden Sie sich bei der Codex-App mit einer Abonnementauthentifizierung an und führen Sie anschließend die Migration erneut aus.                                                  |
| `codex_account_unavailable`                       | Das Konto des App-Servers der Codex-Quell-App konnte nicht gelesen werden.                                                                               | Beheben Sie die Authentifizierung des App-Servers der Codex-Quell-App oder führen Sie den Vorgang mit `--verify-plugin-apps` erneut aus, damit die Inventarisierung der Quell-Apps über die Eignung entscheidet. |
| `marketplace_missing`, `plugin_missing`           | Marketplace oder exaktes Plugin nicht verfügbar; die explizite Kataloganfrage für den Workspace wurde möglicherweise abgelehnt; Workspace-Apps werden restriktiv abgelehnt.  | Überprüfen Sie den unten beschriebenen kompatiblen App-Server-Vertrag und die exakte ID.                                                |
| `plugin_detail_unavailable`                       | OpenClaw konnte die Eigentümerschaftsdetails des Plugins nicht lesen.                                                                                    | Prüfen Sie die Antworten `plugin/list` und `plugin/read` des Ziel-App-Servers.                                             |
| `plugin_disabled`                                 | Codex meldet, dass das Plugin installiert, aber deaktiviert ist.                                                                                     | Eine kuratierte Aktivierung kann dies möglicherweise beheben; aktivieren Sie vor einem erneuten Versuch ein Workspace-Plugin in Codex.                                  |
| `plugin_activation_failed`                        | Die Plugin-Aktivierung wurde nicht abgeschlossen.                                                                                                  | Verwenden Sie die beigefügte Diagnose, um zwischen Fehlern bei Marketplace, Authentifizierung, Aktualisierung oder Workspace-Bereitschaft zu unterscheiden.                |
| `app_inventory_missing`, `app_inventory_stale`    | Die App-Bereitschaft stammte aus einem leeren oder veralteten Cache.                                                                                     | OpenClaw plant automatisch eine asynchrone Aktualisierung; Plugin-Apps bleiben ausgeschlossen, bis Eigentümerschaft und Bereitschaft bekannt sind.  |
| `app_ownership_ambiguous`                         | Die App-Inventarisierung ergab nur anhand des Anzeigenamens eine Übereinstimmung.                                                                                          | Die App bleibt für den Codex-Thread verborgen, bis eine spätere Aktualisierung die Eigentümerschaft nachweist.                                     |

**Workspace-Plugin ist installiert, aber nicht sichtbar:** Vergewissern Sie sich, dass das Workspace-
Ergebnis `plugin/list` die exakt konfigurierte ID als installiert und aktiviert meldet,
und anschließend, dass `app/list` jede zugehörige App für dasselbe Codex-
Konto als zugänglich meldet. OpenClaw kann eine zugängliche App für den Thread aktivieren, selbst wenn die
Kontoinventarisierung diese App derzeit als deaktiviert meldet. Wenn Sie diesen Status geändert haben, nachdem das Gateway die App-
Inventarisierung zwischengespeichert hat, warten Sie auf die stündliche Cache-Aktualisierung oder starten Sie das Gateway neu und verwenden Sie anschließend
`/new` oder `/reset`. OpenClaw repariert oder authentifiziert keine Workspace-Plugins.
Wenn die explizite Anfrage für die Workspace-Liste abgelehnt wird, meldet jeder aktivierte Workspace-
Eintrag `marketplace_missing`; nicht damit zusammenhängende kuratierte Einträge werden weiterhin
anhand der Antwort der Standardliste verarbeitet.

Für `plugin_detail_unavailable` muss eine pfadlose Workspace-Zusammenfassung
`remotePluginId` enthalten; OpenClaw hält zugehörige Apps verborgen, wenn dieser Selektor oder das
nachfolgende Ergebnis `plugin/read` nicht verfügbar ist. Für
`plugin_activation_failed` können kuratierte Plugins einen Fehler bei Marketplace, Authentifizierung oder
Aktualisierung nach der Installation melden. Ein Workspace-Plugin meldet diesen Code, wenn es
nicht bereits aktiv ist; installieren, aktivieren und authentifizieren Sie es außerhalb von OpenClaw.

**Konfiguration geändert, aber der Agent kann das Plugin nicht sehen:** Führen Sie `/codex plugins
list` aus, um den konfigurierten Status zu bestätigen, und anschließend `/new` oder `/reset`. Bestehende
Codex-Thread-Bindungen behalten die App-Konfiguration, mit der sie gestartet wurden, bis OpenClaw
eine neue Harness-Sitzung einrichtet oder eine veraltete Bindung ersetzt.

**Destruktive Aktion wird abgelehnt:** Überprüfen Sie die globalen und Plugin-spezifischen
Werte `allow_destructive_actions`. Selbst mit `true`, `"auto"` oder `"ask"`
werden unsichere Abfrageschemas und eine mehrdeutige Plugin-Identität weiterhin restriktiv abgelehnt.

## Verwandte Themen

- [Codex-Harness](/de/plugins/codex-harness)
- [Codex-Harness-Referenz](/de/plugins/codex-harness-reference)
- [Codex-Harness-Laufzeit](/de/plugins/codex-harness-runtime)
- [Konfigurationsreferenz](/de/gateway/configuration-reference#codex-harness-plugin-config)
- [Migrations-CLI](/de/cli/migrate)
