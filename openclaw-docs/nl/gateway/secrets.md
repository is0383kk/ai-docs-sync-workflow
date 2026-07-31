---
read_when:
    - SecretRefs configureren voor providerreferenties en `auth-profiles.json`-refs
    - Productiegeheimen veilig opnieuw laden, controleren, configureren en toepassen
    - Inzicht in snel afbreken bij opstartfouten, filtering van inactieve oppervlakken en gedrag met de laatst bekende werkende configuratie
sidebarTitle: Secrets management
summary: 'Beheer van geheimen: SecretRef-contract, gedrag van runtime-snapshots en veilig eenrichtingsmatig opschonen'
title: Beheer van geheimen
x-i18n:
    generated_at: "2026-07-27T05:52:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d10989ebbce367c68d28768244d4e3649028af5ab63c9523974352c270a3c55e
    source_path: gateway/secrets.md
    workflow: 16
---

OpenClaw ondersteunt additieve SecretRefs, zodat ondersteunde inloggegevens niet als platte tekst in de configuratie hoeven te staan.

<Note>
Platte tekst blijft werken. SecretRefs zijn opt-in per inloggegeven.
</Note>

<Warning>
Inloggegevens in platte tekst blijven leesbaar voor de agent als ze zich bevinden in bestanden die de agent kan inspecteren, waaronder `openclaw.json`, `auth-profiles.json`, `.env` of gegenereerde `agents/*/agent/models.json`-bestanden. SecretRefs beperken die lokale impact pas wanneer elk ondersteund inloggegeven is gemigreerd en `openclaw secrets audit --check` geen restanten in platte tekst meldt.
</Warning>

## Runtimemodel

- Geheimen worden tijdens de activering direct omgezet in een runtime-snapshot in het geheugen, niet pas wanneer aanvraagpaden ze nodig hebben.
- Bij een koude start van de Gateway wordt een opnieuw te proberen SecretRef-fout geïsoleerd tot een bekende eigenaar buiten de Gateway wanneer die eigenaar isolatie ondersteunt. Toegewezen eigenaarsklassen omvatten modelproviders en skills, media-/TTS-/cronproviders, geschikte authenticatieprofielen, geheugen per agent, sandbox-SSH, kanaalaccounts en door het manifest gedeclareerde Plugin-routes. De Gateway start, registreert de eigenaar als geconfigureerd maar niet beschikbaar en geeft een geredigeerde waarschuwing over de verminderde werking. Gateway-ingangsauthenticatie, structureel ongeldige refs of omgezette waarden, fail-closed-eigenaren en refs waarvan de runtime-eigenaar niet is toegewezen, laten het opstarten nog steeds mislukken.
- Bij opnieuw laden wordt elke toegewezen eigenaar afzonderlijk gevalideerd, waarna één atomische snapshot wordt gepubliceerd. Gezonde eigenaren worden vernieuwd. Een geschikte eigenaar waarbij een fout optreedt, behoudt zijn laatst bekende werkende waarde en wordt alleen verouderd wanneer zijn ref-identiteiten, providerdefinities en volledige niet-geheime eigenaarscontract ongewijzigd zijn; een gewijzigde of nieuwe eigenaar met een fout wordt koud. Een strikte fout wijst het opnieuw laden af en behoudt de actieve snapshot.
- Beleidsschendingen (bijvoorbeeld een authenticatieprofiel in OAuth-modus in combinatie met SecretRef-invoer) laten de activering mislukken vóór de runtimewissel.
- Runtime-aanvragen lezen uitsluitend de actieve snapshot in het geheugen. SecretRef-inloggegevens van modelproviders worden via authenticatieopslag en streamopties als proceslokale sentinels doorgegeven tot aan de uitgang. Paden voor uitgaande aflevering (Discord-antwoorden/threadaflevering, Telegram-actieverzendingen) lezen die snapshot eveneens en zetten refs niet voor elke verzending opnieuw om.

Hierdoor blijven storingen bij geheimenproviders buiten veelgebruikte aanvraagpaden.

Gateway-ingangsbeveiliging, structureel ongeldige configuratie of omgezette waarden, beleidsschendingen en onbekend eigenaarschap blijven fail-closed mislukken. Geïsoleerde eigenaren vallen nooit terug op een bron van inloggegevens met een lagere prioriteit.

## Injectie bij uitgang (sentinels)

Voor inloggegevens van modelproviders die door SecretRefs worden ondersteund, genereert OpenClaw tijdens de omzetting van modelauthenticatie een ondoorzichtige, proceslokale sentinel. Authenticatieopslag, streamopties, SDK-configuratie, logboeken, foutobjecten en de meeste runtime-inspectie zien daarom een waarde zoals `oc-sent-v1-...`, niet het inloggegeven van de provider. De beveiligde model-fetch en beheerde statuscontroles van lokale providers vervangen bekende sentinels in URL- en headerwaarden onmiddellijk voordat elke aanvraag het proces verlaat.

Onbekende waarden met de vorm van een sentinel mislukken fail-closed voordat netwerkactiviteit plaatsvindt. OpenClaw weigert de aanvraag te verzenden in plaats van een niet-omgezette sentinel naar een provider door te sturen. Omgezette geheime waarden worden ook geregistreerd voor logredactie op basis van exacte waarden, als aanvullende beveiligingsmaatregel.

Provideradapters gebruiken het laatst mogelijke injectiepunt dat hun SDK ondersteunt:

