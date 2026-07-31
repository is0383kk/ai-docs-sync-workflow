---
read_when:
    - Beveiligingsniveau of dreigingsscenario's beoordelen
    - Werken aan beveiligingsfuncties of auditreacties
summary: OpenClaw-dreigingsmodel gekoppeld aan het MITRE ATLAS-framework
title: Dreigingsmodel (MITRE ATLAS)
x-i18n:
    generated_at: "2026-07-27T05:16:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c88ffdef850bd2afaf835baab2555304c914a0be1df6b6b9109e0f55d1448392
    source_path: security/THREAT-MODEL-ATLAS.md
    workflow: 16
---

**Versie:** 1.0-concept | **Framework:** [MITRE ATLAS](https://atlas.mitre.org/) (dreigingslandschap van aanvallen op AI-systemen) + gegevensstroomdiagrammen

Dit dreigingsmodel documenteert vijandige dreigingen voor het OpenClaw-platform voor AI-agents en de ClawHub-marktplaats voor Skills. Het is een levend document dat door de OpenClaw-community wordt onderhouden. Zie [Bijdragen aan het dreigingsmodel](/nl/security/CONTRIBUTING-THREAT-MODEL) voor informatie over het melden van nieuwe dreigingen, voorstellen van aanvalsketens of suggereren van risicobeperkende maatregelen.

**Belangrijkste ATLAS-bronnen:** [Technieken](https://atlas.mitre.org/techniques/) | [Tactieken](https://atlas.mitre.org/tactics/) | [Casestudy's](https://atlas.mitre.org/studies/) | [ATLAS GitHub](https://github.com/mitre-atlas/atlas-data) | [Bijdragen aan ATLAS](https://atlas.mitre.org/resources/contribute)

---

## 1. Reikwijdte

| Component                  | Opgenomen    | Opmerkingen                                      |
| -------------------------- | ------------ | ------------------------------------------------ |
| OpenClaw-agentruntime      | Ja           | Uitvoering van de kernagent, toolaanroepen, sessies |
| Gateway                    | Ja           | Authenticatie, routering, kanaalintegratie       |
| Kanaalintegraties          | Ja           | WhatsApp, Telegram, Discord, Signal, Slack, enz. |
| ClawHub-marktplaats        | Ja           | Publicatie, moderatie en distributie van Skills  |
| MCP-servers                | Ja           | Externe toolproviders                            |
| Gebruikersapparaten        | Gedeeltelijk | Mobiele apps, desktopclients                     |

Meldingen die buiten de reikwijdte vallen en patronen van fout-positieven (openbare blootstelling aan internet, ketens met alleen promptinjectie zonder omzeiling van een grens, onderling niet-vertrouwde operators die één Gateway-host delen en andere) worden opgesomd in [`SECURITY.md`](https://github.com/openclaw/openclaw/blob/main/SECURITY.md); dat bestand is de huidige gezaghebbende bron voor de reikwijdte van kwetsbaarheidsmeldingen, niet deze pagina.

## 2. Systeemarchitectuur

### 2.1 Vertrouwensgrenzen

```text
┌─────────────────────────────────────────────────────────────────┐
│                    NIET-VERTROUWDE ZONE                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │  WhatsApp   │  │  Telegram   │  │   Discord   │  ...         │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘              │
│         │                │                │                      │
└─────────┼────────────────┼────────────────┼──────────────────────┘
          │                │                │
          ▼                ▼                ▼
┌─────────────────────────────────────────────────────────────────┐
│              VERTROUWENSGRENS 1: Kanaaltoegang                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                      GATEWAY                              │   │
│  │  • Apparaatkoppeling (TTL van 1 u voor DM-koppeling /     │   │
│    5 min. voor Node-koppeling)                               │   │
│  │  • Validatie van AllowFrom / toelatingslijst              │   │
│  │  • Authenticatie met token / wachtwoord / Tailscale       │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              VERTROUWENSGRENS 2: Sessie-isolatie                │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                   AGENTSESSIES                            │   │
│  │  • Sessiesleutel = agent:channel:peer                     │   │
│  │  • Toolbeleid per agent                                   │   │
│  │  • Logboekregistratie van transcripties                   │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              VERTROUWENSGRENS 3: Tooluitvoering                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                  UITVOERINGSSANDBOX                       │   │
│  │  • Docker-sandbox (standaard) of host (exec-goedkeuringen)│   │
│  │  • Externe uitvoering via Node                            │   │
│  │  • SSRF-bescherming (DNS-pinning + IP-blokkering)         │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              VERTROUWENSGRENS 4: Externe inhoud                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │         OPGEHAALDE URL'S / E-MAILS / WEBHOOKS             │   │
│  │  • Omhulling van externe inhoud (XML-tags met willekeurige │   │
│    begrenzing)                                                │   │
│  │  • Injectie van beveiligingsmeldingen                     │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              VERTROUWENSGRENS 5: Toeleveringsketen              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                      CLAWHUB                              │   │
│  │  • Publicatie van Skills (semver, SKILL.md vereist)       │   │
│  │  • Moderatiescan met statische patronen + AST-nabije      │   │
│    analyse                                                   │   │
│  │  • LLM-gebaseerde agentische risicobeoordeling +          │   │
│    VirusTotal-scan                                           │   │
│  │  • Verificatie van GitHub-accountleeftijd (14 dagen)      │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Gegevensstromen

| Stroom | Bron    | Bestemming | Gegevens                    | Bescherming                    |
| ------ | ------- | ----------- | --------------------------- | ------------------------------ |
| F1     | Kanaal  | Gateway     | Gebruikersberichten         | TLS, AllowFrom                 |
| F2     | Gateway | Agent       | Gerouteerde berichten       | Sessie-isolatie                |
| F3     | Agent   | Tools       | Toolaanroepen               | Beleidshandhaving               |
| F4     | Agent   | Extern      | `web_fetch`-verzoeken | SSRF-blokkering                |
| F5     | ClawHub | Agent       | Skillcode                   | Moderatie, scannen             |
| F6     | Agent   | Kanaal      | Antwoorden                  | Uitvoerfiltering               |

---

## 3. Dreigingsanalyse per ATLAS-tactiek

### 3.1 Verkenning (AML.TA0002)

#### T-RECON-001: Detectie van agent-eindpunten

| Kenmerk                  | Waarde                                                               |
| ------------------------ | -------------------------------------------------------------------- |
| **ATLAS-ID**             | AML.T0006 - Actief scannen                                           |
| **Beschrijving**         | Aanvaller scant op blootgestelde OpenClaw Gateway-eindpunten         |
| **Aanvalsvector**        | Netwerkscans, Shodan-query's, DNS-inventarisatie                     |
| **Getroffen componenten** | Gateway, blootgestelde API-eindpunten                               |
| **Huidige maatregelen**  | Optie voor Tailscale-authenticatie, standaardbinding aan loopback    |
| **Restrisico**           | Gemiddeld - openbare Gateways kunnen worden ontdekt                  |
| **Aanbevelingen**        | Documenteer veilige implementatie, voeg frequentiebeperking toe aan detectie-eindpunten |

#### T-RECON-002: Kanaalintegraties onderzoeken

| Kenmerk                  | Waarde                                                            |
| ------------------------ | ----------------------------------------------------------------- |
| **ATLAS-ID**             | AML.T0006 - Actief scannen                                       |
| **Beschrijving**         | Aanvaller onderzoekt berichtenkanalen om door AI beheerde accounts te identificeren |
| **Aanvalsvector**        | Testberichten verzenden, antwoordpatronen observeren              |
| **Getroffen componenten** | Alle kanaalintegraties                                           |
| **Huidige maatregelen**  | Geen specifieke                                                   |
| **Restrisico**           | Laag - ontdekking alleen heeft beperkte waarde                    |
| **Aanbevelingen**        | Overweeg randomisatie van antwoordtijden                          |

---

### 3.2 Initiële toegang (AML.TA0004)

#### T-ACCESS-001: Onderschepping van koppelcode

| Kenmerk                | Waarde                                                                                                                         |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| **ATLAS-ID**           | AML.T0040 - Toegang tot API voor AI-modelinferentie                                                                            |
| **Beschrijving**       | Aanvaller onderschept een koppelcode tijdens het koppelvenster (1u voor DM/algemeen koppelen, 5m voor het koppelen van een Node) |
| **Aanvalsvector**      | Meekijken over de schouder, netwerkverkeer onderscheppen, social engineering                                                   |
| **Getroffen onderdelen** | Apparaatkoppelsysteem                                                                                                        |
| **Huidige maatregelen** | TTL van 1u (DM/algemeen koppelen), TTL van 5m (Node koppelen); codes worden via het bestaande kanaal verzonden                |
| **Resterend risico**   | Gemiddeld - koppelvenster kan worden misbruikt                                                                                  |
| **Aanbevelingen**      | Verkort het koppelvenster en voeg een bevestigingsstap toe                                                                      |

#### T-ACCESS-002: AllowFrom-spoofing

| Kenmerk                | Waarde                                                                                          |
| ---------------------- | ----------------------------------------------------------------------------------------------- |
| **ATLAS-ID**           | AML.T0040 - Toegang tot API voor AI-modelinferentie                                             |
| **Beschrijving**       | Aanvaller vervalst de identiteit van een toegestane afzender op een kanaal                      |
| **Aanvalsvector**      | Kanaalafhankelijk - vervalsing van telefoonnummers, imitatie van gebruikersnamen                 |
| **Getroffen onderdelen** | AllowFrom-validatie per kanaal                                                                |
| **Huidige maatregelen** | Kanaalspecifieke identiteitsverificatie                                                       |
| **Resterend risico**   | Gemiddeld - sommige kanalen blijven kwetsbaar voor spoofing                                      |
| **Aanbevelingen**      | Documenteer kanaalspecifieke risico's en voeg waar mogelijk cryptografische verificatie toe      |

#### T-ACCESS-003: Tokendiefstal

| Kenmerk                | Waarde                                                                        |
| ---------------------- | ----------------------------------------------------------------------------- |
| **ATLAS-ID**           | AML.T0040 - Toegang tot API voor AI-modelinferentie                           |
| **Beschrijving**       | Aanvaller steelt authenticatietokens uit configuratie- of referentiebestanden |
| **Aanvalsvector**      | Malware, onbevoegde apparaattoegang, blootstelling van configuratieback-ups   |
| **Getroffen onderdelen** | Opslag van kanaal-/providerreferenties, configuratieopslag                  |
| **Huidige maatregelen** | Bestandsmachtigingen                                                        |
| **Resterend risico**   | Hoog - tokens worden als platte tekst op schijf opgeslagen                    |
| **Aanbevelingen**      | Implementeer versleuteling van opgeslagen tokens en voeg tokenrotatie toe      |

---

### 3.3 Uitvoering (AML.TA0005)

#### T-EXEC-001: Directe promptinjectie

| Kenmerk                | Waarde                                                                                                                                                                              |
| ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **ATLAS-ID**           | AML.T0051.000 - LLM-promptinjectie: direct                                                                                                                                          |
| **Beschrijving**       | Aanvaller verzendt speciaal vervaardigde prompts om het gedrag van de agent te manipuleren                                                                                          |
| **Aanvalsvector**      | Kanaalberichten met vijandige instructies                                                                                                                                           |
| **Getroffen onderdelen** | LLM van de agent, alle invoeroppervlakken                                                                                                                                         |
| **Huidige maatregelen** | Patroondetectie, inkapseling van externe inhoud; wordt zonder omzeiling van een grens beschouwd als buiten het bereik van kwetsbaarheidsmeldingen (zie `SECURITY.md`)          |
| **Resterend risico**   | Kritiek - alleen detectie, geen blokkering; geavanceerde aanvallen omzeilen deze                                                                                                     |
| **Aanbevelingen**      | Uitvoervalidatie en gebruikersbevestiging voor gevoelige acties, als extra laag boven op de bestaande detectie                                                                      |

#### T-EXEC-002: Indirecte promptinjectie

| Kenmerk                | Waarde                                                                                                                                |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| **ATLAS-ID**           | AML.T0051.001 - LLM-promptinjectie: indirect                                                                                          |
| **Beschrijving**       | Aanvaller neemt schadelijke instructies op in opgehaalde inhoud                                                                       |
| **Aanvalsvector**      | Schadelijke URL's, vergiftigde e-mails, gecompromitteerde webhooks                                                                    |
| **Getroffen onderdelen** | `web_fetch`, verwerking van e-mails, externe gegevensbronnen                                                                 |
| **Huidige maatregelen** | Inkapseling van inhoud met willekeurige XML-achtige grensmarkeringen, normalisatie van homogliefe/speciale tokens en een beveiligingsmelding |
| **Resterend risico**   | Hoog - het LLM kan de instructies van de inkapseling nog steeds negeren                                                               |
| **Aanbevelingen**      | Afzonderlijke uitvoeringscontexten voor ingekapselde inhoud                                                                           |

#### T-EXEC-003: Injectie van toolargumenten

| Kenmerk                | Waarde                                                            |
| ---------------------- | ----------------------------------------------------------------- |
| **ATLAS-ID**           | AML.T0051.000 - LLM-promptinjectie: direct                        |
| **Beschrijving**       | Aanvaller manipuleert toolargumenten via promptinjectie            |
| **Aanvalsvector**      | Speciaal vervaardigde prompts die de waarden van toolparameters beïnvloeden |
| **Getroffen onderdelen** | Alle toolaanroepen                                              |
| **Huidige maatregelen** | Uitvoeringsgoedkeuringen voor gevaarlijke opdrachten            |
| **Resterend risico**   | Hoog - berust op het oordeel van de gebruiker                      |
| **Aanbevelingen**      | Argumentvalidatie, geparametriseerde toolaanroepen                  |

#### T-EXEC-004: Omzeiling van uitvoeringsgoedkeuring

| Kenmerk                | Waarde                                                                                                                                                                                                         |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **ATLAS-ID**           | AML.T0043 - Vijandige gegevens vervaardigen                                                                                                                                                                    |
| **Beschrijving**       | Aanvaller vervaardigt opdrachten die de goedkeuringsallowlist omzeilen                                                                                                                                         |
| **Aanvalsvector**      | Verhulling van opdrachten, misbruik van aliassen, manipulatie van paden                                                                                                                                        |
| **Getroffen onderdelen** | `src/infra/exec-approvals*.ts`, allowlist voor opdrachten                                                                                                                                                                |
| **Huidige maatregelen** | Allowlist + vraagmodus, plus normalisatie van opdrachten (uitpakken van dispatch-wrappers, detectie van inline-evaluatie, analyse van shellketens)                                                             |
| **Resterend risico**   | Hoog - normalisatie beperkt omzeiling door verhulling, maar sluit deze niet uit; bevindingen die alleen verschillen in gelijkwaardigheid tussen uitvoeringspaden worden beschouwd als versterking, niet als kwetsbaarheden (zie `SECURITY.md`) |
| **Aanbevelingen**      | Blijf de dekking van opdrachtnormalisatie uitbreiden voor nieuwe verhullingstechnieken                                                                                                                         |

---

### 3.4 Persistentie (AML.TA0006)

#### T-PERSIST-001: Installatie van schadelijke skill

| Kenmerk                | Waarde                                                                                                                               |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| **ATLAS-ID**           | AML.T0010.001 - Compromittering van de toeleveringsketen: AI-software                                                                |
| **Beschrijving**       | Aanvaller publiceert een schadelijke skill op ClawHub                                                                                 |
| **Aanvalsvector**      | Account aanmaken, skill met verborgen schadelijke code publiceren                                                                    |
| **Getroffen onderdelen** | ClawHub, laden van skills, uitvoering door de agent                                                                                |
| **Huidige maatregelen** | Verificatie van de leeftijd van het GitHub-account, statische patroon-/AST-gerelateerde scans, LLM-gebaseerde agentische risicobeoordeling, VirusTotal-scans |
| **Resterend risico**   | Hoog - er bestaan detectielagen, maar skills worden nog steeds uitgevoerd met agentrechten en zonder uitvoeringssandboxing            |
| **Aanbevelingen**      | Sandboxen van de uitvoering van skills, uitgebreidere beoordeling door de community                                                   |

#### T-PERSIST-002: Vergiftiging van skillupdates

| Kenmerk                | Waarde                                                                          |
| ---------------------- | ------------------------------------------------------------------------------- |
| **ATLAS-ID**           | AML.T0010.001 - Compromittering van de toeleveringsketen: AI-software           |
| **Beschrijving**       | Aanvaller compromitteert een populaire skill en publiceert een schadelijke update |
| **Aanvalsvector**      | Compromittering van een account, social engineering van de eigenaar van de skill |
| **Getroffen onderdelen** | ClawHub-versiebeheer, automatische updateprocessen                            |
| **Huidige maatregelen** | Versie-fingerprinting, moderatie/scans worden opnieuw uitgevoerd voor nieuwe versies |
| **Resterend risico**   | Hoog - automatische updates kunnen schadelijke versies ophalen voordat de beoordeling is voltooid |
| **Aanbevelingen**      | Ondertekening van updates, mogelijkheid tot terugdraaien, vastzetten van versies |

#### T-PERSIST-003: Manipulatie van agentconfiguratie

| Kenmerk                | Waarde                                                                    |
| ---------------------- | ------------------------------------------------------------------------- |
| **ATLAS-ID**           | AML.T0010.002 - Compromittering van de toeleveringsketen: gegevens        |
| **Beschrijving**       | Aanvaller wijzigt de agentconfiguratie om toegang te behouden              |
| **Aanvalsvector**      | Wijziging van configuratiebestanden, injectie van instellingen             |
| **Getroffen onderdelen** | Agentconfiguratie, toolbeleid                                            |
| **Huidige maatregelen** | Bestandsmachtigingen                                                      |
| **Restrisico**         | Gemiddeld - vereist lokale toegang                                         |
| **Aanbevelingen**      | Integriteitsverificatie van configuratie, auditlogboek voor configuratiewijzigingen |

---

### 3.5 Omzeiling van verdediging (AML.TA0007)

#### T-EVADE-001: Omzeiling van moderatiepatronen

| Kenmerk                | Waarde                                                                                           |
| ---------------------- | ----------------------------------------------------------------------------------------------- |
| **ATLAS-ID**           | AML.T0043 - Adversariële gegevens maken                                                         |
| **Beschrijving**       | Aanvaller maakt Skills-inhoud om de moderatiecontroles van ClawHub te omzeilen                   |
| **Aanvalsvector**      | Unicode-homogliefen, coderingstrucs, dynamisch laden                                             |
| **Getroffen onderdelen** | Moderatie-/scanpijplijn van ClawHub                                                           |
| **Huidige maatregelen** | Statische patroonregels, codeanalyse nabij de AST, LLM-beoordeling van agentrisico's, VirusTotal |
| **Restrisico**         | Gemiddeld - nieuwe obfuscatie kan gelaagde heuristieken nog steeds omzeilen                      |
| **Aanbevelingen**      | Blijf de corpus met patronen en gedragingen uitbreiden wanneer nieuwe omzeilingen worden ontdekt |

#### T-EVADE-002: Ontsnapping uit inhoudswrapper

| Kenmerk                | Waarde                                                                                                                  |
| ---------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| **ATLAS-ID**           | AML.T0043 - Adversariële gegevens maken                                                                                  |
| **Beschrijving**       | Aanvaller maakt inhoud die ontsnapt uit de context van de wrapper voor externe inhoud                                   |
| **Aanvalsvector**      | Tagmanipulatie, contextverwarring, overschrijving van instructies                                                        |
| **Getroffen onderdelen** | Wrapping van externe inhoud                                                                                            |
| **Huidige maatregelen** | XML-achtige markeringen met willekeurige begrenzingen + beveiligingsmelding, plus detectie van markeringsspoofing met homogliefen/witruimtevarianten |
| **Restrisico**         | Gemiddeld - nieuwe ontsnappingsmethoden worden regelmatig ontdekt                                                        |
| **Aanbevelingen**      | Validatie aan de uitvoerzijde naast wrapping aan de invoerzijde                                                          |

---

### 3.6 Verkenning (AML.TA0008)

#### T-DISC-001: Inventarisatie van tools

| Kenmerk                | Waarde                                                      |
| ---------------------- | ----------------------------------------------------------- |
| **ATLAS-ID**           | AML.T0040 - Toegang tot de inferentie-API van het AI-model   |
| **Beschrijving**       | Aanvaller inventariseert beschikbare tools via prompts       |
| **Aanvalsvector**      | Vragen in de trant van "Welke tools heb je?"                 |
| **Getroffen onderdelen** | Register met agenttools                                    |
| **Huidige maatregelen** | Geen specifieke                                             |
| **Restrisico**         | Laag - tools zijn doorgaans gedocumenteerd                   |
| **Aanbevelingen**      | Overweeg beheeropties voor de zichtbaarheid van tools        |

#### T-DISC-002: Extractie van sessiegegevens

| Kenmerk                | Waarde                                                          |
| ---------------------- | --------------------------------------------------------------- |
| **ATLAS-ID**           | AML.T0040 - Toegang tot de inferentie-API van het AI-model       |
| **Beschrijving**       | Aanvaller extraheert gevoelige gegevens uit de sessiecontext     |
| **Aanvalsvector**      | Vragen als "Wat hebben we besproken?", contextonderzoek          |
| **Getroffen onderdelen** | Sessietranscripten, contextvenster                              |
| **Huidige maatregelen** | Sessiescheiding per afzender (`agent:channel:peer`-sleutel)       |
| **Restrisico**         | Gemiddeld - gegevens binnen de sessie zijn bewust toegankelijk   |
| **Aanbevelingen**      | Redactie van gevoelige gegevens in de context                    |

---

### 3.7 Verzameling en exfiltratie (AML.TA0009, AML.TA0010)

#### T-EXFIL-001: Gegevensdiefstal via web_fetch

| Kenmerk                | Waarde                                                                                         |
| ---------------------- | ---------------------------------------------------------------------------------------------- |
| **ATLAS-ID**           | AML.T0009 - Verzameling                                                                        |
| **Beschrijving**       | Aanvaller exfiltreert gegevens door de agent opdracht te geven deze naar een externe URL te sturen |
| **Aanvalsvector**      | Promptinjectie waardoor de agent gegevens via POST naar een server van de aanvaller stuurt      |
| **Getroffen onderdelen** | Tool `web_fetch`                                                                      |
| **Huidige maatregelen** | SSRF-blokkering voor interne/privénetwerken (DNS-pinning + IP-blokkering)                       |
| **Restrisico**         | Hoog - willekeurige externe URL's blijven toegestaan                                           |
| **Aanbevelingen**      | Allowlist voor URL's, bewustzijn van gegevensclassificatie                                      |

#### T-EXFIL-002: Ongeautoriseerde verzending van berichten

| Kenmerk                | Waarde                                                                       |
| ---------------------- | ---------------------------------------------------------------------------- |
| **ATLAS-ID**           | AML.T0009 - Verzameling                                                      |
| **Beschrijving**       | Aanvaller laat de agent berichten met gevoelige gegevens versturen           |
| **Aanvalsvector**      | Promptinjectie waardoor de agent een bericht naar de aanvaller stuurt         |
| **Getroffen onderdelen** | Berichtentool, kanaalintegraties                                            |
| **Huidige maatregelen** | Toegangscontrole voor uitgaande berichten                                   |
| **Restrisico**         | Gemiddeld - de toegangscontrole kan mogelijk worden omzeild                   |
| **Aanbevelingen**      | Expliciete bevestiging voor nieuwe ontvangers                                 |

#### T-EXFIL-003: Verzamelen van aanmeldgegevens

| Kenmerk                | Waarde                                                                                                                                                                          |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **ATLAS-ID**           | AML.T0009 - Verzameling                                                                                                                                                         |
| **Beschrijving**       | Schadelijke Skill verzamelt aanmeldgegevens uit de agentcontext                                                                                                                 |
| **Aanvalsvector**      | Skill-code leest omgevingsvariabelen en configuratiebestanden                                                                                                                   |
| **Getroffen onderdelen** | Uitvoeringsomgeving voor Skills                                                                                                                                               |
| **Huidige maatregelen** | Scannen door ClawHub op patronen van aanmeldgegevens (hardgecodeerde geheimen, toegang tot omgevingsvariabelen met aanmeldgegevens gekoppeld aan netwerkverzendingen); geen uitvoeringssandbox voor Skills tijdens runtime |
| **Restrisico**         | Kritiek - Skills worden uitgevoerd met de rechten van de agent                                                                                                                  |
| **Aanbevelingen**      | Uitvoeringssandbox voor Skills, isolatie van aanmeldgegevens                                                                                                                     |

---

### 3.8 Impact (AML.TA0011)

#### T-IMPACT-001: Ongeautoriseerde uitvoering van opdrachten

| Kenmerk                | Waarde                                                                                                                |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------- |
| **ATLAS-ID**           | AML.T0031 - Integriteit van het AI-model aantasten                                                                     |
| **Beschrijving**       | Aanvaller voert willekeurige opdrachten uit op het systeem van de gebruiker                                           |
| **Aanvalsvector**      | Promptinjectie gecombineerd met omzeiling van uitvoeringsgoedkeuring                                                   |
| **Getroffen onderdelen** | Bash-tool, uitvoering van opdrachten                                                                                 |
| **Huidige maatregelen** | Uitvoeringsgoedkeuringen, Docker-sandboxoptie (standaard runtimebackend)                                              |
| **Restrisico**         | Kritiek - uitvoering op de host is mogelijk wanneer de sandbox is uitgeschakeld                                       |
| **Aanbevelingen**      | Verbeter de gebruikerservaring voor goedkeuringen; implementaties zonder sandbox blijven een bewuste keuze van de beheerder en worden als zodanig gedocumenteerd |

#### T-IMPACT-002: Uitputting van middelen (DoS)

| Kenmerk                | Waarde                                                    |
| ---------------------- | --------------------------------------------------------- |
| **ATLAS-ID**           | AML.T0031 - Integriteit van het AI-model aantasten         |
| **Beschrijving**       | Aanvaller put API-tegoeden of rekenbronnen uit             |
| **Aanvalsvector**      | Geautomatiseerde berichtoverstroming, dure toolaanroepen    |
| **Getroffen onderdelen** | Gateway, agentsessies, API-provider                       |
| **Huidige maatregelen** | Geen                                                      |
| **Restrisico**         | Hoog - geen snelheidsbeperking per afzender                 |
| **Aanbevelingen**      | Snelheidslimieten per afzender, kostenbudgetten             |

#### T-IMPACT-003: Reputatieschade

| Kenmerk                | Waarde                                                         |
| ---------------------- | -------------------------------------------------------------- |
| **ATLAS-ID**           | AML.T0031 - Integriteit van het AI-model aantasten              |
| **Beschrijving**       | Aanvaller laat de agent schadelijke/aanstootgevende inhoud versturen |
| **Aanvalsvector**      | Promptinjectie die ongepaste antwoorden veroorzaakt             |
| **Getroffen onderdelen** | Uitvoergeneratie, kanaalberichten                              |
| **Huidige maatregelen** | Inhoudsbeleid van de LLM-provider                              |
| **Restrisico**         | Gemiddeld - providerfilters zijn niet perfect                    |
| **Aanbevelingen**      | Filterlaag voor uitvoer, gebruikersinstellingen                  |

---

## 4. Analyse van de ClawHub-toeleveringsketen

### 4.1 Huidige beveiligingsmaatregelen

| Beheersmaatregel                | Implementatie                                                                         | Effectiviteit                                                      |
| ------------------------------ | ------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| Leeftijd GitHub-account        | `requireGitHubAccountAge()` (minimaal 14 dagen)                                                | Gemiddeld - verhoogt de drempel voor nieuwe aanvallers              |
| Padopschoning                  | `sanitizePath()`                                                                    | Hoog - voorkomt padtraversal                                        |
| Bestandstypevalidatie          | `isTextFile()`                                                                    | Gemiddeld - alleen tekstbestanden worden gescand, maar misbruik blijft mogelijk |
| Groottelimieten                | Totale bundel van 50MB (`MAX_PUBLISH_TOTAL_BYTES`)                                           | Hoog - voorkomt uitputting van resources                            |
| Verplicht SKILL.md             | Verplichte readme bij publicatie                                                      | Lage beveiligingswaarde - uitsluitend informatief                   |
| Statische + AST-aangrenzende scanning | Patroonengine voor exec, exfiltratie, het verzamelen van inloggegevens, obfuscatie en meer | Gemiddeld-hoog - dekt veel bekende misbruikpatronen, maar blijft patroongebaseerd |
| Agentische risicobeoordeling op basis van LLM | Door een beveiligingsprompt aangestuurd oordeel bij publicatie                       | Gemiddeld-hoog - detecteert gedrag dat statische patronen missen    |
| VirusTotal-scanning            | Gekoppeld aan publicatie- en herscanflows voor Skills en pakketreleases, afhankelijk van een API-sleutel van de operator | Hoog indien ingeschakeld - detectie door statische engines          |
| Moderatiestatus                | Veld `moderationStatus`                                                               | Gemiddeld - handmatige beoordeling mogelijk                         |

### 4.2 Beperkingen van moderatie

De statische scanning van ClawHub inspecteert de code-inhoud van Skills rechtstreeks (niet alleen slug/metadata/frontmatter) en controleert op gevaarlijke exec-aanroepen, dynamische code-uitvoering, het verzamelen van inloggegevens, exfiltratiepatronen, geobfusceerde payloads en meer. Bekende tekortkomingen:

- Patroongebaseerde detectie kan nog steeds worden omzeild met voldoende nieuwe obfuscatietechnieken.
- Beoordeling op basis van LLM en VirusTotal-scanning zijn afhankelijk van ingeschakelde API-sleutels/configuratie aan de zijde van de operator.
- Er is geen sandbox voor runtime-uitvoering die een geïnstalleerde Skill isoleert van de eigen bevoegdheden van de agent.

### 4.3 Badges

Skills en pakketten hebben door moderators toegewezen badges: `highlighted`, `official`, `deprecated`, `redactionApproved` (alleen Skills). Meldingen uit de community (`skillReports`) en auditlogboekregistratie (`auditLogs`) ondersteunen moderatieworkflows.

---

## 5. Risicomatrix

### 5.1 Waarschijnlijkheid versus impact

| Dreigings-ID  | Waarschijnlijkheid | Impact   | Risiconiveau | Prioriteit |
| ------------- | ------------------ | -------- | ------------ | ---------- |
| T-EXEC-001    | Hoog               | Kritiek  | **Kritiek**  | P0         |
| T-PERSIST-001 | Hoog               | Kritiek  | **Kritiek**  | P0         |
| T-EXFIL-003   | Gemiddeld          | Kritiek  | **Kritiek**  | P0         |
| T-IMPACT-001  | Gemiddeld          | Kritiek  | **Hoog**     | P1         |
| T-EXEC-002    | Hoog               | Hoog     | **Hoog**     | P1         |
| T-EXEC-004    | Gemiddeld          | Hoog     | **Hoog**     | P1         |
| T-ACCESS-003  | Gemiddeld          | Hoog     | **Hoog**     | P1         |
| T-EXFIL-001   | Gemiddeld          | Hoog     | **Hoog**     | P1         |
| T-IMPACT-002  | Hoog               | Gemiddeld| **Hoog**     | P1         |
| T-EVADE-001   | Hoog               | Gemiddeld| **Gemiddeld**| P2         |
| T-ACCESS-001  | Laag               | Hoog     | **Gemiddeld**| P2         |
| T-ACCESS-002  | Laag               | Hoog     | **Gemiddeld**| P2         |
| T-PERSIST-002 | Laag               | Hoog     | **Gemiddeld**| P2         |

### 5.2 Aanvalsketens op het kritieke pad

**Keten 1: Gegevensdiefstal via een Skill**

```text
T-PERSIST-001 → T-EVADE-001 → T-EXFIL-003
(Kwaadaardige Skill publiceren) → (Moderatie omzeilen) → (Inloggegevens verzamelen)
```

**Keten 2: Promptinjectie naar RCE**

```text
T-EXEC-001 → T-EXEC-004 → T-IMPACT-001
(Prompt injecteren) → (Exec-goedkeuring omzeilen) → (Opdrachten uitvoeren)
```

**Keten 3: Indirecte injectie via opgehaalde inhoud**

```text
T-EXEC-002 → T-EXFIL-001 → Externe exfiltratie
(URL-inhoud manipuleren) → (Agent haalt inhoud op en volgt instructies) → (Gegevens naar aanvaller verzonden)
```

---

## 6. Samenvatting van aanbevelingen

### 6.1 Onmiddellijk (P0)

| ID    | Aanbeveling                                     | Pakt aan                   |
| ----- | ---------------------------------------------- | -------------------------- |
| R-002 | Implementeer sandboxing voor Skill-uitvoering  | T-PERSIST-001, T-EXFIL-003 |
| R-003 | Voeg uitvoervalidatie toe voor gevoelige acties | T-EXEC-001, T-EXEC-002    |

### 6.2 Korte termijn (P1)

| ID    | Aanbeveling                                                                | Pakt aan     |
| ----- | -------------------------------------------------------------------------- | ------------ |
| R-004 | Implementeer snelheidsbeperking per afzender                               | T-IMPACT-002 |
| R-005 | Voeg versleuteling van tokens in rust toe                                  | T-ACCESS-003 |
| R-006 | Verbeter de UX voor exec-goedkeuring en breid opdrachtnormalisatie verder uit | T-EXEC-004 |
| R-007 | Implementeer een URL-toelatingslijst voor `web_fetch`               | T-EXFIL-001  |

### 6.3 Middellange termijn (P2)

| ID    | Aanbeveling                                                   | Pakt aan      |
| ----- | ------------------------------------------------------------ | ------------- |
| R-008 | Voeg waar mogelijk cryptografische kanaalverificatie toe     | T-ACCESS-002  |
| R-009 | Implementeer integriteitsverificatie van de configuratie     | T-PERSIST-003 |
| R-010 | Voeg ondertekening van updates en versiepinning toe          | T-PERSIST-002 |

---

## 7. Bijlagen

### 7.1 Toewijzing van ATLAS-technieken

| ATLAS-ID      | Naam van techniek              | OpenClaw-dreigingen                                              |
| ------------- | ------------------------------ | ---------------------------------------------------------------- |
| AML.T0006     | Actieve scanning               | T-RECON-001, T-RECON-002                                         |
| AML.T0009     | Verzameling                    | T-EXFIL-001, T-EXFIL-002, T-EXFIL-003                            |
| AML.T0010.001 | Toeleveringsketen: AI-software | T-PERSIST-001, T-PERSIST-002                                     |
| AML.T0010.002 | Toeleveringsketen: gegevens    | T-PERSIST-003                                                    |
| AML.T0031     | Integriteit van AI-model aantasten | T-IMPACT-001, T-IMPACT-002, T-IMPACT-003                      |
| AML.T0040     | Toegang tot API voor AI-modelinferentie | T-ACCESS-001, T-ACCESS-002, T-ACCESS-003, T-DISC-001, T-DISC-002 |
| AML.T0043     | Vijandige gegevens vervaardigen | T-EXEC-004, T-EVADE-001, T-EVADE-002                            |
| AML.T0051.000 | LLM-promptinjectie: direct     | T-EXEC-001, T-EXEC-003                                           |
| AML.T0051.001 | LLM-promptinjectie: indirect   | T-EXEC-002                                                       |

### 7.2 Belangrijkste beveiligingsbestanden

| Pad                                 | Doel                              | Risiconiveau |
| ----------------------------------- | --------------------------------- | ------------ |
| `src/infra/exec-approvals.ts`                  | Logica voor opdrachtgoedkeuring   | **Kritiek**  |
| `src/gateway/auth.ts`                  | Gateway-authenticatie             | **Kritiek**  |
| `src/infra/net/ssrf.ts`                  | SSRF-bescherming                  | **Kritiek**  |
| `src/security/external-content.ts`                  | Beperking van promptinjectie      | **Kritiek**  |
| `src/agents/sandbox/tool-policy.ts`                  | Toestaan/weigeren-beleid voor sandboxtools | **Kritiek** |
| `src/routing/resolve-route.ts`                  | Sessie-isolatie/routering         | **Gemiddeld** |

### 7.3 Woordenlijst

| Term                 | Definitie                                                   |
| -------------------- | ----------------------------------------------------------- |
| **ATLAS**            | MITRE's dreigingslandschap voor vijandige aanvallen op AI-systemen |
| **ClawHub**          | Marktplaats voor OpenClaw-Skills                            |
| **Gateway**          | Laag voor berichtroutering en authenticatie van OpenClaw   |
| **MCP**              | Model Context Protocol - interface voor toolproviders       |
| **Promptinjectie**   | Aanval waarbij kwaadaardige instructies in invoer zijn ingebed |
| **Skill**            | Downloadbare uitbreiding voor OpenClaw-agents               |
| **SSRF**             | Server-Side Request Forgery                                 |

---

_Dit dreigingsmodel is een levend document. Meld beveiligingsproblemen bij `security@openclaw.ai` of raadpleeg de [vertrouwenspagina](https://trust.openclaw.ai)._

## Gerelateerd

- [Bijdragen aan het dreigingsmodel](/nl/security/CONTRIBUTING-THREAT-MODEL)
- [Incidentrespons](/nl/security/incident-response)
- [Netwerkproxy](/nl/security/network-proxy)
- [Formele verificatie](/nl/security/formal-verification)
