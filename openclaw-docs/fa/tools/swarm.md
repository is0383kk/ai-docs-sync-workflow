---
read_when:
    - می‌خواهید یک اسکریپت Code Mode کار را میان چند عامل توزیع کند
    - به نتایج ساخت‌یافتهٔ فرزند، دروازه‌های تصمیم‌گیری یا پایپ‌لاین‌های تکمیل نخست نیاز دارید
    - در حال فعال‌سازی یا تنظیم محدودیت‌های `tools.swarm` هستید
    - می‌خواهید فرزندان گردآورنده را در داشبورد نشست مشاهده کنید
sidebarTitle: Swarm
summary: زیرعامل‌های هم‌زمان را از اسکریپت‌های Code Mode با نتایج ساخت‌یافته، گسترش موازی محدود و پیشرفت زنده هماهنگ کنید
title: ازدحام
x-i18n:
    generated_at: "2026-07-27T17:13:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f0bec17da7a2e144df35189a65d9b35d829815b545a4bb89652e6a681ca971a9
    source_path: tools/swarm.md
    workflow: 16
---

Swarm روشی آزمایشی و اختیاری برای هماهنگ‌سازی تعداد زیادی زیرعامل از طریق یک اسکریپت
[حالت کد](/tools/code-mode) است. از جریان کنترل معمول JavaScript یا TypeScript مانند `Promise.all`، `while` و `if` برای توزیع کار، جمع‌آوری
نتایج و تصمیم‌گیری استفاده کنید.

هیچ DSL گرافی و هیچ قالب گردش‌کار جداگانه‌ای وجود ندارد. خود برنامه همان
هماهنگ‌سازی است. Swarm فرزندان جمع‌آورنده قابل‌انتظار، نتایج ساخت‌یافته،
هم‌زمانی محدود و گزارش پیشرفت را به آن برنامه اضافه می‌کند.

## فعال‌سازی Swarm

مسیر پیشنهادی **Settings → Labs → Swarm** در رابط کاربری کنترل است. این
کلید فوراً اعمال می‌شود و `tools.swarm.enabled` را در
پیکربندی شما می‌نویسد.

همچنین می‌توانید Swarm را مستقیماً در `openclaw.json` فعال کنید:

```json5
{
  tools: {
    swarm: {
      enabled: true,
      maxConcurrent: 8,
      maxChildrenPerGroup: 50,
      maxTotalPerGroup: 200,
      waitTimeoutSecondsMax: 600,
      defaultAgentId: "",
    },
  },
}
```

شکل کوتاه بولی، این قابلیت را با حفظ مقادیر پیش‌فرض همه گزینه‌های دیگر
فعال یا غیرفعال می‌کند:

```json5
{
  tools: {
    swarm: true,
  },
}
```

| فیلد                   | پیش‌فرض | توضیحات                                                                                                                    |
| ----------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `enabled`               | `false` | گزینه‌های ایجاد در حالت جمع‌آورنده، `agents_wait` و API مهمان `agents.*` در حالت کد را در دسترس قرار می‌دهد.                                   |
| `maxConcurrent`         | `8`     | حداکثر تعداد فرزندان جمع‌آورنده که به‌طور هم‌زمان در یک گروه Swarm اجرا می‌شوند. فرزندان پذیرفته‌شده اضافی به‌ترتیب FIFO در صف قرار می‌گیرند.          |
| `maxChildrenPerGroup`   | `50`    | حداکثر تعداد فرزندان جمع‌آورنده زنده در یک گروه.                                                                                  |
| `maxTotalPerGroup`      | `200`   | حداکثر تعداد فرزندان جمع‌آورنده‌ای که یک گروه می‌تواند در طول عمر خود ایجاد کند. این محدودیت نهایی برای جلوگیری از ایجاد مهارنشده است.                            |
| `waitTimeoutSecondsMax` | `600`   | حداکثر مهلت زمانی پذیرفته‌شده توسط یک فراخوانی `agents_wait`. پیش‌فرض فراخوانی 30 ثانیه است.                                            |
| `defaultAgentId`        | `""`    | عامل مقصدی که وقتی ایجاد، `agentId` را مشخص نمی‌کند استفاده می‌شود. مقدار خالی از عامل درخواست‌کننده استفاده می‌کند. فهرست‌های مجاز زیرعامل موجود اعمال می‌شوند. |

