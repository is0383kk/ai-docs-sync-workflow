---
read_when:
    - Je wilt een snelle diagnose van de kanaalstatus en ontvangers van recente sessies
    - Je wilt een plakbare 'alles'-status voor foutopsporing
summary: CLI-referentie voor `openclaw status` (diagnostiek, controles, gebruiksmomentopnamen)
title: openclaw status
x-i18n:
    generated_at: "2026-07-27T05:41:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 52e8076339216f11ddadf35e0ae8e5604322a47a5a9e2ee305468b2624d7cfde
    source_path: cli/status.md
    workflow: 16
---

Diagnostiek voor kanalen + sessies.

```bash
openclaw status
openclaw status --all
openclaw status --deep
openclaw status --usage
```

| Vlag                    | Beschrijving                                                                                                     |
| ----------------------- | --------------------------------------------------------------------------------------------------------------- |
| `--all`                 | Volledige diagnose (alleen-lezen, geschikt om te plakken). Omvat beveiligingsaudit, compatibiliteit van plugins en probes voor geheugenvectoren. |
| `--deep`                | Voert live probes uit (WhatsApp Web + Telegram + Discord + Slack + Signal). Schakelt ook de beveiligingsaudit in.         |
| `--usage`               | Geeft genormaliseerde gebruiksvensters van providers weer als `X% left`.                                                          |
| `--json`                | Machineleesbare uitvoer.                                                                                        |
| `--verbose` / `--debug` | Geeft vóór het rapport ook de onbewerkte doelresolutie van de Gateway weer.                                                 |

Gewone `openclaw status` blijft op het snelle alleen-lezenpad en markeert het geheugen als
`not checked` in plaats van niet beschikbaar wanneer de geheugeninspectie wordt overgeslagen. Zware
beveiligingsaudits, controles van plugincompatibiliteit en probes voor geheugenvectoren worden overgelaten aan
`openclaw status --all`, `openclaw status --deep`, `openclaw security audit`
en `openclaw memory status --deep`.

## Resolutie van sessie en model

- De uitvoer van de sessiestatus maakt onderscheid tussen `Execution:` en `Runtime:`. `Execution`
  is het sandboxpad (`direct`, `docker/*`), terwijl `Runtime` aangeeft
  of de sessie `OpenClaw Default`, `OpenAI Codex`, een CLI-
  backend of een ACP-backend zoals `codex (acp/acpx)` gebruikt. Zie
  [Agentruntimes](/nl/concepts/agent-runtimes) voor het onderscheid tussen provider, model en runtime.
- Wanneer de huidige sessiesnapshot weinig gegevens bevat, kan `/status` de token-
  en cachetellers aanvullen vanuit het meest recente gebruikslogboek van het transcript. Bestaande
  niet-nul livewaarden hebben nog steeds voorrang op terugvalwaarden uit het transcript.
- Terugval op het transcript kan ook het label van het actieve runtimemodel herstellen wanneer
  dit in de live sessievermelding ontbreekt. Als dat transcriptmodel afwijkt
  van het geselecteerde model, bepaalt de status het contextvenster aan de hand van het
  herstelde runtimemodel in plaats van het geselecteerde model.
- Voor de berekening van de promptgrootte geeft terugval op het transcript de voorkeur aan het grotere
  promptgerichte totaal wanneer sessiemetadata ontbreekt of kleiner is, zodat
  sessies van aangepaste providers niet terugvallen op tokenweergaven van `0`.
- Wanneer een sessie is vastgezet op een model dat afwijkt van het geconfigureerde
  primaire model, geeft de status beide waarden, de reden (`session override`) en
  de aanwijzing `/model default` weer. Het geconfigureerde primaire model geldt voor nieuwe of
  niet-vastgezette sessies; bestaande vastgezette sessies behouden hun sessieselectie
  totdat deze wordt gewist.
- De uitvoer omvat sessieopslag per agent wanneer meerdere agents zijn
  geconfigureerd.

## Gebruik en quotum

- `--usage` geeft genormaliseerde gebruiksvensters van providers weer als `X% left`.
- De onbewerkte velden `usage_percent` / `usagePercent` van MiniMax geven het resterende quotum aan,
  dus keert OpenClaw ze om vóór weergave; op aantallen gebaseerde velden hebben voorrang wanneer
  ze aanwezig zijn. `model_remains`-antwoorden geven de voorkeur aan de vermelding van het chatmodel, leiden zo nodig het
  vensterlabel af uit tijdstempels en nemen de modelnaam op in
  het abonnementslabel.
- Mislukte vernieuwingen van modelprijzen worden weergegeven als optionele prijswaarschuwingen.
  Ze betekenen niet dat de Gateway of kanalen niet goed functioneren.

## Overzicht en updatestatus

- Het overzicht bevat, indien beschikbaar, de installatie- en runtimestatus van de Gateway- en node-hostservice,
  plus een compacte procesuptime van de Gateway en systeemuptime van de host.
- Het overzicht bevat het updatekanaal + de git-SHA (voor broncodecheck-outs).
- Update-informatie verschijnt in het overzicht; als er een update beschikbaar is, geeft de status
  een aanwijzing om `openclaw update` uit te voeren (zie [Bijwerken](/nl/install/updating)).

## Geheimen

- Wanneer de actieve Gateway een geïsoleerde SecretRef-eigenaar heeft door het opstarten, opnieuw laden of schrijven van configuratie, bevat de status `degradedSecretOwners` in JSON en een overzichtsrij **Verslechterde geheimen** in voor mensen leesbare uitvoer. Elke vermelding noemt de eigenaar, de verslechteringsstatus (`cold` of `stale`), configuratiepaden en de geredigeerde reden. Koude eigenaren zijn niet beschikbaar; verouderde eigenaren gaan door met de laatst bekende geldige waarden.
- Alleen-lezenstatusoppervlakken (`status`, `status --json`, `status --all`)
  lossen ondersteunde SecretRefs voor hun gerichte configuratiepaden op wanneer
  dat mogelijk is.
- Als een ondersteunde SecretRef voor een kanaal is geconfigureerd maar niet beschikbaar is in het
  huidige commandopad, blijft de status alleen-lezen en rapporteert deze verslechterde uitvoer
  in plaats van te crashen. Voor mensen leesbare uitvoer toont waarschuwingen zoals "geconfigureerd token
  niet beschikbaar in dit commandopad", en JSON-uitvoer bevat
  `secretDiagnostics`.
- Wanneer opdrachtlokale SecretRef-resolutie slaagt, geeft de status de voorkeur aan de
  opgeloste snapshot en worden tijdelijke kanaalmarkeringen voor "geheim niet beschikbaar"
  uit de uiteindelijke uitvoer verwijderd.
- `status --all` bevat een overzichtsrij Geheimen en een diagnosesectie
  die diagnostiek voor geheimen samenvat (ingekort voor leesbaarheid) zonder
  het genereren van het rapport te stoppen.

## Geheugen

`status --json --all` rapporteert geheugendetails vanuit de actieve runtime van de geheugenplugin
die door `plugins.slots.memory` is geselecteerd. Aangepaste geheugenplugins kunnen de ingebouwde
`memory.search.enabled` uitgeschakeld laten en toch hun eigen status voor
bestanden, fragmenten, vectoren en FTS rapporteren.

## Gerelateerd

- [CLI-referentie](/nl/cli)
- [Doctor](/nl/gateway/doctor)
