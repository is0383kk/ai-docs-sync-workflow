---
read_when:
    - Sie möchten zwischen Stable/Extended-Stable/Beta/Dev wechseln
    - Sie möchten eine bestimmte Version, ein bestimmtes Tag oder einen bestimmten SHA anheften
    - Sie taggen oder veröffentlichen Vorabversionen
sidebarTitle: Release Channels
summary: 'Stable-, Extended-Stable-, Beta- und Dev-Kanäle: Semantik, Wechsel, Pinning und Tagging'
title: Release-Kanäle
x-i18n:
    generated_at: "2026-07-26T18:29:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a99e31f5121c0ab8696e638cb10a7ce16e8f32c81e4b2bef1f703eef71191494
    source_path: install/development-channels.md
    workflow: 16
---

OpenClaw wird über vier Update-Kanäle ausgeliefert:

- **stable**: npm-Dist-Tag `latest`. Für die meisten Benutzer empfohlen.
- **extended-stable**: npm-Dist-Tag `extended-stable`. Ein vollständig neuer, nachlaufender
  Paketkanal für unterstützte Monate. Er ist ausschließlich für Pakete vorgesehen und die Installation
  erfolgt nur im Vordergrund. Eine gespeicherte Auswahl erhält schreibgeschützte Update-Hinweise, wenn
  `update.checkOnStart` aktiviert ist, wendet Updates jedoch nie automatisch an.
- **beta**: npm-Dist-Tag `beta`. Fällt auf `latest` zurück, wenn `beta` fehlt
  oder älter als die aktuelle stabile Version ist.
- **dev**: fortlaufend aktualisierter Stand von `main` (Git). npm-Dist-Tag `dev`, sofern veröffentlicht. `main`
  ist für Experimente und die aktive Entwicklung vorgesehen; der Kanal kann unvollständige
  Funktionen oder inkompatible Änderungen enthalten. Verwenden Sie ihn nicht für produktive Gateways.

Stabile Builds werden normalerweise zuerst über **beta** ausgeliefert, dort geprüft und anschließend
ohne Erhöhung der Versionsnummer zu **latest** hochgestuft. Maintainer können auch direkt unter
`latest` veröffentlichen. Dist-Tags sind die maßgebliche Quelle für npm-Installationen.

## Kanäle wechseln

```bash
openclaw update --channel stable
openclaw update --channel extended-stable
openclaw update --channel beta
openclaw update --channel dev
```

`--channel` speichert die Auswahl als `update.channel` in der Konfiguration und steuert beide
Installationspfade:

| Kanal             | npm-/Paketinstallationen                                                                                                                                                               | Git-Installationen                                                                                                                                                  |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `stable`          | Dist-Tag `latest`                                                                                                                                                                      | neuestes stabiles Git-Tag (ausgenommen `-alpha.N`, `-beta.N`, `-rc.N`, `-dev.N`, `-next.N`, `-preview.N`, `-canary.N`, `-nightly.N` und andere benannte Vorabversionssuffixe) |
| `extended-stable` | löst den öffentlichen npm-Selektor `extended-stable` auf, verifiziert das exakt ausgewählte Paket und installiert genau diese Version. Schlägt ohne Rückgriff auf `latest`, `beta` oder `dev` sicher fehl. | nicht unterstützt: OpenClaw lässt den Checkout unverändert und fordert Sie auf, eine Paketinstallation zu verwenden                                                 |
| `beta`            | Dist-Tag `beta`, mit Rückgriff auf `latest`, wenn `beta` fehlt oder älter ist                                                                                                      | neuestes Beta-Git-Tag, mit Rückgriff auf das neueste stabile Git-Tag, wenn Beta fehlt oder älter ist                                                                |
| `dev`             | Dist-Tag `dev` (selten; die meisten Dev-Benutzer verwenden Git-Installationen)                                                                                                                          | ruft Änderungen ab, führt einen Rebase des Checkouts auf den vorgelagerten Branch `main` durch, erstellt den Build und installiert die globale CLI neu  |

Bei Git-Installationen über `dev` ist der standardmäßige Checkout `~/openclaw` (oder
`$OPENCLAW_HOME/openclaw`, wenn `OPENCLAW_HOME` gesetzt ist); überschreiben Sie ihn mit
`OPENCLAW_GIT_DIR`.

<Tip>
Um Stable und Dev parallel zu verwenden, nutzen Sie zwei separate Checkouts und verweisen Sie jedes Gateway auf seinen eigenen Checkout.
</Tip>

## Einmalig eine Version oder ein Tag auswählen

Verwenden Sie `--tag`, um für ein einzelnes Update ein bestimmtes Dist-Tag, eine Version oder eine Paketspezifikation
auszuwählen, **ohne** den gespeicherten Kanal zu ändern:

```bash
# Eine bestimmte Version installieren
openclaw update --tag 2026.4.1-beta.1

# Vom Beta-Dist-Tag installieren (einmalig, wird nicht gespeichert)
openclaw update --tag beta

# Zum fortlaufend aktualisierten GitHub-Main-Checkout wechseln (dauerhaft)
openclaw update --channel dev

# Eine bestimmte npm-Paketspezifikation installieren
openclaw update --tag openclaw@2026.4.1-beta.1

# Einmalig von GitHub Main installieren, ohne den Kanal zu speichern
openclaw update --tag main
```

