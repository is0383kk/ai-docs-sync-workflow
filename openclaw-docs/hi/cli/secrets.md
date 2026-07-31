---
read_when:
    - रनटाइम पर सीक्रेट रेफ़रेंस को फिर से रिज़ॉल्व करना
    - प्लेनटेक्स्ट अवशेषों और अनसुलझे संदर्भों का ऑडिट करना
    - SecretRefs को कॉन्फ़िगर करना और एकतरफ़ा स्क्रब परिवर्तन लागू करना
summary: '`openclaw secrets` के लिए CLI संदर्भ (पुनः लोड, ऑडिट, कॉन्फ़िगर, लागू करें)'
title: गोपनीयताएँ
x-i18n:
    generated_at: "2026-07-27T20:40:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 61f6f81e358ca2e6a97ac9498186b32f7a74d16052d226c398dad0030d47211e
    source_path: cli/secrets.md
    workflow: 16
---

# `openclaw secrets`

SecretRefs प्रबंधित करें और सक्रिय रनटाइम स्नैपशॉट को स्वस्थ बनाए रखें।

| कमांड     | भूमिका                                                                                                                                                                                         |
| ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `reload`    | Gateway RPC (`secrets.reload`): रेफ़ को फिर से रिज़ॉल्व करता है और स्वामी-संवेदी रनटाइम स्नैपशॉट को परमाण्विक रूप से प्रकाशित करता है (कोई कॉन्फ़िगरेशन लेखन नहीं); पात्र स्वामी विफलताएँ कोल्ड या स्टेल चेतावनियों के रूप में प्रकाशित हो सकती हैं |
| `audit`     | प्लेनटेक्स्ट, अनरिज़ॉल्व्ड रेफ़ और प्राथमिकता विचलन के लिए कॉन्फ़िगरेशन/प्रमाणीकरण/जनरेटेड-मॉडल स्टोर तथा लीगेसी अवशेषों का केवल-पठन स्कैन (जब तक `--allow-exec` न हो, exec रेफ़ छोड़ दिए जाते हैं)                      |
| `configure` | प्रदाता सेटअप, लक्ष्य मैपिंग और प्रीफ़्लाइट के लिए इंटरैक्टिव प्लानर (TTY आवश्यक)                                                                                                       |
| `apply`     | सहेजी गई योजना निष्पादित करता है (`--dry-run` केवल सत्यापन करता है और डिफ़ॉल्ट रूप से exec जाँच छोड़ देता है; जब तक `--allow-exec` न हो, लेखन मोड exec वाली योजनाएँ अस्वीकार करता है), फिर लक्षित प्लेनटेक्स्ट अवशेष हटाता है |

अनुशंसित ऑपरेटर लूप:

```bash
openclaw secrets audit --check
openclaw secrets configure
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json
openclaw secrets audit --check
openclaw secrets reload
```

यदि आपकी योजना में `exec` SecretRefs/प्रदाता शामिल हैं, तो ड्राई-रन और लेखन वाले दोनों `apply` कमांड पर `--allow-exec` पास करें।

CI/गेट के लिए निकास कोड:

- `audit --check` निष्कर्ष मिलने पर `1` लौटाता है।
- अनरिज़ॉल्व्ड रेफ़ `2` लौटाते हैं (`--check` की परवाह किए बिना)।

संबंधित: [सीक्रेट प्रबंधन](/hi/gateway/secrets) · [SecretRef क्रेडेंशियल सतह](/hi/reference/secretref-credential-surface) · [सुरक्षा](/hi/gateway/security)

## रनटाइम स्नैपशॉट पुनः लोड करें

```bash
openclaw secrets reload
openclaw secrets reload --json
openclaw secrets reload --url ws://127.0.0.1:18789 --token <token>
```

Gateway RPC विधि `secrets.reload` का उपयोग करता है। स्वस्थ स्वामी स्वतंत्र रूप से रीफ़्रेश होते हैं। पात्र विफल स्वामी केवल तभी स्टेल होते हैं, जब उनकी रेफ़ पहचान, प्रदाता परिभाषाएँ और पूर्ण गैर-सीक्रेट स्वामी अनुबंध अपरिवर्तित हों; नई या परिवर्तित विफलताएँ कोल्ड हो जाती हैं। यह अवनत सक्रियण सफल होता है और `warningCount` की रिपोर्ट करता है। सख्त या अमैप्ड विफलताएँ त्रुटि लौटाती हैं और पहले सक्रिय स्नैपशॉट को सुरक्षित रखती हैं।

