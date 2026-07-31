---
read_when:
    - आप OpenClaw के साथ Meta का उपयोग करना चाहते हैं
    - आपको `MODEL_API_KEY` env var या CLI प्रमाणीकरण विकल्प की आवश्यकता है
summary: Meta सेटअप (प्रमाणीकरण + muse-spark-1.1 मॉडल चयन)
title: Meta
x-i18n:
    generated_at: "2026-07-27T18:54:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f2ce7616d9abc14a2d15ee53ea7725d3e70059af1a38bb61dbfe5b3969106432
    source_path: providers/meta.md
    workflow: 16
---

**Meta API** OpenAI-संगत **Responses API** (`POST /v1/responses`) का उपयोग
`muse-spark-1.1` रीजनिंग मॉडल के लिए करता है। यह प्रोवाइडर एक बंडल किए गए OpenClaw
plugin के रूप में उपलब्ध होता है।

| गुण               | मान                                |
| ----------------- | ---------------------------------- |
| प्रोवाइडर आईडी    | `meta`                 |
| Plugin            | बंडल किया गया प्रोवाइडर            |
| प्रमाणीकरण env var | `MODEL_API_KEY`                |
| ऑनबोर्डिंग फ़्लैग | `--auth-choice meta-api-key`                 |
| प्रत्यक्ष CLI फ़्लैग | `--meta-api-key <key>`               |
| API               | Responses API (`openai-responses`) |
| बेस URL           | `https://api.meta.ai/v1`                 |
| डिफ़ॉल्ट मॉडल     | `meta/muse-spark-1.1`                 |
| डिफ़ॉल्ट रीजनिंग  | `high` (`reasoning.effort`) |

## आरंभ करना

<Steps>
  <Step title="API कुंजी सेट करें">
    <CodeGroup>

```bash Onboarding
openclaw onboard --auth-choice meta-api-key
```

```bash Direct flag
openclaw onboard --non-interactive --accept-risk \
  --auth-choice meta-api-key \
  --meta-api-key "$MODEL_API_KEY"
```

```bash Env only
export MODEL_API_KEY=<key>
```

    </CodeGroup>

  </Step>
  <Step title="सत्यापित करें कि मॉडल उपलब्ध हैं">
    ```bash
    openclaw models list --provider meta
    ```

    स्थिर `muse-spark-1.1` कैटलॉग प्रविष्टि सूचीबद्ध करता है। यदि `MODEL_API_KEY` का समाधान नहीं होता,
    तो `openclaw models status --json` अनुपलब्ध क्रेडेंशियल को
    `auth.unusableProfiles` के अंतर्गत रिपोर्ट करता है।

  </Step>
</Steps>

## गैर-इंटरैक्टिव सेटअप

```bash
openclaw onboard --non-interactive --accept-risk \
  --mode local \
  --auth-choice meta-api-key \
  --meta-api-key "$MODEL_API_KEY"
```

## अंतर्निहित कैटलॉग

| मॉडल रेफ़रेंस         | नाम            | रीजनिंग | कॉन्टेक्स्ट विंडो | अधिकतम आउटपुट |
| --------------------- | -------------- | --------- | -------------- | ---------- |
| `meta/muse-spark-1.1` | Muse Spark 1.1 | हाँ       | 1,048,576      | 131,072    |

क्षमताएँ:

- टेक्स्ट + इमेज इनपुट
- टूल कॉलिंग और स्ट्रीमिंग
- रीजनिंग प्रयास: `minimal`, `low`, `medium`, `high`, `xhigh` (डिफ़ॉल्ट: `high`)
- स्टेटलेस एन्क्रिप्टेड रीजनिंग रीप्ले (`store: false`, `include: ["reasoning.encrypted_content"]`)

<Warning>
`muse-spark-1.1`, `reasoning.effort: "none"` स्वीकार नहीं करता। OpenClaw इस प्रोवाइडर के लिए
`--thinking off` को `minimal` से मैप करता है।
</Warning>

## मैन्युअल कॉन्फ़िगरेशन

```json5
{
  env: { MODEL_API_KEY: "<key>" },
  agents: {
    defaults: {
      model: { primary: "meta/muse-spark-1.1" },
      models: {
        "meta/muse-spark-1.1": { alias: "Muse Spark 1.1" },
      },
    },
  },
}
```

<Note>
यदि Gateway डेमन (launchd, systemd, Docker) के रूप में चलता है, तो सुनिश्चित करें कि
`MODEL_API_KEY` उस प्रोसेस के लिए उपलब्ध हो—उदाहरण के लिए,
`~/.openclaw/.env` में या `env.shellEnv` के माध्यम से। केवल इंटरैक्टिव शेल में एक्सपोर्ट की गई
कुंजी किसी प्रबंधित सेवा के लिए उपयोगी नहीं होगी, जब तक कि env को अलग से इंपोर्ट
न किया जाए।
</Note>

## स्मोक टेस्ट

```bash
export MODEL_API_KEY=<key>
pnpm test:live -- extensions/meta/meta.live.test.ts
```

लाइव टेस्ट `POST /v1/responses` के विरुद्ध `muse-spark-1.1` का उपयोग करते हैं।

## संबंधित

<CardGroup cols={2}>
  <Card title="मॉडल प्रोवाइडर" href="/hi/concepts/model-providers" icon="layers">
    प्रोवाइडर, मॉडल रेफ़रेंस और फ़ेलओवर व्यवहार चुनना।
  </Card>
  <Card title="थिंकिंग मोड" href="/hi/tools/thinking" icon="brain">
    muse-spark-1.1 के लिए रीजनिंग प्रयास के स्तर।
  </Card>
  <Card title="कॉन्फ़िगरेशन संदर्भ" href="/hi/gateway/config-agents#agent-defaults" icon="gear">
    एजेंट डिफ़ॉल्ट और मॉडल कॉन्फ़िगरेशन।
  </Card>
</CardGroup>
