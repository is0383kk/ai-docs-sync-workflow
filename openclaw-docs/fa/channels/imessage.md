---
read_when:
    - راه‌اندازی پشتیبانی از iMessage
    - اشکال‌زدایی ارسال/دریافت iMessage
summary: پشتیبانی بومی از iMessage از طریق imsg (‏JSON-RPC روی stdio)، همراه با کنش‌های API خصوصی برای پاسخ‌ها، واکنش‌های tapback، جلوه‌ها، نظرسنجی‌ها، پیوست‌ها و مدیریت گروه. برای راه‌اندازی‌های جدید iMessage در OpenClaw، در صورت سازگاری الزامات میزبان، گزینهٔ ترجیحی است.
title: iMessage
x-i18n:
    generated_at: "2026-07-27T14:56:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f3e8b1a65c76b25d03615c06a976f86a8af555cd96d5bfdb10cef9c955893ddc
    source_path: channels/imessage.md
    workflow: 16
---

<Note>
برای استقرار معمول iMessage در OpenClaw، Gateway و `imsg` را روی همان میزبان macOS واردشده به Messages اجرا کنید. اگر Gateway شما جای دیگری اجرا می‌شود، `channels.imessage.cliPath` را به یک پوشش شفاف SSH هدایت کنید که `imsg` را روی Mac اجرا می‌کند.

**بازیابی ورودی خودکار است.** پس از راه‌اندازی مجدد پل یا Gateway، iMessage پیام‌هایی را که هنگام توقف از دست رفته‌اند بازپخش می‌کند و «بمب انباشت» قدیمی را که Apple ممکن است پس از بازیابی Push تخلیه کند سرکوب می‌کند؛ همچنین با حذف موارد تکراری مانع می‌شود چیزی دو بار ارسال شود. برای فعال‌سازی نیازی به پیکربندی نیست — [بازیابی ورودی پس از راه‌اندازی مجدد پل یا Gateway](#inbound-recovery-after-a-bridge-or-gateway-restart) را ببینید.
</Note>

<Warning>
پشتیبانی از BlueBubbles حذف شده است. پیکربندی‌های `channels.bluebubbles` را به `channels.imessage` مهاجرت دهید؛ OpenClaw فقط از طریق `imsg` از iMessage پشتیبانی می‌کند. برای اطلاعیه کوتاه، با [حذف BlueBubbles و مسیر imsg برای iMessage](/fa/announcements/bluebubbles-imessage) شروع کنید، یا برای جدول کامل مهاجرت به [مهاجرت از BlueBubbles](/fa/channels/imessage-from-bluebubbles) مراجعه کنید.
</Warning>

وضعیت: یکپارچه‌سازی بومی با CLI خارجی. Gateway، `imsg rpc` را اجرا می‌کند و از طریق stdio با JSON-RPC ارتباط می‌گیرد — بدون daemon یا درگاه جداگانه. برای داشتن یک کانال کامل iMessage، حالت API خصوصی قویاً توصیه می‌شود؛ پاسخ‌ها، tapbackها، جلوه‌ها، نظرسنجی‌ها، پاسخ به پیوست‌ها و عملیات گروهی به `imsg launch` و کاوش موفق API خصوصی نیاز دارند.

برای راه‌اندازی محلی رایج، راه‌انداز OpenClaw می‌تواند نصب یا به‌روزرسانی `imsg` با Homebrew را، پس از تأیید کاربر، روی Mac واردشده به Messages پیشنهاد دهد. راه‌اندازی دستی و توپولوژی‌های پوشش SSH همچنان تحت مدیریت اپراتور هستند: `imsg` را در همان زمینه کاربری نصب یا به‌روزرسانی کنید که Gateway یا پوشش را اجرا خواهد کرد.

<CardGroup cols={3}>
  <Card title="عملیات API خصوصی" icon="wand-sparkles" href="#private-api-actions">
    پاسخ‌ها، tapbackها، جلوه‌ها، نظرسنجی‌ها، پیوست‌ها و مدیریت گروه.
  </Card>
  <Card title="جفت‌سازی" icon="link" href="/fa/channels/pairing">
    پیام‌های خصوصی iMessage به‌طور پیش‌فرض از حالت جفت‌سازی استفاده می‌کنند.
  </Card>
  <Card title="Mac راه‌دور" icon="terminal" href="#remote-mac-over-ssh">
    وقتی Gateway روی Mac مربوط به Messages اجرا نمی‌شود، از یک پوشش SSH استفاده کنید.
  </Card>
  <Card title="مرجع پیکربندی" icon="settings" href="/fa/gateway/config-channels#imessage">
    مرجع کامل فیلدهای iMessage.
  </Card>
</CardGroup>

## راه‌اندازی سریع

<Tabs>
  <Tab title="Mac محلی (مسیر سریع)">
    <Steps>
      <Step title="نصب و تأیید imsg">

```bash
brew install steipete/tap/imsg
brew update && brew upgrade imsg
imsg rpc --help
imsg launch
openclaw channels status --probe
```

        وقتی راهنمای راه‌اندازی محلی نبود فرمان پیش‌فرض `imsg` را تشخیص دهد، می‌تواند نصب `steipete/tap/imsg` از طریق Homebrew را پیشنهاد کند. اگر `imsg` مدیریت‌شده با Homebrew را تشخیص دهد، می‌تواند نصب مجدد یا به‌روزرسانی آن را پیشنهاد کند. پوشش‌های سفارشی `cliPath` تغییر داده نمی‌شوند.

      </Step>

      <Step title="پیکربندی OpenClaw">

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "/usr/local/bin/imsg",
      dbPath: "/Users/user/Library/Messages/chat.db",
    },
  },
}
```

      </Step>

      <Step title="راه‌اندازی Gateway">

```bash
openclaw gateway
```

      </Step>

      <Step title="تأیید نخستین جفت‌سازی پیام خصوصی (dmPolicy پیش‌فرض)">

```bash
openclaw pairing list imessage
openclaw pairing approve imessage <CODE>
```

        درخواست‌های جفت‌سازی پس از 1 ساعت منقضی می‌شوند.
      </Step>
    </Steps>

  </Tab>

  <Tab title="Mac راه‌دور از طریق SSH">
    بیشتر راه‌اندازی‌ها به SSH نیاز ندارند. فقط زمانی از این توپولوژی استفاده کنید که Gateway نتواند روی Mac واردشده به Messages اجرا شود. OpenClaw فقط به یک `cliPath` سازگار با stdio نیاز دارد؛ بنابراین می‌توانید `cliPath` را به اسکریپت پوششی هدایت کنید که با SSH به Mac راه‌دور متصل می‌شود و `imsg` را اجرا می‌کند.
    `imsg` را روی همان Mac راه‌دور نصب و به‌روزرسانی کنید، نه روی میزبان Gateway:

```bash
ssh messages-mac 'brew install steipete/tap/imsg && brew update && brew upgrade imsg'
```

```bash
#!/usr/bin/env bash
exec ssh -T messages-mac imsg "$@"
```

    پیکربندی توصیه‌شده هنگام فعال‌بودن پیوست‌ها:

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "~/.openclaw/scripts/imsg-ssh",
      remoteHost: "user@gateway-host", // برای دریافت پیوست‌ها با SCP استفاده می‌شود
      includeAttachments: true,
      // اختیاری: ریشه‌های مجاز اضافی برای پیوست‌ها (با مقدار پیش‌فرض
      // /Users/*/Library/Messages/Attachments ادغام می‌شوند).
      attachmentRoots: ["/Users/*/Library/Messages/Attachments"],
      remoteAttachmentRoots: ["/Users/*/Library/Messages/Attachments"],
    },
  },
}
```

    اگر `remoteHost` تنظیم نشده باشد، OpenClaw تلاش می‌کند آن را با تجزیه اسکریپت پوشش SSH به‌طور خودکار تشخیص دهد.
    `remoteHost` باید `host` یا `user@host` باشد (بدون فاصله یا گزینه‌های SSH)؛ مقادیر ناامن نادیده گرفته می‌شوند.
    OpenClaw برای SCP از بررسی سخت‌گیرانه کلید میزبان استفاده می‌کند؛ بنابراین کلید میزبان رله باید از قبل در `~/.ssh/known_hosts` وجود داشته باشد.
    مسیرهای پیوست با ریشه‌های مجاز (`attachmentRoots` / `remoteAttachmentRoots`) اعتبارسنجی می‌شوند.