विकल्प: `--url <url>`, `--token <token>`, `--timeout <ms>`, `--json`।

## ऑडिट

निम्न के लिए OpenClaw स्थिति स्कैन करता है:

- प्लेनटेक्स्ट सीक्रेट संग्रहण
- अनरिज़ॉल्व्ड रेफ़
- प्राथमिकता विचलन (`openclaw.json` रेफ़ को छिपाने वाले `auth-profiles.json` क्रेडेंशियल)
- जनरेटेड `agents/*/agent/models.json` अवशेष (प्रदाता `apiKey` मान और संवेदनशील प्रदाता हेडर)
- लीगेसी अवशेष (लीगेसी प्रमाणीकरण स्टोर प्रविष्टियाँ, OAuth अनुस्मारक)

`.env` स्कैन प्रभावी स्थिति डायरेक्टरी और सक्रिय कॉन्फ़िगरेशन वाली डायरेक्टरी को कवर करता है। जब दोनों पथ एक ही फ़ाइल को इंगित करते हैं, तो उसे एक बार स्कैन किया जाता है।

संवेदनशील प्रदाता हेडर का पता लगाना नाम-ह्यूरिस्टिक पर आधारित है: यह उन हेडर को चिह्नित करता है जिनके नाम सामान्य प्रमाणीकरण/क्रेडेंशियल अंशों (`authorization`, `x-api-key`, `token`, `secret`, `password`, `credential`) से मेल खाते हैं।

```bash
openclaw secrets audit
openclaw secrets audit --check
openclaw secrets audit --json
openclaw secrets audit --allow-exec
```

रिपोर्ट संरचना:

- `status`: `clean | findings | unresolved`
- `resolution`: `refsChecked`, `skippedExecRefs`, `resolvabilityComplete`
- `summary`: `plaintextCount`, `unresolvedRefCount`, `shadowedRefCount`, `legacyResidueCount`
- निष्कर्ष कोड: `PLAINTEXT_FOUND`, `REF_UNRESOLVED`, `REF_SHADOWED`, `LEGACY_RESIDUE`

## कॉन्फ़िगर करें (इंटरैक्टिव सहायक)

प्रदाता और SecretRef परिवर्तन इंटरैक्टिव रूप से बनाएँ, प्रीफ़्लाइट चलाएँ और वैकल्पिक रूप से लागू करें:

```bash
openclaw secrets configure
openclaw secrets configure --plan-out /tmp/openclaw-secrets-plan.json
openclaw secrets configure --apply --yes
openclaw secrets configure --providers-only
openclaw secrets configure --skip-provider-setup
openclaw secrets configure --agent ops
openclaw secrets configure --json
```

प्रवाह: पहले प्रदाता सेटअप (`secrets.providers` उपनाम जोड़ें/संपादित करें/हटाएँ), फिर क्रेडेंशियल मैपिंग (फ़ील्ड चुनें, `{source, provider, id}` रेफ़ निर्दिष्ट करें), फिर प्रीफ़्लाइट और वैकल्पिक लागूकरण।

फ़्लैग:

- `--providers-only`: केवल `secrets.providers` कॉन्फ़िगर करें, क्रेडेंशियल मैपिंग छोड़ें
- `--skip-provider-setup`: प्रदाता सेटअप छोड़ें, क्रेडेंशियल को मौजूदा प्रदाताओं से मैप करें
- `--agent <id>`: `auth-profiles.json` लक्ष्य खोज और लेखन को एक एजेंट स्टोर तक सीमित करें
- `--allow-exec`: प्रीफ़्लाइट/लागूकरण के दौरान exec SecretRef जाँच की अनुमति दें (प्रदाता कमांड निष्पादित हो सकते हैं)

`--providers-only` और `--skip-provider-setup` को साथ उपयोग नहीं किया जा सकता।

टिप्पणियाँ:

- इंटरैक्टिव TTY आवश्यक है।
- चयनित एजेंट दायरे के लिए `openclaw.json` और `auth-profiles.json` में सीक्रेट वाले फ़ील्ड को लक्षित करता है; प्रामाणिक समर्थित सतह: [SecretRef क्रेडेंशियल सतह](/hi/reference/secretref-credential-surface)।
- पिकर प्रवाह में सीधे नई `auth-profiles.json` मैपिंग बनाने का समर्थन करता है।
- लागू करने से पहले प्रीफ़्लाइट रिज़ॉल्यूशन चलाता है।
- जनरेटेड योजनाओं में डिफ़ॉल्ट रूप से स्क्रब विकल्प सक्षम होते हैं (`scrubEnv`, `scrubAuthProfilesForProviderTargets`, `scrubLegacyAuthJson`)। स्क्रब किए गए प्लेनटेक्स्ट मानों के लिए लागूकरण एकतरफ़ा है।
- `--plan-out` ऐसी योजना बनाने से इनकार करता है जिसका UTF-8 क्रमबद्ध रूप 16 MiB (16,777,216 बाइट) से अधिक हो, जो `apply --from` इनपुट सीमा के अनुरूप है।
- `--apply` के बिना, CLI प्रीफ़्लाइट के बाद भी `Apply this plan now?` के लिए संकेत देता है।
- `--apply` के साथ (और `--yes` के बिना), CLI अपरिवर्तनीय माइग्रेशन की एक अतिरिक्त पुष्टि माँगता है।
- `--json` योजना + प्रीफ़्लाइट रिपोर्ट प्रिंट करता है, लेकिन फिर भी इंटरैक्टिव TTY आवश्यक है।

### Exec प्रदाता सुरक्षा

Homebrew इंस्टॉल प्रायः `/opt/homebrew/bin/*` के अंतर्गत सिमलिंक किए गए बाइनरी उपलब्ध कराते हैं। विश्वसनीय पैकेज-मैनेजर पथों के लिए आवश्यकता होने पर ही `allowSymlinkCommand: true` सेट करें और इसे `trustedDirs` (उदाहरण के लिए `["/opt/homebrew"]`) के साथ जोड़ें। Windows पर, यदि किसी प्रदाता पथ के लिए ACL सत्यापन उपलब्ध नहीं है, तो OpenClaw सुरक्षित रूप से विफल हो जाता है; केवल विश्वसनीय पथों के लिए, पथ सुरक्षा जाँच को बायपास करने हेतु उस प्रदाता पर `allowInsecurePath: true` सेट करें।

## सहेजी गई योजना लागू करें

```bash
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --json
```

`--dry-run` फ़ाइलें लिखे बिना प्रीफ़्लाइट सत्यापित करता है; ड्राई-रन में exec SecretRef जाँच डिफ़ॉल्ट रूप से छोड़ दी जाती है। जब तक `--allow-exec` न हो, लेखन मोड exec SecretRefs/प्रदाता वाली योजनाएँ अस्वीकार करता है। किसी भी मोड में exec प्रदाता जाँच/निष्पादन के लिए सहमति देने हेतु `--allow-exec` का उपयोग करें।

`--from` को 16 MiB (16,777,216 बाइट) से बड़ी नहीं होने वाली नियमित फ़ाइल की ओर इंगित करना चाहिए। बाइट सीमा रिक्त स्थान सहित पूरी क्रमबद्ध फ़ाइल पर लागू होती है।

`apply` क्या अपडेट कर सकता है:

- `openclaw.json` (SecretRef लक्ष्य + प्रदाता अपसर्ट/हटाना)
- `auth-profiles.json` (प्रदाता-लक्ष्य स्क्रबिंग)
- लीगेसी `auth.json` अवशेष
- प्रभावी स्थिति और सक्रिय-कॉन्फ़िगरेशन डायरेक्टरी में `.env` फ़ाइलें, उन ज्ञात सीक्रेट कुंजियों के लिए जिनके मान माइग्रेट किए गए थे

योजना अनुबंध विवरण (अनुमत लक्ष्य पथ, सत्यापन नियम, विफलता अर्थविज्ञान): [सीक्रेट लागूकरण योजना अनुबंध](/hi/gateway/secrets-plan-contract)।

### रोलबैक बैकअप क्यों नहीं हैं

`secrets apply` जानबूझकर पुराने प्लेनटेक्स्ट मानों वाले रोलबैक बैकअप नहीं लिखता। सुरक्षा सख्त प्रीफ़्लाइट और लगभग-परमाण्विक लागूकरण से आती है, जिसमें विफलता पर सर्वोत्तम-प्रयास इन-मेमोरी पुनर्स्थापन होता है।

## उदाहरण

```bash
openclaw secrets audit --check
openclaw secrets configure
openclaw secrets audit --check
```

यदि `audit --check` अभी भी प्लेनटेक्स्ट निष्कर्षों की रिपोर्ट करता है, तो शेष रिपोर्ट किए गए लक्ष्य पथ अपडेट करें और ऑडिट फिर से चलाएँ।

## संबंधित

- [CLI संदर्भ](/hi/cli)
- [सीक्रेट प्रबंधन](/hi/gateway/secrets)
- [Vault SecretRefs](/hi/plugins/vault)
