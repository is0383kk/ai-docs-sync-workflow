---
read_when: You want a dedicated explanation of sandboxing or need to tune agents.defaults.sandbox.
sidebarTitle: Sandboxing
status: active
summary: 'Hoe sandboxing in OpenClaw werkt: modi, bereiken, werkruimtetoegang en images'
title: Sandboxing
x-i18n:
    generated_at: "2026-07-27T05:34:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a3668dc512a8ff30732290ee68e9dd29a3a2e9c106e6e39077a97bfbd90098f7
    source_path: gateway/sandboxing.md
    workflow: 16
---

OpenClaw kan tooluitvoering binnen een sandbox-backend uitvoeren om de impact te beperken. Sandboxing is standaard uitgeschakeld en wordt geregeld door `agents.defaults.sandbox` (globaal) of `agents.entries.*.sandbox` (per agent). Het Gateway-proces blijft altijd op de host; alleen de tooluitvoering wordt naar de sandbox verplaatst wanneer deze is ingeschakeld.

<Note>
Dit is geen perfecte beveiligingsgrens, maar het beperkt de toegang tot het bestandssysteem en processen aanzienlijk wanneer het model iets doms doet.
</Note>

## Wat in de sandbox wordt uitgevoerd

- Tooluitvoering: `exec`, `read`, `write`, `edit`, `apply_patch`, `process`, enzovoort.
- De optionele browser in de sandbox (`agents.defaults.sandbox.browser`).

Niet in de sandbox uitgevoerd:

- Het Gateway-proces zelf.
- Elke tool waarvoor via `tools.elevated` expliciet is toegestaan dat deze buiten de sandbox wordt uitgevoerd. Uitvoering met verhoogde bevoegdheden omzeilt sandboxing en vindt plaats via het geconfigureerde ontsnappingspad (standaard `gateway`, of `node` wanneer het uitvoeringsdoel `node` is). Als sandboxing is uitgeschakeld, verandert `tools.elevated` niets, omdat de uitvoering al op de host plaatsvindt. Zie [Modus met verhoogde bevoegdheden](/nl/tools/elevated).

## Modi, bereik en backend

Drie onafhankelijke instellingen bepalen het sandboxgedrag:

| Instelling | Sleutel                           | Waarden                      | Standaard |
| ---------- | --------------------------------- | ---------------------------- | --------- |
| Modus      | `agents.defaults.sandbox.mode`    | `off`, `non-main`, `all`     | `off`    |
| Bereik     | `agents.defaults.sandbox.scope`   | `agent`, `session`, `shared` | `agent`  |
| Backend    | `agents.defaults.sandbox.backend` | `docker`, `ssh`, `openshell` | `docker` |

**Modus** bepaalt wanneer sandboxing wordt toegepast:

- `off`: geen sandboxing.
- `non-main`: voer elke sessie in een sandbox uit, behalve de hoofdsessie van de agent. De sleutel van de hoofdsessie is altijd `agent:<agentId>:main` (of `global` wanneer `session.scope` gelijk is aan `"global"`); deze is niet configureerbaar. Groeps-/kanaalsessies gebruiken hun eigen sleutels, waardoor ze altijd als niet-hoofdsessies gelden en in een sandbox worden uitgevoerd.
- `all`: elke sessie wordt in een sandbox uitgevoerd.

**Bereik** bepaalt hoeveel containers/omgevingen worden aangemaakt:

- `agent`: één container per agent.
- `session`: één container per sessie.
- `shared`: één container die door alle sessies in een sandbox wordt gedeeld (overschrijvingen per agent voor `docker`/`ssh`/`browser` worden binnen dit bereik genegeerd).

**Backend** bepaalt welke runtime tools in de sandbox uitvoert. SSH-specifieke configuratie staat onder `agents.defaults.sandbox.ssh`; OpenShell-specifieke configuratie staat onder `plugins.entries.openshell.config`.

