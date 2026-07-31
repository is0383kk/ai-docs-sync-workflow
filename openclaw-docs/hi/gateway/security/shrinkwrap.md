---
read_when:
    - आप जानना चाहते हैं कि OpenClaw रिलीज़ में npm shrinkwrap का क्या अर्थ है
    - आप पैकेज लॉकफ़ाइलों, डिपेंडेंसी परिवर्तनों या सप्लाई-चेन जोखिम की समीक्षा कर रहे हैं
    - आप प्रकाशित करने से पहले रूट या Plugin npm पैकेजों का सत्यापन कर रहे हैं
summary: OpenClaw रिलीज़ में npm shrinkwrap का सरल अंग्रेज़ी और तकनीकी स्पष्टीकरण
title: npm shrinkwrap
x-i18n:
    generated_at: "2026-07-27T19:47:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d1e6c0d4541da9220d50cde0b9db064e5a91b81d6562cb16ac697de7d4017098
    source_path: gateway/security/shrinkwrap.md
    workflow: 16
---

OpenClaw स्रोत चेकआउट `pnpm-lock.yaml` का उपयोग करते हैं। प्रकाशित OpenClaw npm पैकेज `npm-shrinkwrap.json`, यानी npm की प्रकाशित की जा सकने वाली निर्भरता लॉकफ़ाइल, का उपयोग करते हैं, ताकि पैकेज इंस्टॉलेशन में रिलीज़ के दौरान समीक्षित निर्भरता ग्राफ़ का उपयोग हो।

## यह क्यों महत्वपूर्ण है

Shrinkwrap किसी npm पैकेज के साथ भेजे जाने वाले निर्भरता ट्री की रसीद है: यह npm को बताता है कि कौन-से सटीक ट्रांज़िटिव संस्करण इंस्टॉल करने हैं।

| फ़ाइल                  | जहाँ यह महत्वपूर्ण है         | इसका अर्थ                     |
| --------------------- | ------------------------ | --------------------------------- |
| `pnpm-lock.yaml`      | OpenClaw स्रोत चेकआउट | मेंटेनर निर्भरता ग्राफ़       |
| `npm-shrinkwrap.json` | प्रकाशित npm पैकेज    | उपयोगकर्ताओं के लिए npm इंस्टॉल ग्राफ़       |
| `package-lock.json`   | स्थानीय npm ऐप्स           | OpenClaw प्रकाशन अनुबंध नहीं है |

OpenClaw रिलीज़ के लिए इसका अर्थ है:

- प्रकाशित पैकेज npm से इंस्टॉल के समय नया निर्भरता ग्राफ़ गढ़ने को नहीं कहता;
- निर्भरता परिवर्तनों की समीक्षा की जा सकती है, क्योंकि वे लॉकफ़ाइल के डिफ़ में आते हैं;
- रिलीज़ सत्यापन उसी ग्राफ़ का परीक्षण करता है जिसे उपयोगकर्ता इंस्टॉल करेंगे;
- पैकेज आकार या नेटिव निर्भरता से जुड़े अप्रत्याशित परिणाम प्रकाशन से पहले सामने आ जाते हैं।

Shrinkwrap कोई सैंडबॉक्स नहीं है। यह अपने आप किसी निर्भरता को सुरक्षित नहीं बनाता, और यह होस्ट आइसोलेशन, `openclaw security audit`, पैकेज उद्गम, या इंस्टॉल स्मोक परीक्षणों का स्थान नहीं लेता।

OpenClaw एक Gateway, Plugin होस्ट, मॉडल राउटर और एजेंट रनटाइम है, इसलिए डिफ़ॉल्ट इंस्टॉलेशन स्टार्टअप समय, डिस्क उपयोग, नेटिव पैकेज डाउनलोड और सप्लाई-चेन जोखिम को प्रभावित करता है। Shrinkwrap रिलीज़ समीक्षा को एक स्थिर सीमा देता है: समीक्षक ट्रांज़िटिव निर्भरताओं में बदलाव देखते हैं, सत्यापनकर्ता लॉकफ़ाइल में अप्रत्याशित विचलन अस्वीकार करते हैं, और Plugin पैकेज रूट पैकेज पर निर्भर रहने के बजाय अपना लॉक किया हुआ निर्भरता ग्राफ़ रखते हैं।

