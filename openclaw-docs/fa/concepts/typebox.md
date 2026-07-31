---
read_when:
    - به‌روزرسانی طرح‌واره‌های پروتکل یا تولید کد
summary: طرح‌واره‌های TypeBox به‌عنوان تنها منبع حقیقت برای پروتکل Gateway
title: TypeBox
x-i18n:
    generated_at: "2026-07-27T15:12:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 24490edf0d73e918f834e9dd53d09ba0e5183b2bc126ee981a94f8099e76283b
    source_path: concepts/typebox.md
    workflow: 16
---

TypeBox یک کتابخانهٔ طرح‌واره با رویکرد TypeScript-first است. OpenClaw از آن برای تعریف **پروتکل WebSocket مربوط به Gateway** (دست‌دهی، درخواست/پاسخ، رویدادهای سرور) استفاده می‌کند. این طرح‌واره‌ها **اعتبارسنجی زمان اجرا** (AJV)، **صدور JSON Schema** و **تولید کد Swift** برای برنامهٔ macOS را هدایت می‌کنند. یک منبع حقیقت وجود دارد؛ هر چیز دیگری تولید می‌شود.

برای آشنایی با زمینهٔ سطح‌بالاتر پروتکل، از [معماری Gateway](/fa/concepts/architecture) شروع کنید.

## مدل ذهنی (30 ثانیه)

هر پیام WS مربوط به Gateway یکی از سه فریم زیر است:

- **درخواست**: `{ type: "req", id, method, params }`
- **پاسخ**: `{ type: "res", id, ok, payload | error }`
- **رویداد**: `{ type: "event", event, payload, seq?, stateVersion? }`

فریم نخست **باید** یک درخواست `connect` باشد. پس از آن، کلاینت‌ها متدها را فراخوانی می‌کنند (برای مثال `health`، `send`، `chat.send`) و در رویدادها مشترک می‌شوند (برای مثال `presence`، `tick`، `agent`).

جریان اتصال (حداقلی):

```text
کلاینت                    Gateway
  |---- درخواست:اتصال ------->|
  |<---- پاسخ:تأیید-سلام ------|
  |<---- رویداد:تیک -----------|
  |---- درخواست:سلامت -------->|
  |<---- پاسخ:سلامت -----------|
```

متدها و رویدادهای رایج:

| دسته       | نمونه‌ها                                                    | نکات                                         |
| ---------- | ---------------------------------------------------------- | -------------------------------------------- |
| هسته       | `connect`، `health`، `status`                              | `connect` باید نخست باشد                      |
| پیام‌رسانی | `send`، `agent`، `agent.wait`، `system-event`، `logs.tail` | متدهای دارای اثر جانبی به `idempotencyKey` نیاز دارند |
| گفت‌وگو    | `chat.history`، `chat.send`، `chat.abort`                  | WebChat از این‌ها استفاده می‌کند             |
| نشست‌ها    | `sessions.list`، `sessions.patch`، `sessions.delete`       | مدیریت نشست                                  |
| خودکارسازی | `wake`، `cron.list`، `cron.run`، `cron.runs`               | کنترل بیدارسازی و Cron                       |
| Nodeها     | `node.list`، `node.invoke`، `node.pair.*`                  | WS مربوط به Gateway به‌همراه کنش‌های Node    |
| رویدادها   | `tick`، `presence`، `agent`، `chat`، `health`، `shutdown`  | ارسال از سوی سرور                            |

فهرست مرجع و معتبر **کشف قابلیت‌ها** در `src/gateway/server-methods-list.ts` قرار دارد (`listGatewayMethods`، `GATEWAY_EVENTS`).

## محل قرارگیری طرح‌واره‌ها

- فایل تجمیعی منبع: `packages/gateway-protocol/src/schema.ts` ماژول‌های دامنه را در زیر `packages/gateway-protocol/src/schema/*.ts` دوباره صادر می‌کند (`frames.ts` برای پوش‌های سطح‌بالا و دست‌دهی، و `agent.ts`، `sessions.ts`، `cron.ts` و غیره برای هر حوزهٔ قابلیت). `protocol-schemas.ts` رجیستری مرکزی `ProtocolSchemas` است که نام طرح‌واره‌ها را به تعریف‌های TypeBox آن‌ها نگاشت می‌کند.
- اعتبارسنج‌های زمان اجرا (AJV): `packages/gateway-protocol/src/index.ts`
- رجیستری اعلام‌شدهٔ قابلیت‌ها/کشف: `src/gateway/server-methods-list.ts`
- دست‌دهی سرور و توزیع متدها: `src/gateway/server.impl.ts`
- کلاینت Node: `src/gateway/client.ts`
- JSON Schema تولیدشده: `dist/protocol.schema.json` (خروجی ساخت، ثبت‌نشده در مخزن)
- مدل‌های Swift تولیدشده: `apps/shared/OpenClawKit/Sources/OpenClawProtocol/GatewayModels.swift`

## پایپ‌لاین کنونی