Hinweise:

- `--tag` gilt **nur für Paketinstallationen (npm)**; Git-Installationen ignorieren die Option.
- Das Tag wird nicht gespeichert; das nächste `openclaw update` verwendet den konfigurierten
  Kanal.
- `--tag main` wird für diesen einen Durchlauf der npm-kompatiblen Spezifikation `github:openclaw/openclaw#main`
  zugeordnet. Verwenden Sie für eine dauerhaft fortlaufend aktualisierte `main`-Installation
  `openclaw update --channel dev` (Paketinstallationen wechseln zu einem Git-Checkout)
  oder installieren Sie mit der Git-Methode des Installationsprogramms neu:
  `curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git --version main`.
  Der npm-Installationspfad lehnt GitHub-/Git-Quellziele vollständig ab und verweist
  Sie stattdessen auf die Git-Methode.
- Downgrade-Schutz: Wenn die Zielversion älter als die aktuelle
  Version ist, fordert OpenClaw eine Bestätigung an (mit `--yes` überspringen).
- Extended-Stable verwendet stets sein verifiziertes, exaktes Paketziel. Es ist kein
  einmaliger Alias für `--tag extended-stable`, und `--tag` kann nicht mit
  einem wirksamen Extended-Stable-Kanal kombiniert werden.
- `--channel beta` unterscheidet sich von `--tag beta`: Der Kanalablauf kann auf
  Stable/Latest zurückfallen, wenn Beta fehlt oder älter ist, während `--tag beta` für diesen einen
  Durchlauf stets direkt das Dist-Tag `beta` auswählt.

## Probelauf

Zeigen Sie eine Vorschau der Aktionen von `openclaw update` an, ohne Änderungen vorzunehmen:

```bash
openclaw update --dry-run
openclaw update --channel beta --dry-run
openclaw update --tag 2026.4.1-beta.1 --dry-run
openclaw update --dry-run --json
```

Der Probelauf meldet den wirksamen Kanal, die Zielversion, die geplanten Aktionen
und ob eine Downgrade-Bestätigung erforderlich wäre.

## Plugins und Kanäle

Das Wechseln von Kanälen mit `openclaw update` synchronisiert auch die Plugin-Quellen:

- `dev` stellt installierte Plugins mit einem gebündelten Gegenstück wieder auf
  ihre gebündelte Quelle (Git-Checkout) um.
- `stable` und `beta` stellen über npm oder ClawHub installierte Plugin-
  Pakete wieder her.
- `extended-stable` löst infrage kommende offizielle npm-Plugins mit bloßer/standardmäßiger
  oder `latest`-Vorgabe auf die exakt installierte Kernversion auf. Zur Laufzeit werden keine
  Plugin-Tags vom Typ `@extended-stable` abgefragt.
- Über npm installierte Plugins werden aktualisiert, nachdem das Kern-Update abgeschlossen ist.

## Aktuellen Status prüfen

```bash
openclaw update status
```

Zeigt den aktiven Kanal (einschließlich der Quelle, die ihn bestimmt hat: Konfiguration, Git-Tag,
Git-Branch, installierte Version oder Standardwert), die Installationsart (Git oder Paket),
die aktuelle Version und die Verfügbarkeit von Updates an.

## Bewährte Verfahren für Tags

- Versehen Sie Releases, auf denen Git-Checkouts landen sollen, mit Tags: `vYYYY.M.PATCH` für Stable,
  `vYYYY.M.PATCH-beta.N` für Beta. Benannte Vorabversionssuffixe wie
  `-alpha.N`, `-rc.N` und `-next.N` sind keine Stable- oder Beta-Ziele.
- Veraltete numerische Stable-Tags wie `vYYYY.M.PATCH-1` und `v1.0.1-1` werden aus
  Kompatibilitätsgründen weiterhin als stabile Git-Tags erkannt.
- `vYYYY.M.PATCH.beta.N` (durch Punkte getrennt) wird aus Kompatibilitätsgründen ebenfalls erkannt;
  bevorzugen Sie `-beta.N`.
- Halten Sie Tags unveränderlich: Verschieben oder verwenden Sie ein Tag niemals erneut.
- npm-Dist-Tags bleiben die maßgebliche Quelle für npm-Installationen:
  - `latest` -> Stable
  - `extended-stable` -> nachlaufendes Paket-Release für unterstützte Monate
  - `beta` -> Kandidaten-Build oder zuerst als Beta veröffentlichter stabiler Build
  - `dev` -> Main-Snapshot (optional)

## Verfügbarkeit der macOS-App

Beta- und Dev-Builds enthalten möglicherweise **keine** Veröffentlichung der macOS-App. Das ist unproblematisch:

- Das Git-Tag und das npm-Dist-Tag können dennoch unabhängig veröffentlicht werden.
- Weisen Sie in den Versionshinweisen oder im Changelog auf „kein macOS-Build für diese Beta-Version“ hin.

## Verwandte Themen

- [Aktualisierung](/de/install/updating)
- [Interna des Installationsprogramms](/de/install/installer)
