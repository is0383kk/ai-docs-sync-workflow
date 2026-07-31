---
read_when:
    - Je implementeert OpenClaw op een cloud-VM met Docker
    - Je hebt de gedeelde flow voor het inbouwen van binaries, persistentie en updates nodig
summary: Gedeelde stappen voor de Docker-VM-runtime voor lang draaiende OpenClaw Gateway-hosts
title: Docker-VM-runtime
x-i18n:
    generated_at: "2026-07-27T06:18:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d1c474b1f826077ac03c7aaa1e334ed2f38d2de2770f32f2cc907846ecc8bb19
    source_path: install/docker-vm-runtime.md
    workflow: 16
---

Gedeelde runtimestappen voor VM-gebaseerde Docker-installaties, zoals GCP, Hetzner en vergelijkbare VPS-providers.

## Vereiste binaire bestanden in de image opnemen

Binaire bestanden installeren in een actieve container is een valkuil: alles wat
tijdens runtime wordt geïnstalleerd, gaat bij een herstart verloren. Neem elk extern binair bestand dat een skill nodig heeft
tijdens het bouwen op in de image.

De onderstaande voorbeelden behandelen slechts drie binaire bestanden, in alfabetische volgorde:

- `gog` (van `gogcli`) voor toegang tot Gmail
- `goplaces` voor Google Places
- `wacli` voor WhatsApp

Dit zijn voorbeelden, geen volledige lijst. Installeer met hetzelfde patroon zoveel binaire bestanden als je
skills nodig hebben. Wanneer je later een skill toevoegt die een nieuw
binair bestand nodig heeft:

1. Werk het Dockerfile bij.
2. Bouw de image opnieuw.
3. Start de containers opnieuw.

**Voorbeeld-Dockerfile**

```dockerfile
FROM node:24-bookworm

RUN apt-get update && apt-get install -y socat && rm -rf /var/lib/apt/lists/*

# Voorbeeld van binair bestand 1: Gmail-CLI (gogcli — wordt geïnstalleerd als `gog`)
# Kopieer de huidige URL van de Linux-asset van https://github.com/steipete/gogcli/releases
RUN curl -L https://github.com/steipete/gogcli/releases/latest/download/gogcli_linux_amd64.tar.gz \
  | tar -xzO gog > /usr/local/bin/gog; \
  chmod +x /usr/local/bin/gog

# Voorbeeld van binair bestand 2: Google Places-CLI
# Kopieer de huidige URL van de Linux-asset van https://github.com/steipete/goplaces/releases
RUN curl -L https://github.com/steipete/goplaces/releases/latest/download/goplaces_linux_amd64.tar.gz \
  | tar -xzO goplaces > /usr/local/bin/goplaces; \
  chmod +x /usr/local/bin/goplaces

# Voorbeeld van binair bestand 3: WhatsApp-CLI
# Kopieer de huidige URL van de Linux-asset van https://github.com/steipete/wacli/releases
RUN curl -L https://github.com/steipete/wacli/releases/latest/download/wacli-linux-amd64.tar.gz \
  | tar -xzO wacli > /usr/local/bin/wacli; \
  chmod +x /usr/local/bin/wacli

# Voeg hieronder met hetzelfde patroon meer binaire bestanden toe

WORKDIR /app
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc ./
COPY ui/package.json ./ui/package.json
COPY scripts ./scripts

RUN corepack enable
RUN pnpm install --frozen-lockfile

COPY . .
RUN pnpm build
RUN pnpm ui:install
RUN pnpm ui:build

ENV NODE_ENV=production

CMD ["node","dist/index.js"]
```

<Note>
De bovenstaande URL's zijn voorbeelden. Kies voor ARM-gebaseerde VM's de `arm64`-assets. Zet voor reproduceerbare builds de URL's van releases met een specifieke versie vast.
</Note>

## Bouwen en starten

```bash
docker compose build
docker compose up -d openclaw-gateway
```