- `pnpm protocol:gen`، JSON Schema (draft-07) را در `dist/protocol.schema.json` می‌نویسد.
- `pnpm protocol:gen:swift` مدل‌های Swift مربوط به Gateway را تولید می‌کند.
- `pnpm protocol:check` هر دو مولد را اجرا و بررسی می‌کند که خروجی Swift ثبت شده باشد (خروجی JSON Schema یک آرتیفکت ساختِ نادیده‌گرفته‌شده توسط git است).

## نحوهٔ استفاده از طرح‌واره‌ها در زمان اجرا

- **سمت سرور**: هر فریم ورودی با AJV اعتبارسنجی می‌شود. دست‌دهی فقط درخواست `connect` را می‌پذیرد که پارامترهایش با `ConnectParams` مطابقت داشته باشند.
- **سمت کلاینت**: کلاینت JS پیش از استفاده از فریم‌های رویداد و پاسخ، آن‌ها را اعتبارسنجی می‌کند.
- **کشف قابلیت‌ها**: Gateway فهرستی محافظه‌کارانه از `features.methods` و `features.events` را در `hello-ok` و از `listGatewayMethods()` و `GATEWAY_EVENTS` ارسال می‌کند.
- این فهرست کشف، تخلیه‌ای تولیدشده از تمام کمک‌تابع‌های قابل فراخوانی در `coreGatewayHandlers` نیست؛ برخی RPCهای کمکی در `src/gateway/server-methods/*.ts` پیاده‌سازی شده‌اند، بدون آنکه در فهرست قابلیت‌های اعلام‌شده برشمرده شوند.

## نمونه‌فریم‌ها

اتصال (پیام نخست):

```json
{
  "type": "req",
  "id": "c1",
  "method": "connect",
  "params": {
    "minProtocol": 3,
    "maxProtocol": 4,
    "client": {
      "id": "openclaw-macos",
      "displayName": "macos",
      "version": "1.0.0",
      "platform": "macos 15.1",
      "mode": "ui",
      "instanceId": "A1B2"
    }
  }
}
```

پاسخ hello-ok:

```json
{
  "type": "res",
  "id": "c1",
  "ok": true,
  "payload": {
    "type": "hello-ok",
    "protocol": 4,
    "server": { "version": "dev", "connId": "ws-1" },
    "features": { "methods": ["health"], "events": ["tick"] },
    "snapshot": {
      "presence": [],
      "health": {},
      "stateVersion": { "presence": 0, "health": 0 },
      "uptimeMs": 0
    },
    "auth": { "role": "operator", "scopes": ["operator.read"] },
    "policy": { "maxPayload": 1048576, "maxBufferedBytes": 1048576, "tickIntervalMs": 30000 }
  }
}
```

درخواست و پاسخ:

```json
{ "type": "req", "id": "r1", "method": "health" }
```

```json
{ "type": "res", "id": "r1", "ok": true, "payload": { "ok": true } }
```

رویداد:

```json
{ "type": "event", "event": "tick", "payload": { "ts": 1730000000 }, "seq": 12 }
```

## کلاینت حداقلی (Node.js)

کوچک‌ترین جریان کاربردی: اتصال + سلامت.

```ts
import { WebSocket } from "ws";

const ws = new WebSocket("ws://127.0.0.1:18789");

ws.on("open", () => {
  ws.send(
    JSON.stringify({
      type: "req",
      id: "c1",
      method: "connect",
      params: {
        minProtocol: 4,
        maxProtocol: 4,
        client: {
          id: "cli",
          displayName: "example",
          version: "dev",
          platform: "node",
          mode: "cli",
        },
      },
    }),
  );
});

ws.on("message", (data) => {
  const msg = JSON.parse(String(data));
  if (msg.type === "res" && msg.id === "c1" && msg.ok) {
    ws.send(JSON.stringify({ type: "req", id: "h1", method: "health" }));
  }
  if (msg.type === "res" && msg.id === "h1") {
    console.log("health:", msg.payload);
    ws.close();
  }
});
```

## نمونهٔ عملی: افزودن یک متد به‌صورت سرتاسری

نمونه: افزودن یک درخواست جدید `system.echo` که `{ ok: true, text }` را برمی‌گرداند.

1. **طرح‌واره (منبع حقیقت)**

آن را به `packages/gateway-protocol/src/schema/system.ts` (یا نزدیک‌ترین ماژول قابلیت منطبق) اضافه کنید:

```ts
export const SystemEchoParamsSchema = Type.Object(
  { text: NonEmptyString },
  { additionalProperties: false },
);

export const SystemEchoResultSchema = Type.Object(
  { ok: Type.Boolean(), text: NonEmptyString },
  { additionalProperties: false },
);
```

هر دو را در `packages/gateway-protocol/src/schema/protocol-schemas.ts` وارد کنید، آن‌ها را به رجیستری `ProtocolSchemas` بیفزایید و نوع‌های مشتق‌شده را صادر کنید:

```ts
  SystemEchoParams: SystemEchoParamsSchema,
  SystemEchoResult: SystemEchoResultSchema,
```