|                          | Docker                           | SSH                              | OpenShell                                                |
| ------------------------ | -------------------------------- | -------------------------------- | -------------------------------------------------------- |
| **Waar het wordt uitgevoerd** | Lokale container                  | Elke via SSH toegankelijke host  | Door OpenShell beheerde sandbox                          |
| **Installatie**          | `scripts/sandbox-setup.sh`       | SSH-sleutel + doelhost           | OpenShell-plugin ingeschakeld                            |
| **Werkruimtemodel**      | Bind-mount of kopie              | Extern canoniek (eenmalig vullen) | `mirror` of `remote`                                |
| **Netwerkbeheer**        | `docker.network` (standaard: geen) | Afhankelijk van de externe host  | Afhankelijk van OpenShell                                |
| **Browsersandbox**       | Ondersteund                      | Niet ondersteund                 | Nog niet ondersteund                                     |
| **Bind-mounts**          | `docker.binds`                   | N.v.t.                           | N.v.t.                                                   |
| **Het meest geschikt voor** | Lokale ontwikkeling, volledige isolatie | Uitbesteden aan een externe machine | Beheerde externe sandboxes met optionele tweerichtingssynchronisatie |

## Docker-backend

Docker is de standaardbackend zodra sandboxing is ingeschakeld. Het voert tools en sandboxbrowsers lokaal uit via de Docker-daemonsocket (`/var/run/docker.sock`); de isolatie wordt geleverd door Docker-namespaces.

Standaardwaarden: `network: "none"` (geen uitgaand verkeer), `readOnlyRoot: true`, `capDrop: ["ALL"]`, image `openclaw-sandbox:bookworm-slim`.

Stel `agents.defaults.sandbox.docker.gpus` (of de overschrijving per agent) in op een waarde zoals `"all"` of `"device=GPU-uuid"` om host-GPU's beschikbaar te maken. Dit wordt doorgegeven aan de Docker-vlag `--gpus` en vereist een compatibele hostruntime, zoals NVIDIA Container Toolkit.

<Warning>
**Beperkingen van Docker-out-of-Docker (DooD)**

Als je de OpenClaw Gateway zelf als Docker-container implementeert, beheert deze naastliggende sandboxcontainers via de Docker-socket van de host (DooD). Dit introduceert een beperking voor padtoewijzing:

- **Configuratie vereist hostpaden**: `openclaw.json` `workspace` moet het **absolute pad van de host** bevatten (bijvoorbeeld `/home/user/.openclaw/workspaces`), niet het interne pad van de Gateway-container. De Docker-daemon evalueert paden ten opzichte van de naamruimte van het hostbesturingssysteem, niet de eigen naamruimte van de Gateway.
- **Overeenkomende volumetoewijzing vereist**: het Gateway-proces schrijft ook Heartbeat- en bridgebestanden naar dat `workspace`-pad. Geef de Gateway-container een identieke volumetoewijzing (`-v /home/user/.openclaw:/home/user/.openclaw`), zodat hetzelfde hostpad ook vanuit de Gateway-container correct wordt herkend. Niet-overeenkomende toewijzingen verschijnen als `EACCES` wanneer de Gateway zijn Heartbeat probeert te schrijven.
- **Codex-codemodus**: wanneer een OpenClaw-sandbox actief is, schakelt OpenClaw voor die beurt de systeemeigen Code Mode van de Codex-appserver, MCP-servers van gebruikers en door apps ondersteunde Plugin-uitvoering uit (deze worden uitgevoerd vanuit het appserverproces op de Gateway-host, niet vanuit de OpenClaw-sandboxbackend), tenzij het toolbeleid van de sandbox de vereiste tools beschikbaar stelt en je je aanmeldt voor het experimentele uitvoeringsserverpad van de sandbox. Shelltoegang verloopt vervolgens via door de OpenClaw-sandbox ondersteunde tools, zoals `sandbox_exec` en `sandbox_process`. Mount de Docker-socket van de host niet in sandboxcontainers van agents of aangepaste Codex-sandboxes. Zie [Codex-harnas](/nl/plugins/codex-harness) voor het volledige gedrag.

Op Ubuntu-/AppArmor-hosts waarop de Docker-sandboxmodus is ingeschakeld, vereist de shelluitvoering via `workspace-write` van de Codex-appserver onbevoorrechte gebruikersnaamruimten in de sandboxcontainer. Dit kan mislukken voordat de shell wordt gestart wanneer de servicegebruiker deze niet kan aanmaken. Hiervoor is ook een onbevoorrechte netwerknaamruimte nodig wanneer uitgaand verkeer vanuit de Docker-sandbox is uitgeschakeld (`network: "none"`, de standaardwaarde). Veelvoorkomende symptomen: `bwrap: setting up uid map: Permission denied` en `bwrap: loopback: Failed RTM_NEWADDR: Operation not permitted`. Voer `openclaw doctor` uit; als dit een fout meldt bij de Codex-bwrap-naamruimtecontrole, geef dan de voorkeur aan een AppArmor-profiel dat het OpenClaw-serviceproces de vereiste naamruimten toestaat. `kernel.apparmor_restrict_unprivileged_userns=0` is een hostbrede terugvaloptie met beveiligingsafwegingen; gebruik deze alleen wanneer die beveiligingshouding voor de host aanvaardbaar is.
</Warning>

