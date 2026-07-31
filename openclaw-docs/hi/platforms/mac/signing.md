---
read_when:
    - Mac के डीबग बिल्ड बनाना या साइन करना
summary: पैकेजिंग स्क्रिप्ट द्वारा जनरेट किए गए macOS डीबग बिल्ड के लिए साइनिंग चरण
title: macOS साइनिंग
x-i18n:
    generated_at: "2026-07-27T18:10:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 406211dadc9293cf7983e75ae7dd98234f9088351234cf06c33df2f63d1b9b97
    source_path: platforms/mac/signing.md
    workflow: 16
---

# mac हस्ताक्षरण (डीबग बिल्ड)

[`scripts/package-mac-app.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-app.sh) ऐप को एक निश्चित पथ (`dist/OpenClaw.app`) पर बिल्ड और पैकेज करता है, फिर उस पर हस्ताक्षर करने के लिए [`scripts/codesign-mac-app.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/codesign-mac-app.sh) को कॉल करता है। TCC अनुमतियाँ बंडल ID और कोड हस्ताक्षर से जुड़ी होती हैं; दोबारा बिल्ड करने पर इन दोनों को स्थिर रखने (और ऐप को एक निश्चित पथ पर रखने) से macOS, TCC अनुदानों (सूचनाएँ, सुलभता, स्क्रीन रिकॉर्डिंग, माइक्रोफ़ोन, वाक्) को भूलता नहीं है।

- डीबग बंडल आइडेंटिफ़ायर का डिफ़ॉल्ट `ai.openclaw.mac.debug` है (`BUNDLE_ID=...` से ओवरराइड करें)।
- Node: `>=22.22.3 <23`, `>=24.15.0 <25`, या `>=25.9.0` (रेपो `package.json` `engines`)। पैकेजर Control UI भी बिल्ड करता है (`pnpm ui:build`)।
- डिफ़ॉल्ट रूप से वास्तविक हस्ताक्षरण पहचान आवश्यक है; यदि कोई पहचान नहीं मिलती और `ALLOW_ADHOC_SIGNING` सेट नहीं है, तो codesign स्क्रिप्ट त्रुटि के साथ बंद हो जाती है। तदर्थ हस्ताक्षरण (`SIGN_IDENTITY="-"`) के लिए स्पष्ट रूप से ऑप्ट-इन करना पड़ता है और यह दोबारा बिल्ड करने पर TCC अनुमतियों को बनाए नहीं रखता। [macOS अनुमतियाँ](/hi/platforms/mac/permissions) देखें।
- परिवेश से `SIGN_IDENTITY` पढ़ता है (जैसे `export SIGN_IDENTITY="Apple Development: Your Name (TEAMID)"`, या Developer ID Application प्रमाणपत्र)। इसके बिना, `codesign-mac-app.sh` इस क्रम में अपने-आप पहचान चुनता है: Developer ID Application, Apple Distribution, Apple Development, फिर मिलने वाली पहली मान्य कोड-हस्ताक्षरण पहचान।
- `CODESIGN_TIMESTAMP=auto` (डिफ़ॉल्ट) केवल Developer ID Application हस्ताक्षरों के लिए विश्वसनीय टाइमस्टैम्प सक्षम करता है। किसी भी विकल्प को बाध्य करने के लिए `on`/`off` सेट करें।
- Info.plist पर `OpenClawBuildTimestamp` (ISO8601 UTC) और `OpenClawGitCommit` (छोटा हैश, उपलब्ध न होने पर `unknown`) अंकित करता है, ताकि परिचय टैब बिल्ड, git और डीबग/रिलीज़ चैनल दिखा सके।
- हस्ताक्षरण के बाद Team ID ऑडिट चलाता है और यदि बंडल के भीतर किसी Mach-O की Team ID अलग हो, तो विफल हो जाता है। इससे बचने के लिए `SKIP_TEAM_ID_CHECK=1` सेट करें।

## उपयोग

```bash
# रेपो रूट से
scripts/package-mac-app.sh                                                      # पहचान अपने-आप चुनता है; कोई पहचान न मिलने पर त्रुटि देता है
SIGN_IDENTITY="Developer ID Application: Your Name" scripts/package-mac-app.sh   # वास्तविक प्रमाणपत्र
ALLOW_ADHOC_SIGNING=1 scripts/package-mac-app.sh                                 # तदर्थ (अनुमतियाँ बनी नहीं रहेंगी)
SIGN_IDENTITY="-" scripts/package-mac-app.sh                                     # स्पष्ट तदर्थ (वही चेतावनी)
DISABLE_LIBRARY_VALIDATION=1 scripts/package-mac-app.sh                          # केवल डेवलपमेंट के लिए Sparkle Team ID बेमेल का समाधान
```

### तदर्थ हस्ताक्षरण संबंधी टिप्पणी

`SIGN_IDENTITY="-"` Hardened Runtime (`--options runtime`) को अक्षम करता है, ताकि ऐप द्वारा समान Team ID साझा न करने वाले एम्बेडेड फ़्रेमवर्क (जैसे Sparkle) लोड करने पर क्रैश न हो। तदर्थ हस्ताक्षर TCC अनुमति स्थायित्व को भी बाधित करते हैं; पुनर्प्राप्ति के चरणों के लिए [macOS अनुमतियाँ](/hi/platforms/mac/permissions) देखें।

## परिचय के लिए बिल्ड मेटाडेटा

परिचय टैब संस्करण, बिल्ड की तारीख, git कमिट और बिल्ड के DEBUG होने की स्थिति (`#if DEBUG` के माध्यम से) दिखाने के लिए Info.plist से `OpenClawBuildTimestamp` और `OpenClawGitCommit` पढ़ता है। इन मानों को रीफ़्रेश करने के लिए कोड में बदलावों के बाद पैकेजर को दोबारा चलाएँ।

## संबंधित

- [macOS ऐप](/hi/platforms/macos)
- [macOS अनुमतियाँ](/hi/platforms/mac/permissions)
