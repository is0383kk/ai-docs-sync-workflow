---
read_when:
    - IPC-contracten of IPC van de menubalkapp bewerken
summary: macOS-IPC-architectuur voor de OpenClaw-app, Gateway-Node-transport en PeekabooBridge
title: macOS-IPC
x-i18n:
    generated_at: "2026-07-27T05:59:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 39e11af2bb9348d1c1f6e4fe6be95e825d23d5c1aa66e32dae713a89afb12b4f
    source_path: platforms/mac/xpc.md
    workflow: 16
---

# OpenClaw macOS IPC-architectuur

Een lokale Unix-socket verbindt de Node-hostservice met de macOS-app voor uitvoeringsgoedkeuringen en `system.run`. Er bestaat een `openclaw-mac`-debug-CLI (`apps/macos/Sources/OpenClawMacCLI`) voor detectie- en verbindingscontroles; agentacties verlopen nog steeds via de Gateway-WebSocket en `node.invoke`. Het door Node ondersteunde `computer.act`-pad voert ingebouwde Peekaboo-automatisering in hetzelfde proces uit; zelfstandige Peekaboo-clients gebruiken PeekabooBridge.

## Doelen

- Eén GUI-appinstantie die al het TCC-gerelateerde werk beheert (meldingen, schermopname, microfoon, spraak, AppleScript).
- Een beperkt automatiseringsoppervlak: Gateway + Node-opdrachten, `computer.act` in hetzelfde proces, plus PeekabooBridge voor zelfstandige UI-automatiseringsclients.
- Voorspelbare machtigingen: altijd dezelfde ondertekende bundel-ID, gestart door launchd, zodat TCC-toekenningen behouden blijven.

## Werking

### Gateway- en Node-transport

- De app voert de Gateway uit (lokale modus) en maakt er als Node verbinding mee.
- Agentacties worden uitgevoerd via `node.invoke` (bijvoorbeeld `system.run`, `system.notify`, `canvas.*`).
- Node-opdrachten omvatten `canvas.*`, `camera.snap`, `camera.clip`, `screen.snapshot`, `screen.record`, `computer.act`, `system.run` en `system.notify`.
- De Node rapporteert een `permissions`-toewijzing, zodat agenten kunnen zien of toegang tot het scherm, de camera, de microfoon, spraak, automatisering of toegankelijkheidsfuncties beschikbaar is.

### Node-service + app-IPC

- Een headless Node-hostservice maakt verbinding met de Gateway-WebSocket.
- `system.run`-verzoeken worden via een lokale Unix-socket (`ExecApprovalsSocket.swift`) doorgestuurd naar de macOS-app.
- De app voert de opdracht uit in de UI-context, vraagt zo nodig om bevestiging en retourneert de uitvoer.

Diagram (SCI):

```text
Agent -> Gateway -> Node-service (WS)
                      |  IPC (UDS + token + HMAC + TTL)
                      v
                  Mac-app (UI + TCC + system.run)
```

### PeekabooBridge (UI-automatisering)

- De ingebouwde agenttool `computer` gebruikt deze socket **niet**. Een gekoppelde macOS-Node handelt `computer.act` af in het app-proces met ingebouwde Peekaboo-services.
- UI-automatisering gebruikt een afzonderlijke UNIX-socket (`~/Library/Application Support/OpenClaw/<socket>`) en het JSON-protocol van PeekabooBridge.
- Voorkeursvolgorde voor hosts (clientzijde): Peekaboo.app -> Claude.app -> OpenClaw.app -> lokale uitvoering.
- Beveiliging: bridge-hosts vereisen een TeamID op de toelatingslijst (de gebundelde `PeekabooBridgeHostCoordinator` staat een vast team plus het eigen ondertekeningsteam van de app toe); een alleen voor DEBUG beschikbare uitweg voor dezelfde UID wordt afgeschermd door `PEEKABOO_ALLOW_UNSIGNED_SOCKET_CLIENTS=1` (Peekaboo-conventie).
- Zie [Gebruik van PeekabooBridge](/nl/platforms/mac/peekaboo) voor details.

## Operationele processen

- Herstarten/opnieuw bouwen: `scripts/restart-mac.sh` beëindigt bestaande instanties, bouwt opnieuw met Swift, verpakt opnieuw en start de app opnieuw. Het detecteert automatisch een beschikbare ondertekeningsidentiteit en valt terug op `--no-sign` als er geen wordt gevonden; geef `--sign` door om ondertekening te vereisen (mislukt als er geen sleutel beschikbaar is) of `--no-sign` om het niet-ondertekende pad af te dwingen. `SIGN_IDENTITY` in de omgeving wordt op het ondertekende pad verwijderd, zodat de eigen automatische identiteitsdetectie van `scripts/codesign-mac-app.sh` het certificaat selecteert.
- Eén instantie: de app controleert `NSWorkspace.runningApplications` op een dubbele bundel-ID en sluit af als er meer dan één instantie wordt gevonden (`isDuplicateInstance()` in `MenuBar.swift`).

## Opmerkingen over beveiliging

- Geef er de voorkeur aan om voor alle bevoorrechte oppervlakken een overeenkomende TeamID te vereisen.
- PeekabooBridge: `PEEKABOO_ALLOW_UNSIGNED_SOCKET_CLIENTS=1` (alleen DEBUG) kan aanroepers met dezelfde UID toestaan voor lokale ontwikkeling.
- Alle communicatie blijft uitsluitend lokaal; er worden geen netwerksockets beschikbaar gesteld.
- TCC-prompts zijn uitsluitend afkomstig van de GUI-appbundel; houd de ondertekende bundel-ID stabiel bij nieuwe builds.
- Versteviging van de socket voor uitvoeringsgoedkeuringen: bestandsmodus `0600`, gedeeld token, controle van de peer-UID (`getpeereid`), HMAC-SHA256-uitdaging/antwoord en een korte TTL voor verzoeken.

## Gerelateerd

- [macOS-app](/nl/platforms/macos)
- [macOS-IPC-proces (uitvoeringsgoedkeuringen)](/nl/tools/exec-approvals-advanced#macos-ipc-flow)
