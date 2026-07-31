---
read_when:
    - Een nieuwe OpenClaw-agentsessie starten
    - Standaard-Skills inschakelen of controleren
summary: Standaardinstructies voor OpenClaw-agenten en overzicht van Skills voor de configuratie van de persoonlijke assistent
title: Standaard-AGENTS.md
x-i18n:
    generated_at: "2026-07-27T05:48:39Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 645342f8c6e2805135817cf4bbc2c8bd1d57066054ed671eda93876b2762ffb1
    source_path: reference/AGENTS.default.md
    workflow: 16
---

## Eerste uitvoering (aanbevolen)

OpenClaw-agenten gebruiken een werkruimtemap. Standaard: `~/.openclaw/workspace` (configureerbaar via `agents.defaults.workspace`, ondersteunt `~`).

1. Maak de werkruimte:

```bash
mkdir -p ~/.openclaw/workspace
```

2. Kopieer de standaardwerkruimtesjablonen ernaartoe:

```bash
cp docs/reference/templates/AGENTS.md ~/.openclaw/workspace/AGENTS.md
cp docs/reference/templates/SOUL.md ~/.openclaw/workspace/SOUL.md
cp docs/reference/templates/TOOLS.md ~/.openclaw/workspace/TOOLS.md
```

3. Optioneel: gebruik de lijst met persoonlijke-assistentvaardigheden van dit bestand in plaats van de algemene sjabloon:

```bash
cp docs/reference/AGENTS.default.md ~/.openclaw/workspace/AGENTS.md
```

