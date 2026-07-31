---
read_when:
    - Veröffentlichen eines Skills oder Plugins
    - Fehler beim Eigentümer- oder Paketbereich debuggen
    - Veröffentlichungsverhalten für Benutzeroberfläche, CLI oder Backend hinzufügen
summary: So funktioniert die Veröffentlichung auf ClawHub für Skills, Plugins, Eigentümer, Scopes, Releases und Reviews.
x-i18n:
    generated_at: "2026-07-26T18:21:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 582dffaf4429e9f24d7c38f2809cc7dc05f8471e4ae2f9c6be60153cc8604e3f
    source_path: clawhub/publishing.md
    workflow: 16
---

# Veröffentlichen

Beim Veröffentlichen wird ein Skills-Ordner oder Plugin-Paket unter dem von Ihnen
ausgewählten Eigentümer an ClawHub gesendet. ClawHub prüft, ob Ihr Token für diesen
Eigentümer veröffentlichen darf, validiert Metadaten, Namen, Version, Dateien und
Quellinformationen, speichert anschließend das Release und startet automatisierte
Sicherheitsprüfungen.

Wenn die Validierung fehlschlägt, wird nichts veröffentlicht. Neue Releases können
außerdem von den regulären Installations- und Download-Oberflächen ausgeschlossen
bleiben, bis die Überprüfung abgeschlossen ist.

## Skills

Der einfachste Veröffentlichungsweg führt über die CLI. Melden Sie sich an und
veröffentlichen Sie anschließend einen lokalen Skills-Ordner:

```bash
clawhub login
clawhub skill publish ./my-skill \
  --slug my-skill \
  --name "My Skill" \
  --owner <owner>
```

Verwenden Sie `--owner <handle>`, wenn Sie unter einem Organisationseigentümer
veröffentlichen. Lassen Sie die Angabe weg, um als authentifizierter Benutzer zu
veröffentlichen. Unveränderte Inhalte werden beim Veröffentlichen übersprungen.
Ein neuer Skill beginnt bei `1.0.0`, und bei späteren Änderungen wird
automatisch die nächste Patch-Version veröffentlicht. Übergeben Sie
`--version` nur, wenn Sie eine explizite Version benötigen.