### Browser in de sandbox

- De sandboxbrowser wordt automatisch gestart (zodat CDP bereikbaar is) wanneer de browsertool deze nodig heeft. Configureer dit via `agents.defaults.sandbox.browser.autoStart` (standaard `true`) en `autoStartTimeoutMs` (standaard 12s).
- Sandboxbrowsercontainers gebruiken een speciaal Docker-netwerk (`openclaw-sandbox-browser`) in plaats van het globale `bridge`-netwerk. Configureer dit met `agents.defaults.sandbox.browser.network`.
- `agents.defaults.sandbox.browser.cdpSourceRange` beperkt inkomend CDP-verkeer aan de containerrand met een CIDR-toestaanlijst (bijvoorbeeld `172.21.0.1/32`).
- Waarnemerstoegang via noVNC is standaard met een wachtwoord beveiligd; OpenClaw genereert een URL met een kortlevend token die een lokale opstartpagina aanbiedt en noVNC opent met het wachtwoord in het URL-fragment (niet in de querytekenreeks of headerlogboeken).
- `agents.defaults.sandbox.browser.allowHostControl` (standaard `false`) laat sessies in een sandbox expliciet de hostbrowser als doel gebruiken.
- Optionele toestaanlijsten bewaken `target: "custom"`: `allowedControlUrls`, `allowedControlHosts`, `allowedControlPorts`.

## SSH-backend

Gebruik `backend: "ssh"` om `exec`, bestandstools en het lezen van media in een sandbox uit te voeren op een willekeurige via SSH toegankelijke machine.

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "ssh",
        scope: "session",
        workspaceAccess: "rw",
        ssh: {
          target: "user@gateway-host:22",
          workspaceRoot: "/tmp/openclaw-sandboxes",
          strictHostKeyChecking: true,
          updateHostKeys: true,
          identityFile: "~/.ssh/id_ed25519",
          certificateFile: "~/.ssh/id_ed25519-cert.pub",
          knownHostsFile: "~/.ssh/known_hosts",
          // Of gebruik SecretRefs / inline-inhoud in plaats van lokale bestanden:
          // identityData: { source: "env", provider: "default", id: "SSH_IDENTITY" },
          // certificateData: { source: "env", provider: "default", id: "SSH_CERTIFICATE" },
          // knownHostsData: { source: "env", provider: "default", id: "SSH_KNOWN_HOSTS" },
        },
      },
    },
  },
}
```

Standaardwaarden: `command: "ssh"`, `workspaceRoot: "/tmp/openclaw-sandboxes"`, `strictHostKeyChecking: true`, `updateHostKeys: true`.

- **Levenscyclus**: OpenClaw maakt onder `sandbox.ssh.workspaceRoot` een externe hoofdmap per bereik aan. Bij het eerste gebruik na aanmaken of opnieuw aanmaken wordt die externe werkruimte eenmaal gevuld vanuit de lokale werkruimte. Daarna worden `exec`, `read`, `write`, `edit`, `apply_patch`, het lezen van promptmedia en het klaarzetten van inkomende media rechtstreeks via SSH uitgevoerd op de externe werkruimte. OpenClaw synchroniseert externe wijzigingen niet automatisch terug naar de lokale werkruimte.
- **Authenticatiemateriaal**: `identityFile`/`certificateFile`/`knownHostsFile` verwijzen naar bestaande lokale bestanden. `identityData`/`certificateData`/`knownHostsData` accepteren inline-tekenreeksen of SecretRefs, die via de normale runtime-snapshot voor geheimen worden opgelost, met modus `0600` naar tijdelijke bestanden worden geschreven en worden verwijderd wanneer de SSH-sessie eindigt. Als voor hetzelfde item zowel een `*File`- als een `*Data`-variant is ingesteld, heeft `*Data` voor die sessie voorrang.
- **Gevolgen van extern canoniek gebruik**: de externe SSH-werkruimte wordt na de eerste vulling de werkelijke sandboxstatus. Lokale wijzigingen op de host die na het vullen buiten OpenClaw worden aangebracht, zijn extern niet zichtbaar totdat je de sandbox opnieuw aanmaakt. `openclaw sandbox recreate` verwijdert de externe hoofdmap per bereik en vult deze bij het volgende gebruik opnieuw vanuit de lokale werkruimte. Browsersandboxing wordt niet ondersteund op deze backend en de instellingen van `sandbox.docker.*` zijn er niet op van toepassing.

## OpenShell-backend

Gebruik `backend: "openshell"` om tools in een door OpenShell beheerde externe omgeving in een sandbox uit te voeren. OpenShell hergebruikt hetzelfde SSH-transport en dezelfde externe bestandssysteembridge als de algemene SSH-backend en voegt de OpenShell-levenscyclus (`sandbox create/get/delete/ssh-config`) plus een optionele `mirror`-modus voor werkruimtesynchronisatie toe.

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
        scope: "session",
        workspaceAccess: "rw",
      },
    },
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "remote", // spiegelen | extern
        },
      },
    },
  },
}
```

