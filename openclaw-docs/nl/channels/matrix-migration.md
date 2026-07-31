---
read_when:
    - Een bestaande Matrix-installatie upgraden
    - Versleutelde Matrix-geschiedenis en apparaatstatus migreren
summary: Hoe OpenClaw de vorige Matrix-plugin ter plaatse bijwerkt, inclusief beperkingen voor het herstel van versleutelde toestand en handmatige herstelstappen.
title: Matrix-migratie
x-i18n:
    generated_at: "2026-07-27T04:56:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 475c96914900a5597f37001264bd3d8f69a69dbd0600f2704c2a1be46924fac4
    source_path: channels/matrix-migration.md
    workflow: 16
---

Upgrade van de vorige openbare `matrix`-plugin naar de huidige implementatie.

Voor de meeste gebruikers verloopt de upgrade zonder verdere aanpassingen:

- de plugin blijft `@openclaw/matrix`
- het kanaal blijft `matrix`
- je configuratie blijft onder `channels.matrix`
- in de cache opgeslagen referenties worden verplaatst naar de gedeelde pluginstatus van `state/openclaw.sqlite`
- de runtimestatus blijft onder `~/.openclaw/matrix/`

Je hoeft configuratiesleutels niet te hernoemen of de plugin onder een nieuwe naam opnieuw te installeren.
Het hoofdpackage `openclaw` bundelt niet langer Matrix-runtimecode of
afhankelijkheden van de Matrix-SDK. Als `openclaw channels status` aangeeft dat Matrix is geconfigureerd, maar de
plugin niet is geïnstalleerd, voer dan `openclaw doctor --fix` of
`openclaw plugins install @openclaw/matrix` uit; installeer geen Matrix-SDK-packages
in het hoofdpackage van OpenClaw.

## Wat de migratie automatisch doet

De Matrix-migratie wordt uitgevoerd wanneer je [`openclaw doctor --fix`](/nl/gateway/doctor) uitvoert. Bestandsgebaseerde sidecars naast de afzonderlijke Matrix-opslag behouden hun terugvaloptie bij het starten van de client, maar de import van referentiebestanden wordt alleen door Doctor uitgevoerd; de runtime leest uitsluitend de canonieke SQLite-status van referenties.

De Doctor-migratie omvat:

- het importeren en verifiëren van uitgefaseerde `~/.openclaw/credentials/matrix/credentials*.json`-bestanden voordat ze worden gearchiveerd
- het behouden van dezelfde accountselectie en `channels.matrix`-configuratie
- het importeren van bestandsgebaseerde sidecarstatus (`bot-storage.json`-synchronisatiecache, `recovery-key.json`, `legacy-crypto-migration.json`, IndexedDB-momentopnamen) naar de SQLite-status van Matrix; gemigreerde bestanden worden gearchiveerd met het achtervoegsel `.migrated`
- het opnieuw gebruiken van de meest volledige bestaande opslagroot voor tokenhashes voor hetzelfde Matrix-account, dezelfde homeserver, gebruiker en hetzelfde apparaat wanneer het toegangstoken later verandert

## Upgraden vanaf OpenClaw-releases ouder dan 2026.4

Releases tot en met de 2026.6-reeks migreerden ook de oorspronkelijke platte Matrix-indeling met één opslaglocatie
(`~/.openclaw/matrix/bot-storage.json` plus
`~/.openclaw/matrix/crypto/`) en bereidden herstel van versleutelde status uit de
oude cryptografische Rust-opslag voor. Huidige releases bevatten die migratie niet meer.

Als je een installatie upgradet die nog steeds de platte indeling gebruikt, upgrade dan eerst
naar een 2026.6-release, voer `openclaw doctor --fix` uit en start de Gateway
één keer, zodat de platte opslag en eventuele herstelbare ruimtesleutels worden gemigreerd. Werk daarna
bij naar de nieuwste release.

De vorige openbare Matrix-plugin maakte **niet** automatisch back-ups van Matrix-ruimtesleutels. Als je oude installatie uitsluitend lokale versleutelde geschiedenis bevatte waarvan nooit een back-up was gemaakt, kunnen sommige oudere versleutelde berichten na de upgrade onleesbaar blijven, ongeacht het migratiepad.

## Aanbevolen upgradeprocedure

1. Werk OpenClaw en de Matrix-plugin op de gebruikelijke manier bij.
2. Voer het volgende uit:

   ```bash
   openclaw doctor --fix
   ```

3. Start of herstart de Gateway.
4. Controleer de huidige verificatie- en back-upstatus:

   ```bash
   openclaw matrix verify status
   openclaw matrix verify backup status
   ```

