---
read_when:
    - Je installeert, configureert of controleert de diffs-language-pack-plugin.
summary: Voegt syntaxisaccentuering toe voor talen die niet in de standaardset van de diffviewer staan.
title: Plugin voor het Diffs-taalpakket
x-i18n:
    generated_at: "2026-07-27T06:27:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e63f896b937be27bd00a7a728b128ec0d1d5eee91d6f1023862274e32afe5db1
    source_path: plugins/reference/diffs-language-pack.md
    workflow: 16
---

# Plugin voor het Diffs-taalpakket

Voegt syntaxisaccentuering toe voor talen buiten de standaardset van de diffs-viewer.

## Distributie

- Pakket: `@openclaw/diffs-language-pack`
- Installatieroute: npm; ClawHub: `clawhub:@openclaw/diffs-language-pack`

## Oppervlak

Plugin

<!-- openclaw-plugin-reference:manual-start -->

## Toegevoegde talen

De basis-Plugin `diffs` accentueert al de gangbare talen die in [Diffs](/nl/tools/diffs) worden beschreven. Installeer dit taalpakket als je syntaxisaccentuering wilt voor een bredere verzameling door Shiki ondersteunde talen. Als het pakket niet is geïnstalleerd, worden die bestanden nog steeds als leesbare platte tekst weergegeven.

Voorbeelden zijn Astro, Vue, Svelte, MDX, GraphQL, Terraform/HCL, Nix, Clojure, Elixir, Haskell, OCaml, Scala, Zig, Solidity, Verilog/VHDL, Fortran, MATLAB, LaTeX, Mermaid, Sass/Less/SCSS, Nginx, Apache, CSV, dotenv, INI en diff-bestanden.

Zie [Shiki-talen](https://shiki.style/languages) voor Shiki's upstreamcatalogus met talen en aliassen.

<!-- openclaw-plugin-reference:manual-end -->