مقادیر عددی باید اعداد صحیح مثبت باشند. OpenClaw مقدار
`maxConcurrent` را به `1`–`1000`، `maxChildrenPerGroup` را به `1`–`10000`،
`maxTotalPerGroup` را به `1`–`100000` و `waitTimeoutSecondsMax` را به
`1`–`86400` محدود می‌کند.

می‌توانید Swarm را برای یک عامل پیکربندی‌شده با
`agents.entries.*.tools.swarm` بازنویسی کنید. شیء مختص عامل روی شیء سطح بالای
`tools.swarm` ادغام می‌شود.

## الزامات

متغیرهای سراسری مهمان `agents.run`، `phase` و `log` هم به Swarm و هم به
حالت کد OpenClaw نیاز دارند:

```json5
{
  tools: {
    codeMode: true,
    swarm: true,
  },
}
```

حالت کد همچنین باید دسترسی مؤثر به `sessions_spawn` داشته باشد. پروفایل‌های ابزار،
سیاست مجاز/غیرمجاز، قواعد ارائه‌دهنده و سیاست سندباکس می‌توانند آن ابزار را حذف کنند.
اگر اسکریپتی گزارش می‌دهد که `sessions_spawn`
در دسترس نیست، به [فعال‌سازی حالت کد](/tools/code-mode#activation) و
[زیرعامل‌ها](/fa/tools/subagents) مراجعه کنید.

مقادیر `defaultAgentId` و `agentId` مختص هر اجرا باید نام یک مقصد پیکربندی‌شده
و مجاز طبق سیاست `subagents.allowAgents` درخواست‌کننده را مشخص کنند. OpenClaw
به‌جای بازگشت به عامل دیگری، مقصد ناشناخته یا غیرمجاز را رد می‌کند.

## نوشتن اسکریپت Swarm

وقتی Swarm فعال باشد، حالت کد این API مهمان را در دسترس قرار می‌دهد:

```typescript
type AgentRunOptions = {
  label?: string;
  model?: string;
  thinking?: string;
  fastMode?: boolean | "auto";
  agentId?: string;
  schema?: Record<string, unknown>;
  phase?: string;
};

agents.run(prompt: string, options?: AgentRunOptions & { schema?: undefined }): Promise<string>;
agents.run<T>(prompt: string, options: AgentRunOptions & { schema: Record<string, unknown> }): Promise<T>;
phase(title: string): void;
log(message: string): void;
```

بدون `schema`، مقدار `agents.run()` به متن نهایی فرزند تبدیل می‌شود. با یک
JSON Schema، مقدار آن به مقداری تبدیل می‌شود که از طریق ابزار
`structured_output` فرزند ارسال شده است. فرزند ناموفق، متوقف‌شده، منقضی‌شده یا دارای طرح‌واره نامعتبر،
وعده را با یک `SwarmAgentError` رد می‌کند. اعلان‌های تولیدشده دقیق
و الگوهای کوتاه هماهنگ‌سازی را از `API.read("agents.d.ts")`
درون حالت کد بخوانید.

برای نامی قابل‌شناسایی برای فرزند در داشبورد و نوار کناری از `label` استفاده کنید. برای انتشار یک مرحله درست پیش از شروع آن فرزند،
از `phase` در گزینه‌ها استفاده کنید، یا هنگامی‌که چند فرزند به یک مرحله تعلق دارند،
`phase()` را فراخوانی کنید.
`log()` یک یادداشت کوتاه پیشرفت منتشر می‌کند. فراخوانی‌های پیشرفت بدون انتظار اجرا می‌شوند؛
اگر رابط کاربری در دسترس نباشد، اسکریپت را به تأخیر نمی‌اندازند.

### توزیع موازی با نتایج ساخت‌یافته

این مثال برای هر موضوع یک پژوهشگر راه‌اندازی می‌کند، منتظر همه آن‌ها می‌ماند و سپس
از فرزند نهایی می‌خواهد گزارش‌های ساخت‌یافته آن‌ها را ترکیب کند:

```javascript
const reportSchema = {
  type: "object",
  properties: {
    finding: { type: "string" },
    evidence: { type: "array", items: { type: "string" } },
    confidence: { type: "number" },
  },
  required: ["finding", "evidence", "confidence"],
  additionalProperties: false,
};

const topics = ["احراز هویت", "ذخیره‌سازی", "بازیابی"];
phase("بازبینی مستقل");

const reports = await Promise.all(
  topics.map((topic) =>
    agents.run(`مسیر ${topic} را بازبینی کنید. یک یافته همراه با شواهد بازگردانید.`, {
      label: `review-${topic}`,
      thinking: "high",
      fastMode: "auto",
      schema: reportSchema,
    }),
  ),
);

phase("ترکیب");
log(`${reports.length} گزارش مستقل جمع‌آوری شد.`);

return await agents.run(
  `این گزارش‌ها را تطبیق دهید و اختلاف‌ها را توضیح دهید:\n${JSON.stringify(reports)}`,
  { label: "synthesis" },
);
```

`Promise.all` مرز توزیع و تجمیع است. OpenClaw تا
`maxConcurrent` فرزند را برای گروه آغاز می‌کند و بقیه را به‌ترتیب ارسال
در صف قرار می‌دهد.

حالت کد به‌طور جداگانه فراخوانی‌های هم‌زمان پل مهمان را با
`tools.codeMode.maxPendingToolCalls` (پیش‌فرض `16`، حداکثر `128`) محدود می‌کند. برای گروه‌های بسیار
بزرگ، دسته‌های محدود را پایین‌تر از آن حد راه‌اندازی کنید و برای
`phase()`، `log()` و گذارهای انتظار فرزند ظرفیت آزاد باقی بگذارید. `maxConcurrent` تعداد فرزندان
در حال اجرا را محدود می‌کند؛ حد فراخوانی پل مهمان را افزایش نمی‌دهد.

### تکرار بر اساس دروازه تصمیم‌گیری

وقتی هر گذر تعیین می‌کند آیا گذر دیگری لازم است، از یک حلقه محدود
`while` استفاده کنید:

```javascript
const gateSchema = {
  type: "object",
  properties: {
    ready: { type: "boolean" },
    reason: { type: "string" },
    nextAction: { type: "string" },
  },
  required: ["ready", "reason", "nextAction"],
  additionalProperties: false,
};

let pass = 0;
let decision = { ready: false, reason: "بررسی نشده", nextAction: "بازبینی" };

while (!decision.ready && pass < 4) {
  pass += 1;
  phase(`گذر تصمیم‌گیری ${pass}`);
  decision = await agents.run(
    `بررسی کنید آیا شواهد انتشار کامل است. تصمیم قبلی: ${JSON.stringify(decision)}`,
    {
      label: `release-gate-${pass}`,
      schema: gateSchema,
    },
  );
  log(decision.reason);
}

if (!decision.ready) {
  throw new Error(`دروازه پس از ${pass} گذر همچنان بسته است: ${decision.nextAction}`);
}

return decision;
```

همیشه حلقه‌های تصمیم‌گیری را محدود کنید. `maxTotalPerGroup` آخرین سازوکار ایمنی است،
نه جایگزینی برای یک شرط توقف روشن.

### پردازش نخستین فرزند تکمیل‌شده

`agents.run()` یک وعده عادی بازمی‌گرداند، بنابراین `Promise.race` می‌تواند به
نخستین فرزند حالت کد واکنش نشان دهد. برای چارچوب‌هایی که ابزارهای سطح پایین‌تر را فراخوانی می‌کنند،
`agents_wait` همان مرز نخستین تکمیل را فراهم می‌کند: به‌محض
تکمیل حداقل یکی از اجراهای درخواست‌شده یا پایان مهلت زمانی محدود، بازمی‌گردد.
برای حلقه تخلیه کامل، به [استفاده از Swarm در چارچوب‌های دیگر](#use-swarm-from-other-harnesses) مراجعه کنید.

## رفتار فرزندان جمع‌آورنده

فرزندان جمع‌آورنده نشست‌های زیرعامل عادی و ایزوله‌ای با مسیر تکمیل متفاوت
هستند. آن‌ها نتیجه‌ای پایدار برای جمع‌آورنده می‌نویسند تا والد منتظر آن بماند،
به‌جای آن‌که پاسخی را اعلام کنند یا به نشست والد بازگردانند.

عامل مقصد به این ترتیب تعیین می‌شود:

1. `agentId` در فراخوانی ایجاد یا `agents.run()`.
2. `tools.swarm.defaultAgentId`.
3. عامل درخواست‌کننده.

یک عامل کارگر اختصاصی و سبک زمانی مفید است که فرزندان Swarm به سطح ابزار کوچک‌تر،
مدل ارزان‌تر یا سیاست سندباکس سخت‌گیرانه‌تری نیاز دارند. OpenClaw هیچ شناسه عامل داخلی
`worker` ارائه نمی‌کند؛ پیش از تعیین آن به‌عنوان پیش‌فرض، یکی را پیکربندی کنید.
آن کارگر را با `tools.swarm: false` در پیکربندی مختص عامل آن سخت‌سازی کنید تا
قابل ایجاد باشد، اما نتواند از نشست‌های سطح بالای خود Swarm آغاز کند:

```json5
{
  tools: { swarm: { enabled: true, defaultAgentId: "worker" } },
  agents: {
    list: [
      {
        id: "main",
        default: true,
        subagents: { allowAgents: ["worker"] },
      },
      { id: "worker", tools: { swarm: false } },
    ],
  },
}
```

تأییدهای جمع‌آورنده در صورت نبود مجوز رد می‌شوند. فرزند هرگز درخواست تأیید
اپراتور را باز نمی‌کند. اقدام ابزاری که به تأیید نیاز داشته باشد رد می‌شود و فرزند می‌تواند
این رد را در نتیجه خود گزارش کند تا اسکریپت درباره اقدام بعدی تصمیم بگیرد.

برای خروجی ساخت‌یافته، OpenClaw یک ابزار مصنوعی `structured_output` به
فرزند اضافه می‌کند و بار آن را با JSON Schema ارائه‌شده اعتبارسنجی می‌کند. برای
بار نامعتبر یا مفقود، یک یادآوری اصلاحی ارسال می‌شود. اگر تلاش مجدد نیز
اعتبارسنجی نشود، تکمیل جمع‌آورنده متن خام فرزند را حفظ می‌کند،
`structured` را تنظیم‌نشده باقی می‌گذارد و `schemaError` را دربر می‌گیرد. نتیجه سطح پایین `agents_wait`
این فیلدها را برای منطق بازیابی صریح در دسترس قرار می‌دهد.

### فرزندان برگ هستند

فرزندان Swarm به‌طور پیش‌فرض برگ هستند. محافظ عمومی
`agents.defaults.subagents.maxSpawnDepth` از ایجاد
فرزندان خودِ فرزند در عمق پیش‌فرض `1` جلوگیری می‌کند. الگوی معمول هماهنگ‌سازی این است
که کار به والد بازگردانده شود، نه این‌که فرزند کار بیشتری ایجاد کند:

```javascript
const plan = await agents.run("این کار را به‌صورت وظایف مستقل برنامه‌ریزی کنید.", {
  schema: {
    type: "object",
    properties: { tasks: { type: "array", items: { type: "string" } } },
    required: ["tasks"],
    additionalProperties: false,
  },
});
return await Promise.all(plan.tasks.map((task) => agents.run(task)));
```

زیرعامل‌های تودرتو گزینه‌ای اختیاری برای اپراتور از طریق
`agents.defaults.subagents.maxSpawnDepth` هستند و برای Swarm توصیه نمی‌شوند.
محدودیت‌های گروه، بودجه‌ها و مشاهده‌پذیری همگی گروه‌های جمع‌آورنده تخت را فرض می‌کنند.

هر فرزند یک مالک پذیرش دارد. فرزندان اعلامی و تعاملی از
`agents.defaults.subagents.maxChildrenPerAgent` (پیش‌فرض `5`) استفاده می‌کنند و فرزندان
جمع‌آورنده را محاسبه نمی‌کنند. فرزندان جمع‌آورنده فقط از `maxChildrenPerGroup` و
`maxTotalPerGroup` استفاده می‌کنند؛ آن‌ها بودجه فرزند مختص نشست را مصرف نمی‌کنند. محافظ
عمق ایجاد همچنان بر هر دو حالت اعمال می‌شود.

پس از پذیرش، فرزندان بالاتر از `maxConcurrent` به‌ترتیب FIFO در گروه Swarm خود
و درون مسیر سراسری زیرعامل در صف قرار می‌گیرند. این لایه‌های هم‌زمانی به‌جای رد کردن،
کار را در صف قرار می‌دهند. ایجاد جمع‌آورنده‌ای که از هرکدام از محدودیت‌های گروه عبور کند،
با کلید پیکربندی مربوطه در خطا رد می‌شود.

## مشاهده یک Swarm

هنگامی‌که Swarm فعال است، داشبورد نشست والد را در رابط کاربری کنترل باز کنید.
ویجت Swarm هر گروه جمع‌آورنده فعال را به‌شکل یک نقطه برای هر فرزند، با وضعیت
در صف، در حال اجرا، انجام‌شده یا ناموفق نمایش می‌دهد. برچسب‌ها در راهنمای نقاط ظاهر می‌شوند، بنابراین برچسب‌های کوتاه
و پایدار خواندن Swarmهای بزرگ‌تر را آسان‌تر می‌کنند.

نوار کناری نشست، درخت عادی والد/فرزند را حفظ می‌کند. ردیف والد را باز کنید
تا یک فرزند جمع‌آورنده را بررسی کنید یا رونوشت آن را بدون از دست دادن سلسله‌مراتب Swarm
باز کنید.

نتایج گردآورنده تا زمانی که گروهشان بایگانی شود، قابل انتظار باقی می‌مانند. پس از آنکه همهٔ
اعضا به مهلت نگه‌داری خود رسیدند، OpenClaw فرزندان گروه را به‌صورت دسته‌ای
بایگانی می‌کند تا ازدحام‌های تکمیل‌شده در درخت نشست فعال باقی نمانند.

## استفاده از Swarm در چارچوب‌های اجرایی دیگر

می‌توان از Swarm بدون Code Mode در OpenClaw استفاده کرد. ابزارهای اصلی آن
مستقل از چارچوب اجرایی هستند: فرزندان گردآورنده را با
`sessions_spawn({ collect: true })` آغاز کنید و با فراخوانی‌های محدود `agents_wait`
آن‌ها را تخلیه کنید.

Codex Code Mode به‌طور خودکار ابزارهای پویا و واجد شرایط OpenClaw را زیر
`tools.*` در دسترس قرار می‌دهد. این حالت از API مهمان QuickJS متعلق به OpenClaw استفاده نمی‌کند و به
`tools.codeMode` نیاز ندارد، اما `tools.swarm` همچنان باید فعال باشد. فراخوانی‌های
`agents_wait` در چارچوب اجرایی Codex از مهلت زمانی کامل 600 ثانیه‌ای پشتیبانی می‌کنند.

در زمان اجرای Codex که در حال حاضر پشتیبانی می‌شود، نتایج ابزارهای پویای OpenClaw به‌شکل
متن JSON به Code Mode می‌رسند. پیش از خواندن فیلدها، هر نتیجه را تجزیه کنید. Codex همچنین
فراخوانی‌های ابزار پویا را به‌صورت ترتیبی اجرا می‌کند، بنابراین `Promise.all` چندین فراخوانی
`sessions_spawn` را هم‌زمان ارسال نمی‌کند. گردآورنده‌ها را در یک حلقهٔ محدود راه‌اندازی کنید؛
فرزندان ازپیش‌پذیرفته‌شده می‌توانند هنگام ارسال راه‌اندازی‌های بعدی همچنان اجرا شوند.

```javascript
function parseToolResult(value) {
  if (typeof value !== "string") return value;
  return JSON.parse(value);
}

const tasks = [
  "مسیر احراز هویت را بررسی کنید.",
  "مسیر ذخیره‌سازی را بررسی کنید.",
  "مسیر بازیابی را بررسی کنید.",
];
const launches = [];

for (const [index, task] of tasks.entries()) {
  const launch = parseToolResult(
    await tools.sessions_spawn({
      task,
      collect: true,
      label: `review-${index + 1}`,
    }),
  );
  if (launch.status !== "accepted") {
    throw new Error(launch.error ?? "راه‌اندازی گردآورنده پذیرفته نشد.");
  }
  launches.push(launch);
}

const pending = new Set(launches.map((launch) => launch.runId));
const completed = [];

while (pending.size > 0) {
  const ids = [...pending].slice(0, 1000);
  const batch = parseToolResult(
    await tools.agents_wait({
      ids,
      timeoutSeconds: 30,
    }),
  );

  // این پنجرهٔ محدود را به پشت شناسه‌هایی که هنوز بررسی نشده‌اند بچرخانید.
  for (const runId of ids) {
    if (pending.delete(runId)) pending.add(runId);
  }

  for (const item of batch.completed) {
    pending.delete(item.runId);
    if (item.status !== "done") {
      throw new Error(item.schemaError ?? item.result ?? `${item.runId}: ${item.status}`);
    }
    completed.push(item); // هر نتیجه را به‌محض پایان پردازش کنید.
  }

  for (const failure of batch.errors ?? []) {
    pending.delete(failure.runId);
    throw new Error(`${failure.runId}: ${failure.error}`);
  }
}

return completed;
```

هر فراخوانی `agents_wait` تعداد 1–1000 شناسهٔ اجرا را می‌پذیرد. خروجی آن:

```typescript
type AgentsWaitResult = {
  completed: Array<{
    runId: string;
    status: "done" | "failed" | "killed" | "timeout";
    result: string;
    structured?: unknown;
    schemaError?: string;
    sessionKey: string;
    label?: string;
    usage?: { inputTokens: number; outputTokens: number };
  }>;
  pending: string[];
  errors?: Array<{
    runId: string;
    error: "not_found" | "not_owner";
  }>;
};
```

فراخوانی زمانی فوراً بازمی‌گردد که هرکدام از فرزندان درخواست‌شده از قبل کامل شده باشد،
دست‌کم یک فرزند معلق کامل شود، هیچ شناسهٔ معلق معتبری باقی نماند،
یا مهلت زمانی آن به پایان برسد. رکوردهای تکمیل‌شده هم‌توان هستند، بنابراین ارسال
شناسهٔ اجرای ازپیش‌تکمیل‌شده، نتیجهٔ آن را دوباره بازمی‌گرداند. فقط نشست ایجادکننده
یا زنجیرهٔ والد مجاز آن می‌تواند منتظر یک گردآورنده بماند.

این یک نظرسنجی بلندمدت محدود است، نه یک حلقهٔ پرتکرار وضعیت. فقط
شناسه‌های اجرای باقی‌مانده را ارسال کنید تا `pending` خالی شود. حالت گردآورنده از زیرعامل‌های بومی
OpenClaw پشتیبانی می‌کند؛ این حالت از زمان اجرای ACP، اتصال رشته، نشست‌های قابل‌مشاهده
یا حالت نشست پایدار پشتیبانی نمی‌کند.

## محدودیت‌ها و نقشهٔ راه

Swarm v1 فرزندان گردآورندهٔ یک‌مرحله‌ای را اجرا می‌کند؛ API برنامه‌ریزی‌شدهٔ `agents.session()`
کارگرهای چندمرحله‌ای دارای وضعیت را اضافه خواهد کرد. فرزندان در حال حاضر در مسیر زیرعامل
Gateway محلی اجرا می‌شوند؛ استقرار ابری به‌عنوان گزینه‌ای صریح برای ایجاد برنامه‌ریزی شده است.
تعریف‌های گردش‌کار ذخیره‌شده و DSL گراف بخشی از جهت‌گیری فعلی Swarm نیستند.

## مرتبط

- [Code Mode](/tools/code-mode) برای زمان اجرای مهمان QuickJS و قواعد فعال‌سازی
- [زیرعامل‌ها](/fa/tools/subagents) برای خط‌مشی فرزند، جداسازی و رفتار نشست
- [ابزارهای محیط ایزولهٔ چندعاملی](/fa/tools/multi-agent-sandbox-tools) برای محدودیت‌های مختص هر عامل
- [نمای کلی ابزارها](/fa/tools) برای پروفایل ابزارها و مسیریابی خط‌مشی
