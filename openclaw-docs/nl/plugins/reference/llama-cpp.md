---
read_when:
    - Je installeert, configureert of controleert de llama-cpp-plugin
summary: Lokale GGUF-tekstinferentie en embeddings via node-llama-cpp.
title: Llama Cpp-Plugin
x-i18n:
    generated_at: "2026-07-27T05:15:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2756d4b3e00bbe37b4dedec1d54d28bfe6662e8105504317a402293254ce0240
    source_path: plugins/reference/llama-cpp.md
    workflow: 16
---

# Llama Cpp-plugin

Lokale GGUF-tekstinferentie en embeddings via node-llama-cpp.

## Distributie

- Pakket: `@openclaw/llama-cpp-provider`
- Installatieroute: npm; ClawHub

## Oppervlak

providers: `llama-cpp`; contracten: `embeddingProviders`

<!-- openclaw-plugin-reference:manual-start -->

## Standaard tekstmodel

Tijdens de interactieve configuratie biedt OpenClaw Gemma 4 E4B IT Q4_K_M aan als een
gebundelde download van ongeveer 5.0 GB. Dit aanbod vereist ten minste 16 GiB
totaal RAM. Bestaande modellen in de cache worden nog steeds gedetecteerd op kleinere machines.

Als je een ander model wilt gebruiken, stel je `params.modelPath` in op een aangepaste GGUF. Voor aangepaste modellen
geldt de RAM-vereiste voor de gebundelde download niet. Op machines die niet aan de
vereiste voldoen, kun je ook een kleiner model uitvoeren via Ollama of LM Studio, of
een cloudprovider kiezen.

<!-- openclaw-plugin-reference:manual-end -->

## Gerelateerde documentatie

- [llama-cpp](/nl/plugins/llama-cpp)
