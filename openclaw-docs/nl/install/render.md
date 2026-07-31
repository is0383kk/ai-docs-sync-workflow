---
read_when:
    - OpenClaw implementeren op Render
    - Je wilt een declaratieve cloudimplementatie met Render Blueprints
summary: Implementeer OpenClaw op Render met Infrastructure-as-Code
title: Render
x-i18n:
    generated_at: "2026-07-27T06:19:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a5fbb3c6df04e186df958a62a6130da4e3e485acfeecc7e85fee0d5b69a0438f
    source_path: install/render.mdx
    workflow: 16
---

Implementeer OpenClaw op [Render](https://render.com) met behulp van de `render.yaml`-Blueprint van de repository. Deze declareert de service, schijf en omgevingsvariabelen in één bestand.

## Vereisten

- Een [Render-account](https://render.com) (gratis abonnement beschikbaar)
- Een API-sleutel van je gewenste [modelprovider](/nl/providers)

## Implementeren

[Implementeren op Render](https://render.com/deploy?repo=https://github.com/openclaw/openclaw)

Hiermee wordt een Render-service gemaakt op basis van `render.yaml`, de Docker-image gebouwd en deze geïmplementeerd. De URL van je service volgt het patroon `https://<service-name>.onrender.com`.

## De Blueprint

```yaml
services:
  - type: web
    name: openclaw
    runtime: docker
    plan: starter
    healthCheckPath: /health
    envVars:
      - key: OPENCLAW_GATEWAY_PORT
        value: "8080"
      - key: OPENCLAW_STATE_DIR
        value: /data/.openclaw
      - key: OPENCLAW_WORKSPACE_DIR
        value: /data/workspace
      - key: OPENCLAW_GATEWAY_TOKEN
        generateValue: true # genereert automatisch een veilig token
    disk:
      name: openclaw-data
      mountPath: /data
      sizeGB: 1
```

| Functie               | Doel                                                    |
| --------------------- | ---------------------------------------------------------- |
| `runtime: docker`     | Bouwt vanuit het Dockerfile van de repository                          |
| `healthCheckPath`     | Render bewaakt `/health` en start ongezonde instanties opnieuw |
| `generateValue: true` | Genereert automatisch een cryptografisch veilige waarde            |
| `disk`                | Permanente opslag die behouden blijft na nieuwe implementaties                 |

## Een abonnement kiezen

| Abonnement      | Uitschakelen         | Schijf          | Meest geschikt voor                      |
| --------- | ----------------- | ------------- | ----------------------------- |
| Free      | Na 15 min. inactiviteit | Niet beschikbaar | Tests, demo's                |
| Starter   | Nooit             | 1GB+          | Persoonlijk gebruik, kleine teams     |
| Standard+ | Nooit             | 1GB+          | Productie, meerdere kanalen |

De Blueprint gebruikt standaard `starter`. Als je het gratis abonnement wilt gebruiken, wijzig je `plan: free` in `render.yaml` van je fork. Houd er rekening mee dat zonder permanente schijf de status van OpenClaw bij elke implementatie wordt gereset.

## Na de implementatie

### De Control UI openen

Het webdashboard is beschikbaar op `https://<your-service>.onrender.com/`. Maak verbinding met het gedeelde geheim: de automatisch gegenereerde `OPENCLAW_GATEWAY_TOKEN` (te vinden onder **Dashboard → your service → Environment**), of met je wachtwoord als je bent overgeschakeld op wachtwoordauthenticatie.

### Logboeken

**Dashboard → your service → Logs** toont buildlogboeken (aanmaken van de Docker-image), implementatielogboeken (opstarten van de service) en runtimelogboeken (uitvoer van de toepassing).

### Shell-toegang

**Dashboard → your service → Shell** opent een shellsessie. De permanente schijf is gekoppeld aan `/data`.

### Omgevingsvariabelen

Bewerk variabelen onder **Dashboard → your service → Environment**. Wijzigingen activeren automatisch een nieuwe implementatie.

### Automatisch implementeren

Render implementeert automatisch opnieuw wanneer de branch van de gekoppelde repository een nieuwe commit krijgt. Als je rechtstreeks vanuit `openclaw/openclaw` hebt geïmplementeerd in plaats van vanuit je eigen fork, heb je geen pushtoegang om dit te activeren. Werk de service daarom bij door handmatig een Blueprint-synchronisatie vanuit het Dashboard uit te voeren, of koppel de service aan je eigen fork.

## Aangepast domein

1. **Dashboard → your service → Settings → Custom Domains**
2. Voeg je domein toe
3. Configureer DNS volgens de instructies (CNAME naar `*.onrender.com`)
4. Render verstrekt automatisch een TLS-certificaat

## Schalen

- **Verticaal**: wijzig het abonnement voor meer CPU/RAM. Dit is meestal voldoende voor OpenClaw.
- **Horizontaal**: verhoog het aantal instanties (Standard-abonnement en hoger). Hiervoor zijn sticky sessions of extern statusbeheer vereist, omdat OpenClaw de runtimestatus op de lokale schijf bewaart.

## Back-ups en migratie

Exporteer op elk gewenst moment de status, configuratie, authenticatieprofielen en werkruimte vanuit de shell van het Render Dashboard:

```bash
openclaw backup create
```

Hiermee wordt een overdraagbaar back-uparchief gemaakt. Zie [Back-up](/nl/cli/backup).

## Probleemoplossing

### Service start niet

Controleer de implementatielogboeken in het Render Dashboard. Veelvoorkomende problemen:

- Ontbrekende `OPENCLAW_GATEWAY_TOKEN` — controleer of deze is ingesteld onder **Dashboard → Environment**
- Poort komt niet overeen — zorg voor `OPENCLAW_GATEWAY_PORT=8080`, zodat de Gateway wordt gekoppeld aan de poort die Render verwacht

### Langzame koude starts (gratis abonnement)

Services met een gratis abonnement worden na 15 minuten inactiviteit uitgeschakeld. Het duurt enkele seconden voordat het eerste verzoek na uitschakeling wordt verwerkt terwijl de container wordt gestart. Upgrade naar Starter om de service altijd actief te houden.

### Gegevensverlies na nieuwe implementatie

Dit gebeurt bij het gratis abonnement (geen permanente schijf). Upgrade naar een betaald abonnement of exporteer regelmatig een back-up met `openclaw backup create` vanuit de Render-shell.

### Mislukte statuscontroles

Als builds slagen maar implementaties mislukken, duurt het mogelijk te lang voordat de service start of is `/health` mogelijk niet bereikbaar. Controleer:

- Buildlogboeken op fouten
- Of de container lokaal wordt uitgevoerd met `docker build && docker run`

## Volgende stappen

- Stel berichtenkanalen in: [Kanalen](/nl/channels)
- Configureer de Gateway: [Gateway-configuratie](/nl/gateway/configuration)
- Houd OpenClaw up-to-date: [Bijwerken](/nl/install/updating)
