---
read_when:
    - OpenClaw.app में PeekabooBridge की होस्टिंग
    - Swift Package Manager के माध्यम से Peekaboo का एकीकरण
    - PeekabooBridge प्रोटोकॉल/पाथ बदलना
    - PeekabooBridge, Codex Computer Use और cua-driver MCP के बीच चयन करना
summary: macOS UI स्वचालन के लिए PeekabooBridge एकीकरण
title: पीकाबू ब्रिज
x-i18n:
    generated_at: "2026-07-27T20:05:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 24d4187b2f5c5f11f44a24e25b350adaa3b068f24dce640ec695d52eb61f8e9a
    source_path: platforms/mac/peekaboo.md
    workflow: 16
---

OpenClaw **PeekabooBridge** को स्थानीय, अनुमति-जागरूक UI स्वचालन ब्रोकर के रूप में होस्ट कर सकता है (`PeekabooBridgeHostCoordinator`, जो `steipete/Peekaboo` Swift पैकेज द्वारा समर्थित है)। इससे `peekaboo` CLI, macOS ऐप की TCC अनुमतियों का पुनः उपयोग करते हुए UI स्वचालन चला सकता है।

## यह क्या है (और क्या नहीं है)

- **होस्ट**: OpenClaw.app, PeekabooBridge होस्ट के रूप में कार्य कर सकता है।
- **क्लाइंट**: `peekaboo` CLI (कोई अलग `openclaw ui ...` सतह नहीं है)।
- **UI**: दृश्य ओवरले Peekaboo.app में ही रहते हैं; OpenClaw एक हल्का ब्रोकर होस्ट है।

## अन्य डेस्कटॉप-नियंत्रण मार्गों से संबंध

OpenClaw में डेस्कटॉप नियंत्रण के चार मार्ग हैं, जिन्हें जानबूझकर अलग रखा गया है:

- **PeekabooBridge होस्ट**: OpenClaw.app स्थानीय PeekabooBridge सॉकेट को होस्ट करता है। `peekaboo` CLI क्लाइंट है और स्क्रीनशॉट, क्लिक, मेनू, संवाद, Dock कार्रवाइयों तथा विंडो प्रबंधन के लिए OpenClaw.app की macOS अनुमतियों का उपयोग करता है।
- **एजेंट-संचालित कंप्यूटर उपयोग (`computer.act`)**: Gateway एजेंट का अंतर्निहित `computer` टूल, `screen.snapshot` के माध्यम से स्क्रीनशॉट कैप्चर करता है और खतरनाक `computer.act` Node कमांड के जरिए पॉइंटर तथा कीबोर्ड चलाता है। macOS Node, PeekabooBridge सॉकेट या `peekaboo` CLI से गुजरे बिना, इस ब्रिज द्वारा उपलब्ध कराई गई एम्बेडेड Peekaboo स्वचालन सेवाओं और सीमित CoreGraphics प्रिमिटिव का उपयोग करके `computer.act` को प्रक्रिया के भीतर पूरा करता है। [कंप्यूटर उपयोग](/hi/nodes/computer-use) देखें।
- **Codex कंप्यूटर उपयोग**: बंडल किया गया `codex` Plugin, Codex के `computer-use` MCP Plugin (`extensions/codex/src/app-server/computer-use.ts`) की जाँच करता है और उसे इंस्टॉल कर सकता है, फिर Codex-मोड टर्न के दौरान Codex को मूल डेस्कटॉप-नियंत्रण टूल कॉल का स्वामित्व देता है। OpenClaw उन कार्रवाइयों को PeekabooBridge के माध्यम से प्रॉक्सी नहीं करता।
- **प्रत्यक्ष `cua-driver` MCP**: OpenClaw, TryCua के अपस्ट्रीम `cua-driver mcp` सर्वर को सामान्य MCP सर्वर के रूप में पंजीकृत कर सकता है, जिससे एजेंटों को Codex मार्केटप्लेस या PeekabooBridge सॉकेट के माध्यम से रूट किए बिना CUA ड्राइवर की अपनी स्कीमा और pid/window/element-index कार्यप्रवाह मिलता है।

OpenClaw.app के अनुमति-जागरूक ब्रिज होस्ट के माध्यम से व्यापक macOS स्वचालन सतह के लिए Peekaboo का उपयोग करें। जब Gateway एजेंट को एक समान `computer.act` Node कमांड के माध्यम से डेस्कटॉप देखना और नियंत्रित करना हो, जिसे कोई भी विज़न मॉडल चला सके, तब एजेंट-संचालित कंप्यूटर उपयोग का इस्तेमाल करें। जब Codex-मोड एजेंट को Codex के मूल Plugin पर निर्भर रहना हो, तब Codex कंप्यूटर उपयोग का इस्तेमाल करें। किसी भी OpenClaw-प्रबंधित रनटाइम के लिए CUA ड्राइवर को सामान्य MCP सर्वर के रूप में उपलब्ध कराने हेतु प्रत्यक्ष `cua-driver mcp` का उपयोग करें।