`mode: "mirror"` (standaard) houdt de lokale werkruimte canoniek: OpenClaw synchroniseert lokaal naar de sandbox vóór `exec` en synchroniseert daarna terug. `mode: "remote"` initialiseert de externe werkruimte eenmaal vanuit de lokale werkruimte en voert vervolgens `exec`/`read`/`write`/`edit`/`apply_patch` rechtstreeks uit op de externe werkruimte zonder terug te synchroniseren; lokale wijzigingen na de initialisatie zijn niet zichtbaar totdat je `openclaw sandbox recreate`. Onder `scope: "agent"` of `scope: "shared"` wordt die externe werkruimte binnen hetzelfde bereik gedeeld. Huidige beperkingen: de sandboxbrowser wordt nog niet ondersteund en `sandbox.docker.binds` is niet van toepassing op deze backend.

`openclaw sandbox list`/`recreate`/opschonen behandelen OpenShell-runtimes allemaal hetzelfde als Docker-runtimes; de opschoonlogica houdt rekening met de backend.

Zie [OpenShell](/nl/gateway/openshell) voor alle vereisten, het configuratieoverzicht, de vergelijking van werkruimtemodi en details over de levenscyclus.

## Toegang tot de werkruimte

`agents.defaults.sandbox.workspaceAccess` bepaalt wat de sandbox kan zien:

| Waarde            | Gedrag                                                                                  |
| ---------------- | ----------------------------------------------------------------------------------------- |
| `none` (standaard) | Tools zien een geïsoleerde sandboxwerkruimte onder `~/.openclaw/sandboxes`.                    |
| `ro`             | Koppelt de agentwerkruimte als alleen-lezen aan `/agent` (schakelt `write`/`edit`/`apply_patch` uit). |
| `rw`             | Koppelt de agentwerkruimte als lezen/schrijven aan `/workspace`.                                    |

Met de OpenShell-backend gebruikt de modus `mirror` nog steeds de lokale werkruimte als canonieke bron tussen uitvoeringsbeurten, gebruikt de modus `remote` na de initiële initialisatie de externe OpenShell-werkruimte als canonieke bron en beperken `workspaceAccess: "ro"`/`"none"` het schrijfgedrag nog steeds op dezelfde manier.

Inkomende media worden naar de actieve sandboxwerkruimte gekopieerd (`media/inbound/*`).

<Note>
**Skills**: de tool `read` is geworteld in de sandbox. Met `workspaceAccess: "none"` spiegelt OpenClaw geschikte Skills naar de sandboxwerkruimte (`.../skills`), zodat ze kunnen worden gelezen. Met `"rw"` zijn werkruimte-Skills leesbaar vanuit `/workspace/skills` en worden geschikte beheerde, gebundelde of Plugin-Skills beschikbaar gemaakt via het gegenereerde alleen-lezenpad `/workspace/.openclaw/sandbox-skills/skills`.
</Note>

## Meerdere mappen voor één agent

Gebruik Docker-bindmounts wanneer één agent in een sandbox meer nodig heeft dan de primaire werkruimte. Elk item wijst een hostmap toe aan een containerpad met een expliciete toegangsmodus:

```text
host-directory:container-directory:ro
host-directory:container-directory:rw
```

- `ro` maakt de gekoppelde map alleen-lezen binnen de sandbox.
- `rw` staat toe dat tools en processen in de sandbox de hostmap wijzigen.
- Het containerpad is het pad dat de agent gebruikt. Hostpaden worden niet automatisch blootgesteld.

