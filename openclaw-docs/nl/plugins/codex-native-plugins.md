---
read_when:
    - Je wilt dat OpenClaw-agents in Codex-modus native Codex-plugins gebruiken
    - Je migreert vanuit de bron geïnstalleerde, door OpenAI samengestelde Codex-plugins
    - Je configureert een bestaande Codex-plugin in een werkruimtemap
    - Je lost problemen op met codexPlugins, de app-inventaris, destructieve acties of diagnostiek voor Plugin-apps
summary: Configureer native Codex-plugins voor OpenClaw-agents in Codex-modus
title: Native Codex-plugins
x-i18n:
    generated_at: "2026-07-27T05:23:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0b1cfa39838d4dbd1f33a1e5b7f52faec4b033f9fa98ef5c029003177c2e27e5
    source_path: plugins/codex-native-plugins.md
    workflow: 16
---

Native ondersteuning voor Codex-plugins laat een OpenClaw-agent in Codex-modus de eigen app- en pluginmogelijkheden van de Codex
app-server gebruiken binnen dezelfde Codex-thread die
de OpenClaw-beurt afhandelt. Pluginaanroepen blijven in het native Codex-transcript;
Codex app-server beheert de app-ondersteunde MCP-uitvoering. OpenClaw vertaalt
Codex-plugins niet naar synthetische `codex_plugin_*` dynamische OpenClaw-tools.

Gebruik deze pagina nadat de basis-[Codex-harness](/nl/plugins/codex-harness)
werkt.

## Vereisten

- De agentruntime moet de native Codex-harness zijn.
- `plugins.entries.codex.enabled` is `true`.
- `plugins.entries.codex.config.codexPlugins.enabled` is `true`.
- De beoogde Codex app-server kan de verwachte marketplace-, plugin- en
  app-inventaris zien.
- Migratie ondersteunt alleen `openai-curated`-plugins waarvan is vastgesteld dat ze
  vanuit de broncode zijn geïnstalleerd in de Codex-bronhomemap.
- Handmatig geconfigureerde `workspace-directory`-plugins vereisen een Codex app-server
  waarvan `plugin/list` `marketplaceKinds` accepteert en waarvan padloze werkruimte-
  samenvattingen `remotePluginId` bevatten. De plugin moet al geïnstalleerd en
  ingeschakeld zijn en de apps waarvan deze eigenaar is, moeten toegankelijk zijn in `app/list`.

`codexPlugins` heeft geen effect op uitvoeringen via OpenClaw-providers, ACP-gespreks-
koppelingen of andere harnesses, omdat die paden nooit Codex
app-server-threads met native `apps`-configuratie maken.

Het Codex-account aan de OpenAI-zijde, de beschikbaarheid van apps en de app-/plugininstellingen voor werkruimten
zijn afkomstig van het aangemelde Codex-account. Zie
[Codex gebruiken met je ChatGPT-abonnement](https://help.openai.com/en/articles/11369540-using-codex-with-your-chatgpt-plan)
voor het OpenAI-account- en beheerdersmodel.

## Snelstart

Bekijk een voorbeeld van de migratie vanuit de Codex-bronhomemap:

```bash
openclaw migrate codex --dry-run
```

Voeg `--verify-plugin-apps` toe om tijdens de migratie `app/list` op de bron aan te roepen en
te vereisen dat elke app waarvan de plugin eigenaar is aanwezig, ingeschakeld en toegankelijk is voordat
native activering wordt gepland:

```bash
openclaw migrate codex --dry-run --verify-plugin-apps
```

Pas de migratie toe wanneer het plan er goed uitziet:

```bash
openclaw migrate apply codex --yes
```

Migratie schrijft expliciete `codexPlugins`-vermeldingen voor geschikte plugins en
roept `plugin/install` van Codex app-server aan voor geselecteerde plugins. Een gemigreerde
configuratie ziet er als volgt uit:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          codexPlugins: {
            enabled: true,
            allow_destructive_actions: true,
            plugins: {
              "google-calendar": {
                enabled: true,
                marketplaceName: "openai-curated",
                pluginName: "google-calendar",
              },
            },
          },
        },
      },
    },
  },
}
```

Migratie blijft beperkt tot `openai-curated`. Om een bestaande
`workspace-directory`-plugin te gebruiken, voeg je deze handmatig toe met de exacte
marketplace-gekwalificeerde `summary.id` die door `plugin/list` wordt geretourneerd. Als
Codex bijvoorbeeld `example-plugin@workspace-directory` retourneert, configureer je die volledige
waarde in plaats van de weergavenaam:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          codexPlugins: {
            enabled: true,
            plugins: {
              "example-plugin": {
                enabled: true,
                marketplaceName: "workspace-directory",
                pluginName: "example-plugin@workspace-directory",
              },
            },
          },
        },
      },
    },
  },
}
```

