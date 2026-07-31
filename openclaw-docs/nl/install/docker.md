---
read_when:
    - Je wilt een gecontaineriseerde Gateway in plaats van lokale installaties
    - Je valideert de Docker-flow
summary: Optionele Docker-gebaseerde installatie en onboarding voor OpenClaw
title: Docker
x-i18n:
    generated_at: "2026-07-27T05:02:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c1784bd49f6847db75633840a4d5a8e49205200728bd2e9d59b646a446e508d6
    source_path: install/docker.md
    workflow: 16
---

Docker is **optioneel**. Gebruik het voor een geïsoleerde, tijdelijke Gateway-omgeving of een host zonder lokale installaties. Als je al op je eigen machine ontwikkelt, gebruik dan de normale installatiestroom.

De standaardsandboxbackend gebruikt Docker wanneer `agents.defaults.sandbox` is ingeschakeld, maar sandboxing is standaard uitgeschakeld en vereist niet dat de Gateway zelf in Docker wordt uitgevoerd. SSH- en OpenShell-sandboxbackends zijn ook beschikbaar; zie [Sandboxing](/nl/gateway/sandboxing).

Host je meerdere gebruikers? Zie [Multitenanthosting](/nl/gateway/multi-tenant-hosting) voor het model met één cel per tenant.

## Vereisten

- Docker Desktop (of Docker Engine) + Docker Compose v2
- Minstens 2 GB RAM voor het bouwen van de image (`pnpm install` kan op hosts met 1 GB wegens onvoldoende geheugen worden beëindigd met afsluitcode 137)
- Voldoende schijfruimte voor images en logboeken
- Controleer op een VPS/openbare host [Beveiligingsversterking voor netwerktoegang](/nl/gateway/security), met name de Docker-firewallketen `DOCKER-USER`

## Gateway in een container

