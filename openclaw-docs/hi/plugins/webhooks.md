---
read_when:
    - आप किसी बाहरी सिस्टम से TaskFlows को ट्रिगर या संचालित करना चाहते हैं
    - आप बंडल किए गए वेबहुक Plugin को कॉन्फ़िगर कर रहे हैं
summary: 'Webhooks Plugin: विश्वसनीय बाहरी स्वचालन के लिए प्रमाणीकृत TaskFlow इनग्रेस'
title: Webhooks Plugin
x-i18n:
    generated_at: "2026-07-27T20:17:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 77e455450d6183635c76a1e8002feeb287deb4ff242dbd555ef9d0f2b21ce5f6
    source_path: plugins/webhooks.md
    workflow: 16
---

Webhooks Plugin प्रमाणित HTTP रूट जोड़ता है, ताकि कोई विश्वसनीय बाहरी
सिस्टम (Zapier, n8n, कोई CI जॉब, कोई आंतरिक सेवा) कस्टम Plugin लिखे बिना
HTTP के माध्यम से प्रबंधित OpenClaw TaskFlows बना और संचालित कर सके।

Plugin Gateway प्रक्रिया के भीतर चलता है। किसी रिमोट Gateway के लिए, इसे
उस होस्ट पर इंस्टॉल और कॉन्फ़िगर करें, फिर Gateway को पुनः आरंभ करें। यह बिना
किसी कॉन्फ़िगर किए गए रूट के उपलब्ध होता है, इसलिए कम-से-कम एक रूट जोड़ने तक यह कोई कार्रवाई नहीं करता।

## रूट कॉन्फ़िगर करें

कॉन्फ़िगरेशन को `plugins.entries.webhooks.config` के अंतर्गत सेट करें:

```json5
{
  plugins: {
    entries: {
      webhooks: {
        enabled: true,
        config: {
          routes: {
            zapier: {
              path: "/plugins/webhooks/zapier",
              sessionKey: "agent:main:main",
              secret: {
                source: "env",
                provider: "default",
                id: "OPENCLAW_WEBHOOK_SECRET",
              },
              controllerId: "webhooks/zapier",
              description: "Zapier TaskFlow bridge",
            },
          },
        },
      },
    },
  },
}
```

रूट फ़ील्ड:

| फ़ील्ड          | आवश्यक | डिफ़ॉल्ट                       | टिप्पणियाँ                                         |
| -------------- | -------- | ----------------------------- | --------------------------------------------- |
| `enabled`      | नहीं       | `true`                        |                                               |
| `path`         | नहीं       | `/plugins/webhooks/<routeId>` | सभी रूट में अद्वितीय होना चाहिए।                 |
| `sessionKey`   | हाँ      | -                             | वह सेशन जिसके स्वामित्व में संबद्ध TaskFlows हैं।        |
| `secret`       | हाँ      | -                             | सादा स्ट्रिंग या SecretRef (नीचे)।          |
| `controllerId` | नहीं       | `webhooks/<routeId>`          | डिफ़ॉल्ट `create_flow` नियंत्रक के रूप में उपयोग किया जाता है। |
| `description`  | नहीं       | -                             | केवल ऑपरेटर के लिए टिप्पणी।                           |

`secret` सादा स्ट्रिंग या SecretRef स्वीकार करता है: `{ source: "env" | "file" | "exec", provider: "default", id: "..." }`।

SecretRefs का समाधान Gateway के स्टार्टअप कॉन्फ़िगरेशन स्नैपशॉट में किया जाता है। जब किसी रूट के
सीक्रेट का समाधान नहीं हो पाता, तो Gateway चलता रहता है और वही रूट
पंजीकृत लेकिन निष्क्रिय रहता है: अनुरोधों को सामान्य प्रमाणीकरण विफलता मिलती है (`401`)।
अन्य रूट उपलब्ध रहते हैं। SecretRef स्रोत ठीक करें, फिर नया स्नैपशॉट सक्रिय
करने के लिए Gateway को रीलोड या पुनः आरंभ करें। सार्वजनिक अनुरोध पथ पर
SecretRef मानों का समाधान कभी नहीं किया जाता।

## सुरक्षा मॉडल

प्रत्येक रूट अपने कॉन्फ़िगर किए गए `sessionKey` के TaskFlow अधिकार के साथ कार्य करता है: यह
उस सेशन के स्वामित्व वाले किसी भी TaskFlow का निरीक्षण और परिवर्तन कर सकता है। TaskFlow एक्सेस
हमेशा `api.runtime.tasks.managedFlows.bindSession(...)` के माध्यम से होता है, इसलिए कोई
रूट कभी भी अपने संबद्ध सेशन के बाहर कार्य नहीं कर सकता। प्रभाव क्षेत्र सीमित करने के लिए:

- प्रत्येक रूट के लिए एक मजबूत, अद्वितीय सीक्रेट उपयोग करें।
- इनलाइन सादे टेक्स्ट वाले सीक्रेट के बजाय SecretRef को प्राथमिकता दें।
- रूट को वर्कफ़्लो के अनुकूल सबसे सीमित सेशन से संबद्ध करें।
- केवल वही विशिष्ट Webhook पथ उपलब्ध कराएँ जिसकी आपको आवश्यकता है।