<Warning>
هر پوشش `cliPath` یا پراکسی SSH که جلوی `imsg` قرار می‌دهید باید برای JSON-RPC طولانی‌مدت مانند یک لوله شفاف stdio رفتار کند. OpenClaw در تمام طول عمر کانال، پیام‌های کوچک JSON-RPC قاب‌بندی‌شده با خط جدید را از طریق stdin/stdout پوشش مبادله می‌کند:

- هر قطعه/خط stdin را **به‌محض در دسترس بودن بایت‌ها** ارسال کنید — منتظر EOF نمانید.
- هر قطعه/خط stdout را بی‌درنگ در جهت معکوس ارسال کنید.
- خطوط جدید را حفظ کنید.
- از خواندن‌های مسدودکننده با اندازه ثابت (`read(4096)`، `cat | buffer`، `read` پیش‌فرض پوسته) که ممکن است قاب‌های کوچک را معطل کنند، اجتناب کنید.
- stderr را از جریان stdout مربوط به JSON-RPC جدا نگه دارید.

پوششی که stdin را تا پرشدن یک بلوک بزرگ بافر می‌کند، علائمی شبیه قطعی iMessage ایجاد خواهد کرد — `imsg rpc timeout (chats.list)` یا راه‌اندازی‌های مجدد مکرر کانال — حتی اگر خود `imsg rpc` سالم باشد. `ssh -T host imsg "$@"` (در بالا) ایمن است، زیرا آرگومان‌های `cliPath` مربوط به OpenClaw، مانند `rpc` و `--db`، را منتقل می‌کند. پایپ‌لاین‌هایی مانند `ssh host imsg | grep -v '^DEBUG'` ایمن نیستند — ابزارهای با بافر خطی همچنان ممکن است قاب‌ها را نگه دارند؛ اگر ناچار به فیلترکردن هستید، در همه مراحل از `stdbuf -oL -eL` استفاده کنید.
</Warning>

  </Tab>
</Tabs>

## الزامات و مجوزها (macOS)

