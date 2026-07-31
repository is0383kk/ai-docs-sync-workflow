---
read_when:
    - Sie möchten ein kostenloses Aktionsangebot für ein Modell von ClawHub ausprobieren
    - Sie konfigurieren einen Provider im Rahmen einer Aktion statt über das Onboarding.
summary: CLI-Referenz für `openclaw promos` (Aktionsangebote für Modelle auflisten und beanspruchen)
title: Werbeaktionen
x-i18n:
    generated_at: "2026-07-26T17:43:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 779eab2e9500b7376fabf9accb333e83ff5f84b085d51b7d551b5507b1e73adb
    source_path: cli/promos.md
    workflow: 16
---

# `openclaw promos`

Entdecken und beanspruchen Sie auf ClawHub veröffentlichte Aktionsangebote für Modelle. Beim Beanspruchen einer Aktion wird der Provider konfiguriert (Authentifizierung und Plugin, falls erforderlich), und die Modelle der Aktion werden registriert — ohne das Onboarding erneut auszuführen und ohne Ihr Standardmodell zu ändern, sofern Sie dies nicht ausdrücklich angeben.

Verwandte Themen:

- Standardmodell und Fallbacks: [Modelle](/de/cli/models)
- Einrichtung der Provider-Authentifizierung: [Erste Schritte](/de/start/getting-started)

## Befehle

```bash
openclaw promos list
openclaw promos claim <slug>
openclaw promos claim <slug> --api-key <key> --set-default
```

## `openclaw promos list`

Listet derzeit aktive Aktionen mit ihren Modellen, dem vorgeschlagenen Standardmodell, der verbleibenden Zeit und dem exakten Befehl zum Beanspruchen auf. `--json` gibt die unverarbeiteten Nutzdaten aus.

## `openclaw promos claim <slug>`

Beansprucht eine aktive Aktion:

1. Ruft die Aktion von ClawHub ab und überprüft, ob sie innerhalb ihres Gültigkeitszeitraums liegt.
2. Validiert den Provider, die Authentifizierungsmethode und die deklarierten Plugin-Pakete der Aktion anhand Ihrer installierten OpenClaw-Version. Unbekannte IDs oder nicht übereinstimmende Pakete werden abgelehnt — eine Aktion kann die CLI niemals dazu veranlassen, etwas auszuführen, das sie nicht bereits ausführen kann.
3. Verwendet Ihre vorhandenen Provider-Anmeldedaten, sofern verfügbar. Andernfalls wird der reguläre Authentifizierungsablauf des Providers durchlaufen (zuerst wird die Registrierungs-URL der Aktion für einen kostenlosen Schlüssel ausgegeben). `--api-key <key>` schließt die API-Schlüssel-Authentifizierung ohne Eingabeaufforderungen ab, entsprechend den nicht interaktiven Flags von `openclaw onboard`; um den Schlüssel nicht in der Befehlszeile anzugeben, exportieren Sie stattdessen die Umgebungsvariable des Providers (zum Beispiel `OPENROUTER_API_KEY`) — vorhandene Anmeldedaten aus der Umgebung werden automatisch erkannt, und es ist kein Flag erforderlich.
4. Registriert die Modelle der Aktion mit ihren Aliasnamen. Vorhandene Aliasnamen werden niemals überschrieben.
5. Bietet an, das vorgeschlagene Modell der Aktion als Ihr Standardmodell festzulegen — `--set-default` überspringt die Frage; andernfalls werden Ihre Standardeinstellungen nicht geändert.

Nach Ablauf des Aktionszeitraums stellt der Provider die kostenlosen Modelle nicht mehr bereit; Ihre Konfiguration und Anmeldedaten bleiben unverändert. Mit `openclaw models set <model>` können Sie jederzeit zurückwechseln.

## Passive Erkennung in `models list`

`openclaw models list` zeigt Aktionen auch an, ohne dass Sie ClawHub direkt abfragen:

- Aktive Angebote, deren Modelle Sie nicht konfiguriert haben, werden unterhalb der Tabelle in einer Gruppe „Über Aktion verfügbar“ angezeigt, jeweils mit dem zugehörigen Befehl zum Beanspruchen.
- Modelle, die Sie über `promos claim` registriert haben, tragen das Tag `promo`, das nach Ablauf des Angebotszeitraums zu `promo ended` wechselt.
- Wenn ein neues Angebot erstmals erkannt wird, verweist ein einmaliger Hinweis auf `openclaw promos list`. Bereits aufgelistete oder beanspruchte Angebote werden nicht erneut angekündigt.

Hierbei wird eine lokal zwischengespeicherte Kopie des von ClawHub gehosteten Aktionsfeeds gelesen (normalerweise einmal täglich durch eine bedingte Anfrage aktualisiert oder früher, wenn der zwischengespeicherte Snapshot abläuft; Aktualisierungsfehler werden stillschweigend übersprungen). Eine Aktualisierung veralteter Daten wartet höchstens 2,5 Sekunden und beeinträchtigt die Auflistung niemals. Die Ausgaben von `--json` und `--plain` bleiben maschinenlesbar: Sie enthalten weder Aktionsabschnitte noch Hinweise. Beim Beanspruchen erfolgt stets eine erneute Validierung anhand der Live-API von ClawHub. Daher wird ein vorzeitig zurückgezogenes Angebot auch dann abgelehnt, wenn es in einer zwischengespeicherten Kopie noch angezeigt wird.
