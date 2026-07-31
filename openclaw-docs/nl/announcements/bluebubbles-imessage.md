---
read_when:
    - Je gebruikte het oude BlueBubbles-kanaal en moet overstappen op iMessage
    - Je kiest de ondersteunde OpenClaw-configuratie voor iMessage
    - Je hebt een korte uitleg nodig over de verwijdering van BlueBubbles
summary: BlueBubbles-ondersteuning is verwijderd uit OpenClaw. Gebruik de meegeleverde iMessage-plugin met imsg voor nieuwe en gemigreerde iMessage-configuraties.
title: Verwijdering van BlueBubbles en het imsg-pad voor iMessage
x-i18n:
    generated_at: "2026-07-27T05:00:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7dec7d3f27e0df6431494d864b0c7ae7457574797e199f9a2cb6931d28feacd0
    source_path: announcements/bluebubbles-imessage.md
    workflow: 16
---

# Verwijdering van BlueBubbles en het imsg-pad voor iMessage

OpenClaw levert het BlueBubbles-kanaal niet langer mee. iMessage-ondersteuning verloopt via de meegeleverde `imessage`-plugin: de Gateway start [`imsg`](https://github.com/steipete/imsg) als een onderliggend proces, lokaal of via een SSH-wrapper, en communiceert via JSON-RPC over stdin/stdout. Geen server, geen Webhook, geen poort.

Als je configuratie nog `channels.bluebubbles` bevat, migreer dit dan naar `channels.imessage`. De oude documentatie-URL `/channels/bluebubbles` verwijst door naar [Overstappen vanaf BlueBubbles](/nl/channels/imessage-from-bluebubbles), met de volledige tabel voor configuratievertaling en de checklist voor de omschakeling.

## Wat is gewijzigd

- Het ondersteunde iMessage-pad heeft geen BlueBubbles-HTTP-server, Webhook-route, REST-wachtwoord of runtime voor de BlueBubbles-plugin.
- OpenClaw leest en volgt Berichten via `imsg` op de Mac waarop bij Messages.app is ingelogd.
- Voor eenvoudig verzenden, ontvangen, geschiedenis en media worden de gebruikelijke `imsg`-interfaces en macOS-machtigingen gebruikt.
- Voor geavanceerde acties (antwoorden in threads, tapbacks, bewerken, verzenden ongedaan maken, effecten, leesbevestigingen, typindicatoren en groepsbeheer) is de privé-API-bridge vereist: voer `imsg launch` uit, waarvoor SIP uitgeschakeld moet zijn.
- Gateways op Linux en Windows kunnen iMessage nog steeds gebruiken door `channels.imessage.cliPath` naar een SSH-wrapper te laten verwijzen die `imsg` uitvoert op de Mac waarop is ingelogd.

## Wat je moet doen

1. Installeer en verifieer `imsg` op de Mac met Berichten:

   ```bash
   brew install steipete/tap/imsg
   imsg --version
   imsg chats --limit 3
   imsg rpc --help
   ```

2. Verleen volledige schijftoegang en automatiseringsmachtigingen aan de procescontext waarin `imsg` en OpenClaw worden uitgevoerd.

3. Vertaal de oude configuratie:

   ```json5
   {
     channels: {
       imessage: {
         enabled: true,
         cliPath: "/opt/homebrew/bin/imsg",
         dmPolicy: "pairing",
         allowFrom: ["+15555550123"],
         groupPolicy: "allowlist",
         groupAllowFrom: ["+15555550123"],
         groups: {
           "*": { requireMention: true },
         },
         includeAttachments: true,
       },
     },
   }
   ```

4. Start de Gateway opnieuw en verifieer:

   ```bash
   openclaw channels status --probe
   ```

5. Test privéberichten, groepen, bijlagen en alle privé-API-acties waarvan je afhankelijk bent voordat je de oude BlueBubbles-server verwijdert.

## Migratieopmerkingen

- `channels.bluebubbles.serverUrl` en `channels.bluebubbles.password` hebben geen iMessage-equivalent; er is geen server om te bereiken of om je bij te authenticeren.
- `allowFrom`, `groupAllowFrom`, `groups`, `includeAttachments`, `attachmentRoots`, `mediaMaxMb`, `textChunkLimit` en `actions.*` behouden hun betekenis onder `channels.imessage`.
- `channels.imessage.includeAttachments` is standaard nog steeds uitgeschakeld. Stel dit expliciet in als je verwacht dat inkomende foto's, spraakmemo's, video's of bestanden de agent bereiken.
- Kopieer bij `groupPolicy: "allowlist"` het oude `groups`-blok, inclusief eventuele jokertekenvermelding voor `"*"`. Toegestane groepsafzenders en het groepsregister zijn afzonderlijke controles; een `groups`-blok met vermeldingen maar zonder overeenkomende `chat_id` (of zonder `"*"`) negeert het bericht tijdens runtime, en een leeg `groups`-blok registreert een waarschuwing bij het opstarten, hoewel berichten nog steeds worden doorgelaten door de afzenderfiltering.
- ACP-koppelingen met `match.channel: "bluebubbles"` moeten worden gewijzigd naar `"imessage"`.
- Oude BlueBubbles-sessiesleutels worden geen iMessage-sessiesleutels. Goedkeuringen voor koppeling zijn gebaseerd op afzenderhandgrepen, waardoor gekopieerde `allowFrom`-vermeldingen blijven werken, maar gespreksgeschiedenis onder BlueBubbles-sessiesleutels wordt niet overgenomen.

## Zie ook

- [Overstappen vanaf BlueBubbles](/nl/channels/imessage-from-bluebubbles)
- [iMessage](/nl/channels/imessage)
- [Configuratiereferentie - iMessage](/nl/gateway/config-channels#imessage)