Dit voorbeeld geeft de agent `research` een beschrijfbare primaire werkruimte, alleen-lezenreferentiemateriaal op `/reference` en een afzonderlijke beschrijfbare uitvoermap op `/drafts`:

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        scope: "agent",
      },
    },
    list: [
      {
        id: "research",
        workspace: "/srv/openclaw/research-workspace",
        sandbox: {
          workspaceAccess: "rw",
          docker: {
            binds: ["/srv/shared/reference:/reference:ro", "/srv/shared/drafts:/drafts:rw"],
            // Vereist omdat deze bronnen zich buiten de agentwerkruimte bevinden.
            dangerouslyAllowExternalBindSources: true,
          },
        },
      },
    ],
  },
}
```

`workspaceAccess` en bindmodi zijn onafhankelijk:

| Instelling                          | Bepaalt                                                                    |
| -------------------------------- | --------------------------------------------------------------------------- |
| `workspaceAccess: "none"`        | Gebruikt een geïsoleerde sandboxwerkruimte; stelt de agentwerkruimte niet beschikbaar.    |
| `workspaceAccess: "ro"`          | Koppelt de agentwerkruimte als alleen-lezen aan `/agent`.                           |
| `workspaceAccess: "rw"`          | Koppelt de agentwerkruimte als lezen/schrijven aan `/workspace`.                      |
| `docker.binds`-item `:ro`/`:rw` | Bepaalt alleen de toegang tot die aanvullende hostmap via het geconfigureerde containerpad. |

Het wijzigen van `workspaceAccess` verandert een aanvullende bind niet van `ro` in `rw`, of omgekeerd. Globale en agentspecifieke `docker.binds` worden samengevoegd. Behoud `scope: "agent"` of `"session"` voor agentspecifieke binds; `scope: "shared"` negeert alle agentspecifieke Docker-overschrijvingen en gebruikt alleen globale binds.

Bindmounts vormen de ondersteunde grens voor meerdere mappen, omdat Docker met mountisolatie de bestandssysteemweergave van de container samenstelt en de modus `ro`/`rw` van toepassing is op elk proces in de sandbox. Die grens omvat `exec`, bestandssysteemtools, onderliggende processen en bibliotheken, zonder padmachtigingscontroles in elk OpenClaw-codepad te dupliceren. Een allowlist voor paden aan de hostzijde kan niet dezelfde volledige afbakening bieden wanneer een toegestane shell of afhankelijkheid rechtstreeks toegang tot bestanden kan krijgen.

De optionele `dangerouslyAllowExternalBindSources` staat alleen bronnen buiten de werkruimtehoofdmappen toe. Hiermee worden OpenClaws controles op geblokkeerde systeemlocaties, referenties, Docker-sockets, symlink-bovenliggende mappen of gereserveerde doelen niet uitgeschakeld. Geef de voorkeur aan de kleinste map, gebruik `ro` tenzij schrijftoegang vereist is en maak de sandbox opnieuw aan nadat je mounts hebt gewijzigd:

```bash
openclaw sandbox recreate --agent research
```

### Overig bindgedrag

`agents.defaults.sandbox.docker.binds` configureert globale mounts. De indeling is dezelfde `host:container:mode`-vorm (bijvoorbeeld `"/home/user/source:/source:rw"`).

`agents.defaults.sandbox.browser.binds` koppelt aanvullende hostmappen alleen aan de container van de **sandboxbrowser**. Wanneer dit is ingesteld (inclusief `[]`), vervangt het `docker.binds` voor de browsercontainer; wanneer het is weggelaten, valt de browsercontainer terug op `docker.binds`.

```json5
{
  agents: {
    defaults: {
      sandbox: {
        docker: {
          binds: ["/home/user/source:/source:ro", "/var/data/myapp:/data:ro"],
        },
      },
    },
    list: [
      {
        id: "build",
        sandbox: {
          docker: {
            binds: ["/mnt/cache:/cache:rw"],
          },
        },
      },
    ],
  },
}
```

<Warning>
**Bindbeveiliging**

- Binds omzeilen het bestandssysteem van de sandbox: ze stellen hostpaden beschikbaar met de modus die je instelt (`:ro` of `:rw`).
- OpenClaw blokkeert standaard gevaarlijke bindbronnen: systeempaden (`/etc`, `/proc`, `/sys`, `/dev`, `/root`, `/boot`), Docker-socketmappen (`/run`, `/var/run` en hun `docker.sock`-varianten) en gangbare hoofdmappen voor referenties in de thuismap (`~/.aws`, `~/.cargo`, `~/.config`, `~/.docker`, `~/.gnupg`, `~/.netrc`, `~/.npm`, `~/.ssh`).
- De validatie normaliseert het bronpad en herleidt het vervolgens opnieuw via de diepste bestaande bovenliggende map voordat geblokkeerde paden en toegestane hoofdmappen opnieuw worden gecontroleerd. Daardoor worden ontsnappingen via bovenliggende symlinkmappen standaard geweigerd, zelfs wanneer het uiteindelijke blad nog niet bestaat (bijvoorbeeld: `/workspace/run-link/new-file` wordt nog steeds herleid tot `/var/run/...` als `run-link` daarheen verwijst).
- Binddoelen die de gereserveerde containerkoppelpunten (`/workspace`, `/agent`) overschaduwen, worden eveneens standaard geblokkeerd; overschrijf dit met `agents.defaults.sandbox.docker.dangerouslyAllowReservedContainerTargets: true`.
- Bindbronnen buiten de toegestane hoofdmappen van de werkruimte/agentwerkruimte worden standaard geblokkeerd; overschrijf dit met `agents.defaults.sandbox.docker.dangerouslyAllowExternalBindSources: true`. Toegestane hoofdmappen worden op dezelfde manier gecanonicaliseerd. Daardoor wordt een pad dat vóór het herleiden van symlinks alleen binnen de allowlist lijkt te liggen, toch geweigerd omdat het buiten de toegestane hoofdmappen ligt.
- Gevoelige mounts (geheimen, SSH-sleutels, servicereferenties) moeten `:ro` zijn, tenzij iets anders absoluut noodzakelijk is.
- Combineer dit met `workspaceAccess: "ro"` als je alleen leestoegang tot de werkruimte nodig hebt; bindmodi blijven onafhankelijk.
- Zie [Sandbox versus toolbeleid versus verhoogde rechten](/nl/gateway/sandbox-vs-tool-policy-vs-elevated) voor hoe binds samenwerken met toolbeleid en uitvoering met verhoogde rechten.

</Warning>

## Images en installatie

Standaard Docker-image: `openclaw-sandbox:bookworm-slim`

<Note>
**Broncheckout versus npm-installatie**

De hulpscripts `scripts/sandbox-setup.sh`, `scripts/sandbox-common-setup.sh` en `scripts/sandbox-browser-setup.sh` zijn alleen beschikbaar wanneer je vanuit een [broncheckout](https://github.com/openclaw/openclaw) werkt. Ze zijn niet opgenomen in het npm-pakket.

Als je OpenClaw via `npm install -g openclaw` hebt geïnstalleerd, gebruik je in plaats daarvan de onderstaande inline `docker build`-opdrachten.
</Note>

<Steps>
  <Step title="De standaardimage bouwen">
    Vanuit een broncheckout:

    ```bash
    scripts/sandbox-setup.sh
    ```

    Vanuit een npm-installatie (geen broncheckout nodig):

    ```bash
    docker build -t openclaw-sandbox:bookworm-slim - <<'DOCKERFILE'
    FROM debian:bookworm-slim
    ENV DEBIAN_FRONTEND=noninteractive
    RUN apt-get update && apt-get install -y --no-install-recommends \
      bash ca-certificates curl git jq python3 ripgrep \
      && rm -rf /var/lib/apt/lists/*
    RUN useradd --create-home --shell /bin/bash sandbox
    USER sandbox
    WORKDIR /home/sandbox
    CMD ["sleep", "infinity"]
    DOCKERFILE
    ```

    De standaardimage bevat **geen** Node. Als een Skill Node (of andere runtimes) nodig heeft, bouw je een aangepaste image of installeer je deze via `sandbox.docker.setupCommand` (vereist uitgaand netwerkverkeer + een beschrijfbare hoofdmap + de rootgebruiker).

    OpenClaw vervangt een ontbrekende `openclaw-sandbox:bookworm-slim` niet stilzwijgend door gewone `debian:bookworm-slim`. Sandboxuitvoeringen die de standaardimage als doel hebben, mislukken direct met een bouwinstructie totdat je deze bouwt, omdat de gebundelde image `python3` bevat voor de schrijf- en bewerkingshulpmiddelen van de sandbox.

  </Step>
  <Step title="Optioneel: de algemene image bouwen">
    Voor een functionelere sandboximage met gangbare tools (bijvoorbeeld `curl`, `jq`, Node 24, pnpm, `python3` en `git`):

    Vanuit een broncheckout:

    ```bash
    scripts/sandbox-common-setup.sh
    ```

    Bouw vanuit een npm-installatie eerst de standaardimage (zie hierboven) en bouw vervolgens de algemene image daarop voort met [`scripts/docker/sandbox/Dockerfile.common`](https://github.com/openclaw/openclaw/blob/main/scripts/docker/sandbox/Dockerfile.common) uit de repository.

    Stel daarna `agents.defaults.sandbox.docker.image` in op `openclaw-sandbox-common:bookworm-slim`.

  </Step>
  <Step title="Optioneel: de image voor de sandboxbrowser bouwen">
    Vanuit een broncheckout:

    ```bash
    scripts/sandbox-browser-setup.sh
    ```

    Bouw vanuit een npm-installatie met [`scripts/docker/sandbox/Dockerfile.browser`](https://github.com/openclaw/openclaw/blob/main/scripts/docker/sandbox/Dockerfile.browser) uit de repository.

  </Step>
</Steps>

Docker-sandboxcontainers worden standaard uitgevoerd **zonder netwerk**. Overschrijf dit met `agents.defaults.sandbox.docker.network`.

<AccordionGroup>
  <Accordion title="Standaardinstellingen van Chromium in de sandboxbrowser">
    De gebundelde image voor de sandboxbrowser past voorzichtige Chromium-opstartvlaggen toe voor gecontaineriseerde workloads:

    - `--remote-debugging-address=127.0.0.1`
    - `--remote-debugging-port=<derived from OPENCLAW_BROWSER_CDP_PORT>`
    - `--user-data-dir=${HOME}/.chrome`
    - `--no-first-run`
    - `--no-default-browser-check`
    - `--disable-dev-shm-usage`
    - `--disable-background-networking`
    - `--disable-breakpad`
    - `--disable-crash-reporter`
    - `--no-zygote`
    - `--metrics-recording-only`
    - `--password-store=basic`
    - `--use-mock-keychain`
    - `--headless=new` wanneer `browser.headless` is ingeschakeld.
    - `--no-sandbox --disable-setuid-sandbox` wanneer `browser.noSandbox` is ingeschakeld.
    - `--disable-3d-apis`, `--disable-gpu`, `--disable-software-rasterizer` standaard; deze opties voor grafische beveiliging helpen containers zonder GPU-ondersteuning. Stel `OPENCLAW_BROWSER_DISABLE_GRAPHICS_FLAGS=0` in als je werklast WebGL of andere 3D-functies nodig heeft.
    - `--disable-extensions` standaard; stel `OPENCLAW_BROWSER_DISABLE_EXTENSIONS=0` in voor flows die afhankelijk zijn van extensies.
    - `--renderer-process-limit=2` standaard; wordt beheerd door `OPENCLAW_BROWSER_RENDERER_PROCESS_LIMIT=<N>`, waarbij `0` de standaardinstelling van Chromium behoudt.

    Als je een ander runtimeprofiel nodig hebt, gebruik je een aangepaste browserimage en geef je je eigen entrypoint op. Gebruik voor lokale Chromium-profielen (buiten een container) `browser.extraArgs` om extra opstartopties toe te voegen.

  </Accordion>
  <Accordion title="Standaardinstellingen voor netwerkbeveiliging">
    - `network: "host"` wordt geblokkeerd.
    - `network: "container:<id>"` wordt standaard geblokkeerd (risico op omzeiling door deelname aan een namespace).
    - Noodoplossing: `agents.defaults.sandbox.docker.dangerouslyAllowContainerNamespaceJoin: true`.

  </Accordion>
</AccordionGroup>

Docker-installaties en de Gateway in een container vind je hier: [Docker](/nl/install/docker)

Voor Docker-implementaties van de Gateway kan `scripts/docker/setup.sh` de sandboxconfiguratie initialiseren. Stel `OPENCLAW_SANDBOX=1` (of `true`/`yes`/`on`) in om dit pad in te schakelen. Overschrijf de socketlocatie met `OPENCLAW_DOCKER_SOCKET`. Volledige installatie- en omgevingsreferentie: [Docker](/nl/install/docker#agent-sandbox).

## setupCommand (eenmalige containerinstallatie)

`setupCommand` wordt **eenmaal** uitgevoerd nadat de sandboxcontainer is gemaakt (niet bij elke uitvoering). De opdracht wordt in de container uitgevoerd via `sh -lc`.

Paden:

- Globaal: `agents.defaults.sandbox.docker.setupCommand`
- Per agent: `agents.entries.*.sandbox.docker.setupCommand`

<AccordionGroup>
  <Accordion title="Veelvoorkomende valkuilen">
    - De standaardwaarde van `docker.network` is `"none"` (geen uitgaand verkeer), waardoor pakketinstallaties mislukken.
    - `docker.network: "container:<id>"` vereist `dangerouslyAllowContainerNamespaceJoin: true` en is uitsluitend bedoeld als noodoplossing.
    - `readOnlyRoot: true` voorkomt schrijfbewerkingen; stel `readOnlyRoot: false` in of bouw een aangepaste image.
    - `user` moet root zijn voor pakketinstallaties (laat `user` weg of stel `user: "0:0"` in).
    - Uitvoering in de sandbox neemt de `process.env` van de host **niet** over. Gebruik `agents.defaults.sandbox.docker.env` (of een aangepaste image) voor API-sleutels van Skills.
    - Waarden in `agents.defaults.sandbox.docker.env` worden als expliciete omgevingsvariabelen voor de Docker-container doorgegeven. Iedereen met toegang tot de Docker-daemon kan deze inspecteren met Docker-metadataopdrachten zoals `docker inspect`. Gebruik een aangepaste image, een gekoppeld geheimenbestand of een ander pad voor het aanleveren van geheimen als deze blootstelling via metadata niet acceptabel is.

  </Accordion>
</AccordionGroup>

## Toolbeleid en ontsnappingsroutes

Beleid voor het toestaan of weigeren van tools wordt nog steeds vóór de sandboxregels toegepast. Als een tool globaal of per agent wordt geweigerd, maakt sandboxing deze niet opnieuw beschikbaar.

`tools.elevated` is een expliciete ontsnappingsroute die `exec` buiten de sandbox uitvoert (standaard `gateway`, of `node` wanneer het uitvoeringsdoel `node` is). `/exec`-instructies zijn alleen van toepassing op geautoriseerde afzenders en blijven per sessie behouden; om `exec` volledig uit te schakelen, gebruik je een weigering in het toolbeleid (zie [Sandbox versus toolbeleid versus verhoogde rechten](/nl/gateway/sandbox-vs-tool-policy-vs-elevated)).

Probleemoplossing:

- `openclaw sandbox list` toont sandboxcontainers, status, overeenkomst met de image, leeftijd, inactieve tijd en de gekoppelde sessie/agent.
- `openclaw sandbox explain [--session <key>] [--agent <id>]` inspecteert de effectieve sandboxmodus, de werkruimte van de host, de runtimewerkmap, Docker-koppelingen, het toolbeleid en configuratiesleutels voor herstel. Het veld `workspaceRoot` blijft de geconfigureerde sandboxroot; `effectiveHostWorkspaceRoot` toont waar de actieve werkruimte zich daadwerkelijk bevindt.
- `openclaw sandbox recreate [--all | --session <key> | --agent <id>] [--browser] [--force]` verwijdert containers/omgevingen, zodat deze bij het volgende gebruik opnieuw worden gemaakt met de huidige configuratie.
- Zie [Sandbox versus toolbeleid versus verhoogde rechten](/nl/gateway/sandbox-vs-tool-policy-vs-elevated) voor het denkmodel achter 'waarom wordt dit geblokkeerd?'.

## Overschrijvingen voor meerdere agents

Elke agent kan de sandbox en tools overschrijven: `agents.entries.*.sandbox` en `agents.entries.*.tools` (plus `agents.entries.*.tools.sandbox.tools` voor het toolbeleid van de sandbox). Zie [Sandbox en tools voor meerdere agents](/nl/tools/multi-agent-sandbox-tools) voor de prioriteitsvolgorde.

## Minimaal voorbeeld voor inschakeling

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",
        scope: "session",
        workspaceAccess: "none",
      },
    },
  },
}
```

## Gerelateerd

- [Sandbox en tools voor meerdere agents](/nl/tools/multi-agent-sandbox-tools) -- overschrijvingen per agent en prioriteitsvolgorde
- [OpenShell](/nl/gateway/openshell) -- installatie van de beheerde sandboxbackend, werkruimtemodi en configuratiereferentie
- [Sandboxconfiguratie](/nl/gateway/config-agents#agentsdefaultssandbox)
- [Sandbox versus toolbeleid versus verhoogde rechten](/nl/gateway/sandbox-vs-tool-policy-vs-elevated) -- problemen oplossen rond 'waarom wordt dit geblokkeerd?'
- [Beveiliging](/nl/gateway/security)
