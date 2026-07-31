---
read_when:
    - De macOS-onboardingassistent ontwerpen
    - Authenticatie- of identiteitsconfiguratie implementeren
sidebarTitle: 'Onboarding: macOS App'
summary: Installatieproces bij de eerste uitvoering voor OpenClaw (macOS-app)
title: Onboarding (macOS-app)
x-i18n:
    generated_at: "2026-07-27T06:13:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 55154774886c530de92b2110d367af24e2142fac48b901f288582d8552a6ca10
    source_path: start/onboarding.md
    workflow: 16
---

De eerste configuratie van de macOS-app: kies waar de Gateway draait, verbind een
geverifieerde AI-backend, verleen machtigingen en draag het over aan het eigen
bootstrapritueel van de agent.
Zie [Overzicht van onboarding](/nl/start/onboarding-overview) voor onboarding via de CLI en een vergelijking van beide trajecten.

<Steps>
<Step title="macOS-waarschuwing goedkeuren">
<Frame>
<img src="/assets/macos-onboarding/01-macos-warning.jpeg" alt="" />
</Frame>
</Step>
<Step title="Lokale netwerken zoeken goedkeuren">
<Frame>
<img src="/assets/macos-onboarding/02-local-networks.jpeg" alt="" />
</Frame>
</Step>
<Step title="Welkomstbericht en beveiligingsmelding">
<Frame caption="Lees de weergegeven beveiligingsmelding en beslis op basis daarvan">
<img src="/assets/macos-onboarding/03-security-notice.png" alt="" />
</Frame>

Vertrouwensmodel voor beveiliging:

- OpenClaw is standaard een persoonlijke agent: één grens met één vertrouwde beheerder.
- Gedeelde configuraties en configuraties voor meerdere gebruikers moeten worden afgeschermd: scheid vertrouwensgrenzen, houd toegang tot tools minimaal en volg [Beveiliging](/nl/gateway/security).
- Bij lokale onboarding gebruiken nieuwe configuraties standaard `tools.profile: "coding"`, zodat nieuwe installaties tools voor het bestandssysteem en de runtime behouden zonder het onbeperkte profiel `full`.
- Als hooks/webhooks of andere invoerbronnen met niet-vertrouwde inhoud zijn ingeschakeld, gebruik dan een krachtig modern modelniveau en handhaaf een strikt toolbeleid en strikte sandboxing.

</Step>
<Step title="Lokaal versus extern">
<Frame>
<img src="/assets/macos-onboarding/04-choose-gateway.png" alt="" />
</Frame>

Waar draait de **Gateway**?

- **Deze Mac (alleen lokaal):** onboarding configureert authenticatie en schrijft aanmeldgegevens lokaal weg.
- **Extern (via SSH/Tailnet):** onboarding configureert **geen** lokale authenticatie;
  de aanmeldgegevens moeten al op de Gateway-host aanwezig zijn. In het veld voor
  het externe Gateway-token wordt het token opgeslagen waarmee de macOS-app
  verbinding maakt met die Gateway; bestaande `gateway.remote.token` SecretRef-waarden
  blijven behouden totdat je ze vervangt.
- **Later configureren:** sla de configuratie over en laat de app ongeconfigureerd.

<Tip>
**Tip voor Gateway-authenticatie:**

- De authenticatiemodus van de Gateway is standaard `token`, zelfs voor loopback-bindingen. Lokale WS-clients moeten zich dus authenticeren.
- Met `gateway.auth.mode: "none"` kan elk lokaal proces verbinding maken; gebruik dit alleen op volledig vertrouwde machines.
- Gebruik een token voor toegang vanaf meerdere machines of voor bindingen die niet aan loopback zijn gekoppeld.

</Tip>
</Step>
<Step title="CLI">
  Bij lokale configuratie wordt de algemene CLI `openclaw` geïnstalleerd via npm, pnpm of bun,
  waarbij eerst de voorkeur uitgaat naar npm. Node blijft de aanbevolen runtime voor de
  Gateway zelf. Bestaande compatibele installaties worden hergebruikt.
</Step>
<Step title="Verbind je AI">
  Een verbonden Gateway waarop al een agentmodel is geconfigureerd, slaat deze
  pagina volledig over en opent de normale agentinterface. De configuratie van
  OpenClaw en providers wordt alleen uitgevoerd voor een nieuwe of onvolledig
  geconfigureerde Gateway.