प्रत्येक पथ के लिए अनुरोध संसाधन क्रम: HTTP विधि (केवल `POST`) और
`Content-Type: application/json` जाँच, फिर निश्चित-विंडो दर सीमांकन (प्रत्येक path+client-IP कुंजी पर प्रति 60-सेकंड विंडो में 120
अनुरोध, अधिकतम 4,096 ट्रैक की गई
कुंजियाँ), फिर प्रगति पर मौजूद अनुरोधों की सीमा (प्रति कुंजी 8 समवर्ती अनुरोध, अधिकतम
4,096 ट्रैक की गई कुंजियाँ), फिर साझा-सीक्रेट प्रमाणीकरण, फिर 256 KB /
15-सेकंड की JSON बॉडी रीड। किसी शुरुआती जाँच में विफल होने वाले अनुरोध
बाद की जाँचों तक कभी नहीं पहुँचते।

## अनुरोध प्रारूप

`Content-Type: application/json` और `Authorization: Bearer <secret>` या `x-openclaw-webhook-secret: <secret>` में से किसी एक के साथ
`POST` अनुरोध भेजें:

```bash
curl -X POST https://gateway.example.com/plugins/webhooks/zapier \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer YOUR_SHARED_SECRET' \
  -d '{"action":"create_flow","goal":"Review inbound queue"}'
```

## समर्थित क्रियाएँ

| क्रिया             | उद्देश्य                                                            |
| ------------------ | ------------------------------------------------------------------ |
| `create_flow`      | रूट के सेशन के लिए एक प्रबंधित TaskFlow बनाएँ।                 |
| `get_flow`         | आईडी द्वारा एक TaskFlow प्राप्त करें।                                          |
| `list_flows`       | रूट के सेशन के TaskFlows सूचीबद्ध करें।                            |
| `find_latest_flow` | सबसे हाल में अपडेट किया गया TaskFlow प्राप्त करें।                          |
| `resolve_flow`     | अपारदर्शी टोकन द्वारा TaskFlow का समाधान करें।                                |
| `get_task_summary` | किसी TaskFlow का कार्य सारांश प्राप्त करें।                             |
| `set_waiting`      | वैकल्पिक स्थिति/प्रतीक्षा डेटा के साथ TaskFlow को प्रतीक्षारत चिह्नित करें।            |
| `resume_flow`      | प्रतीक्षारत/अवरुद्ध TaskFlow को फिर से शुरू करें।                                 |
| `finish_flow`      | TaskFlow को पूर्ण चिह्नित करें।                                          |
| `fail_flow`        | TaskFlow को विफल चिह्नित करें।                                            |
| `request_cancel`   | सहकारी रद्दीकरण का अनुरोध करें।                                  |
| `cancel_flow`      | TaskFlow रद्द करें (यदि चाइल्ड अभी भी सक्रिय हैं, तो `202` लौटाया जा सकता है)। |
| `run_task`         | किसी मौजूदा TaskFlow के भीतर एक प्रबंधित चाइल्ड कार्य बनाएँ।           |

परिवर्तनकारी क्रियाओं (`set_waiting`, `resume_flow`, `finish_flow`, `fail_flow`,
`request_cancel`) को आशावादी समवर्ती नियंत्रण के लिए `flowId` और `expectedRevision`
की आवश्यकता होती है; कोई पुराना संशोधन `409 revision_conflict` लौटाता है।

### `create_flow`

```json
{
  "action": "create_flow",
  "goal": "Review inbound queue",
  "status": "queued",
  "notifyPolicy": "done_only"
}
```

### `run_task`

अनुमत `runtime` मान: `subagent`, `acp`। `startedAt`, `lastEventAt`, और
`progressSummary` केवल तभी मान्य हैं जब `status`, `"running"` हो; उन्हें
किसी अन्य स्थिति के साथ भेजने पर `400 invalid_request` लौटता है।

```json
{
  "action": "run_task",
  "flowId": "flow_123",
  "runtime": "acp",
  "childSessionKey": "agent:main:acp:worker",
  "task": "Inspect the next message batch"
}
```

## प्रतिक्रिया संरचना

```json
{
  "ok": true,
  "routeId": "zapier",
  "result": {}
}
```

```json
{
  "ok": false,
  "routeId": "zapier",
  "code": "not_found",
  "error": "TaskFlow not found.",
  "result": {}
}
```

प्रवाह और कार्य दृश्य कभी भी स्वामी/सेशन मेटाडेटा शामिल नहीं करते, इसलिए प्रतिक्रियाएँ
रूट के संबद्ध `sessionKey` को लीक नहीं कर सकतीं। `code` मानों में `not_found`,
`not_managed`, `revision_conflict`, `persist_failed`, `cancel_requested`,
`cancel_pending`, `terminal`, `invalid_request`, `request_rejected`, और
क्रिया-विशिष्ट फ़ॉलबैक कोड (`mutation_rejected`, `create_rejected`,
`task_not_created`, `cancel_rejected`) शामिल हैं, जब कोई परिवर्तन ऐसे
कारण से अस्वीकार किया जाता है जिसे ऊपर दिए गए नामित कोड में शामिल नहीं किया गया है।

## संबंधित

- [हुक्स](/hi/automation/hooks) - आंतरिक घटना-संचालित हुक्स बनाम यह HTTP-आधारित TaskFlow ब्रिज
- [Gateway Webhooks (`hooks.*` कॉन्फ़िगरेशन)](/hi/automation/cron-jobs#webhooks) - अलग सामान्य Gateway HTTP एंडपॉइंट सुविधा; इस Plugin के रूट के समान नहीं
- [Plugin रनटाइम SDK](/hi/plugins/sdk-runtime)
- [CLI Webhooks](/hi/cli/webhooks)