OpenClaw roept `plugin/install` niet aan en start geen authenticatie voor een
`workspace-directory`-plugin. Installeer, schakel en authenticeer deze in Codex
voordat je het OpenClaw-beleid toevoegt of inschakelt. OpenClaw houdt apps verborgen wanneer
het antwoord de exacte marketplace, plugin-id, detail-id of het bewijs van appgereedheid
weglaat. Als Codex het expliciete `plugin/list`-verzoek voor de werkruimte afwijst,
rapporteert OpenClaw `marketplace_missing` voor elke ingeschakelde werkruimteplugin en
blijven afzonderlijk gedetecteerde beheerde plugins beschikbaar.

Na een wijziging van `codexPlugins` nemen nieuwe Codex-gesprekken de bijgewerkte
appverzameling automatisch over. Voer `/new` of `/reset` uit om het huidige
gesprek te vernieuwen. Voor wijzigingen waarbij plugins worden in- of uitgeschakeld,
is geen herstart van de Gateway vereist.

## Plugins beheren vanuit de chat

`/codex plugins` inspecteert of wijzigt geconfigureerde native Codex-plugins vanuit
dezelfde chat waarin je de Codex-harness bedient:

```text
/codex plugins
/codex plugins list
/codex plugins disable google-calendar
/codex plugins enable google-calendar
```

`/codex plugins` is een alias voor `/codex plugins list`. De lijst toont voor elke
geconfigureerde plugin de sleutel, aan/uit-status, Codex-pluginnaam en marketplace
uit `plugins.entries.codex.config.codexPlugins.plugins`.

`enable`/`disable` schrijven alleen naar `~/.openclaw/openclaw.json`; ze bewerken nooit
`~/.codex/config.toml` en installeren geen nieuwe Codex-plugins. Alleen de eigenaar of een
Gateway-client met het bereik `operator.admin` kan deze uitvoeren.

Door een geconfigureerde plugin in te schakelen, wordt ook de globale
schakelaar `codexPlugins.enabled` ingeschakeld. Als een beheerde plugin uitgeschakeld is weggeschreven omdat de migratie
`auth_required` retourneerde, autoriseer de app dan opnieuw in Codex voordat je deze in OpenClaw inschakelt.
Voor een `workspace-directory`-vermelding wijzigt inschakelen hier alleen het OpenClaw-
beleid; de plugin en app moeten al actief zijn in Codex.

## Hoe native pluginconfiguratie werkt

De integratie houdt drie toestanden bij:

| Toestand     | Betekenis                                                                                                                          |
| ------------ | ---------------------------------------------------------------------------------------------------------------------------------- |
| Geïnstalleerd | Codex heeft de pluginbundel in de runtime van de beoogde app-server.                                                              |
| Ingeschakeld | Codex rapporteert dat de plugin is ingeschakeld en de OpenClaw-configuratie staat deze toe voor beurten van de Codex-harness.      |
| Toegankelijk | Codex app-server bevestigt dat de appvermeldingen van de plugin beschikbaar zijn voor het actieve account en overeenkomen met de geconfigureerde pluginidentiteit. |

Voor `openai-curated`-plugins is migratie de duurzame stap voor installatie en geschiktheid:

- Tijdens de planning leest OpenClaw de details van `plugin/read` uit de Codex-bron
  en controleert het of het account van de Codex app-server in de bron een ChatGPT-abonnementsaccount
  is. Bij een niet-ChatGPT-account of een ontbrekend accountantwoord worden app-ondersteunde
  plugins overgeslagen met `codex_subscription_required`.