- SDK's met een aangepaste fetch-optie ontvangen de beveiligde fetch van OpenClaw, zodat de SDK de sentinel behoudt.
- SDK's zonder aangepaste fetch-optie pakken de sentinel onmiddellijk vóór het maken van de client uit. Providerstreams en agentharnassen die eigendom zijn van een Plugin pakken deze uit bij de laatste overdracht die eigendom is van de kern, omdat deze transporten de beveiligde fetch van OpenClaw niet delen.

Sentinels beperken blootstelling van platte tekst in de keten van modelaanroepen, maar bieden geen procesisolatie. De echte waarde bestaat nog steeds in het geheugen van hetzelfde proces en verschijnt bij de laatste adaptergrens. Gewone inloggegevens uit de omgeving die niet via SecretRefs zijn geconfigureerd, blijven platte tekst en vallen buiten dit mechanisme.

Stel `OPENCLAW_SECRET_SENTINELS=off` in (accepteert ook `0` of `false`, niet hoofdlettergevoelig) om het genereren van sentinels tijdens incidentrespons of compatibiliteitsprobleemoplossing uit te schakelen. De noodschakelaar schakelt de registratie voor redactie op basis van exacte waarden niet uit.

## Grens voor agenttoegang

SecretRefs voorkomen dat inloggegevens in configuratiebestanden en gegenereerde modelbestanden worden opgeslagen, maar vormen geen grens voor procesisolatie. Een inloggegeven in platte tekst dat op schijf achterblijft op een pad dat de agent kan lezen, blijft leesbaar via bestands- of shelltools, waarbij redactie op API-niveau wordt omzeild.

Beschouw voor productie-implementaties waarbij voor de agent toegankelijke bestanden binnen het bereik vallen, de migratie alleen als voltooid wanneer aan al deze voorwaarden is voldaan:

- Ondersteunde inloggegevens gebruiken SecretRefs in plaats van waarden in platte tekst.
- Achtergebleven verouderde platte tekst is verwijderd uit `openclaw.json`, `auth-profiles.json`, `.env` en gegenereerde `models.json`-bestanden.
- `openclaw secrets audit --check` is na de migratie schoon.
- Alle resterende niet-ondersteunde of roterende inloggegevens worden beschermd door isolatie van het besturingssysteem, containerisolatie of een externe proxy voor inloggegevens.

Daarom is de workflow voor controleren/configureren/toepassen een beveiligingspoort voor migratie en niet alleen een gemakshulpmiddel.

<Warning>
SecretRefs maken willekeurige leesbare bestanden niet veilig. Back-ups, gekopieerde configuraties, oude gegenereerde modelcatalogi en niet-ondersteunde klassen van inloggegevens blijven productiegeheimen totdat ze zijn verwijderd, buiten de vertrouwensgrens van de agent zijn verplaatst of afzonderlijk zijn geïsoleerd.
</Warning>

## Filteren op actieve oppervlakken

SecretRefs worden alleen gevalideerd op daadwerkelijk actieve oppervlakken:

- **Ingeschakelde oppervlakken**: opnieuw te proberen fouten voor toegewezen, isoleerbare eigenaren leiden tot koude of verouderde verminderde werking. Strikte, fail-closed, voor de Gateway vereiste of niet-toegewezen fouten blokkeren het opstarten/opnieuw laden.
- **Inactieve oppervlakken**: niet-omgezette refs blokkeren het opstarten/opnieuw laden niet; ze geven een niet-fatale `SECRETS_REF_IGNORED_INACTIVE_SURFACE`-diagnose.

<Accordion title="Voorbeelden van inactieve oppervlakken">
- Uitgeschakelde kanaal-/accountvermeldingen.
- Kanaalinloggegevens op het hoogste niveau die door geen enkel ingeschakeld account worden overgenomen.
- Uitgeschakelde tool-/functieoppervlakken.
- Providerspecifieke sleutels voor zoeken op het web die niet door `tools.web.search.provider` zijn geselecteerd. In de automatische modus (provider niet ingesteld) worden sleutels volgens prioriteit geraadpleegd voor automatische detectie totdat er één wordt omgezet; na selectie zijn sleutels van niet-geselecteerde providers inactief.
- Sandbox-SSH-authenticatiemateriaal (`agents.defaults.sandbox.ssh.identityData`, `certificateData`, `knownHostsData`, plus overschrijvingen per agent) is alleen actief wanneer de effectieve sandbox-backend `ssh` is en de sandboxmodus niet `off` is, voor de standaardagent of een ingeschakelde agent.
- `gateway.remote.token` / `gateway.remote.password` SecretRefs zijn actief als aan een van deze voorwaarden wordt voldaan:
  - `gateway.mode=remote`
  - `gateway.remote.url` is geconfigureerd
  - `gateway.tailscale.mode` is `serve` of `funnel`
  - In lokale modus zonder die externe oppervlakken: `gateway.remote.token` is actief wanneer tokenauthenticatie kan prevaleren en er geen omgevings-/authenticatietoken is geconfigureerd; `gateway.remote.password` is alleen actief wanneer wachtwoordauthenticatie kan prevaleren en er geen omgevings-/authenticatiewachtwoord is geconfigureerd.
- `gateway.auth.token` SecretRef is inactief voor de omzetting van opstartauthenticatie wanneer `OPENCLAW_GATEWAY_TOKEN` is ingesteld, omdat tokeninvoer uit de omgeving voor die runtime prevaleert.

</Accordion>

## Diagnostiek voor Gateway-authenticatieoppervlakken

Wanneer een SecretRef is ingesteld op `gateway.auth.token`, `gateway.auth.password`, `gateway.remote.token` of `gateway.remote.password`, registreert het opstarten/opnieuw laden van de Gateway de toestand van het oppervlak onder code `SECRETS_GATEWAY_AUTH_SURFACE`:

- `active`: de SecretRef maakt deel uit van het effectieve authenticatieoppervlak en moet worden omgezet.
- `inactive`: een ander authenticatieoppervlak prevaleert, of externe authenticatie is uitgeschakeld/niet actief.

De logboekvermelding bevat de reden die het beleid voor actieve oppervlakken heeft gebruikt.

## Voorafgaande controle van onboardingverwijzingen

Wanneer tijdens interactieve onboarding voor SecretRef-opslag wordt gekozen, wordt vóór het opslaan een voorafgaande validatie uitgevoerd:

- Omgevingsrefs: valideert de naam van de omgevingsvariabele en bevestigt dat tijdens de installatie een niet-lege waarde zichtbaar is.
- Providerrefs (`file` of `exec`): valideert de providerselectie, zet `id` om en controleert het type van de omgezette waarde.
- Snelstartworkflow: wanneer `gateway.auth.token` al een SecretRef is, zet onboarding deze vóór de initialisatie van de probe/het dashboard om (voor `env`-, `file`- en `exec`-refs) met dezelfde direct afbrekende poort.

Bij een validatiefout wordt de fout weergegeven en kun je het opnieuw proberen.

## SecretRef-contract

Overal één objectvorm:

```json5
{ source: "env" | "file" | "exec", provider: "default", id: "..." }
```

<Tabs>
  <Tab title="env">
    ```json5
    { source: "env", provider: "default", id: "OPENAI_API_KEY" }
    ```

    Verkorte tekenreeksen worden ook geaccepteerd voor SecretInput-velden:

    ```json5
    "${OPENAI_API_KEY}"
    "$OPENAI_API_KEY"
    ```

    Validatie:

    - `provider` moet overeenkomen met `^[a-z][a-z0-9_-]{0,63}$`
    - `id` moet overeenkomen met `^[A-Z][A-Z0-9_]{0,127}$`

  </Tab>
  <Tab title="file">
    ```json5
    { source: "file", provider: "filemain", id: "/providers/openai/apiKey" }
    ```

    Validatie:

    - `provider` moet overeenkomen met `^[a-z][a-z0-9_-]{0,63}$`
    - `id` moet een absolute JSON-pointer (`/...`) zijn, of de letterlijke waarde `value` voor `singleValue`-providers
    - RFC 6901-escaping in segmenten: `~` wordt `~0`, `/` wordt `~1`

  </Tab>
  <Tab title="exec">
    ```json5
    { source: "exec", provider: "vault", id: "providers/openai/apiKey#value" }
    ```

    Validatie:

    - `provider` moet overeenkomen met `^[a-z][a-z0-9_-]{0,63}$`
    - `id` moet overeenkomen met `^[A-Za-z0-9][A-Za-z0-9._:/#-]{0,255}$` (ondersteunt selectors zoals `secret#json_key`)
    - `id` mag geen `.` of `..` bevatten als door schuine strepen gescheiden padsegmenten (`a/../b` wordt bijvoorbeeld afgewezen)

  </Tab>
</Tabs>

## Providerconfiguratie

Definieer providers onder `secrets.providers`:

```json5
{
  secrets: {
    providers: {
      default: { source: "env" },
      filemain: {
        source: "file",
        path: "~/.openclaw/secrets.json",
        mode: "json", // or "singleValue"
      },
      vault: {
        source: "exec",
        command: "/usr/local/bin/openclaw-vault-resolver",
        args: ["--profile", "prod"],
        passEnv: ["PATH", "VAULT_ADDR"],
        jsonOnly: true,
      },
      "team-secrets": {
        source: "exec",
        pluginIntegration: {
          pluginId: "acme-secrets",
          integrationId: "secret-store",
        },
      },
    },
    defaults: {
      env: "default",
      file: "filemain",
      exec: "vault",
    },
  },
}
```

<Accordion title="Omgevingsprovider">
- Optionele acceptatielijst met exacte namen via `allowlist`.
- Ontbrekende of lege omgevingswaarden laten de omzetting mislukken.

</Accordion>

<Accordion title="Bestandsprovider">
- Leest het lokale bestand op `path`.
- `mode: "json"` (standaard) verwacht een JSON-object als payload en zet `id` om als een JSON-pointer.
- `mode: "singleValue"` verwacht ref-id `"value"` en retourneert de onbewerkte bestandsinhoud (afsluitende nieuwe regel verwijderd).
- Het pad moet de controles op eigenaarschap/machtigingen doorstaan; `timeoutMs` (standaard 5000) en `maxBytes` (standaard 1 MiB) begrenzen het lezen.
- Fail-closed op Windows: als ACL-verificatie niet beschikbaar is voor het pad, mislukt de omzetting. Stel uitsluitend voor vertrouwde paden `allowInsecurePath: true` in voor die provider om de controle over te slaan.

</Accordion>

