---
read_when:
    - Standaardinstellingen, toelatingslijsten of gedrag van slash-opdrachten voor de verhoogde modus aanpassen
    - Begrijpen hoe agents in een sandbox toegang kunnen krijgen tot de host
summary: 'Uitgebreide uitvoeringsmodus: voer opdrachten buiten de sandbox uit vanuit een gesandboxte agent'
title: Modus met verhoogde rechten
x-i18n:
    generated_at: "2026-07-27T06:15:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 40627217acf56122acfc48b689be1b9e2c61d889fe698e9c3c8fd91270d4a6cf
    source_path: tools/elevated.md
    workflow: 16
---

Wanneer een agent in een sandbox wordt uitgevoerd, zijn de `exec`-opdrachten beperkt tot de sandboxomgeving. Met de **verhoogde modus** kan de agent deze beperking omzeilen en in plaats daarvan opdrachten buiten de sandbox uitvoeren, met configureerbare goedkeuringspoorten.

<Info>
  De verhoogde modus verandert het gedrag alleen wanneer de agent **in een sandbox** wordt uitgevoerd. Voor agents zonder sandbox wordt exec al op de host uitgevoerd.
</Info>

## Directieven

Beheer de verhoogde modus per sessie met slash-opdrachten:

| Directief        | Functie                                                                                                                    |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `/elevated on`   | Buiten de sandbox uitvoeren op het geconfigureerde hostpad, met behoud van goedkeuringen                                                             |
| `/elevated ask`  | Hetzelfde als `on` (alias)                                                                                                            |
| `/elevated full` | Buiten de sandbox uitvoeren op het geconfigureerde hostpad en goedkeuringen overslaan wanneer het goedkeuringsbeleid voor de modus/host al permissief is |
| `/elevated off`  | Terugkeren naar uitvoering die tot de sandbox is beperkt                                                                                            |

Ook beschikbaar als `/elev on|off|ask|full`.

Stuur `/elevated` zonder argument om het huidige niveau te bekijken.

## Werking

<Steps>
  <Step title="Beschikbaarheid controleren">
    Elevated moet in de configuratie zijn ingeschakeld en de afzender moet op de toelatingslijst staan:

    ```json5
    {
      tools: {
        elevated: {
          enabled: true,
          allowFrom: {
            discord: ["user-id-123"],
            whatsapp: ["+15555550123"],
          },
        },
      },
    }
    ```

  </Step>

  <Step title="Het niveau instellen">
    Stuur een bericht dat alleen een directief bevat om de standa ardwaarde voor de sessie in te stellen:

    ```
    /elevated full
    ```

    Of gebruik het inline (geldt alleen voor dat bericht):

    ```
    /elevated on voer het implementatiescript uit
    ```

  </Step>

  <Step title="Opdrachten worden buiten de sandbox uitgevoerd">
    Wanneer elevated actief is, verlaten `exec`-aanroepen de sandbox. De effectieve host is
    standaard `gateway`, of `node` wanneer het geconfigureerde uitvoerdoel of dat van de sessie
    `node` is. In de modus `full` worden exec-goedkeuringen overgeslagen wanneer het opgeloste goedkeuringsbeleid voor de exec-
    modus/host al volledig permissief is (beveiliging `full`,
    vragen `off`); anders blijft het normale goedkeuringsbeleid van toepassing. In de modus
    `on`/`ask` zijn de geconfigureerde goedkeuringsregels altijd van toepassing.
  </Step>
</Steps>

## Volgorde van verwerking

1. **Inline directief** in het bericht (geldt alleen voor dat bericht)
2. **Sessieoverschrijving** (ingesteld door een bericht te sturen dat alleen een directief bevat)
3. **Globale standaardwaarde** (`agents.defaults.elevatedDefault` in de configuratie)

## Beschikbaarheid en toelatingslijsten

- **Globale poort**: `tools.elevated.enabled` (moet `true` zijn)
- **Toelatingslijst voor afzenders**: `tools.elevated.allowFrom` met lijsten per kanaal
- **Poort per agent**: `agents.entries.*.tools.elevated.enabled` (kan alleen verdere beperkingen opleggen; zowel de globale poort als de poort per agent moeten `true` zijn)
- **Toelatingslijst per agent**: `agents.entries.*.tools.elevated.allowFrom` (de afzender moet overeenkomen met zowel de globale lijst als die per agent)
- **Door het kanaal geleverde terugvaltoelatingslijst**: kanaalplugins kunnen optioneel een terugvaltoelatingslijst aanbieden via een SDK-adapterhook, die wordt gebruikt wanneer `tools.elevated.allowFrom.<provider>` niet is geconfigureerd. Momenteel implementeert geen enkel meegeleverd kanaal deze hook, dus in de praktijk heeft elke provider momenteel een expliciete `tools.elevated.allowFrom.<provider>`-vermelding nodig.
- **Alle poorten moeten worden doorstaan**; anders wordt elevated als niet beschikbaar beschouwd

Indelingen voor vermeldingen in de toelatingslijst:

| Voorvoegsel                  | Komt overeen met                         |
| ----------------------- | ------------------------------- |
| (geen)                  | Afzender-ID, E.164 of From-veld |
| `name:`                 | Weergavenaam van afzender             |
| `username:`             | Gebruikersnaam van afzender                 |
| `tag:`                  | Tag van afzender                      |
| `id:`, `from:`, `e164:` | Expliciete identiteitstoewijzing     |

## Wat elevated niet beheert

- **Toolbeleid**: als `exec` door het toolbeleid wordt geweigerd, kan elevated dit niet omzeilen.
- **Hostselectiebeleid**: elevated verandert `auto` niet in een vrije mogelijkheid om naar een andere host over te schakelen. Het gebruikt de geconfigureerde regels voor het uitvoerdoel of die van de sessie en kiest `node` alleen wanneer het doel al `node` is.
- **Los van `/exec`**: het directief `/exec` past de standaardwaarden voor exec per sessie aan (host, beveiliging, vragen, node) voor geautoriseerde afzenders en vereist de verhoogde modus niet.

<Note>
  De bash-chatopdracht (voorvoegsel `!`; alias `/bash`) is een afzonderlijke poort waarvoor `tools.elevated` moet zijn ingeschakeld naast de eigen vlag `tools.bash.enabled`. Als elevated wordt uitgeschakeld, worden ook `!`-shellopdrachten geblokkeerd.
</Note>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Exec-tool" href="/nl/tools/exec" icon="terminal">
    Uitvoering van shellopdrachten vanuit de agent.
  </Card>
  <Card title="Exec-goedkeuringen" href="/nl/tools/exec-approvals" icon="shield">
    Goedkeurings- en toelatingslijstsysteem voor `exec`.
  </Card>
  <Card title="Sandboxing" href="/nl/gateway/sandboxing" icon="box">
    Sandboxconfiguratie op Gateway-niveau.
  </Card>
  <Card title="Sandbox versus toolbeleid versus elevated" href="/nl/gateway/sandbox-vs-tool-policy-vs-elevated" icon="scale-balanced">
    Hoe de drie poorten tijdens een toolaanroep worden gecombineerd.
  </Card>
</CardGroup>
