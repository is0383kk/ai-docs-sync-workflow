---
read_when:
    - Node is verbonden, maar de camera-/canvas-/scherm-/exec-tools werken niet
    - Je hebt het mentale model voor Node-koppeling versus goedkeuringen nodig
summary: Problemen oplossen met Node-koppeling, vereisten voor de voorgrond, machtigingen en toolfouten
title: Probleemoplossing voor Node
x-i18n:
    generated_at: "2026-07-27T05:19:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4a7ee9e48985805e91cd5acfa1b9f6b676b7e67236ce29fe91e2c8d03002e5c4
    source_path: nodes/troubleshooting.md
    workflow: 16
---

Gebruik deze pagina wanneer een Node zichtbaar is in de status, maar Node-tools niet werken.

## Opdrachtvolgorde

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

Voer daarna Node-specifieke controles uit:

```bash
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
openclaw approvals get --node <idOrNameOrIp>
```

Signalen dat alles goed werkt:

- De Node is verbonden en gekoppeld voor de rol `node`.
- `nodes describe` bevat de aangeroepen mogelijkheid.
- Uitvoeringsgoedkeuringen tonen de verwachte modus/toelatingslijst.

## Vereisten voor de voorgrond

`canvas.*`, `camera.*` en `screen.*` werken op iOS-/Android-Nodes alleen op de voorgrond.

Snelle controle en oplossing:

```bash
openclaw nodes describe --node <idOrNameOrIp>
openclaw nodes canvas snapshot --node <idOrNameOrIp>
openclaw logs --follow
```

Als je `NODE_BACKGROUND_UNAVAILABLE` ziet, breng je de Node-app naar de voorgrond en probeer je het opnieuw.

## Machtigingenmatrix

| Mogelijkheid                   | iOS                                     | Android                                      | macOS-Node-app                   | Gebruikelijke foutcode                          |
| ---------------------------- | --------------------------------------- | -------------------------------------------- | -------------------------------- | --------------------------------------------- |
| `camera.snap`, `camera.clip` | Camera (+ microfoon voor clipaudio)           | Camera (+ microfoon voor clipaudio)                | Camera (+ microfoon voor clipaudio)    | `*_PERMISSION_REQUIRED`                       |
| `screen.record`              | Screen Recording (+ microfoon optioneel)       | Prompt voor schermopname (+ microfoon optioneel)       | Screen Recording                 | `*_PERMISSION_REQUIRED`                       |
| `computer.act`               | n.v.t.                                     | n.v.t.                                          | Accessibility + Screen Recording | `COMPUTER_DISABLED`, `ACCESSIBILITY_REQUIRED` |
| `location.get`               | While Using of Always (afhankelijk van de modus) | Voorgrond-/achtergrondlocatie op basis van de modus | Locatiemachtiging              | `LOCATION_PERMISSION_REQUIRED`                |
| `system.run`                 | n.v.t. (pad van Node-host)                    | n.v.t. (pad van Node-host)                         | Uitvoeringsgoedkeuringen vereist          | `SYSTEM_RUN_DENIED`                           |

## Koppeling versus goedkeuringen

Drie afzonderlijke poorten bepalen of een Node-opdracht slaagt:

1. **Apparaatkoppeling**: kan deze Node verbinding maken met de Gateway?
2. **Beleid voor Gateway-Node-opdrachten**: is de RPC-opdracht-ID toegestaan door `gateway.nodes.commands.allow` / `gateway.nodes.commands.deny` en de platformstandaarden?
3. **Uitvoeringsgoedkeuringen**: kan deze Node lokaal een specifieke shellopdracht uitvoeren?

Node-koppeling is een poort voor identiteit en vertrouwen, geen goedkeuringsoppervlak per opdracht. Voor `system.run` bevindt het beleid per Node zich in het bestand met uitvoeringsgoedkeuringen van die Node (`openclaw approvals get --node ...`), niet in de koppelingsregistratie van de Gateway.

Snelle controles:

```bash
openclaw devices list
openclaw nodes status
openclaw approvals get --node <idOrNameOrIp>
openclaw approvals allowlist add --node <idOrNameOrIp> "/usr/bin/uname"
```

