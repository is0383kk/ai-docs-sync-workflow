---
read_when:
    - Je wilt begrijpen waar je agent 'leeft'
    - Je verwacht dezelfde context, of je nu via Telegram, WhatsApp of het web schrijft
    - Je wilt dat je agent weet wat er in groepen en zijgesprekken gebeurt
summary: 'Eén doorlopend gesprek in al je kanalen: de standaard voor de persoonlijke agent'
title: De hoofdsessie
x-i18n:
    generated_at: "2026-07-27T05:31:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fb77382ebdce269a05a03ab6fa39b44b1e9f1856166f1d9cb79111dccb547f69
    source_path: concepts/main-session.md
    workflow: 16
---

OpenClaw is in de eerste plaats een persoonlijke agent. Standaard komt elk direct bericht dat je
ernaartoe stuurt — vanuit Telegram, WhatsApp, iMessage, Slack-DM's, de webapp, waar dan ook —
terecht in **één doorlopend gesprek**: de hoofdsessie. Vraag iets op je
telefoon, stel een vervolgvraag vanaf je laptop en de agent heeft op beide
apparaten dezelfde context. Er is één brein en hier denkt het.

Onder de motorkap is de hoofdsessie een gewone sessie met de sleutel
`agent:<agentId>:main` (bijvoorbeeld `agent:main:main`). Wat deze sessie bijzonder maakt,
is dat het standaard-DM-bereik alle directe berichten erin samenbrengt en dat
de rest van het systeem deze als de basis van de agent behandelt: Heartbeats activeren de sessie,
achtergrondwerk rapporteert eraan terug en activiteit elders stroomt ernaartoe.

## Start

In de webapp is de hoofdsessie de pagina **Start** — het eerste item in de
zijbalk. De identiteitsrij bovenaan is je agent (klik erop voor het agentmenu);
op Start praat je ermee. Sessies die van het hoofdgesprek worden afgesplitst,
verschijnen onder **Threads**, groepschats onder **Groepen** en
codeer-/CLI-sessies onder **Coderen**.

## Wat er naar de hoofdsessie stroomt

De hoofdsessie is niet alleen een chatlogboek; het is de plek waar de wereld
van je agent samenkomt:

- **Groepsactiviteit.** Groeps- en ruimtesessies blijven geïsoleerd (zie hieronder), maar
  binnen het standaard-DM-bereik houdt de hoofdsessie ze automatisch in de gaten.
  Activiteit wordt verzameld als compacte meldingen — samengevoegd per gesprek, nooit
  één activering per bericht — en de agent ziet ze wanneer deze de volgende keer wordt uitgevoerd: bij
  je volgende bericht of een geplande Heartbeat. De agent kan ook de
  gevolgde sessies lezen, zodat 'wat heb ik gemist in de familiegroep?' werkt.
- **Achtergrondwerk.** Subagents en aangemaakte sessies melden hun resultaten
  terug aan de sessie die ze heeft gestart, zodat werk dat de agent vanuit
  Start heeft geïnitieerd, aan Start wordt teruggemeld.
- **Heartbeats.** Geplande Heartbeats zijn gericht op de hoofdsessie, waardoor
  meldingen in de wachtrij worden opgemerkt, zelfs wanneer je niets hebt geschreven.

## Geheugen tussen resets en gesprekken

Het doorlopende gesprek wordt begrensd door het contextvenster van het model,
dus continuïteit komt van de lagen eromheen:

- `MEMORY.md`, het samengestelde langetermijngeheugen van de agent, wordt in elke
  nieuwe sessie geladen. Dagelijkse notities (`memory/YYYY-MM-DD.md`) zijn op verzoek doorzoekbaar
  en recente notities worden na een `/new` of `/reset` opnieuw in de context geladen. Vóór Compaction
  schrijft de agent blijvende feiten weg naar de dagelijkse notities, zodat lange gesprekken
  deze niet ongemerkt verliezen.
- **Geheugenherinnering tussen gesprekken** laat de agent inhoud uit
  diens andere privésessies ophalen. In persoonlijke configuraties — waarbij de globale
  `session.dmScope` wordt omgezet naar `main` zonder DM-overschrijvingen per binding — is dit
  standaard ingeschakeld; elke geconfigureerde DM-isolatie schakelt dit uit, tenzij je
  het expliciet inschakelt. Zie [Geheugenconfiguratie](/nl/reference/memory-config).

## Een doorlopende sessie met duurzame geschiedenis

De hoofdsessie loopt door via resets en Compaction, in plaats van
het model de volledige geschiedenis in één keer te laten meenemen:

- Standaard vindt er geen automatische reset plaats; Compaction houdt de actieve context
  begrensd en behoudt tegelijkertijd de doorlopende sessie. Dagelijkse resets en resets wegens inactiviteit zijn
  optioneel (zie [Sessiebeheer](/nl/concepts/session)). Bij `/new` en `/reset`
  wordt het laatste deel van het eindigende gesprek opgeslagen in dagelijkse geheugennotities en
  laadt de volgende sessie recente notities opnieuw in de context. Een reset wijst een nieuwe actieve sessie-id toe, maar
  houdt het vorige SQLite-transcript doorzoekbaar onder dezelfde sleutel van de
  hoofdsessie.
- Wanneer het gesprek het contextvenster nadert, vat Compaction het samen
  en gaat het op dezelfde plek verder — de transcriptgeschiedenis blijft in de sessieopslag.
- Sessielijsten tonen het huidige actieve gesprek, niet elke historische
  sessie-id erachter.
- Wanneer de fysieke database, WAL en sessieartefacten van de opslag per agent
  het schijfquotum overschrijden (standaard 10 GB), haalt OpenClaw de oudste
  geschiedenis waarnaar niet wordt verwezen op en plaatst deze in een geverifieerd gecomprimeerd archief voordat de bijbehorende
  databaserijen worden verwijderd. Actieve, gerouteerde en lopende sessies worden nooit verwijderd vanwege het quotum.

## Wanneer je in plaats daarvan isolatie wilt

De gedeelde hoofdsessie is de juiste standaard voor een agent waarmee alleen jij
praat. Als meerdere personen je agent berichten kunnen sturen, isoleer je de DM's:

```json5
{
  session: {
    dmScope: "per-channel-peer",
  },
}
```

Met een isolerend bereik krijgt elke afzender een eigen sessie, wordt het volgen van groepen
vanuit de hoofdsessie uitgeschakeld en is geheugenherinnering tussen gesprekken
standaard uitgeschakeld. `openclaw security audit` raadt isolatie aan wanneer meerdere
DM-afzenders worden gedetecteerd. De volledige bereikmatrix, identiteitskoppeling en overschrijvingen
per route worden behandeld in [Sessiebeheer](/nl/concepts/session) en
[Kanaalroutering](/nl/channels/channel-routing).

## Gerelateerd

- [Sessiebeheer](/nl/concepts/session) — routering, bereiken, resets
- [Kanaalroutering](/nl/channels/channel-routing) — hoe agents en sessies worden geselecteerd
- [Geheugen](/nl/concepts/memory) — duurzame geheugenlagen
- [Meerdere agents](/nl/concepts/multi-agent) — meerdere geïsoleerde agents uitvoeren
