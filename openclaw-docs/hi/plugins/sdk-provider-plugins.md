---
read_when:
    - आप एक नया मॉडल प्रदाता Plugin बना रहे हैं
    - आप OpenClaw में OpenAI-संगत प्रॉक्सी या कस्टम LLM जोड़ना चाहते हैं
    - आपको प्रदाता प्रमाणीकरण, कैटलॉग और रनटाइम हुक को समझना होगा
sidebarTitle: Provider plugins
summary: OpenClaw के लिए मॉडल प्रदाता Plugin बनाने की चरण-दर-चरण मार्गदर्शिका
title: प्रोवाइडर Plugin बनाना
x-i18n:
    generated_at: "2026-07-27T18:49:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f9d175fafc034bd52e996d47e047df104f079f2aba66662b22e8dbdf6c21e7e0
    source_path: plugins/sdk-provider-plugins.md
    workflow: 16
---

OpenClaw में मॉडल प्रदाता (LLM) जोड़ने के लिए एक प्रदाता Plugin बनाएँ: एक मॉडल
कैटलॉग, API-कुंजी प्रमाणीकरण और डायनेमिक मॉडल रिज़ॉल्यूशन।

<Info>
  OpenClaw plugins में नए हैं? पैकेज संरचना और मैनिफ़ेस्ट सेटअप के लिए पहले
  [शुरुआत करना](/hi/plugins/building-plugins) पढ़ें।
</Info>

<Tip>
  प्रदाता plugins OpenClaw के सामान्य इन्फ़रेंस लूप में मॉडल जोड़ते हैं। यदि
  मॉडल को किसी ऐसे नेटिव एजेंट डेमन के माध्यम से चलना आवश्यक है जो थ्रेड्स, Compaction
  या टूल इवेंट्स का स्वामी है, तो डेमन प्रोटोकॉल का विवरण कोर में रखने के बजाय प्रदाता को
  [एजेंट हार्नेस](/hi/plugins/sdk-agent-harness) के साथ जोड़ें।
</Tip>

## चरण-दर-चरण विवरण

