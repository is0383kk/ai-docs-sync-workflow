---
read_when:
    - Kiezen tussen auto, ask, allowlist, full of deny voor opdrachtmachtigingen
    - Door Codex Guardian beoordeelde goedkeuringen configureren via tools.exec.mode
    - OpenClaw-uitvoeringsgoedkeuringen vergelijken met ACPX-harnasrechten
summary: Toestemmingsmodi voor uitvoering op de host, Codex Guardian-goedkeuringen en ACPX-harnesssessies
title: Toestemmingsmodi
x-i18n:
    generated_at: "2026-07-27T06:36:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f580e66508c1f69e868ed26a62d88a675f86a4d1ca738650dc5af82e967f3ac3
    source_path: tools/permission-modes.md
    workflow: 16
---

Toestemmingsmodi bepalen hoeveel bevoegdheid een agent heeft voordat deze hostopdrachten uitvoert, bestanden schrijft of een backend-harnas om extra toegang vraagt.

<Note>
  De toestemmingsmodus staat los van `tools.exec.host=auto`. `tools.exec.host`
  bepaalt waar een opdracht wordt uitgevoerd. `tools.exec.mode` bepaalt hoe hostuitvoering
  wordt goedgekeurd.
</Note>

## Aanbevolen standaardinstelling

Gebruik `auto` voor programmeeragents die nuttige hosttoegang nodig hebben zonder dat elke niet-overeenkomende opdracht een menselijke prompt veroorzaakt:

```bash
openclaw config set tools.exec.mode auto
openclaw approvals get
openclaw gateway restart
```

Controleer vervolgens het effectieve beleid:

```bash
openclaw exec-policy show
```

## OpenClaw-modi voor hostuitvoering

`tools.exec.mode` is het genormaliseerde beleidsoppervlak voor `exec` op de host. Elke modus wordt omgezet in een onderliggend paar `security` (strengheid van de toelatingslijst) en `ask` (vragen bij geen overeenkomst):

| Modus        | security / ask          | Gedrag                                                                                      | Gebruiken wanneer                                              |
| ----------- | ----------------------- | --------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| `deny`      | `deny` / `off`          | Hostuitvoering volledig blokkeren.                                                                     | Hostopdrachten zijn niet toegestaan.                         |
| `allowlist` | `allowlist` / `off`     | Alleen opdrachten op de toelatingslijst uitvoeren; niet-overeenkomende opdrachten stilzwijgend weigeren.                                          | Je een bekende, veilige opdrachtenset hebt.                    |
| `ask`       | `allowlist` / `on-miss` | Overeenkomsten met de toelatingslijst uitvoeren; bij geen overeenkomst een mens om goedkeuring vragen.                                                 | Een mens elke nieuwe opdracht moet beoordelen.              |
| `auto`      | `allowlist` / `on-miss` | Overeenkomsten met de toelatingslijst uitvoeren; niet-overeenkomende opdrachten automatisch laten beoordelen voordat wordt teruggevallen op menselijke goedkeuring. | Programmeersessies praktische, beveiligde toegang nodig hebben.        |
| `full`      | `full` / `off`          | Hostuitvoering zonder prompts uitvoeren.                                                                | Deze vertrouwde host/sessie goedkeuringspoorten moet overslaan. |

`ask` en `auto` gebruiken dezelfde instellingen voor de toelatingslijst en vragen; `auto` schakelt daarnaast de ingebouwde automatische beoordelaar in, die niet-overeenkomende opdrachten zelf beoordeelt en deze alleen doorstuurt naar de geconfigureerde route voor menselijke goedkeuring wanneer veilige goedkeuring niet mogelijk is.

Zie [Goedkeuringen voor uitvoering](/nl/tools/exec-approvals) voor het volledige beleid voor hostuitvoering, het lokale goedkeuringsbestand, het schema van de toelatingslijst, veilige binaire bestanden en het doorstuurgedrag.

## Toewijzing van Codex Guardian

Voor systeemeigen Codex-appserversessies stuurt `tools.exec.mode: "auto"` Codex naar door Guardian beoordeelde goedkeuringen wanneer de lokale Codex-vereisten dit toestaan. Typische resulterende waarden:

| Codex-veld         | Typische waarde     |
| ------------------- | ----------------- |
| `approvalPolicy`    | `on-request`      |
| `approvalsReviewer` | `auto_review`     |
| `sandbox`           | `workspace-write` |

De modus `auto` dwingt dit beleid af boven eventueel geconfigureerde Codex-overschrijvingen voor sandboxing en goedkeuringen, zodat verouderde onveilige combinaties zoals `approvalPolicy: "never"` met `sandbox: "danger-full-access"` niet behouden blijven. `tools.exec.mode: "deny"` en `"allowlist"` blokkeren lokale uitvoering door de Codex-appserver volledig. Gebruik `tools.exec.mode: "full"` alleen wanneer je bewust geen goedkeuringen wilt gebruiken.

Zie [Codex-harnas](/nl/plugins/codex-harness) voor de configuratie van de appserver, de authenticatievolgorde en details over de systeemeigen Codex-runtime.

## Toestemmingen voor het ACPX-harnas

ACPX-sessies zijn niet-interactief en kunnen daarom niet op een TTY-toestemmingsprompt klikken. ACPX gebruikt afzonderlijke instellingen op harnasniveau onder `plugins.entries.acpx.config`:

| Instelling                     | Waarden          | Betekenis                                     |
| --------------------------- | --------------- | ------------------------------------------- |
| `permissionMode`            | `approve-reads` | Alleen leesbewerkingen automatisch goedkeuren.                    |
| `permissionMode`            | `approve-all`   | Schrijfbewerkingen en shellopdrachten automatisch goedkeuren.     |
| `permissionMode`            | `deny-all`      | Alle toestemmingsprompts weigeren.                |
| `nonInteractivePermissions` | `fail`          | Afbreken wanneer een prompt vereist zou zijn.      |
| `nonInteractivePermissions` | `deny`          | De prompt weigeren en waar mogelijk doorgaan. |

Stel ACPX-toestemmingen afzonderlijk in van OpenClaw-goedkeuringen voor uitvoering:

```bash
openclaw config set plugins.entries.acpx.config.permissionMode approve-all
openclaw config set plugins.entries.acpx.config.nonInteractivePermissions fail
openclaw gateway restart
```

Gebruik `approve-all` als het ACPX-noodalternatief voor een harnassessie zonder prompts. Zie [Configuratie van ACP-agents](/nl/tools/acp-agents-setup#permission-configuration) voor configuratiedetails en foutmodi.

## Een modus kiezen

| Doel                                          | Configureren                                                   |
| --------------------------------------------- | ----------------------------------------------------------- |
| Hostopdrachten volledig blokkeren                | `tools.exec.mode: "deny"`                                   |
| Alleen bekende, veilige opdrachten laten uitvoeren              | `tools.exec.mode: "allowlist"`                              |
| Voor elke nieuwe opdrachtvorm een mens om goedkeuring vragen       | `tools.exec.mode: "ask"`                                    |
| Automatische beoordeling door Codex/OpenClaw gebruiken vóór menselijke beoordeling  | `tools.exec.mode: "auto"`                                   |
| Goedkeuringen voor hostuitvoering volledig overslaan             | `tools.exec.mode: "full"` plus een overeenkomend hostgoedkeuringsbestand |
| Niet-interactieve ACPX-sessies laten schrijven/uitvoeren | `plugins.entries.acpx.config.permissionMode: "approve-all"` |

Als een opdracht na het wijzigen van de modus nog steeds een prompt veroorzaakt of mislukt, controleer je beide lagen:

```bash
openclaw approvals get
openclaw exec-policy show
```

Voor hostuitvoering geldt het strengste resultaat van de OpenClaw-configuratie en het hostlokale goedkeuringsbestand. Toestemmingen voor het ACPX-harnas versoepelen de goedkeuringen voor hostuitvoering niet, en goedkeuringen voor hostuitvoering versoepelen de prompts van het ACPX-harnas niet.

## Gerelateerd

- [Goedkeuringen voor uitvoering](/nl/tools/exec-approvals)
- [Goedkeuringen voor uitvoering - geavanceerd](/nl/tools/exec-approvals-advanced)
- [Codex-harnas](/nl/plugins/codex-harness)
- [Configuratie van ACP-agents](/nl/tools/acp-agents-setup#permission-configuration)
