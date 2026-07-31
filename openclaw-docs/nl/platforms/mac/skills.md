---
read_when:
    - De macOS-instellingeninterface voor Skills bijwerken
    - Gating of installatiegedrag van Skills wijzigen
summary: macOS-instellingeninterface voor Skills en door de Gateway ondersteunde status
title: Skills (macOS)
x-i18n:
    generated_at: "2026-07-27T05:11:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fd9d8f1190320889029335e008c3605bd4bf0194f83398cedd4ae658fd90065c
    source_path: platforms/mac/skills.md
    workflow: 16
---

De macOS-app maakt OpenClaw-skills beschikbaar via de Gateway; de app verwerkt skills niet lokaal.

## Gegevensbron

- `skills.status` (Gateway) retourneert alle skills, inclusief geschiktheid en ontbrekende vereisten, waaronder blokkeringen via toelatingslijsten voor meegeleverde skills.
- Vereisten zijn afkomstig uit `metadata.openclaw.requires` in elke `SKILL.md`.

## Installatieacties

- `metadata.openclaw.install` definieert installatieopties (brew/node/go/uv/download).
- De app roept `skills.install` aan om installatieprogramma's op de Gateway-host uit te voeren.
- Door de operator beheerde `security.installPolicy` (`enabled`, `targets`, `exec`) kan via de Gateway uitgevoerde installaties van skills blokkeren voordat de metagegevens van het installatieprogramma worden verwerkt. De ingebouwde scan op gevaarlijke code (gebruikt voor installaties van plugins) is niet gekoppeld aan de installatiestroom voor skills.
- Als elke installatieoptie `download` is, toont de Gateway alle downloadkeuzes.
- Anders kiest de Gateway één voorkeursinstallatieprogramma op basis van de huidige installatievoorkeuren (`skills.install.preferBrew`, `skills.install.nodeManager`) en binaire bestanden op de host: eerst Homebrew wanneer `preferBrew` is ingeschakeld en `brew` aanwezig is, daarna `uv`, vervolgens het geconfigureerde Node-beheerprogramma, daarna opnieuw Homebrew indien beschikbaar (zelfs zonder `preferBrew`), vervolgens `go` en ten slotte `download`.
- Labels voor Node-installaties weerspiegelen het geconfigureerde Node-beheerprogramma, waaronder `yarn`.

## Omgevingsvariabelen/API-sleutels

- De app slaat sleutels op in `~/.openclaw/openclaw.json` onder `skills.entries.<skillKey>`.
- `skills.update` past `enabled`, `apiKey` en `env` aan.

## Externe modus

- Installatie- en configuratie-updates vinden plaats op de Gateway-host, niet op de lokale Mac.

## Gerelateerd

- [Skills](/nl/tools/skills)
- [macOS-app](/nl/platforms/macos)