## ब्रिज सक्षम करें

macOS ऐप में: **Settings -> Enable Peekaboo Bridge**। इस टॉगल के लिए **Allow Computer Control** चालू होना आवश्यक है, क्योंकि दोनों स्थानीय UI स्वचालन की अनुमति देते हैं; Computer Control बंद होने पर टॉगल अक्षम रहता है और होस्ट नहीं चलता। Computer Control के बिना Peekaboo चलाने के लिए, इसके बजाय Peekaboo के अपने Mac ऐप को होस्ट के रूप में चलाएँ।

सक्षम होने पर (और Computer Control चालू होने पर), OpenClaw `~/Library/Application Support/OpenClaw/<socket-name>` पर एक स्थानीय UNIX सॉकेट सर्वर शुरू करता है। अक्षम होने पर, होस्ट रुक जाता है और `peekaboo` अन्य उपलब्ध होस्ट पर फ़ॉलबैक करता है। समन्वयक पुराने `peekaboo` इंस्टॉलेशन के लिए वर्तमान सॉकेट की ओर संकेत करने वाले विरासत सॉकेट सिमलिंक (`clawdbot`, `clawdis`, Application Support के अंतर्गत `moltbot`) भी बनाए रखता है।

## क्लाइंट खोज क्रम

Peekaboo क्लाइंट सामान्यतः इस क्रम में होस्ट आज़माते हैं:

1. Peekaboo.app (पूर्ण UX)
2. Claude.app (यदि इंस्टॉल है)
3. OpenClaw.app (हल्का ब्रोकर)

कौन-सा होस्ट सक्रिय है और कौन-सा सॉकेट पथ उपयोग में है, यह देखने के लिए `peekaboo bridge status --verbose` का उपयोग करें। इसे इससे ओवरराइड करें:

```bash
export PEEKABOO_BRIDGE_SOCKET=/path/to/bridge.sock
```

## सुरक्षा और अनुमतियाँ

- ब्रिज **कॉलर कोड हस्ताक्षरों** को सत्यापित करता है; TeamIDs की अनुमत सूची लागू की जाती है (Peekaboo होस्ट TeamID और चल रहे ऐप का अपना TeamID)।
- Accessibility के लिए सामान्य `node` रनटाइम के बजाय हस्ताक्षरित ब्रिज/ऐप पहचान को प्राथमिकता दें। `node` को Accessibility देने से उस Node एक्ज़ीक्यूटेबल द्वारा शुरू किया गया कोई भी पैकेज GUI स्वचालन पहुँच प्राप्त कर सकता है; [macOS अनुमतियाँ](/hi/platforms/mac/permissions#accessibility-grants-for-node-and-cli-runtimes) देखें।
- अनुरोध 10 सेकंड (`requestTimeoutSec: 10`) के बाद टाइम आउट हो जाते हैं।
- यदि आवश्यक अनुमतियाँ मौजूद नहीं हैं, तो ब्रिज System Settings खोलने के बजाय एक स्पष्ट त्रुटि संदेश लौटाता है।

## स्नैपशॉट व्यवहार (स्वचालन)

स्नैपशॉट 10-मिनट की वैधता अवधि और 50 स्नैपशॉट की सीमा (`InMemorySnapshotManager`) के साथ मेमोरी में संग्रहीत किए जाते हैं; क्लीनअप के दौरान आर्टिफ़ैक्ट नहीं हटाए जाते। यदि आपको अधिक समय तक प्रतिधारण चाहिए, तो क्लाइंट से दोबारा कैप्चर करें।

## समस्या निवारण

- यदि `peekaboo` "bridge client is not authorized" रिपोर्ट करता है, तो सुनिश्चित करें कि क्लाइंट उचित रूप से हस्ताक्षरित है या केवल **debug** मोड में `PEEKABOO_ALLOW_UNSIGNED_SOCKET_CLIENTS=1` के साथ होस्ट चलाएँ।
- यदि कोई होस्ट नहीं मिलता, तो किसी होस्ट ऐप (Peekaboo.app या OpenClaw.app) को खोलें और पुष्टि करें कि अनुमतियाँ दी गई हैं।

## संबंधित

- [macOS ऐप](/hi/platforms/macos)
- [macOS अनुमतियाँ](/hi/platforms/mac/permissions)