- Standaard slaat migratie de `app/list`-aanroep op de bron over: app-ondersteunde bronplugins
  die de accountcontrole doorstaan, worden gepland zonder verificatie van de toegankelijkheid van bronapps,
  en transportfouten bij het opzoeken van het account leiden tot overslaan met `codex_account_unavailable`.
- Met `--verify-plugin-apps` maakt migratie een nieuwe `app/list`-momentopname
  van de bron en vereist het dat elke app waarvan de plugin eigenaar is aanwezig, ingeschakeld en
  toegankelijk is voordat native activering wordt gepland. Transportfouten bij het opzoeken van het account
  vallen dan terug op de app-inventariscontrole van de bron in plaats van direct
  over te slaan.

Voor `workspace-directory`-plugins vindt de configuratie buiten OpenClaw plaats. OpenClaw
bevraagt die marketplace alleen wanneer ten minste één ingeschakelde werkruimtevermelding is
geconfigureerd, zoekt elke plugin op via de exacte `summary.id` en hergebruikt de bestaande
eigenaarschapscontroles van `plugin/read` en gereedheidscontroles van `app/list`. Een niet-geïnstalleerde,
uitgeschakelde, ontoegankelijke of niet-geauthenticeerde plugin stelt geen apps beschikbaar; OpenClaw
probeert geen installatie of authenticatie uit te voeren.

De runtime-appinventaris is de toegankelijkheidscontrole voor de doelsessie, zowel voor
gemigreerde beheerde plugins als voor handmatig geconfigureerde werkruimteplugins. De sessieconfiguratie
van de Codex-harness berekent een beperkende thread-appconfiguratie op basis van de ingeschakelde
en toegankelijke pluginapps; deze wordt niet bij elke beurt opnieuw berekend, dus
`/codex plugins enable`/`disable` zijn alleen van invloed op
nieuwe Codex-gesprekken. Gebruik `/new` of `/reset` om de wijziging in het
huidige gesprek over te nemen.

## Ondersteuningsgrens van V1

- Alleen `openai-curated`-plugins die al in de inventaris van de Codex
  app-server in de bron zijn geïnstalleerd, komen in aanmerking voor migratie.
- De runtime ondersteunt ook expliciete `workspace-directory`-vermeldingen op app-server-
  builds waarvan `plugin/list` `marketplaceKinds` implementeert en
  `remotePluginId` retourneert voor padloze werkruimtesamenvattingen. Deze vermeldingen moeten
  hun exacte marketplace-gekwalificeerde `summary.id` gebruiken en moeten al geïnstalleerd,
  ingeschakeld en voor apps toegankelijk zijn. Een afgewezen verzoek om de werkruimtelijst produceert de
  bestaande `marketplace_missing`-diagnose per plugin; ontbrekend bewijs voor marketplace,
  plugin, details of apps stelt geen werkruimteapp beschikbaar. Beheerde inventaris
  uit het standaardlijstverzoek blijft bruikbaar.
- App-ondersteunde bronplugins moeten de abonnementscontrole tijdens de migratie doorstaan.
  `--verify-plugin-apps` voegt de app-inventariscontrole van de bron toe. Accounts die door de
  abonnementscontrole worden geweigerd, en in verificatiemodus ontoegankelijke/uitgeschakelde/ontbrekende bron-
  apps of fouten bij het vernieuwen van de app-inventaris, worden gerapporteerd als overgeslagen handmatige
  items in plaats van ingeschakelde configuratievermeldingen. Onleesbare plugindetails worden
  vóór de app-inventariscontrole overgeslagen.
- Migratie schrijft expliciete pluginidentiteiten (`marketplaceName` en
  `pluginName`); er worden geen lokale `marketplacePath`-cachepaden geschreven.
- `codexPlugins.enabled` is de enige globale inschakelschakelaar; er is geen
  `plugins["*"]`-jokerteken of configuratiesleutel die willekeurige installatie-
  bevoegdheid verleent.
- Niet-beheerde marketplaces, gecachte pluginbundels, hooks en Codex-configuratie-
  bestanden worden in het migratierapport bewaard voor handmatige beoordeling en niet
  automatisch geactiveerd. De runtime accepteert handmatig geconfigureerde `workspace-directory`-
  vermeldingen; andere marketplaces blijven niet ondersteund.

