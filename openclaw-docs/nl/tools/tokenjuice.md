---
read_when:
    - Je wilt kortere `exec`- of `bash`-toolresultaten in OpenClaw
    - Je wilt de Tokenjuice-plugin installeren of inschakelen
    - Je moet begrijpen wat Tokenjuice verandert en wat het onbewerkt laat staan
summary: Comprimeer rumoerige resultaten van exec- en bash-tools met de optionele Tokenjuice-plugin
title: Tokenjuice
x-i18n:
    generated_at: "2026-07-27T05:38:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 96b110563a2600429dd9f0d38997cf7cc5ae4952b7f146a6ab64c96f2f202440
    source_path: tools/tokenjuice.md
    workflow: 16
---

`tokenjuice` is een optionele externe plugin die uitgebreide toolresultaten van `exec` en `bash`
compact maakt nadat de opdracht al is uitgevoerd.

De plugin wijzigt de geretourneerde `tool_result`, niet de opdracht zelf. Tokenjuice
herschrijft geen shellinvoer, voert opdrachten niet opnieuw uit en wijzigt geen afsluitcodes.

Dit geldt momenteel voor in OpenClaw ingesloten uitvoeringen en dynamische OpenClaw-tools in de Codex-
app-serverharness. Tokenjuice haakt in op de middleware voor toolresultaten van OpenClaw en
kort de uitvoer in voordat deze wordt teruggestuurd naar de actieve harnesssessie.

## De plugin inschakelen

Eenmalig installeren:

```bash
openclaw plugins install clawhub:@openclaw/tokenjuice
```

Daarna inschakelen:

```bash
openclaw config set plugins.entries.tokenjuice.enabled true
```

Equivalent:

```bash
openclaw plugins enable tokenjuice
```

Als je de configuratie liever rechtstreeks bewerkt:

```json5
{
  plugins: {
    entries: {
      tokenjuice: {
        enabled: true,
      },
    },
  },
}
```

## Wat Tokenjuice wijzigt

- Maakt uitgebreide resultaten van `exec` en `bash` compact voordat ze opnieuw aan de sessie worden doorgegeven.
- Laat de oorspronkelijke uitvoering van de opdracht ongewijzigd.
- Past een beleid voor veilige inventarisatie toe: exacte leesbewerkingen van bestandsinhoud blijven onbewerkt, zelfstandige opdrachten voor repository-inventarisatie kunnen compact worden gemaakt en onveilige gemengde opdrachtreeksen blijven onbewerkt.
- Blijft optioneel: schakel de plugin uit als je overal de letterlijke uitvoer wilt.

## Controleren of de plugin werkt

1. Schakel de plugin in.
2. Start een sessie die `exec` kan aanroepen.
3. Voer een uitgebreide opdracht uit, zoals `git status`.
4. Controleer of het geretourneerde toolresultaat korter en beter gestructureerd is dan de onbewerkte shelluitvoer.

## De plugin uitschakelen

```bash
openclaw config set plugins.entries.tokenjuice.enabled false
```

Of:

```bash
openclaw plugins disable tokenjuice
```

## Gerelateerd

- [Exec-tool](/nl/tools/exec)
- [Denkniveaus](/nl/tools/thinking)
- [Contextengine](/nl/concepts/context-engine)
