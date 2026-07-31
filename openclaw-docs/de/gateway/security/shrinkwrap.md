---
read_when:
    - Sie möchten wissen, was npm Shrinkwrap in einem OpenClaw-Release bedeutet
    - Sie prüfen Paket-Lockfiles, Änderungen an Abhängigkeiten oder Risiken für die Software-Lieferkette
    - Sie validieren npm-Pakete des Stammprojekts oder von Plugins vor der Veröffentlichung
summary: Allgemeinverständliche und technische Erläuterung von npm Shrinkwrap in OpenClaw-Releases
title: npm-Shrinkwrap
x-i18n:
    generated_at: "2026-07-26T18:29:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d1e6c0d4541da9220d50cde0b9db064e5a91b81d6562cb16ac697de7d4017098
    source_path: gateway/security/shrinkwrap.md
    workflow: 16
---

OpenClaw-Quell-Checkouts verwenden `pnpm-lock.yaml`. Veröffentlichte OpenClaw-npm-Pakete verwenden `npm-shrinkwrap.json`, die veröffentlichbare Abhängigkeitssperrdatei von npm, sodass Paketinstallationen den während des Releases geprüften Abhängigkeitsgraphen verwenden.

## Warum dies wichtig ist

Shrinkwrap ist ein Beleg für den Abhängigkeitsbaum, der mit einem npm-Paket ausgeliefert wird: Es teilt npm mit, welche exakten transitiven Versionen installiert werden sollen.

| Datei                 | Wo sie relevant ist      | Was sie bedeutet                  |
| --------------------- | ------------------------ | --------------------------------- |
| `pnpm-lock.yaml`      | OpenClaw-Quell-Checkout | Abhängigkeitsgraph der Maintainer |
| `npm-shrinkwrap.json` | Veröffentlichtes npm-Paket | npm-Installationsgraph für Benutzer |
| `package-lock.json`   | Lokale npm-Anwendungen   | Nicht der Veröffentlichungsvertrag von OpenClaw |

Für OpenClaw-Releases bedeutet dies:

- das veröffentlichte Paket fordert npm nicht dazu auf, bei der Installation einen neuen Abhängigkeitsgraphen zu erzeugen;
- Abhängigkeitsänderungen sind überprüfbar, da sie in einem Lockfile-Diff eingehen;
- die Release-Validierung testet denselben Graphen, den Benutzer installieren werden;
- Überraschungen bei der Paketgröße oder nativen Abhängigkeiten werden vor der Veröffentlichung sichtbar.

Shrinkwrap ist keine Sandbox. Es macht eine Abhängigkeit nicht von sich aus sicher und ersetzt weder die Host-Isolierung, `openclaw security audit`, die Paketherkunft noch Installations-Smoke-Tests.

OpenClaw ist ein Gateway, Plugin-Host, Modell-Router und eine Agentenlaufzeit, daher wirkt sich eine Standardinstallation auf Startzeit, Speicherplatzbedarf, Downloads nativer Pakete und die Gefährdung durch die Lieferkette aus. Shrinkwrap bietet der Release-Prüfung eine stabile Grenze: Prüfer sehen Änderungen an transitiven Abhängigkeiten, Validatoren lehnen unerwartete Abweichungen der Sperrdatei ab und Plugin-Pakete führen ihren eigenen gesperrten Abhängigkeitsgraphen mit, statt sich auf das Root-Paket zu verlassen.

## Generieren und prüfen

Das Root-npm-Paket `openclaw`, OpenClaw-eigene npm-Plugin-Pakete (zum Beispiel `@openclaw/discord`) und veröffentlichbare Workspace-Pakete wie [`@openclaw/ai`](/de/reference/openclaw-ai) enthalten bei der Veröffentlichung `npm-shrinkwrap.json`. Workspace-Abhängigkeiten werden im Root-Shrinkwrap ausgelassen, da sie zusammen mit dem Root-Paket veröffentlicht werden; stattdessen fixiert jedes veröffentlichbare Workspace-Paket seinen eigenen transitiven Baum. Geeignete Plugin-Pakete können außerdem mit explizitem `bundledDependencies` veröffentlicht werden, wobei ihre Laufzeitabhängigkeitsdateien im Plugin-Tarball enthalten sind, statt sich ausschließlich auf die Auflösung während der Installation zu verlassen.

```bash
# Alle von Shrinkwrap verwalteten Pakete (Root + veröffentlichbare Plugins)
pnpm deps:shrinkwrap:generate
pnpm deps:shrinkwrap:check

# Nur Root-Paket
pnpm deps:shrinkwrap:root:generate
pnpm deps:shrinkwrap:root:check

# Nur von der aktuellen Änderungsmenge betroffene Pakete
pnpm deps:shrinkwrap:changed:generate
pnpm deps:shrinkwrap:changed:check
```

Der Generator löst das veröffentlichbare Sperrformat von npm auf, lehnt jedoch generierte Paketversionen ab, die nicht bereits in `pnpm-lock.yaml` vorhanden sind. Dadurch bleibt die Prüfgrenze für Alter, Überschreibungen und Patches der pnpm-Abhängigkeiten erhalten.

Prüfen Sie Folgendes als sicherheitskritisch:

- `pnpm-lock.yaml`
- `npm-shrinkwrap.json`
- Abhängigkeitsinhalte gebündelter Plugins
- jeden `package-lock.json`-Diff

OpenClaw-Paketvalidatoren verlangen Shrinkwrap in neuen Root-Paket-Tarballs und lehnen `package-lock.json` für veröffentlichte Pakete ab. Der npm-Veröffentlichungspfad für Plugins prüft das Plugin-lokale Shrinkwrap, installiert paketlokale gebündelte Abhängigkeiten und packt oder veröffentlicht anschließend das Paket.

## Ein veröffentlichtes Paket untersuchen

Root-Paket:

```bash
npm pack openclaw@<version> --json --pack-destination /tmp/openclaw-pack
tar -tf /tmp/openclaw-pack/openclaw-<version>.tgz | grep '^package/npm-shrinkwrap.json$'
```

Plugin-Paket:

```bash
npm pack @openclaw/discord@<version> --json --pack-destination /tmp/openclaw-plugin-pack
tar -tf /tmp/openclaw-plugin-pack/openclaw-discord-<version>.tgz | grep '^package/npm-shrinkwrap.json$'
tar -tf /tmp/openclaw-plugin-pack/openclaw-discord-<version>.tgz | grep '^package/node_modules/'
```

Hintergrund: [npm-shrinkwrap.json](https://docs.npmjs.com/cli/v11/configuring-npm/npm-shrinkwrap-json).