## App-inventaris en eigenaarschap

OpenClaw leest de Codex-appinventaris via `app/list` van de app-server, bewaart deze
één uur in het geheugen en vernieuwt verouderde of ontbrekende vermeldingen
asynchroon. De cache is proceslokaal; door de CLI of Gateway opnieuw te starten
wordt deze verwijderd en OpenClaw bouwt deze opnieuw op vanaf de volgende `app/list`-lezing.

Migratie en runtime gebruiken afzonderlijke cachesleutels:

- Verificatie van de bronmigratie gebruikt de Codex-bronhomemap en start-
  opties. Deze wordt alleen uitgevoerd met `--verify-plugin-apps` en dwingt voor die planningsuitvoering
  een nieuwe doorloop van `app/list` op de bron af.
- De configuratie van de doelruntime gebruikt de Codex app-server-identiteit van de doelagent bij
  het opbouwen van de thread-appconfiguratie. Activering van een beheerde plugin maakt die
  doelcachesleutel ongeldig en vernieuwt deze daarna geforceerd na `plugin/install`.
  Bij de configuratie van `workspace-directory` wordt dit activeringspad nooit uitgevoerd.

Een pluginapp wordt alleen beschikbaar gesteld wanneer OpenClaw deze via stabiel eigenaarschap kan
terugkoppelen aan de geconfigureerde plugin: een exacte app-id uit de plugindetails, een bekende
MCP-servernaam of unieke stabiele metadata. Eigenaarschap dat alleen op de weergavenaam berust of
dubbelzinnig is, wordt uitgesloten totdat de volgende inventarisvernieuwing het eigenaarschap bewijst.

## Apps van verbonden accounts

