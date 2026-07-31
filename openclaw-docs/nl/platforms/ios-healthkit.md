---
read_when:
    - HealthKit-samenvattingen inschakelen op een iOS-node
    - health.summary aanroepen of problemen met ontbrekende statusstatistieken oplossen
    - Controleren welke gezondheidsgegevens een iOS-apparaat kunnen verlaten
summary: Privacybeschermde HealthKit-samenvattingen inschakelen en aanroepen vanaf een iOS-node
title: HealthKit-samenvattingen
x-i18n:
    generated_at: "2026-07-27T05:05:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b8ac13d2870c55e2083a5e3a14c3d04238c2780a9e83d091f31923eb738476af
    source_path: platforms/ios-healthkit.md
    workflow: 16
---

# HealthKit-samenvattingen

OpenClaw kan een alleen-lezen samenvatting van de huidige kalenderdag opvragen bij een
verbonden iPhone- of iPad-node. Het apparaat berekent de totalen op het apparaat en retourneert
alleen stappen, slaapduur, gemiddelde hartslag in rust en het aantal/de duur van
work-outs. Afzonderlijke HealthKit-metingen, bronnen, metagegevens, klinische
dossiers, verwerking op de achtergrond en schrijfbewerkingen worden niet ondersteund.

Deze functie is standaard uitgeschakeld. Hiervoor zijn afzonderlijke toestemming op het iOS-apparaat en
autorisatie op de Gateway vereist.

## Vereisten

- Een iPhone of iPad waarop de OpenClaw iOS-app draait en waarvoor HealthKit aangeeft dat gezondheidsgegevens
  beschikbaar zijn.
- Een verbonden en goedgekeurde iOS-node. Zie [iOS-app instellen](/nl/platforms/ios).
- Een actuele Gateway die de iOS-node kan bereiken.
- Leesbare gezondheidsgegevens voor alle meetwaarden die je verwacht te zien. Een Apple Watch kan
  gegevens bijdragen aan de Apple Health-opslag, maar de OpenClaw watchOS-app is
  niet vereist voor HealthKit-samenvattingen.

## Toegang inschakelen

### 1. Autoriseer de Gateway-opdracht

Voeg `health.summary` toe aan de bestaande `gateway.nodes.commands.allow`-array in
`openclaw.json`. Behoud alle reeds aanwezige opdrachten:

```json5
{
  gateway: {
    nodes: {
      commands: { allow: ["health.summary"] },
    },
  },
}
```

