---
read_when:
    - Onderzoek naar een crash van de tsx/esbuild-loader die een ontbrekende __name-helper vermeldt
summary: Historische crash in Node + tsx met ‘__name is not a function’ en de oorzaak ervan
title: Node + tsx-crash
x-i18n:
    generated_at: "2026-07-27T04:58:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 97d2f62d24860cee65753027ba84c14c8d4ffb910ee17bb0032cf0409c427589
    source_path: debug/node-issue.md
    workflow: 16
---

# Crash in Node + tsx: "\_\_name is not a function"

## Status

Opgelost. Deze crash is niet reproduceerbaar met de huidige `tsx`-versie die is vastgezet in
`package.json` (`4.22.3`), noch met actuele Node-releases. Dit blijft hier staan voor het geval een
toekomstige upgrade van `tsx`/esbuild het probleem opnieuw introduceert.

## Oorspronkelijk symptoom

Het uitvoeren van OpenClaw-ontwikkelscripts via `tsx` mislukte bij het opstarten met:

```text
[openclaw] Starten van CLI mislukt: TypeError: __name is not a function
    at createSubsystemLogger (src/logging/subsystem.ts)
    at <caller> (src/agents/auth-profiles/constants.ts)
```

Regelnummers zijn weggelaten; beide bestanden zijn sinds de oorspronkelijke crash gewijzigd
en de specifieke regels komen niet meer overeen.

Dit trad op nadat ontwikkelscripts waren overgeschakeld van Bun naar `tsx` (`2871657e`,
2026-01-06) om Bun optioneel te maken. Het equivalente pad op basis van Bun crashte niet.
Het probleem werd oorspronkelijk waargenomen met Node v25.3.0 op macOS; andere platforms waarop
Node 25 draait, werden waarschijnlijk ook als getroffen beschouwd.

## Oorzaak

`tsx` transformeert TS/ESM via esbuild met `keepNames: true` hardgecodeerd in
de transformatieopties. Door die instelling verpakt esbuild benoemde functie- en klasse-
declaraties in een aanroep van een `__name`-helper, zodat `fn.name` behouden blijft bij minificatie
en bundeling. De crash betekent dat de helper op de aanroeplocatie voor die module
in de getroffen combinatie van `tsx`/Node ontbrak of werd overschaduwd, waardoor `__name(...)`
een fout genereerde in plaats van de verpakte waarde terug te geven.

## Huidige reproductiecontrole

```bash
node --version
pnpm install
node --import tsx src/entry.ts status
```

Minimale geïsoleerde reproductie (laadt alleen de module uit de oorspronkelijke stacktrace):

```bash
node --import tsx scripts/repro/tsx-name-repro.ts
```

Beide opdrachten worden momenteel zonder fouten afgesloten. Als een van beide opnieuw `__name is not a
function` genereert, leg dan vóór het upstream melden de exacte Node-versie, de versie van `tsx`
(`node_modules/tsx/package.json`) en de volledige stacktrace vast.

## Tijdelijke oplossingen (als de crash terugkeert)

- Voer ontwikkelscripts uit met Bun in plaats van `node --import tsx`.
- Voer `pnpm tsgo` uit voor typecontrole en voer vervolgens de gebouwde uitvoer uit in plaats van de
  broncode via `tsx`:

  ```bash
  pnpm tsgo
  node openclaw.mjs status
  ```

- Probeer een andere versie van `tsx` (`pnpm add -D tsx@<version>` is een wijziging
  van een afhankelijkheid en vereist volgens het repositorybeleid goedkeuring) om door middel van bisectie vast te stellen of de meegeleverde
  esbuild-versie de bug opnieuw heeft geïntroduceerd.
- Test met een andere hoofd-/subversie van Node om te bepalen of de fout
  versiespecifiek is.

## Referenties

- [https://esbuild.github.io/api/#keep-names](https://esbuild.github.io/api/#keep-names)
- [https://github.com/evanw/esbuild/issues/1031](https://github.com/evanw/esbuild/issues/1031)

## Gerelateerd

- [Node.js installeren](/nl/install/node)
- [Problemen met de Gateway oplossen](/nl/gateway/troubleshooting)
