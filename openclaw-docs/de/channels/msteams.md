---
read_when:
    - Arbeiten an Funktionen des Microsoft-Teams-Kanals
summary: Status, Funktionen und Konfiguration der Microsoft Teams-Bot-Unterstützung
title: Microsoft Teams
x-i18n:
    generated_at: "2026-07-26T18:49:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5a4cf686da27e28b58f7afaad8cc837dbddb93219cde0c37285f9f6895f6fb8c
    source_path: channels/msteams.md
    workflow: 16
---

Status: Text- und DM-Anhänge werden unterstützt; das Senden von Dateien in Kanälen/Gruppen erfordert `sharePointSiteId` + Graph-Berechtigungen (siehe [Dateien in Gruppenchats senden](#sending-files-in-group-chats)). Umfragen werden über Adaptive Cards gesendet. Nachrichtenaktionen stellen explizit `upload-file` für Sendungen bereit, bei denen die Datei an erster Stelle steht.

## Gebündeltes Plugin

Microsoft Teams wird in aktuellen OpenClaw-Versionen als gebündeltes Plugin ausgeliefert; im normalen paketierten Build ist keine separate Installation erforderlich.

Installieren Sie bei einem älteren Build oder einer benutzerdefinierten Installation, die das gebündelte Teams ausschließt, das npm-Paket direkt:

```bash
openclaw plugins install @openclaw/msteams
```

Verwenden Sie das Paket ohne Versionsangabe, um dem aktuellen offiziellen Release-Tag zu folgen. Fixieren Sie eine exakte Version nur, wenn Sie eine reproduzierbare Installation benötigen.

Lokaler Checkout (Ausführung aus einem Git-Repository):

```bash
openclaw plugins install ./path/to/local/msteams-plugin
```

Details: [Plugins](/de/tools/plugin)

## Schnelleinrichtung

[`@microsoft/teams.cli`](https://www.npmjs.com/package/@microsoft/teams.cli) übernimmt Bot-Registrierung, Manifest-Erstellung und Generierung von Anmeldedaten mit einem einzigen Befehl.

**1. Installieren und anmelden**

```bash
npm install -g @microsoft/teams.cli@preview
teams login
teams status   # Prüfen, ob Sie angemeldet sind, und Mandanteninformationen anzeigen
```

<Note>
Die Teams CLI befindet sich derzeit in der Vorschauphase. Befehle und Flags können sich zwischen Releases ändern.
</Note>

**2. Einen Tunnel starten** (Teams kann localhost nicht erreichen)

Installieren und authentifizieren Sie bei Bedarf die devtunnel CLI ([Leitfaden für die ersten Schritte](https://learn.microsoft.com/en-us/azure/developer/dev-tunnels/get-started)).

```bash
# Einmalige Einrichtung (dauerhafte URL über Sitzungen hinweg):
devtunnel create my-openclaw-bot --allow-anonymous
devtunnel port create my-openclaw-bot -p 3978 --protocol auto

# Für jede Entwicklungssitzung:
devtunnel host my-openclaw-bot
# Ihr Endpunkt: https://<tunnel-id>.devtunnels.ms/api/messages
```

<Note>
`--allow-anonymous` ist erforderlich, da Teams sich nicht bei devtunnels authentifizieren kann. Jede eingehende Bot-Anfrage wird weiterhin durch das Teams SDK validiert.
</Note>

Alternativen: `ngrok http 3978` oder `tailscale funnel 3978` (URLs können sich bei jeder Sitzung ändern).

**3. Die App erstellen**

```bash
teams app create \
  --name "OpenClaw" \
  --endpoint "https://<your-tunnel-url>/api/messages"
```

Dadurch wird eine Entra ID-Anwendung (Azure AD) erstellt, ein Client-Geheimnis generiert, ein Teams-App-Manifest (mit Symbolen) erstellt und hochgeladen sowie ein von Teams verwalteter Bot registriert (kein Azure-Abonnement erforderlich). Die Ausgabe enthält `CLIENT_ID`, `CLIENT_SECRET`, `TENANT_ID` und eine **Teams App ID**; außerdem wird angeboten, die App direkt in Teams zu installieren.

**4. OpenClaw konfigurieren** – verwenden Sie dazu die Anmeldedaten aus der Ausgabe:

```json5
{
  channels: {
    msteams: {
      enabled: true,
      appId: "<CLIENT_ID>",
      appPassword: "<CLIENT_SECRET>",
      tenantId: "<TENANT_ID>",
      webhook: { port: 3978, path: "/api/messages" },
    },
  },
}
```

Alternativ können Sie direkt Umgebungsvariablen verwenden: `MSTEAMS_APP_ID`, `MSTEAMS_APP_PASSWORD`, `MSTEAMS_TENANT_ID`.

**5. Die App in Teams installieren**

`teams app create` fordert Sie zur Installation der App auf; wählen Sie "Install in Teams". So erhalten Sie den Installationslink später:

```bash
teams app get <teamsAppId> --install-link
```

**6. Prüfen, ob alles funktioniert**

```bash
teams app doctor <teamsAppId>
```

Führt Diagnosen für Bot-Registrierung, AAD-App-Konfiguration, Manifestgültigkeit und SSO-Einrichtung aus.

Erwägen Sie für den Produktionseinsatz [föderierte Authentifizierung](#federated-authentication-certificate-plus-managed-identity) (Zertifikat oder verwaltete Identität) anstelle von Client-Geheimnissen.

<Note>
Gruppenchats sind standardmäßig blockiert (`channels.msteams.groupPolicy: "allowlist"`). Um Gruppenantworten zuzulassen, setzen Sie `channels.msteams.groupAllowFrom`, oder verwenden Sie `groupPolicy: "open"`, um jedes Mitglied zuzulassen (Erwähnung erforderlich).
</Note>

## Ziele

- Kommunizieren Sie über Teams-DMs, Gruppenchats oder Kanäle mit OpenClaw.
- Halten Sie das Routing deterministisch: Antworten gehen immer an den Kanal zurück, über den sie eingegangen sind.
- Verwenden Sie standardmäßig ein sicheres Kanalverhalten (Erwähnungen erforderlich, sofern nicht anders konfiguriert).

## Konfigurationsänderungen

Standardmäßig kann Microsoft Teams durch `/config set|unset` ausgelöste Konfigurationsänderungen schreiben (erfordert `commands.config: true`).

Deaktivieren mit:

```json5
{
  channels: { msteams: { configWrites: false } },
}
```

## Zugriffskontrolle (DMs + Gruppen)

**DM-Zugriff**

- Standard: `channels.msteams.dmPolicy = "pairing"`. Unbekannte Absender werden ignoriert, bis sie genehmigt wurden.
- `channels.msteams.allowFrom` sollte stabile AAD-Objekt-IDs oder statische Absenderzugriffsgruppen wie `accessGroup:core-team` verwenden.
- Verlassen Sie sich bei Zulassungslisten nicht auf den Abgleich von UPNs/Anzeigenamen; diese können sich ändern. OpenClaw deaktiviert den direkten Namensabgleich standardmäßig; aktivieren Sie ihn mit `channels.msteams.dangerouslyAllowNameMatching: true`.
- Der Assistent kann Namen über Microsoft Graph in IDs auflösen, sofern die Anmeldedaten dies erlauben.

**Gruppenzugriff**

- Standard: `channels.msteams.groupPolicy = "allowlist"` (blockiert, sofern Sie nicht `groupAllowFrom` hinzufügen). `channels.defaults.groupPolicy` kann den gemeinsamen Standardwert überschreiben, wenn `channels.msteams.groupPolicy` nicht gesetzt ist.
- `channels.msteams.groupAllowFrom` steuert, welche Absender oder statischen Absenderzugriffsgruppen in Gruppenchats/Kanälen eine Aktion auslösen können (greift auf `channels.msteams.allowFrom` zurück).
- Setzen Sie `groupPolicy: "open"`, um jedes Mitglied zuzulassen (standardmäßig ist weiterhin eine Erwähnung erforderlich).
- Um **alle** Kanäle zu blockieren, setzen Sie `channels.msteams.groupPolicy: "disabled"`.

Beispiel:

```json5
{
  channels: {
    msteams: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["00000000-0000-0000-0000-000000000000", "accessGroup:core-team"],
    },
  },
}
```

**Zulassungsliste für Team + Kanal**

- Begrenzen Sie Gruppen-/Kanalantworten, indem Sie Teams und Kanäle unter `channels.msteams.teams` aufführen.
- Verwenden Sie stabile Teams-Unterhaltungs-IDs aus Teams-Links als Schlüssel, nicht veränderliche Anzeigenamen (siehe [Team- und Kanal-IDs](#team-and-channel-ids-common-gotcha)).
- Wenn `groupPolicy="allowlist"` und eine Teams-Zulassungsliste vorhanden sind, werden nur aufgeführte Teams/Kanäle akzeptiert (Erwähnung erforderlich).
- Der Konfigurationsassistent akzeptiert `Team/Channel`-Einträge und speichert sie für Sie.
- Beim Start löst OpenClaw die Namen in Team-/Kanal- und Benutzerzulassungslisten in IDs auf (sofern Graph-Berechtigungen dies erlauben) und protokolliert die Zuordnung. Nicht aufgelöste Namen bleiben wie eingegeben erhalten, werden beim Routing jedoch ignoriert, sofern `channels.msteams.dangerouslyAllowNameMatching: true` nicht gesetzt ist.

Beispiel:

```json5
{
  channels: {
    msteams: {
      groupPolicy: "allowlist",
      teams: {
        "My Team": {
          channels: {
            General: { requireMention: true },
          },
        },
      },
    },
  },
}
```

<details>
<summary><strong>Manuelle Einrichtung (ohne Teams CLI)</strong></summary>

### Funktionsweise

1. Stellen Sie sicher, dass das Microsoft Teams-Plugin verfügbar ist (in aktuellen Releases gebündelt).
2. Erstellen Sie einen **Azure Bot** (App-ID + Geheimnis + Mandanten-ID).
3. Erstellen Sie ein **Teams-App-Paket**, das auf den Bot verweist und die unten aufgeführten RSC-Berechtigungen enthält.
4. Laden/installieren Sie die Teams-App in einem Team (oder im persönlichen Bereich für DMs).
5. Konfigurieren Sie `msteams` in `~/.openclaw/openclaw.json` (oder Umgebungsvariablen) und starten Sie das Gateway.
6. Das Gateway lauscht standardmäßig unter `/api/messages` auf Webhook-Datenverkehr des Bot Framework.

### Schritt 1: Azure Bot erstellen

1. Rufen Sie [Create Azure Bot](https://portal.azure.com/#create/Microsoft.AzureBot) auf.
2. Füllen Sie die Registerkarte **Basics** aus:

   | Feld               | Wert                                                               |
   | ------------------ | ------------------------------------------------------------------ |
   | **Bot handle**     | Ihr Bot-Name, z. B. `openclaw-msteams` (muss eindeutig sein)       |
   | **Subscription**   | Wählen Sie Ihr Azure-Abonnement aus                                |
   | **Resource group** | Erstellen Sie eine neue oder verwenden Sie eine vorhandene         |
   | **Pricing tier**   | **Free** für Entwicklung/Tests                                     |
   | **Type of App**    | **Single Tenant** (empfohlen; siehe Hinweis unten)                  |
   | **Creation type**  | **Create new Microsoft App ID**                                    |

<Warning>
Die Erstellung neuer mandantenfähiger Bots wurde nach dem 2025-07-31 eingestellt. Verwenden Sie für neue Bots **Single Tenant**.
</Warning>

3. Klicken Sie auf **Review + create** und anschließend auf **Create** (~1-2 Minuten).

### Schritt 2: Anmeldedaten abrufen

1. Azure Bot-Ressource → **Configuration** → kopieren Sie die **Microsoft App ID** (Ihre `appId`).
2. **Manage Password** → App-Registrierung → **Certificates & secrets** → **New client secret** → kopieren Sie den **Value** (Ihr `appPassword`).
3. **Overview** → kopieren Sie die **Directory (tenant) ID** (Ihre `tenantId`).

### Schritt 3: Messaging-Endpunkt konfigurieren

1. Azure Bot → **Configuration**.
2. Legen Sie den **Messaging endpoint** fest:
   - Produktion: `https://your-domain.com/api/messages`
   - Lokale Entwicklung: Verwenden Sie einen Tunnel (siehe [Lokale Entwicklung](#local-development-tunneling)).

### Schritt 4: Teams-Kanal aktivieren

1. Azure Bot → **Channels**.
2. Klicken Sie auf **Microsoft Teams** → Configure → Save.
3. Akzeptieren Sie die Nutzungsbedingungen.

### Schritt 5: Teams-App-Manifest erstellen

- Fügen Sie einen `bot`-Eintrag mit `botId = <App ID>` ein.
- Bereiche: `personal`, `team`, `groupChat`.
- `supportsFiles: true` (für die Dateiverarbeitung im persönlichen Bereich erforderlich).
- Fügen Sie RSC-Berechtigungen hinzu (siehe [RSC-Berechtigungen](#current-teams-rsc-permissions-manifest)).
- Erstellen Sie Symbole: `outline.png` (32x32) und `color.png` (192x192).
- Komprimieren Sie `manifest.json`, `outline.png` und `color.png` gemeinsam in eine ZIP-Datei.

### Schritt 6: OpenClaw konfigurieren

```json5
{
  channels: {
    msteams: {
      enabled: true,
      appId: "<APP_ID>",
      appPassword: "<APP_PASSWORD>",
      tenantId: "<TENANT_ID>",
      webhook: { port: 3978, path: "/api/messages" },
    },
  },
}
```

Umgebungsvariablen: `MSTEAMS_APP_ID`, `MSTEAMS_APP_PASSWORD`, `MSTEAMS_TENANT_ID`.

### Schritt 7: Gateway ausführen

Der Teams-Kanal startet automatisch, wenn das Plugin verfügbar ist und die `msteams`-Konfiguration Anmeldedaten enthält.

</details>

## Föderierte Authentifizierung (Zertifikat plus verwaltete Identität)

Für den Produktionseinsatz unterstützt OpenClaw über `channels.msteams.authType: "federated"` **föderierte Authentifizierung** als Alternative zu Client-Geheimnissen. Es gibt zwei Methoden:

### Option A: Zertifikatbasierte Authentifizierung

Verwenden Sie ein PEM-Zertifikat, das bei Ihrer Entra ID-App-Registrierung registriert ist.

**Einrichtung:**

1. Generieren oder beschaffen Sie ein Zertifikat (PEM-Format mit privatem Schlüssel).
2. Entra ID → App-Registrierung → **Certificates & secrets** → **Certificates** → laden Sie das öffentliche Zertifikat hoch.

**Konfiguration:**

```json5
{
  channels: {
    msteams: {
      enabled: true,
      appId: "<APP_ID>",
      tenantId: "<TENANT_ID>",
      authType: "federated",
      certificatePath: "/path/to/cert.pem",
      webhook: { port: 3978, path: "/api/messages" },
    },
  },
}
```

**Umgebungsvariablen:**

- `MSTEAMS_AUTH_TYPE=federated`
- `MSTEAMS_CERTIFICATE_PATH=/path/to/cert.pem`

### Option B: Verwaltete Azure-Identität

Verwenden Sie eine verwaltete Azure-Identität für die kennwortlose Authentifizierung auf Azure-Infrastruktur (AKS, App Service, Azure-VMs).

**Funktionsweise:**

1. Der Bot-Pod/die VM verfügt über eine verwaltete Identität (system- oder benutzerseitig zugewiesen).
2. Anmeldedaten für eine föderierte Identität verknüpfen die verwaltete Identität mit der Entra ID-App-Registrierung.
3. Zur Laufzeit verwendet OpenClaw `@azure/identity`, um Token vom Azure-IMDS-Endpunkt abzurufen.
4. Das Token wird zur Bot-Authentifizierung an das Teams SDK übergeben.

**Voraussetzungen:**

- Azure-Infrastruktur mit aktivierter verwalteter Identität (AKS-Workloadidentität, App Service, VM).
- Für die Entra-ID-App-Registrierung erstellte Anmeldeinformationen für die Verbundidentität.
- Netzwerkzugriff auf IMDS (`169.254.169.254:80`) vom Pod bzw. von der VM.

**Konfiguration (systemseitig zugewiesene verwaltete Identität):**

```json5
{
  channels: {
    msteams: {
      enabled: true,
      appId: "<APP_ID>",
      tenantId: "<TENANT_ID>",
      authType: "federated",
      useManagedIdentity: true,
      webhook: { port: 3978, path: "/api/messages" },
    },
  },
}
```

**Konfiguration (benutzerseitig zugewiesene verwaltete Identität):** Fügen Sie dem obigen Block `managedIdentityClientId: "<MI_CLIENT_ID>"` hinzu.

**Umgebungsvariablen:**

- `MSTEAMS_AUTH_TYPE=federated`
- `MSTEAMS_USE_MANAGED_IDENTITY=true`
- `MSTEAMS_MANAGED_IDENTITY_CLIENT_ID=<client-id>` (nur benutzerseitig zugewiesen)

### Einrichtung der AKS-Workloadidentität

Für AKS-Bereitstellungen mit Workloadidentität:

1. **Aktivieren Sie die Workloadidentität** für Ihren AKS-Cluster.
2. **Erstellen Sie Anmeldeinformationen für die Verbundidentität** in der Entra-ID-App-Registrierung:

   ```bash
   az ad app federated-credential create --id <APP_OBJECT_ID> --parameters '{
     "name": "my-bot-workload-identity",
     "issuer": "<AKS_OIDC_ISSUER_URL>",
     "subject": "system:serviceaccount:<NAMESPACE>:<SERVICE_ACCOUNT>",
     "audiences": ["api://AzureADTokenExchange"]
   }'
   ```

3. **Versehen Sie das Kubernetes-Dienstkonto mit einer Annotation** für die Client-ID der App:

   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: my-bot-sa
     annotations:
       azure.workload.identity/client-id: "<APP_CLIENT_ID>"
   ```

4. **Versehen Sie den Pod mit einem Label** für die Einbindung der Workloadidentität:

   ```yaml
   metadata:
     labels:
       azure.workload.identity/use: "true"
   ```

5. **Erlauben Sie den Netzwerkzugriff** auf IMDS (`169.254.169.254`): Wenn Sie NetworkPolicy verwenden, fügen Sie für `169.254.169.254/32` an Port 80 eine ausgehende Regel hinzu.

### Vergleich der Authentifizierungstypen

| Methode                         | Konfiguration                                  | Vorteile                                   | Nachteile                                              |
| ------------------------------- | ---------------------------------------------- | ------------------------------------------ | ------------------------------------------------------ |
| **Clientgeheimnis**             | `appPassword`                             | Einfache Einrichtung                       | Rotation des Geheimnisses erforderlich, weniger sicher |
| **Zertifikat**                  | `authType: "federated"` + `certificatePath`        | Kein gemeinsames Geheimnis über das Netzwerk | Aufwand für die Zertifikatsverwaltung                  |
| **Verwaltete Identität**        | `authType: "federated"` + `useManagedIdentity`        | Kennwortlos, keine Geheimnisse zu verwalten | Azure-Infrastruktur erforderlich                       |

`certificateThumbprint` kann zusammen mit `certificatePath` festgelegt werden, wird vom Authentifizierungspfad derzeit jedoch nicht gelesen; der Wert wird ausschließlich aus Gründen der Vorwärtskompatibilität akzeptiert.

**Standard:** Wenn `authType` nicht festgelegt ist, verwendet OpenClaw die Authentifizierung mit Clientgeheimnis (`appPassword`). Bestehende Konfigurationen funktionieren unverändert weiter.

## Lokale Entwicklung (Tunneling)

Teams kann `localhost` nicht erreichen. Verwenden Sie einen persistenten Entwicklungstunnel, damit die URL sitzungsübergreifend stabil bleibt:

```bash
# Einmalige Einrichtung:
devtunnel create my-openclaw-bot --allow-anonymous
devtunnel port create my-openclaw-bot -p 3978 --protocol auto

# Bei jeder Entwicklungssitzung:
devtunnel host my-openclaw-bot
```

Alternativen: `ngrok http 3978` oder `tailscale funnel 3978` (URLs können sich bei jeder Sitzung ändern).

Wenn sich die Tunnel-URL ändert, aktualisieren Sie den Endpunkt:

```bash
teams app update <teamsAppId> --endpoint "https://<new-url>/api/messages"
```

## Bot testen

**Diagnose ausführen:**

```bash
teams app doctor <teamsAppId>
```

Prüft Bot-Registrierung, AAD-App, Manifest und SSO-Konfiguration in einem Durchlauf.

**Testnachricht senden:**

1. Installieren Sie die Teams-App (Installationslink aus `teams app get <id> --install-link`).
2. Suchen Sie den Bot in Teams und senden Sie ihm eine Direktnachricht.
3. Prüfen Sie die Gateway-Protokolle auf eingehende Aktivitäten.

## Umgebungsvariablen

Diese authentifizierungsbezogenen Konfigurationsschlüssel können anstelle von `openclaw.json` über Umgebungsvariablen festgelegt werden (andere Konfigurationsschlüssel wie `groupPolicy` oder `historyLimit` können nur in der Konfiguration festgelegt werden):

| Umgebungsvariable                     | Konfigurationsschlüssel    | Hinweise                                    |
| ------------------------------------- | -------------------------- | ------------------------------------------- |
| `MSTEAMS_APP_ID`                    | `appId`         |                                             |
| `MSTEAMS_APP_PASSWORD`                    | `appPassword`         |                                             |
| `MSTEAMS_TENANT_ID`                    | `tenantId`         |                                             |
| `MSTEAMS_AUTH_TYPE`                    | `authType`         | `"secret"` oder `"federated"` |
| `MSTEAMS_CERTIFICATE_PATH`                    | `certificatePath`         | Verbundidentität + Zertifikat               |
| `MSTEAMS_CERTIFICATE_THUMBPRINT`                    | `certificateThumbprint`         | akzeptiert, für die Authentifizierung nicht erforderlich |
| `MSTEAMS_USE_MANAGED_IDENTITY`                    | `useManagedIdentity`         | Verbundidentität + verwaltete Identität     |
| `MSTEAMS_MANAGED_IDENTITY_CLIENT_ID`                    | `managedIdentityClientId`         | nur benutzerseitig zugewiesene verwaltete Identität |

## Aktion für Mitgliedsinformationen

OpenClaw stellt für Microsoft Teams eine Graph-gestützte Aktion `member-info` bereit, damit Agenten und Automatisierungen verifizierte Teilnehmerdetails für eine konfigurierte Unterhaltung auflösen können.

Anforderungen:

- `ChannelSettings.Read.Group`- und `TeamMember.Read.Group`-RSC-Berechtigungen (bereits im empfohlenen Manifest enthalten).

Die Aktion ist immer verfügbar, wenn Graph-Anmeldeinformationen konfiguriert sind; es gibt keinen separaten Schalter `channels.msteams.actions.memberInfo`.
Abfragen für Standardkanäle geben die passende Identität aus der Teammitgliederliste, den Anzeigenamen, die E-Mail-Adresse und die Rollen zurück.
In der aktuellen Direktnachricht oder im aktuellen Gruppenchat kann die Aktion die stabile Benutzer-ID des vertrauenswürdigen Absenders zurückgeben.
Abfragen von Mitgliedern privater/freigegebener Kanäle und nicht aktueller Chats erfordern zusätzliche Berechtigungen für Mitgliederlisten
und werden von der standardmäßigen Berechtigungsbasis abgelehnt.

## Verlaufskontext

- `channels.msteams.historyLimit` steuert, wie viele der letzten Kanal-/Gruppennachrichten in den Prompt eingebettet werden. Als Rückfallwert wird `messages.groupChat.historyLimit` verwendet, danach gilt standardmäßig 50. Legen Sie `0` fest, um die Funktion zu deaktivieren.
- Der abgerufene Threadverlauf wird anhand der Absender-Zulassungslisten (`allowFrom` / `groupAllowFrom`) gefiltert, sodass die anfängliche Befüllung des Threadkontexts nur Nachrichten von zulässigen Absendern enthält.
- Der Kontext zitierter Anhänge (aus dem HTML des Skype-Reply-Schemas in den eigenen Anhängen einer Antwort analysiert) wird ungefiltert weitergegeben; derzeit wendet nur die anfängliche Befüllung aus dem Threadverlauf den Filter der Absender-Zulassungsliste an.
- Der Verlauf von Direktnachrichten kann mit `channels.msteams.dmHistoryLimit` (Benutzerbeiträge) begrenzt werden. Benutzerspezifische Überschreibungen: `channels.msteams.dms["<user_id>"].historyLimit`.

## Aktuelle Teams-RSC-Berechtigungen (Manifest)

Dies sind die **vorhandenen resourceSpecific-Berechtigungen** in unserem Teams-App-Manifest. Sie gelten nur innerhalb des Teams/Chats, in dem die App installiert ist.

**Für Kanäle (Teambereich):**

- `ChannelMessage.Read.Group` (Application) – alle Kanalnachrichten ohne @Erwähnung empfangen
- `ChannelMessage.Send.Group` (Application)
- `Member.Read.Group` (Application)
- `Owner.Read.Group` (Application)
- `ChannelSettings.Read.Group` (Application)
- `TeamMember.Read.Group` (Application)
- `TeamSettings.Read.Group` (Application)

**Für Gruppenchats:**

- `ChatMessage.Read.Chat` (Application) – alle Gruppenchatnachrichten ohne @Erwähnung empfangen

Fügen Sie RSC-Berechtigungen über die Teams-CLI hinzu:

```bash
teams app rsc add <teamsAppId> ChannelMessage.Read.Group --type Application
```

## Beispiel für ein Teams-Manifest (geschwärzt)

Minimales, gültiges Beispiel mit den erforderlichen Feldern. Ersetzen Sie IDs und URLs.

```json5
{
  $schema: "https://developer.microsoft.com/en-us/json-schemas/teams/v1.23/MicrosoftTeams.schema.json",
  manifestVersion: "1.23",
  version: "1.0.0",
  id: "00000000-0000-0000-0000-000000000000",
  name: { short: "OpenClaw" },
  developer: {
    name: "Ihre Organisation",
    websiteUrl: "https://example.com",
    privacyUrl: "https://example.com/privacy",
    termsOfUseUrl: "https://example.com/terms",
  },
  description: { short: "OpenClaw in Teams", full: "OpenClaw in Teams" },
  icons: { outline: "outline.png", color: "color.png" },
  accentColor: "#5B6DEF",
  bots: [
    {
      botId: "11111111-1111-1111-1111-111111111111",
      scopes: ["personal", "team", "groupChat"],
      isNotificationOnly: false,
      supportsCalling: false,
      supportsVideo: false,
      supportsFiles: true,
    },
  ],
  webApplicationInfo: {
    id: "11111111-1111-1111-1111-111111111111",
  },
  authorization: {
    permissions: {
      resourceSpecific: [
        { name: "ChannelMessage.Read.Group", type: "Application" },
        { name: "ChannelMessage.Send.Group", type: "Application" },
        { name: "Member.Read.Group", type: "Application" },
        { name: "Owner.Read.Group", type: "Application" },
        { name: "ChannelSettings.Read.Group", type: "Application" },
        { name: "TeamMember.Read.Group", type: "Application" },
        { name: "TeamSettings.Read.Group", type: "Application" },
        { name: "ChatMessage.Read.Chat", type: "Application" },
      ],
    },
  },
}
```

### Einschränkungen des Manifests (Pflichtfelder)

- `bots[].botId` **muss** mit der App-ID des Azure-Bots übereinstimmen.
- `webApplicationInfo.id` **muss** mit der App-ID des Azure-Bots übereinstimmen.
- `bots[].scopes` muss die Oberflächen enthalten, die Sie verwenden möchten (`personal`, `team`, `groupChat`).
- `bots[].supportsFiles: true` ist für die Dateiverarbeitung im persönlichen Bereich erforderlich.
- `authorization.permissions.resourceSpecific` muss Lese-/Sendeberechtigungen für Kanalverkehr enthalten.

### Vorhandene App aktualisieren

```bash
# Manifest herunterladen, bearbeiten und erneut hochladen
teams app manifest download <teamsAppId> manifest.json
# manifest.json lokal bearbeiten ...
teams app manifest upload manifest.json <teamsAppId>
# Die Version wird automatisch erhöht, wenn sich der Inhalt geändert hat
```

Installieren Sie die App nach der Aktualisierung in jedem Team neu und **beenden Sie Teams vollständig und starten Sie es neu** (schließen Sie nicht nur das Fenster), um zwischengespeicherte App-Metadaten zu löschen.

<details>
<summary>Manuelle Manifestaktualisierung (ohne CLI)</summary>

1. Aktualisieren Sie `manifest.json` mit den neuen Einstellungen.
2. **Erhöhen Sie das Feld `version`** (z. B. `1.0.0` → `1.1.0`).
3. **Erstellen Sie die ZIP-Datei erneut** und schließen Sie die Symbole ein (`manifest.json`, `outline.png`, `color.png`).
4. Laden Sie die neue ZIP-Datei hoch:
   - **Teams Admin Center:** Teams apps → Manage apps → suchen Sie Ihre App → Upload new version.
   - **Querladen:** Teams → Apps → Manage your apps → Upload a custom app.

</details>

## Funktionen: nur RSC im Vergleich zu Graph

### Mit **nur Teams RSC** (App installiert, keine Graph-API-Berechtigungen)

Funktioniert:

- **Textinhalt** von Kanalnachrichten lesen.
- **Textinhalt** von Kanalnachrichten senden.
- Dateianhänge in **persönlichen Nachrichten (Direktnachrichten)** empfangen.

Funktioniert NICHT:

- **Bild- oder Dateiinhalte** in Kanälen/Gruppen (die Nutzlast enthält nur einen HTML-Platzhalter).
- In SharePoint/OneDrive gespeicherte Anhänge herunterladen.
- Nachrichtenverlauf über das Live-Webhook-Ereignis hinaus lesen.

### Mit **Teams RSC + Microsoft-Graph-Anwendungsberechtigungen**

Ermöglicht zusätzlich:

- Gehostete Inhalte herunterladen (in Nachrichten eingefügte Bilder).
- In SharePoint/OneDrive gespeicherte Dateianhänge herunterladen.
- Kanal-/Chatnachrichtenverlauf über Graph lesen.

### RSC im Vergleich zur Graph-API

| Funktion                 | RSC-Berechtigungen     | Graph API                                      |
| ------------------------ | ---------------------- | ---------------------------------------------- |
| **Echtzeitnachrichten**  | Ja (über Webhook)      | Nein (nur Polling)                             |
| **Historische Nachrichten** | Nein                | Ja (Verlauf kann abgefragt werden)             |
| **Einrichtungskomplexität** | Nur App-Manifest    | Erfordert Administratoreinwilligung + Token-Ablauf |
| **Funktioniert offline** | Nein (muss ausgeführt werden) | Ja (jederzeit abfragbar)                 |

**Fazit:** RSC dient zum Echtzeit-Mithören; die Graph API dient dem Zugriff auf den Verlauf. Um offline verpasste Nachrichten nachträglich abzurufen, benötigen Sie die Graph API mit `ChannelMessage.Read.All` (erfordert Administratoreinwilligung).

## Graph-gestützte Medien und Verlauf

Aktivieren Sie nur die Microsoft-Graph-Anwendungsberechtigungen, die für die von Ihnen verwendeten Teams-Bereiche und -Daten erforderlich sind:

1. Entra ID (Azure AD) **App Registration** → Graph-**Anwendungsberechtigungen** hinzufügen:
   - `ChannelMessage.Read.All` für Kanalanhänge und Kanalverlauf.
   - `Chat.Read.All` für Gruppenchat-Anhänge und Gruppenchat-Verlauf.
   - `Files.Read.All`, wenn Anhangsdaten aus dem SharePoint-/OneDrive-Speicher heruntergeladen werden müssen; reine Verlaufsinstallationen benötigen diese Berechtigung nicht.
2. **Grant admin consent** für den Mandanten.
3. Die **Manifestversion** der Teams-App erhöhen, erneut hochladen und **die App in Teams neu installieren**.
4. **Teams vollständig beenden und neu starten**, um zwischengespeicherte App-Metadaten zu löschen.

### Wiederherstellung von Kanal-/Gruppendateien (`graphMediaFallback`)

Teams kann Dateimarkierungen aus der HTML-Aktivität entfernen, die an einen Bot gesendet wird. In diesem Fall ist die Bot-Framework-Aktivität nicht von einer gewöhnlichen HTML-Nachricht zu unterscheiden; die vollständige Anhangsreferenz ist nur in der Graph-Kopie der Nachricht vorhanden.

Aktivieren Sie nach Erteilung der oben genannten Berechtigungen den Fallback:

```json5
{
  channels: {
    msteams: {
      graphMediaFallback: true,
    },
  },
}
```

Dies gilt nur für Kanäle und Gruppenchats. Es fügt immer dann eine Graph-Nachrichtenabfrage hinzu, wenn eine HTML-Aktivität keine direkt herunterladbaren Medien erzeugt hat, einschließlich gewöhnlicher Nachrichten oder Nachrichten, die nur eine Erwähnung enthalten. Der Standardwert ist `false`, damit bestehende Installationen nicht automatisch zusätzlichen Graph-Datenverkehr oder Berechtigungsfehler verursachen.

**Benutzererwähnungen:** @Erwähnungen funktionieren standardmäßig für Benutzer, die bereits an der Unterhaltung beteiligt sind. Um Benutzer, die **nicht an der aktuellen Unterhaltung beteiligt sind**, dynamisch zu suchen und zu erwähnen, fügen Sie die Berechtigung `User.Read.All` (Anwendung) hinzu und erteilen Sie die Administratoreinwilligung.

## Bekannte Einschränkungen

### Webhook-Zeitüberschreitungen

Teams übermittelt Nachrichten über einen HTTP-Webhook. OpenClaw wendet auf diesen Webhook-Listener feste HTTP-Server-
Zeitüberschreitungen an: 30s Inaktivität, 30s Gesamtanforderungsdauer und 15s
für den Empfang der Header. Für optionale eingehende Medien und die Kontextanreicherung gilt ein gemeinsames
Budget von 10 Sekunden. Das SDK kehrt zurück, nachdem die Rohaktivität dauerhaft angefügt wurde;
der Agent-Durchlauf wird unabhängig abgearbeitet und antwortet proaktiv. Wenn die
Anforderungsverarbeitung oder dauerhafte Annahme das Transportzeitfenster verpasst, versucht Teams möglicherweise erneut, die
Aktivität zuzustellen, und der Ingress-Tombstone weist eine wiederholte Ereignis-ID zurück.

### Unterstützung für Teams-Clouds und Dienst-URLs

Dieser SDK-gestützte Teams-Pfad wird live für die öffentliche Microsoft-Teams-Cloud validiert.

Eingehende Antworten verwenden den Teams-SDK-Durchlaufkontext der eingehenden Nachricht. Kontextunabhängige proaktive Vorgänge – Senden, Bearbeiten, Löschen, Karten, Umfragen, Dateieinwilligungsnachrichten und in die Warteschlange gestellte lang laufende Antworten – verwenden die gespeicherte Unterhaltungsreferenz `serviceUrl`. Die öffentliche Cloud verwendet standardmäßig die öffentliche Cloud-Umgebung des Teams SDK und erlaubt gespeicherte Referenzen auf dem öffentlichen Teams-Connector-Host: `https://smba.trafficmanager.net/`.

Die öffentliche Cloud ist die Standardeinstellung. Für normale Bots in der öffentlichen Cloud müssen Sie `channels.msteams.cloud` oder `channels.msteams.serviceUrl` nicht festlegen.

Legen Sie für nicht öffentliche Teams-Clouds `cloud` und die entsprechende proaktive Begrenzung fest, sobald Microsoft eine veröffentlicht:

- `channels.msteams.cloud` wählt die Cloud-Voreinstellung des Teams SDK für Authentifizierung, JWT-Validierung, Token-Dienste und den Graph-Bereich aus.
- `channels.msteams.serviceUrl` wählt die Bot-Connector-Endpunktbegrenzung aus, mit der gespeicherte Unterhaltungsreferenzen vor proaktivem Senden, Bearbeiten, Löschen, Karten, Umfragen, Dateieinwilligungsnachrichten und in die Warteschlange gestellten lang laufenden Antworten validiert werden. Sie ist für die SDK-Clouds USGov und DoD erforderlich. Für China/21Vianet verwendet OpenClaw die SDK-Voreinstellung `China` und akzeptiert gespeicherte/konfigurierte Dienst-URLs nur auf Azure-China-Hosts des Bot-Framework-Kanals.

Microsoft veröffentlicht die globalen proaktiven Bot-Connector-Endpunkte im Abschnitt [Unterhaltung erstellen](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/conversations/send-proactive-messages?tabs=dotnet#create-the-conversation) der Teams-Dokumentation zu proaktiven Nachrichten. Verwenden Sie `serviceUrl` der eingehenden Aktivität, wenn verfügbar; verwenden Sie andernfalls die nachfolgende Tabelle von Microsoft.

| Teams-Umgebung | OpenClaw-Konfiguration                                     | Proaktive `serviceUrl`                         |
| ----------------- | ----------------------------------------------------------- | -------------------------------------------------- |
| Öffentlich       | keine Cloud-/serviceUrl-Konfiguration erforderlich           | `https://smba.trafficmanager.net/teams`            |
| GCC               | `serviceUrl` festlegen; es gibt keine separate Cloud-Voreinstellung des Teams SDK | `https://smba.infra.gcc.teams.microsoft.com/teams` |
| GCC High          | `cloud: "USGov"` + `serviceUrl`                             | `https://smba.infra.gov.teams.microsoft.us/teams`  |
| DoD               | `cloud: "USGovDoD"` + `serviceUrl`                          | `https://smba.infra.dod.teams.microsoft.us/teams`  |
| China/21Vianet    | `cloud: "China"`                                            | `serviceUrl` der eingehenden Aktivität verwenden           |

Beispiel für GCC, für das Microsoft eine separate proaktive Dienst-URL dokumentiert, das Teams SDK jedoch keine separate GCC-Cloud-Voreinstellung bereitstellt:

```json
{
  "channels": {
    "msteams": {
      "serviceUrl": "https://smba.infra.gcc.teams.microsoft.com/teams"
    }
  }
}
```

Beispiel für GCC High:

```json
{
  "channels": {
    "msteams": {
      "cloud": "USGov",
      "serviceUrl": "https://smba.infra.gov.teams.microsoft.us/teams"
    }
  }
}
```

`channels.msteams.serviceUrl` ist auf unterstützte Microsoft-Teams-Bot-Connector-Hosts beschränkt. Wenn eine Dienst-URL konfiguriert ist, prüft OpenClaw, ob `serviceUrl` der gespeicherten Unterhaltung denselben Host verwendet, bevor proaktives Senden, Bearbeiten, Löschen, Karten, Umfragen oder in die Warteschlange gestellte lang laufende Antworten ausgeführt werden. Bei der standardmäßigen Konfiguration für die öffentliche Cloud verweigert OpenClaw den Vorgang, wenn eine gespeicherte Unterhaltung auf einen Host außerhalb des öffentlichen Teams-Connector-Hosts verweist. Empfangen Sie nach einer Änderung der Cloud-/Dienst-URL-Einstellungen eine neue Nachricht aus der Unterhaltung, damit die gespeicherte Unterhaltungsreferenz aktuell ist.

China/21Vianet hat in Microsofts Teams-Tabelle für proaktive Endpunkte keine separate globale proaktive `smba`-URL. Konfigurieren Sie `cloud: "China"`, damit das Teams SDK die Azure-China-Endpunkte für Authentifizierung, Token und JWT verwendet. Proaktives Senden erfordert dann eine gespeicherte Unterhaltungsreferenz aus einer eingehenden China-Teams-Aktivität oder eine explizit konfigurierte Dienst-URL innerhalb der Azure-China-Begrenzung des Bot-Framework-Kanals (`*.botframework.azure.cn`). Graph-gestützte Teams-Hilfsfunktionen sind für `cloud: "China"` deaktiviert, bis OpenClaw Graph-Anfragen über den Azure-China-Graph-Endpunkt weiterleitet.

### Formatierung

Teams-Markdown ist stärker eingeschränkt als Slack oder Discord:

- Grundlegende Formatierung funktioniert: **fett**, _kursiv_, `code`, Links.
- Komplexes Markdown (Tabellen, verschachtelte Listen) wird möglicherweise nicht korrekt dargestellt.
- Adaptive Cards werden für Umfragen und semantische Präsentationssendungen unterstützt (siehe unten).

## Konfiguration

Wichtige Einstellungen (gemeinsame Kanalmuster finden Sie unter [/gateway/configuration](/de/gateway/configuration)):

- `channels.msteams.enabled`: den Kanal aktivieren/deaktivieren.
- `channels.msteams.appId`, `channels.msteams.appPassword`, `channels.msteams.tenantId`: Bot-Anmeldedaten.
- `channels.msteams.cloud`: Teams-SDK-Cloud-Umgebung (`Public`, `USGov`, `USGovDoD` oder `China`; Standardwert `Public`). Für USGov-/DoD-SDK-Clouds mit `serviceUrl` festlegen; China verwendet die SDK-Voreinstellung und gespeicherte Azure-China-Bot-Framework-Konversationsreferenzen, wobei Graph-basierte Hilfsfunktionen deaktiviert bleiben, bis Azure-China-Graph-Routing verfügbar ist.
- `channels.msteams.serviceUrl`: URL-Grenze des Bot-Connector-Dienstes für proaktive SDK-Vorgänge. Die öffentliche Cloud verwendet den SDK-Standardwert; für GCC (`https://smba.infra.gcc.teams.microsoft.com/teams`), GCC High oder DoD festlegen. China akzeptiert Azure-China-Bot-Framework-Kanalhosts, wenn die gespeicherte Konversationsreferenz aus dem von 21Vianet betriebenen Teams stammt.
- `channels.msteams.webhook.port` (Standardwert `3978`).
- `channels.msteams.webhook.path` (Standardwert `/api/messages`).
- `channels.msteams.dmPolicy`: `pairing | allowlist | open | disabled` (Standardwert `pairing`).
- `channels.msteams.allowFrom`: Zulassungsliste für Direktnachrichten (AAD-Objekt-IDs empfohlen). Der Assistent löst Namen während der Einrichtung in IDs auf, wenn Graph-Zugriff verfügbar ist.
- `channels.msteams.dangerouslyAllowNameMatching`: Notfall-Umschalter, um den veränderlichen Abgleich von UPN/Anzeigenamen und das direkte Routing über Team-/Kanalnamen wieder zu aktivieren.
- `channels.msteams.textChunkLimit`: Größe ausgehender Textabschnitte in Zeichen (Standardwert `4000`; unabhängig von einem höher konfigurierten Wert fest auf `4000` begrenzt).
- `channels.msteams.streaming.chunkMode`: `length` (Standardwert) oder `newline`, um vor der längenbasierten Aufteilung an Leerzeilen (Absatzgrenzen) zu trennen.
- `channels.msteams.mediaAllowHosts`: Zulassungsliste für Hosts eingehender Anhänge (standardmäßig Microsoft-/Teams-Domains: Graph, SharePoint/OneDrive, Teams CDN, Bot Framework, Azure Media Services).
- `channels.msteams.mediaAuthAllowHosts`: Zulassungsliste für das Anhängen von Authorization-Headern bei erneuten Medienabrufen (standardmäßig Graph- und Bot-Framework-Hosts).
- `channels.msteams.graphMediaFallback`: Graph-Nachrichtensuchen aktivieren, wenn Kanal-/Gruppen-HTML keine Dateimarkierungen enthält (Standardwert `false`; siehe [Wiederherstellung von Kanal-/Gruppendateien](#channelgroup-file-recovery-graphmediafallback)).
- `channels.msteams.mediaMaxMb`: kanalspezifische Überschreibung der Mediengrößenbegrenzung in MB. Fällt auf `agents.defaults.mediaMaxMb` zurück, wenn nicht festgelegt.
- `channels.msteams.requireMention`: @Erwähnung in Kanälen/Gruppen verlangen (Standardwert `true`).
- `channels.msteams.replyStyle`: `thread | top-level` (siehe [Antwortstil](#reply-style-threads-vs-posts)).
- `channels.msteams.teams.<teamId>.replyStyle`: teamspezifische Überschreibung.
- `channels.msteams.teams.<teamId>.requireMention`: teamspezifische Überschreibung.
- `channels.msteams.teams.<teamId>.tools`: standardmäßige teamspezifische Überschreibungen der Werkzeugrichtlinie (`allow`/`deny`/`alsoAllow`), die verwendet werden, wenn eine Kanalüberschreibung fehlt.
- `channels.msteams.teams.<teamId>.toolsBySender`: standardmäßige teamspezifische und absenderspezifische Überschreibungen der Werkzeugrichtlinie (Platzhalter `"*"` wird unterstützt).
- `channels.msteams.teams.<teamId>.channels.<conversationId>.replyStyle`: kanalspezifische Überschreibung.
- `channels.msteams.teams.<teamId>.channels.<conversationId>.requireMention`: kanalspezifische Überschreibung.
- `channels.msteams.teams.<teamId>.channels.<conversationId>.tools`: kanalspezifische Überschreibungen der Werkzeugrichtlinie (`allow`/`deny`/`alsoAllow`).
- `channels.msteams.teams.<teamId>.channels.<conversationId>.toolsBySender`: kanal- und absenderspezifische Überschreibungen der Werkzeugrichtlinie (Platzhalter `"*"` wird unterstützt).
- `toolsBySender`-Schlüssel sollten explizite Präfixe verwenden: `channel:`, `id:`, `e164:`, `username:`, `name:` (ältere Schlüssel ohne Präfix werden weiterhin nur `id:` zugeordnet).
- `channels.msteams.authType`: Authentifizierungstyp – `"secret"` (Standardwert) oder `"federated"`.
- `channels.msteams.certificatePath`: Pfad zur PEM-Zertifikatsdatei (föderierte Authentifizierung und Zertifikatsauthentifizierung).
- `channels.msteams.certificateThumbprint`: Zertifikatfingerabdruck; wird akzeptiert, ist für die Authentifizierung jedoch nicht erforderlich.
- `channels.msteams.useManagedIdentity`: Authentifizierung mit verwalteter Identität aktivieren (föderierter Modus).
- `channels.msteams.managedIdentityClientId`: Client-ID für eine benutzerseitig zugewiesene verwaltete Identität.
- `channels.msteams.sharePointSiteId`: SharePoint-Website-ID für Datei-Uploads in Gruppenchats/Kanälen (siehe [Dateien in Gruppenchats senden](#sending-files-in-group-chats)).
- `channels.msteams.welcomeCard`, `channels.msteams.groupWelcomeCard`, `channels.msteams.promptStarters`: Adaptive Card zur Begrüßung, die beim ersten Direktnachrichten-/Gruppenkontakt angezeigt wird, sowie deren Schaltflächen mit vorgeschlagenen Prompts.
- `channels.msteams.responsePrefix`: Text, der ausgehenden Antworten vorangestellt wird.
- `channels.msteams.feedbackEnabled` (Standardwert `true`), `channels.msteams.feedbackReflection` (Standardwert `true`), `channels.msteams.feedbackReflectionCooldownMs`: Feedback per Daumen hoch/runter zu Antworten und anschließende Reflexion bei negativem Feedback.
- `channels.msteams.sso`, `channels.msteams.delegatedAuth`: Bot-Framework-OAuth-Verbindung und delegierte Graph-Bereiche für SSO-basierte Abläufe; `sso.enabled: true` erfordert `sso.connectionName`.

## Routing und Sitzungen

- Sitzungsschlüssel folgen dem standardmäßigen Agentenformat (siehe [/concepts/session](/de/concepts/session)):
  - Direktnachrichten verwenden gemeinsam die Hauptsitzung (`agent:<agentId>:<mainKey>`).
  - Kanal-/Gruppennachrichten verwenden die Konversations-ID:
    - `agent:<agentId>:msteams:channel:<conversationId>`
    - `agent:<agentId>:msteams:group:<conversationId>`

## Antwortstil: Threads oder Beiträge

Teams bietet für dasselbe zugrunde liegende Datenmodell zwei Kanaloberflächen:

| Stil                     | Beschreibung                                                | Empfohlenes `replyStyle` |
| ------------------------ | ----------------------------------------------------------- | ------------------------ |
| **Beiträge** (klassisch) | Nachrichten erscheinen als Karten mit darunter angeordneten Antworten in Threads | `thread` (Standardwert) |
| **Threads** (Slack-ähnlich) | Nachrichten verlaufen linear, ähnlich wie in Slack       | `top-level`              |

**Das Problem:** Die Teams-API gibt nicht an, welche Oberfläche ein Kanal verwendet. Wenn Sie das falsche `replyStyle` verwenden:

- `thread` in einem Kanal mit Threads-Oberfläche → Antworten erscheinen unübersichtlich verschachtelt.
- `top-level` in einem Kanal mit Beitragsoberfläche → Antworten erscheinen als separate Beiträge auf oberster Ebene statt innerhalb des Threads.

**Lösung:** Konfigurieren Sie `replyStyle` für jeden Kanal entsprechend seiner Einrichtung:

```json5
{
  channels: {
    msteams: {
      replyStyle: "thread",
      teams: {
        "19:abc...@thread.tacv2": {
          channels: {
            "19:xyz...@thread.tacv2": {
              replyStyle: "top-level",
            },
          },
        },
      },
    },
  },
}
```

### Auflösungsreihenfolge

Wenn der Bot eine Antwort in einen Kanal sendet, wird `replyStyle` von der spezifischsten Überschreibung bis zum Standardwert aufgelöst. Der erste Wert, der nicht `undefined` ist, wird verwendet:

1. **Pro Kanal** – `channels.msteams.teams.<teamId>.channels.<conversationId>.replyStyle`
2. **Pro Team** – `channels.msteams.teams.<teamId>.replyStyle`
3. **Global** – `channels.msteams.replyStyle`
4. **Impliziter Standardwert** – abgeleitet aus `requireMention`:
   - `requireMention: true` → `thread`
   - `requireMention: false` → `top-level`

Wenn Sie `requireMention: false` global ohne ein explizites `replyStyle` festlegen, erscheinen Erwähnungen in Kanälen mit Beitragsoberfläche als Beiträge auf oberster Ebene, selbst wenn die eingehende Nachricht eine Thread-Antwort war. Legen Sie `replyStyle: "thread"` auf globaler, Team- oder Kanalebene fest, um Überraschungen zu vermeiden.

Für proaktive Sendevorgänge in eine gespeicherte Kanalkonversation (Antworten auf Werkzeugaufrufe in der Warteschlange, lang laufende Agenten) gilt dieselbe Team-/Kanalauflösung; Gruppenchats und persönliche Konversationen (Direktnachrichten) werden bei proaktiven Sendevorgängen unabhängig von `replyStyle` immer zu `top-level` aufgelöst.

### Beibehaltung des Thread-Kontexts

Wenn `replyStyle: "thread"` gilt und der Bot innerhalb eines Kanal-Threads per @Erwähnung angesprochen wurde, hängt OpenClaw den ursprünglichen Thread-Ausgangspunkt wieder an die ausgehende Konversationsreferenz (`19:...@thread.tacv2;messageid=<root>`) an, damit die Antwort im selben Thread landet. Dies gilt sowohl für direkte Sendevorgänge (innerhalb des aktuellen Durchlaufs) als auch für proaktive Sendevorgänge, nachdem der Bot-Framework-Durchlaufkontext abgelaufen ist (z. B. lang laufende Agenten oder Antworten auf Werkzeugaufrufe in der Warteschlange über `mcp__openclaw__message`).

Der Thread-Ausgangspunkt wird dem gespeicherten `threadId` der Konversationsreferenz entnommen. Ältere gespeicherte Referenzen, die vor `threadId` erstellt wurden, fallen auf `activityId` zurück (die eingehende Aktivität, die die Konversation zuletzt initialisiert hat), sodass bestehende Bereitstellungen ohne erneute Initialisierung weiter funktionieren.

Wenn `replyStyle: "top-level"` gilt, werden eingehende Kanal-Thread-Nachrichten absichtlich als neue Beiträge auf oberster Ebene beantwortet; es wird kein Thread-Suffix angehängt. Dies ist für Kanäle mit Threads-Oberfläche korrekt. Wenn Beiträge auf oberster Ebene erscheinen, obwohl Sie Thread-Antworten erwartet haben, ist `replyStyle` für diesen Kanal falsch festgelegt.

## Anhänge und Bilder

**Aktuelle Einschränkungen:**

- **Direktnachrichten:** Bilder und Dateianhänge funktionieren über die Teams-Bot-Datei-APIs.
- **Kanäle/Gruppen:** Anhänge befinden sich im M365-Speicher (SharePoint/OneDrive). Die Webhook-Nutzlast enthält nur ein HTML-Fragment, nicht die tatsächlichen Dateibytes. **Zum Herunterladen von Kanalanhängen sind Graph-API-Berechtigungen erforderlich.**
- Verwenden Sie für explizite dateiorientierte Sendevorgänge `action=upload-file` mit `media` / `filePath` / `path`; das optionale `message` wird zum begleitenden Text/Kommentar und `filename` (oder `title`) überschreibt den Namen der hochgeladenen Datei.

Ohne Graph-Berechtigungen gehen Kanalnachrichten mit Bildern nur als Text ein (der Bot kann nicht auf den Bildinhalt zugreifen).
Standardmäßig lädt OpenClaw Medien nur von Microsoft-/Teams-Hostnamen herunter. Dies kann mit `channels.msteams.mediaAllowHosts` überschrieben werden (verwenden Sie `["*"]`, um jeden Host zuzulassen).
Authorization-Header werden nur für Hosts in `channels.msteams.mediaAuthAllowHosts` angehängt (standardmäßig Graph- und Bot-Framework-Hosts). Halten Sie diese Liste strikt (vermeiden Sie mandantenübergreifende Suffixe).

## Dateien in Gruppenchats senden

Bots können Dateien in Direktnachrichten über den integrierten FileConsentCard-Ablauf senden. **Das Senden von Dateien in Gruppenchats/Kanälen** erfordert eine zusätzliche Einrichtung:

| Kontext                  | So werden Dateien gesendet                   | Erforderliche Einrichtung                       |
| ------------------------ | -------------------------------------------- | ----------------------------------------------- |
| **Direktnachrichten**    | FileConsentCard → Benutzer akzeptiert → Bot lädt hoch | Funktioniert ohne zusätzliche Einrichtung |
| **Gruppenchats/Kanäle**  | Upload zu SharePoint → native Dateikarte     | Erfordert `sharePointSiteId` und Graph-Berechtigungen |
| **Bilder (jeder Kontext)** | Base64-kodiert und eingebettet              | Funktioniert ohne zusätzliche Einrichtung      |

### Warum Gruppenchats SharePoint benötigen

Bots verwenden eine Anwendungsidentität, während die `/me`-Ressource von Microsoft Graph [einen angemeldeten Benutzer erfordert](https://learn.microsoft.com/en-us/graph/api/user-get?view=graph-rest-1.0). Um Dateien in Gruppenchats/Kanälen zu senden, lädt der Bot sie auf eine **SharePoint-Website** hoch und erstellt einen Freigabelink.

### Einrichtung

1. **Fügen Sie Graph-API-Berechtigungen** unter Entra ID (Azure AD) → App Registration hinzu:
   - `Sites.ReadWrite.All` (Application) – Dateien zu SharePoint hochladen.
   - `ChatMember.Read.All` (Application) – mandantenweite Berechtigung mit den geringsten Rechten für das Senden von Dateien in Gruppenchats. `Chat.Read.All` funktioniert ebenfalls und deckt dies bereits ab, wenn der Gruppenchatverlauf aktiviert ist. Verwenden Sie als chatbezogene Alternative die [ressourcenspezifische Einwilligungsberechtigung](https://learn.microsoft.com/en-us/microsoftteams/platform/graph-api/rsc/resource-specific-consent) `ChatMember.Read.Chat`.
2. **Erteilen Sie die Administratoreinwilligung** für den Mandanten.
3. **Rufen Sie Ihre SharePoint-Website-ID ab:**

   ```bash
   # Über Graph Explorer oder curl mit einem gültigen Token:
   curl -H "Authorization: Bearer $TOKEN" \
     "https://graph.microsoft.com/v1.0/sites/{hostname}:/{site-path}"

   # Beispiel: für eine Website unter "contoso.sharepoint.com/sites/BotFiles"
   curl -H "Authorization: Bearer $TOKEN" \
     "https://graph.microsoft.com/v1.0/sites/contoso.sharepoint.com:/sites/BotFiles"

   # Die Antwort enthält: "id": "contoso.sharepoint.com,guid1,guid2"
   ```

4. **OpenClaw konfigurieren:**

   ```json5
   {
     channels: {
       msteams: {
         // ... weitere Konfiguration ...
         sharePointSiteId: "contoso.sharepoint.com,guid1,guid2",
       },
     },
   }
   ```

### Freigabeverhalten

| Kontext und Berechtigung                                                | Freigabeverhalten                                               |
| ----------------------------------------------------------------------- | --------------------------------------------------------------- |
| Kanal + `Sites.ReadWrite.All`                                         | Organisationsweiter Freigabelink (alle in der Organisation haben Zugriff) |
| Gruppenchat + `Sites.ReadWrite.All` + eine unterstützte Leseberechtigung für Chatmitglieder | Benutzerspezifischer Freigabelink (nur Chatmitglieder haben Zugriff) |
| Gruppenchat ohne unterstützte Leseberechtigung für Chatmitglieder       | Senden schlägt sicher geschlossen fehl                          |

Die benutzerspezifische Freigabe ist sicherer, da nur Chatteilnehmende auf die Datei zugreifen können. OpenClaw erfordert für Gruppenchats eine erfolgreiche Mitgliedersuche; Zeitüberschreitungen, Transportfehler, leere Ergebnisse und Ablehnungen durch die Graph API führen dazu, dass das Senden fehlschlägt, statt den Zugriff auf die Organisation auszuweiten.

### Fallback-Verhalten

| Szenario                                                         | Ergebnis                                           |
| ---------------------------------------------------------------- | -------------------------------------------------- |
| Gruppenchat + Datei + SharePoint- und Mitgliederberechtigungen konfiguriert | In SharePoint hochladen, native Dateikarte senden |
| Gruppenchat + Datei + fehlende SharePoint- oder Mitgliederberechtigungen | Mit einem umsetzbaren Konfigurationsfehler fehlschlagen |
| Kanal + Datei + `sharePointSiteId` konfiguriert                   | In SharePoint hochladen, native Dateikarte senden |
| Persönlicher Chat + Datei                                        | FileConsentCard-Ablauf (funktioniert ohne SharePoint) |
| Beliebiger Kontext + Bild                                        | Base64-codiert inline (funktioniert ohne SharePoint) |

### Speicherort der Dateien

Hochgeladene Dateien werden in einem Ordner `/OpenClawShared/` in der standardmäßigen Dokumentbibliothek der konfigurierten SharePoint-Website gespeichert.

## Umfragen (Adaptive Cards)

OpenClaw sendet Teams-Umfragen als Adaptive Cards (es gibt keine native Teams-Umfrage-API).

- CLI: `openclaw message poll --channel msteams --target conversation:<id> --poll-question "..." --poll-option "..." --poll-option "..."`.
- Stimmen werden vom Gateway im SQLite-Plugin-Status von OpenClaw unter `state/openclaw.sqlite` gespeichert.
- Vorhandene `msteams-polls.json`-Dateien werden durch `openclaw doctor --fix` importiert, nicht durch das laufende Plugin.
- Das Gateway muss online bleiben, um Stimmen zu erfassen.
- Umfragen veröffentlichen nicht automatisch Ergebniszusammenfassungen, und es gibt noch keine CLI für Umfrageergebnisse.

## Präsentationskarten

Senden Sie semantische Präsentationsnutzlasten mit dem Tool `message`, der CLI oder der normalen Antwortzustellung an Teams-Benutzer oder -Unterhaltungen. OpenClaw rendert sie gemäß dem generischen Präsentationsvertrag als Teams Adaptive Cards.

Der Parameter `presentation` akzeptiert semantische Blöcke. Wenn `presentation` angegeben ist, ist der Nachrichtentext optional. Schaltflächen werden als Adaptive-Card-Übermittlungs- oder URL-Aktionen gerendert. Auswahlmenüs sind im Teams-Renderer nicht nativ verfügbar, daher stuft OpenClaw sie vor der Zustellung zu lesbarem Text herab.

**Agent-Tool:**

```json5
{
  action: "send",
  channel: "msteams",
  target: "user:<id>",
  presentation: {
    title: "Hallo",
    blocks: [{ type: "text", text: "Hallo!" }],
  },
}
```

**CLI:**

```bash
openclaw message send --channel msteams \
  --target "conversation:19:abc...@thread.tacv2" \
  --presentation '{"title":"Hallo","blocks":[{"type":"text","text":"Hallo!"}]}'
```

Details zu Zielformaten finden Sie weiter unten unter [Zielformate](#target-formats).

## Zielformate

MSTeams-Ziele verwenden Präfixe, um zwischen Benutzern und Unterhaltungen zu unterscheiden:

| Zieltyp             | Format                           | Beispiel                                                                                               |
| ------------------- | -------------------------------- | ------------------------------------------------------------------------------------------------------ |
| Benutzer (nach ID)  | `user:<aad-object-id>`           | `user:40a1a0ed-4ff2-4164-a219-55518990c197`                                                            |
| Benutzer (nach Name) | `user:<display-name>`            | `user:John Smith` (erfordert Graph API)                                                                 |
| Gruppe/Kanal        | `conversation:<conversation-id>` | `conversation:19:abc123...@thread.tacv2`                                                               |
| Gruppe/Kanal (roh)  | `<conversation-id>`              | `19:abc123...@thread.tacv2`, `19:...@unq.gbl.spaces` oder eine reine `a:`-/`8:orgid:`-/`29:`-Bot-Framework-ID |

**CLI-Beispiele:**

```bash
# An einen Benutzer nach ID senden
openclaw message send --channel msteams --target "user:40a1a0ed-..." --message "Hallo"

# An einen Benutzer nach Anzeigename senden (löst eine Graph-API-Suche aus)
openclaw message send --channel msteams --target "user:John Smith" --message "Hallo"

# An einen Gruppenchat oder Kanal senden
openclaw message send --channel msteams --target "conversation:19:abc...@thread.tacv2" --message "Hallo"

# Eine Präsentationskarte an eine Unterhaltung senden
openclaw message send --channel msteams --target "conversation:19:abc...@thread.tacv2" \
  --presentation '{"title":"Hallo","blocks":[{"type":"text","text":"Hallo"}]}'
```

**Beispiele für das Agent-Tool:**

```json5
{
  action: "send",
  channel: "msteams",
  target: "user:John Smith",
  message: "Hallo!",
}
```

```json5
{
  action: "send",
  channel: "msteams",
  target: "conversation:19:abc...@thread.tacv2",
  presentation: {
    title: "Hallo",
    blocks: [{ type: "text", text: "Hallo" }],
  },
}
```

<Note>
Ohne das Präfix `user:` werden Namen standardmäßig als Gruppe oder Team aufgelöst. Verwenden Sie immer `user:`, wenn Sie Personen anhand ihres Anzeigenamens adressieren.
</Note>

## Proaktive Nachrichten

- Proaktive Nachrichten sind nur möglich, **nachdem** ein Benutzer interagiert hat, da OpenClaw zu diesem Zeitpunkt Unterhaltungsreferenzen speichert.
- Informationen zu `dmPolicy` und zur Zulassungslistensteuerung finden Sie unter [/gateway/configuration](/de/gateway/configuration).

## Team- und Kanal-IDs (häufiger Stolperstein)

Der Abfrageparameter `groupId` in Teams-URLs ist **NICHT** die für die Konfiguration verwendete Team-ID. Extrahieren Sie die IDs stattdessen aus dem URL-Pfad:

**Team-URL:**

```text
https://teams.microsoft.com/l/team/19%3ABk4j...%40thread.tacv2/conversations?groupId=...
                                    └────────────────────────────┘
                                    Team-Unterhaltungs-ID (URL-decodieren)
```

**Kanal-URL:**

```text
https://teams.microsoft.com/l/channel/19%3A15bc...%40thread.tacv2/ChannelName?groupId=...
                                      └─────────────────────────┘
                                      Kanal-ID (URL-decodieren)
```

**Für die Konfiguration:**

- Team-Schlüssel = Pfadsegment nach `/team/` (URL-decodiert, z. B. `19:Bk4j...@thread.tacv2`; ältere Mandanten zeigen möglicherweise `@thread.skype`, was ebenfalls gültig ist).
- Kanal-Schlüssel = Pfadsegment nach `/channel/` (URL-decodiert).
- **Ignorieren Sie** den Abfrageparameter `groupId` für das OpenClaw-Routing. Dabei handelt es sich um die Microsoft-Entra-Gruppen-ID, nicht um die Bot-Framework-Unterhaltungs-ID, die in eingehenden Teams-Aktivitäten verwendet wird.

## Private Kanäle

Bots werden in privaten Kanälen nur eingeschränkt unterstützt:

| Funktion                     | Standardkanäle    | Private Kanäle               |
| ---------------------------- | ----------------- | ----------------------------- |
| Bot-Installation             | Ja                | Eingeschränkt                 |
| Echtzeitnachrichten (Webhook) | Ja               | Funktionieren möglicherweise nicht |
| RSC-Berechtigungen           | Ja                | Können sich anders verhalten  |
| @Erwähnungen                 | Ja                | Wenn der Bot erreichbar ist   |
| Graph-API-Verlauf            | Ja                | Ja (mit Berechtigungen)        |

**Problemumgehungen, wenn private Kanäle nicht funktionieren:**

1. Verwenden Sie Standardkanäle für Bot-Interaktionen.
2. Verwenden Sie Direktnachrichten; Benutzer können dem Bot jederzeit direkt schreiben.
3. Verwenden Sie die Graph API für den historischen Zugriff (erfordert `ChannelMessage.Read.All`).

## Fehlerbehebung

### Häufige Probleme

- **Bilder werden in Kanälen nicht angezeigt:** Graph-Berechtigungen oder Administratoreinwilligung fehlen. Installieren Sie die Teams-App neu, beenden Sie Teams vollständig und öffnen Sie es erneut.
- **Keine Antworten im Kanal:** Erwähnungen sind standardmäßig erforderlich; legen Sie `channels.msteams.requireMention=false` fest oder konfigurieren Sie dies je Team/Kanal.
- **Versionsabweichung (Teams zeigt weiterhin das alte Manifest):** Entfernen Sie die App, fügen Sie sie erneut hinzu, beenden Sie Teams vollständig und öffnen Sie es zur Aktualisierung erneut.
- **401 Unauthorized vom Webhook:** Wird beim manuellen Testen ohne Azure-JWT erwartet; dies bedeutet, dass der Endpunkt erreichbar ist, die Authentifizierung jedoch fehlgeschlagen ist. Verwenden Sie Azure Web Chat für einen ordnungsgemäßen Test.

### Fehler beim Hochladen des Manifests

- **"Icon file cannot be empty":** Das Manifest verweist auf Symboldateien mit einer Größe von 0 Byte. Erstellen Sie gültige PNG-Symbole (32x32 für `outline.png`, 192x192 für `color.png`).
- **"webApplicationInfo.Id already in use":** Die App ist noch in einem anderen Team/Chat installiert. Suchen und deinstallieren Sie sie zuerst oder warten Sie 5-10 Minuten auf die Übernahme der Änderung.
- **"Something went wrong" beim Hochladen:** Laden Sie sie stattdessen über [https://admin.teams.microsoft.com](https://admin.teams.microsoft.com) hoch, öffnen Sie die Browser-Entwicklertools (F12) → Registerkarte Network und prüfen Sie den Antworttext auf den tatsächlichen Fehler.
- **Querladen schlägt fehl:** Versuchen Sie "Upload an app to your org's app catalog" anstelle von "Upload a custom app"; dadurch werden Einschränkungen für das Querladen häufig umgangen.

### RSC-Berechtigungen funktionieren nicht

1. Überprüfen Sie, ob `webApplicationInfo.id` exakt mit der App-ID Ihres Bots übereinstimmt.
2. Laden Sie die App erneut hoch und installieren Sie sie im Team/Chat neu.
3. Prüfen Sie, ob der Administrator Ihrer Organisation RSC-Berechtigungen blockiert hat.
4. Vergewissern Sie sich, dass Sie den richtigen Bereich verwenden: `ChannelMessage.Read.Group` für Teams, `ChatMessage.Read.Chat` für Gruppenchats.

## Referenzen

- [Azure Bot erstellen](https://learn.microsoft.com/en-us/azure/bot-service/bot-service-quickstart-registration) - Einrichtungsanleitung für Azure Bot
- [Teams Developer Portal](https://dev.teams.microsoft.com/apps) - Teams-Apps erstellen/verwalten
- [Schema des Teams-App-Manifests](https://learn.microsoft.com/en-us/microsoftteams/platform/resources/schema/manifest-schema)
- [Kanalnachrichten mit RSC empfangen](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/conversations/channel-messages-with-rsc)
- [Referenz zu RSC-Berechtigungen](https://learn.microsoft.com/en-us/microsoftteams/platform/graph-api/rsc/resource-specific-consent)
- [Dateiverarbeitung für Teams-Bots](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/bots-filesv4) (Kanal/Gruppe erfordert Graph)
- [Proaktive Nachrichten](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/conversations/send-proactive-messages)
- [@microsoft/teams.cli](https://www.npmjs.com/package/@microsoft/teams.cli) - Teams-CLI zur Bot-Verwaltung

## Verwandte Themen

- [Kanalübersicht](/de/channels) - alle unterstützten Kanäle
- [Kopplung](/de/channels/pairing) - DM-Authentifizierung und Kopplungsablauf
- [Gruppen](/de/channels/groups) - Verhalten von Gruppenchats und erwähnungsbasierte Freigabe
- [Kanal-Routing](/de/channels/channel-routing) - Sitzungs-Routing für Nachrichten
- [Sicherheit](/de/gateway/security) - Zugriffsmodell und Absicherung
