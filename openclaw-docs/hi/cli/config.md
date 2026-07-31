---
read_when:
    - आप कॉन्फ़िगरेशन को गैर-संवादात्मक रूप से पढ़ना या संपादित करना चाहते हैं
sidebarTitle: Config
summary: '`openclaw config` के लिए CLI संदर्भ (get/set/patch/unset/file/schema/validate)'
title: कॉन्फ़िगरेशन
x-i18n:
    generated_at: "2026-07-27T19:26:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4c4f8edb19737070e421c9107f7da8886e5617d9a043d8647666505c7ac9638d
    source_path: cli/config.md
    workflow: 16
---

`openclaw.json` के लिए गैर-इंटरैक्टिव सहायक: पथ के अनुसार कोई मान प्राप्त/सेट/पैच/अनसेट करें, स्कीमा प्रिंट करें, सत्यापन करें, या सक्रिय फ़ाइल पथ प्रिंट करें। `openclaw configure` जैसा ही निर्देशित विज़ार्ड खोलने के लिए `openclaw config` को बिना किसी उपकमांड के चलाएँ।

<Note>
जब `OPENCLAW_NIX_MODE=1`, तब OpenClaw `openclaw.json` को अपरिवर्तनीय मानता है। केवल-पढ़ने वाले कमांड (`config get`, `config file`, `config schema`, `config validate`) फिर भी काम करते हैं; कॉन्फ़िग लेखक इनकार कर देते हैं। इसके बजाय इंस्टॉल के लिए Nix स्रोत संपादित करें; प्रथम-पक्ष nix-openclaw वितरण के लिए, [nix-openclaw त्वरित शुरुआत](https://github.com/openclaw/nix-openclaw#quick-start) का उपयोग करें और मानों को `programs.openclaw.config` या `instances.<name>.config` के अंतर्गत सेट करें।
</Note>

## रूट विकल्प

<ParamField path="--section <section>" type="string">
  जब आप `openclaw config` को बिना किसी उपकमांड के चलाते हैं, तब दोहराया जा सकने वाला निर्देशित-सेटअप अनुभाग फ़िल्टर।
</ParamField>

निर्देशित अनुभाग: `workspace`, `model`, `web`, `gateway`, `daemon`, `channels`, `plugins`, `skills`, `health`।

## उदाहरण

```bash
openclaw config file
openclaw config --section model
openclaw config --section gateway --section daemon
openclaw config schema
openclaw config get browser.executablePath
openclaw config set browser.executablePath "/usr/bin/google-chrome"
openclaw config set browser.profiles.work.executablePath "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
openclaw config set agents.defaults.heartbeat.every "2h"
openclaw config set 'agents.entries.main.tools.exec.node' "node-id-or-name"
openclaw config set agents.defaults.models '{"openai/gpt-5.4":{}}' --strict-json --merge
openclaw config set channels.discord.token --ref-provider default --ref-source env --ref-id DISCORD_BOT_TOKEN
openclaw config set secrets.providers.vaultfile --provider-source file --provider-path /etc/openclaw/secrets.json --provider-mode json
openclaw config patch --file ./openclaw.patch.json5 --dry-run
openclaw config unset plugins.entries.brave.config.webSearch.apiKey
openclaw config set channels.discord.token --ref-provider default --ref-source env --ref-id DISCORD_BOT_TOKEN --dry-run
openclaw config validate
openclaw config validate --json
```

### पथ

डॉट या ब्रैकेट संकेतन। शेल उदाहरणों में ब्रैकेट पथों को उद्धरण चिह्नों में रखें, ताकि zsh `[0]` को ग्लोब-विस्तारित न करे:

```bash
openclaw config get agents.defaults.workspace
openclaw config get agents.entries.main
openclaw config get agents.entries
openclaw config set 'agents.entries.work.tools.exec.node' "node-id-or-name"
```

### `config get`

संशोधित कॉन्फ़िग स्नैपशॉट से कोई मान पढ़ता है (गोपनीय मान कभी प्रिंट नहीं होते)। `--json` मूल मान को JSON के रूप में प्रिंट करता है; अन्यथा स्ट्रिंग/संख्या/बूलियन बिना आवरण के प्रिंट होते हैं और ऑब्जेक्ट/ऐरे स्वरूपित JSON के रूप में प्रिंट होते हैं।

जब पथ अनुपस्थित होता है, तब `--json` stdout पर `{ "error": "Config path not found: <path>" }` लिखता है और स्थिति 1 के साथ बाहर निकलता है। `--json` के बिना, निदान stderr पर ही रहता है।

```bash
openclaw config get browser.executablePath
openclaw config get agents.defaults.model --json
```

### `config file`

`OPENCLAW_CONFIG_PATH` या डिफ़ॉल्ट स्थान से निर्धारित सक्रिय कॉन्फ़िग फ़ाइल पथ प्रिंट करता है। पथ किसी नियमित फ़ाइल को निर्दिष्ट करता है, सिमलिंक को नहीं; [लेखन सुरक्षा](#write-safety) देखें।

### `config schema`

`openclaw.json` के लिए जनरेट किया गया JSON स्कीमा stdout पर प्रिंट करता है।

<AccordionGroup>
  <Accordion title="इसमें क्या शामिल है">
    - वर्तमान रूट कॉन्फ़िग स्कीमा, साथ ही संपादक टूलिंग के लिए एक रूट `$schema` स्ट्रिंग फ़ील्ड।
    - Control UI द्वारा उपयोग किया जाने वाला फ़ील्ड `title` / `description` दस्तावेज़ मेटाडेटा।
    - जब मेल खाने वाले फ़ील्ड दस्तावेज़ मौजूद हों, तब नेस्टेड ऑब्जेक्ट, वाइल्डकार्ड (`*`), और ऐरे-आइटम (`[]`) नोड समान `title` / `description` मेटाडेटा इनहेरिट करते हैं।
    - `anyOf` / `oneOf` / `allOf` शाखाएँ भी समान दस्तावेज़ मेटाडेटा इनहेरिट करती हैं।
    - रनटाइम मैनिफ़ेस्ट लोड किए जा सकने पर सर्वोत्तम-प्रयास वाला लाइव Plugin + चैनल स्कीमा मेटाडेटा।
    - वर्तमान कॉन्फ़िग अमान्य होने पर भी एक स्वच्छ फ़ॉलबैक स्कीमा।

  </Accordion>
  <Accordion title="संबंधित रनटाइम RPC">
    `config.schema.lookup` एक सामान्यीकृत कॉन्फ़िग पथ को उथले स्कीमा नोड (`title`, `description`, `type`, `enum`, `const`, सामान्य सीमाएँ), मेल खाने वाले UI संकेत मेटाडेटा और निकटतम चाइल्ड सारांशों के साथ लौटाता है। Control UI या कस्टम क्लाइंट में पथ-स्कोप्ड ड्रिल-डाउन के लिए इसका उपयोग करें।
  </Accordion>
</AccordionGroup>

```bash
openclaw config schema
openclaw config schema > openclaw.schema.json
```

### `config validate`

Gateway शुरू किए बिना वर्तमान कॉन्फ़िग को सक्रिय स्कीमा के विरुद्ध सत्यापित करता है।

```bash
openclaw config validate
openclaw config validate --json
```

<Note>
यदि सत्यापन पहले से विफल हो रहा है, तो `openclaw configure` या `openclaw doctor --fix` से शुरू करें। `openclaw chat` अमान्य-कॉन्फ़िग गार्ड को बायपास नहीं करता।
</Note>

## मान

जहाँ संभव हो, मानों को JSON5 के रूप में पार्स किया जाता है; अन्यथा उन्हें मूल स्ट्रिंग माना जाता है। बिना किसी स्ट्रिंग फ़ॉलबैक के मानक JSON आवश्यक करने के लिए `--strict-json` का उपयोग करें (तब केवल-JSON5 सिंटैक्स, जैसे टिप्पणियाँ, अनुगामी कॉमा या उद्धरण-रहित कुंजियाँ, अस्वीकार कर दिए जाते हैं)। `config set` पर `--json`, `--strict-json` का एक पुराना उपनाम है।

```bash
openclaw config set agents.defaults.heartbeat.every "0m"
openclaw config set gateway.port 19001 --strict-json
openclaw config set channels.whatsapp.groups '["*"]' --strict-json
```

`config get <path> --json` टर्मिनल-स्वरूपित टेक्स्ट के बजाय मूल मान को JSON के रूप में प्रिंट करता है।

जब कोई लेखन `agents.defaults.model` या प्रति-एजेंट `agents.entries.*.model` को बदलता है, तब OpenClaw लिखने से पहले प्रत्येक बदले हुए प्राथमिक या फ़ॉलबैक को कॉन्फ़िगर किए गए प्रदाता कैटलॉग के माध्यम से निर्धारित करता है। अज्ञात मॉडल संदर्भ सक्रिय कॉन्फ़िग को बदले बिना अस्वीकार कर दिए जाते हैं; उपलब्ध मॉडल देखने के लिए `openclaw models list` चलाएँ।

<Note>
ऑब्जेक्ट असाइनमेंट डिफ़ॉल्ट रूप से लक्ष्य पथ को प्रतिस्थापित करता है। सामान्यतः उपयोगकर्ता द्वारा जोड़ी गई प्रविष्टियाँ रखने वाले संरक्षित पथ ऐसे प्रतिस्थापनों को अस्वीकार करते हैं जो मौजूदा प्रविष्टियाँ हटा देंगे, जब तक कि आप `--replace` न दें: `agents.defaults.models`, `agents.entries`, `models.providers`, `models.providers.<id>`, `models.providers.<id>.models`, `plugins.entries`, और `auth.profiles`।
</Note>

उन मैप में प्रविष्टियाँ जोड़ते समय `--merge` का उपयोग करें:

```bash
openclaw config set agents.defaults.models '{"openai/gpt-5.4":{}}' --strict-json --merge
openclaw config set models.providers.ollama.models '[{"id":"llama3.2","name":"Llama 3.2"}]' --strict-json --merge
```

`--replace` का उपयोग केवल तभी करें, जब दिया गया मान जानबूझकर पूर्ण लक्ष्य मान बनना चाहिए।

## `config set` मोड

<Tabs>
  <Tab title="मान मोड">
    ```bash
    openclaw config set <path> <value>
    ```
  </Tab>
  <Tab title="SecretRef बिल्डर मोड">
    ```bash
    openclaw config set channels.discord.token \
      --ref-provider default \
      --ref-source env \
      --ref-id DISCORD_BOT_TOKEN
    ```
  </Tab>
  <Tab title="प्रदाता बिल्डर मोड">
    केवल `secrets.providers.<alias>` पथों को लक्षित करता है:

    ```bash
    openclaw config set secrets.providers.vault \
      --provider-source exec \
      --provider-command /usr/local/bin/openclaw-vault \
      --provider-arg read \
      --provider-arg openai/api-key \
      --provider-timeout-ms 5000
    ```

  </Tab>
  <Tab title="बैच मोड">
    ```bash
    openclaw config set --batch-json '[
      {
        "path": "secrets.providers.default",
        "provider": { "source": "env" }
      },
      {
        "path": "channels.discord.token",
        "ref": { "source": "env", "provider": "default", "id": "DISCORD_BOT_TOKEN" }
      }
    ]'
    ```

    ```bash
    openclaw config set --batch-file ./config-set.batch.json --dry-run
    ```

    बैच फ़ाइलें अधिकतम 8 MiB तक सीमित हैं।

  </Tab>
</Tabs>

<Warning>
SecretRef असाइनमेंट असमर्थित रनटाइम-परिवर्तनीय सतहों पर अस्वीकार किए जाते हैं (उदाहरण के लिए `hooks.token`, `commands.ownerDisplaySecret`, Discord थ्रेड-बाइंडिंग Webhook टोकन और WhatsApp क्रेडेंशियल JSON)। [SecretRef क्रेडेंशियल सतह](/hi/reference/secretref-credential-surface) देखें।
</Warning>

बैच पार्सिंग हमेशा बैच पेलोड (`--batch-json`/`--batch-file`) को सत्य का स्रोत मानती है; `--strict-json` / `--json` बैच पार्सिंग व्यवहार को नहीं बदलते।

JSON पथ/मान मोड सीधे SecretRefs और प्रदाताओं के लिए भी काम करता है:

```bash
openclaw config set channels.discord.token \
  '{"source":"env","provider":"default","id":"DISCORD_BOT_TOKEN"}' \
  --strict-json

openclaw config set secrets.providers.vaultfile \
  '{"source":"file","path":"/etc/openclaw/secrets.json","mode":"json"}' \
  --strict-json
```

### प्रदाता बिल्डर फ़्लैग

प्रदाता बिल्डर लक्ष्यों में पथ के रूप में `secrets.providers.<alias>` का उपयोग होना आवश्यक है।

<AccordionGroup>
  <Accordion title="सामान्य फ़्लैग">
    - `--provider-source <env|file|exec>`
    - `--provider-timeout-ms <ms>` (`file`, `exec`)

  </Accordion>
  <Accordion title="Env प्रदाता (--provider-source env)">
    - `--provider-allowlist <ENV_VAR>` (दोहराया जा सकता है)

  </Accordion>
  <Accordion title="फ़ाइल प्रदाता (--provider-source file)">
    - `--provider-path <path>` (आवश्यक)
    - `--provider-mode <singleValue|json>`
    - `--provider-max-bytes <bytes>`
    - `--provider-allow-insecure-path`

  </Accordion>
  <Accordion title="Exec प्रदाता (--provider-source exec)">
    - `--provider-command <path>` (आवश्यक)
    - `--provider-arg <arg>` (दोहराया जा सकता है)
    - `--provider-no-output-timeout-ms <ms>`
    - `--provider-max-output-bytes <bytes>`
    - `--provider-json-only`
    - `--provider-env <KEY=VALUE>` (दोहराया जा सकता है)
    - `--provider-pass-env <ENV_VAR>` (दोहराया जा सकता है)
    - `--provider-trusted-dir <path>` (दोहराया जा सकता है)
    - `--provider-allow-insecure-path`
    - `--provider-allow-symlink-command`

  </Accordion>
</AccordionGroup>

सुदृढ़ किया गया exec प्रदाता उदाहरण:

```bash
openclaw config set secrets.providers.vault \
  --provider-source exec \
  --provider-command /usr/local/bin/openclaw-vault \
  --provider-arg read \
  --provider-arg openai/api-key \
  --provider-json-only \
  --provider-pass-env VAULT_TOKEN \
  --provider-trusted-dir /usr/local/bin \
  --provider-timeout-ms 5000
```

## `config patch`

कई पथ-आधारित `config set` कमांड चलाने के बजाय कॉन्फ़िग-आकार का JSON5 पैच चिपकाएँ या पाइप करें। ऑब्जेक्ट पुनरावर्ती रूप से मर्ज होते हैं; ऐरे और स्केलर मान लक्ष्य को प्रतिस्थापित करते हैं; `null` लक्ष्य पथ को हटाता है।

```bash
openclaw config patch --file ./openclaw.patch.json5 --dry-run
openclaw config patch --file ./openclaw.patch.json5
```

पैच फ़ाइलें अधिकतम 8 MiB तक सीमित हैं। पाइप किए गए `--stdin` पैच अधिकतम 1 MiB तक सीमित हैं।

दूरस्थ सेटअप स्क्रिप्ट के लिए stdin के माध्यम से पैच पाइप करें:

```bash
ssh user@gateway-host 'openclaw config patch --stdin --dry-run' < ./openclaw.patch.json5
ssh user@gateway-host 'openclaw config patch --stdin' < ./openclaw.patch.json5
```

उदाहरण पैच:

```json5
{
  channels: {
    slack: {
      enabled: true,
      mode: "socket",
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      appToken: { source: "env", provider: "default", id: "SLACK_APP_TOKEN" },
      groupPolicy: "open",
      requireMention: false,
    },
    discord: {
      enabled: true,
      token: { source: "env", provider: "default", id: "DISCORD_BOT_TOKEN" },
      dmPolicy: "disabled",
      dm: { enabled: false },
      groupPolicy: "allowlist",
    },
  },
  agents: {
    defaults: {
      model: { primary: "openai/gpt-5.6-sol" },
      models: {
        "openai/gpt-5.6-sol": { params: { fastMode: true } },
      },
    },
  },
}
```

जब किसी ऑब्जेक्ट या ऐरे को पुनरावर्ती रूप से पैच किए जाने के बजाय ठीक दिए गए मान के बराबर बनाना हो, तब `--replace-path <path>` का उपयोग करें:

```bash
openclaw config patch --file ./discord.patch.json5 --replace-path 'channels.discord.guilds["123"].channels'
```

`--dry-run` बिना लिखे स्कीमा और SecretRef की समाधान-क्षमता की जाँच करता है। ड्राई रन के दौरान Exec-समर्थित SecretRefs डिफ़ॉल्ट रूप से छोड़ दिए जाते हैं; जब आप जानबूझकर चाहते हों कि ड्राई रन प्रदाता कमांड निष्पादित करे, तब `--allow-exec` जोड़ें।

## ड्राई रन

`--dry-run`, `openclaw.json` लिखे बिना परिवर्तनों को सत्यापित करता है। यह `config set`, `config patch`, और `config unset` पर उपलब्ध है।

```bash
openclaw config set channels.discord.token \
  --ref-provider default \
  --ref-source env \
  --ref-id DISCORD_BOT_TOKEN \
  --dry-run \
  --json

openclaw config set channels.discord.token \
  --ref-provider vault \
  --ref-source exec \
  --ref-id discord/token \
  --dry-run \
  --allow-exec
```

<AccordionGroup>
  <Accordion title="ड्राई-रन का व्यवहार">
    - बिल्डर मोड: बदले गए refs/प्रदाताओं के लिए SecretRef की समाधान-क्षमता की जाँच करता है।
    - JSON मोड (`--strict-json`, `--json`, या बैच मोड): स्कीमा सत्यापन के साथ SecretRef की समाधान-क्षमता की जाँच करता है।
    - नीति सत्यापन परिवर्तन के बाद की पूरी कॉन्फ़िगरेशन पर चलता है, इसलिए पैरेंट-ऑब्जेक्ट लेखन (उदाहरण के लिए `hooks` को ऑब्जेक्ट के रूप में सेट करना) असमर्थित सतह के सत्यापन को बायपास नहीं कर सकता।
    - कमांड के दुष्प्रभावों से बचने के लिए Exec SecretRef जाँच डिफ़ॉल्ट रूप से छोड़ दी जाती है; इसे चुनने के लिए `--allow-exec` पास करें (इससे प्रदाता कमांड निष्पादित हो सकते हैं)। `--allow-exec` केवल ड्राई-रन के लिए है और `--dry-run` के बिना त्रुटि देता है।

  </Accordion>
  <Accordion title="--dry-run --json फ़ील्ड">
    - `ok`: ड्राई-रन सफल हुआ या नहीं
    - `operations`: मूल्यांकित असाइनमेंट की संख्या
    - `checks`: स्कीमा/समाधान-क्षमता की जाँच चली या नहीं
    - `checks.resolvabilityComplete`: समाधान-क्षमता की जाँच पूरी हुई या नहीं (exec refs छोड़े जाने पर false)
    - `refsChecked`: ड्राई-रन के दौरान वास्तव में समाधान किए गए refs की संख्या
    - `skippedExecRefs`: `--allow-exec` सेट न होने के कारण छोड़े गए exec refs की संख्या
    - `errors`: `ok=false` होने पर संरचित अनुपलब्ध-पथ, स्कीमा, या समाधान-क्षमता विफलताएँ

  </Accordion>
</AccordionGroup>

### JSON आउटपुट की संरचना

```json5
{
  ok: boolean,
  operations: number,
  configPath: string,
  inputModes: ["value" | "json" | "builder" | "unset", ...],
  checks: {
    schema: boolean,
    resolvability: boolean,
    resolvabilityComplete: boolean,
  },
  refsChecked: number,
  skippedExecRefs: number,
  errors?: [
    {
      kind: "missing-path" | "schema" | "resolvability" | "model",
      message: string,
      ref?: string, // समाधान-क्षमता संबंधी त्रुटियों के लिए मौजूद
    },
  ],
}
```

<Tabs>
  <Tab title="सफलता का उदाहरण">
    ```json
    {
      "ok": true,
      "operations": 1,
      "configPath": "~/.openclaw/openclaw.json",
      "inputModes": ["builder"],
      "checks": {
        "schema": false,
        "resolvability": true,
        "resolvabilityComplete": true
      },
      "refsChecked": 1,
      "skippedExecRefs": 0
    }
    ```
  </Tab>
  <Tab title="विफलता का उदाहरण">
    ```json
    {
      "ok": false,
      "operations": 1,
      "configPath": "~/.openclaw/openclaw.json",
      "inputModes": ["builder"],
      "checks": {
        "schema": false,
        "resolvability": true,
        "resolvabilityComplete": true
      },
      "refsChecked": 1,
      "skippedExecRefs": 0,
      "errors": [
        {
          "kind": "resolvability",
          "message": "त्रुटि: पर्यावरण चर \"MISSING_TEST_SECRET\" सेट नहीं है।",
          "ref": "env:default:MISSING_TEST_SECRET"
        }
      ]
    }
    ```
  </Tab>
</Tabs>

<AccordionGroup>
  <Accordion title="यदि ड्राई-रन विफल हो">
    - `config schema validation failed`: परिवर्तन के बाद आपकी कॉन्फ़िगरेशन संरचना अमान्य है; पथ/मान या प्रदाता/ref ऑब्जेक्ट संरचना ठीक करें।
    - `Config policy validation failed: unsupported SecretRef usage`: उस क्रेडेंशियल को वापस प्लेनटेक्स्ट/स्ट्रिंग इनपुट में ले जाएँ; SecretRefs को केवल समर्थित सतहों पर रखें।
    - `SecretRef assignment(s) could not be resolved`: संदर्भित प्रदाता/ref का वर्तमान में समाधान नहीं किया जा सकता (अनुपलब्ध env var, अमान्य फ़ाइल पॉइंटर, exec प्रदाता की विफलता, या प्रदाता/स्रोत का असंगत होना)।
    - `model reference validation failed`: बदला गया टेक्स्ट-मॉडल प्राथमिक या फ़ॉलबैक अज्ञात है; `openclaw models list` चलाएँ और उपलब्ध मॉडल चुनें।
    - `Dry run note: skipped <n> exec SecretRef resolvability check(s)`: यदि आपको exec समाधान-क्षमता सत्यापन चाहिए, तो `--allow-exec` के साथ दोबारा चलाएँ।
    - बैच मोड के लिए, विफल प्रविष्टियाँ ठीक करें और लिखने से पहले `--dry-run` दोबारा चलाएँ।

  </Accordion>
</AccordionGroup>

## परिवर्तन लागू करना

प्रत्येक सफल `config set` / `config patch` / `config unset` के बाद, CLI तीन संकेतों में से एक प्रिंट करता है, ताकि आपको पता चले कि Gateway को पुनः आरंभ करने की आवश्यकता है या नहीं:

| संकेत                                                | अर्थ                                |
| --------------------------------------------------- | -------------------------------------- |
| `Restart the gateway to apply.`                     | बदले गए पथ के लिए पूर्ण पुनः आरंभ आवश्यक है। |
| `Change will apply without restarting the gateway.` | हॉट रीलोड इसे स्वचालित रूप से अपना लेता है।  |
| `No gateway restart needed.`                        | रनटाइम से संबंधित कुछ भी नहीं बदला।      |

`plugins.entries` (या उसके किसी भी उपपथ) में लेखन के लिए हमेशा पुनः आरंभ आवश्यक होता है, क्योंकि CLI यह प्रमाणित नहीं कर सकता कि प्रत्येक Plugin का रीलोड मेटाडेटा लोड हुआ है।

## लेखन सुरक्षा

`openclaw config set` और OpenClaw के स्वामित्व वाले अन्य कॉन्फ़िगरेशन राइटर, डिस्क पर कमिट करने से पहले परिवर्तन के बाद की पूरी कॉन्फ़िगरेशन को सत्यापित करते हैं। यदि नया पेलोड स्कीमा सत्यापन में विफल होता है या विनाशकारी ओवरराइट जैसा दिखता है, तो सक्रिय कॉन्फ़िगरेशन को अपरिवर्तित छोड़ा जाता है और अस्वीकृत पेलोड को उसके पास `openclaw.json.rejected.*` के रूप में सहेजा जाता है।

OpenClaw के स्वामित्व वाले लेखन JSON5 को मानक JSON के रूप में फिर से क्रमबद्ध करते हैं। जब स्रोत में टिप्पणियाँ होती हैं, तो राइटर उन्हें हटाने से ठीक पहले चेतावनी देता है; टिप्पणियाँ सुरक्षित रखना महत्वपूर्ण हो तो सीधे एडिटर का उपयोग करें।

<Warning>
सक्रिय कॉन्फ़िगरेशन पथ एक नियमित फ़ाइल होना चाहिए। सिमलिंक किए गए `openclaw.json` लेआउट लेखन के लिए असमर्थित हैं; इसके बजाय वास्तविक फ़ाइल को सीधे इंगित करने के लिए `OPENCLAW_CONFIG_PATH` का उपयोग करें।
</Warning>

छोटे संपादनों के लिए CLI लेखन को प्राथमिकता दें:

```bash
openclaw config set gateway.reload.mode hybrid --dry-run
openclaw config set gateway.reload.mode hybrid
openclaw config validate
```

यदि कोई लेखन अस्वीकार हो जाता है, तो सहेजे गए पेलोड का निरीक्षण करें और पूरी कॉन्फ़िगरेशन संरचना ठीक करें:

```bash
CONFIG="$(openclaw config file)"
ls -lt "$CONFIG".rejected.* 2>/dev/null | head
openclaw config validate
```

सीधे एडिटर से लिखना अभी भी अनुमत है, लेकिन चल रहा Gateway सत्यापन होने तक उन्हें अविश्वसनीय मानता है। अमान्य सीधे संपादन स्टार्टअप को विफल कर देते हैं या हॉट रीलोड द्वारा छोड़ दिए जाते हैं; Gateway `openclaw.json` को फिर से नहीं लिखता। उपसर्गयुक्त/ओवरराइट की गई कॉन्फ़िगरेशन की मरम्मत करने या अंतिम ज्ञात-सही प्रति पुनर्स्थापित करने के लिए `openclaw doctor --fix` चलाएँ। [Gateway समस्या निवारण](/hi/gateway/troubleshooting#gateway-rejected-invalid-config) देखें।

पूरी फ़ाइल की पुनर्प्राप्ति केवल डॉक्टर द्वारा मरम्मत के लिए आरक्षित है। Plugin स्कीमा परिवर्तन या `minHostVersion` असंगति, मॉडल, प्रदाता, प्रमाणीकरण प्रोफ़ाइल, चैनल, Gateway एक्सपोज़र, टूल, मेमोरी, ब्राउज़र, या Cron कॉन्फ़िगरेशन जैसी असंबंधित उपयोगकर्ता सेटिंग्स को वापस लेने के बजाय स्पष्ट त्रुटि देते हैं।

## मरम्मत चक्र

`openclaw config validate` सफल होने के बाद, स्थानीय TUI का उपयोग करके किसी एम्बेडेड एजेंट से सक्रिय कॉन्फ़िगरेशन की दस्तावेज़ों से तुलना करवाएँ, जबकि आप उसी टर्मिनल से प्रत्येक परिवर्तन को सत्यापित करें:

```bash
openclaw chat
```

TUI के अंदर, आरंभिक `!` एक शाब्दिक स्थानीय शेल कमांड चलाता है (प्रति सत्र एक बार पुष्टिकरण संकेत मिलने के बाद):

```text
!openclaw config file
!openclaw docs gateway auth token secretref
!openclaw config validate
!openclaw doctor
```

<Steps>
  <Step title="दस्तावेज़ों से तुलना करें">
    एजेंट से अपनी वर्तमान कॉन्फ़िगरेशन की संबंधित दस्तावेज़ पृष्ठ से तुलना करने और सबसे छोटा सुधार सुझाने के लिए कहें।
  </Step>
  <Step title="लक्षित संपादन लागू करें">
    `openclaw config set` या `openclaw configure` के साथ लक्षित संपादन लागू करें।
  </Step>
  <Step title="दोबारा सत्यापित करें">
    प्रत्येक परिवर्तन के बाद `openclaw config validate` दोबारा चलाएँ।
  </Step>
  <Step title="रनटाइम समस्याओं के लिए डॉक्टर">
    यदि सत्यापन सफल हो जाता है लेकिन रनटाइम अभी भी अस्वस्थ है, तो माइग्रेशन और मरम्मत सहायता के लिए `openclaw doctor` या `openclaw doctor --fix` चलाएँ।
  </Step>
</Steps>

## संबंधित

- [CLI संदर्भ](/hi/cli)
- [कॉन्फ़िगरेशन](/hi/gateway/configuration)