## जनरेट करना और जाँचना

रूट `openclaw` npm पैकेज, OpenClaw के स्वामित्व वाले npm Plugin पैकेज (उदाहरण के लिए `@openclaw/discord`), और प्रकाशित किए जा सकने वाले वर्कस्पेस पैकेज, जैसे [`@openclaw/ai`](/hi/reference/openclaw-ai), प्रकाशन के समय `npm-shrinkwrap.json` शामिल करते हैं। वर्कस्पेस निर्भरताएँ रूट shrinkwrap से हटा दी जाती हैं, क्योंकि वे रूट पैकेज के साथ प्रकाशित होती हैं; इसके बजाय, प्रत्येक प्रकाशित किया जा सकने वाला वर्कस्पेस पैकेज अपने ट्रांज़िटिव ट्री को पिन करता है। उपयुक्त Plugin पैकेज स्पष्ट `bundledDependencies` के साथ भी प्रकाशित हो सकते हैं, जिससे वे केवल इंस्टॉल-समय रिज़ॉल्यूशन पर निर्भर रहने के बजाय अपनी रनटाइम निर्भरता फ़ाइलें Plugin टारबॉल में रखते हैं।

```bash
# shrinkwrap द्वारा प्रबंधित सभी पैकेज (रूट + प्रकाशित किए जा सकने वाले Plugin)
pnpm deps:shrinkwrap:generate
pnpm deps:shrinkwrap:check

# केवल रूट पैकेज
pnpm deps:shrinkwrap:root:generate
pnpm deps:shrinkwrap:root:check

# केवल वर्तमान परिवर्तन-समूह से प्रभावित पैकेज
pnpm deps:shrinkwrap:changed:generate
pnpm deps:shrinkwrap:changed:check
```

जनरेटर npm के प्रकाशित किए जा सकने वाले लॉक प्रारूप को रिज़ॉल्व करता है, लेकिन ऐसे जनरेट किए गए पैकेज संस्करणों को अस्वीकार करता है जो पहले से `pnpm-lock.yaml` में उपस्थित नहीं हैं। इससे pnpm निर्भरता की आयु, ओवरराइड और पैच-समीक्षा सीमा अक्षुण्ण रहती है।

इनकी सुरक्षा-संवेदनशील रूप में समीक्षा करें:

- `pnpm-lock.yaml`
- `npm-shrinkwrap.json`
- बंडल किए गए Plugin निर्भरता पेलोड
- कोई भी `package-lock.json` डिफ़

OpenClaw पैकेज सत्यापनकर्ता नए रूट पैकेज टारबॉल में shrinkwrap आवश्यक बनाते हैं और प्रकाशित पैकेजों के लिए `package-lock.json` अस्वीकार करते हैं। Plugin npm प्रकाशन पथ Plugin-स्थानीय shrinkwrap की जाँच करता है, पैकेज-स्थानीय बंडल की गई निर्भरताएँ इंस्टॉल करता है, और फिर पैक या प्रकाशित करता है।

## प्रकाशित पैकेज का निरीक्षण करना

रूट पैकेज:

```bash
npm pack openclaw@<version> --json --pack-destination /tmp/openclaw-pack
tar -tf /tmp/openclaw-pack/openclaw-<version>.tgz | grep '^package/npm-shrinkwrap.json$'
```

Plugin पैकेज:

```bash
npm pack @openclaw/discord@<version> --json --pack-destination /tmp/openclaw-plugin-pack
tar -tf /tmp/openclaw-plugin-pack/openclaw-discord-<version>.tgz | grep '^package/npm-shrinkwrap.json$'
tar -tf /tmp/openclaw-plugin-pack/openclaw-discord-<version>.tgz | grep '^package/node_modules/'
```

पृष्ठभूमि: [npm-shrinkwrap.json](https://docs.npmjs.com/cli/v11/configuring-npm/npm-shrinkwrap-json)।
