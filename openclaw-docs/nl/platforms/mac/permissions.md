---
read_when:
    - Ontbrekende of vastgelopen macOS-toestemmingsmeldingen debuggen
    - Beslissen of je toegankelijkheidstoegang verleent aan Node of een CLI-runtime
    - De macOS-app verpakken of ondertekenen
    - Bundle-ID's of installatiepaden van apps wijzigen
summary: Persistentie van macOS-machtigingen (TCC) en ondertekeningsvereisten
title: macOS-machtigingen
x-i18n:
    generated_at: "2026-07-27T05:59:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e561aa641e44fc1e1b95a3db244f31124e4e51d13ae709bee188d86054301e34
    source_path: platforms/mac/permissions.md
    workflow: 16
---

macOS-machtigingstoekenningen zijn kwetsbaar. TCC koppelt een machtigingstoekenning aan de codehandtekening, bundel-ID en locatie op schijf van de app. Als een daarvan verandert, beschouwt macOS de app als nieuw en worden prompts mogelijk verwijderd of verborgen.

## Vereisten voor stabiele machtigingen

- Hetzelfde pad: voer de app uit vanaf een vaste locatie (voor OpenClaw: `dist/OpenClaw.app`).
- Dezelfde bundel-ID: de bundel-ID van OpenClaw is `ai.openclaw.mac`; als je deze wijzigt, ontstaat een nieuwe machtigingsidentiteit.
- Ondertekende app: bij niet-ondertekende of ad-hoc ondertekende builds blijven machtigingen niet behouden.
- Consistente handtekening: gebruik een echt Apple Development- of Developer ID-certificaat, zodat de handtekening bij nieuwe builds stabiel blijft.

Ad-hoc handtekeningen genereren bij elke build een nieuwe identiteit. macOS vergeet eerdere toekenningen en prompts kunnen volledig verdwijnen totdat de verouderde vermeldingen zijn gewist.

## Toegankelijkheidstoekenningen voor Node- en CLI-runtimes

Geef Toegankelijkheid bij voorkeur aan OpenClaw.app, Peekaboo.app of een ander ondertekend hulpprogramma met een eigen bundel-ID, in plaats van aan een algemeen `node`-binair bestand.

macOS TCC kent Toegankelijkheid toe aan de code-identiteit van het proces dat het waarneemt. Als door een Homebrew-, nvm-, pnpm- of npm-workflow een gedeeld uitvoerbaar bestand `node` Toegankelijkheid krijgt, kan elk JavaScript-pakket dat via hetzelfde uitvoerbare bestand wordt gestart, machtigingen voor GUI-automatisering erven.

Beschouw een vermelding `node` in Systeeminstellingen als een brede machtiging voor die Node-runtime, niet als een machtiging voor één npm-pakket. Geef geen Toegankelijkheid aan `node`, tenzij je elk script en pakket vertrouwt dat via precies die Node-installatie wordt gestart.

Goedkeuring voor Toegankelijkheid schakelt het delen van activiteit niet in. **Settings -> Permissions -> Active computer detection** is een afzonderlijke functie die standaard is uitgeschakeld en waarmee een begrensde duur van inactiviteit met je Gateway wordt gedeeld. Als je deze functie uitschakelt, wordt de bewaarde activiteit gewist zonder Toegankelijkheid in te trekken of de Node te ontkoppelen.

Als je per ongeluk Toegankelijkheid aan `node` hebt toegekend, verwijder je die vermelding via Systeeminstellingen -> Privacy en beveiliging -> Toegankelijkheid. Ken de machtiging vervolgens toe aan de ondertekende app of het ondertekende hulpprogramma dat verantwoordelijk moet zijn voor UI-automatisering.

## Herstelchecklist wanneer prompts verdwijnen

1. Sluit de app.
2. Verwijder de appvermelding in Systeeminstellingen -> Privacy en beveiliging.
3. Start de app opnieuw vanaf hetzelfde pad en ken de machtigingen opnieuw toe.
4. Als de prompt nog steeds niet verschijnt, stel je de TCC-vermeldingen opnieuw in met `tccutil` en probeer je het opnieuw.
5. Sommige machtigingen verschijnen pas opnieuw nadat macOS volledig opnieuw is opgestart.

Voorbeelden van resets (met de bundel-ID van OpenClaw, `ai.openclaw.mac`):

```bash
sudo tccutil reset Accessibility ai.openclaw.mac
sudo tccutil reset ScreenCapture ai.openclaw.mac
sudo tccutil reset AppleEvents
```

## Machtigingen voor bestanden en mappen (Bureaublad/Documenten/Downloads)

macOS kan ook de toegang tot Bureaublad, Documenten en Downloads beperken voor terminal- en achtergrondprocessen. Als het lezen van bestanden of het weergeven van mapinhoud blijft hangen, geef je toegang aan dezelfde procescontext die de bestandsbewerkingen uitvoert (bijvoorbeeld Terminal/iTerm, een door LaunchAgent gestarte app of een SSH-proces).

Tijdelijke oplossing: verplaats bestanden naar de OpenClaw-werkruimte (`~/.openclaw/workspace`) als je machtigingen per map wilt vermijden.

Als je machtigingen test, moet je altijd met een echt certificaat ondertekenen. Ad-hoc builds zijn alleen geschikt voor snelle lokale uitvoeringen waarbij machtigingen niet van belang zijn.

## Gerelateerd

- [macOS-app](/nl/platforms/macos)
- [macOS-ondertekening](/nl/platforms/mac/signing)
