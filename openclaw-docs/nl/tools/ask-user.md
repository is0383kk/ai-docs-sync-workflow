---
read_when:
    - Je wilt dat een agent de gebruiker een gestructureerde vraag stelt
    - Je beantwoordt of debugt een ask_user-prompt
    - Je hebt het schema, de time-out of het kanaalgedrag van ask_user nodig
summary: Hoe ask_user een agentbeurt pauzeert voor een gestructureerde menselijke beslissing
title: Gebruiker vragen
x-i18n:
    generated_at: "2026-07-27T05:52:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 32556314a34c26054c3aabfdd8ecc474cf85196e5cc71adb833face596edbd24
    source_path: tools/ask-user.md
    workflow: 16
---

`ask_user` laat de agent één tot drie gestructureerde vragen aan de mens stellen en
op de antwoorden wachten. De tool is bedoeld voor beslissingen die daadwerkelijk aan de gebruiker toebehoren,
niet voor routinematige bevestiging of informatie die de agent kan afleiden uit het verzoek,
de code of een verstandige standaardwaarde.

De tool is alleen beschikbaar in de hoofdsessie. Subagents en andere niet-primaire
uitvoeringen hebben er geen toegang toe.

## Een vraag beantwoorden

Je kunt antwoorden via elk ondersteund gespreksoppervlak:

- De web-Control UI plaatst een vragenpaneel direct boven het invoerveld. Bij
  prompts met meerdere vragen toont het paneel één vraag tegelijk en doorloopt het
  een korte stappenreeks. Na beantwoording sluit het paneel en behoudt de chat
  alleen een compacte samenvatting van de antwoorden.
- Telegram, Discord en Slack tonen systeemeigen knoppen voor een prompt met één vraag
  en één antwoordkeuze.
- Een antwoord in platte tekst werkt op elk kanaal. Antwoord met een nummer, een optielabel
  of je eigen antwoord.

OpenClaw schakelt altijd een vrij tekstveld voor het antwoord **Other** in. De agent mag geen
optie `Other` toevoegen aan de opgestelde optielijst.

## Platformgedrag

Antwoorden werken via elk ondersteund gespreksoppervlak. De web-Control UI gebruikt een
vastgezet stappenpaneel dat in uitgeklapte toestand het invoerveld vervangt; bij inklappen verschijnt
het volledige invoerveld weer onder een smalle vragenbalk. iOS, macOS en Android tonen
inline kaarten; meerdere vragen blijven opzettelijk gestapeld als aanraakvriendelijk
patroon. Elk platform bewaart de samenvatting van vragen en antwoorden zonder tijdgebonden
verwijdering in de actieve chattijdlijn, en **Skip** is overal beschikbaar.

Prompts waarvoor geen systeemeigen knoppen kunnen worden gebruikt, waaronder prompts met meerdere vragen en
meervoudige selectie, worden op kanalen omgezet in leesbare tekst. De Control UI
behoudt het volledige gestructureerde stappenpaneel.

## Time-out en geen antwoord

De standaardtime-out is 900 seconden. `timeoutSeconds` wordt begrensd tot het bereik
van 30 tot en met 3600 seconden.

Als de vraag verloopt of wordt geannuleerd voordat er een antwoord binnenkomt, retourneert de tool
`status: "no_answer"`. De agent gaat dan verder naar eigen beste inzicht.
Een afgebroken agentuitvoering annuleert de openstaande Gateway-vraag.

## Toolschema

```ts
{
  questions: Array<{
    id: string; // unieke antwoordsleutel in snake_case
    header: string; // kort label; afgekapt tot 12 tekens
    question: string; // één zin
    options: Array<{
      label: string;
      description?: string;
    }>; // 2-4 opties
    multiSelect?: boolean;
  }>; // 1-3 vragen
  timeoutSeconds?: number; // geheel getal; standaard 900, begrensd tot 30-3600
}
```

Met `multiSelect: true` kan de gebruiker meer dan één optie kiezen. Antwoordwaarden
worden voor elke vraag als een array geretourneerd.

Voorbeeld van een beantwoord resultaat:

```json
{
  "status": "answered",
  "answers": {
    "answers": {
      "deploy_target": ["Staging (Recommended)"]
    }
  }
}
```

## Richtlijnen voor het model

Het modelgerichte contract instrueert de agent om:

- alleen vragen te stellen wanneer die wordt geblokkeerd door een beslissing die daadwerkelijk aan de gebruiker toebehoort;
- bij voorkeur één vraag te stellen en er niet meer dan drie te gebruiken;
- de aanbevolen optie als eerste te plaatsen en `(Recommended)` aan het einde van het label toe te voegen;
- een opgestelde optie `Other` weg te laten omdat vrije tekst automatisch wordt toegevoegd;
- na `no_answer` naar eigen beste inzicht verder te gaan.

De agent mag `ask_user` niet gebruiken om te vragen of die mag doorgaan of om
het eigen plan te bevestigen.