```ts
export type SystemEchoParams = Static<typeof SystemEchoParamsSchema>;
export type SystemEchoResult = Static<typeof SystemEchoResultSchema>;
```

2. **اعتبارسنجی**

در `packages/gateway-protocol/src/index.ts` یک اعتبارسنج AJV صادر کنید:

```ts
export const validateSystemEchoParams = ajv.compile<SystemEchoParams>(SystemEchoParamsSchema);
```

3. **رفتار سرور**

یک مدیریت‌کننده در `src/gateway/server-methods/system.ts` اضافه کنید:

```ts
export const systemHandlers: GatewayRequestHandlers = {
  "system.echo": ({ params, respond }) => {
    const text = String(params.text ?? "");
    respond(true, { ok: true, text });
  },
};
```

آن را در `src/gateway/server-methods.ts` ثبت کنید (که از قبل `systemHandlers` را ادغام می‌کند)، سپس `"system.echo"` را به ورودی `listGatewayMethods` در `src/gateway/server-methods-list.ts` اضافه کنید.

اگر متد توسط کلاینت‌های اپراتور یا Node قابل فراخوانی است، آن را در `src/gateway/method-scopes.ts` نیز طبقه‌بندی کنید تا اعمال محدودهٔ دسترسی و اعلام قابلیت `hello-ok` هم‌راستا بمانند.

4. **تولید مجدد**

```bash
pnpm protocol:check
```

5. **آزمون‌ها و مستندات**

یک آزمون سرور در `src/gateway/server.*.test.ts` اضافه کنید و متد را در مستندات ذکر کنید.

## رفتار تولید کد Swift

مولد Swift موارد زیر را تولید می‌کند:

- یک enum از نوع `GatewayFrame` با حالت‌های `req`، `res`، `event` و `unknown`
- ساختارها/enumهای payload با نوع‌دهی قوی
- مقادیر `ErrorCode`، `GATEWAY_PROTOCOL_VERSION` و `GATEWAY_MIN_PROTOCOL_VERSION`

نوع‌های ناشناختهٔ فریم برای سازگاری رو‌به‌جلو به‌شکل payload خام حفظ می‌شوند.

## نسخه‌بندی و سازگاری

- `PROTOCOL_VERSION` در `packages/gateway-protocol/src/version.ts` قرار دارد (مقدار کنونی: `4`).
- کلاینت‌ها `minProtocol` و `maxProtocol` را ارسال می‌کنند؛ سرور بازه‌هایی را که پروتکل کنونی آن را شامل نمی‌شوند رد می‌کند.
- مدل‌های Swift برای جلوگیری از خرابی کلاینت‌های قدیمی‌تر، نوع‌های ناشناختهٔ فریم را حفظ می‌کنند.

## الگوها و قراردادهای طرح‌واره

- بیشتر اشیا برای payloadهای سخت‌گیرانه از `additionalProperties: false` استفاده می‌کنند.
- `NonEmptyString` (`Type.String({ minLength: 1 })`) مقدار پیش‌فرض برای شناسه‌ها و نام متدها/رویدادها است.
- `GatewayFrame` سطح‌بالا روی `type` از یک **تمایزدهنده** استفاده می‌کند.
- متدهای دارای اثر جانبی معمولاً در پارامترها به یک `idempotencyKey` نیاز دارند (نمونه: `send`، `poll`، `agent`، `chat.send`).
- `agent` یک `internalEvents` اختیاری را برای زمینهٔ هماهنگ‌سازی تولیدشده در زمان اجرا می‌پذیرد (برای مثال تحویل تکمیل وظیفهٔ زیرعامل/Cron)؛ این مورد را سطح API داخلی در نظر بگیرید.

## JSON زندهٔ طرح‌واره

JSON Schema تولیدشده یک آرتیفکت ساخت است و در مخزن ثبت نمی‌شود. فایل خام منتشرشده معمولاً در این نشانی در دسترس است:

- [https://raw.githubusercontent.com/openclaw/openclaw/main/dist/protocol.schema.json](https://raw.githubusercontent.com/openclaw/openclaw/main/dist/protocol.schema.json)

## هنگام تغییر طرح‌واره‌ها

1. طرح‌واره‌های TypeBox را در ماژول مالک `packages/gateway-protocol/src/schema/*.ts` به‌روزرسانی و آن‌ها را در `protocol-schemas.ts` ثبت کنید.
2. متد/رویداد را در `src/gateway/server-methods-list.ts` ثبت کنید.
3. هنگامی که RPC جدید به طبقه‌بندی محدودهٔ اپراتور یا Node نیاز دارد، `src/gateway/method-scopes.ts` را به‌روزرسانی کنید.
4. `pnpm protocol:check` را اجرا کنید.
5. مدل‌های Swift دوباره‌تولیدشده را ثبت کنید.

## مرتبط

- [پروتکل خروجی غنی](/fa/reference/rich-output-protocol)
- [آداپتورهای RPC](/fa/reference/rpc)