Zodra de Gateway gereed is, zoekt onboarding naar AI-toegang waarover je al beschikt:
een aanmelding bij Claude Code of Codex, `OPENAI_API_KEY` / `ANTHROPIC_API_KEY`, of een
model met toolondersteuning en ten minste 16K aan gemeten effectieve context dat al
is geïnstalleerd op een bereikbare Ollama- of LM Studio-server. De detectie wordt
uitgevoerd op de Gateway-host, ook wanneer de macOS-app verbinding maakt met een
Linux-Gateway. De beste optie wordt met een echte voltooiing getest en pas opgeslagen
nadat deze antwoordt. Wanneer een test mislukt, probeert de app automatisch de volgende
optie en toont deze waarom de vorige is mislukt. Als meerdere opties worden gevonden,
kun je ertussen wisselen voordat je doorgaat. Bij automatische lokale detectie wordt
nooit een model opgehaald of gedownload.

Als je een Claude-abonnement wilt gebruiken wanneer de Gateway-host geen aanmelding
voor de Claude CLI heeft, voer je `claude setup-token` uit op een machine waarop Claude
Code is geïnstalleerd. Plak vervolgens het weergegeven token als **Anthropic setup-token**
onder **Connect with an API key or token**.

Geïnstalleerde CLI's voor Gemini CLI, Antigravity, Pi en OpenCode worden ter context
weergegeven wanneer ze niet kunnen worden geselecteerd als herbruikbare inferentieroute
voor begeleide configuratie. Gemini en Antigravity kunnen de inferentieproef zonder
tools niet afdwingen. Pi en OpenCode zijn volledige agentframeworks in plaats van
inferentieroutes voor configuratie; hun sessie-integraties vereisen afzonderlijke
runtime- en Plugin-configuratie.

Je kunt je ook aanmelden via de eigen OAuth- of apparaatkoppelingsprocedure van de
provider. De ingebouwde keuzes omvatten OpenAI/ChatGPT, OpenRouter, GitHub Copilot,
Google Gemini CLI, xAI, MiniMax Global en CN, en Chutes. De lijst is afkomstig van de
actieve plugins voor tekstinferentieproviders van de Gateway en niet van een vaste
applijst. Daardoor kan een andere provider deelnemen zonder providerspecifieke
macOS-code toe te voegen.

De handmatige kiezer voor sleutels en tokens gebruikt hetzelfde providerregister.
Bij elke route levert de provider het startmodel en de configuratie; OpenClaw
verifieert de aanmeldgegevens met dezelfde livetest voordat het authenticatieprofiel
wordt opgeslagen. Volgende blijft vergrendeld totdat één backend is geslaagd, zodat
de eerste agentchat niet zonder werkende inferentie kan beginnen. Nadat die livecontrole
is geslaagd, kan OpenClaw helpen om de resterende werkruimte, Gateway, kanalen en andere
optionele functies te configureren. Wanneer OpenClaw een korte lijst met keuzes aanbiedt,
toont de app systeemeigen optiekaarten. Als je er een kiest, wordt de selectie verzonden,
en met **Skip for now** blijft de keuze altijd optioneel. OpenClaw is later ook beschikbaar
onder Settings → OpenClaw.
</Step>
<Step title="Herinneringen importeren (weergegeven wanneer gedetecteerd)">
Voor een lokale Gateway controleert onboarding de Mac op herinneringen van ondersteunde
AI-tools: automatisch geheugen van Claude Code, samengevoegde herinneringen van Codex
en Hermes-geheugenbestanden. Wanneer er herinneringen worden gevonden, toont deze
pagina elke bron met het aantal herinneringen en kun je de geselecteerde bronnen
importeren in de agentwerkruimte onder `memory/imports/` voor geïndexeerd terugzoeken.
Bestanden die al zijn geïmporteerd, worden overgeslagen. De pagina wordt nooit
weergegeven wanneer er niets te importeren is. Overslaan is veilig; op de pagina voor
geheugenimport van het dashboard kun je dezelfde import later per bestand beheren.
</Step>
<Step title="Machtigingen">

<Frame caption="Kies welke machtigingen je aan OpenClaw wilt verlenen">
<img src="/assets/macos-onboarding/05-permissions.png" alt="" />
</Frame>

Onboarding vraagt TCC-machtigingen aan voor: automatisering (AppleScript), meldingen, toegankelijkheid, schermopname, microfoon, spraakherkenning, camera en locatie.

</Step>
<Step title="Voltooien">
  Nadat de inferentie is geslaagd, beheert OpenClaw de resterende optionele
  configuratie en kan het je doorsturen naar de normale agentchat. Na voltooiing
  van de rondleiding door de machtigingen wordt diezelfde chat geopend; de app
  maakt vóór OpenClaw geen werkruimte en start geen afzonderlijk
  configuratiegesprek voor de agent. Zie
  [Bootstrapping](/nl/start/bootstrapping) voor wat er op de Gateway-host gebeurt
  tijdens de eerste echte beurt van de agent.
</Step>
</Steps>

## Gerelateerd

- [Overzicht van onboarding](/nl/start/onboarding-overview)
- [Aan de slag](/nl/start/getting-started)