<Accordion title="Exec-provider">
- Voert het geconfigureerde absolute binaire pad rechtstreeks uit, zonder shell.
- Standaard moet `command` een gewoon bestand zijn, geen symbolische koppeling. Stel `allowSymlinkCommand: true` in om opdrachtpaden met symbolische koppelingen toe te staan (bijvoorbeeld Homebrew-shims) en combineer dit met `trustedDirs` (bijvoorbeeld `["/opt/homebrew"]`), zodat alleen paden van pakketbeheerders in aanmerking komen.
- Ondersteunt `timeoutMs` (standaard 5000), `noOutputTimeoutMs` (standaard gelijk aan `timeoutMs`), `maxOutputBytes` (standaard 1 MiB), de toelatingslijst `env`/`passEnv` en `trustedDirs`.
- `jsonOnly` is standaard `true`. Met `jsonOnly: false` en één aangevraagde id wordt gewone niet-JSON-standaarduitvoer geaccepteerd als de waarde van die id.
- Windows werkt gesloten bij fouten: als ACL-verificatie niet beschikbaar is voor het opdrachtpad, mislukt de omzetting. Stel uitsluitend voor vertrouwde paden `allowInsecurePath: true` in voor die provider om de controle over te slaan.
- Door plugins beheerde exec-providers kunnen `pluginIntegration` gebruiken in plaats van een gekopieerde `command`/`args`. OpenClaw haalt tijdens het starten/herladen de actuele opdrachtgegevens uit het manifest van de geïnstalleerde plugin; als de plugin is uitgeschakeld, verwijderd of niet vertrouwd wordt, of de integratie niet meer declareert, werken actieve SecretRefs bij die provider gesloten bij fouten.

Aanvraagpayload (stdin):

```json
{ "protocolVersion": 1, "provider": "vault", "ids": ["providers/openai/apiKey"] }
```

Antwoordpayload (stdout):

```jsonc
{ "protocolVersion": 1, "values": { "providers/openai/apiKey": "<openai-api-key>" } } // pragma: allowlist secret
```

Optionele fouten per id:

```json
{
  "protocolVersion": 1,
  "values": {},
  "errors": { "providers/openai/apiKey": { "code": "NOT_FOUND" } }
}
```

`code` is een optionele, machineleesbare diagnose. OpenClaw toont de herkende
codes `NOT_FOUND` en `AMBIGUOUS_DUPLICATE_KEY` met de provider en ref-id. Andere
codes en vrije velden zoals `message` worden geaccepteerd voor compatibiliteit met protocol-v1,
maar worden niet weergegeven omdat resolveruitvoer referentiemateriaal kan bevatten.

</Accordion>

## API-sleutels uit bestanden

Plaats geen `file:...`-tekenreeksen in het `env`-blok van de configuratie. Dat blok is letterlijk en niet-overschrijvend, dus `file:...` wordt daar nooit omgezet.

Gebruik in plaats daarvan een SecretRef naar een bestand in een ondersteund referentieveld:

```json5
{
  secrets: {
    providers: {
      xai_key_file: {
        source: "file",
        path: "~/.openclaw/secrets/xai-api-key.txt",
        mode: "singleValue",
      },
    },
  },
  models: {
    providers: {
      xai: {
        apiKey: { source: "file", provider: "xai_key_file", id: "value" },
      },
    },
  },
}
```

Voor `mode: "singleValue"` is de SecretRef `id` gelijk aan `"value"`. Gebruik voor `mode: "json"` een absolute JSON-pointer, zoals `"/providers/xai/apiKey"`.

Zie [SecretRef-referentieoppervlak](/nl/reference/secretref-credential-surface) voor de velden die SecretRefs accepteren.

## Voorbeelden van exec-integraties

Zie [1Password](/nl/gateway/1password) voor een specifieke 1Password-handleiding over serviceaccounts, de meegeleverde agent-Skill en probleemoplossing.

