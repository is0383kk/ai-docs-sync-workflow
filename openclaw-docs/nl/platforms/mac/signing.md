---
read_when:
    - Mac-foutopsporingsbuilds bouwen of ondertekenen
summary: Ondertekeningsstappen voor macOS-debugbuilds die door verpakkingsscripts worden gegenereerd
title: macOS-ondertekening
x-i18n:
    generated_at: "2026-07-27T05:05:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 406211dadc9293cf7983e75ae7dd98234f9088351234cf06c33df2f63d1b9b97
    source_path: platforms/mac/signing.md
    workflow: 16
---

# Mac-ondertekening (debugbuilds)

[`scripts/package-mac-app.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-app.sh) bouwt en verpakt de app op een vast pad (`dist/OpenClaw.app`) en roept vervolgens [`scripts/codesign-mac-app.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/codesign-mac-app.sh) aan om deze te ondertekenen. TCC-machtigingen zijn gekoppeld aan de bundel-ID en codehandtekening; door beide (en het vaste pad van de app) bij elke nieuwe build gelijk te houden, voorkom je dat macOS de TCC-toestemmingen vergeet (meldingen, toegankelijkheid, schermopname, microfoon, spraak).

- De debugbundel-ID is standaard `ai.openclaw.mac.debug` (overschrijf met `BUNDLE_ID=...`).
- Node: `>=22.22.3 <23`, `>=24.15.0 <25` of `>=25.9.0` (repo `package.json` `engines`). Het verpakkingsprogramma bouwt ook de bedieningsinterface (`pnpm ui:build`).
- Vereist standaard een echte ondertekeningsidentiteit; het ondertekeningsscript wordt met een fout afgesloten als er geen wordt gevonden en `ALLOW_ADHOC_SIGNING` niet is ingesteld. Ad-hocondertekening (`SIGN_IDENTITY="-"`) moet expliciet worden ingeschakeld en behoudt TCC-machtigingen niet tussen nieuwe builds. Zie [macOS-machtigingen](/nl/platforms/mac/permissions).
- Leest `SIGN_IDENTITY` uit de omgeving (bijvoorbeeld `export SIGN_IDENTITY="Apple Development: Your Name (TEAMID)"` of een Developer ID Application-certificaat). Zonder deze waarde selecteert `codesign-mac-app.sh` automatisch een identiteit in deze volgorde: Developer ID Application, Apple Distribution, Apple Development en vervolgens de eerste geldige ondertekeningsidentiteit die wordt gevonden.
- `CODESIGN_TIMESTAMP=auto` (standaard) schakelt vertrouwde tijdstempels alleen in voor Developer ID Application-handtekeningen. Stel `on`/`off` in om een van beide opties af te dwingen.
- Voorziet Info.plist van `OpenClawBuildTimestamp` (ISO8601 UTC) en `OpenClawGitCommit` (korte hash, `unknown` indien niet beschikbaar), zodat het tabblad Info de build, Git-gegevens en het debug-/releasekanaal kan weergeven.
- Voert na ondertekening een Team ID-controle uit en mislukt als een Mach-O in de bundel een andere Team ID heeft. Stel `SKIP_TEAM_ID_CHECK=1` in om dit over te slaan.

## Gebruik

```bash
# vanuit de hoofdmap van de repo
scripts/package-mac-app.sh                                                      # selecteert automatisch een identiteit; geeft een fout als er geen wordt gevonden
SIGN_IDENTITY="Developer ID Application: Your Name" scripts/package-mac-app.sh   # echt certificaat
ALLOW_ADHOC_SIGNING=1 scripts/package-mac-app.sh                                 # ad-hoc (machtigingen blijven niet behouden)
SIGN_IDENTITY="-" scripts/package-mac-app.sh                                     # expliciet ad-hoc (zelfde voorbehoud)
DISABLE_LIBRARY_VALIDATION=1 scripts/package-mac-app.sh                          # tijdelijke oplossing, alleen voor ontwikkeling, voor niet-overeenkomende Sparkle Team ID
```

### Opmerking over ad-hocondertekening

`SIGN_IDENTITY="-"` schakelt de Hardened Runtime (`--options runtime`) uit om crashes te voorkomen wanneer de app ingesloten frameworks (zoals Sparkle) laadt die niet dezelfde Team ID hebben. Ad-hochandtekeningen verhinderen ook dat TCC-machtigingen behouden blijven; zie [macOS-machtigingen](/nl/platforms/mac/permissions) voor herstelstappen.

## Buildmetadata voor Info

Het tabblad Info leest `OpenClawBuildTimestamp` en `OpenClawGitCommit` uit Info.plist om de versie, builddatum, Git-commit en of de build DEBUG is (via `#if DEBUG`) weer te geven. Voer het verpakkingsprogramma na codewijzigingen opnieuw uit om deze waarden te vernieuwen.

## Gerelateerd

- [macOS-app](/nl/platforms/macos)
- [macOS-machtigingen](/nl/platforms/mac/permissions)