- Messages باید روی Mac اجراکننده `imsg` وارد حساب شده باشد.
- دسترسی کامل به دیسک برای زمینه فرایندی که OpenClaw/`imsg` را اجرا می‌کند الزامی است (برای دسترسی به پایگاه داده Messages).
- مجوز Automation برای ارسال پیام از طریق Messages.app الزامی است.
- برای عملیات پیشرفته (واکنش / ویرایش / لغو ارسال / پاسخ رشته‌ای / جلوه‌ها / نظرسنجی‌ها / عملیات گروهی)، System Integrity Protection باید غیرفعال باشد — [فعال‌سازی API خصوصی imsg](#enabling-the-imsg-private-api) را ببینید. ارسال و دریافت پایه متن و رسانه بدون آن کار می‌کند.

<Tip>
مجوزها به‌ازای هر زمینه فرایند اعطا می‌شوند. اگر Gateway بدون رابط تعاملی اجرا می‌شود (LaunchAgent/SSH)، یک فرمان تعاملی یک‌باره را در همان زمینه اجرا کنید تا درخواست‌های مجوز فعال شوند:

```bash
imsg chats --limit 1
# یا
imsg send <handle> "آزمایش"
```

</Tip>

<Accordion title="ارسال از طریق پوشش SSH با AppleEvents -1743 ناموفق است">
  یک راه‌اندازی SSH راه‌دور ممکن است بتواند گفت‌وگوها را بخواند، `channels status --probe` را با موفقیت اجرا کند و پیام‌های ورودی را پردازش کند، اما ارسال‌های خروجی همچنان با خطای مجوز AppleEvents شکست بخورند:

```text
مجوز ارسال رویدادهای Apple به Messages وجود ندارد. (-1743)
```

پایگاه داده TCC کاربر واردشده به Mac یا System Settings > Privacy & Security > Automation را بررسی کنید. اگر ورودی Automation به‌جای فرایند `imsg` یا پوسته محلی برای `/usr/libexec/sshd-keygen-wrapper` ثبت شده باشد، ممکن است macOS یک کلید قابل‌استفاده Messages برای آن کلاینت سمت سرور SSH ارائه نکند:

```text
kTCCServiceAppleEvents | /usr/libexec/sshd-keygen-wrapper | auth_value=0 | com.apple.MobileSMS
```

در این وضعیت، تکرار `tccutil reset AppleEvents` یا اجرای مجدد `imsg send` از طریق همان پوشش SSH ممکن است همچنان شکست بخورد، زیرا زمینه فرایندی که به Automation برای Messages نیاز دارد پوشش SSH است، نه برنامه‌ای که رابط کاربری بتواند به آن مجوز دهد.

به‌جای آن، از یکی از زمینه‌های فرایندی پشتیبانی‌شده `imsg` استفاده کنید:

- Gateway، یا دست‌کم پل `imsg`، را در نشست محلی کاربر واردشده به Messages اجرا کنید.
- پس از اعطای دسترسی کامل به دیسک و Automation از همان نشست، Gateway را با یک LaunchAgent برای آن کاربر راه‌اندازی کنید.
- اگر توپولوژی SSH دوکاربره را حفظ می‌کنید، پیش از فعال‌سازی کانال بررسی کنید که ارسال واقعی خروجی با `imsg send` از طریق همان پوشش دقیقاً موفق می‌شود. اگر امکان اعطای Automation به آن وجود ندارد، به‌جای اتکا به پوشش SSH برای ارسال‌ها، راه‌اندازی تک‌کاربره `imsg` را پیکربندی کنید.

</Accordion>

## فعال‌سازی API خصوصی imsg

`imsg` با دو حالت عملیاتی عرضه می‌شود. برای OpenClaw، حالت API خصوصی راه‌اندازی توصیه‌شده است، زیرا عملیات بومی iMessage مورد انتظار کاربران را در اختیار کانال قرار می‌دهد. حالت پایه همچنان برای نصب‌های کم‌خطر، تأیید اولیه یا میزبان‌هایی که SIP در آن‌ها قابل غیرفعال‌سازی نیست مفید است.

- **حالت پایه** (پیش‌فرض، بدون نیاز به تغییر SIP): متن و رسانه خروجی از طریق `send`، پایش/تاریخچه ورودی و فهرست گفت‌وگوها. این همان چیزی است که با یک `brew install steipete/tap/imsg` تازه و مجوزهای استاندارد macOS در بالا به‌صورت آماده دریافت می‌کنید.
- **حالت API خصوصی**: `imsg` یک dylib کمکی را به `Messages.app` تزریق می‌کند تا توابع داخلی `IMCore` را فراخوانی کند. این کار `react`، `edit`، `unsend`، `reply` (رشته‌ای)، `sendWithEffect`، `poll` و `poll-vote` (نظرسنجی‌های بومی Messages)، `renameGroup`، `setGroupIcon`، `addParticipant`، `removeParticipant`، `leaveGroup`، به‌همراه نشانگرهای در حال تایپ و رسیدهای خواندن را فعال می‌کند.

سطح عملیات توصیه‌شده در این صفحه به حالت API خصوصی نیاز دارد. README مربوط به `imsg` این الزام را صریحاً بیان می‌کند:

> قابلیت‌های پیشرفته‌ای مانند `read`، `typing`، `launch`، ارسال غنی با پشتیبانی پل، تغییر پیام و مدیریت گفت‌وگو اختیاری هستند. آن‌ها به غیرفعال‌بودن SIP و تزریق یک dylib کمکی به `Messages.app` نیاز دارند. `imsg launch` هنگام فعال‌بودن SIP از تزریق خودداری می‌کند.

روش تزریق ابزار کمکی از dylib خود `imsg` برای دسترسی به APIهای خصوصی Messages استفاده می‌کند. در مسیر iMessage مربوط به OpenClaw هیچ سرور شخص ثالث یا زمان‌اجرای BlueBubbles وجود ندارد.

<Warning>
**غیرفعال‌کردن SIP یک مصالحه امنیتی واقعی است.** SIP یکی از محافظت‌های اصلی macOS در برابر اجرای کد سیستمی تغییریافته است؛ غیرفعال‌کردن آن در کل سیستم، سطح حمله و عوارض جانبی بیشتری ایجاد می‌کند. نکته مهم اینکه **غیرفعال‌کردن SIP در Macهای Apple Silicon، امکان نصب و اجرای برنامه‌های iOS روی Mac را نیز غیرفعال می‌کند**.

این کار را یک انتخاب عملیاتی آگاهانه در نظر بگیرید، به‌ویژه روی یک Mac شخصی اصلی. برای iMessage با کیفیت عملیاتی در OpenClaw، ترجیحاً از یک Mac اختصاصی یا کاربر ربات macOS استفاده کنید که در آن با فعال‌سازی پل راحت هستید. اگر مدل تهدید شما غیرفعال‌بودن SIP را در هیچ‌جا نمی‌پذیرد، iMessage همراه محصول به حالت پایه محدود می‌شود — فقط ارسال و دریافت متن و رسانه، بدون واکنش‌ها / ویرایش / لغو ارسال / جلوه‌ها / عملیات گروهی.
</Warning>

### راه‌اندازی

1. **`imsg` را نصب (یا ارتقا) کنید** روی Macی که Messages.app را اجرا می‌کند:

   ```bash
   brew install steipete/tap/imsg
   brew update && brew upgrade imsg
   imsg --version
   imsg status --json
   ```

   خروجی `imsg status --json`، `bridge_version`، `rpc_methods` و `selectors` به‌ازای هر متد را گزارش می‌کند تا پیش از شروع ببینید ساخت فعلی از چه قابلیت‌هایی پشتیبانی می‌کند.

2. **محافظت از یکپارچگی سیستم و (در نسخه‌های جدید macOS) اعتبارسنجی کتابخانه را غیرفعال کنید.** تزریق یک dylib کمکی غیر Apple به `Messages.app` امضاشده توسط Apple مستلزم خاموش‌بودن SIP **و** تسهیل اعتبارسنجی کتابخانه است. مرحله SIP در حالت Recovery به نسخه macOS بستگی دارد:
   - **macOS 10.13-10.15 (Sierra-Catalina):** اعتبارسنجی کتابخانه را از طریق Terminal غیرفعال کنید، سیستم را در Recovery Mode راه‌اندازی مجدد کنید، `csrutil disable` را اجرا کنید و دوباره راه‌اندازی کنید.
   - **macOS 11+ (Big Sur و نسخه‌های بعدی)، Intel:** وارد Recovery Mode (یا Internet Recovery) شوید، `csrutil disable` را اجرا کنید و دوباره راه‌اندازی کنید.
   - **macOS 11+، Apple Silicon:** برای ورود به Recovery از توالی راه‌اندازی با دکمه روشن/خاموش استفاده کنید؛ در نسخه‌های اخیر macOS، هنگام کلیک روی Continue کلید **Left Shift** را نگه دارید، سپس `csrutil disable` را اجرا کنید. راه‌اندازی‌های ماشین مجازی جریان جداگانه‌ای دارند، بنابراین ابتدا از VM یک snapshot بگیرید.

   **در macOS 11 و نسخه‌های بعدی، معمولاً `csrutil disable` به‌تنهایی کافی نیست.** Apple همچنان اعتبارسنجی کتابخانه را برای `Messages.app` به‌عنوان یک باینری پلتفرم اعمال می‌کند، بنابراین حتی با خاموش‌بودن SIP نیز یک ابزار کمکی با امضای adhoc رد می‌شود (`Library Validation failed: ... platform binary, but mapped file is not`). پس از غیرفعال‌کردن SIP، اعتبارسنجی کتابخانه را نیز غیرفعال و سیستم را دوباره راه‌اندازی کنید:

   ```bash
   sudo defaults write /Library/Preferences/com.apple.security.libraryvalidation.plist DisableLibraryValidation -bool true
   ```

   **macOS 26 (Tahoe)، تأییدشده روی 26.5.1:** خاموش‌بودن SIP **به‌همراه** فرمان `DisableLibraryValidation` بالا برای تزریق ابزار کمکی در نسخه‌های 26.0 تا 26.5.x کافی است. **هیچ boot-argsای لازم نیست.** فایل plist عامل تعیین‌کننده و رایج‌ترین مرحله فراموش‌شده هنگام شکست تزریق در Tahoe است:
   - **با plist:** `imsg launch` تزریق می‌شود و `imsg status` مقدار `advanced_features: true` را گزارش می‌کند.
   - **بدون plist (حتی با SIP خاموش):** `imsg launch` با `Failed to launch: Timeout waiting for Messages.app to initialize` شکست می‌خورد. AMFI ابزار کمکی adhoc را هنگام بارگذاری رد می‌کند، بنابراین bridge هرگز آماده نمی‌شود و راه‌اندازی به مهلت زمانی می‌رسد. این مهلت زمانی همان نشانه‌ای است که بیشتر افراد در Tahoe با آن مواجه می‌شوند؛ راه‌حل، plist بالا است، نه اقدام شدیدتری.

   اگر پس از ارتقای macOS، تزریق `imsg launch` یا `selectors` مشخصی شروع به بازگرداندن false کردند، معمولاً علت همین گیت است. پیش از آن‌که فرض کنید خود مرحله SIP شکست خورده است، وضعیت SIP و اعتبارسنجی کتابخانه را بررسی کنید. اگر این تنظیمات درست‌اند و bridge همچنان نمی‌تواند تزریق شود، `imsg status --json` را به‌همراه خروجی `imsg launch` جمع‌آوری کنید و به‌جای تضعیف کنترل‌های امنیتی بیشتر در سراسر سیستم، آن را به پروژه `imsg` گزارش دهید.

3. **ابزار کمکی را تزریق کنید.** با SIP غیرفعال و ورود انجام‌شده در Messages.app:

   ```bash
   imsg launch
   ```

   وقتی SIP همچنان فعال باشد، `imsg launch` از تزریق خودداری می‌کند؛ بنابراین این کار تأییدی نیز بر انجام مرحله 2 است.

4. **bridge را از OpenClaw تأیید کنید:**

   ```bash
   openclaw channels status --probe
   ```

   ورودی iMessage باید `works` را گزارش کند و `imsg status --json | jq '{rpc_methods, selectors}'` باید قابلیت‌های ارائه‌شده توسط build نسخه macOS شما را نشان دهد. ایجاد نظرسنجی به `selectors.pollPayloadMessage` نیاز دارد؛ رأی‌دادن هم به `selectors.pollVoteMessage` و هم به متد RPC مربوط به `poll.vote` نیاز دارد. Plugin مربوط به OpenClaw فقط کنش‌هایی را اعلام می‌کند که probe ذخیره‌شده در cache پشتیبانی می‌کند، درحالی‌که cache خالی خوش‌بینانه باقی می‌ماند و هنگام نخستین dispatch عمل probe را انجام می‌دهد.

اگر `openclaw channels status --probe` کانال را به‌صورت `works` گزارش می‌کند، اما کنش‌های مشخص هنگام dispatch خطای «iMessage `<action>` به bridge مربوط به API خصوصی imsg نیاز دارد» می‌دهند، دوباره `imsg launch` را اجرا کنید — ابزار کمکی ممکن است از دسترس خارج شود (راه‌اندازی مجدد Messages.app، به‌روزرسانی سیستم‌عامل و غیره) و وضعیت cacheشده `available: true` تا زمانی که probe بعدی آن را تازه کند، همچنان کنش‌ها را اعلام خواهد کرد.

### وقتی SIP فعال باقی می‌ماند

اگر غیرفعال‌کردن SIP برای مدل تهدید شما پذیرفتنی نیست:

- `imsg` به حالت پایه بازمی‌گردد — فقط متن + رسانه + دریافت.
- Plugin مربوط به OpenClaw همچنان ارسال متن/رسانه و پایش ورودی را اعلام می‌کند؛ اما `react`، `edit`، `unsend`، `reply`، `sendWithEffect` و عملیات گروه را از سطح کنش پنهان می‌کند (بر اساس گیت قابلیت هر متد).
- می‌توانید یک Mac جداگانه غیر Apple-Silicon (یا یک Mac اختصاصی برای ربات) را با SIP خاموش برای بار کاری iMessage اجرا کنید و هم‌زمان SIP را در دستگاه‌های اصلی خود فعال نگه دارید. بخش [کاربر اختصاصی ربات در macOS (هویت جداگانه iMessage)](#deployment-patterns) را در ادامه ببینید.

## کنترل دسترسی و مسیریابی

<Tabs>
  <Tab title="سیاست پیام مستقیم">
    `channels.imessage.dmPolicy` پیام‌های مستقیم را کنترل می‌کند:

    - `pairing` (پیش‌فرض)
    - `allowlist` (حداقل به یک ورودی `allowFrom` نیاز دارد)
    - `open` (لازم است `allowFrom` شامل `"*"` باشد)
    - `disabled`

    فیلد فهرست مجاز: `channels.imessage.allowFrom`.

    ورودی‌های فهرست مجاز باید فرستندگان را مشخص کنند: handleها یا گروه‌های ثابت دسترسی فرستنده (`accessGroup:<name>`). برای مقصدهای گفت‌وگو مانند `chat_id:*`، `chat_guid:*` یا `chat_identifier:*` از `channels.imessage.groupAllowFrom` استفاده کنید؛ برای کلیدهای عددی رجیستری `chat_id` از `channels.imessage.groups` استفاده کنید.

  </Tab>

  <Tab title="سیاست گروه + اشاره‌ها">
    `channels.imessage.groupPolicy` مدیریت گروه را کنترل می‌کند:

    - `allowlist` (پیش‌فرض)
    - `open`
    - `disabled`

    فهرست مجاز فرستندگان گروه: `channels.imessage.groupAllowFrom`.

    ورودی‌های `groupAllowFrom` می‌توانند به گروه‌های ثابت دسترسی فرستنده (`accessGroup:<name>`) نیز ارجاع دهند.

    بازگشت Runtime: اگر `groupAllowFrom` تنظیم نشده باشد، بررسی فرستندگان گروه iMessage از `allowFrom` استفاده می‌کند؛ وقتی پذیرش پیام مستقیم و گروه باید متفاوت باشد، `groupAllowFrom` را تنظیم کنید. مقدار صراحتاً خالی `groupAllowFrom: []` بازگشت انجام نمی‌دهد — همه فرستندگان گروه را تحت `allowlist` مسدود می‌کند.
    نکته Runtime: اگر `channels.imessage` کاملاً وجود نداشته باشد، Runtime به `groupPolicy="allowlist"` بازمی‌گردد و یک هشدار ثبت می‌کند (حتی اگر `channels.defaults.groupPolicy` تنظیم شده باشد).

    <Warning>
    مسیریابی گروه تحت `groupPolicy: "allowlist"` **دو** گیت را پشت‌سرهم اجرا می‌کند:

    1. **فهرست مجاز فرستندگان** (`channels.imessage.groupAllowFrom`) — handle، `accessGroup:<name>`، `chat_guid`، `chat_identifier` یا `chat_id`. فهرست مؤثر خالی (بدون `groupAllowFrom` و بدون بازگشت `allowFrom`) همه فرستندگان گروه را مسدود می‌کند.
    2. **رجیستری گروه** (`channels.imessage.groups`) — پس از آن‌که map دارای ورودی شود اعمال می‌شود: گفت‌وگو باید با یک ورودی صریح برای هر `chat_id` یا wildcard مربوط به `groups: { "*": { ... } }` مطابقت داشته باشد. وقتی `groups` خالی یا مفقود باشد، فقط فهرست مجاز فرستندگان درباره پذیرش تصمیم می‌گیرد.

    اگر هیچ فهرست مجاز مؤثری برای فرستندگان گروه پیکربندی نشده باشد، همه پیام‌های گروه پیش از گیت رجیستری کنار گذاشته می‌شوند. هر گیت در سطح پیش‌فرض گزارش‌گیری، سیگنال سطح `warn` مخصوص خود را دارد و هرکدام راه‌حل متفاوتی را نام می‌برند:

    - یک‌بار برای هر حساب هنگام راه‌اندازی، وقتی فهرست مجاز مؤثر فرستندگان گروه خالی است: `imessage: groupPolicy="allowlist" for account "<id>" but no group sender allowlist is configured ...` — با تنظیم `channels.imessage.groupAllowFrom` (یا `allowFrom`) برطرف می‌شود؛ صرفاً افزودن ورودی‌های `groups` باعث می‌شود گیت 1 همچنان همه فرستندگان را مسدود کند.
    - یک‌بار برای هر `chat_id` هنگام Runtime، وقتی فرستنده از گیت 1 عبور کرده اما گفت‌وگو در رجیستری پرشده `groups` وجود ندارد: `imessage: dropping group message from chat_id=<id> ...` — با افزودن آن `chat_id` (یا `"*"`) زیر `channels.imessage.groups` برطرف می‌شود.

    پیام‌های مستقیم تحت‌تأثیر نیستند — آن‌ها از مسیر کد متفاوتی عبور می‌کنند.

    پیکربندی پیشنهادی برای جریان گروه تحت `groupPolicy: "allowlist"`:

    ```json5
    {
      channels: {
        imessage: {
          groupPolicy: "allowlist",
          groupAllowFrom: ["+15555550123"],
          groups: { "*": { "requireMention": true } },
        },
      },
    }
    ```

    `groupAllowFrom` به‌تنهایی این فرستندگان را در هر گروهی می‌پذیرد؛ برای محدودکردن گفت‌وگوهای مجاز (و تنظیم گزینه‌های هر گفت‌وگو مانند `requireMention`) بلوک `groups` را اضافه کنید.
    </Warning>

    گیت اشاره برای گروه‌ها:

    - iMessage هیچ فراداده بومی برای اشاره ندارد
    - تشخیص اشاره از الگوهای regex استفاده می‌کند (`agents.entries.*.groupChat.mentionPatterns`، با بازگشت به `messages.groupChat.mentionPatterns`)
    - بدون الگوهای پیکربندی‌شده، گیت اشاره قابل اعمال نیست
    - فرمان‌های کنترلی از فرستندگان مجاز، گیت اشاره را دور می‌زنند

    `systemPrompt` برای هر گروه:

    هر ورودی زیر `channels.imessage.groups.*` یک رشته اختیاری `systemPrompt` می‌پذیرد که در هر نوبتی که پیامی از آن گروه را مدیریت می‌کند، به اعلان سیستمی عامل تزریق می‌شود. تفکیک آن مشابه `channels.whatsapp.groups` است:

    1. **اعلان سیستمی مخصوص گروه** (`groups["<chat_id>"].systemPrompt`): وقتی ورودی گروه مشخص در map وجود داشته باشد **و** کلید `systemPrompt` آن تعریف شده باشد، استفاده می‌شود. اگر `systemPrompt` رشته‌ای خالی (`""`) باشد، wildcard سرکوب می‌شود و هیچ اعلان سیستمی برای آن گروه اعمال نمی‌شود.
    2. **اعلان سیستمی wildcard گروه** (`groups["*"].systemPrompt`): وقتی ورودی گروه مشخص کاملاً در map وجود نداشته باشد، یا وجود داشته باشد اما هیچ کلید `systemPrompt` تعریف نکند، استفاده می‌شود.

    ```json5
    {
      channels: {
        imessage: {
          groupPolicy: "allowlist",
          groupAllowFrom: ["+15555550123"],
          groups: {
            "*": { systemPrompt: "از املای بریتانیایی استفاده کنید." },
            "8421": {
              requireMention: true,
              systemPrompt: "این گفت‌وگوی نوبت آماده‌باش است. پاسخ‌ها را به کمتر از 3 جمله محدود کنید.",
            },
            "9907": {
              // سرکوب صریح: wildcard «از املای بریتانیایی استفاده کنید.» اینجا اعمال نمی‌شود
              systemPrompt: "",
            },
          },
        },
      },
    }
    ```

    اعلان‌های هر گروه فقط برای پیام‌های گروه اعمال می‌شوند — پیام‌های مستقیم تحت‌تأثیر نیستند.

  </Tab>

  <Tab title="نشست‌ها و پاسخ‌های قطعی">
    - پیام‌های مستقیم از مسیریابی مستقیم و گروه‌ها از مسیریابی گروه استفاده می‌کنند.
    - با مقدار پیش‌فرض `session.dmScope=main`، پیام‌های مستقیم iMessage در نشست اصلی عامل ادغام می‌شوند.
    - نشست‌های گروه ایزوله هستند (`agent:<agentId>:imessage:group:<chat_id>`).
    - پاسخ‌ها با استفاده از فراداده کانال/مقصد مبدأ، دوباره به iMessage مسیریابی می‌شوند.

    رفتار رشته‌های شبیه گروه:

    برخی رشته‌های iMessage با چند مشارکت‌کننده ممکن است با `is_group=false` دریافت شوند.
    اگر آن `chat_id` صراحتاً زیر `channels.imessage.groups` پیکربندی شده باشد، OpenClaw آن را ترافیک گروه در نظر می‌گیرد (گیت گروه + ایزوله‌سازی نشست گروه).

  </Tab>
</Tabs>

## اتصال گفت‌وگوهای ACP

گفت‌وگوهای iMessage را می‌توان به نشست‌های ACP متصل کرد.

جریان سریع اپراتور:

- `/acp spawn codex --bind here` را داخل پیام مستقیم یا گفت‌وگوی گروه مجاز اجرا کنید.
- پیام‌های آینده در همان گفت‌وگوی iMessage به نشست ACP ایجادشده مسیریابی می‌شوند.
- `/new` و `/reset` همان نشست ACP متصل را درجا بازنشانی می‌کنند.
- `/acp close` نشست ACP را می‌بندد و اتصال را حذف می‌کند.

اتصال‌های دائمی پیکربندی‌شده از ورودی‌های سطح‌بالای `bindings[]` با `type: "acp"` و `match.channel: "imessage"` استفاده می‌کنند.

`match.peer.id` می‌تواند از موارد زیر استفاده کند:

- handle نرمال‌شده پیام مستقیم مانند `+15555550123` یا `user@example.com`
- `chat_id:<id>` (برای اتصال‌های پایدار گروه توصیه می‌شود)
- `chat_guid:<guid>`
- `chat_identifier:<identifier>`

مثال:

```json5
{
  agents: {
    list: [
      {
        id: "codex",
        runtime: {
          type: "acp",
          acp: { agent: "codex", backend: "acpx", mode: "persistent" },
        },
      },
    ],
  },
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "imessage",
        accountId: "default",
        peer: { kind: "group", id: "chat_id:123" },
      },
      acp: { label: "codex-group" },
    },
  ],
}
```

برای رفتار مشترک اتصال ACP، بخش [عامل‌های ACP](/fa/tools/acp-agents) را ببینید.

## الگوهای استقرار

<AccordionGroup>
  <Accordion title="کاربر اختصاصی ربات در macOS (هویت جداگانه iMessage)">
    از یک Apple ID و کاربر macOS اختصاصی استفاده کنید تا ترافیک ربات از نمایه شخصی Messages شما جدا باشد.

    جریان معمول:

    1. یک کاربر اختصاصی macOS ایجاد کنید/وارد آن شوید.
    2. در آن کاربر، با Apple ID ربات وارد Messages شوید.
    3. `imsg` را در آن کاربر نصب کنید.
    4. یک پوشش SSH ایجاد کنید تا OpenClaw بتواند `imsg` را در زمینهٔ آن کاربر اجرا کند.
    5. `channels.imessage.accounts.<id>.cliPath` و `.dbPath` را به پروفایل آن کاربر ارجاع دهید.

    نخستین اجرا ممکن است در نشست کاربر ربات به تأییدهای رابط گرافیکی (Automation + Full Disk Access) نیاز داشته باشد.

  </Accordion>

  <Accordion title="Mac راه‌دور از طریق Tailscale (نمونه)">
    توپولوژی متداول:

    - gateway روی Linux/VM اجرا می‌شود
    - iMessage و `imsg` روی یک Mac در tailnet شما اجرا می‌شوند
    - پوشش `cliPath` برای اجرای `imsg` از SSH استفاده می‌کند
    - `remoteHost` دریافت پیوست‌ها با SCP را فعال می‌کند

    نمونه:

    ```json5
    {
      channels: {
        imessage: {
          enabled: true,
          cliPath: "~/.openclaw/scripts/imsg-ssh",
          remoteHost: "bot@mac-mini.tailnet-1234.ts.net",
          includeAttachments: true,
          dbPath: "/Users/bot/Library/Messages/chat.db",
        },
      },
    }
    ```

    ```bash
    #!/usr/bin/env bash
    exec ssh -T bot@mac-mini.tailnet-1234.ts.net imsg "$@"
    ```

    از کلیدهای SSH استفاده کنید تا SSH و SCP هر دو غیرتعاملی باشند.
    ابتدا مطمئن شوید کلید میزبان مورد اعتماد است (برای مثال `ssh bot@mac-mini.tailnet-1234.ts.net`) تا `known_hosts` پر شود.

  </Accordion>

  <Accordion title="الگوی چندحسابی">
    iMessage از پیکربندی جداگانهٔ هر حساب در `channels.imessage.accounts` پشتیبانی می‌کند.

    هر حساب می‌تواند فیلدهایی مانند `cliPath`، `dbPath`، `allowFrom`، `groupPolicy`، `mediaMaxMb`، تنظیمات تاریخچه و فهرست‌های مجاز ریشهٔ پیوست‌ها را بازنویسی کند.

  </Accordion>

  <Accordion title="تاریخچهٔ پیام مستقیم">
    برای مقداردهی اولیهٔ نشست‌های جدید پیام مستقیم با تاریخچهٔ رمزگشایی‌شدهٔ اخیر `imsg` مربوط به آن مکالمه، `channels.imessage.dmHistoryLimit` را تنظیم کنید. برای بازنویسی‌های مختص هر فرستنده، از `channels.imessage.dms["<sender>"].historyLimit` استفاده کنید؛ از جمله `0` برای غیرفعال‌کردن تاریخچهٔ یک فرستنده.

    تاریخچهٔ پیام مستقیم iMessage هنگام نیاز از `imsg` واکشی می‌شود. تنظیم‌نکردن `dmHistoryLimit` مقداردهی اولیهٔ سراسری تاریخچهٔ پیام مستقیم را غیرفعال می‌کند، اما مقدار مثبت `channels.imessage.dms["<sender>"].historyLimit` برای هر فرستنده همچنان مقداردهی اولیه را برای همان فرستنده فعال می‌کند.

  </Accordion>
</AccordionGroup>

## رسانه، قطعه‌بندی و مقصدهای تحویل

<AccordionGroup>
  <Accordion title="پیوست‌ها و رسانه">
    - دریافت پیوست ورودی **به‌طور پیش‌فرض غیرفعال است** — برای ارسال عکس‌ها، یادداشت‌های صوتی، ویدئو و دیگر پیوست‌ها به عامل، `channels.imessage.includeAttachments: true` را تنظیم کنید. در صورت غیرفعال‌بودن آن، iMessageهایی که فقط پیوست دارند پیش از رسیدن به عامل حذف می‌شوند و ممکن است اصلاً هیچ خط گزارش `Inbound message` تولید نکنند.
    - وقتی `remoteHost` تنظیم شده باشد، مسیرهای پیوست راه‌دور را می‌توان از طریق SCP واکشی کرد
    - مسیرهای پیوست باید با ریشه‌های مجاز مطابقت داشته باشند:
      - `channels.imessage.attachmentRoots` (محلی)
      - `channels.imessage.remoteAttachmentRoots` (حالت SCP راه‌دور)
      - ریشه‌های پیکربندی‌شده الگوی ریشهٔ پیش‌فرض `/Users/*/Library/Messages/Attachments` را گسترش می‌دهند (ادغام می‌شوند، جایگزین نمی‌شوند)
    - SCP از بررسی سخت‌گیرانهٔ کلید میزبان استفاده می‌کند (`StrictHostKeyChecking=yes`)
    - اندازهٔ رسانهٔ خروجی از `channels.imessage.mediaMaxMb` استفاده می‌کند (پیش‌فرض 16 MB)

  </Accordion>

  <Accordion title="متن خروجی و قطعه‌بندی">
    - محدودیت قطعهٔ متن: `channels.imessage.textChunkLimit` (پیش‌فرض 4000)
    - حالت قطعه‌بندی: `channels.imessage.streaming.chunkMode`
      - `length` (پیش‌فرض)
      - `newline` (تقسیم ابتدا بر اساس بندها)
    - قالب‌بندی ضخیم/مورب/زیرخط‌دار/خط‌خوردهٔ Markdown خروجی به متن قالب‌بندی‌شدهٔ بومی تبدیل می‌شود (گیرندگان macOS 15+ قالب‌بندی را نمایش می‌دهند؛ گیرندگان قدیمی‌تر متن ساده را بدون نشانه‌ها می‌بینند)؛ جدول‌های Markdown مطابق حالت جدول Markdown کانال تبدیل می‌شوند
    - `channels.imessage.sendTransport` (`auto` پیش‌فرض، `bridge`، `applescript`) چگونگی تحویل ارسال‌ها توسط `imsg` را تعیین می‌کند

  </Accordion>

  <Accordion title="قالب‌های نشانی‌دهی">
    مقصدهای صریح ترجیحی:

    - `chat_id:123` (برای مسیریابی پایدار توصیه می‌شود)
    - `chat_guid:...`
    - `chat_identifier:...`

    مقصدهای مبتنی بر شناسه نیز پشتیبانی می‌شوند:

    - `imessage:+1555...`
    - `sms:+1555...`
    - `user@example.com`

    ```bash
    imsg chats --limit 20
    ```

  </Accordion>
</AccordionGroup>

## کنش‌های API خصوصی

وقتی `imsg launch` در حال اجرا است و `openclaw channels status --probe` مقدار `privateApi.available: true` را گزارش می‌کند، ابزار پیام می‌تواند علاوه بر ارسال عادی متن، از کنش‌های بومی iMessage استفاده کند.

همهٔ کنش‌ها به‌طور پیش‌فرض فعال‌اند؛ برای غیرفعال‌کردن هر کنش، از `channels.imessage.actions` استفاده کنید:

```json5
{
  channels: {
    imessage: {
      actions: {
        reactions: true,
        edit: true,
        unsend: true,
        reply: true,
        sendWithEffect: true,
        sendAttachment: true,
        renameGroup: true,
        setGroupIcon: true,
        addParticipant: true,
        removeParticipant: true,
        leaveGroup: true,
        polls: true,
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="کنش‌های موجود">
    - **react**: افزودن/حذف tapbackهای iMessage (`messageId`، `emoji`، `remove`). tapbackهای پشتیبانی‌شده به عشق، پسندیدن، نپسندیدن، خنده، تأکید و پرسش نگاشت می‌شوند. حذف بدون ایموجی، هر tapback تنظیم‌شده‌ای را پاک می‌کند.
    - **reply**: ارسال پاسخ رشته‌ای به یک پیام موجود (`messageId`، `text` یا `message`، به‌علاوهٔ `chatGuid`، `chatId`، `chatIdentifier` یا `to`). پاسخ همراه پیوست علاوه بر این به ساختی از `imsg` نیاز دارد که `send-rich` آن از `--file` پشتیبانی کند.
    - **sendWithEffect**: ارسال متن با یک جلوهٔ iMessage (`text` یا `message`، `effect` یا `effectId`). نام‌های کوتاه: slam، loud، gentle، invisibleink، confetti، lasers، fireworks، balloon، heart، echo، happybirthday، shootingstar، sparkles، spotlight.
    - **edit**: ویرایش پیام ارسال‌شده در نسخه‌های پشتیبانی‌شدهٔ macOS/API خصوصی (`messageId`، `text` یا `newText`). فقط پیام‌هایی که خود Gateway ارسال کرده است قابل ویرایش‌اند.
    - **unsend**: پس‌گرفتن پیام ارسال‌شده در نسخه‌های پشتیبانی‌شدهٔ macOS/API خصوصی (`messageId`). فقط پیام‌هایی که خود Gateway ارسال کرده است قابل پس‌گرفتن‌اند.
    - **upload-file**: ارسال رسانه/فایل‌ها (`buffer` به‌صورت base64 یا یک `media`/`path`/`filePath` آماده‌شده، `filename`، و `asVoice` اختیاری). نام مستعار قدیمی: `sendAttachment`.
    - **renameGroup**، **setGroupIcon**، **addParticipant**، **removeParticipant**، **leaveGroup**: مدیریت گفت‌وگوهای گروهی هنگامی که مقصد فعلی یک مکالمهٔ گروهی است. این کنش‌ها هویت Messages میزبان را تغییر می‌دهند، بنابراین به یک فرستندهٔ مالک یا یک کلاینت Gateway با `operator.admin` نیاز دارند.
    - **poll**: ایجاد یک نظرسنجی بومی Apple Messages (`pollQuestion`، تکرار `pollOption` از 2 تا 12 بار، به‌علاوهٔ `chatGuid`، `chatId`، `chatIdentifier` یا `to`). گیرندگان دارای iOS/iPadOS/macOS 26+ آن را به‌صورت بومی می‌بینند و رأی می‌دهند؛ نسخه‌های قدیمی‌تر سیستم‌عامل متن جایگزین "یک نظرسنجی ارسال شد" را دریافت می‌کنند. به `selectors.pollPayloadMessage` نیاز دارد.
    - **poll-vote**: رأی‌دادن در یک نظرسنجی موجود (`pollId` یا `messageId`، به‌علاوهٔ دقیقاً یکی از `pollOptionIndex`، `pollOptionId` یا `pollOptionText`). به `selectors.pollVoteMessage` و متد RPC به نام `poll.vote` نیاز دارد.

    نظرسنجی‌های ورودی پذیرفته‌شده برای عامل همراه با پرسش، برچسب‌های شماره‌دار گزینه‌ها، تعداد رأی‌ها و شناسهٔ پیام نظرسنجی موردنیاز `poll-vote` نمایش داده می‌شوند.

  </Accordion>

  <Accordion title="شناسه‌های پیام">
    زمینهٔ iMessage ورودی، در صورت وجود، هم مقادیر کوتاه `MessageSid` و هم GUIDهای کامل پیام (`MessageSidFull`) را شامل می‌شود. شناسه‌های کوتاه به حافظهٔ نهان پاسخ اخیرِ مبتنی بر SQLite محدودند و پیش از استفاده در برابر گفت‌وگوی فعلی بررسی می‌شوند. اگر یک شناسهٔ کوتاه منقضی شد، هنگام هدف‌گیری مکالمه‌ای که آن را ارائه کرده است، با `MessageSidFull` آن دوباره تلاش کنید. شناسه‌های کامل مقیدبودن به مکالمه یا حساب را دور نمی‌زنند؛ بنابراین شناسه‌ای از گفت‌وگویی دیگر را با شناسه‌ای از مقصد فعلی جایگزین کنید. فراخوانی‌های واگذارشدهٔ راه‌دور ممکن است شناسه‌های کامل قدیمی را هنگامی که شواهد مکالمهٔ فعلی در دسترس نیست رد کنند.

  </Accordion>

  <Accordion title="تشخیص قابلیت">
    OpenClaw کنش‌های API خصوصی را فقط زمانی پنهان می‌کند که وضعیت کاوش ذخیره‌شده در حافظهٔ نهان نشان دهد پل در دسترس نیست. اگر وضعیت ناشناخته باشد، کنش‌ها قابل‌مشاهده می‌مانند و هنگام ارسال، کاوش‌ها را به‌صورت تنبل اجرا می‌کنند تا نخستین کنش بتواند پس از `imsg launch` بدون تازه‌سازی دستی جداگانهٔ وضعیت موفق شود.

  </Accordion>

  <Accordion title="رسید خواندن و نشانگر تایپ">
    وقتی پل API خصوصی فعال است، گفت‌وگوهای ورودی پذیرفته‌شده به‌عنوان خوانده‌شده علامت‌گذاری می‌شوند و گفت‌وگوهای مستقیم به‌محض پذیرش نوبت، درحالی‌که عامل زمینه را آماده و پاسخ را تولید می‌کند، حباب تایپ را نشان می‌دهند. علامت‌گذاری خوانده‌شدن را با این تنظیم غیرفعال کنید:

    ```json5
    {
      channels: {
        imessage: {
          sendReadReceipts: false,
        },
      },
    }
    ```

    ساخت‌های قدیمی‌تر `imsg` که پیش از فهرست قابلیت‌های هر متد ساخته شده‌اند، تایپ/خواندن را بی‌سروصدا غیرفعال می‌کنند؛ OpenClaw در هر راه‌اندازی مجدد یک هشدار یک‌باره ثبت می‌کند تا علت نبود رسید مشخص باشد.

  </Accordion>

  <Accordion title="tapbackهای ورودی">
    OpenClaw در tapbackهای iMessage مشترک می‌شود و واکنش‌های پذیرفته‌شده را به‌جای متن عادی پیام به‌صورت رویدادهای سیستمی مسیریابی می‌کند؛ بنابراین tapback کاربر یک حلقهٔ پاسخ عادی را فعال نمی‌کند.

    حالت اعلان با `channels.imessage.reactionNotifications` کنترل می‌شود:

    - `"own"` (پیش‌فرض): فقط وقتی کاربران به پیام‌های نوشته‌شده توسط ربات واکنش نشان می‌دهند اعلان کنید.
    - `"all"`: برای همهٔ tapbackهای ورودی از فرستندگان مجاز اعلان کنید.
    - `"off"`: tapbackهای ورودی را نادیده بگیرید.

    بازنویسی‌های مختص هر حساب از `channels.imessage.accounts.<id>.reactionNotifications` استفاده می‌کنند.

  </Accordion>

  <Accordion title="واکنش‌های تأیید (👍 / 👎)">
    وقتی `approvals.exec.enabled` یا `approvals.plugin.enabled` مقدار true دارد و درخواست به iMessage مسیریابی می‌شود، Gateway یک درخواست تأیید را به‌صورت بومی تحویل می‌دهد و برای تعیین تکلیف آن یک tapback را می‌پذیرد:

    - `👍` (tapback پسندیدن) → `allow-once`
    - `👎` (tapback نپسندیدن) → `deny`
    - `allow-always` به‌عنوان راه‌حل جایگزین دستی باقی می‌ماند: `/approve <id> allow-always` را به‌صورت یک پاسخ عادی ارسال کنید.

    پردازش واکنش مستلزم آن است که شناسهٔ کاربر واکنش‌دهنده صراحتاً در میان تأییدکنندگان باشد. فهرست تأییدکنندگان از `channels.imessage.allowFrom` (یا `channels.imessage.accounts.<id>.allowFrom`) خوانده می‌شود؛ شمارهٔ تلفن کاربر را با قالب E.164 یا ایمیل Apple ID او اضافه کنید (مقصدهای گفت‌وگو مانند `chat_id:*` ورودی معتبر تأییدکننده نیستند). ورودی عام `"*"` پذیرفته می‌شود، اما به هر فرستنده‌ای اجازهٔ تأیید می‌دهد؛ فهرست خالی تأییدکنندگان میان‌بر واکنش را کاملاً غیرفعال می‌کند. میان‌بر واکنش عمداً `reactionNotifications`، `dmPolicy` و `groupAllowFrom` را دور می‌زند، زیرا فهرست مجاز صریح تأییدکنندگان تنها محدودیتی است که برای تعیین تکلیف تأیید اهمیت دارد.

    مجوزدهی فرمان متنی `/approve` از همان فهرست پیروی می‌کند: وقتی `channels.imessage.allowFrom` خالی نیست، `/approve <id> <decision>` در برابر همان فهرست تأییدکنندگان مجاز می‌شود (نه فهرست مجاز گسترده‌تر پیام مستقیم) و فرستندگانی که در فهرست مجاز پیام مستقیم اجازه دارند اما در `allowFrom` نیستند، یک رد صریح دریافت می‌کنند. وقتی `allowFrom` خالی است، راه‌حل جایگزین همان گفت‌وگو برقرار می‌ماند و `/approve` هر کسی را که فهرست مجاز پیام مستقیم اجازه می‌دهد مجاز می‌کند. هر اپراتوری را که باید تأیید کند — از طریق `/approve` یا واکنش‌ها — به `allowFrom` اضافه کنید.

    یادداشت‌های اپراتور:
    - اتصال واکنش هم در حافظه و هم در ذخیره‌ساز کلیددار پایدار Gateway نگه‌داری می‌شود (TTL با زمان انقضای تأیید مطابقت دارد) و Gateway همچنین درخواست‌های در انتظار را برای tapbackها بررسی می‌کند؛ بنابراین tapbackی که اندکی پس از راه‌اندازی مجدد Gateway برسد، همچنان تأیید را نهایی می‌کند.
    - tapback متعلق به خود اپراتور در `is_from_me=true` (برای مثال از یک دستگاه Apple جفت‌شده) زمانی تأیید را نهایی می‌کند که آن شناسه به‌صراحت تأییدکننده باشد.
    - درخواست‌های تأیید فقط زمانی به گفت‌وگوی گروهی هدایت می‌شوند که تأییدکنندگان صریح پیکربندی شده باشند؛ در غیر این صورت، هر عضو گروه می‌تواند تأیید کند.
    - tapbackهای متنی قدیمی (`Liked "…"` متن ساده از کلاینت‌های بسیار قدیمی Apple) نمی‌توانند تأییدها را نهایی کنند، زیرا GUID پیام ندارند؛ نهایی‌سازی واکنش به فرادادهٔ ساختاریافتهٔ tapback نیاز دارد که کلاینت‌های فعلی macOS / iOS تولید می‌کنند.

  </Accordion>

  <Accordion title="واکنش‌های پرسش (1️⃣ / 2️⃣ / 3️⃣ / 4️⃣)">
    برای یک درخواست `ask_user` با یک پرسش غیرمحرمانه و تک‌انتخابی و یک تا چهار گزینه، OpenClaw گزینه‌های ایموجی شماره‌دار اضافه می‌کند. برای پاسخ‌دادن، با شمارهٔ متناظر به درخواست تحویل‌شده واکنش نشان دهید. واکنش باید GUID پایدار پیام نوشته‌شده توسط ربات را داشته باشد؛ سپس OpenClaw از طریق Gateway شماره را به گزینهٔ متعارف نگاشت می‌کند. لمس‌های منقضی یا تکراری نادیده گرفته می‌شوند.

    درخواست‌های چندپرسشی، چندانتخابی و متن آزاد همچنان فقط با پاسخ متنی کار می‌کنند. واکنش‌های پرسش از قواعد معمول پذیرش پیام خصوصی/گروهی iMessage پیروی می‌کنند. این واکنش‌ها حتی زمانی که `reactionNotifications` عمومی روی `"off"` تنظیم شده باشد شناسایی می‌شوند، بدون آنکه واکنش‌های نامرتبط به رویدادهای عامل تبدیل شوند.

  </Accordion>
</AccordionGroup>

## نوشتن پیکربندی

iMessage به‌طور پیش‌فرض اجازه می‌دهد کانال تغییرات پیکربندی را آغاز کند (برای `/config set|unset` هنگامی که `commands.config: true`).

غیرفعال‌سازی:

```json5
{
  channels: {
    imessage: {
      configWrites: false,
    },
  },
}
```

<a id="coalescing-split-send-dms-command--url-in-one-composition"></a>

## ادغام پیام‌های خصوصی چندبخشی (دستور + URL در یک ترکیب)

Apple می‌تواند یک دستور و پیش‌نمایش URL آن را به‌صورت ردیف‌های فیزیکی جداگانهٔ `chat.db` ذخیره کند. نسخهٔ `imsg` 0.13.1 و جدیدتر، پیش از آنکه پایش، تاریخچه یا جست‌وجو پیام را برگرداند، این ردیف‌ها را ادغام می‌کند؛ بنابراین OpenClaw بدون افزودن تأخیر مخصوص کانال به پیام خصوصی، یک پیام ورودی منطقی دریافت می‌کند.

هیچ تنظیمی برای ادغام iMessage لازم نیست. کلید بازنشستهٔ `channels.imessage.coalesceSameSenderDms` توسط `openclaw doctor --fix` حذف می‌شود. debounce عمومی `messages.inbound` زمانی همچنان در دسترس است که عمداً بخواهید پیام‌های متنی سریع را در سراسر یک کانال دسته‌بندی کنید.

اگر ارسال‌های شامل دستور و URL به‌صورت نوبت‌های جداگانهٔ عامل می‌رسند، `imsg` را در Mac مربوط به Messages به‌روزرسانی کنید:

```bash
brew update && brew upgrade imsg
```

## بازیابی ورودی پس از راه‌اندازی مجدد پل یا Gateway

iMessage پیام‌هایی را که هنگام ازکارافتادگی Gateway از دست رفته‌اند بازیابی می‌کند و هم‌زمان «انباشت انفجاری» قدیمی را که Apple ممکن است پس از بازیابی Push تخلیه کند، سرکوب می‌کند. رفتار پیش‌فرض همیشه فعال است و بر ورودی پایدار همراه با محدودیت سنی متکی است.

- **محافظت پایدار در برابر بازپخش.** پیش از جلو بردن مکان‌نمای بازیابی، OpenClaw هر ردیف خام را با GUID مربوط به Apple به‌عنوان شناسهٔ رویداد، در صف ورودی SQLite مشترک ثبت می‌کند. یک ردیف تکمیل‌شده حدود 4 ساعت، با سقف 10,000 ورودی، یک نشان حذف باقی می‌گذارد؛ بنابراین بازپخشی با همان GUID حتی پس از راه‌اندازی مجدد کنار گذاشته می‌شود. یک ردیف در انتظار تا زمانی که فرایند ارسال آن را بپذیرد، قابل بازیابی باقی می‌ماند.
- **بازیابی زمان ازکارافتادگی.** هنگام راه‌اندازی، پایشگر آخرین rowid ردیف `chat.db` پذیرفته‌شده به‌صورت پایدار را به خاطر می‌سپارد (یک مکان‌نمای پایدار برای هر حساب) و آن را به‌صورت `since_rowid` به `imsg watch.subscribe` می‌دهد تا imsg ردیف‌هایی را که هنوز ثبت نشده‌اند بازپخش کند و سپس رویدادهای زنده را دنبال کند. ردیف‌های ثبت‌شده پیش از خرابی از SQLite ادامه می‌یابند. بازپخش به جدیدترین 500 ردیف و پیام‌هایی با حداکثر سن حدود 2 ساعت محدود است و نشان‌های حذف GUID هر مورد از قبل پردازش‌شده را کنار می‌گذارند.
- **محدودیت سنی انباشت قدیمی.** ردیف‌های بالاتر از مرز راه‌اندازی واقعاً زنده‌اند؛ ردیفی که تاریخ ارسالش بیش از حدود 15 دقیقه از زمان رسیدنش قدیمی‌تر باشد، انباشت حاصل از تخلیهٔ Push است و سرکوب می‌شود. ردیف‌های بازپخش‌شده (در مرز یا پایین‌تر از آن) به‌جای آن از بازهٔ بازیابی گسترده‌تر استفاده می‌کنند؛ بنابراین پیام تازه ازدست‌رفته تحویل می‌شود، اما تاریخچهٔ بسیار قدیمی خیر.

بازیابی هم در راه‌اندازی‌های محلی و هم راه‌دور `cliPath` کار می‌کند، زیرا بازپخش `since_rowid` از همان اتصال RPC مربوط به `imsg` عبور می‌کند. تفاوت در بازه است: هنگامی که Gateway بتواند `chat.db` را بخواند (محلی)، مرز rowid راه‌اندازی را تثبیت می‌کند، گسترهٔ بازپخش را محدود می‌کند و پیام‌های ازدست‌رفته با قدمت حداکثر چند ساعت را تحویل می‌دهد. از طریق یک `cliPath` راه‌دور مبتنی بر SSH نمی‌تواند پایگاه داده را بخواند؛ بنابراین بازپخش بدون سقف است و هر ردیف از محدودیت سنی زنده استفاده می‌کند — همچنان پیام‌های تازه ازدست‌رفته را بازیابی و انباشت قدیمی را سرکوب می‌کند، اما با بازهٔ زندهٔ محدودتر. برای استفاده از بازهٔ بازیابی گسترده‌تر، Gateway را روی Mac مربوط به Messages اجرا کنید.

### نشانهٔ قابل‌مشاهده برای اپراتور

انباشت سرکوب‌شده در سطح پیش‌فرض ثبت می‌شود و هرگز بی‌سروصدا کنار گذاشته نمی‌شود (پرچم `recovery` نشان می‌دهد کدام بازه اعمال شده است):

```text
imessage: انباشت ورودی قدیمی سرکوب شد account=<id> sent=<iso> recovery=<bool> (<N> مورد از زمان شروع سرکوب شده است)
```

### مهاجرت

`channels.imessage.catchup.*` منسوخ شده است — بازیابی زمان ازکارافتادگی خودکار است و برای راه‌اندازی‌های جدید به هیچ پیکربندی‌ای نیاز ندارد. پیکربندی‌های موجود دارای `catchup.enabled: true` همچنان به‌عنوان نمایهٔ سازگاری برای بازهٔ بازپخش بازیابی رعایت می‌شوند. بلوک‌های catchup غیرفعال (`enabled: false` یا بدون `enabled: true`) بازنشسته شده‌اند؛ `openclaw doctor --fix` آن‌ها را حذف می‌کند.

## عیب‌یابی

<AccordionGroup>
  <Accordion title="imsg یافت نشد یا RPC پشتیبانی نمی‌شود">
    فایل اجرایی و پشتیبانی RPC را اعتبارسنجی کنید:

    ```bash
    imsg rpc --help
    imsg status --json
    openclaw channels status --probe
    ```

    اگر بررسی پشتیبانی‌نشدن RPC را گزارش کرد، `imsg` را به‌روزرسانی کنید. اگر کنش‌های API خصوصی در دسترس نیستند، `imsg launch` را در نشست کاربر واردشدهٔ macOS اجرا کنید و دوباره بررسی کنید. اگر Gateway روی macOS اجرا نمی‌شود، به‌جای مسیر محلی پیش‌فرض `imsg` از راه‌اندازی Remote Mac از طریق SSH در بالا استفاده کنید.

  </Accordion>

  <Accordion title="پیام‌ها ارسال می‌شوند، اما iMessageهای ورودی نمی‌رسند">
    ابتدا مشخص کنید آیا پیام به Mac محلی رسیده است. اگر `chat.db` تغییر نکند، حتی زمانی که `imsg status --json` سلامت پل را گزارش می‌کند، OpenClaw نمی‌تواند پیام را دریافت کند.

```bash
imsg chats --limit 10 --json
imsg watch --chat-id <chat-id> --json
sqlite3 ~/Library/Messages/chat.db \
  "select datetime(max(date)/1000000000 + 978307200, 'unixepoch', 'localtime'), max(ROWID) from message;"
```

    اگر پیام‌های ارسال‌شده از تلفن هیچ ردیف جدیدی ایجاد نمی‌کنند، پیش از تغییر پیکربندی OpenClaw، لایه‌های Messages در macOS و Apple Push را تعمیر کنید. یک تازه‌سازی یک‌بارهٔ سرویس اغلب کافی است:

```bash
launchctl kickstart -k system/com.apple.apsd
launchctl kickstart -k gui/$(id -u)/com.apple.CommCenter
launchctl kickstart -k gui/$(id -u)/com.apple.identityservicesd
launchctl kickstart -k gui/$(id -u)/com.apple.imagent
imsg launch
openclaw gateway restart
```

    یک iMessage جدید از تلفن ارسال کنید و پیش از عیب‌یابی نشست‌های OpenClaw، ایجاد یک ردیف جدید `chat.db` یا رویداد `imsg watch` را تأیید کنید. این کار را به‌صورت حلقهٔ دوره‌ای راه‌اندازی مجدد پل اجرا نکنید؛ اجرای مکرر `imsg launch` همراه با راه‌اندازی مجدد Gateway هنگام کار فعال می‌تواند تحویل‌ها را مختل و اجراهای در حال انجام کانال را معلق کند.

  </Accordion>

  <Accordion title="Gateway روی macOS اجرا نمی‌شود">
    `cliPath: "imsg"` پیش‌فرض باید روی Mac واردشده به Messages اجرا شود. در Linux یا Windows، `channels.imessage.cliPath` را روی یک اسکریپت پوششی تنظیم کنید که از طریق SSH به آن Mac متصل شود و `imsg "$@"` را اجرا کند.

```bash
#!/usr/bin/env bash
exec ssh -T messages-mac imsg "$@"
```

    سپس اجرا کنید:

```bash
openclaw channels status --probe --channel imessage
```

  </Accordion>

  <Accordion title="پیام‌های خصوصی نادیده گرفته می‌شوند">
    بررسی کنید:

    - `channels.imessage.dmPolicy`
    - `channels.imessage.allowFrom`
    - تأییدهای جفت‌سازی (`openclaw pairing list imessage`)

  </Accordion>

  <Accordion title="پیام‌های گروهی نادیده گرفته می‌شوند">
    بررسی کنید:

    - `channels.imessage.groupPolicy`
    - `channels.imessage.groupAllowFrom`
    - رفتار فهرست مجاز `channels.imessage.groups`
    - پیکربندی الگوی اشاره (`agents.entries.*.groupChat.mentionPatterns`)

  </Accordion>

  <Accordion title="پیوست‌های راه‌دور ناموفق‌اند">
    بررسی کنید:

    - `channels.imessage.remoteHost`
    - `channels.imessage.remoteAttachmentRoots`
    - احراز هویت کلیدی SSH/SCP از میزبان Gateway
    - وجود کلید میزبان در `~/.ssh/known_hosts` روی میزبان Gateway
    - خوانا بودن مسیر راه‌دور روی Mac اجراکنندهٔ Messages

  </Accordion>

  <Accordion title="درخواست‌های مجوز macOS از دست رفته‌اند">
    در یک ترمینال گرافیکی تعاملی، در همان بافت کاربر/نشست، دوباره اجرا و درخواست‌ها را تأیید کنید:

    ```bash
    imsg chats --limit 1
    imsg send <handle> "test"
    ```

    تأیید کنید که Full Disk Access + Automation برای بافت فرایندی که OpenClaw/`imsg` را اجرا می‌کند اعطا شده‌اند.

  </Accordion>
</AccordionGroup>

## ارجاعات مرجع پیکربندی

- [مرجع پیکربندی - iMessage](/fa/gateway/config-channels#imessage)
- [پیکربندی Gateway](/fa/gateway/configuration)
- [جفت‌سازی](/fa/channels/pairing)

## مطالب مرتبط

- [نمای کلی کانال‌ها](/fa/channels) — همهٔ کانال‌های پشتیبانی‌شده
- [حذف BlueBubbles و مسیر imsg برای iMessage](/fa/announcements/bluebubbles-imessage) — اطلاعیه و خلاصهٔ مهاجرت
- [مهاجرت از BlueBubbles](/fa/channels/imessage-from-bluebubbles) — جدول تبدیل پیکربندی و انتقال گام‌به‌گام
- [جفت‌سازی](/fa/channels/pairing) — احراز هویت پیام خصوصی و جریان جفت‌سازی
- [گروه‌ها](/fa/channels/groups) — رفتار گفت‌وگوی گروهی و محدودسازی بر اساس اشاره
- [مسیریابی کانال](/fa/channels/channel-routing) — مسیریابی نشست برای پیام‌ها
- [امنیت](/fa/gateway/security) — مدل دسترسی و مقاوم‌سازی