<AccordionGroup>
  <Accordion title="1Password CLI">
    ```json5
    {
      secrets: {
        providers: {
          onepassword_openai: {
            source: "exec",
            command: "/opt/homebrew/bin/op",
            allowSymlinkCommand: true, // vereist voor binaire Homebrew-bestanden met symbolische koppelingen
            trustedDirs: ["/opt/homebrew"],
            args: ["read", "op://Personal/OpenClaw QA API Key/password"],
            passEnv: ["HOME"],
            jsonOnly: false,
          },
        },
      },
      models: {
        providers: {
          openai: {
            baseUrl: "https://api.openai.com/v1",
            models: [{ id: "gpt-5", name: "gpt-5" }],
            apiKey: { source: "exec", provider: "onepassword_openai", id: "value" },
          },
        },
      },
    }
    ```
  </Accordion>
  <Accordion title="Bitwarden Secrets Manager (`bws`)">
    Gebruik een resolverwrapper om SecretRef-id's toe te wijzen aan itemsleutels van Bitwarden Secrets Manager. De repository bevat `scripts/secrets/openclaw-bws-resolver.mjs`; installeer of kopieer deze naar een absoluut vertrouwd pad op de host waarop de Gateway draait.

    Vereisten:

    - Bitwarden Secrets Manager CLI (`bws`) geïnstalleerd op de Gateway-host.
    - `BWS_ACCESS_TOKEN` beschikbaar voor de Gateway-service.
    - `PATH` doorgegeven aan de resolver, of `BWS_BIN` ingesteld op het absolute pad van het binaire bestand `bws`.
    - `BWS_SERVER_URL` ingesteld in de omgeving bij gebruik van een zelfgehoste Bitwarden-instantie.

    ```json5
    {
      secrets: {
        providers: {
          bws: {
            source: "exec",
            command: "/usr/local/bin/openclaw-bws-resolver.mjs",
            passEnv: ["BWS_ACCESS_TOKEN", "BWS_SERVER_URL", "PATH", "BWS_BIN"],
            jsonOnly: true,
          },
        },
      },
      models: {
        providers: {
          openai: {
            baseUrl: "https://api.openai.com/v1",
            models: [{ id: "gpt-5", name: "gpt-5" }],
            apiKey: {
              source: "exec",
              provider: "bws",
              id: "openclaw/providers/openai/apiKey",
            },
          },
        },
      },
    }
    ```

    De resolver verwerkt aangevraagde id's in batches, voert `bws secret list` uit en retourneert waarden voor overeenkomende geheime `key`-velden. Gebruik sleutels die voldoen aan het id-contract voor exec-SecretRefs, zoals `openclaw/providers/openai/apiKey`; sleutels in de stijl van omgevingsvariabelen met onderstrepingstekens worden geweigerd voordat de resolver wordt uitgevoerd. Als meer dan één zichtbaar Bitwarden-geheim de aangevraagde sleutel deelt, laat de resolver die id mislukken wegens ambiguïteit in plaats van te gokken. Verifieer na het bijwerken van de configuratie het resolverpad:

    ```bash
    openclaw secrets audit --allow-exec
    ```

  </Accordion>
  <Accordion title="HashiCorp Vault CLI">
    ```json5
    {
      secrets: {
        providers: {
          vault_openai: {
            source: "exec",
            command: "/opt/homebrew/bin/vault",
            allowSymlinkCommand: true, // vereist voor binaire Homebrew-bestanden met symbolische koppelingen
            trustedDirs: ["/opt/homebrew"],
            args: ["kv", "get", "-field=OPENAI_API_KEY", "secret/openclaw"],
            passEnv: ["VAULT_ADDR", "VAULT_TOKEN"],
            jsonOnly: false,
          },
        },
      },
      models: {
        providers: {
          openai: {
            baseUrl: "https://api.openai.com/v1",
            models: [{ id: "gpt-5", name: "gpt-5" }],
            apiKey: { source: "exec", provider: "vault_openai", id: "value" },
          },
        },
      },
    }
    ```
  </Accordion>
  <Accordion title="password-store (`pass`)">
    Gebruik een kleine resolverwrapper om SecretRef-id's rechtstreeks toe te wijzen aan `pass`-vermeldingen. Sla deze op als een uitvoerbaar bestand op een absoluut pad dat de padcontroles van je exec-provider doorstaat, bijvoorbeeld `/usr/local/bin/openclaw-pass-resolver`. De `#!/usr/bin/env node`-shebang zoekt `node` via de `PATH` van het resolverproces, dus neem `PATH` op in `passEnv`. Als `pass` niet in die `PATH` staat, stel dan `PASS_BIN` in de bovenliggende omgeving in en neem deze ook op in `passEnv`:

    ```js
    #!/usr/bin/env node
    const { spawnSync } = require("node:child_process");

    let stdin = "";
    process.stdin.setEncoding("utf8");
    process.stdin.on("data", (chunk) => {
      stdin += chunk;
    });
    process.stdin.on("error", (err) => {
      process.stderr.write(`${err.message}\n`);
      process.exit(1);
    });
    process.stdin.on("end", () => {
      let request;
      try {
        request = JSON.parse(stdin || "{}");
      } catch (err) {
        process.stderr.write(`Kan aanvraag niet ontleden: ${err.message}\n`);
        process.exit(1);
      }

      const passBin = process.env.PASS_BIN || "pass";
      const values = {};
      const errors = {};

      for (const id of request.ids ?? []) {
        const result = spawnSync(passBin, ["show", id], { encoding: "utf8" });
        if (result.status === 0) {
          values[id] = result.stdout.split(/\r?\n/, 1)[0] ?? "";
        } else {
          errors[id] = { message: (result.stderr || `pass is afgesloten met ${result.status}`).trim() };
        }
      }

      process.stdout.write(JSON.stringify({ protocolVersion: 1, values, errors }));
    });
    ```

    Configureer vervolgens de exec-provider en laat `apiKey` verwijzen naar het pad van de `pass`-vermelding:

    ```json5
    {
      secrets: {
        providers: {
          pass_store: {
            source: "exec",
            command: "/usr/local/bin/openclaw-pass-resolver",
            passEnv: ["PATH", "HOME", "GNUPGHOME", "GPG_TTY", "PASSWORD_STORE_DIR", "PASS_BIN"],
            jsonOnly: true,
          },
        },
      },
      models: {
        providers: {
          openai: {
            baseUrl: "https://api.openai.com/v1",
            models: [{ id: "gpt-5", name: "gpt-5" }],
            apiKey: {
              source: "exec",
              provider: "pass_store",
              id: "openclaw/providers/openai/apiKey",
            },
          },
        },
      },
    }
    ```

    Bewaar het geheim op de eerste regel van de `pass`-vermelding, of pas de wrapper aan om in plaats daarvan de volledige `pass show`-uitvoer te retourneren. Verifieer na het bijwerken van de configuratie zowel de statische audit als het pad van de exec-resolver:

    ```bash
    openclaw secrets audit --check
    openclaw secrets audit --allow-exec
    ```

  </Accordion>
  <Accordion title="sops">
    ```json5
    {
      secrets: {
        providers: {
          sops_openai: {
            source: "exec",
            command: "/opt/homebrew/bin/sops",
            allowSymlinkCommand: true, // vereist voor binaire Homebrew-bestanden met symbolische koppelingen
            trustedDirs: ["/opt/homebrew"],
            args: ["-d", "--extract", '["providers"]["openai"]["apiKey"]', "/path/to/secrets.enc.json"],
            passEnv: ["SOPS_AGE_KEY_FILE"],
            jsonOnly: false,
          },
        },
      },
      models: {
        providers: {
          openai: {
            baseUrl: "https://api.openai.com/v1",
            models: [{ id: "gpt-5", name: "gpt-5" }],
            apiKey: { source: "exec", provider: "sops_openai", id: "value" },
          },
        },
      },
    }
    ```
  </Accordion>
</AccordionGroup>

## Omgevingsvariabelen van MCP-servers

