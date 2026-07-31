---
read_when:
    - Je installeert, configureert of controleert de anthropic-vertex-plugin
summary: OpenClaw Anthropic Vertex-providerplugin voor Claude-modellen op Google Vertex AI.
title: Anthropic Vertex-Plugin
x-i18n:
    generated_at: "2026-07-27T05:08:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bd73b80b4e49a85cd6b1d8e47df6bf8d2d791c36a677124112f299027bfd9af5
    source_path: plugins/reference/anthropic-vertex.md
    workflow: 16
---

# Anthropic Vertex-plugin

OpenClaw Anthropic Vertex-providerplugin voor Claude-modellen op Google Vertex AI.

## Distributie

- Pakket: `@openclaw/anthropic-vertex-provider`
- Installatieroute: npm; ClawHub

## Oppervlak

providers: `anthropic-vertex`

<!-- openclaw-plugin-reference:manual-start -->

## Claude Fable 5

Gebruik `anthropic-vertex/claude-fable-5` waar het model beschikbaar is in jouw Google Cloud-regio.
Fable 5 gebruikt altijd adaptief denken en gebruikt standaard `high` als inspanningsniveau. `/think off` en
`/think minimal` gebruiken `low` als inspanningsniveau, omdat het model het uitschakelen van denken niet ondersteunt.

## Claude Sonnet 5

Gebruik `anthropic-vertex/claude-sonnet-5` met het `global`-, `us`- of `eu`-
endpoint van Vertex. Sonnet 5 gebruikt standaard adaptief denken met `high` als inspanningsniveau en ondersteunt
`/think off` of de systeemeigen `/think xhigh|max`-niveaus. OpenClaw publiceert automatisch het
contextvenster van 1.000.000 tokens en de uitvoerlimiet van 128.000 tokens.

De catalogusprijzen volgen tot en met 31 augustus 2026 het wereldwijde introductietarief van Vertex van `$2/$10` per
miljoen invoer-/uitvoertokens en vanaf 1 september `$3/$15`. De multiregionale endpoints
`us` en `eu` gebruiken de door Vertex gedocumenteerde
toeslag van 10%.

<!-- openclaw-plugin-reference:manual-end -->
