---
read_when:
    - ऐसी होस्ट टूलिंग बनाना जो Gateway WebSocket RPC क्लाइंट का उपयोग नहीं कर सकती
    - निजी विश्वसनीय इनग्रेस के पीछे Gateway व्यवस्थापन स्वचालन को उपलब्ध कराना
    - Gateway विधियों तक HTTP पहुँच के सुरक्षा मॉडल का ऑडिट करना
summary: बंडल किए गए, ऑप्ट-इन admin-http-rpc Plugin के माध्यम से चुनी गई Gateway कंट्रोल-प्लेन विधियों को उपलब्ध कराएँ
title: एडमिन HTTP RPC Plugin
x-i18n:
    generated_at: "2026-07-27T20:07:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0709081efd0ce65cef7edac54df9a71978cbad17e2b25df83ac9075de938376c
    source_path: plugins/admin-http-rpc.md
    workflow: 16
---

बंडल किया गया `admin-http-rpc` Plugin HTTP के माध्यम से Gateway कंट्रोल-प्लेन विधियों का एक अनुमत-सूचीबद्ध सेट उपलब्ध कराता है, ऐसे विश्वसनीय होस्ट ऑटोमेशन के लिए जो Gateway WebSocket कनेक्शन खुला नहीं रख सकता।

यह OpenClaw के साथ आता है, लेकिन डिफ़ॉल्ट रूप से अक्षम रहता है; अक्षम होने पर रूट पंजीकृत नहीं होता। सक्षम होने पर, यह Gateway (`http://<gateway-host>:<port>/api/v1/admin/rpc`) वाले ही लिसनर पर `POST /api/v1/admin/rpc` जोड़ता है।

इसे केवल निजी होस्ट टूलिंग, टेलनेट ऑटोमेशन या किसी विश्वसनीय आंतरिक इनग्रेस के लिए सक्षम करें। इस रूट को कभी भी सीधे सार्वजनिक इंटरनेट पर उजागर न करें।

## इसे सक्षम करने से पहले

एडमिन HTTP RPC पूर्ण ऑपरेटर कंट्रोल-प्लेन सतह है: Gateway HTTP प्रमाणीकरण पास करने वाला कोई भी कॉलर नीचे दी गई अनुमत-सूचीबद्ध विधियों को इनवोक कर सकता है। इसे केवल तभी सक्षम करें जब ये सभी शर्तें पूरी हों:

- कॉलर Gateway को संचालित करने के लिए विश्वसनीय है।
- कॉलर WebSocket RPC क्लाइंट का उपयोग नहीं कर सकता।
- रूट केवल लूपबैक, टेलनेट या निजी प्रमाणीकृत इनग्रेस पर पहुँच योग्य है।
- आपने अनुमत विधियों की समीक्षा की है और वे उस ऑटोमेशन से मेल खाती हैं जिसे आप चलाने वाले हैं।

ऐसे OpenClaw क्लाइंट और इंटरैक्टिव टूल जो Gateway WebSocket कनेक्शन खुला रख सकते हैं, उनके लिए इसके बजाय WebSocket RPC का उपयोग करें।

## सक्षम करें

बंडल किया गया Plugin सक्षम करें:

<Tabs>
  <Tab title="CLI">
    ```bash
    openclaw plugins enable admin-http-rpc
    openclaw gateway restart
    ```
  </Tab>
  <Tab title="कॉन्फ़िगरेशन">
    ```json5
    {
      plugins: {
        entries: {
          "admin-http-rpc": { enabled: true },
        },
      },
    }
    ```
  </Tab>
</Tabs>

Plugin स्टार्टअप के दौरान रूट पंजीकृत होता है, इसलिए Plugin कॉन्फ़िगरेशन बदलने के बाद Gateway को पुनः आरंभ करें।

जब आपको HTTP सतह की आवश्यकता न रहे, तो इसे अक्षम कर दें:

```bash
openclaw plugins disable admin-http-rpc
openclaw gateway restart
```

## रूट सत्यापित करें

सबसे छोटे सुरक्षित अनुरोध के रूप में `health` का उपयोग करें:

```bash
curl -sS http://<gateway-host>:<port>/api/v1/admin/rpc \
  -H 'Authorization: Bearer <gateway-token>' \
  -H 'Content-Type: application/json' \
  -d '{"method":"health","params":{}}'
```

सफल प्रतिक्रिया में `ok: true` होता है:

```json
{
  "id": "generated-request-id",
  "ok": true,
  "payload": {
    "status": "ok"
  }
}
```

Plugin अक्षम होने पर, रूट `404` लौटाता है क्योंकि वह पंजीकृत नहीं होता।

## प्रमाणीकरण

Plugin रूट Gateway HTTP प्रमाणीकरण का उपयोग करता है।

सामान्य प्रमाणीकरण पथ:

- साझा-गुप्त प्रमाणीकरण (`gateway.auth.mode="token"` या `"password"`): `Authorization: Bearer <token-or-password>`
- विश्वसनीय पहचान-युक्त HTTP प्रमाणीकरण (`gateway.auth.mode="trusted-proxy"`): कॉन्फ़िगर किए गए पहचान-जागरूक प्रॉक्सी के माध्यम से रूट करें और उसे आवश्यक पहचान हेडर इंजेक्ट करने दें
- निजी-इनग्रेस खुला प्रमाणीकरण (`gateway.auth.mode="none"`): किसी प्रमाणीकरण हेडर की आवश्यकता नहीं

## सुरक्षा मॉडल

इस Plugin को पूर्ण Gateway ऑपरेटर सतह मानें।

- Plugin को सक्षम करना जानबूझकर `/api/v1/admin/rpc` पर अनुमत-सूचीबद्ध एडमिन RPC विधियों की पहुँच प्रदान करता है।
- Plugin आरक्षित `contracts.gatewayMethodDispatch: ["authenticated-request"]` मैनिफ़ेस्ट अनुबंध घोषित करता है, जो इसके Gateway-प्रमाणीकृत HTTP रूट को उसी प्रक्रिया में कंट्रोल-प्लेन विधियाँ डिस्पैच करने देता है। यह सैंडबॉक्स नहीं है: अनुबंध आरक्षित SDK हेल्पर के आकस्मिक उपयोग को रोकता है, लेकिन विश्वसनीय Plugin फिर भी Gateway प्रक्रिया में चलते हैं।
- साझा-गुप्त बेयरर प्रमाणीकरण (`token`/`password` मोड) Gateway ऑपरेटर गुप्त के स्वामित्व को प्रमाणित करता है; उस पथ पर अधिक सीमित `x-openclaw-scopes` हेडर अनदेखे किए जाते हैं और सामान्य पूर्ण ऑपरेटर डिफ़ॉल्ट पुनर्स्थापित किए जाते हैं।
- विश्वसनीय पहचान-युक्त HTTP प्रमाणीकरण (`trusted-proxy` मोड) उपस्थित होने पर `x-openclaw-scopes` का सम्मान करता है।
- `gateway.auth.mode="none"` का अर्थ है कि Plugin सक्षम होने पर यह रूट अप्रमाणीकृत है। इसका उपयोग केवल ऐसे निजी इनग्रेस के पीछे करें जिस पर आपको पूर्ण विश्वास हो।
- Plugin रूट प्रमाणीकरण पास होने के बाद, अनुरोध उन्हीं Gateway विधि हैंडलर और स्कोप जाँचों के माध्यम से डिस्पैच होते हैं जिनका उपयोग WebSocket RPC करता है।
- तैयार निलंबन लीज़ के दौरान रूट पहुँच योग्य बना रहता है। सीमित अनुरोध सत्यापन और स्थानीय `commands.list` डिस्कवरी प्रतिक्रिया उपलब्ध रहती है। Gateway में डिस्पैच की जाने वाली विधियों में से, प्रवेश बंद होने पर केवल `gateway.suspend.prepare`, `gateway.suspend.status` और `gateway.suspend.resume` चल सकती हैं; अन्य अनुमत-सूचीबद्ध विधियाँ सामान्य पुनः प्रयास योग्य Gateway `UNAVAILABLE` प्रतिक्रिया लौटाती हैं।
- इस रूट को लूपबैक, टेलनेट या निजी विश्वसनीय इनग्रेस पर रखें। इसे सीधे सार्वजनिक इंटरनेट पर उजागर न करें। जब कॉलर विश्वास सीमाओं को पार करते हों, तो अलग-अलग Gateway का उपयोग करें।

## अनुरोध

```http
POST /api/v1/admin/rpc
Authorization: Bearer <gateway-token>
Content-Type: application/json
```

```json
{
  "id": "optional-request-id",
  "method": "health",
  "params": {}
}
```

फ़ील्ड:

- `id` (स्ट्रिंग, वैकल्पिक): प्रतिक्रिया में कॉपी किया जाता है। छोड़े जाने पर एक UUID जनरेट किया जाता है।
- `method` (स्ट्रिंग, आवश्यक): अनुमत Gateway विधि का नाम।
- `params` (कोई भी, वैकल्पिक): विधि-विशिष्ट पैरामीटर।

डिफ़ॉल्ट अधिकतम अनुरोध बॉडी आकार 1 MB है।

## प्रतिक्रिया

सफल प्रतिक्रियाएँ Gateway RPC आकार का उपयोग करती हैं:

```json
{
  "id": "optional-request-id",
  "ok": true,
  "payload": {}
}
```

Gateway विधि त्रुटियाँ इसका उपयोग करती हैं:

```json
{
  "id": "optional-request-id",
  "ok": false,
  "error": {
    "code": "INVALID_REQUEST",
    "message": "bad params"
  }
}
```

HTTP स्थिति त्रुटि कोड के अनुसार होती है:

| त्रुटि कोड                 | HTTP स्थिति |
| -------------------------- | ----------- |
| `INVALID_REQUEST`          | 400         |
| `APPROVAL_NOT_FOUND`       | 404         |
| `NOT_LINKED`, `NOT_PAIRED` | 409         |
| `UNAVAILABLE`              | 503         |
| `AGENT_TIMEOUT`            | 504         |
| कोई अन्य कोड             | 500         |

## अनुमत विधियाँ

- डिस्कवरी: `commands.list`
  इस Plugin द्वारा अनुमत HTTP RPC विधि नाम लौटाता है।
- Gateway: `health`, `status`, `logs.tail`, `usage.status`, `usage.cost`, `gateway.restart.request`, `gateway.suspend.prepare`, `gateway.suspend.status`, `gateway.suspend.resume`
- कॉन्फ़िगरेशन: `config.get`, `config.schema`, `config.schema.lookup`, `config.set`, `config.patch`, `config.apply`
- चैनल: `channels.status`, `channels.start`, `channels.stop`, `channels.logout`
- वेब: `web.login.start`, `web.login.wait`
- मॉडल: `models.list`, `models.authStatus`
- एजेंट: `agents.list`, `agents.create`, `agents.update`, `agents.delete`
- अनुमोदन: `exec.approvals.get`, `exec.approvals.set`, `exec.approvals.node.get`, `exec.approvals.node.set`
- Cron: `cron.status`, `cron.list`, `cron.get`, `cron.runs`, `cron.add`, `cron.update`, `cron.remove`, `cron.run`
- डिवाइस: `device.pair.list`, `device.pair.approve`, `device.pair.reject`, `device.pair.remove`
- Node: `node.list`, `node.describe`, `node.pair.list`, `node.pair.approve`, `node.pair.reject`, `node.pair.remove`, `node.rename`
- कार्य: `tasks.list`, `tasks.get`, `tasks.cancel`
- निदान: `doctor.memory.status`, `update.status`

अन्य Gateway विधियाँ तब तक अवरुद्ध रहती हैं जब तक उन्हें जानबूझकर नहीं जोड़ा जाता।

## WebSocket तुलना

सामान्य Gateway WebSocket RPC पथ OpenClaw क्लाइंट के लिए पसंदीदा कंट्रोल-प्लेन API बना रहता है। एडमिन HTTP RPC का उपयोग केवल उस होस्ट टूलिंग के लिए करें जिसे अनुरोध/प्रतिक्रिया वाली HTTP सतह की आवश्यकता हो।

विश्वसनीय डिवाइस पहचान के बिना साझा-टोकन WebSocket क्लाइंट कनेक्ट करते समय स्वयं एडमिन स्कोप घोषित नहीं कर सकते। एडमिन HTTP RPC जानबूझकर मौजूदा विश्वसनीय HTTP ऑपरेटर मॉडल का अनुसरण करता है: Plugin सक्षम होने पर, साझा-गुप्त बेयरर प्रमाणीकरण को इस एडमिन सतह के लिए पूर्ण ऑपरेटर पहुँच माना जाता है।

## समस्या निवारण

`404 Not Found`

: Plugin अक्षम है, उसे सक्षम करने के बाद से Gateway पुनः आरंभ नहीं हुआ है या अनुरोध किसी अलग Gateway प्रक्रिया पर जा रहा है।

`401 Unauthorized`

: अनुरोध Gateway HTTP प्रमाणीकरण को पूरा नहीं करता। बेयरर टोकन या विश्वसनीय-प्रॉक्सी पहचान हेडर जाँचें।

`405 Method Not Allowed`

: अनुरोध में `POST` के अलावा किसी अन्य चीज़ का उपयोग किया गया।

`413 Payload Too Large`

: अनुरोध बॉडी 1 MB की सीमा से अधिक थी।

`400 INVALID_REQUEST`

: अनुरोध बॉडी मान्य JSON नहीं है, `method` फ़ील्ड अनुपस्थित है, विधि Plugin की अनुमत सूची में नहीं है या निलंबन पुनः आरंभ ID सक्रिय लीज़ से मेल नहीं खाती।

`503 UNAVAILABLE`

: Gateway विधि आरंभ हो रही है, दर-सीमित है, निलंबित है या किसी प्रतिस्पर्धी निलंबन/पुनः आरंभ कार्रवाई की प्रतीक्षा कर रही है। उपस्थित होने पर `error.details` का निरीक्षण करें और पुनः प्रयास करने से पहले `error.retryAfterMs` का पालन करें।

## संबंधित

- [ऑपरेटर स्कोप](/hi/gateway/operator-scopes)
- [Gateway सुरक्षा](/hi/gateway/security)
- [दूरस्थ पहुँच](/hi/gateway/remote)
- [Plugin मैनिफ़ेस्ट](/hi/plugins/manifest#contracts-reference)
- [SDK उपपथ](/hi/plugins/sdk-subpaths)
