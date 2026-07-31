---
read_when:
    - De macOS-ontwikkelomgeving instellen
summary: Installatiehandleiding voor ontwikkelaars die aan de OpenClaw-app voor macOS werken
title: macOS-ontwikkelomgeving instellen
x-i18n:
    generated_at: "2026-07-27T06:21:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ff72bb449e70b94b8a13504414955ab7fe411a674b65e670939484a5863b5f48
    source_path: platforms/mac/dev-setup.md
    workflow: 16
---

# macOS-ontwikkelaarsconfiguratie

Bouw de OpenClaw-macOS-app vanuit de broncode en voer deze uit.

## Vereisten

- **Xcode 26.2+** (Swift 6.2-toolchain), op de nieuwste beschikbare macOS-versie in
  Software Update.
- **Node.js 24.15+ en pnpm** voor de Gateway, CLI en pakketteringsscripts. Node
  22.22.3+ werkt ook.

## 1. Afhankelijkheden installeren

```bash
pnpm install
```

## 2. De app bouwen en verpakken

```bash
./scripts/package-mac-app.sh
```

Levert `dist/OpenClaw.app` op. Zonder een Apple Developer ID-certificaat valt het
script terug op ad-hocondertekening.

Zie voor ontwikkeluitvoermodi, ondertekeningsvlaggen en probleemoplossing voor Team ID
[apps/macos/README.md](https://github.com/openclaw/openclaw/blob/main/apps/macos/README.md).
Snelle ontwikkelcyclus vanuit de hoofdmap van de repository: `scripts/restart-mac.sh` (voeg `--no-sign` toe voor
ad-hocondertekening; TCC-machtigingen blijven niet behouden met `--no-sign`).

<Note>
Ad-hocondertekende apps kunnen beveiligingsmeldingen activeren. Als de app
onmiddellijk crasht met "Abort trap 6", zie [Probleemoplossing](#troubleshooting).
</Note>

## 3. De CLI en Gateway installeren

De verpakte app bevat het canonieke `scripts/install-cli.sh`-installatieprogramma. Kies bij een
nieuw profiel **This Mac** tijdens de onboarding; de app installeert de
bijpassende CLI en runtime in de gebruikersruimte voordat de Gateway-wizard wordt gestart.

Installeer voor handmatig ontwikkelherstel zelf de bijpassende CLI:

```bash
npm install -g openclaw@<version>
```

`pnpm add -g openclaw@<version>` en `bun add -g openclaw@<version>`
werken ook. Node blijft de aanbevolen runtime voor de Gateway zelf.

## Probleemoplossing

### Bouwen mislukt: toolchain- of SDK-versie komt niet overeen

Voor het bouwen van de macOS-app zijn de nieuwste macOS-SDK en de Swift 6.2-toolchain
(Xcode 26.2+) vereist.

```bash
xcodebuild -version
xcrun swift --version
```

Als de versies niet overeenkomen, werk je macOS/Xcode bij en voer je de build opnieuw uit.

### App crasht bij het verlenen van toestemming

Als de app crasht wanneer je toegang tot **Speech Recognition** of
**Microphone** probeert toe te staan, kan de oorzaak een beschadigde TCC-cache of niet-overeenkomende ondertekening zijn.

1. Stel de TCC-machtigingen voor de bundel-ID voor foutopsporing opnieuw in:

   ```bash
   tccutil reset All ai.openclaw.mac.debug
   ```

2. Als dat mislukt, wijzig dan tijdelijk `BUNDLE_ID` in
   [`scripts/package-mac-app.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-app.sh)
   om macOS met een schone lei te laten beginnen.

### Gateway blijft voor onbepaalde tijd op "Starting..." staan

Controleer of een zombieproces de poort bezet:

```bash
openclaw gateway status
openclaw gateway stop

# Als je geen LaunchAgent gebruikt (ontwikkelmodus/handmatige uitvoeringen), zoek je het luisterende proces:
lsof -nP -iTCP:18789 -sTCP:LISTEN
```

Als een handmatige uitvoering de poort bezet, stop je deze (Ctrl+C) of beëindig je als
laatste redmiddel de hierboven gevonden PID.

## Gerelateerd

- [macOS-app](/nl/platforms/macos)
- [Installatieoverzicht](/nl/install)