5. Plaats de herstelsleutel voor het Matrix-account dat je herstelt in een accountspecifieke omgevingsvariabele. Voor één standaardaccount is `MATRIX_RECOVERY_KEY` geschikt. Gebruik voor meerdere accounts één variabele per account, bijvoorbeeld `MATRIX_RECOVERY_KEY_ASSISTANT`, en voeg `--account assistant` toe aan de opdracht.

6. Als OpenClaw aangeeft dat een herstelsleutel nodig is, voer dan de opdracht voor het bijbehorende account uit:

   ```bash
   printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify backup restore --recovery-key-stdin
   printf '%s\n' "$MATRIX_RECOVERY_KEY_ASSISTANT" | openclaw matrix verify backup restore --recovery-key-stdin --account assistant
   ```

7. Als dit apparaat nog steeds niet is geverifieerd, voer dan de opdracht voor het bijbehorende account uit:

   ```bash
   printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify device --recovery-key-stdin
   printf '%s\n' "$MATRIX_RECOVERY_KEY_ASSISTANT" | openclaw matrix verify device --recovery-key-stdin --account assistant
   ```

   Als de herstelsleutel wordt geaccepteerd en de back-up bruikbaar is, maar `Cross-signing verified`
   nog steeds `no` is, voltooi dan de zelfverificatie vanuit een andere Matrix-client:

   ```bash
   openclaw matrix verify self
   ```

   Accepteer het verzoek in een andere Matrix-client, vergelijk de emoji of decimalen
   en typ alleen `yes` als ze overeenkomen. De opdracht wacht op volledig vertrouwen in de Matrix-
   identiteit voordat succes wordt gemeld.

8. Als je bewust niet-herstelbare oude geschiedenis opgeeft en een nieuwe back-upbasis voor toekomstige berichten wilt, voer dan het volgende uit:

   ```bash
   openclaw matrix verify backup reset --yes
   ```

   Voeg `--rotate-recovery-key` alleen toe wanneer de oude herstelsleutel de nieuwe back-up niet meer mag kunnen ontgrendelen.

9. Als er nog geen back-up van sleutels op de server bestaat, maak er dan een voor toekomstig herstel:

   ```bash
   openclaw matrix verify bootstrap
   ```

## Veelvoorkomende meldingen en hun betekenis

`Failed migrating legacy Matrix client storage: ...`

- Betekenis: de terugvaloptie aan de clientzijde van Matrix heeft bestandsgebaseerde sidecarstatus gevonden, maar het importeren naar SQLite is mislukt. OpenClaw draait voltooide verplaatsingen terug en breekt die terugvaloptie af, in plaats van ongemerkt met een nieuwe opslag te starten.
- Wat te doen: controleer bestandssysteemmachtigingen of conflicten, houd de oude status intact en probeer het opnieuw nadat je de fout hebt opgelost.

`Matrix is installed from a custom path: ...`

- Betekenis: Matrix is vastgezet op een installatie via een pad, waardoor reguliere updates deze niet automatisch vervangen door het standaardpackage van Matrix.
- Wat te doen: installeer opnieuw met `openclaw plugins install @openclaw/matrix` wanneer je wilt terugkeren naar de standaardplugin voor Matrix.

`Matrix is installed from a custom path that no longer exists: ...`

- Betekenis: de installatierecord van je plugin verwijst naar een lokaal pad dat niet meer bestaat.
- Wat te doen: installeer opnieuw met `openclaw plugins install @openclaw/matrix`, of gebruik `openclaw plugins install ./path/to/local/matrix-plugin` als je vanuit een uitgecheckte repository werkt. `openclaw doctor --fix` kan ook de verouderde verwijzingen naar de Matrix-plugin voor je verwijderen.

### Meldingen voor handmatig herstel

`openclaw matrix verify status` en `openclaw matrix verify backup status` tonen een regel `Backup issue:` plus richtlijnen voor `Next steps:` wanneer de back-up van ruimtesleutels op dit apparaat niet in orde is:

| Back-upprobleem                                                        | Betekenis                                          | Oplossing                                                                                                                                 |
| --------------------------------------------------------------------- | -------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `no room-key backup exists on the homeserver`                         | niets om van te herstellen                         | `openclaw matrix verify bootstrap` om een back-up van ruimtesleutels te maken                                                                            |
| `backup decryption key is not loaded on this device`                  | sleutel bestaat, maar is hier niet actief          | `openclaw matrix verify backup restore`; als de sleutel nog steeds niet kan worden geladen, leid je de herstelsleutel via `--recovery-key-stdin` door                |
| `backup decryption key could not be loaded from secret storage (...)` | het laden van geheime opslag is mislukt of wordt niet ondersteund | leid de herstelsleutel door: `printf '%s\n' "$MATRIX_RECOVERY_KEY" \| openclaw matrix verify backup restore --recovery-key-stdin`               |
| `backup key mismatch (...)`                                           | opgeslagen sleutel komt niet overeen met de actieve serverback-up | voer `verify backup restore --recovery-key-stdin` opnieuw uit met de sleutel van de actieve serverback-up, of `verify backup reset --yes` voor een nieuwe basis |
| `backup signature chain is not trusted by this device`                | apparaat vertrouwt de keten voor kruislingse ondertekening nog niet | `verify device --recovery-key-stdin`, daarna `verify self` vanuit een andere geverifieerde client als het vertrouwen nog onvolledig is                        |
| `backup exists but is not active on this device`                      | serverback-up aanwezig, lokale sessie inactief     | verifieer eerst het apparaat en controleer daarna opnieuw met `openclaw matrix verify backup status`                                                         |
| `backup trust state could not be fully determined`                    | diagnose leverde geen uitsluitsel                  | `openclaw matrix verify status --verbose`                                                                                                 |

Andere herstelfouten:

`Matrix recovery key is required`

- Betekenis: je hebt een herstelstap geprobeerd zonder een herstelsleutel op te geven terwijl die vereist was.
- Wat te doen: voer de opdracht opnieuw uit met `--recovery-key-stdin`, bijvoorbeeld `printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify device --recovery-key-stdin`.

`Invalid Matrix recovery key: ...`

- Betekenis: de opgegeven sleutel kon niet worden geparseerd of kwam niet overeen met de verwachte indeling.
- Wat te doen: probeer het opnieuw met de exacte herstelsleutel uit je Matrix-client of uit de export van de herstelsleutel.

`Matrix recovery key was applied, but this device still lacks full Matrix identity trust.`

- Betekenis: de herstelsleutel heeft bruikbaar back-upmateriaal ontgrendeld, maar Matrix heeft nog geen volledig identiteitsvertrouwen via kruislingse ondertekening voor dit apparaat vastgesteld. Controleer de uitvoer van de opdracht op `Recovery key accepted`, `Backup usable`, `Cross-signing verified` en `Device verified by owner`.
- Wat te doen: voer `openclaw matrix verify self` uit, accepteer het verzoek in een andere Matrix-client, vergelijk de SAS en typ alleen `yes` als deze overeenkomt. Gebruik `printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify bootstrap --recovery-key-stdin --force-reset-cross-signing` alleen wanneer je bewust de huidige identiteit voor kruislingse ondertekening wilt vervangen.

Als je accepteert dat niet-herstelbare oude versleutelde geschiedenis verloren gaat, kun je in plaats daarvan de
huidige back-upbasis opnieuw instellen met `openclaw matrix verify backup reset --yes`. Wanneer het
opgeslagen back-upgeheim defect is, herstelt die reset ook de geheime opslag, zodat de
nieuwe back-upsleutel na een herstart correct kan worden geladen.

## Als de versleutelde geschiedenis nog steeds niet terugkomt

Voer deze controles in de aangegeven volgorde uit:

```bash
openclaw matrix verify status --verbose
openclaw matrix verify backup status --verbose
printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify backup restore --recovery-key-stdin --verbose
```

Als de back-up met succes wordt hersteld, maar de geschiedenis in sommige oude ruimtes nog steeds ontbreekt, is er door de vorige plugin waarschijnlijk nooit een back-up van die ontbrekende sleutels gemaakt.

## Als je opnieuw wilt beginnen voor toekomstige berichten

Als je accepteert dat niet-herstelbare oude versleutelde geschiedenis verloren gaat en voortaan alleen een schone back-upbasis wilt, voer je deze opdrachten in de aangegeven volgorde uit:

```bash
openclaw matrix verify backup reset --yes
openclaw matrix verify backup status --verbose
openclaw matrix verify status
```

Als het apparaat daarna nog steeds niet is geverifieerd, voltooi je de verificatie vanuit je Matrix-client door de SAS-emoji of decimale codes te vergelijken en te bevestigen dat ze overeenkomen.

## Gerelateerd

- [Matrix](/nl/channels/matrix): kanaalconfiguratie en configuratie.
- [Matrix-pushregels](/nl/channels/matrix-push-rules): routering van meldingen.
- [Doctor](/nl/gateway/doctor): statuscontrole en automatische migratietrigger.
- [Migratiehandleiding](/nl/install/migrating): alle migratiepaden (machineverplaatsingen, imports tussen systemen).
- [Plugins](/nl/tools/plugin): installatie en registratie van plugins.
