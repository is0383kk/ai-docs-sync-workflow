---
read_when:
    - Documentatie schrijven die tokens, API-sleutels of fragmenten van aanmeldgegevens bevat
    - Voorbeelden bijwerken die mogelijk door tooling voor geheimdetectie worden gescand
summary: Conventies voor geheimscannerv veilige tijdelijke aanduidingen in documentatie en voorbeelden
title: Conventies voor tijdelijke aanduidingen van geheimen
x-i18n:
    generated_at: "2026-07-27T06:08:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0864f0fcc6fb1e4a3147b4b2ce0aac475437a19d694f3d059374782428c7f248
    source_path: reference/secret-placeholder-conventions.md
    workflow: 16
---

# Conventies voor tijdelijke aanduidingen van geheimen

Gebruik tijdelijke aanduidingen die leesbaar zijn voor mensen, maar niet op echte geheimen lijken.

## Aanbevolen stijl

- Geef de voorkeur aan beschrijvende waarden zoals `example-openai-key-not-real` of `example-discord-bot-token`.
- Geef voor shell-fragmenten de voorkeur aan `${OPENAI_API_KEY}` boven inline tekenreeksen die op tokens lijken.
- Houd voorbeelden duidelijk fictief en afgestemd op het doel (provider, kanaal, authenticatietype).

## Vermijd deze patronen in documentatie

- Letterlijke kop- of voettekst van een PEM-privésleutel.
- Voorvoegsels die op actieve inloggegevens lijken, bijvoorbeeld `sk-...`, `xoxb-...`, `AKIA...`.
- Realistisch ogende bearer-tokens die uit runtimelogboeken zijn gekopieerd.

## Voorbeeld

```bash
# Goed
export OPENAI_API_KEY="example-openai-key-not-real"

# Beter (wanneer het document over het koppelen van omgevingsvariabelen gaat)
export OPENAI_API_KEY="${OPENAI_API_KEY}"
```