<Steps>
  <Step title="पैकेज और मैनिफ़ेस्ट">
    ### चरण 1: पैकेज और मैनिफ़ेस्ट

    <CodeGroup>
    ```json package.json
    {
      "name": "@myorg/openclaw-acme-ai",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "providers": ["acme-ai"],
        "compat": {
          "pluginApi": ">=2026.3.24-beta.2",
          "minGatewayVersion": "2026.3.24-beta.2"
        },
        "build": {
          "openclawVersion": "2026.3.24-beta.2",
          "pluginSdkVersion": "2026.3.24-beta.2"
        }
      }
    }
    ```

    ```json openclaw.plugin.json
    {
      "id": "acme-ai",
      "name": "Acme AI",
      "description": "Acme AI मॉडल प्रदाता",
      "providers": ["acme-ai"],
      "modelSupport": {
        "modelPrefixes": ["acme-"]
      },
      "setup": {
        "providers": [
          {
            "id": "acme-ai",
            "envVars": ["ACME_AI_API_KEY"]
          }
        ]
      },
      "providerAuthAliases": {
        "acme-ai-coding": "acme-ai"
      },
      "providerAuthChoices": [
        {
          "provider": "acme-ai",
          "method": "api-key",
          "choiceId": "acme-ai-api-key",
          "choiceLabel": "Acme AI API कुंजी",
          "groupId": "acme-ai",
          "groupLabel": "Acme AI",
          "cliFlag": "--acme-ai-api-key",
          "cliOption": "--acme-ai-api-key <key>",
          "cliDescription": "Acme AI API कुंजी"
        }
      ],
      "configSchema": {
        "type": "object",
        "additionalProperties": false
      }
    }
    ```
    </CodeGroup>

    `setup.providers[].envVars` आपके Plugin रनटाइम को लोड किए बिना OpenClaw को
    क्रेडेंशियल्स का पता लगाने देता है। जब किसी प्रदाता वेरिएंट को किसी अन्य प्रदाता आईडी
    के प्रमाणीकरण का पुनः उपयोग करना हो, तो `providerAuthAliases` जोड़ें। `modelSupport`
    वैकल्पिक है और रनटाइम हुक उपलब्ध होने से पहले OpenClaw को
    `acme-large` जैसे संक्षिप्त मॉडल आईडी से आपके प्रदाता Plugin को स्वतः लोड करने देता है।
    `package.json` में `openclaw.compat` और `openclaw.build` ClawHub
    पर प्रकाशित करने के लिए आवश्यक हैं (`openclaw.compat.pluginApi` और `openclaw.build.openclawVersion`
    दो आवश्यक फ़ील्ड हैं; छोड़े जाने पर `minGatewayVersion` के लिए
    `openclaw.install.minHostVersion` का उपयोग किया जाता है)।

  </Step>

  <Step title="प्रदाता पंजीकृत करें">
    न्यूनतम टेक्स्ट प्रदाता को `id`, `label`, `auth` और `catalog` की आवश्यकता होती है।
    `catalog` प्रदाता के स्वामित्व वाला रनटाइम/कॉन्फ़िगरेशन हुक है; यह लाइव
    विक्रेता API को कॉल कर सकता है और `models.providers` प्रविष्टियाँ लौटाता है।

    ```typescript index.ts
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
    import { createProviderApiKeyAuthMethod } from "openclaw/plugin-sdk/provider-auth";

    export default definePluginEntry({
      id: "acme-ai",
      name: "Acme AI",
      description: "Acme AI model provider",
      register(api) {
        api.registerProvider({
          id: "acme-ai",
          label: "Acme AI",
          docsPath: "/providers/acme-ai",
          envVars: ["ACME_AI_API_KEY"],

          auth: [
            createProviderApiKeyAuthMethod({
              providerId: "acme-ai",
              methodId: "api-key",
              label: "Acme AI API key",
              hint: "API key from your Acme AI dashboard",
              optionKey: "acmeAiApiKey",
              flagName: "--acme-ai-api-key",
              envVar: "ACME_AI_API_KEY",
              promptMessage: "Enter your Acme AI API key",
              defaultModel: "acme-ai/acme-large",
            }),
          ],

          catalog: {
            order: "simple",
            run: async (ctx) => {
              const apiKey =
                ctx.resolveProviderApiKey("acme-ai").apiKey;
              if (!apiKey) return null;
              return {
                provider: {
                  baseUrl: "https://api.acme-ai.com/v1",
                  apiKey,
                  api: "openai-completions",
                  models: [
                    {
                      id: "acme-large",
                      name: "Acme Large",
                      reasoning: true,
                      input: ["text", "image"],
                      cost: { input: 3, output: 15, cacheRead: 0.3, cacheWrite: 3.75 },
                      contextWindow: 200000,
                      maxTokens: 32768,
                    },
                    {
                      id: "acme-small",
                      name: "Acme Small",
                      reasoning: false,
                      input: ["text"],
                      cost: { input: 1, output: 5, cacheRead: 0.1, cacheWrite: 1.25 },
                      contextWindow: 128000,
                      maxTokens: 8192,
                    },
                  ],
                },
              };
            },
          },
        });

        api.registerModelCatalogProvider({
          provider: "acme-ai",
          kinds: ["text"],
          liveCatalog: async (ctx) => {
            const apiKey = ctx.resolveProviderApiKey("acme-ai").apiKey;
            if (!apiKey) return null;
            return [
              {
                kind: "text",
                provider: "acme-ai",
                model: "acme-large",
                label: "Acme Large",
                source: "live",
              },
            ];
          },
        });
      },
    });
    ```

    `registerModelCatalogProvider` सूची/सहायता/चयनकर्ता UI के लिए नया कंट्रोल-प्लेन कैटलॉग
    सरफ़ेस है, जो `text`, `voice`, `image_generation`,
    `video_generation` और `music_generation` पंक्तियों को कवर करता है। विक्रेता एंडपॉइंट
    कॉल और प्रतिक्रिया मैपिंग को Plugin में रखें; साझा पंक्ति
    आकार, स्रोत लेबल और सहायता रेंडरिंग का स्वामित्व OpenClaw के पास है।

    यह एक कार्यशील प्रदाता है। उपयोगकर्ता अब
    `openclaw onboard --acme-ai-api-key <key>` चला सकते हैं और अपने मॉडल के रूप में
    `acme-ai/acme-large` चुन सकते हैं।

    ### लाइव मॉडल खोज

    यदि आपका प्रदाता OpenAI-संगत `/models` API उपलब्ध कराता है, तो
    एकल-प्रदाता सहायक को साझा खोज के लिए सक्षम करें:

    ```typescript
    catalog: {
      buildProvider: () => ({
        api: "openai-completions",
        baseUrl: "https://api.acme-ai.com/v1",
        models: [...STATIC_MODELS],
      }),
      buildStaticProvider: () => ({
        api: "openai-completions",
        baseUrl: "https://api.acme-ai.com/v1",
        models: [...STATIC_MODELS],
      }),
      liveModelDiscovery: true,
    },
    ```

    `liveModelDiscovery: true` निम्न व्यवहारों वाला एक सार्वजनिक Plugin SDK अनुबंध है:

    | क्षेत्र | अनुबंध |
    | --- | --- |
    | क्रेडेंशियल्स | खोज कैटलॉग के रिज़ॉल्व किए गए प्रदाता क्रेडेंशियल का उपयोग करती है और प्रमाणीकरण द्वारा उपलब्ध कराए जाने पर `discoveryApiKey` को प्राथमिकता देती है। गुप्त-संदर्भ मार्कर कभी टोकन के रूप में नहीं भेजे जाते। डिफ़ॉल्ट अनुरोध `Authorization: Bearer <token>` का उपयोग करता है; किसी अन्य विक्रेता प्रमाणीकरण योजना के लिए `buildRequestHeaders` का उपयोग करें। |
    | एंडपॉइंट | डिफ़ॉल्ट URL प्रभावी प्रदाता `baseUrl` के सापेक्ष `models` है, जिसमें `allowExplicitBaseUrl` सक्षम होने पर ऑपरेटर ओवरराइड भी शामिल है। किसी अन्य सापेक्ष पथ के लिए `endpointPath` का उपयोग करें। केवल निश्चित विक्रेता URL के लिए `endpointUrl: { url, requireBaseUrl }` का उपयोग करें; जब तक प्रभावी बेस URL अभी भी `requireBaseUrl` के बराबर न हो, खोज छोड़ दी जाती है, ताकि कस्टम प्रॉक्सी क्रेडेंशियल विक्रेता को न भेजा जाए। |
    | नेटवर्क सीमाएँ | फ़ेच OpenClaw के SSRF गार्ड, पेजिनेशन में कुल 5-सेकंड टाइमआउट बजट, प्रति पृष्ठ 4 MiB प्रतिक्रिया सीमा और 50-पृष्ठ सीमा का उपयोग करते हैं। क्रॉस-ओरिजिन पेजिनेशन लिंक अस्वीकार किए जाते हैं; क्रॉस-ओरिजिन रीडायरेक्ट के बाद क्रेडेंशियल्स हटा दिए जाते हैं। |
    | कैश | सफल, गैर-रिक्त कैटलॉग प्रदाता, एंडपॉइंट और रिज़ॉल्व किए गए क्रेडेंशियल के आधार पर 60 सेकंड के लिए कैश किए जाते हैं। रिक्त या अनुपयोगी परिणाम कैश नहीं किए जाते। |
    | फ़िल्टरिंग | सटीक लाइव आईडी अपना विश्वसनीय स्थिर मेटाडेटा बनाए रखते हैं। नई पंक्तियाँ सावधानीपूर्वक टेक्स्ट/चैट मॉडल के रूप में प्रोजेक्ट की जाती हैं। अक्षम, आर्काइव किए गए, अप्रचलित, स्पष्ट रूप से गैर-चैट, एम्बेडिंग, री-रैंकिंग, मॉडरेशन, स्पीच, केवल-इमेज और केवल-वीडियो पंक्तियाँ बाहर रखी जाती हैं। गैर-मानक प्रतिक्रिया एनवेलप से पंक्तियाँ चुनने के लिए ही `readRows` का उपयोग करें; प्रदाता-विशिष्ट मॉडल अर्थ-विज्ञान फिर भी कस्टम कैटलॉग में ही होना चाहिए। |
    | विफलता | लाइव खोज परामर्शात्मक है। प्रमाणीकरण, नेटवर्क, टाइमआउट, पेजिनेशन, पार्सिंग, रिक्त-कैटलॉग और फ़िल्टरिंग विफलताएँ प्रदाता को हटाने के बजाय प्रदाता के स्वामित्व वाला स्थिर सीड लौटाती हैं। |

    गैर-Bearer या गैर-मानक सूची एंडपॉइंट के लिए
    `true` के बजाय विकल्प पास करें:

    ```typescript
    liveModelDiscovery: {
      endpointPath: "model-catalog",
      buildRequestHeaders: ({ apiKey, discoveryApiKey }) => ({
        "vendor-version": "2026-01-01",
        "x-api-key": discoveryApiKey ?? apiKey ?? "",
      }),
      readRows: (body) =>
        body && typeof body === "object" &&
        Array.isArray((body as { models?: unknown }).models)
          ? (body as { models: unknown[] }).models
          : [],
    },
    ```

    `endpointUrl` का बिना शर्त वैकल्पिक होस्ट के रूप में उपयोग न करें। इसकी
    `requireBaseUrl` जाँच उन प्रदाताओं के लिए क्रेडेंशियल-पृथक्करण सीमा है
    जिनका मॉडल-सूची होस्ट उनके इन्फ़रेंस होस्ट से अलग होता है।

    यदि प्रदाता को सावधानीपूर्ण OpenAI-संगत प्रोजेक्शन के बजाय कस्टम मॉडल
    अर्थ-विज्ञान की आवश्यकता है, तो उस प्रोजेक्शन को Plugin में रखें और साझा फ़ेच
    जीवनचक्र के लिए `openclaw/plugin-sdk/provider-catalog-live-runtime` का उपयोग करें। सहायक आपको प्रदाता नीति को
    OpenClaw कोर में रखे बिना सुरक्षित HTTP फ़ेच, प्रदाता-प्रमाणीकरण हेडर,
    संरचित HTTP त्रुटियाँ, TTL कैशिंग और स्थिर फ़ॉलबैक व्यवहार देता है।

    जब लाइव API केवल यह बताता हो कि प्रदाता के स्वामित्व वाली स्थिर कैटलॉग
    पंक्तियों में से कौन-सी वर्तमान में उपलब्ध हैं, तब `buildLiveModelProviderConfig` का उपयोग करें:

    ```typescript index.ts
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
    import {
      buildLiveModelProviderConfig,
      type LiveModelCatalogFetchGuard,
    } from "openclaw/plugin-sdk/provider-catalog-live-runtime";

    const STATIC_MODELS = [
      {
        id: "acme-large",
        name: "Acme Large",
        reasoning: true,
        input: ["text", "image"],
        cost: { input: 3, output: 15, cacheRead: 0.3, cacheWrite: 3.75 },
        contextWindow: 200000,
        maxTokens: 32768,
      },
      {
        id: "acme-small",
        name: "Acme Small",
        reasoning: false,
        input: ["text"],
        cost: { input: 1, output: 5, cacheRead: 0.1, cacheWrite: 1.25 },
        contextWindow: 128000,
        maxTokens: 8192,
      },
    ] as const;

    async function buildAcmeLiveProvider(params: {
      apiKey: string;
      discoveryApiKey?: string;
      fetchGuard?: LiveModelCatalogFetchGuard;
    }) {
      return await buildLiveModelProviderConfig({
        providerId: "acme-ai",
        endpoint: "https://api.acme-ai.com/v1/models",
        providerConfig: {
          baseUrl: "https://api.acme-ai.com/v1",
          api: "openai-completions",
        },
        models: STATIC_MODELS,
        apiKey: params.apiKey,
        discoveryApiKey: params.discoveryApiKey,
        fetchGuard: params.fetchGuard,
        ttlMs: 60_000,
        auditContext: "acme-ai-model-discovery",
      });
    }

    export default definePluginEntry({
      id: "acme-ai",
      name: "Acme AI",
      register(api) {
        api.registerProvider({
          id: "acme-ai",
          label: "Acme AI",
          catalog: {
            order: "simple",
            run: async (ctx) => {
              const auth = ctx.resolveProviderAuth("acme-ai");
              const apiKey =
                auth.apiKey ?? ctx.resolveProviderApiKey("acme-ai").apiKey;
              if (!apiKey) return null;
              return {
                provider: await buildAcmeLiveProvider({
                  apiKey,
                  discoveryApiKey: auth.discoveryApiKey,
                }),
              };
            },
          },
          staticCatalog: {
            order: "simple",
            run: async () => ({
              provider: {
                baseUrl: "https://api.acme-ai.com/v1",
                api: "openai-completions",
                models: [...STATIC_MODELS],
              },
            }),
          },
        });
      },
    });
    ```

    जब प्रदाता API अधिक समृद्ध मेटाडेटा लौटाता है और Plugin को पंक्तियों को स्वयं OpenClaw मॉडल
    परिभाषाओं में प्रक्षेपित करना होता है, तब `getCachedLiveProviderModelRows` का उपयोग करें:

    ```typescript index.ts
    import {
      getCachedLiveProviderModelRows,
      LiveModelCatalogHttpError,
    } from "openclaw/plugin-sdk/provider-catalog-live-runtime";

    async function discoverAcmeModels(apiKey: string) {
      try {
        const rows = await getCachedLiveProviderModelRows({
          providerId: "acme-ai",
          endpoint: "https://api.acme-ai.com/v1/models",
          apiKey,
          ttlMs: 60_000,
          auditContext: "acme-ai-model-discovery",
        });
        return rows
          .map((row) => projectAcmeModel(row))
          .filter((model) => model !== null);
      } catch (error) {
        if (error instanceof LiveModelCatalogHttpError) {
          return STATIC_MODELS;
        }
        throw error;
      }
    }
    ```

    `run` को प्रमाणीकरण द्वारा नियंत्रित रहना चाहिए और कोई उपयोग योग्य क्रेडेंशियल
    उपलब्ध न होने पर `null` लौटाना चाहिए। एक ऑफ़लाइन `staticRun` या स्थिर फ़ॉलबैक रखें, ताकि सेटअप, दस्तावेज़,
    परीक्षण और चयनकर्ता सतहें लाइव नेटवर्क पहुँच पर निर्भर न हों। मॉडल-सूची की ताज़गी के लिए
    उपयुक्त TTL का उपयोग करें, अनुरोध के समय फ़ाइल-सिस्टम पोलिंग से बचें,
    और प्रदाता-विशिष्ट `readRows` / `readModelId` केवल तभी पास करें, जब
    अपस्ट्रीम प्रतिक्रिया OpenAI-संगत `{ data: [{ id, object }] }`
    आकार में न हो।

    यदि अपस्ट्रीम प्रदाता OpenClaw से भिन्न नियंत्रण टोकन का उपयोग करता है, तो स्ट्रीम पथ को
    बदलने के बजाय एक छोटा द्विदिश पाठ रूपांतरण जोड़ें:

    ```typescript
    api.registerTextTransforms({
      input: [
        { from: /red basket/g, to: "blue basket" },
        { from: /paper ticket/g, to: "digital ticket" },
        { from: /left shelf/g, to: "right shelf" },
      ],
      output: [
        { from: /blue basket/g, to: "red basket" },
        { from: /digital ticket/g, to: "paper ticket" },
        { from: /right shelf/g, to: "left shelf" },
      ],
    });
    ```

    `input` परिवहन से पहले अंतिम सिस्टम प्रॉम्प्ट और पाठ संदेश की सामग्री को
    पुनर्लिखता है। `output` OpenClaw द्वारा अपने नियंत्रण मार्कर पार्स करने या
    चैनल पर भेजने से पहले सहायक के पाठ डेल्टा और अंतिम पाठ को पुनर्लिखता है।

    ऐसे बंडल किए गए प्रदाताओं के लिए, जो API-कुंजी प्रमाणीकरण के साथ केवल एक पाठ प्रदाता
    और एकल कैटलॉग-समर्थित रनटाइम पंजीकृत करते हैं, अधिक सीमित
    `defineSingleProviderPluginEntry(...)` हेल्पर को प्राथमिकता दें:

    ```typescript
    import { defineSingleProviderPluginEntry } from "openclaw/plugin-sdk/provider-entry";

    export default defineSingleProviderPluginEntry({
      id: "acme-ai",
      name: "Acme AI",
      description: "Acme AI मॉडल प्रदाता",
      provider: {
        label: "Acme AI",
        docsPath: "/providers/acme-ai",
        auth: [
          {
            methodId: "api-key",
            label: "Acme AI API कुंजी",
            hint: "आपके Acme AI डैशबोर्ड की API कुंजी",
            optionKey: "acmeAiApiKey",
            flagName: "--acme-ai-api-key",
            envVar: "ACME_AI_API_KEY",
            promptMessage: "अपनी Acme AI API कुंजी दर्ज करें",
            defaultModel: "acme-ai/acme-large",
          },
        ],
        catalog: {
          buildProvider: () => ({
            api: "openai-completions",
            baseUrl: "https://api.acme-ai.com/v1",
            models: [{ id: "acme-large", name: "Acme Large" }],
          }),
          buildStaticProvider: () => ({
            api: "openai-completions",
            baseUrl: "https://api.acme-ai.com/v1",
            models: [{ id: "acme-large", name: "Acme Large" }],
          }),
        },
      },
    });
    ```

    `buildProvider` वह लाइव कैटलॉग पथ है जिसका उपयोग तब किया जाता है, जब OpenClaw वास्तविक
    प्रदाता प्रमाणीकरण को हल कर सकता है। यह प्रदाता-विशिष्ट खोज कर सकता है। प्रमाणीकरण
    कॉन्फ़िगर होने से पहले सुरक्षित रूप से दिखाई जा सकने वाली ऑफ़लाइन पंक्तियों के लिए ही
    `buildStaticProvider` का उपयोग करें; इसे क्रेडेंशियल की आवश्यकता नहीं होनी चाहिए और न ही
    नेटवर्क अनुरोध करने चाहिए। OpenClaw का `models list --all` प्रदर्शन वर्तमान में स्थिर कैटलॉग
    केवल बंडल किए गए प्रदाता Plugin के लिए निष्पादित करता है, जिसमें कॉन्फ़िग और एन्वायरनमेंट
    रिक्त होते हैं तथा कोई एजेंट/वर्कस्पेस पथ नहीं होता।

    यदि आपके प्रमाणीकरण प्रवाह को ऑनबोर्डिंग के दौरान `models.providers.*`, उपनाम और
    एजेंट का डिफ़ॉल्ट मॉडल भी पैच करना है, तो
    `openclaw/plugin-sdk/provider-onboard` के प्रीसेट हेल्पर का उपयोग करें। सबसे सीमित हेल्पर
    `createDefaultModelPresetAppliers(...)`,
    `createDefaultModelsPresetAppliers(...)`, और
    `createModelCatalogPresetAppliers(...)` हैं।

    जब किसी प्रदाता का नेटिव एंडपॉइंट सामान्य `openai-completions` परिवहन पर स्ट्रीम किए गए
    उपयोग ब्लॉक का समर्थन करता है, तो प्रदाता-id जाँच को हार्डकोड करने के बजाय
    `openclaw/plugin-sdk/provider-catalog-shared` में साझा कैटलॉग हेल्पर को प्राथमिकता दें।
    `supportsNativeStreamingUsageCompat(...)` और
    `applyProviderNativeStreamingUsageCompat(...)` एंडपॉइंट क्षमता मानचित्र से समर्थन का पता लगाते हैं, इसलिए नेटिव
    Moonshot/DashScope-शैली के एंडपॉइंट तब भी विकल्प चुनते हैं, जब कोई Plugin कस्टम प्रदाता id
    का उपयोग कर रहा हो।

    ऊपर दिए गए लाइव खोज उदाहरण `/models`-शैली के प्रदाता API को समेटते हैं। उस खोज को
    `catalog.run` के भीतर रखें, उपयोग योग्य प्रमाणीकरण द्वारा नियंत्रित करें, और ऑफ़लाइन
    कैटलॉग निर्माण के लिए `staticRun` को नेटवर्क-मुक्त रखें।

  </Step>

  <Step title="डायनेमिक मॉडल समाधान जोड़ें">
    यदि आपका प्रदाता मनमाने मॉडल ID स्वीकार करता है (जैसे प्रॉक्सी या राउटर),
    तो `resolveDynamicModel` जोड़ें:

    ```typescript
    api.registerProvider({
      // ... ऊपर से id, लेबल, प्रमाणीकरण, कैटलॉग

      resolveDynamicModel: (ctx) => ({
        id: ctx.modelId,
        name: ctx.modelId,
        provider: "acme-ai",
        api: "openai-completions",
        baseUrl: "https://api.acme-ai.com/v1",
        reasoning: false,
        input: ["text"],
        cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
        contextWindow: 128000,
        maxTokens: 8192,
      }),
    });
    ```

    यदि समाधान के लिए नेटवर्क कॉल आवश्यक है, तो असिंक्रोनस वार्म-अप हेतु
    `prepareDynamicModel` का उपयोग करें—इसके पूर्ण होने के बाद `resolveDynamicModel`
    फिर से चलता है।

  </Step>

  <Step title="रनटाइम हुक जोड़ें (आवश्यकतानुसार)">
    अधिकांश प्रदाताओं को केवल `catalog` + `resolveDynamicModel` की आवश्यकता होती है।
    आपके प्रदाता को आवश्यकता होने पर क्रमिक रूप से हुक जोड़ें।

    साझा हेल्पर बिल्डर अब सबसे सामान्य रीप्ले/टूल-संगतता
    परिवारों को समेटते हैं, इसलिए Plugin को सामान्यतः प्रत्येक हुक को एक-एक करके स्वयं जोड़ने
    की आवश्यकता नहीं होती:

    ```typescript
    import { buildProviderReplayFamilyHooks } from "openclaw/plugin-sdk/provider-model-shared";
    import { buildProviderStreamFamilyHooks } from "openclaw/plugin-sdk/provider-stream";
    import { buildProviderToolCompatFamilyHooks } from "openclaw/plugin-sdk/provider-tools";

    const GOOGLE_FAMILY_HOOKS = {
      ...buildProviderReplayFamilyHooks({ family: "google-gemini" }),
      ...buildProviderStreamFamilyHooks("google-thinking"),
      ...buildProviderToolCompatFamilyHooks("gemini"),
    };

    api.registerProvider({
      id: "acme-gemini-compatible",
      // ...
      ...GOOGLE_FAMILY_HOOKS,
    });
    ```

    वर्तमान में उपलब्ध रीप्ले परिवार:

    | परिवार | यह क्या जोड़ता है | बंडल किए गए उदाहरण |
    | --- | --- | --- |
    | `openai-compatible` | OpenAI-संगत परिवहन के लिए साझा OpenAI-शैली की रीप्ले नीति, जिसमें टूल-कॉल-id स्वच्छीकरण, सहायक-प्रथम क्रम सुधार और जहाँ परिवहन को आवश्यकता हो वहाँ सामान्य Gemini-टर्न सत्यापन शामिल हैं | `moonshot`, `ollama`, `xai`, `zai` |
    | `anthropic-by-model` | `modelId` द्वारा चुनी गई Claude-जागरूक रीप्ले नीति, ताकि Anthropic-संदेश परिवहन को Claude-विशिष्ट थिंकिंग-ब्लॉक सफ़ाई केवल तभी मिले, जब समाधान किया गया मॉडल वास्तव में Claude id हो | `amazon-bedrock` |
    | `native-anthropic-by-model` | `anthropic-by-model` जैसी ही मॉडल-आधारित Claude नीति, साथ में उन परिवहनों के लिए टूल-कॉल-id स्वच्छीकरण और नेटिव Anthropic टूल-उपयोग id का संरक्षण, जिन्हें विक्रेता-नेटिव id बनाए रखना आवश्यक है | `anthropic-vertex`, `clawrouter` |
    | `google-gemini` | नेटिव Gemini रीप्ले नीति और बूटस्ट्रैप रीप्ले स्वच्छीकरण। साझा परिवार पाठ-आउटपुट Gemini CLI को टैग किए गए रीजनिंग पर रखता है; प्रत्यक्ष `google` प्रदाता `resolveReasoningOutputMode` को `native` से ओवरराइड करता है, क्योंकि Gemini API थिंकिंग नेटिव विचार भागों के रूप में आती है। | `google`, `google-gemini-cli` |
    | `passthrough-gemini` | OpenAI-संगत प्रॉक्सी परिवहन के माध्यम से चलने वाले Gemini मॉडल के लिए Gemini विचार-हस्ताक्षर स्वच्छीकरण; यह नेटिव Gemini रीप्ले सत्यापन या बूटस्ट्रैप पुनर्लेखन सक्षम नहीं करता | `openrouter`, `kilocode`, `opencode`, `opencode-go` |
    | `hybrid-anthropic-openai` | ऐसे प्रदाताओं के लिए हाइब्रिड नीति, जो एक Plugin में Anthropic-संदेश और OpenAI-संगत मॉडल सतहों को मिलाते हैं; वैकल्पिक केवल-Claude थिंकिंग-ब्लॉक हटाना Anthropic पक्ष तक सीमित रहता है | `minimax` |

    वर्तमान में उपलब्ध स्ट्रीम परिवार:

    | परिवार | यह किसे जोड़ता है | बंडल किए गए उदाहरण |
    | --- | --- | --- |
    | `google-thinking` | साझा स्ट्रीम पथ पर Gemini थिंकिंग पेलोड सामान्यीकरण | `google`, `google-gemini-cli` |
    | `kilocode-thinking` | साझा प्रॉक्सी स्ट्रीम पथ पर Kilo रीजनिंग रैपर, जिसमें `kilo-auto/balanced` और असमर्थित प्रॉक्सी रीजनिंग आईडी इंजेक्ट की गई थिंकिंग को छोड़ देते हैं | `kilocode` |
    | `moonshot-thinking` | कॉन्फ़िगरेशन + `/think` स्तर से Moonshot बाइनरी नेटिव-थिंकिंग पेलोड मैपिंग | `moonshot` |
    | `minimax-fast-mode` | साझा स्ट्रीम पथ पर MiniMax फ़ास्ट-मोड मॉडल पुनर्लेखन | `minimax`, `minimax-portal` |
    | `openai-responses-defaults` | साझा नेटिव OpenAI/Codex Responses रैपर: एट्रिब्यूशन हेडर, `/fast`/`serviceTier`, टेक्स्ट वर्बोसिटी, नेटिव Codex वेब खोज, रीजनिंग-संगत पेलोड संरचना, और Responses संदर्भ प्रबंधन | `openai` |
    | `openrouter-thinking` | प्रॉक्सी रूट के लिए OpenRouter रीजनिंग रैपर, जिसमें असमर्थित-मॉडल/`auto` स्किप केंद्रीय रूप से संभाले जाते हैं | `openrouter` |
    | `tool-stream-default-on` | Z.AI जैसे प्रदाताओं के लिए डिफ़ॉल्ट रूप से चालू `tool_stream` रैपर, जिन्हें स्पष्ट रूप से अक्षम किए जाने तक टूल स्ट्रीमिंग चाहिए | `zai` |

    <Accordion title="फ़ैमिली बिल्डर को शक्ति देने वाले SDK सीम">
      प्रत्येक फ़ैमिली बिल्डर उसी पैकेज से निर्यात किए गए निम्न-स्तरीय सार्वजनिक हेल्पर से बना है, जिनका उपयोग तब किया जा सकता है जब किसी प्रदाता को सामान्य पैटर्न से अलग जाना हो:

      - `openclaw/plugin-sdk/provider-model-shared` - `ProviderReplayFamily`, `buildProviderReplayFamilyHooks(...)`, और रॉ रीप्ले बिल्डर (`buildOpenAICompatibleReplayPolicy`, `buildAnthropicReplayPolicyForModel`, `buildGoogleGeminiReplayPolicy`, `buildHybridAnthropicOrOpenAIReplayPolicy`)। Gemini रीप्ले हेल्पर (`sanitizeGoogleGeminiReplayHistory`, `resolveTaggedReasoningOutputMode`) और एंडपॉइंट/मॉडल हेल्पर (`resolveProviderEndpoint`, `normalizeProviderId`, `normalizeGooglePreviewModelId`) भी निर्यात करता है।
      - `openclaw/plugin-sdk/provider-stream` - `ProviderStreamFamily`, `buildProviderStreamFamilyHooks(...)`, `composeProviderStreamWrappers(...)`, साथ ही साझा OpenAI/Codex रैपर (`createOpenAIAttributionHeadersWrapper`, `createOpenAIFastModeWrapper`, `createOpenAIServiceTierWrapper`, `createOpenAIResponsesContextManagementWrapper`, `createCodexNativeWebSearchWrapper`), DeepSeek V4 OpenAI-संगत रैपर (`createDeepSeekV4OpenAICompatibleThinkingWrapper`), Anthropic Messages थिंकिंग प्रीफ़िल क्लीनअप (`createAnthropicThinkingPrefillPayloadWrapper`), प्लेन-टेक्स्ट टूल-कॉल संगतता (`createPlainTextToolCallCompatWrapper`), और साझा प्रॉक्सी/प्रदाता रैपर (`createOpenRouterWrapper`, `createToolStreamWrapper`, `createMinimaxFastModeWrapper`)।
      - `openclaw/plugin-sdk/provider-stream-shared` - हॉट प्रदाता पथों के लिए हल्के पेलोड और इवेंट रैपर, जिनमें `createOpenAICompatibleCompletionsThinkingOffWrapper`, `createPayloadPatchStreamWrapper`, `createPlainTextToolCallCompatWrapper`, `normalizeOpenAICompatibleReasoningPayload(...)`, और `setQwenChatTemplateThinking(...)` शामिल हैं।
      - `openclaw/plugin-sdk/provider-tools` - `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks("deepseek" | "gemini" | "openai")`, और अंतर्निहित प्रदाता स्कीमा हेल्पर।

      Gemini-फ़ैमिली प्रदाताओं के लिए, रीजनिंग-आउटपुट मोड को
      ट्रांसपोर्ट के अनुरूप रखें। प्रत्यक्ष Google Gemini API प्रदाताओं को `native`
      रीजनिंग आउटपुट का उपयोग करना चाहिए, ताकि OpenClaw
      `<think>` / `<final>` प्रॉम्प्ट निर्देश जोड़े बिना नेटिव थॉट पार्ट का उपयोग कर सके। केवल-टेक्स्ट वाले Gemini CLI-शैली
      बैकएंड, जो अंतिम JSON/टेक्स्ट प्रतिक्रिया को पार्स करते हैं, साझा
      `google-gemini` टैग किए गए अनुबंध को बनाए रख सकते हैं।

      कुछ स्ट्रीम हेल्पर जानबूझकर प्रदाता-स्थानीय रहते हैं। `@openclaw/anthropic-provider` अपने सार्वजनिक `api.ts` / `contract-api.ts` सीम में `wrapAnthropicProviderStream`, `resolveAnthropicBetas`, `resolveAnthropicFastMode`, `resolveAnthropicServiceTier`, और निम्न-स्तरीय Anthropic रैपर बिल्डर रखता है, क्योंकि वे Claude OAuth बीटा हैंडलिंग और `context1m` गेटिंग को एन्कोड करते हैं। इसी तरह xAI Plugin नेटिव xAI Responses संरचना को अपने `wrapStreamFn` (`/fast` उपनाम, डिफ़ॉल्ट `tool_stream`, असमर्थित स्ट्रिक्ट-टूल क्लीनअप, xAI-विशिष्ट रीजनिंग-पेलोड निष्कासन) में रखता है।

      यही पैकेज-रूट पैटर्न `@openclaw/openai-provider` (प्रदाता बिल्डर, डिफ़ॉल्ट-मॉडल हेल्पर, रीयलटाइम प्रदाता बिल्डर) और `@openclaw/openrouter-provider` (प्रदाता बिल्डर तथा ऑनबोर्डिंग/कॉन्फ़िगरेशन हेल्पर) को भी आधार देता है।
    </Accordion>

    <Tabs>
      <Tab title="टोकन एक्सचेंज">
        उन प्रदाताओं के लिए जिन्हें प्रत्येक इन्फ़रेंस कॉल से पहले टोकन एक्सचेंज की आवश्यकता होती है:

        ```typescript
        prepareRuntimeAuth: async (ctx) => {
          const exchanged = await exchangeToken(ctx.apiKey);
          return {
            apiKey: exchanged.token,
            baseUrl: exchanged.baseUrl,
            expiresAt: exchanged.expiresAt,
          };
        },
        ```
      </Tab>
      <Tab title="कस्टम हेडर">
        उन प्रदाताओं के लिए जिन्हें कस्टम अनुरोध हेडर या बॉडी संशोधन चाहिए:

        ```typescript
        // wrapStreamFn, ctx.streamFn से व्युत्पन्न StreamFn लौटाता है
        wrapStreamFn: (ctx) => {
          if (!ctx.streamFn) return undefined;
          const inner = ctx.streamFn;
          return async (params) => {
            params.headers = {
              ...params.headers,
              "X-Acme-Version": "2",
            };
            return inner(params);
          };
        },
        ```
      </Tab>
      <Tab title="नेटिव ट्रांसपोर्ट पहचान">
        उन प्रदाताओं के लिए जिन्हें जेनेरिक HTTP या WebSocket ट्रांसपोर्ट पर
        नेटिव अनुरोध/सत्र हेडर या मेटाडेटा चाहिए:

        ```typescript
        resolveTransportTurnState: (ctx) => ({
          headers: {
            "x-request-id": ctx.turnId,
          },
          metadata: {
            session_id: ctx.sessionId ?? "",
            turn_id: ctx.turnId,
          },
        }),
        resolveWebSocketSessionPolicy: (ctx) => ({
          headers: {
            "x-session-id": ctx.sessionId ?? "",
          },
          degradeCooldownMs: 60_000,
        }),
        ```
      </Tab>
      <Tab title="उपयोग और बिलिंग">
        उपयोग/बिलिंग डेटा उपलब्ध कराने वाले प्रदाताओं के लिए:

        ```typescript
        resolveUsageAuth: async (ctx) => {
          const auth = await ctx.resolveOAuthToken();
          return auth ? { token: auth.token } : null;
        },
        fetchUsageSnapshot: async (ctx) => {
          return await fetchAcmeUsage(ctx.token, ctx.timeoutMs);
        },
        ```

        `resolveUsageAuth` के तीन परिणाम होते हैं। जब
        प्रदाता के पास उपयोग/बिलिंग क्रेडेंशियल हो, तब
        `{ token, accountId?, subscriptionType?, rateLimitTier? }` लौटाएँ (वैकल्पिक फ़ील्ड समाधान की गई प्रोफ़ाइल से
        गैर-गोपनीय प्लान मेटाडेटा को
        `fetchUsageSnapshot` में ले जाते हैं)। `{ handled: true }` केवल तभी लौटाएँ जब प्रदाता ने उपयोग
        प्रमाणीकरण को निश्चित रूप से संभाला हो, लेकिन उसके पास उपयोग योग्य उपयोग टोकन न हो, और OpenClaw को जेनेरिक
        API-कुंजी/OAuth फ़ॉलबैक छोड़ना आवश्यक हो। जब प्रदाता ने
        अनुरोध नहीं संभाला हो और OpenClaw को जेनेरिक फ़ॉलबैक जारी रखना चाहिए, तब `null` या `undefined` लौटाएँ।

        प्रदाता आईडी को `contracts.usageProviders` में घोषित करें। जब वह मैनिफ़ेस्ट
        अनुबंध और **दोनों** हुक मौजूद हों, तो OpenClaw असंबंधित प्रदाता
        Plugins लोड किए बिना प्रदाता को उपयोग संग्रह में स्वतः शामिल कर देता है।
        किसी कोर अलावलिस्ट अपडेट की आवश्यकता नहीं होती।
        `fetchUsageSnapshot` साझा प्रदाता-निरपेक्ष संरचना लौटाता है:

        - `plan`: प्रदाता द्वारा रिपोर्ट किया गया सब्सक्रिप्शन या कुंजी लेबल
        - `windows`: उपयोग किए गए प्रतिशत के रूप में रीसेट किए जा सकने वाले कोटा विंडो
        - `billing`: टाइप किए गए `balance`, `spend`, या `budget` प्रविष्टियाँ; `unit`
          एक ISO मुद्रा या `credits` जैसी प्रदाता इकाई हो सकती है
        - `summary`: संक्षिप्त प्रदाता-विशिष्ट संदर्भ जो उन
          संरचित फ़ील्ड में समाहित नहीं होता

        मुद्रा का अर्थ सटीक रखें। जब तक अपस्ट्रीम
        अनुबंध ऐसा न कहे, प्रदाता क्रेडिट USD नहीं होता। केवल
        `fetchUsageSnapshot` लागू करने वाला Plugin स्पष्ट/सिंथेटिक कॉलर के लिए उपलब्ध रहता है, लेकिन
        स्वतः खोजा नहीं जाता, क्योंकि OpenClaw उसके उपयोग क्रेडेंशियल को हल नहीं कर सकता।
      </Tab>
    </Tabs>

    <Accordion title="सामान्य प्रदाता हुक">
      OpenClaw मॉडल/प्रदाता Plugins के लिए हुक को लगभग इसी क्रम में कॉल करता है।
      अधिकांश प्रदाता केवल 2-3 का उपयोग करते हैं। यह संपूर्ण `ProviderPlugin`
      अनुबंध नहीं है—पूर्ण और वर्तमान में सटीक हुक सूची तथा फ़ॉलबैक टिप्पणियों के लिए [आंतरिक संरचना: प्रदाता रनटाइम
      हुक](/hi/plugins/architecture-internals#provider-runtime-hooks) देखें।
      केवल-संगतता वाले प्रदाता फ़ील्ड, जिन्हें OpenClaw अब कॉल नहीं करता, जैसे
      `ProviderPlugin.capabilities` और `suppressBuiltInModel`, यहाँ
      सूचीबद्ध नहीं हैं।

      | हुक | कब उपयोग करें |
      | --- | --- |
      | `catalog` | मॉडल कैटलॉग या बेस URL डिफ़ॉल्ट |
      | `applyConfigDefaults` | कॉन्फ़िगरेशन मटेरियलाइज़ेशन के दौरान प्रदाता-स्वामित्व वाले वैश्विक डिफ़ॉल्ट |
      | `normalizeModelId` | लुकअप से पहले लेगेसी/प्रीव्यू मॉडल-आईडी उपनाम क्लीनअप |
      | `normalizeTransport` | जेनेरिक मॉडल असेंबली से पहले प्रदाता-फ़ैमिली `api` / `baseUrl` क्लीनअप |
      | `normalizeConfig` | `models.providers.<id>` कॉन्फ़िगरेशन को सामान्यीकृत करें |
      | `applyNativeStreamingUsageCompat` | कॉन्फ़िगरेशन प्रदाताओं के लिए नेटिव स्ट्रीमिंग-उपयोग संगतता पुनर्लेखन |
      | `resolveConfigApiKey` | प्रदाता-स्वामित्व वाला एनवायरनमेंट-मार्कर प्रमाणीकरण समाधान |
      | `resolveSyntheticAuth` | स्थानीय/स्वयं-होस्टेड या कॉन्फ़िगरेशन-समर्थित सिंथेटिक प्रमाणीकरण |
      | `resolveExternalAuthProfiles` | CLI/ऐप-प्रबंधित क्रेडेंशियल के लिए प्रदाता-स्वामित्व वाली बाहरी प्रमाणीकरण प्रोफ़ाइल को ओवरले करें |
      | `shouldDeferSyntheticProfileAuth` | एनवायरनमेंट/कॉन्फ़िगरेशन प्रमाणीकरण के पीछे सिंथेटिक संग्रहीत-प्रोफ़ाइल प्लेसहोल्डर को कम प्राथमिकता दें |
      | `resolveDynamicModel` | मनमाने अपस्ट्रीम मॉडल आईडी स्वीकार करें |
      | `prepareDynamicModel` | समाधान से पहले एसिंक्रोनस मेटाडेटा फ़ेच |
      | `normalizeResolvedModel` | रनर से पहले ट्रांसपोर्ट पुनर्लेखन |
      | `normalizeToolSchemas` | पंजीकरण से पहले प्रदाता-स्वामित्व वाला टूल-स्कीमा क्लीनअप |
      | `inspectToolSchemas` | प्रदाता-स्वामित्व वाला टूल-स्कीमा निदान |
      | `resolveReasoningOutputMode` | टैग किया गया बनाम नेटिव रीजनिंग-आउटपुट अनुबंध |
      | `prepareExtraParams` | डिफ़ॉल्ट अनुरोध पैरामीटर |
      | `createStreamFn` | पूर्णतः कस्टम StreamFn ट्रांसपोर्ट |
      | `wrapStreamFn` | सामान्य स्ट्रीम पथ पर कस्टम हेडर/बॉडी रैपर |
      | `resolveTransportTurnState` | नेटिव प्रति-टर्न हेडर/मेटाडेटा |
      | `resolveWebSocketSessionPolicy` | नेटिव WS सत्र हेडर/कूल-डाउन |
      | `formatApiKey` | कस्टम रनटाइम टोकन संरचना |
      | `refreshOAuth` | कस्टम OAuth रिफ़्रेश |
      | `buildAuthDoctorHint` | प्रमाणीकरण सुधार मार्गदर्शन |
      | `matchesContextOverflowError` | प्रदाता-स्वामित्व वाली ओवरफ़्लो पहचान |
      | `classifyFailoverReason` | प्रदाता-स्वामित्व वाला रेट-लिमिट/ओवरलोड वर्गीकरण |
      | `isCacheTtlEligible` | प्रॉम्प्ट कैश TTL गेटिंग |
      | `buildMissingAuthMessage` | कस्टम अनुपस्थित-प्रमाणीकरण संकेत |
      | `augmentModelCatalog` | सिंथेटिक फ़ॉरवर्ड-संगतता पंक्तियाँ (अप्रचलित—`registerModelCatalogProvider` को प्राथमिकता दें) |
      | `resolveThinkingProfile` | मॉडल-विशिष्ट `/think` विकल्प सेट |
      | `isBinaryThinking` | बाइनरी थिंकिंग चालू/बंद संगतता (अप्रचलित—`resolveThinkingProfile` को प्राथमिकता दें) |
      | `supportsXHighThinking` | `xhigh` रीजनिंग समर्थन संगतता (अप्रचलित—`resolveThinkingProfile` को प्राथमिकता दें) |
      | `resolveDefaultThinkingLevel` | डिफ़ॉल्ट `/think` नीति संगतता (अप्रचलित—`resolveThinkingProfile` को प्राथमिकता दें) |
      | `isModernModelRef` | लाइव/स्मोक मॉडल मिलान |
      | `prepareRuntimeAuth` | इन्फ़रेंस से पहले टोकन एक्सचेंज |
      | `resolveUsageAuth` | कस्टम उपयोग क्रेडेंशियल पार्सिंग |
      | `fetchUsageSnapshot` | कस्टम उपयोग एंडपॉइंट |
      | `createEmbeddingProvider` | मेमोरी/खोज के लिए प्रदाता-स्वामित्व वाला एम्बेडिंग एडाप्टर |
      | `buildReplayPolicy` | कस्टम ट्रांसक्रिप्ट रीप्ले/Compaction नीति |
      | `sanitizeReplayHistory` | जेनेरिक क्लीनअप के बाद प्रदाता-विशिष्ट रीप्ले पुनर्लेखन |
      | `validateReplayTurns` | एम्बेडेड रनर से पहले सख्त रीप्ले-टर्न सत्यापन |
      | `onModelSelected` | चयन-पश्चात कॉलबैक (जैसे टेलीमेट्री) |

      रनटाइम फ़ॉलबैक टिप्पणियाँ:

      - `normalizeConfig` प्रत्येक provider id के लिए एक स्वामी plugin को निर्धारित करता है (पहले बंडल किए गए providers, फिर मेल खाने वाला runtime plugin) और केवल उसी hook को कॉल करता है—अन्य providers में कोई स्कैन नहीं होता। Google का अपना `normalizeConfig` hook ही `google` / `google-vertex` / `google-antigravity` config प्रविष्टियों को सामान्यीकृत करता है; यह कोई अलग core fallback नहीं है।
      - `resolveConfigApiKey` उपलब्ध होने पर provider hook का उपयोग करता है। Amazon Bedrock अपने provider plugin में AWS env-marker समाधान रखता है; `auth: "aws-sdk"` के साथ कॉन्फ़िगर किए जाने पर runtime auth अब भी AWS SDK की डिफ़ॉल्ट श्रृंखला का उपयोग करता है।
      - `resolveThinkingProfile(ctx)` चयनित `provider`, `modelId`, वैकल्पिक रूप से मर्ज किया गया `reasoning` कैटलॉग संकेत और वैकल्पिक रूप से मर्ज किए गए मॉडल के `compat` तथ्य प्राप्त करता है। `compat` का उपयोग केवल provider की thinking UI/profile चुनने के लिए करें।
      - `resolveSystemPromptContribution` किसी provider को एक मॉडल परिवार के लिए कैश-जागरूक system-prompt मार्गदर्शन इंजेक्ट करने देता है। जब व्यवहार किसी एक provider/model परिवार से संबंधित हो और स्थिर/गतिशील कैश विभाजन को बनाए रखना चाहिए, तब पुराने plugin-व्यापी `before_prompt_build` hook के बजाय इसे प्राथमिकता दें।

    </Accordion>

  </Step>

  <Step title="अतिरिक्त क्षमताएँ जोड़ें (वैकल्पिक)">
    ### चरण 5: अतिरिक्त क्षमताएँ जोड़ें

    एक provider plugin टेक्स्ट inference के साथ embeddings, speech, realtime transcription,
    realtime voice, media understanding, image generation, video generation,
    web fetch और web search पंजीकृत कर सकता है। OpenClaw इसे
    **hybrid-capability** plugin के रूप में वर्गीकृत करता है—कंपनी plugins के लिए अनुशंसित पैटर्न
    (प्रति vendor एक plugin)। देखें
    [आंतरिक संरचना: क्षमता स्वामित्व](/hi/plugins/architecture#capability-ownership-model)।

    अपनी मौजूदा `api.registerProvider(...)` कॉल के साथ `register(api)` के भीतर
    प्रत्येक क्षमता पंजीकृत करें। केवल आवश्यक टैब चुनें:

    <Tabs>
      <Tab title="स्पीच (TTS)">
        ```typescript
        import {
          assertOkOrThrowProviderError,
          postJsonRequest,
        } from "openclaw/plugin-sdk/provider-http";

        api.registerSpeechProvider({
          id: "acme-ai",
          label: "Acme Speech",
          defaultTimeoutMs: 120_000,
          isConfigured: ({ config }) => Boolean(config.messages?.tts),
          synthesize: async (req) => {
            const { response, release } = await postJsonRequest({
              url: "https://api.example.com/v1/speech",
              headers: new Headers({ "Content-Type": "application/json" }),
              body: { text: req.text },
              timeoutMs: req.timeoutMs,
              fetchFn: fetch,
              auditContext: "acme speech",
            });
            try {
              await assertOkOrThrowProviderError(response, "Acme Speech API error");
              return {
                audioBuffer: Buffer.from(await response.arrayBuffer()),
                outputFormat: "mp3",
                fileExtension: ".mp3",
                voiceCompatible: false,
              };
            } finally {
              await release();
            }
          },
        });
        ```

        provider HTTP विफलताओं के लिए `assertOkOrThrowProviderError(...)` का उपयोग करें, ताकि
        plugins सीमित error-body रीड, JSON त्रुटि पार्सिंग और
        request-id प्रत्यय साझा करें।
      </Tab>
      <Tab title="रीयलटाइम ट्रांसक्रिप्शन">
        `createRealtimeTranscriptionWebSocketSession(...)` को प्राथमिकता दें—साझा
        सहायक proxy कैप्चर, reconnect backoff, close flushing, ready
        handshakes, ऑडियो कतारबद्ध करना और close-event निदान संभालता है। आपका plugin
        केवल upstream events को मैप करता है।

        ```typescript
        api.registerRealtimeTranscriptionProvider({
          id: "acme-ai",
          label: "Acme Realtime Transcription",
          isConfigured: () => true,
          createSession: (req) => {
            const apiKey = String(req.providerConfig.apiKey ?? "");
            return createRealtimeTranscriptionWebSocketSession({
              providerId: "acme-ai",
              callbacks: req,
              url: "wss://api.example.com/v1/realtime-transcription",
              headers: { Authorization: `Bearer ${apiKey}` },
              onMessage: (event, transport) => {
                if (event.type === "session.created") {
                  transport.sendJson({ type: "session.update" });
                  transport.markReady();
                  return;
                }
                if (event.type === "transcript.final") {
                  req.onTranscript?.(event.text);
                }
              },
              sendAudio: (audio, transport) => {
                transport.sendJson({
                  type: "audio.append",
                  audio: audio.toString("base64"),
                });
              },
              onClose: (transport) => {
                transport.sendJson({ type: "audio.end" });
              },
            });
          },
        });
        ```

        multipart ऑडियो POST करने वाले बैच STT providers को
        `openclaw/plugin-sdk/provider-http` से
        `buildAudioTranscriptionFormData(...)` का उपयोग करना चाहिए। सहायक अपलोड
        फ़ाइल नामों को सामान्यीकृत करता है, जिसमें वे AAC अपलोड भी शामिल हैं जिन्हें
        संगत transcription APIs के लिए M4A-शैली के फ़ाइल नाम की आवश्यकता होती है।
      </Tab>
      <Tab title="रीयलटाइम वॉइस">
        ```typescript
        api.registerRealtimeVoiceProvider({
          id: "acme-ai",
          label: "Acme Realtime Voice",
          capabilities: {
            transports: ["gateway-relay"],
            inputAudioFormats: [{ encoding: "pcm16", sampleRateHz: 24000, channels: 1 }],
            outputAudioFormats: [{ encoding: "pcm16", sampleRateHz: 24000, channels: 1 }],
            supportsBargeIn: true,
            handlesInputAudioBargeIn: true,
            supportsToolCalls: true,
          },
          isConfigured: ({ providerConfig }) => Boolean(providerConfig.apiKey),
          createBridge: (req) => ({
            // इसे केवल तभी सेट करें जब provider एक कॉल के लिए कई tool responses
            // स्वीकार करता हो, उदाहरण के लिए तत्काल "काम जारी है" response के बाद
            // अंतिम परिणाम।
            supportsToolResultContinuation: false,
            connect: async () => {},
            sendAudio: () => {},
            setMediaTimestamp: () => {},
            handleBargeIn: () => {},
            submitToolResult: () => {},
            acknowledgeMark: () => {},
            close: () => {},
            isConnected: () => true,
          }),
        });
        ```

        `capabilities` घोषित करें, ताकि `talk.catalog` ब्राउज़र और नेटिव Talk
        clients के लिए मान्य modes, transports, ऑडियो formats और feature flags उपलब्ध करा सके।
        जब कोई transport यह पता लगा सकता हो कि कोई व्यक्ति assistant playback में
        बाधा डाल रहा है और provider सक्रिय ऑडियो response को छोटा करने या साफ़ करने का
        समर्थन करता हो, तब `handleBargeIn` लागू करें।
        `submitToolResult` समकालिक सबमिशन के लिए `void`, या provider
        bridge द्वारा उपलब्ध कराई जा सकने वाली अतुल्यकालिक पूर्णता सीमा के लिए
        `Promise<void>` लौटा सकता है। Gateway relay sessions अंतिम परिणाम की
        पुष्टि करने या लिंक किए गए run को साफ़ करने से पहले उस promise की प्रतीक्षा करते हैं;
        सबमिशन विफल होने पर उसे अस्वीकार करें।
        जब provider `options.suppressResponse` का पालन नहीं कर सकता हो, तब
        `supportsToolResultSuppression: false` सेट करें। इसके बाद OpenClaw आंतरिक forced-consult और
        cancellation परिणामों के लिए suppression से बचता है तथा चुपचाप response शुरू करने के
        बजाय सीधे suppressed-result अनुरोधों को अस्वीकार करता है।
        `createRealtimeVoiceBridgeSession` के उपभोक्ता इसी प्रकार `onToolCall` से
        promise लौटा सकते हैं; समकालिक throws और rejections को session के
        `onError` callback पर भेजा जाता है।
        `handlesInputAudioBargeIn` केवल तभी सेट करें जब provider VAD
        `onClearAudio("barge-in")` को कॉल करके किसी बाधा की पुष्टि करता हो। जो providers
        यह flag छोड़ देते हैं, वे OpenClaw की स्थानीय input-audio fallback पहचान का उपयोग करते हैं।
      </Tab>
      <Tab title="मीडिया समझ">
        ```typescript
        api.registerMediaUnderstandingProvider({
          id: "acme-ai",
          capabilities: ["image", "audio"],
          describeImage: async (req) => ({ text: "इसका एक फ़ोटो..." }),
          transcribeAudio: async (req) => ({ text: "प्रतिलेख..." }),
        });
        ```

        स्थानीय या स्वयं-होस्ट किए गए media providers, जिन्हें जानबूझकर
        credentials की आवश्यकता नहीं होती, `resolveAuth` उपलब्ध करा सकते हैं और
        `kind: "none"` लौटा सकते हैं। जो providers स्पष्ट रूप से opt in नहीं करते,
        उनके लिए OpenClaw सामान्य auth gate बनाए रखता है। मौजूदा providers
        `req.apiKey` पढ़ना जारी रख सकते हैं; नए providers को
        `req.auth` प्राथमिकता देनी चाहिए।

        ```typescript
        api.registerMediaUnderstandingProvider({
          id: "local-audio",
          capabilities: ["audio"],
          resolveAuth: () => ({
            kind: "none",
            source: "local-audio plugin no-auth",
          }),
          transcribeAudio: async (req) => ({ text: "प्रतिलेख..." }),
        });
        ```
      </Tab>
      <Tab title="एम्बेडिंग्स">
        ```typescript
        api.registerEmbeddingProvider({
          id: "acme-ai",
          defaultModel: "acme-embed",
          transport: "remote",
          authProviderId: "acme-ai",
          create: async ({ model }) => ({
            provider: {
              id: "acme-ai",
              model,
              dimensions: 1536,
              embed: async (input) => {
                const text = typeof input === "string" ? input : input.text;
                return fetchAcmeEmbedding(text);
              },
              embedBatch: async (inputs) =>
                Promise.all(
                  inputs.map((input) =>
                    fetchAcmeEmbedding(typeof input === "string" ? input : input.text),
                  ),
                ),
            },
          }),
        });
        ```

        `contracts.embeddingProviders` में वही id घोषित करें। यह
        पुनः उपयोग योग्य vector generation के लिए सामान्य embedding contract है, जिसमें
        memory search भी शामिल है। `registerMemoryEmbeddingProvider(...)` मौजूदा
        memory-विशिष्ट adapters के लिए पदावनत compatibility है।
      </Tab>
      <Tab title="इमेज और वीडियो जनरेशन">
        इमेज और वीडियो क्षमताएँ **mode-aware** संरचना का उपयोग करती हैं। इमेज
        providers आवश्यक `generate` और `edit` capability blocks घोषित करते हैं;
        वीडियो providers `generate`, `imageToVideo` और
        `videoToVideo` घोषित करते हैं। `maxInputImages` /
        `maxInputVideos` / `maxDurationSeconds` जैसे flat aggregate fields,
        transform-mode समर्थन या अक्षम modes को स्पष्ट रूप से प्रदर्शित करने के लिए पर्याप्त नहीं हैं।
        म्यूज़िक जनरेशन भी इसी `generate` / `edit` पैटर्न का पालन करता है।

        ```typescript
        api.registerImageGenerationProvider({
          id: "acme-ai",
          label: "Acme छवियाँ",
          capabilities: {
            generate: { maxCount: 4, supportsSize: true },
            edit: { enabled: false },
          },
          generateImage: async (req) => ({ images: [] }),
        });

        api.registerVideoGenerationProvider({
          id: "acme-ai",
          label: "Acme वीडियो",
          defaultTimeoutMs: 600_000,
          models: ["acme-video", "acme-image-video"],
          capabilities: {
            generate: { maxVideos: 1, maxDurationSeconds: 10, supportsResolution: true },
            imageToVideo: {
              enabled: true,
              maxVideos: 1,
              maxInputImages: 1,
              maxInputImagesByModel: { "acme/reference-to-video": 9 },
              maxDurationSeconds: 5,
            },
            videoToVideo: { enabled: false },
          },
          catalogByModel: {
            "acme-image-video": {
              modes: ["imageToVideo"],
              capabilities: {
                imageToVideo: {
                  enabled: true,
                  maxVideos: 1,
                  maxInputImages: 1,
                  resolutions: ["480P", "720P", "1080P"],
                  supportsResolution: true,
                },
                videoToVideo: { enabled: false },
              },
            },
          },
          generateVideo: async (req) => ({ videos: [] }),
        });
        ```

        `capabilities` दोनों प्रदाता प्रकारों पर आवश्यक है; `edit` और
        वीडियो रूपांतरण ब्लॉक (`imageToVideo`, `videoToVideo`) के लिए हमेशा एक
        स्पष्ट `enabled` फ़्लैग आवश्यक होता है।

        जब किसी सूचीबद्ध मॉडल के स्थिर मोड या क्षमताएँ प्रदाता के डिफ़ॉल्ट से
        भिन्न हों, तब `catalogByModel` का उपयोग करें। यह मेटाडेटा प्रदाता कोड
        लागू किए बिना `video_generate action=list` और मॉडल कैटलॉग को सटीक
        रखता है। अनुरोध के समय क्षमता खोजना और उसे लागू करना
        अभी भी `resolveModelCapabilities` और `generateVideo` में होना चाहिए; संभव होने पर
        दोनों पथों के लिए समान क्षमता स्थिरांक का पुनः उपयोग करें।
      </Tab>
      <Tab title="वेब फ़ेच और खोज">
        ```typescript
        api.registerWebFetchProvider({
          id: "acme-ai-fetch",
          label: "Acme फ़ेच",
          hint: "Acme के रेंडरिंग बैकएंड के माध्यम से पृष्ठ फ़ेच करें।",
          envVars: ["ACME_FETCH_API_KEY"],
          placeholder: "acme-...",
          signupUrl: "https://acme.example.com/fetch",
          credentialPath: "plugins.entries.acme.config.webFetch.apiKey",
          getCredentialValue: (fetchConfig) => fetchConfig?.acme?.apiKey,
          setCredentialValue: (fetchConfigTarget, value) => {
            const acme = (fetchConfigTarget.acme ??= {});
            acme.apiKey = value;
          },
          createTool: () => ({
            description: "Acme फ़ेच के माध्यम से एक पृष्ठ फ़ेच करें।",
            parameters: {},
            execute: async (args) => ({ content: [] }),
          }),
        });

        api.registerWebSearchProvider({
          id: "acme-ai-search",
          label: "Acme खोज",
          hint: "Acme के खोज बैकएंड के माध्यम से वेब पर खोजें।",
          envVars: ["ACME_SEARCH_API_KEY"],
          placeholder: "acme-...",
          signupUrl: "https://acme.example.com/search",
          credentialPath: "plugins.entries.acme.config.webSearch.apiKey",
          getCredentialValue: (searchConfig) => searchConfig?.acme?.apiKey,
          setCredentialValue: (searchConfigTarget, value) => {
            const acme = (searchConfigTarget.acme ??= {});
            acme.apiKey = value;
          },
          createTool: () => ({
            description: "Acme खोज के माध्यम से वेब पर खोजें।",
            parameters: {},
            execute: async (args) => ({ content: [] }),
          }),
        });
        ```

        दोनों प्रदाता प्रकार समान क्रेडेंशियल-वायरिंग संरचना साझा करते हैं:
        `hint`, `envVars`, `placeholder`, `signupUrl`, `credentialPath`,
        `getCredentialValue`, `setCredentialValue`, और `createTool` सभी
        आवश्यक हैं।
      </Tab>
    </Tabs>

  </Step>

  <Step title="परीक्षण">
    ### चरण 6: परीक्षण

    ```typescript src/provider.test.ts
    import { describe, it, expect } from "vitest";
    // अपने प्रदाता कॉन्फ़िगरेशन ऑब्जेक्ट को index.ts या किसी समर्पित फ़ाइल से निर्यात करें
    import { acmeProvider } from "./provider.js";

    describe("acme-ai प्रदाता", () => {
      it("डायनेमिक मॉडल रिज़ॉल्व करता है", () => {
        const model = acmeProvider.resolveDynamicModel!({
          modelId: "acme-beta-v3",
        } as any);
        expect(model.id).toBe("acme-beta-v3");
        expect(model.provider).toBe("acme-ai");
      });

      it("कुंजी उपलब्ध होने पर कैटलॉग लौटाता है", async () => {
        const result = await acmeProvider.catalog!.run({
          resolveProviderApiKey: () => ({ apiKey: "test-key" }),
        } as any);
        expect(result?.provider?.models).toHaveLength(2);
      });

      it("कुंजी न होने पर null कैटलॉग लौटाता है", async () => {
        const result = await acmeProvider.catalog!.run({
          resolveProviderApiKey: () => ({ apiKey: undefined }),
        } as any);
        expect(result).toBeNull();
      });
    });
    ```

  </Step>
</Steps>

## ClawHub पर प्रकाशित करें

प्रदाता Plugins भी किसी अन्य बाहरी कोड Plugin की तरह ही प्रकाशित होते हैं:

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

`clawhub skill publish <path>` किसी skill फ़ोल्डर को प्रकाशित करने के लिए एक अलग कमांड है,
Plugin पैकेज के लिए नहीं—यहाँ इसका उपयोग न करें।

## फ़ाइल संरचना

```
<bundled-plugin-root>/acme-ai/
├── package.json              # openclaw.providers मेटाडेटा
├── openclaw.plugin.json      # प्रदाता प्रमाणीकरण मेटाडेटा सहित मैनिफ़ेस्ट
├── index.ts                  # definePluginEntry + registerProvider
└── src/
    ├── provider.test.ts      # परीक्षण
    └── usage.ts              # उपयोग एंडपॉइंट (वैकल्पिक)
```

## कैटलॉग क्रम संदर्भ

`catalog.order` यह नियंत्रित करता है कि आपका कैटलॉग अंतर्निहित
प्रदाताओं के सापेक्ष कब मर्ज होता है:

| क्रम      | कब            | उपयोग का मामला                                  |
| --------- | ------------- | ----------------------------------------------- |
| `simple`  | पहला चरण     | सामान्य API-कुंजी प्रदाता                       |
| `profile` | सरल के बाद   | प्रमाणीकरण प्रोफ़ाइल पर निर्भर प्रदाता            |
| `paired`  | प्रोफ़ाइल के बाद | एकाधिक संबंधित प्रविष्टियाँ संश्लेषित करना   |
| `late`    | अंतिम चरण    | मौजूदा प्रदाताओं को ओवरराइड करना (टकराव में जीतता है) |

## अगले चरण

- [चैनल Plugins](/hi/plugins/sdk-channel-plugins) - यदि आपका Plugin एक चैनल भी प्रदान करता है
- [SDK रनटाइम](/hi/plugins/sdk-runtime) - `api.runtime` सहायक (TTS, खोज, सबएजेंट)
- [SDK अवलोकन](/hi/plugins/sdk-overview) - संपूर्ण सबपाथ इम्पोर्ट संदर्भ
- [Plugin आंतरिक संरचना](/hi/plugins/architecture-internals#provider-runtime-hooks) - हुक विवरण और बंडल किए गए उदाहरण

## संबंधित

- [Plugin SDK सेटअप](/hi/plugins/sdk-setup)
- [Plugins बनाना](/hi/plugins/building-plugins)
- [चैनल Plugins बनाना](/hi/plugins/sdk-channel-plugins)