Verwenden Sie für Katalog-Repositorys den wiederverwendbaren
[`skill-publish.yml`-Workflow](https://github.com/openclaw/clawhub/blob/main/.github/workflows/skill-publish.yml)
von ClawHub. Er ruft `skill publish` für jeden direkt unter
`root` liegenden Skills-Ordner auf (Standard:
`skills`) oder nur für den als `skill_path` angegebenen Ordner.

```yaml
jobs:
  publish:
    uses: openclaw/clawhub/.github/workflows/skill-publish.yml@main
    with:
      owner: <owner>
      dry_run: false
    secrets:
      clawhub_token: ${{ secrets.CLAWHUB_TOKEN }}
```

Verwenden Sie `dry_run: true`, um neue und geänderte Skills in einer Vorschau
anzuzeigen, ohne sie zu veröffentlichen.

## Plugins

Plugins verwenden Paketnamen im npm-Stil. Paketnamen mit Scope enthalten den
Eigentümer im ersten Teil des Namens:

```text
@owner/package-name
```

Der Scope muss mit dem ausgewählten Veröffentlichungseigentümer übereinstimmen.
Wenn Ihr Paket `@openclaw/dronzer` heißt, kann es nur als `@openclaw`
veröffentlicht werden. Wenn Sie als `@vintageayu` veröffentlichen, benennen
Sie das Paket in `@vintageayu/dronzer` um.

Dadurch wird verhindert, dass ein Paket den Namensraum einer Organisation
beansprucht, über den der Veröffentlichende keine Kontrolle hat.

Wenn Sie der rechtmäßige Eigentümer einer Organisation, Marke, eines Paket-Scopes,
Eigentümer-Handles oder Namensraums sind, der auf ClawHub bereits beansprucht oder
reserviert ist, erstellen Sie ein
[Anliegen zur Beanspruchung einer Organisation/eines Namensraums](https://github.com/openclaw/clawhub/issues/new?template=org-namespace-claim.yml)
mit öffentlichen, nicht vertraulichen Nachweisen. Unter
[Beanspruchung von Organisationen und Namensräumen](/de/clawhub/namespace-claims)
erfahren Sie, welche Angaben erforderlich sind und welche nicht in öffentliche
Anliegen gehören.

### Vor dem Veröffentlichen eines Plugins

- Wählen Sie einen Eigentümer, der dem Paket-Scope entspricht.
- Fügen Sie `openclaw.plugin.json` hinzu. Code-Plugins benötigen außerdem
  `package.json` mit `openclaw.compat.pluginApi` und `openclaw.build.openclawVersion`.
- Um auf der Startseite und den Plugin-Listenseiten ein
  benutzerdefiniertes Plugin-Katalogsymbol anzuzeigen, fügen Sie
  `icon` mit einer beliebigen HTTPS-Bild-URL zu
  `openclaw.plugin.json` hinzu.
- Geben Sie das Quell-Repository und die Metadaten des exakten
  Commits an oder verwenden Sie die CLI aus einem GitHub-basierten Checkout,
  damit sie diese automatisch erkennen kann.
- Führen Sie vor dem Veröffentlichen `clawhub package validate <source>` aus.
  Informationen zur Behebung von Problemen mit Paketen, Manifesten,
  SDK-Importen oder Artefakten finden Sie unter
  [Fehlerbehebung bei der Plugin-Validierung](/de/clawhub/plugin-validation-fixes).
- Führen Sie `clawhub package publish <source> --dry-run` aus, bevor Sie ein Release erstellen.
- Rechnen Sie damit, dass neue Releases von öffentlichen
  Installationsoberflächen ausgeschlossen bleiben, bis automatisierte
  Sicherheitsprüfungen und die Verifizierung abgeschlossen sind.

### Vertrauenswürdiges Veröffentlichen von Paketen

Das Einrichten des vertrauenswürdigen Veröffentlichens von Paketen erfolgt in
zwei Schritten:

1. Veröffentlichen Sie das Paket einmal über das reguläre manuelle
   oder Token-authentifizierte `clawhub package publish`. Dadurch wird der Paketeintrag
   erstellt und festgelegt, welche Paketverwalter die Konfiguration des
   vertrauenswürdigen Veröffentlichenden ändern können.
2. Ein Paketverwalter legt die Konfiguration des vertrauenswürdigen
   Veröffentlichenden für GitHub Actions fest:

```bash
clawhub package trusted-publisher set @owner/package-name \
  --repository owner/repo \
  --workflow-filename package-publish.yml
```

Nachdem die Konfiguration festgelegt wurde, können künftig unterstützte
Veröffentlichungen über GitHub Actions OIDC beziehungsweise vertrauenswürdiges
Veröffentlichen nutzen, ohne ein langlebiges ClawHub-Token im Repository zu
speichern. Das konfigurierte Repository und der Workflow-Dateiname müssen mit
dem OIDC-Claim von GitHub Actions übereinstimmen. Wenn Sie zusätzlich
`--environment <name>` übergeben, muss der Environment-Claim von GitHub Actions
exakt mit diesem Namen übereinstimmen.

ClawHub verifiziert das konfigurierte GitHub-Repository, wenn die Konfiguration
des vertrauenswürdigen Veröffentlichenden festgelegt wird. Öffentliche
Repositorys können anhand öffentlicher GitHub-Metadaten verifiziert werden.
Bei privaten Repositorys benötigt ClawHub Zugriff auf das betreffende
GitHub-Repository, beispielsweise über eine zukünftige Installation der ClawHub
GitHub App oder eine andere autorisierte GitHub-Integration.

Der aktuelle wiederverwendbare Paketveröffentlichungs-Workflow unterstützt
geheimnisfreies vertrauenswürdiges Veröffentlichen für
`workflow_dispatch`-Veröffentlichungen, wenn `id-token: write` verfügbar ist.
Reale Veröffentlichungen durch das Pushen von Tags benötigen weiterhin
`clawhub_token`. Halten Sie daher `CLAWHUB_TOKEN` für Tag-Releases,
Erstveröffentlichungen, nicht vertrauenswürdige Pakete oder
Notfallveröffentlichungen verfügbar.

Prüfen oder entfernen Sie die Konfiguration mit:

```bash
clawhub package trusted-publisher get @owner/package-name
clawhub package trusted-publisher delete @owner/package-name
```

Das Löschen der Konfiguration des vertrauenswürdigen Veröffentlichenden ist der
Rollback-Pfad. Dadurch wird die zukünftige Ausstellung von Tokens für
vertrauenswürdiges Veröffentlichen deaktiviert, bis ein Paketverwalter die
Konfiguration erneut festlegt.

## Häufig gestellte Fragen

### Der Paket-Scope muss mit dem ausgewählten Eigentümer übereinstimmen

Wenn Paket-Scope und ausgewählter Eigentümer nicht übereinstimmen, lehnt ClawHub
die Veröffentlichung ab:

```text
Paket-Scope "@openclaw" muss mit dem ausgewählten Eigentümer "@vintageayu" übereinstimmen.
Veröffentlichen Sie als "@openclaw" oder benennen Sie dieses Paket in "@vintageayu/dronzer" um.
```

Wählen Sie zur Behebung entweder den durch den Paket-Scope angegebenen Eigentümer
oder benennen Sie das Paket so um, dass der Scope mit dem Eigentümer übereinstimmt,
unter dem Sie veröffentlichen können.

Wenn der Paketname bereits den richtigen Scope hat, das Paket jedoch dem falschen
Veröffentlichenden gehört, übertragen Sie stattdessen die Eigentümerschaft:

```sh
clawhub package transfer @opik/opik-openclaw --to opik
```

Verwenden Sie die Übertragung eines Pakets oder Skills nur, wenn Sie sowohl auf
den aktuellen Eigentümer als auch auf den Zielveröffentlichenden administrativen
Zugriff haben. Durch eine Paketübertragung können Sie nicht in einem Scope
veröffentlichen, den Sie nicht verwalten können.

Wenn Sie keinen Zugriff auf den aktuellen Eigentümer haben, aber davon ausgehen,
dass Ihre Organisation, Ihr Projekt oder Ihre Marke der rechtmäßige Eigentümer
des Namensraums ist, erstellen Sie ein
[Anliegen zur Beanspruchung einer Organisation/eines Namensraums](https://github.com/openclaw/clawhub/issues/new?template=org-namespace-claim.yml)
mit öffentlichen, nicht vertraulichen Nachweisen zur Überprüfung durch das
Personal. Lesen Sie vor der Einreichung
[Beanspruchung von Organisationen und Namensräumen](/de/clawhub/namespace-claims).

Dies schützt die Namensräume von Organisationen. Ein Paket namens
`@openclaw/dronzer` beansprucht den Namensraum `@openclaw`, sodass es nur
von Veröffentlichenden mit Zugriff auf den Eigentümer `@openclaw`
veröffentlicht werden kann.