Omgevingsvariabelen van MCP-servers die via `plugins.entries.acpx.config.mcpServers` zijn geconfigureerd, accepteren SecretInput, zodat API-sleutels en tokens buiten de configuratie in platte tekst blijven:

```json5
{
  plugins: {
    entries: {
      acpx: {
        enabled: true,
        config: {
          mcpServers: {
            github: {
              command: "npx",
              args: ["-y", "@modelcontextprotocol/server-github"],
              env: {
                GITHUB_PERSONAL_ACCESS_TOKEN: {
                  source: "env",
                  provider: "default",
                  id: "MCP_GITHUB_PAT",
                },
              },
            },
          },
        },
      },
    },
  },
}
```

Tekenreekswaarden in platte tekst blijven werken. Verwijzingen naar omgevingssjablonen zoals `${MCP_SERVER_API_KEY}` en SecretRef-objecten worden omgezet tijdens de activering van de Gateway, voordat het MCP-serverproces wordt gestart. Net als bij andere SecretRef-oppervlakken blokkeren niet-omgezette verwijzingen de activering alleen wanneer de plugin `acpx` daadwerkelijk actief is.

## SSH-authenticatiemateriaal voor de sandbox

De kernbackend `ssh` voor de sandbox ondersteunt ook SecretRefs voor SSH-authenticatiemateriaal:

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "ssh",
        ssh: {
          target: "user@gateway-host:22",
          identityData: { source: "env", provider: "default", id: "SSH_IDENTITY" },
          certificateData: { source: "env", provider: "default", id: "SSH_CERTIFICATE" },
          knownHostsData: { source: "env", provider: "default", id: "SSH_KNOWN_HOSTS" },
        },
      },
    },
  },
}
```

Runtimegedrag:

- OpenClaw lost deze verwijzingen op tijdens de activering van de sandbox, niet pas bij elke SSH-aanroep.
- Opgeloste waarden worden naar een tijdelijke map met beperkende bestandsmachtigingen (`0o600`) geschreven en in de gegenereerde SSH-configuratie gebruikt.
- Als de effectieve sandbox-backend niet `ssh` is (of de sandboxmodus `off` is), blijven deze verwijzingen inactief en blokkeren ze het opstarten niet.

## Ondersteund referentiebereik voor inloggegevens

De canonieke ondersteunde en niet-ondersteunde inloggegevens staan vermeld in [Referentiebereik voor SecretRef-inloggegevens](/nl/reference/secretref-credential-surface).

<Note>
Tijdens runtime aangemaakte of roterende inloggegevens en OAuth-vernieuwingsmateriaal zijn bewust uitgesloten van alleen-lezen SecretRef-resolutie.
</Note>

## Vereist gedrag en voorrang

- Veld zonder verwijzing: ongewijzigd.
- Veld met een verwijzing: vereist op actieve oppervlakken tijdens activering.
- Als zowel platte tekst als een verwijzing aanwezig is, krijgt de verwijzing voorrang op ondersteunde voorrangspaden.
- De redactiesentinel `__OPENCLAW_REDACTED__` is gereserveerd voor interne redactie en herstel van configuratie en wordt geweigerd als letterlijk ingediende configuratiegegevens.

Waarschuwings- en auditsignalen:

- `SECRETS_REF_OVERRIDES_PLAINTEXT` (runtimewaarschuwing)
- `REF_SHADOWED` (auditbevinding wanneer `auth-profiles.json`-inloggegevens voorrang krijgen op `openclaw.json`-verwijzingen)

Google Chat `serviceAccount` accepteert inline-JSON of een SecretRef. Doctor verplaatst het buiten gebruik gestelde nevenveld `serviceAccountRef` naar dit canonieke veld wanneer dit niet is ingesteld.

## Activeringstriggers

Geheimactivering wordt uitgevoerd bij:

- Opstarten (voorcontrole plus definitieve activering)
- Hot-apply-pad voor het opnieuw laden van de configuratie
- Pad voor herstartcontrole bij het opnieuw laden van de configuratie
- Handmatig opnieuw laden via `secrets.reload`
- Voorcontrole van de Gateway-RPC voor het schrijven van configuratie (`config.set` / `config.apply` / `config.patch`), waarbij SecretRefs op actieve oppervlakken binnen de ingediende configuratiepayload worden gevalideerd voordat wijzigingen worden opgeslagen

Activeringscontract:

- Bij succes wordt de momentopname atomair vervangen.
- Een strikte opstartfout breekt het opstarten van de Gateway af.
- Tijdens een koude start kan een opnieuw te proberen resolutiefout voor een toegewezen, isoleerbare eigenaar die niet de Gateway is, de momentopname publiceren waarbij precies die eigenaar als geconfigureerd maar niet beschikbaar wordt gemarkeerd. Aanvragen voor de eigenaar mislukken met `SECRET_SURFACE_UNAVAILABLE`; eigenaars van modelproviders vallen na het mislukken van een expliciete verwijzing niet terug op inloggegevens uit de omgeving of een authenticatieprofiel.
- Opnieuw laden en herstartcontrole isoleren daarvoor geschikte toegewezen eigenaars. Ongewijzigde verwijzingsidentiteiten met ongewijzigde providerdefinities en een ongewijzigd, volledig, niet-geheim eigenaarscontract behouden hun exacte laatst bekende werkende waarden als verouderd; gewijzigde of nieuw geconfigureerde niet-opgeloste verwijzingen worden alleen voor die eigenaar koud gepubliceerd. Een strikte fout tijdens opnieuw laden behoudt de eerder actieve momentopname.
- `config.set`, `config.apply` en `config.patch` accepteren syntactisch geldige, niet-opgeloste verwijzingen voor isoleerbare eigenaars en retourneren een geredigeerd `degradedSecretOwners`-rapport. Gateway-ingangsauthenticatie, structureel ongeldige configuratie of opgeloste waarden, beleidsschendingen en onbekende eigenaars worden nog steeds geweigerd voordat de schijf wordt gewijzigd.
- Gezonde neveneigenaars worden normaal opgelost en gepubliceerd, zelfs wanneer een andere eigenaar koud of verouderd is.
- Het opgeven van een expliciet kanaaltoken per aanroep aan een uitgaande helper-/toolaanroep activeert SecretRef-activering niet; de activeringspunten blijven opstarten, opnieuw laden en expliciete `secrets.reload`.

## Signalen voor verminderde werking en herstel

Wanneer activering tijdens opnieuw laden na een gezonde toestand mislukt, gaat OpenClaw over naar een toestand met verminderde werking van geheimen en worden eenmalige systeemgebeurtenissen en logcodes uitgezonden:

- `SECRETS_RELOADER_DEGRADED`
- `SECRETS_RELOADER_RECOVERED`

Gedrag:

- Verminderde werking: gezonde eigenaars worden vernieuwd, verouderde eigenaars behouden hun laatst bekende werkende waarde en koude eigenaars blijven niet beschikbaar.
- Hersteld: eenmaal uitgezonden na de volgende geslaagde activering.
- Herhaalde fouten terwijl de werking al verminderd is, worden als waarschuwing vastgelegd, maar de gebeurtenis wordt niet opnieuw uitgezonden.
- Een strikte opstartfout zendt nooit een gebeurtenis voor verminderde werking uit, omdat de runtime nooit actief is geworden. Een geslaagde opstart met koude eigenaars legt de verminderde werking van de eigenaar vast, maar zendt geen gebeurtenis van de herlader uit.
- Opstart- en herlaadfouten die tot een verwijzing beperkt zijn, zenden voor elke getroffen eigenaar een gestructureerde waarschuwing `SECRETS_DEGRADED` uit. Storingen die tot een provider beperkt zijn, zenden één waarschuwing `SECRETS_PROVIDER_DEGRADED` uit met de provider en de volledige lijst van getroffen eigenaars, in plaats van de providerfout per eigenaar te herhalen. Waarschuwingen bevatten een geredigeerde reden, de eigenaarstoestand `cold` of `stale` en de aanwijzing voor opnieuw proberen `openclaw secrets reload`. Ze bevatten nooit opgeloste waarden of SecretRef-id's.
- `openclaw doctor` vermeldt koude en verouderde eigenaars met hun getroffen configuratiepaden, geredigeerde reden en richtlijnen voor opnieuw proberen.

## Resolutie van opdrachtpaden

Opdrachtpaden kunnen via een Gateway-momentopname-RPC ondersteunde SecretRef-resolutie inschakelen. Er gelden twee algemene gedragingen:

<Tabs>
  <Tab title="Strikte opdrachtpaden">
    Bijvoorbeeld externe-geheugenpaden van `openclaw memory` en `openclaw qr --remote` wanneer hiervoor externe gedeelde-geheimverwijzingen nodig zijn. Ze lezen uit de actieve momentopname en mislukken onmiddellijk wanneer een vereiste SecretRef niet beschikbaar is.
  </Tab>
  <Tab title="Alleen-lezen opdrachtpaden">
    Bijvoorbeeld `openclaw status`, `openclaw status --all`, `openclaw channels status`, `openclaw channels resolve`, `openclaw security audit` en alleen-lezen herstelstromen voor Doctor/configuratie. Ook zij geven de voorkeur aan de actieve momentopname, maar werken met verminderde functionaliteit door in plaats van af te breken wanneer een gerichte SecretRef niet beschikbaar is.

    Alleen-lezen gedrag:

    - Wanneer de Gateway actief is, lezen deze opdrachten eerst uit de actieve momentopname.
    - Als de Gateway-resolutie onvolledig is of de Gateway niet beschikbaar is, proberen ze een gerichte lokale terugval voor dat opdrachtoppervlak.
    - Als een gerichte SecretRef nog steeds niet beschikbaar is, gaat de opdracht door met alleen-lezen uitvoer met verminderde functionaliteit en een expliciete diagnose dat de verwijzing is geconfigureerd maar niet beschikbaar is in dit opdrachtpad.
    - Dit gedrag met verminderde functionaliteit is alleen lokaal voor de opdracht; het verzwakt de runtimepaden voor opstarten, opnieuw laden of verzenden/authenticatie niet.

  </Tab>
</Tabs>

Overige opmerkingen:

- Het vernieuwen van de momentopname na rotatie van een backendgeheim wordt afgehandeld door `openclaw secrets reload`.
- Gateway-RPC-methode die door deze opdrachtpaden wordt gebruikt: `secrets.resolve`.

## Workflow voor audit en configuratie

Standaardworkflow voor operators:

<Steps>
  <Step title="Huidige toestand auditen">
    ```bash
    openclaw secrets audit --check
    ```
  </Step>
  <Step title="SecretRefs configureren en toepassen">
    ```bash
    openclaw secrets configure --apply
    ```
  </Step>
  <Step title="Opnieuw auditen">
    ```bash
    openclaw secrets audit --check
    ```
  </Step>
</Steps>

Beschouw de migratie pas als voltooid wanneer de nieuwe audit geen bevindingen oplevert. Als de audit nog steeds plattetekstwaarden in opslag meldt, blijft het risico op toegang door agenten bestaan, zelfs wanneer runtime-API's geredigeerde waarden retourneren.

Als je tijdens `configure` een plan opslaat in plaats van het toe te passen, pas je dat opgeslagen plan vóór de nieuwe audit toe met `openclaw secrets apply --from <plan-path>`.

<AccordionGroup>
  <Accordion title="geheimen auditen">
    Bevindingen omvatten:

    - Plattetekstwaarden in opslag (`openclaw.json`, `auth-profiles.json`, `.env` en gegenereerde `agents/*/agent/models.json`).
    - Resterende gevoelige providerheaders in gegenereerde `models.json`-vermeldingen.
    - Niet-opgeloste verwijzingen.
    - Overschaduwing door voorrang (`auth-profiles.json` krijgt voorrang op `openclaw.json`-verwijzingen).
    - Restanten van verouderde configuratie (`auth.json`, OAuth-herinneringen).

    Opmerking over exec: standaard slaat de audit controles op de oplosbaarheid van exec-SecretRefs over om neveneffecten van opdrachten te voorkomen. Gebruik `openclaw secrets audit --allow-exec` om exec-providers tijdens de audit uit te voeren.

    Opmerking over resterende headers: detectie van gevoelige providerheaders is gebaseerd op heuristieken voor namen (veelvoorkomende namen en fragmenten van authenticatie-/inloggegevensheaders, zoals `authorization`, `x-api-key`, `token`, `secret`, `password` en `credential`).

  </Accordion>
  <Accordion title="geheimen configureren">
    Interactieve helper die:

    - Eerst `secrets.providers` configureert (`env`/`file`/`exec`, toevoegen/bewerken/verwijderen).
    - Je ondersteunde velden met geheimen laat selecteren in `openclaw.json` plus `auth-profiles.json` voor één agentbereik.
    - Rechtstreeks in de doelkiezer een nieuwe `auth-profiles.json`-toewijzing kan maken.
    - SecretRef-gegevens vastlegt (`source`, `provider`, `id`).
    - Voorafgaande resolutie uitvoert en deze onmiddellijk kan toepassen.

    Opmerking over exec: de voorcontrole slaat controles van exec-SecretRefs over, tenzij `--allow-exec` is ingesteld. Als je rechtstreeks vanuit `configure --apply` toepast en het plan exec-verwijzingen/-providers bevat, laat je `--allow-exec` ook voor de toepassingsstap ingesteld.

    Handige modi:

    - `openclaw secrets configure --providers-only`
    - `openclaw secrets configure --skip-provider-setup`
    - `openclaw secrets configure --agent <id>`

    Standaardinstellingen voor toepassen met `configure`:

    - Overeenkomende statische inloggegevens uit `auth-profiles.json` verwijderen voor de geselecteerde providers.
    - Verouderde statische `api_key`-vermeldingen uit `auth.json` verwijderen.
    - Overeenkomende bekende geheimregels verwijderen uit de bestanden `.env` van de effectieve toestand en actieve configuratie (ontdubbeld wanneer beide paden overeenkomen).

  </Accordion>
  <Accordion title="geheimen toepassen">
    Een opgeslagen plan toepassen:

    ```bash
    openclaw secrets apply --from /tmp/openclaw-secrets-plan.json
    openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --allow-exec
    openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
    openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run --allow-exec
    ```

    Opmerking over exec: een proefuitvoering slaat exec-controles over, tenzij `--allow-exec` is ingesteld; de schrijfmodus weigert plannen die exec-SecretRefs/-providers bevatten, tenzij `--allow-exec` is ingesteld.

    Zie [Contract voor het toepassingsplan van geheimen](/nl/gateway/secrets-plan-contract) voor details over het strikte doel-/padcontract en de exacte weigeringsregels.

  </Accordion>
</AccordionGroup>

## Eenrichtingsveiligheidsbeleid

<Warning>
OpenClaw schrijft bewust geen rollbackback-ups die historische geheime waarden in platte tekst bevatten.
</Warning>

Veiligheidsmodel:

- De voorcontrole moet slagen vóór de schrijfmodus.
- Runtimeactivering wordt vóór de commit gevalideerd.
- Toepassen werkt bestanden bij met atomische bestandsvervanging en herstel naar beste vermogen bij fouten.

## Opmerkingen over compatibiliteit met verouderde authenticatie

Voor statische inloggegevens is de runtime niet langer afhankelijk van verouderde authenticatieopslag in platte tekst.

- De bron voor runtime-inloggegevens is de opgeloste momentopname in het geheugen.
- Verouderde statische `api_key`-vermeldingen worden verwijderd wanneer ze worden aangetroffen.
- OAuth-gerelateerd compatibiliteitsgedrag blijft afzonderlijk.

## Opmerking over de webinterface

Sommige SecretInput-unions zijn gemakkelijker te configureren in de onbewerkte editormodus dan in de formuliermodus.

## Gerelateerd

- [Authenticatie](/nl/gateway/authentication) - authenticatie instellen
- [CLI: geheimen](/nl/cli/secrets) - CLI-opdrachten
- [Vault SecretRefs](/nl/plugins/vault) - HashiCorp Vault-provider instellen
- [Omgevingsvariabelen](/nl/help/environment) - prioriteit van omgevingsvariabelen
- [Referentie voor SecretRef-inloggegevens](/nl/reference/secretref-credential-surface) - referentie voor inloggegevens
- [Contract voor het toepassen van het geheimenplan](/nl/gateway/secrets-plan-contract) - details van het plancontract
- [Beveiliging](/nl/gateway/security) - beveiligingsbeleid