Door de eigenaar beheerde agents kunnen zich aanmelden voor elke app die al met hun Codex-
account is verbonden, zonder dat een overeenkomend pluginpakket vereist is:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          codexPlugins: {
            enabled: true,
            allow_all_plugins: true,
            allow_destructive_actions: "auto",
          },
        },
      },
    },
  },
}
```

`allow_all_plugins: true` maakt een volledige `app/list`-momentopname wanneer een nieuwe native
Codex-thread wordt opgezet en laat alleen apps toe die voor dat
account als toegankelijk zijn gemarkeerd. Hiermee worden apps niet globaal geïnstalleerd, geauthenticeerd of ingeschakeld. Bestaande
threads behouden hun opgeslagen appverzameling; gebruik `/new`, `/reset` of start de
Gateway opnieuw om nieuw verbonden of ingetrokken apps over te nemen.

Accountapps nemen de globale waarde `codexPlugins.allow_destructive_actions` over,
die `true`, `false`, `"auto"` of `"ask"` accepteert. Expliciet beleid per Plugin
overschrijft het globale beleid voor overlappende app-id's. Inventarisatiefouten worden
gesloten afgehandeld in plaats van terug te vallen op een onbeperkte standaardinstelling.

## Configuratie van threadapps

OpenClaw injecteert een beperkende `config.apps`-patch voor de Codex-thread:
`_default` is uitgeschakeld en alleen apps die eigendom zijn van ingeschakelde, geconfigureerde Plugins of
toegankelijke accountapps die door `allow_all_plugins` zijn toegelaten, worden ingeschakeld.

`destructive_enabled` voor elke app is afkomstig van het effectieve globale of
per-Plugin `allow_destructive_actions`-beleid; `true`, `"auto"` en `"ask"`
stellen allemaal `destructive_enabled: true` in, en `false` stelt dit in op `false`. Codex blijft
metadata voor destructieve tools afdwingen via de eigen annotaties van de apptools.
`_default` wordt uitgeschakeld met `open_world_enabled: false`; ingeschakelde Plugin-apps
krijgen `open_world_enabled: true`. OpenClaw biedt geen afzonderlijke
beleidsoptie op Plugin-niveau voor een open wereld en onderhoudt geen
weigerlijsten per Plugin met namen van destructieve tools.

De modus voor toolgoedkeuring staat standaard op automatisch voor toegelaten apps, zodat niet-destructieve
leestools zonder goedkeuringsprompt in dezelfde thread worden uitgevoerd. Destructieve tools blijven
onder het `destructive_enabled`-beleid van elke app vallen.

## Beleid voor destructieve acties

Destructieve verzoeken van Plugins zijn standaard toegestaan voor geconfigureerde Codex-
Plugins, terwijl onveilige schema's en dubbelzinnig eigenaarschap gesloten worden afgehandeld:

- Globaal `allow_destructive_actions` is standaard ingesteld op `true`.
- Per-Plugin `allow_destructive_actions` overschrijft het globale beleid voor
  die Plugin.
- `false`: OpenClaw retourneert een deterministische afwijzing.
- `true`: OpenClaw accepteert alleen automatisch veilige schema's die aan een goedkeuringsantwoord
  kunnen worden gekoppeld, zoals een booleaans goedkeuringsveld.
- `"auto"`: OpenClaw stelt destructieve Plugin-acties beschikbaar aan Codex en
  zet vervolgens MCP-goedkeuringsverzoeken waarvan het eigenaarschap is bewezen om in OpenClaw-Plugin-
  goedkeuringen voordat het Codex-goedkeuringsantwoord wordt geretourneerd.
- `"ask"`: OpenClaw gebruikt dezelfde Codex-beperking voor schrijf- en destructieve acties als
  `"auto"`, wist permanente Codex-goedkeuringsoverschrijvingen per tool voor de app
  voordat de thread start, en biedt alleen eenmalige goedkeuring of weigering, zodat
  permanente goedkeuringen latere prompts voor schrijfacties niet kunnen onderdrukken. Voor elke
  toegelaten app die `"ask"` gebruikt, selecteert OpenClaw Codex' beoordelaar voor menselijke goedkeuringen
  voor die app, zodat Codex zijn goedkeuringsverzoeken naar
  OpenClaw stuurt; andere apps en niet-appgebonden threadgoedkeuringen behouden hun geconfigureerde
  beoordelaar en beleid.
- Een ontbrekende Plugin-identiteit, dubbelzinnig eigenaarschap, een ontbrekend of niet-overeenkomend
  beurt-id of een onveilig verzoeksschema leidt tot afwijzing in plaats van een prompt.

## Probleemoplossing

| Code                                              | Betekenis                                                                                                                              | Oplossing                                                                                                                    |
| ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------- |
| `auth_required`                                   | De migratie heeft de Plugin geïnstalleerd, maar een van de apps moet nog worden geverifieerd. De vermelding wordt uitgeschakeld opgeslagen totdat je opnieuw autoriseert. | Autoriseer de app opnieuw in Codex en schakel daarna de Plugin in OpenClaw in.                                                      |
| `app_inaccessible`, `app_disabled`, `app_missing` | Met `--verify-plugin-apps` gaf de inventaris van Codex-bronapps niet aan dat alle apps van de eigenaar aanwezig, ingeschakeld en toegankelijk waren.         | Autoriseer de app opnieuw of schakel deze in Codex in en voer daarna de migratie opnieuw uit met `--verify-plugin-apps`.                              |
| `app_inventory_unavailable`                       | Er is strikte verificatie van bronapps aangevraagd, maar het vernieuwen van de inventaris van Codex-bronapps is mislukt.                                      | Herstel de toegang tot de Codex-appserver van de bron of probeer het opnieuw zonder `--verify-plugin-apps` om het snellere, door het account beperkte plan te accepteren.   |
| `codex_subscription_required`                     | Het account van de Codex-appserver van de bron was geen ChatGPT-abonnementsaccount.                                                          | Meld je met abonnementsverificatie aan bij de Codex-app en voer de migratie daarna opnieuw uit.                                                  |
| `codex_account_unavailable`                       | Het account van de Codex-appserver van de bron kon niet worden gelezen.                                                                               | Herstel de verificatie van de Codex-appserver van de bron of voer de migratie opnieuw uit met `--verify-plugin-apps`, zodat de bronappinventaris bepaalt of de app in aanmerking komt. |
| `marketplace_missing`, `plugin_missing`           | Marketplace of exacte Plugin niet beschikbaar; het expliciete verzoek om de werkruimtecatalogus is mogelijk geweigerd; werkruimteapps worden gesloten afgehandeld.  | Controleer het compatibele appservercontract en de exacte ID die hieronder worden beschreven.                                                |
| `plugin_detail_unavailable`                       | OpenClaw kon de eigendomsgegevens van de Plugin niet lezen.                                                                                    | Inspecteer de antwoorden `plugin/list` en `plugin/read` van de doelappserver.                                             |
| `plugin_disabled`                                 | Codex meldt dat de Plugin is geïnstalleerd maar uitgeschakeld.                                                                                     | Gecureerde activering kan dit herstellen; schakel een werkruimte-Plugin in Codex in voordat je het opnieuw probeert.                                  |
| `plugin_activation_failed`                        | De activering van de Plugin is niet voltooid.                                                                                                  | Gebruik de bijgevoegde diagnostische gegevens om onderscheid te maken tussen fouten in de marketplace, verificatie, vernieuwing of gereedheid van de werkruimte.                |
| `app_inventory_missing`, `app_inventory_stale`    | De gereedheidsstatus van de app kwam uit een lege of verouderde cache.                                                                                     | OpenClaw plant automatisch een asynchrone vernieuwing; Plugin-apps blijven uitgesloten totdat eigenaarschap en gereedheid bekend zijn.  |
| `app_ownership_ambiguous`                         | De appinventaris kwam alleen overeen op basis van de weergavenaam.                                                                                          | De app blijft verborgen voor de Codex-thread totdat een latere vernieuwing het eigenaarschap bewijst.                                     |

**Werkruimte-Plugin is geïnstalleerd maar niet zichtbaar:** controleer of het resultaat voor werkruimte-
`plugin/list` de exact geconfigureerde ID als geïnstalleerd en ingeschakeld meldt,
en controleer daarna of `app/list` meldt dat elke app van de eigenaar toegankelijk is voor hetzelfde Codex-
account. OpenClaw kan een toegankelijke app voor de thread inschakelen, zelfs wanneer de
accountinventaris die app momenteel als uitgeschakeld meldt. Als je die status hebt gewijzigd nadat de Gateway de app-
inventaris in de cache heeft opgeslagen, wacht dan op de cachevernieuwing na één uur of start de Gateway opnieuw en gebruik daarna
`/new` of `/reset`. OpenClaw herstelt of verifieert werkruimte-Plugins niet.
Als het expliciete verzoek om de werkruimtelijst wordt geweigerd, meldt elke ingeschakelde werkruimte-
vermelding `marketplace_missing`; niet-gerelateerde gecureerde vermeldingen gaan nog steeds verder
op basis van het antwoord van de standaardlijst.

Voor `plugin_detail_unavailable` moet een werkruimtesamenvatting zonder pad
`remotePluginId` bevatten; OpenClaw houdt apps van de eigenaar verborgen wanneer die selector of het
daaropvolgende resultaat van `plugin/read` niet beschikbaar is. Voor
`plugin_activation_failed` kunnen gecureerde Plugins een fout in de marketplace, verificatie of
vernieuwing na installatie melden. Een werkruimte-Plugin meldt deze code wanneer deze
nog niet actief is; installeer, activeer en verifieer deze buiten OpenClaw.

**Configuratie gewijzigd, maar de agent kan de Plugin niet zien:** voer `/codex plugins
list` uit om de geconfigureerde status te controleren en daarna `/new` of `/reset`. Bestaande
Codex-threadkoppelingen behouden de appconfiguratie waarmee ze zijn gestart totdat OpenClaw
een nieuwe harness-sessie tot stand brengt of een verouderde koppeling vervangt.

**Destructieve actie wordt geweigerd:** controleer de globale en per-Plugin
`allow_destructive_actions`-waarden. Zelfs met `true`, `"auto"` of `"ask"`
worden onveilige verzoeksschema's en een dubbelzinnige Plugin-identiteit nog steeds gesloten afgehandeld.

## Gerelateerd

- [Codex-harness](/nl/plugins/codex-harness)
- [Codex-harnessreferentie](/nl/plugins/codex-harness-reference)
- [Codex-harnessruntime](/nl/plugins/codex-harness-runtime)
- [Configuratiereferentie](/nl/gateway/configuration-reference#codex-harness-plugin-config)
- [Migratie-CLI](/nl/cli/migrate)