<Steps>
  <Step title="Bouw de image">
    Vanuit de hoofdmap van de repository:

    ```bash
    ./scripts/docker/setup.sh
    ```

    Hiermee wordt de Gateway-image lokaal gebouwd als `openclaw:local`. Zo gebruik je in plaats daarvan een vooraf gebouwde image:

    ```bash
    export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"
    ./scripts/docker/setup.sh
    ```

    Vooraf gebouwde images worden eerst gepubliceerd in de [GitHub Container Registry](https://github.com/openclaw/openclaw/pkgs/container/openclaw). GHCR is het primaire register voor releaseautomatisering, vastgezette implementaties en herkomstcontroles. Dezelfde release publiceert een Docker Hub-mirror op `openclaw/openclaw`:

    ```bash
    export OPENCLAW_IMAGE="openclaw/openclaw:latest"
    ./scripts/docker/setup.sh
    ```

    Gebruik `ghcr.io/openclaw/openclaw` of `openclaw/openclaw` en vermijd niet-officiële mirrors, omdat die niet hetzelfde releaseschema of bewaarbeleid als OpenClaw hanteren. Versiespecifieke tags omvatten releases zoals `2026.2.26` en prereleases zoals `2026.2.26-beta.1`. Stabiele releases verplaatsen `latest` en `main`; Gateway-releases voor de voorgaande maand verplaatsen alleen `extended-stable`. Varianten omvatten `slim`, `main-slim`, `extended-stable-slim`, `latest-browser`, `main-browser` en `extended-stable-browser`. De standaardimages bevatten de plugins `codex` en `diagnostics-otel`. Er wordt ook een variant `-browser` geleverd waarin Chromium is ingebouwd, wat handig is voor de tool [browser in een sandbox](/nl/gateway/sandboxing#sandboxed-browser) zonder dat Playwright bij de eerste uitvoering hoeft te worden geïnstalleerd.

  </Step>

  <Step title="Opnieuw uitvoeren zonder netwerkverbinding">
    Draag op offline hosts eerst de image over en laad deze:

    ```bash
    docker load -i openclaw-image.tar
    export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"
    ./scripts/docker/setup.sh --offline
    ```

    `--offline` controleert of `OPENCLAW_IMAGE` al lokaal bestaat, schakelt impliciete pulls/builds van Compose uit en voert vervolgens de normale stroom uit: synchronisatie van `.env`, correcties van machtigingen, onboarding, synchronisatie van de Gateway-configuratie en het starten van Compose.

    Als `OPENCLAW_SANDBOX=1`, controleert de offline-installatie ook de geconfigureerde standaard- en agentspecifieke sandboximages op de daemon achter `OPENCLAW_DOCKER_SOCKET`, inclusief het browsercontractlabel op Docker-gebaseerde browserimages. Als een vereiste image ontbreekt of verouderd is, wordt de installatie afgesloten zonder de sandboxconfiguratie te wijzigen, in plaats van ten onrechte een geslaagd resultaat te melden.

  </Step>

  <Step title="Voltooi de onboarding">
    Het installatiescript voert de onboarding automatisch uit:

    - vraagt om API-sleutels van providers
    - genereert een Gateway-token en schrijft dit naar `.env`
    - maakt de map voor de geheime sleutel van het authenticatieprofiel
    - start de Gateway via Docker Compose

    Onboarding vóór het starten en het schrijven van configuratie worden rechtstreeks uitgevoerd via `openclaw-gateway` (met `--no-deps --entrypoint node`), omdat `openclaw-cli` de netwerknaamruimte van de Gateway deelt en pas werkt nadat de Gateway-container bestaat.

  </Step>

  <Step title="Open de Control UI">
    Open `http://127.0.0.1:18789/` en plak het token dat naar `.env` is geschreven in Settings. Als je de container hebt overgeschakeld op wachtwoordauthenticatie, gebruik dan in plaats daarvan dat wachtwoord.

    Heb je de URL opnieuw nodig?

    ```bash
    docker compose run --rm openclaw-cli dashboard --no-open
    ```

  </Step>

  <Step title="Configureer kanalen (optioneel)">
    ```bash
    # WhatsApp (QR)
    docker compose run --rm openclaw-cli channels login

    # Telegram
    docker compose run --rm openclaw-cli channels add --channel telegram --token "<token>"

    # Discord
    docker compose run --rm openclaw-cli channels add --channel discord --token "<token>"
    ```

    Documentatie: [WhatsApp](/nl/channels/whatsapp), [Telegram](/nl/channels/telegram), [Discord](/nl/channels/discord)

  </Step>
</Steps>

### Handmatige stroom

```bash
BUILD_GIT_COMMIT="$(git rev-parse HEAD)"
BUILD_TIMESTAMP="$(date -u +%Y-%m-%dT%H:%M:%SZ)"
docker build \
  --build-arg "GIT_COMMIT=${BUILD_GIT_COMMIT}" \
  --build-arg "OPENCLAW_BUILD_TIMESTAMP=${BUILD_TIMESTAMP}" \
  -t openclaw:local -f Dockerfile .
docker compose run --rm --no-deps --entrypoint node openclaw-gateway \
  dist/index.js onboard --mode local --no-install-daemon
docker compose run --rm --no-deps --entrypoint node openclaw-gateway \
  dist/index.js config set --batch-json '[{"path":"gateway.mode","value":"local"},{"path":"gateway.bind","value":"lan"},{"path":"gateway.controlUi.allowedOrigins","value":["http://localhost:18789","http://127.0.0.1:18789"]}]'
docker compose up -d openclaw-gateway
```

De Docker-context sluit `.git` uit. Geef de bronidentiteit door als buildargumenten,
zoals hierboven weergegeven, zodat in het scherm Info van de image de uitgecheckte commit en
één buildtijdstempel worden weergegeven. `scripts/docker/setup.sh` bepaalt beide waarden en geeft ze
automatisch door.

<Note>
Voer `docker compose` uit vanuit de hoofdmap van de repository. Als je `OPENCLAW_EXTRA_MOUNTS` of `OPENCLAW_HOME_VOLUME` hebt ingeschakeld, schrijft het installatiescript `docker-compose.extra.yml`; voeg dit toe na elke `docker-compose.override.yml` die je zelf beheert, bijvoorbeeld `-f docker-compose.yml -f docker-compose.override.yml -f docker-compose.extra.yml`.
</Note>

### Containerimages upgraden

Wanneer je de OpenClaw-image vervangt maar dezelfde gekoppelde status/configuratie behoudt, voert de
nieuwe Gateway vóór gereedheid opstartveilige upgrademigraties en Plugin-convergentie uit.
Voor routinematige image-upgrades zou geen afzonderlijke uitvoering van
`openclaw doctor --fix` nodig moeten zijn.

Als tijdens het opstarten deze reparaties niet veilig kunnen worden voltooid, wordt de Gateway afgesloten in plaats van
zich als gezond te melden. Met een herstartbeleid kunnen Docker, Podman of Kubernetes aangeven
dat de Gateway-container opnieuw wordt gestart. Behoud het gekoppelde statusvolume en voer vervolgens
dezelfde image eenmaal uit met `openclaw doctor --fix` als containeropdracht, met
dezelfde status-/configuratiekoppelingen die de Gateway gebruikt:

```bash
docker run --rm -v <openclaw-state>:/home/node/.openclaw <image> openclaw doctor --fix
podman run --rm -v <openclaw-state>:/home/node/.openclaw <image> openclaw doctor --fix
```

Nadat doctor is voltooid, start je de Gateway-container opnieuw met de standaardopdracht.
Voer in Kubernetes dezelfde opdracht uit in een eenmalige Job of debugpod die aan hetzelfde
PVC is gekoppeld en start vervolgens de Deployment of StatefulSet opnieuw.

### Omgevingsvariabelen

Optionele variabelen die door `scripts/docker/setup.sh` worden geaccepteerd (en, voor de Gateway-container, rechtstreeks door `docker-compose.yml`):

| Variabele                                       | Doel                                                                                                              |
| ----------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `OPENCLAW_IMAGE`                                | Een externe image gebruiken in plaats van deze lokaal te bouwen                                                   |
| `OPENCLAW_IMAGE_APT_PACKAGES`                   | Extra apt-pakketten installeren tijdens de build (gescheiden door spaties). Verouderde alias: `OPENCLAW_DOCKER_APT_PACKAGES` |
| `OPENCLAW_IMAGE_PIP_PACKAGES`                   | Extra Python-pakketten installeren tijdens de build (gescheiden door spaties)                                     |
| `OPENCLAW_EXTENSIONS`                           | Geselecteerde ondersteunde plugins compileren/verpakken en hun runtimeafhankelijkheden installeren (ID's gescheiden door komma's of spaties) |
| `OPENCLAW_DOCKER_BUILD_NODE_OPTIONS`            | De Node-opties voor de lokale bronbuild overschrijven (standaard `--max-old-space-size=8192`)                    |
| `OPENCLAW_DOCKER_BUILD_TSDOWN_MAX_OLD_SPACE_MB` | De tsdown-heap voor de lokale bronbuild overschrijven in MB                                                       |
| `OPENCLAW_DOCKER_BUILD_SKIP_DTS`                | Declaratie-uitvoer overslaan tijdens lokale imagebuilds die alleen voor runtime zijn bedoeld (standaard `1`) |
| `OPENCLAW_INSTALL_BROWSER`                      | Chromium + Xvfb tijdens de build in de image inbouwen                                                             |
| `OPENCLAW_EXTRA_MOUNTS`                         | Extra bindmounts van de host (door komma's gescheiden `source:target[:opts]`)                                      |
| `OPENCLAW_HOME_VOLUME`                          | `/home/node` behouden in een benoemd Docker-volume                                                        |
| `OPENCLAW_SANDBOX`                              | Sandboxbootstrap inschakelen (`1`, `true`, `yes`, `on`) |
| `OPENCLAW_SKIP_ONBOARDING`                      | De interactieve onboardingstap overslaan (`1`, `true`, `yes`, `on`) |
| `OPENCLAW_DOCKER_SOCKET`                        | Het pad naar de Docker-socket overschrijven                                                                       |
| `OPENCLAW_DISABLE_BONJOUR`                      | Bonjour-/mDNS-advertenties geforceerd inschakelen (`0`) of uitschakelen (`1`); zie [Bonjour / mDNS](#bonjour--mdns) |
| `OPENCLAW_DISABLE_BUNDLED_SOURCE_OVERLAYS`      | Bindmountoverlays voor de broncode van gebundelde plugins uitschakelen                                            |
| `OTEL_EXPORTER_OTLP_ENDPOINT`                   | Gedeeld OTLP/HTTP-collectoreindpunt voor OpenTelemetry-export                                                     |
| `OTEL_EXPORTER_OTLP_*_ENDPOINT`                 | Signaalspecifieke OTLP-eindpunten voor traces, metrische gegevens of logboeken                                    |
| `OTEL_EXPORTER_OTLP_PROTOCOL`                   | Overschrijving van het OTLP-protocol. Momenteel wordt alleen `http/protobuf` ondersteund                     |
| `OTEL_SERVICE_NAME`                             | Servicenaam die voor OpenTelemetry-resources wordt gebruikt                                                       |
| `OTEL_SEMCONV_STABILITY_OPT_IN`                 | De nieuwste experimentele semantische GenAI-attributen inschakelen                                                |
| `OPENCLAW_OTEL_PRELOADED`                       | Voorkomen dat een tweede OpenTelemetry-SDK wordt gestart wanneer er al een is voorgeladen                          |

De officiële image bevat geen Homebrew. Tijdens de onboarding verbergt OpenClaw installatieprogramma's voor Skill-afhankelijkheden die alleen brew ondersteunen in een Linux-container zonder `brew`; lever deze afhankelijkheden via een aangepaste image of installeer ze handmatig. Gebruik `OPENCLAW_IMAGE_APT_PACKAGES` voor afhankelijkheden uit Debian-pakketten en `OPENCLAW_IMAGE_PIP_PACKAGES` voor Python-afhankelijkheden (voert `python3 -m pip install --break-system-packages` uit tijdens de build; zet versies daarom vast en gebruik alleen indexen die je vertrouwt).

Als Docker `ResourceExhausted` of `cannot allocate memory` meldt, of tijdens `tsdown` wordt afgebroken, verhoog dan de geheugenlimiet van de Docker-builder of probeer het opnieuw met kleinere expliciete heaps:

```bash
OPENCLAW_DOCKER_BUILD_NODE_OPTIONS=--max-old-space-size=4096 OPENCLAW_DOCKER_BUILD_TSDOWN_MAX_OLD_SPACE_MB=4096
```

### Vanuit bron gebouwde images met geselecteerde plugins

`OPENCLAW_EXTENSIONS` selecteert pluginmanifest-id's uit de broncheckout;
bestaande namen van bronmappen worden ook geaccepteerd wanneer ze afwijken. De Docker-
build zet de selectie eenmaal om naar bronmappen, installeert productie-
afhankelijkheden en compileert, wanneer een geselecteerde plugin afzonderlijk wordt gepubliceerd met
`openclaw.build.bundledDist: false`, de runtime ervan in de gebundelde
hoofddistributie. Deze uitsluitend voor Docker bestemde verpakking wijzigt het npm- of ClawHub-
artefactcontract van de plugin niet. Onbekende, ongeldige of dubbelzinnige id's laten de imagebuild mislukken.
Bekende id's die alleen voor afhankelijkheden/bronnen dienen, behouden hun bestaande staging van bronnen en afhankelijkheden
zonder een gecompileerde vermelding in de hoofddistributie te krijgen. Een geselecteerde plugin met
geünificeerde buildvermeldingen moet met succes worden gecompileerd; bron- en runtime-uitvoer van
niet-geselecteerde externe plugins worden verwijderd.

Deze opdrachten bouwen bijvoorbeeld afzonderlijke, zelfstandige FakeCo Gateway-images
voor meerdere architecturen voor ClickClack, Slack en Microsoft Teams. ClawRouter maakt
al deel uit van de OpenClaw-hoofdruntime, dus selecteert de ClickClack-image alleen
`clickclack`. Het expliciete lege browserargument houdt de standaardimage vrij
van Chromium:

```bash
SOURCE_SHA="$(git rev-parse HEAD)"
BUILD_TIMESTAMP="$(date -u +%Y-%m-%dT%H:%M:%SZ)"
REGISTRY="registry.example.com/fakeco"

build_gateway_image() {
  gateway="$1"
  selected_plugin="$2"
  docker buildx build \
    --platform linux/amd64,linux/arm64 \
    --build-arg "GIT_COMMIT=${SOURCE_SHA}" \
    --build-arg "OPENCLAW_BUILD_TIMESTAMP=${BUILD_TIMESTAMP}" \
    --build-arg "OPENCLAW_EXTENSIONS=${selected_plugin}" \
    --build-arg OPENCLAW_INSTALL_BROWSER= \
    --provenance=mode=max \
    --sbom=true \
    --tag "${REGISTRY}/openclaw-${gateway}:${SOURCE_SHA}" \
    --push \
    .
}

build_gateway_image clickclack clickclack
build_gateway_image slack slack
build_gateway_image teams msteams
```

Gebruik `--platform linux/arm64 --load` of `--platform linux/amd64 --load` voor één
native lokale build. Uitvoer voor meerdere platforms en bijgevoegde SBOM/herkomstgegevens
vereisen een registry of andere Buildx-uitvoer die attesten behoudt. Inspecteer na
het pushen het manifest en implementeer de onveranderlijke digest in plaats van de
wijzigbare bron-SHA-tag:

```bash
docker buildx imagetools inspect \
  "${REGISTRY}/openclaw-clickclack:${SOURCE_SHA}"
# Implementeren: registry.example.com/fakeco/openclaw-clickclack@sha256:<manifest-digest>
```

Deze images zijn bedoeld voor zelfstandige OCI-gebaseerde gateways en algemene Docker-gebruikers.
Door Crabhelm beheerde gateways gebruiken ze niet: dat leveringspad bouwt een
afzonderlijk x86_64-appliancearchief met een OpenClaw-npm-tarball en legt
de digests van Node, het archief en het manifest vast. Bouw die appliance onafhankelijk
vanuit dezelfde gelande OpenClaw-bron.

Als je gebundelde pluginbroncode wilt testen tegen een verpakte image, koppel je één pluginbronmap over het verpakte bronpad ervan, bijvoorbeeld `OPENCLAW_EXTRA_MOUNTS=/path/to/fork/extensions/synology-chat:/app/extensions/synology-chat:ro`. Daarmee wordt de overeenkomstige gecompileerde `/app/dist/extensions/synology-chat`-bundel voor hetzelfde plugin-id overschreven.

### Observeerbaarheid

OpenTelemetry-export verloopt uitgaand vanuit de Gateway-container naar je OTLP-collector; hiervoor hoeft geen Docker-poort te worden gepubliceerd. Zo neem je de gebundelde exporter op in een lokaal gebouwde image:

```bash
export OPENCLAW_EXTENSIONS="diagnostics-otel"
export OTEL_EXPORTER_OTLP_ENDPOINT="http://otel-collector:4318"
export OTEL_SERVICE_NAME="openclaw-gateway"
./scripts/docker/setup.sh
```

Officiële vooraf gebouwde images bundelen `diagnostics-otel` al; installeer `clawhub:@openclaw/diagnostics-otel` alleen zelf als je deze hebt verwijderd. Om export in te schakelen, sta je de plugin `diagnostics-otel` toe en schakel je deze in de configuratie in. Stel vervolgens `diagnostics.otel.enabled=true` in (zie het volledige voorbeeld in [OpenTelemetry-export](/nl/gateway/opentelemetry)). Authenticatieheaders voor de collector worden doorgegeven via `diagnostics.otel.headers`, niet via Docker-omgevingsvariabelen.

Prometheus-metrieken gebruiken de al gepubliceerde Gateway-poort opnieuw. Installeer `clawhub:@openclaw/diagnostics-prometheus`, schakel de plugin `diagnostics-prometheus` in en scrape vervolgens:

```text
http://<gateway-host>:18789/api/diagnostics/prometheus
```

De route wordt beschermd door Gateway-authenticatie; stel geen afzonderlijke openbare `/metrics`-poort of niet-geverifieerd reverse-proxypad beschikbaar. Zie [Prometheus-metrieken](/nl/gateway/prometheus).

### Statuscontroles

Probe-eindpunten voor containers (geen authenticatie vereist):

```bash
curl -fsS http://127.0.0.1:18789/healthz   # activiteit
curl -fsS http://127.0.0.1:18789/readyz     # gereedheid
```

De ingebouwde `HEALTHCHECK` van de image pingt `/healthz`; bij herhaalde fouten wordt de container gemarkeerd als `unhealthy`, zodat orchestrators deze opnieuw kunnen starten of vervangen.

Diepgaande geverifieerde statusmomentopname:

```bash
docker compose exec openclaw-gateway node dist/index.js health --token "$OPENCLAW_GATEWAY_TOKEN"
```

### LAN versus loopback

`scripts/docker/setup.sh` gebruikt standaard `OPENCLAW_GATEWAY_BIND=lan`, zodat `http://127.0.0.1:18789` op de host werkt met Docker-poortpublicatie.

- `lan` (standaard): de hostbrowser en host-CLI kunnen de gepubliceerde Gateway-poort bereiken.
- `loopback`: alleen processen binnen de netwerknaamruimte van de container kunnen de Gateway rechtstreeks bereiken.

<Note>
Gebruik bindmoduswaarden in `gateway.bind` (`lan` / `loopback` / `custom` / `tailnet` / `auto`), geen hostaliassen zoals `0.0.0.0` of `127.0.0.1`.
</Note>

### Lokale providers op de host

Binnen de container verwijst `127.0.0.1` naar de container zelf, niet naar de host. Gebruik `host.docker.internal` voor providers die op de host draaien:

| Provider  | Standaard-URL op host    | Docker-installatie-URL              |
| --------- | ------------------------ | ----------------------------------- |
| LM Studio | `http://127.0.0.1:1234`  | `http://host.docker.internal:1234`  |
| Ollama    | `http://127.0.0.1:11434` | `http://host.docker.internal:11434` |

De gebundelde installatie gebruikt die URL's als onboardingstandaarden voor LM Studio/Ollama, en `docker-compose.yml` wijst `host.docker.internal` toe aan de host-Gateway op Linux Docker Engine (Docker Desktop biedt dezelfde alias op macOS/Windows). Hostservices moeten luisteren op een adres dat Docker kan bereiken:

```bash
lms server start --port 1234 --bind 0.0.0.0
OLLAMA_HOST=0.0.0.0:11434 ollama serve
```

Gebruik je je eigen Compose-bestand of `docker run`? Voeg dan zelf dezelfde toewijzing toe, bijvoorbeeld `--add-host=host.docker.internal:host-gateway`.

### Claude CLI-backend in Docker

De officiële image installeert Claude Code niet vooraf. Installeer deze en meld je aan binnen de gebruiker `node` van de container. Maak vervolgens die container-home permanent, zodat image-upgrades het binaire bestand of de authenticatiestatus niet wissen.

Schakel voor een nieuwe installatie een permanent `/home/node`-volume in voordat je de installatie uitvoert:

```bash
export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"
export OPENCLAW_HOME_VOLUME="openclaw_home"
./scripts/docker/setup.sh
```

Stop bij een bestaande installatie de stack en laad eerst de huidige waarden uit `.env` opnieuw — het installatiescript herschrijft `.env` altijd op basis van de huidige shell en standaardwaarden; het leest het bestand niet zelf:

```bash
set -a
. ./.env
set +a
export OPENCLAW_HOME_VOLUME="${OPENCLAW_HOME_VOLUME:-openclaw_home}"
./scripts/docker/setup.sh
```

Als `.env` waarden bevat die je shell niet kan inladen, exporteer dan eerst handmatig opnieuw wat je gebruikt (`OPENCLAW_IMAGE`, poorten, bindmodus, aangepaste paden, `OPENCLAW_EXTRA_MOUNTS`, sandbox, onboarding overslaan). De gegenereerde overlay koppelt het homevolume voor zowel `openclaw-gateway` als `openclaw-cli`; voer de overige opdrachten uit met die overlay (en eerst `docker-compose.override.yml`, als je die gebruikt):

```bash
docker compose -f docker-compose.yml -f docker-compose.extra.yml run --rm \
  --entrypoint sh openclaw-cli -lc \
  'curl -fsSL https://claude.ai/install.sh | bash'
```

Het native installatieprogramma schrijft `claude` naar `/home/node/.local/bin/claude`. De
OpenClaw-image bevat `/home/node/.local/bin` in `PATH`, zodat de gebundelde
Anthropic-plugin deze zonder overschrijving van de adapterconfiguratie kan vinden.

Meld je aan en verifieer vanuit dezelfde permanente home:

```bash
docker compose -f docker-compose.yml -f docker-compose.extra.yml run --rm \
  --entrypoint /home/node/.local/bin/claude openclaw-cli auth login
docker compose -f docker-compose.yml -f docker-compose.extra.yml run --rm \
  --entrypoint /home/node/.local/bin/claude openclaw-cli auth status --text
docker compose -f docker-compose.yml -f docker-compose.extra.yml run --rm \
  openclaw-cli models auth login \
  --provider anthropic --method cli --set-default
docker compose -f docker-compose.yml -f docker-compose.extra.yml run --rm \
  openclaw-cli models list --provider anthropic
```

Gebruik vervolgens de gebundelde `claude-cli`-backend:

```bash
docker compose -f docker-compose.yml -f docker-compose.extra.yml run --rm \
  openclaw-cli agent \
  --agent main \
  --model claude-cli/claude-sonnet-4-6 \
  --message "Zeg hallo vanuit Docker Claude CLI"
```

`OPENCLAW_HOME_VOLUME` bewaart de native installatie onder `/home/node/.local/bin` en `/home/node/.local/share/claude`, plus de instellingen/authenticatie van Claude Code onder `/home/node/.claude` en `/home/node/.claude.json`. Alleen `/home/node/.openclaw` permanent maken is niet voldoende; als je `OPENCLAW_EXTRA_MOUNTS` gebruikt in plaats van een homevolume, koppel dan al die Claude-paden aan beide services.

<Note>
Geef voor gedeelde productieautomatisering of voorspelbare Anthropic-facturering de voorkeur aan het pad met de Anthropic-API-sleutel. Hergebruik van Claude CLI volgt de geïnstalleerde versie, accountaanmelding, facturering en het updategedrag van Claude Code.
</Note>

### Bonjour / mDNS

Docker-bridgenetwerken sturen Bonjour/mDNS-multicast (`224.0.0.251:5353`) doorgaans niet betrouwbaar door. Wanneer `OPENCLAW_DISABLE_BONJOUR` niet is ingesteld, schakelt de gebundelde Bonjour-plugin LAN-advertering automatisch uit zodra deze detecteert dat hij in een container draait. Zo blijft hij niet in een crashlus proberen multicast te verzenden die door de bridge wordt verworpen. Stel `OPENCLAW_DISABLE_BONJOUR=1` in om dit ongeacht de detectie gedwongen uit te schakelen, of `0` om het gedwongen in te schakelen (alleen bij hostnetwerken, macvlan of een ander netwerk waarvan bekend is dat mDNS-multicast werkt).

Gebruik anders de gepubliceerde Gateway-URL, Tailscale of wide-area DNS-SD voor Docker-hosts. Zie [Bonjour-detectie](/nl/gateway/bonjour) voor aandachtspunten en probleemoplossing.

### Opslag en persistentie

Docker Compose koppelt `OPENCLAW_CONFIG_DIR` aan `/home/node/.openclaw`, `OPENCLAW_WORKSPACE_DIR` aan `/home/node/.openclaw/workspace` en `OPENCLAW_AUTH_PROFILE_SECRET_DIR` aan `/home/node/.config/openclaw`, zodat die paden behouden blijven wanneer de container wordt vervangen. Wanneer een variabele niet is ingesteld, valt `docker-compose.yml` terug op een pad onder `${HOME}`, of `/tmp` als `HOME` zelf ontbreekt, zodat `docker compose up` in kale omgevingen nooit een volumespecificatie met een lege bron genereert.

Die gekoppelde configuratiemap bevat:

- `openclaw.json` voor gedragsconfiguratie
- `agents/<agentId>/agent/auth-profiles.json` voor opgeslagen OAuth-/API-sleutelauthenticatie van providers
- `.env` voor door de omgeving geleverde runtimegeheimen zoals `OPENCLAW_GATEWAY_TOKEN`

De geheimenmap voor authenticatieprofielen bevat de lokale encryptiesleutel voor het tokenmateriaal van door OAuth ondersteunde authenticatieprofielen. Bewaar deze bij de statusgegevens van je Docker-host, maar gescheiden van `OPENCLAW_CONFIG_DIR`.

Geïnstalleerde downloadbare plugins slaan pakketstatus op onder de gekoppelde OpenClaw-home, zodat installatierecords en pakkethoofdmappen behouden blijven wanneer de container wordt vervangen; bij het starten van de Gateway worden afhankelijkheidsstructuren van gebundelde plugins niet opnieuw gegenereerd.

Zie [Docker VM-runtime - Wat blijft waar behouden](/nl/install/docker-vm-runtime#what-persists-where) voor volledige details over persistentie van VM's.

**Belangrijkste bronnen van schijfgroei:** `media/`, SQLite-databases per agent, verouderde JSONL-transcripten van sessies, de gedeelde SQLite-statusdatabase, pakketbasismappen van geïnstalleerde plugins en roterende bestandslogboeken onder `/tmp/openclaw/`.

### Shell-hulpfuncties (optioneel)

Installeer [ClawDock](/nl/install/clawdock) voor kortere dagelijkse opdrachten:

```bash
mkdir -p ~/.clawdock && curl -sL https://raw.githubusercontent.com/openclaw/openclaw/main/scripts/clawdock/clawdock-helpers.sh -o ~/.clawdock/clawdock-helpers.sh
echo 'source ~/.clawdock/clawdock-helpers.sh' >> ~/.zshrc && source ~/.zshrc
```

Als je via het oudere pad `scripts/shell-helpers/clawdock-helpers.sh` hebt geïnstalleerd, voer je de bovenstaande opdracht opnieuw uit, zodat je lokale hulpfunctie de huidige locatie volgt. Gebruik daarna `clawdock-start`, `clawdock-stop`, `clawdock-dashboard`, enzovoort (voer `clawdock-help` uit voor de volledige lijst).

<AccordionGroup>
  <Accordion title="Agentsandbox inschakelen voor Docker-gateway">
    ```bash
    export OPENCLAW_SANDBOX=1
    ./scripts/docker/setup.sh
    ```

    Aangepast socketpad (bijv. rootless Docker):

    ```bash
    export OPENCLAW_SANDBOX=1
    export OPENCLAW_DOCKER_SOCKET=/run/user/1000/docker.sock
    ./scripts/docker/setup.sh
    ```

    Het script koppelt `docker.sock` pas nadat aan de sandboxvereisten is voldaan. Als de sandboxconfiguratie niet kan worden voltooid, stelt het `agents.defaults.sandbox.mode` opnieuw in op `off`. De Codex-codemodus is uitgeschakeld voor beurten waarin de OpenClaw-sandbox actief is (zie [Sandboxing § Docker-backend](/nl/gateway/sandboxing#docker-backend)); koppel de Docker-socket van de host nooit aan agentsandboxcontainers.

  </Accordion>

  <Accordion title="Automatisering / CI (niet-interactief)">
    Schakel de pseudo-TTY-toewijzing van Compose uit met `-T`:

    ```bash
    docker compose run -T --rm openclaw-cli gateway probe
    docker compose run -T --rm openclaw-cli devices list --json
    ```

  </Accordion>

  <Accordion title="Beveiligingsopmerking voor gedeeld netwerk">
    `openclaw-cli` gebruikt `network_mode: "service:openclaw-gateway"`, zodat CLI-opdrachten de Gateway via `127.0.0.1` kunnen bereiken. Behandel dit als een gedeelde vertrouwensgrens. De Compose-configuratie verwijdert `NET_RAW`/`NET_ADMIN` en schakelt `no-new-privileges` in op zowel `openclaw-gateway` als `openclaw-cli`.
  </Accordion>

  <Accordion title="DNS-fouten van Docker Desktop in openclaw-cli">
    Bij sommige Docker Desktop-configuraties mislukken DNS-opzoekingen vanuit de `openclaw-cli`-sidecar op het gedeelde netwerk nadat `NET_RAW` is verwijderd. Dit verschijnt als `EAI_AGAIN` tijdens npm-gebaseerde opdrachten zoals `openclaw plugins install`. Gebruik voor normaal bedrijf het standaard geharde Compose-bestand. De onderstaande override herstelt de standaardmogelijkheden uitsluitend voor de `openclaw-cli`-container — gebruik deze voor de eenmalige opdracht die toegang tot het register nodig heeft, niet als je standaardaanroep:

    ```bash
    printf '%s\n' \
      'services:' \
      '  openclaw-cli:' \
      '    cap_drop: !reset []' \
      > docker-compose.cli-no-dropped-caps.local.yml

    docker compose -f docker-compose.yml -f docker-compose.cli-no-dropped-caps.local.yml run --rm openclaw-cli plugins install <package>
    ```

    Als je al een langlopende `openclaw-cli`-container hebt gemaakt, maak je deze opnieuw met dezelfde override — `docker compose exec`/`docker exec` kunnen de Linux-mogelijkheden van een reeds gemaakte container niet wijzigen.

  </Accordion>

  <Accordion title="Machtigingen en EACCES">
    De image wordt uitgevoerd als `node` (uid 1000). Als je machtigingsfouten ziet voor `/home/node/.openclaw`, zorg er dan voor dat je bind mounts op de host eigendom zijn van uid 1000:

    ```bash
    sudo chown -R 1000:1000 /path/to/openclaw-config /path/to/openclaw-workspace
    ```

    Dezelfde discrepantie kan verschijnen als `blocked plugin candidate: suspicious ownership (... uid=1000, expected uid=0 or root)` gevolgd door `plugin present but blocked` — de proces-uid en de eigenaar van de gekoppelde plug-inmap komen niet overeen. Voer bij voorkeur uit met de standaard-uid 1000 en corrigeer het eigendom van de bind mount. Wijzig het eigendom van `/path/to/openclaw-config/npm` alleen naar `root:root` als je OpenClaw bewust langdurig als root uitvoert.

  </Accordion>

  <Accordion title="Snellere herbouwprocessen">
    Rangschik je Dockerfile zodat afhankelijkheidslagen in de cache worden opgeslagen en `pnpm install` niet opnieuw wordt uitgevoerd, tenzij lockfiles wijzigen:

    ```dockerfile
    FROM node:24-bookworm
    RUN curl -fsSL https://bun.sh/install | bash
    ENV PATH="/root/.bun/bin:${PATH}"
    RUN corepack enable
    WORKDIR /app
    COPY package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc ./
    COPY ui/package.json ./ui/package.json
    COPY scripts ./scripts
    RUN pnpm install --frozen-lockfile
    COPY . .
    RUN pnpm build
    RUN pnpm ui:install
    RUN pnpm ui:build
    ENV NODE_ENV=production
    CMD ["node","dist/index.js"]
    ```

  </Accordion>

  <Accordion title="Containeropties voor ervaren gebruikers">
    De standaardimage stelt beveiliging voorop en wordt zonder rootrechten uitgevoerd als `node`. Voor een container met meer functies:

    1. **`/home/node` permanent opslaan**: `export OPENCLAW_HOME_VOLUME="openclaw_home"`
    2. **Systeemafhankelijkheden in de image opnemen**: `export OPENCLAW_IMAGE_APT_PACKAGES="git curl jq"`
    3. **Python-afhankelijkheden in de image opnemen**: `export OPENCLAW_IMAGE_PIP_PACKAGES="requests==2.32.5 humanize==4.14.0"`
    4. **Playwright Chromium in de image opnemen**: `export OPENCLAW_INSTALL_BROWSER=1`, of gebruik de officiële `-browser`-imagetag
    5. **Of Playwright-browsers in een permanent volume installeren**:
       ```bash
       docker compose run --rm openclaw-cli \
         node /app/node_modules/playwright-core/cli.js install chromium
       ```
    6. **Browserdownloads permanent opslaan**: gebruik `OPENCLAW_HOME_VOLUME` of `OPENCLAW_EXTRA_MOUNTS`. OpenClaw detecteert op Linux automatisch het door Playwright beheerde Chromium van de image.

  </Accordion>

  <Accordion title="OpenAI Codex OAuth (headless Docker)">
    Als je in de wizard OpenAI Codex OAuth kiest, wordt een browser-URL geopend. Kopieer in Docker- of headless-configuraties de volledige omleidings-URL waarop je terechtkomt en plak deze terug in de wizard om de authenticatie te voltooien.
  </Accordion>

  <Accordion title="Metadata van de basisimage">
    De runtime-image gebruikt `node:24-bookworm-slim` en voert `tini` uit als PID 1, zodat zombieprocessen worden opgeruimd en signalen correct worden afgehandeld in langlopende containers. De image publiceert OCI-annotaties voor basisimages, waaronder `org.opencontainers.image.base.name` en `org.opencontainers.image.source`. Dependabot vernieuwt de vastgezette digest van de Node-basisimage; releasebuilds voeren geen afzonderlijke upgrade van de distributielaag uit. Zie [OCI-imageannotaties](https://github.com/opencontainers/image-spec/blob/main/annotations.md).
  </Accordion>
</AccordionGroup>

### Uitvoeren op een VPS?

Zie [Hetzner (Docker-VPS)](/nl/install/hetzner) en [Docker-VM-runtime](/nl/install/docker-vm-runtime) voor implementatiestappen voor een gedeelde VM, waaronder het opnemen van binaire bestanden in de image, permanente opslag en updates.

## Agentsandbox

Wanneer `agents.defaults.sandbox` is ingeschakeld met de Docker-backend, voert de Gateway agenttools (shell, bestanden lezen/schrijven enzovoort) uit in geïsoleerde Docker-containers, terwijl de Gateway zelf op de host blijft — een harde scheiding rond niet-vertrouwde agentsessies of agentsessies met meerdere tenants, zonder de volledige Gateway in een container uit te voeren.

Het sandboxbereik kan per agent (standaard), per sessie of gedeeld zijn; elk bereik krijgt een eigen werkruimte die op `/workspace` wordt gekoppeld. Je kunt ook beleid voor toegestane/geweigerde tools, netwerkisolatie, resourcelimieten en browsercontainers configureren.

Voor de volledige configuratie, images, beveiligingsopmerkingen en profielen met meerdere agents:

- [Sandboxing](/nl/gateway/sandboxing) -- volledige sandboxreferentie
- [OpenShell](/nl/gateway/openshell) -- interactieve shelltoegang tot sandboxcontainers
- [Sandbox en tools voor meerdere agents](/nl/tools/multi-agent-sandbox-tools) -- overrides per agent

### Snel inschakelen

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main", // uit | niet-main | alles
        scope: "agent", // sessie | agent | gedeeld
      },
    },
  },
}
```

Bouw de standaard-sandboximage (vanuit een broncodecheckout):

```bash
scripts/sandbox-setup.sh
```

Zie voor npm-installaties zonder broncodecheckout [Sandboxing § Images en configuratie](/nl/gateway/sandboxing#images-and-setup) voor inline `docker build`-opdrachten.

## Problemen oplossen

<AccordionGroup>
  <Accordion title="Image ontbreekt of sandboxcontainer start niet">
    Bouw de sandboximage met [`scripts/sandbox-setup.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/sandbox-setup.sh) (broncodecheckout) of de inline `docker build`-opdracht uit [Sandboxing § Images en configuratie](/nl/gateway/sandboxing#images-and-setup) (npm-installatie), of stel `agents.defaults.sandbox.docker.image` in op je aangepaste image. Containers worden indien nodig automatisch per sessie gemaakt.
  </Accordion>

  <Accordion title="Machtigingsfouten in de sandbox">
    Stel `docker.user` in op een UID:GID die overeenkomt met het eigendom van je gekoppelde werkruimte, of wijzig het eigendom van de werkruimtemap.
  </Accordion>

  <Accordion title="Aangepaste tools niet gevonden in de sandbox">
    OpenClaw voert opdrachten uit met `sh -lc` (login-shell), die `/etc/profile` inleest en PATH mogelijk opnieuw instelt. Stel `docker.env.PATH` in om je aangepaste toolpaden vooraan toe te voegen, of voeg in je Dockerfile een script toe onder `/etc/profile.d/`.
  </Accordion>

  <Accordion title="Door OOM beëindigd tijdens het bouwen van de image (afsluitcode 137)">
    De VM heeft minimaal 2 GB RAM nodig. Gebruik een grotere machineklasse en probeer het opnieuw.
  </Accordion>

  <Accordion title="Niet geautoriseerd of koppeling vereist in de Control UI">
    Haal een nieuwe dashboardlink op en keur het browserapparaat goed:

    ```bash
    docker compose run --rm openclaw-cli dashboard --no-open
    docker compose run --rm openclaw-cli devices list
    docker compose run --rm openclaw-cli devices approve <requestId>
    ```

    Meer informatie: [Dashboard](/nl/web/dashboard), [Apparaten](/nl/cli/devices).

  </Accordion>

  <Accordion title="Gateway-doel toont ws://172.x.x.x of koppelingsfouten vanuit de Docker-CLI">
    Stel de Gateway-modus en binding opnieuw in:

    ```bash
    docker compose run --rm openclaw-cli config set --batch-json '[{"path":"gateway.mode","value":"local"},{"path":"gateway.bind","value":"lan"}]'
    docker compose run --rm openclaw-cli devices list --url ws://127.0.0.1:18789
    ```

  </Accordion>
</AccordionGroup>

## Gerelateerd

- [Installatieoverzicht](/nl/install) — alle installatiemethoden
- [Podman](/nl/install/podman) — Podman-alternatief voor Docker
- [ClawDock](/nl/install/clawdock) — communityconfiguratie met Docker Compose
- [Bijwerken](/nl/install/updating) — OpenClaw up-to-date houden
- [Configuratie](/nl/gateway/configuration) — Gateway-configuratie na installatie