Als de build tijdens `pnpm install --frozen-lockfile` mislukt met `Killed` of afsluitcode 137, heeft de VM onvoldoende geheugen. Gebruik een grotere machineklasse voordat je het opnieuw probeert.

Controleer de binaire bestanden:

```bash
docker compose exec openclaw-gateway which gog
docker compose exec openclaw-gateway which goplaces
docker compose exec openclaw-gateway which wacli
```

Verwachte uitvoer:

```text
/usr/local/bin/gog
/usr/local/bin/goplaces
/usr/local/bin/wacli
```

Controleer of de Gateway actief is:

```bash
docker compose logs -f openclaw-gateway
curl -fsS http://127.0.0.1:18789/healthz
```

Als `/healthz` een 200-respons retourneert, bevestigt dit dat het Gateway-proces luistert en gezond is; de ingebouwde `HEALTHCHECK` van de image controleert hetzelfde eindpunt.

## Wat waar persistent blijft

OpenClaw draait in Docker, maar Docker is niet de bron van waarheid. Alle langdurige status moet herstarts, nieuwe builds en reboots overleven.

| Component              | Locatie                                                | Persistentiemechanisme    | Opmerkingen                                                                                                                  |
| ---------------------- | ------------------------------------------------------ | ------------------------ | ---------------------------------------------------------------------------------------------------------------------------- |
| Gateway-configuratie   | `/home/node/.openclaw/`                                     | Volumekoppeling host     | Bevat `openclaw.json`                                                                                                     |
| Kanaal-/providerreferenties | `/home/node/.openclaw/credentials/`                                | Volumekoppeling host     | Referentiemateriaal voor kanalen en providers                                                                                 |
| Modelauthenticatieprofielen | `/home/node/.openclaw/agents/`                                 | Volumekoppeling host     | `agents/<agentId>/agent/auth-profiles.json` (OAuth, API-sleutels)                                                                                      |
| Verouderd OAuth-sleutelbestand | `/home/node/.config/openclaw/`                              | Volumekoppeling host     | Alleen-lezencompatibiliteit voor OAuth-sidecars van vóór de migratie; `openclaw doctor --fix` migreert deze naar `auth-profiles.json` |
| Skill-configuraties    | `/home/node/.openclaw/skills/`                                     | Volumekoppeling host     | Status op skillniveau                                                                                                        |
| Agentwerkruimte        | `/home/node/.openclaw/workspace/`                                     | Volumekoppeling host     | Code en agentartefacten                                                                                                      |
| WhatsApp-sessie        | `/home/node/.openclaw/`                                     | Volumekoppeling host     | Behoudt de QR-aanmelding                                                                                                     |
| Gmail-sleutelring      | `/home/node/.openclaw/`                                     | Hostvolume + wachtwoord  | Vereist `GOG_KEYRING_PASSWORD`                                                                                                   |
| Plugin-pakketten       | `/home/node/.openclaw/npm`, `/home/node/.openclaw/git`                | Volumekoppeling host     | Hoofdmappen van downloadbare pluginpakketten                                                                                  |
| Externe binaire bestanden | `/usr/local/bin/`                                  | Docker-image             | Moeten tijdens het bouwen worden opgenomen                                                                                    |
| Node-runtime           | Containerbestandssysteem                               | Docker-image             | Wordt bij elke imagebuild opnieuw opgebouwd                                                                                   |
| OS-pakketten           | Containerbestandssysteem                               | Docker-image             | Niet tijdens runtime installeren                                                                                              |
| Docker-container       | Tijdelijk                                              | Herstartbaar             | Kan veilig worden vernietigd                                                                                                  |

## Updates

OpenClaw op de VM bijwerken:

```bash
git pull
docker compose build
docker compose up -d
```

## Gerelateerd

- [Docker](/nl/install/docker)
- [Podman](/nl/install/podman)
- [ClawDock](/nl/install/clawdock)