`health.summary` wordt geclassificeerd als privacygevoelig en is volgens de
standaardinstelling van het iOS-platform nooit toegestaan. Een vermelding in `gateway.nodes.commands.deny` heeft voorrang op de
toestaan-vermelding. Zie [Beleid voor Node-opdrachten](/nl/nodes#command-policy).

### 2. Delen inschakelen op het iOS-apparaat

In de iOS-app:

1. Open **Settings -> Permissions** en zoek **Apple Health Summaries** in het
   altijd zichtbare gedeelte **Apple Health**.
2. Tik op **Enable Apple Health Summaries**.
3. Lees de toelichting en kies vervolgens in het toestemmingsvenster van Apple welke Health-categorieën OpenClaw mag lezen.

De schakelaar registreert je uitdrukkelijke keuze om gegevens met OpenClaw te delen. Deze geeft niet aan
dat Apple toestemming heeft verleend voor elke aangevraagde categorie.

Door Health-samenvattingen in te schakelen wordt `health.summary` toegevoegd aan het gedeclareerde opdrachtenoppervlak
van de node. Keur de resulterende update van de nodekoppeling goed:

```bash
openclaw nodes pending
openclaw nodes approve <requestId>
```

Controleer vervolgens of het verbonden iOS-apparaat een effectieve opdracht `health.summary`
beschikbaar stelt:

```bash
openclaw nodes describe --node "<iOS device name>"
```

## De samenvatting van vandaag opvragen

Alleen `today` wordt ondersteund. Deze beslaat de periode vanaf lokale middernacht tot het tijdstip van de aanvraag,
op basis van de huidige kalender en tijdzone van het iOS-apparaat.

```bash
openclaw nodes invoke \
  --node "<iOS device name>" \
  --command health.summary \
  --params '{"period":"today"}' \
  --json
```

Agents kunnen dezelfde opdracht aanroepen met de tool `nodes`:

```json
{
  "action": "invoke",
  "node": "<iOS device name>",
  "invokeCommand": "health.summary",
  "invokeParamsJson": "{\"period\":\"today\"}"
}
```

De samenvattingspayload bevat:

| Veld                     | Betekenis                                         |
| ------------------------ | ------------------------------------------------- |
| `period`       | Altijd `today`                         |
| `startISO`       | Lokaal begin van de dag, gecodeerd als ISO-tijdstip |
| `endISO`       | Tijdstip van de aanvraag, gecodeerd als ISO-tijdstip |
| `timeZoneIdentifier`       | Tijdzone-id van het iOS-apparaat                  |
| `stepCount`       | Afgerond cumulatief aantal stappen                |
| `sleepDurationMinutes`       | Ontdubbelde slaaptijd, beperkt tot vandaag        |
| `restingHeartRateBpm`       | Gemiddelde hartslag in rust                       |
| `workoutCount`       | Work-outs die vandaag zijn begonnen               |
| `workoutDurationMinutes`       | Totale duur van die work-outs                     |

Meetwaardevelden zijn optioneel en worden weggelaten wanneer HealthKit geen leesbare
waarde retourneert. Slaapfasen en overlappende bronnen worden samengevoegd voordat de duur wordt
berekend, zodat dezelfde minuut niet twee keer wordt geteld.

## Privacygedrag

- De aggregatie vindt plaats op het iOS-apparaat. Ruwe metingen verlaten het apparaat niet.
- Het opgevraagde totaal verlaat het apparaat via je Gateway. Wanneer een agent
  dit opvraagt, bereikt het totaal de geconfigureerde AI-provider en kan het in de
  chatgeschiedenis bewaard blijven. Een directe CLI-aanroep retourneert het aan de CLI-operator.
- OpenClaw vraagt alleen leestoegang aan. Het kan geen gezondheidsgegevens toevoegen of wijzigen.
- OpenClaw leest HealthKit alleen wanneer `health.summary` wordt aangeroepen. Er vindt geen
  verwerking van gezondheidsgegevens op de achtergrond plaats.
- HealthKit maakt bewust niet bekend of leestoegang is geweigerd. Een
  ontbrekende meetwaarde kan duiden op geweigerde toegang, het ontbreken van overeenkomende metingen of een niet-beschikbaar
  gegevenstype. OpenClaw kan deze gevallen niet van elkaar onderscheiden.
- De samenvatting is bedoeld als context voor persoonlijke gezondheid en fitness, niet voor diagnoses of
  medisch advies.

Als je het delen wilt stoppen, ga je terug naar **Apple Health Summaries** en tik je op **Turn Off Summaries**.
Het iOS-apparaat verwijdert vervolgens de Health-mogelijkheid en de opdracht `health.summary` uit het
node-oppervlak. Je kunt `health.summary` ook verwijderen uit
`gateway.nodes.commands.allow` om de poort aan de Gateway-zijde te sluiten.

## Probleemoplossing

### Opdracht is niet gedeclareerd door de node

Controleer of Apple Health-samenvattingen zijn ingeschakeld in de iOS-app en of het apparaat verbonden is.
Voer `openclaw nodes pending` uit, keur eventuele updates van mogelijkheden goed en controleer vervolgens
`openclaw nodes describe --node "<iOS device name>"` opnieuw.

### Opdracht vereist uitdrukkelijke aanmelding

Voeg `health.summary` toe aan `gateway.nodes.commands.allow`. Controleer ook of
`gateway.nodes.commands.deny` deze niet bevat; de weigeringslijst heeft voorrang.

### `HEALTH_ACCESS_DISABLED`

De schakelaar voor delen in de app staat uit. Schakel **Apple Health Summaries** in onder
**Settings -> Permissions -> Apple Health** op het iOS-apparaat.

### Samenvatting slaagt, maar meetwaarden ontbreken

Open de Health-app van Apple en controleer of er gegevens voor vandaag bestaan. Controleer
de toegang van OpenClaw in de Health-instellingen van Apple, maar beschouw een leeg resultaat
niet als bewijs dat toegang is geweigerd: HealthKit houdt dat onderscheid bewust verborgen.

### Oudere perioden mislukken

De opdracht accepteert alleen `{"period":"today"}`. Meerdaagse en historische
samenvattingen worden niet ondersteund.

## Gerelateerd

- [iOS-app](/nl/platforms/ios)
- [Nodes](/nl/nodes)
- [Naslaginformatie voor Gateway-configuratie](/nl/gateway/configuration-reference#gateway)
- [Beveiligingsaudit](/nl/gateway/security)