4. Optioneel: verwijs naar een andere werkruimte:

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
}
```

## Standaardinstellingen voor veiligheid

- Stort geen mappen of geheimen in chats.
- Voer geen destructieve opdrachten uit, tenzij daar expliciet om wordt gevraagd.
- Inspecteer eerst de bestaande toestand voordat je configuratie of planners (crontab, systemd-eenheden, nginx-configuraties, shell-rc-bestanden) wijzigt, en behoud of combineer deze standaard.
- Stuur geen gedeeltelijke of gestreamde antwoorden naar externe berichtenkanalen (alleen definitieve antwoorden).

## Voorcontrole op bestaande oplossingen

Controleer voordat je een aangepast systeem, functie, workflow, hulpmiddel, integratie of automatisering voorstelt of bouwt of er opensourceprojecten, onderhouden bibliotheken, bestaande OpenClaw-plugins of gratis platforms zijn die dit al goed genoeg oplossen. Geef hieraan de voorkeur als ze voldoen. Bouw alleen iets aangepast als de bestaande opties ongeschikt, te duur, niet onderhouden, onveilig of niet-conform zijn, of als de gebruiker expliciet om maatwerk vraagt. Vermijd aanbevelingen voor betaalde diensten, tenzij de gebruiker expliciet toestemming geeft voor uitgaven. Houd dit beperkt: een voorcontrole, geen onderzoeksopdracht.

## Begin van de sessie (vereist)

- Lees `SOUL.md`, `USER.md` en vandaag+gisteren in `memory/` voordat je antwoordt.
- Lees `MEMORY.md` indien aanwezig.

## Ziel (vereist)

- `SOUL.md` definieert identiteit, toon en grenzen. Houd dit actueel.
- Vertel het de gebruiker als je `SOUL.md` wijzigt.
- Je bent elke sessie een nieuwe instantie; de continuïteit bevindt zich in deze bestanden.

## Gedeelde omgevingen (aanbevolen)

- Je bent niet de stem van de gebruiker; wees voorzichtig in groepschats of openbare kanalen.
- Deel geen privégegevens, contactgegevens of interne notities.

## Geheugensysteem (aanbevolen)

- Daglogboek: `memory/YYYY-MM-DD.md` (maak indien nodig `memory/`).
- Langetermijngeheugen: `MEMORY.md` voor blijvende feiten, voorkeuren en beslissingen.
- De kleine letters in `memory.md` dienen alleen als invoer voor verouderd herstel; bewaar niet opzettelijk beide hoofdbestanden.
- Lees bij het begin van een sessie vandaag + gisteren + `MEMORY.md` indien aanwezig.
- Lees geheugenbestanden voordat je erin schrijft; schrijf alleen concrete updates, nooit lege tijdelijke aanduidingen.
- Leg vast: beslissingen, voorkeuren, beperkingen en openstaande punten.
- Vermijd geheimen, tenzij daar expliciet om wordt gevraagd.

## Hulpmiddelen en Skills

- Hulpmiddelen bevinden zich in Skills; volg de `SKILL.md` van elke Skill wanneer je die nodig hebt.
- Bewaar omgevingsspecifieke notities in `TOOLS.md` (notities voor Skills).

## Back-uptip (aanbevolen)

Beschouw deze werkruimte als het geheugen van de assistent: maak er een git-repository van (bij voorkeur privé), zodat van `AGENTS.md` en geheugenbestanden een back-up wordt gemaakt.

```bash
cd ~/.openclaw/workspace
git init
git add AGENTS.md
git commit -m "Werkruimte toevoegen"
# Optioneel: voeg een privé-remote toe en push
```

## Wat OpenClaw doet

- Voert een Gateway voor berichtenkanalen uit (WhatsApp, Telegram, Discord, Signal, iMessage, Slack en meer), plus een ingebouwde agent, zodat de assistent chats kan lezen en schrijven, context kan ophalen en Skills kan uitvoeren via de hostmachine.
- De macOS-app beheert machtigingen (schermopname, meldingen, microfoon) en stelt de `openclaw` CLI beschikbaar via het meegeleverde binaire bestand.
- Directe chats worden standaard samengevoegd in de `main`-sessie van de agent; groepen en kanalen/ruimten krijgen hun eigen sessiesleutels. Zie [Kanaalroutering](/nl/channels/channel-routing) voor de exacte sleutelindelingen. Heartbeats houden achtergrondtaken actief.

## Kern-Skills (inschakelen via Settings → Skills)

Voorbeeldlijst voor een werkruimte voor een persoonlijke assistent; vervang deze door de Skills die bij jouw configuratie passen.

- **mcporter** - runtime/CLI voor toolservers om externe Skill-backends te beheren.
- **Peekaboo** - snelle macOS-schermafbeeldingen met optionele beeldanalyse door AI.
- **camsnap** - leg frames, clips of bewegingsmeldingen vast van RTSP-/ONVIF-beveiligingscamera's.
- **oracle** - agent-CLI die geschikt is voor OpenAI, met het opnieuw afspelen van sessies en browserbesturing.
- **eightctl** - regel je slaap vanaf de terminal.
- **imsg** - verstuur, lees en stream iMessage en sms.
- **wacli** - WhatsApp-CLI: synchroniseren, zoeken, versturen.
- **discord** - Discord-acties: reageren, stickers, peilingen. Gebruik `user:<id>`- of `channel:<id>`-doelen (losse numerieke id's zijn dubbelzinnig).
- **gog** - CLI voor Google Suite: Gmail, Calendar, Drive, Contacts.
- **spotify-player** - Spotify-client voor de terminal om afspelen te zoeken, in de wachtrij te zetten en te bedienen.
- **sag** - ElevenLabs-spraak met een macOS-achtige gebruikservaring voor say; streamt standaard naar luidsprekers.
- **Sonos CLI** - bedien Sonos-luidsprekers (detectie/status/afspelen/volume/groepering) vanuit scripts.
- **blucli** - speel BluOS-spelers af, groepeer ze en automatiseer ze vanuit scripts.
- **OpenHue CLI** - bedien Philips Hue-verlichting voor scènes en automatiseringen.
- **OpenAI Whisper** - lokale spraak-naar-tekst voor snel dicteren en transcripties van voicemail.
- **Gemini CLI** - Google Gemini-modellen vanuit de terminal voor snelle vragen en antwoorden.
- **agent-tools** - verzameling hulpprogramma's voor automatiseringen en hulpscripts.

## Gebruiksnotities

- Geef voor scripts de voorkeur aan de `openclaw` CLI; de desktop-app handelt machtigingen af.
- Voer installaties uit via het tabblad Skills; de installatieknop is verborgen zodra een vereist binair bestand al aanwezig is.
- Houd Heartbeats ingeschakeld, zodat de assistent herinneringen kan plannen, postvakken kan bewaken en cameraopnamen kan activeren.
- De Canvas-gebruikersinterface wordt schermvullend uitgevoerd met native overlays. Plaats geen essentiële bedieningselementen aan de randen linksboven, rechtsboven of onderaan; voeg expliciete marges aan de lay-out toe in plaats van te vertrouwen op safe-area-insets.
- Gebruik voor browsergestuurde verificatie de `openclaw browser` CLI (meegeleverde `browser`-Plugin) met het door OpenClaw beheerde Chrome-/Brave-/Edge-/Chromium-profiel.
- Beheren: `status`, `doctor [--deep]`, `start [--headless]`, `stop`, `tabs`, `tab [new|select|close]`, `open <url>`, `focus <id>`, `close <id>`.
- Inspecteren: `screenshot [--full-page|--ref|--labels]`, `snapshot [--format ai|aria|--interactive|--efficient]`, `console`, `errors`, `requests`, `pdf`, `responsebody`.
- Handelen: `navigate`, `click <ref>`, `type <ref> <text>`, `press`, `hover`, `drag`, `select`, `upload`, `download`, `fill`, `dialog`, `wait`, `evaluate --fn <js>`, `highlight`. Voor acties is een `ref` uit `snapshot` vereist (CSS-selectors worden niet geaccepteerd voor acties); gebruik `evaluate` wanneer je op `document.querySelector` gebaseerde doelbepaling nodig hebt.
- Voeg `--json` toe voor machineleesbare uitvoer bij elke inspectieopdracht.

## Gerelateerd

- [Agentwerkruimte](/nl/concepts/agent-workspace)
- [Agentruntime](/nl/concepts/agent)
- [Kanaalroutering](/nl/channels/channel-routing)