- Koppeling ontbreekt: keur eerst het Node-apparaat goed.
- `nodes describe` mist een opdracht: controleer het beleid voor Gateway-Node-opdrachten en of de Node die opdracht bij het verbinden daadwerkelijk heeft gedeclareerd.
- Koppeling is in orde, maar `system.run` mislukt: herstel de uitvoeringsgoedkeuringen/toelatingslijst op die Node.

Voor door goedkeuring ondersteunde uitvoeringen van `host=node` koppelt de Gateway de uitvoering ook aan de voorbereide canonieke `systemRunPlan`. Als een latere aanroeper de opdracht, werkmap of sessiemetadata wijzigt voordat de goedgekeurde uitvoering wordt doorgestuurd, weigert de Gateway de uitvoering wegens een niet-overeenkomende goedkeuring in plaats van de bewerkte payload te vertrouwen.

## Veelvoorkomende Node-foutcodes

| Code                                   | Betekenis                                                                                                                                                                                 |
| -------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `NODE_BACKGROUND_UNAVAILABLE`          | De app bevindt zich op de achtergrond; breng deze naar de voorgrond.                                                                                                                                        |
| `CAMERA_DISABLED`                      | De cameraschakelaar is uitgeschakeld in de Node-instellingen.                                                                                                                                                |
| `*_PERMISSION_REQUIRED`                | OS-machtiging ontbreekt of is geweigerd.                                                                                                                                                           |
| `LOCATION_DISABLED`                    | De locatiemodus is uitgeschakeld.                                                                                                                                                                   |
| `LOCATION_PERMISSION_REQUIRED`         | De aangevraagde locatiemodus is niet toegestaan.                                                                                                                                                    |
| `LOCATION_BACKGROUND_UNAVAILABLE`      | De app bevindt zich op de achtergrond, maar alleen de machtiging While Using is verleend.                                                                                                                             |
| `COMPUTER_DISABLED`                    | Schakel **Allow Computer Control** in de macOS-app in en keur daarna de bijgewerkte koppeling goed.                                                                                                    |
| `ACCESSIBILITY_REQUIRED`               | Verleen Accessibility aan de huidige OpenClaw-appbundel in macOS System Settings.                                                                                                        |
| `SYSTEM_RUN_DENIED: approval required` | Het uitvoeringsverzoek vereist expliciete goedkeuring.                                                                                                                                                   |
| `SYSTEM_RUN_DENIED: allowlist miss`    | De opdracht wordt geblokkeerd door de toelatingslijstmodus. Op Windows-Node-hosts worden shellwrappervormen zoals `cmd.exe /c ...` in de toelatingslijstmodus als ontbrekend in de toelatingslijst behandeld, tenzij ze via de vraagflow worden goedgekeurd. |

## Snelle herstellus

```bash
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
openclaw approvals get --node <idOrNameOrIp>
openclaw logs --follow
```

Als het probleem aanhoudt:

- Keur de apparaatkoppeling opnieuw goed.
- Open de Node-app opnieuw (op de voorgrond).
- Verleen de OS-machtigingen opnieuw.
- Maak het beleid voor uitvoeringsgoedkeuringen opnieuw of pas het aan.

Controleer voor computerbesturing ook of een agent met visiemogelijkheden de tool `computer` beschikbaar stelt, `screen.snapshot` slaagt met de machtiging Screen Recording en `/phone status` de bedoelde tijdelijke of permanente Gateway-autorisatie toont. Een vermelding van `gateway.nodes.commands.deny` heeft altijd voorrang op `gateway.nodes.commands.allow`.

## Gerelateerd

- [Overzicht van Nodes](/nl/nodes)
- [Camera-Nodes](/nl/nodes/camera)
- [Locatieopdracht](/nl/nodes/location-command)
- [Computergebruik](/nl/nodes/computer-use)
- [Uitvoeringsgoedkeuringen](/nl/tools/exec-approvals)
- [Gateway-koppeling](/nl/gateway/pairing)
- [Problemen met de Gateway oplossen](/nl/gateway/troubleshooting)
- [Problemen met kanalen oplossen](/nl/channels/troubleshooting)
