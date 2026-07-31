---
summary: مرجع تفصیلی حوزه‌های محصول و بررسی‌های زیربنای کارنامه بلوغ OpenClaw.
title: رده‌بندی بلوغ
x-i18n:
    generated_at: "2026-07-27T16:43:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: dbed5794f43b230d8c8adb211734d93eb81f02623fe57df078e1dfd5a6d2586c
    source_path: maturity/taxonomy.md
    workflow: 16
---

# رده‌بندی بلوغ

<div className="maturity-hero maturity-hero-compact">
  <p className="maturity-kicker">مدل زیربنایی کارت امتیاز</p>
  <p className="maturity-hero-title">سطوح &gt; دسته‌ها &gt; قابلیت‌ها &gt; شواهد.</p>
  <p>50 سطح در 4 خانواده گروه‌بندی شده‌اند و هر دسته به مستندات مرجع و شناسه‌های پوشش QA پیوند دارد.</p>
  <p className="maturity-jump-links"><a href="#product-areas">مرور حوزه‌های محصول</a> / <a href="#taxonomy-details">باز کردن رده‌بندی تفصیلی</a> / <a href="/fa/maturity/scorecard">مشاهده امتیازها</a></p>
</div>

## نحوه خواندن این صفحه

سطح، حوزه‌ای از محصول مانند زمان اجرای Gateway،‏ Discord یا برنامه macOS است. هر سطح شامل دسته‌هایی است و هر دسته بررسی‌های سطح قابلیت را در بر می‌گیرد که سناریوهای QA پوشش می‌دهند. برای ارزیابی در سطح انتشار از کارت امتیاز استفاده کنید؛ برای بررسی مدل زیربنایی آن، از این صفحه استفاده کنید.

## سطوح بلوغ

<div className="maturity-level-list">
  <div className="maturity-level-row"><span className="maturity-level-title"><span className="maturity-level-pill maturity-level-experimental"><span className="maturity-level-code">M0</span><span>برنامه‌ریزی‌شده</span></span></span><span>مسیر کلی مشخص است، اما هیچ مسیر کاربری پشتیبانی‌شده‌ای وجود ندارد.</span><span className="maturity-level-promotion">ارتقا: مسئله طراحی، مسئول و سطح هدف وجود دارند.</span></div>
  <div className="maturity-level-row"><span className="maturity-level-title"><span className="maturity-level-pill maturity-level-experimental"><span className="maturity-level-code">M1</span><span>آزمایشی</span></span></span><span>با محدودیت‌ها، پرچم‌ها، ساخت‌های منبع یا جریان‌های مختص نگه‌دارندگان پیاده‌سازی شده است.</span><span className="maturity-level-promotion">ارتقا: نگه‌دارنده می‌تواند سناریو را از نسخه فعلی شاخه main اجرا کند.</span></div>
  <div className="maturity-level-row"><span className="maturity-level-title"><span className="maturity-level-pill maturity-level-alpha"><span className="maturity-level-code">M2</span><span>آلفا</span></span></span><span>کاربران واقعی می‌توانند آن را امتحان کنند، اما انتظار تغییرات ناسازگار و تجربه کاربری ناقص وجود دارد.</span><span className="maturity-level-promotion">ارتقا: راه‌اندازی مستندشده، آزمون‌های پایه، محدودیت‌های شناخته‌شده و دست‌کم یک اثبات در محیط واقعی.</span></div>
  <div className="maturity-level-row"><span className="maturity-level-title"><span className="maturity-level-pill maturity-level-beta"><span className="maturity-level-code">M3</span><span>بتا</span></span></span><span>مسیر عمومی وجود دارد و گردش‌کار اصلی با محدودیت‌های مشخص قابل‌استفاده است.</span><span className="maturity-level-promotion">ارتقا: مستندات نصب/به‌روزرسانی، آزمون‌های رگرسیون، راهنمای عملیات پشتیبانی و اثبات موفق سناریو در سراسر محیط مورد انتظار.</span></div>
  <div className="maturity-level-row"><span className="maturity-level-title"><span className="maturity-level-pill maturity-level-stable"><span className="maturity-level-code">M4</span><span>پایدار</span></span></span><span>مسیر پیشنهادی برای کاربران عادی. خرابی‌ها به‌عنوان رگرسیون در نظر گرفته می‌شوند.</span><span className="maturity-level-promotion">ارتقا: دروازه انتشار، مسیر doctor/عیب‌یابی، مستندات گسترده و اثبات مکرر در دنیای واقعی.</span></div>
  <div className="maturity-level-row"><span className="maturity-level-title"><span className="maturity-level-pill maturity-level-clawesome"><span className="maturity-level-code">M5</span><span>Clawesome</span></span></span><span>پرداخت‌شده، لذت‌بخش، دارای ابزارهای پایش مناسب و رقابت‌پذیر با بهترین گردش‌کار مشابه.</span><span className="maturity-level-promotion">ارتقا: سطح پایدار به‌علاوه قبولی در کارت امتیاز کاربران میان کاربران نماینده.</span></div>
</div>

## حوزه‌های محصول

<a id="product-areas" />

<Tabs>
  <Tab title="هسته">

    <a className="maturity-surface-link" href="#cli">
      <span className="maturity-surface-title">CLI</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-stable"><span className="maturity-level-code">M4</span><span>پایدار</span></span><span>7 حوزه - 90% تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#gateway-runtime">
      <span className="maturity-surface-title">زمان اجرای Gateway</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-stable"><span className="maturity-level-code">M4</span><span>پایدار</span></span><span>13 حوزه - 89% تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#agent-runtime">
      <span className="maturity-surface-title">زمان اجرای عامل</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-beta"><span className="maturity-level-code">M3</span><span>بتا</span></span><span>9 حوزه - 79% تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#session-memory-and-context-engine">
      <span className="maturity-surface-title">موتور نشست، حافظه و زمینه</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-beta"><span className="maturity-level-code">M3</span><span>بتا</span></span><span>9 حوزه - 79% تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#channel-framework">
      <span className="maturity-surface-title">چارچوب کانال</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-beta"><span className="maturity-level-code">M3</span><span>بتا</span></span><span>8 حوزه - 79% تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#observability">
      <span className="maturity-surface-title">مشاهده‌پذیری</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-beta"><span className="maturity-level-code">M3</span><span>بتا</span></span><span>5 حوزه - 79% تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#gateway-web-app">
      <span className="maturity-surface-title">برنامه وب Gateway</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-beta"><span className="maturity-level-code">M3</span><span>بتا</span></span><span>6 حوزه - 79% تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#plugins">
      <span className="maturity-surface-title">Pluginها</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-beta"><span className="maturity-level-code">M3</span><span>بتا</span></span><span>9 حوزه - 79% تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#security-auth-pairing-and-secrets">
      <span className="maturity-surface-title">امنیت، احراز هویت، جفت‌سازی و اسرار</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-beta"><span className="maturity-level-code">M3</span><span>بتا</span></span><span>6 حوزه - 79% تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#automation-cron-hooks-tasks-polling">
      <span className="maturity-surface-title">خودکارسازی: cron، هوک‌ها، وظایف، نظرسنجی</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-beta"><span className="maturity-level-code">M3</span><span>بتا</span></span><span>6 حوزه - 79% تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#media-understanding-and-media-generation">
      <span className="maturity-surface-title">درک و تولید رسانه</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-alpha"><span className="maturity-level-code">M2</span><span>آلفا</span></span><span>6 حوزه - 68% تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#voice-and-realtime-talk">
      <span className="maturity-surface-title">صدا و گفت‌وگوی بلادرنگ</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-alpha"><span className="maturity-level-code">M2</span><span>آلفا</span></span><span>6 حوزه - 68% تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#tui">
      <span className="maturity-surface-title">TUI</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-alpha"><span className="maturity-level-code">M2</span><span>آلفا</span></span><span>5 حوزه - 66% تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#clawhub">
      <span className="maturity-surface-title">ClawHub</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-alpha"><span className="maturity-level-code">M2</span><span>آلفا</span></span><span>4 حوزه - 62% تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#openclaw-app-sdk">
      <span className="maturity-surface-title">SDK برنامه OpenClaw</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-alpha"><span className="maturity-level-code">M2</span><span>آلفا</span></span><span>6 حوزه - 53% تکمیل‌شده</span></span>
    </a>

  </Tab>
  <Tab title="پلتفرم">

    <a className="maturity-surface-link" href="#linux-gateway-host">
      <span className="maturity-surface-title">میزبان Linux برای Gateway</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-stable"><span className="maturity-level-code">M4</span><span>پایدار</span></span><span>5 حوزه - 89% تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#macos-gateway-host">
      <span className="maturity-surface-title">میزبان macOS برای Gateway</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-stable"><span className="maturity-level-code">M4</span><span>پایدار</span></span><span>7 حوزه - 88% تکمیل‌شده</span></span>
    </a>
    <a className="maturity-surface-link" href="#android-app">
      <span className="maturity-surface-title">برنامه Android</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-stable"><span className="maturity-level-code">M4</span><span>پایدار</span></span><span>7 حوزه - 80% تکمیل‌شده</span></span>
    </a>
    <a className="maturity-surface-link" href="#ios-app">
      <span className="maturity-surface-title">برنامه iOS</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-stable"><span className="maturity-level-code">M4</span><span>پایدار</span></span><span>8 حوزه - 80% تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#docker-and-podman-hosting">
      <span className="maturity-surface-title">میزبانی با Docker و Podman</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-beta"><span className="maturity-level-code">M3</span><span>بتا</span></span><span>4 حوزه - 79% تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#windows-via-wsl2">
      <span className="maturity-surface-title">Windows از طریق WSL2</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-beta"><span className="maturity-level-code">M3</span><span>بتا</span></span><span>6 حوزه - 79% تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#raspberry-pi-and-small-linux-devices">
      <span className="maturity-surface-title">Raspberry Pi و دستگاه‌های کوچک Linux</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-beta"><span className="maturity-level-code">M3</span><span>بتا</span></span><span>4 حوزه - 79% تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#macos-companion-app">
      <span className="maturity-surface-title">برنامه همراه macOS</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-beta"><span className="maturity-level-code">M3</span><span>بتا</span></span><span>8 حوزه - 78٪ تکمیل‌شده</span></span>
    </a>


    <a className="maturity-surface-link" href="#native-windows">
      <span className="maturity-surface-title">Windows بومی</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-alpha"><span className="maturity-level-code">M2</span><span>آلفا</span></span><span>4 حوزه - 66٪ تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#kubernetes-hosting">
      <span className="maturity-surface-title">میزبانی Kubernetes</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-alpha"><span className="maturity-level-code">M2</span><span>آلفا</span></span><span>4 حوزه - 61٪ تکمیل‌شده</span></span>
    </a>


    <a className="maturity-surface-link" href="#nix-install-path">
      <span className="maturity-surface-title">مسیر نصب Nix</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-experimental"><span className="maturity-level-code">M1</span><span>آزمایشی</span></span><span>5 حوزه - 44٪ تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#watchos-companion-surfaces">
      <span className="maturity-surface-title">سطوح همراه watchOS</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-experimental"><span className="maturity-level-code">M1</span><span>آزمایشی</span></span><span>5 حوزه - 44٪ تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#linux-companion-app">
      <span className="maturity-surface-title">برنامه همراه Linux</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-experimental"><span className="maturity-level-code">M0</span><span>برنامه‌ریزی‌شده</span></span><span>5 حوزه - 21٪ تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#native-windows-companion-app">
      <span className="maturity-surface-title">برنامه همراه بومی Windows</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-experimental"><span className="maturity-level-code">M0</span><span>برنامه‌ریزی‌شده</span></span><span>5 حوزه - 21٪ تکمیل‌شده</span></span>
    </a>

  </Tab>
  <Tab title="کانال">

    <a className="maturity-surface-link" href="#discord">
      <span className="maturity-surface-title">Discord</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-stable"><span className="maturity-level-code">M4</span><span>پایدار</span></span><span>6 حوزه - 87٪ تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#telegram">
      <span className="maturity-surface-title">Telegram</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-beta"><span className="maturity-level-code">M3</span><span>بتا</span></span><span>5 حوزه - 78٪ تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#slack">
      <span className="maturity-surface-title">Slack</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-beta"><span className="maturity-level-code">M3</span><span>بتا</span></span><span>5 حوزه - 78٪ تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#imessage-and-bluebubbles">
      <span className="maturity-surface-title">iMessage و BlueBubbles</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-beta"><span className="maturity-level-code">M3</span><span>بتا</span></span><span>5 حوزه - 78٪ تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#whatsapp">
      <span className="maturity-surface-title">WhatsApp</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-beta"><span className="maturity-level-code">M3</span><span>بتا</span></span><span>5 حوزه - 78٪ تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#matrix">
      <span className="maturity-surface-title">Matrix</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-alpha"><span className="maturity-level-code">M2</span><span>آلفا</span></span><span>6 حوزه - 67٪ تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#google-chat">
      <span className="maturity-surface-title">Google Chat</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-alpha"><span className="maturity-level-code">M2</span><span>آلفا</span></span><span>5 حوزه - 66٪ تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#microsoft-teams">
      <span className="maturity-surface-title">Microsoft Teams</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-alpha"><span className="maturity-level-code">M2</span><span>آلفا</span></span><span>5 حوزه - 66٪ تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#signal">
      <span className="maturity-surface-title">Signal</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-alpha"><span className="maturity-level-code">M2</span><span>آلفا</span></span><span>5 حوزه - 66٪ تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#feishu-qq-bot-wechat-yuanbao-zalo-zalo-personal-regional-channels">
      <span className="maturity-surface-title">Feishu، QQ Bot، WeChat، Yuanbao، Zalo، Zalo Personal، کانال‌های منطقه‌ای</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-alpha"><span className="maturity-level-code">M2</span><span>آلفا</span></span><span>4 حوزه - 58٪ تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#mattermost-line-irc-nextcloud-talk-nostr-twitch-tlon-synology-chat">
      <span className="maturity-surface-title">Mattermost، LINE، IRC، Nextcloud Talk، Nostr، Twitch، Tlon، Synology Chat</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-alpha"><span className="maturity-level-code">M2</span><span>آلفا</span></span><span>4 حوزه - 54٪ تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#voice-call-channel">
      <span className="maturity-surface-title">کانال تماس صوتی</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-experimental"><span className="maturity-level-code">M1</span><span>آزمایشی</span></span><span>5 حوزه - 44٪ تکمیل‌شده</span></span>
    </a>

  </Tab>
  <Tab title="ارائه‌دهنده و ابزار">

    <a className="maturity-surface-link" href="#browser-automation-exec-and-sandbox-tools">
      <span className="maturity-surface-title">ابزارهای خودکارسازی مرورگر، exec و sandbox</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-beta"><span className="maturity-level-code">M3</span><span>بتا</span></span><span>3 حوزه - 79٪ تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#openai-and-codex-provider-path">
      <span className="maturity-surface-title">مسیر ارائه‌دهنده OpenAI و Codex</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-beta"><span className="maturity-level-code">M3</span><span>بتا</span></span><span>5 حوزه - 79٪ تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#web-search-tools">
      <span className="maturity-surface-title">ابزارهای جست‌وجوی وب</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-beta"><span className="maturity-level-code">M3</span><span>بتا</span></span><span>4 حوزه - 79٪ تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#anthropic-provider-path">
      <span className="maturity-surface-title">مسیر ارائه‌دهنده Anthropic</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-beta"><span className="maturity-level-code">M3</span><span>بتا</span></span><span>5 حوزه - 78٪ تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#google-provider-path">
      <span className="maturity-surface-title">مسیر ارائه‌دهنده Google</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-beta"><span className="maturity-level-code">M3</span><span>بتا</span></span><span>5 حوزه - 78٪ تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#openrouter-provider-path">
      <span className="maturity-surface-title">مسیر ارائه‌دهنده OpenRouter</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-beta"><span className="maturity-level-code">M3</span><span>بتا</span></span><span>4 حوزه - 78٪ تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#image-video-and-music-generation-tools">
      <span className="maturity-surface-title">ابزارهای تولید تصویر، ویدئو و موسیقی</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-alpha"><span className="maturity-level-code">M2</span><span>آلفا</span></span><span>5 حوزه - 68٪ تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#local-model-providers-ollama-vllm-sglang-lm-studio">
      <span className="maturity-surface-title">ارائه‌دهندگان مدل محلی: Ollama، vLLM، SGLang، LM Studio</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-alpha"><span className="maturity-level-code">M2</span><span>آلفا</span></span><span>5 حوزه - 68٪ تکمیل‌شده</span></span>
    </a>

    <a className="maturity-surface-link" href="#long-tail-hosted-providers">
      <span className="maturity-surface-title">ارائه‌دهندگان میزبانی‌شده کم‌کاربرد</span>
      <span className="maturity-surface-meta"><span className="maturity-level-pill maturity-level-alpha"><span className="maturity-level-code">M2</span><span>آلفا</span></span><span>3 حوزه - 68٪ تکمیل‌شده</span></span>
    </a>

  </Tab>
</Tabs>

## جزئیات

<a id="taxonomy-details" />

### هسته

<AccordionGroup>
  <Accordion title="CLI - M4 پایدار - 7 حوزه">
    <a id="cli" />

    مسیرهای عادی راه‌اندازی و تعمیر در مستندات نصب، CLI و Gateway شرح داده شده‌اند. مسیرهای ویژه پلتفرم Windows در ردیف‌های Windows از طریق WSL2 و Windows بومی پیگیری می‌شوند.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 4٪</span><span>کیفیت پایدار - 83٪</span><span>کامل‌بودن پایدار - 90٪</span><span><span className="maturity-lts maturity-lts-partial">جزئی - 6</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">راه‌اندازی CLI</span>
          <span>6 قابلیت / پشتیبانی‌شده در LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>17%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "17%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>89%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "89%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>90%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "90%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [نمایه](/fa/install/index)، [نصب‌کننده](/fa/install/installer)، [Node](/fa/install/node)، [به‌روزرسانی](/fa/install/updating)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">راه‌اندازی آغاز به کار و احراز هویت</span>
          <span>5 قابلیت / پشتیبانی‌شده در LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>75%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "75%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>89%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "89%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [آغاز به کار](/fa/cli/onboard)، [پیکربندی](/fa/cli/configure)، [نمای کلی آغاز به کار](/fa/start/onboarding-overview)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">راه‌اندازی Plugin و کانال</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>75%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "75%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>89%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "89%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [آغاز به کار](/fa/cli/onboard)، [Pluginها](/fa/cli/plugins)، [کانال‌ها](/fa/cli/channels)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">مدیریت سرویس Gateway</span>
          <span>5 قابلیت / پشتیبانی‌شده در LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>14%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "14%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>87%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "87%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>90%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "90%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Gateway](/fa/cli/gateway)، [به‌روزرسانی](/fa/install/updating)، [عیب‌یابی](/fa/gateway/troubleshooting)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">مشاهده‌پذیری CLI</span>
          <span>5 قابلیت / پشتیبانی‌شده در LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>89%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "89%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>90%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "90%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [وضعیت](/fa/cli/status)، [سلامت](/fa/cli/health)، [گزارش‌ها](/fa/cli/logs)، [تشخیص مشکلات](/fa/gateway/diagnostics)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">Doctor</span>
          <span>10 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>89%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "89%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>90%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "90%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Doctor](/fa/cli/doctor)، [Doctor](/fa/gateway/doctor)، [اسرار](/fa/gateway/secrets)، [عیب‌یابی](/fa/gateway/troubleshooting)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">به‌روزرسانی‌ها و ارتقاها</span>
          <span>5 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>75%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "75%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>89%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "89%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [به‌روزرسانی](/fa/install/updating)، [به‌روزرسانی](/fa/cli/update)، [عیب‌یابی](/fa/gateway/troubleshooting)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="زمان اجرای Gateway - M4 پایدار - 13 حوزه">
    <a id="gateway-runtime" />

    معماری هسته، احراز هویت، جفت‌سازی، مستندات پروتکل، مستندات سرویس پس‌زمینه و راهنماهای عملیاتی CLI گسترده و به‌روز هستند.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 6%</span><span>کیفیت پایدار - 81%</span><span>کامل‌بودن پایدار - 89%</span><span><span className="maturity-lts maturity-lts-partial">جزئی - 12</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">تأییدها و اجرای از راه دور</span>
          <span>6 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>75%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "75%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>89%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "89%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [پروتکل](/fa/gateway/protocol)، [نمایه](/fa/gateway/security/index)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">APIهای HTTP</span>
          <span>4 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>25%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "25%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>90%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "90%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>90%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "90%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [نمایه](/fa/gateway/index)، [API HTTP اپن‌ای‌آی](/fa/gateway/openai-http-api)، [API HTTP پاسخ‌های باز](/fa/gateway/openresponses-http-api)، [API HTTP فراخوانی ابزارها](/fa/gateway/tools-invoke-http-api)، [هوک‌ها](/fa/automation/hooks)، [نمایه](/fa/web/index)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">سطح وب میزبانی‌شده</span>
          <span>4 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>89%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "89%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>90%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "90%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [نمایه](/fa/gateway/index)، [معماری](/fa/concepts/architecture)، [رابط کاربری کنترل](/fa/web/control-ui)، [گفت‌وگوی وبی](/fa/web/webchat)، [بوم](/fa/refactor/canvas)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">APIها و رویدادهای RPC در Gateway</span>
          <span>20 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>9%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "9%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>90%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "90%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>90%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "90%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [پروتکل](/fa/gateway/protocol)، [نمایه](/fa/gateway/index)، [معماری](/fa/concepts/architecture)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">احراز هویت و جفت‌سازی دستگاه</span>
          <span>10 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>75%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "75%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>89%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "89%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [پروتکل](/fa/gateway/protocol)، [جفت‌سازی](/fa/gateway/pairing)، [نمایه](/fa/gateway/security/index)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">دسترسی و کشف شبکه</span>
          <span>6 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>75%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "75%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>89%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "89%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [نمایه](/fa/gateway/index)، [کشف](/fa/gateway/discovery)، [پروتکل](/fa/gateway/protocol)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">Nodeها و قابلیت‌های راه دور</span>
          <span>8 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>75%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "75%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>89%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "89%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [پروتکل](/fa/gateway/protocol)، [معماری](/fa/concepts/architecture)، [نمایه](/fa/nodes/index)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">سلامت، عیب‌یابی و ترمیم</span>
          <span>7 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>75%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "75%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>89%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "89%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [نمایه](/fa/gateway/index)، [عیب‌یابی](/fa/gateway/diagnostics)، [پزشک](/fa/gateway/doctor)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">سازگاری پروتکل</span>
          <span>7 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>75%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "75%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>89%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "89%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [پروتکل](/fa/gateway/protocol)، [معماری](/fa/concepts/architecture)، [Typebox](/fa/concepts/typebox)، [پروتکل پل](/fa/gateway/bridge-protocol)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">نقش‌ها و مجوزها</span>
          <span>5 قابلیت / با پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>75%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "75%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>89%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "89%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [پروتکل](/fa/gateway/protocol)، [نمایه](/fa/gateway/security/index)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">چرخه حیات Gateway</span>
          <span>7 قابلیت / با پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>33%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "33%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>90%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "90%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>90%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "90%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [نمایه](/fa/gateway/index)، [معماری](/fa/concepts/architecture)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">کنترل‌های امنیتی</span>
          <span>6 قابلیت / با پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>75%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "75%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>89%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "89%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [نمایه](/fa/gateway/security/index)، [پروتکل](/fa/gateway/protocol)، [کشف](/fa/gateway/discovery)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">اتصال WebSocket</span>
          <span>8 قابلیت / با پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>13%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "13%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>90%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "90%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>90%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "90%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [پروتکل](/fa/gateway/protocol)، [معماری](/fa/concepts/architecture)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="زمان اجرای عامل - بتای M3 - 9 حوزه">
    <a id="agent-runtime" />

    حلقه اصلی، مدل‌ها، مسیریابی ارائه‌دهنده و استریم ابزارها از قابلیت‌های درجه‌یک هستند، اما رفتار ارائه‌دهندگان هر هفته تغییر می‌کند و برای هر انتشار به اثبات سناریویی نیاز دارد.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 33%</span><span>کیفیت بتا - 78%</span><span>کامل‌بودن بتا - 79%</span><span><span className="maturity-lts maturity-lts-partial">جزئی - 6</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">اجرای نوبت عامل</span>
          <span>3 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>29%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "29%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [حلقه عامل](/fa/concepts/agent-loop)، [عامل](/fa/cli/agent)، [زمان‌های اجرای عامل](/fa/concepts/agent-runtimes)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">زمان‌های اجرای خارجی و عامل‌های فرعی</span>
          <span>4 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>30%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "30%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [زمان‌های اجرای عامل](/fa/concepts/agent-runtimes)، [Anthropic](/fa/providers/anthropic)، [Google](/fa/providers/google)، [عامل‌های فرعی](/fa/tools/subagents)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">اجرای ارائه‌دهنده میزبانی‌شده</span>
          <span>5 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>20%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "20%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [OpenAI](/fa/providers/openai)، [Anthropic](/fa/providers/anthropic)، [Google](/fa/providers/google)، [مدل‌ها](/fa/concepts/models)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">ارائه‌دهندگان محلی و خودمیزبان</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Ollama](/fa/providers/ollama)، [مدل‌ها](/fa/concepts/models)، [عامل](/fa/cli/agent)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">انتخاب مدل و محیط اجرا</span>
          <span>4 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>25%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "25%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [مدل‌ها](/fa/concepts/models)، [مدل‌ها](/fa/cli/models)، [OpenAI](/fa/providers/openai)، [محیط‌های اجرای عامل](/fa/concepts/agent-runtimes)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">احراز هویت ارائه‌دهنده</span>
          <span>10 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>24%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "24%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [مدل‌ها](/fa/concepts/models)، [عامل](/fa/cli/agent)، [مدل‌ها](/fa/cli/models)، [OpenAI](/fa/providers/openai)، [Anthropic](/fa/providers/anthropic)، [Google](/fa/providers/google)، [عامل‌های فرعی](/fa/tools/subagents)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">جریان‌سازی و پیشرفت</span>
          <span>2 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>56%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "56%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [جریان‌سازی](/fa/concepts/streaming)، [حلقه عامل](/fa/concepts/agent-loop)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">فراخوانی ابزارها و مدیریت پاسخ</span>
          <span>3 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>65%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "65%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [حلقه عامل](/fa/concepts/agent-loop)، [Ollama](/fa/providers/ollama)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">کنترل‌های اجرای ابزار</span>
          <span>6 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>50%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "50%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [سندباکس در برابر خط‌مشی ابزار در برابر دسترسی ارتقایافته](/fa/gateway/sandbox-vs-tool-policy-vs-elevated)، [حلقه عامل](/fa/concepts/agent-loop)، [عامل‌های فرعی](/fa/tools/subagents)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="نشست، حافظه و موتور زمینه - بتای M3 - 9 حوزه">
    <a id="session-memory-and-context-engine" />

    مستندات قوی و پیاده‌سازی فعال است. بلوغ به ماندگاری رونوشت، کیفیت Compaction و برابری میان کلاینت‌ها بستگی دارد.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 30%</span><span>کیفیت بتا - 77%</span><span>کامل‌بودن بتا - 79%</span><span><span className="maturity-lts maturity-lts-partial">جزئی - 6</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">مدیریت نشست و رونوشت CLI</span>
          <span>2 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [نشست](/fa/concepts/session)، [Compaction مدیریت نشست](/fa/reference/session-management-compaction)، [نشست‌ها](/fa/cli/sessions)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">مدیریت توکن</span>
          <span>3 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>20%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "20%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Compaction](/fa/concepts/compaction)، [زمینه](/fa/concepts/context)، [Compaction مدیریت نشست](/fa/reference/session-management-compaction)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">موتور زمینه</span>
          <span>2 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>57%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "57%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [زمینه](/fa/concepts/context)، [موتور زمینه](/fa/concepts/context-engine)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">هم‌ترازی تاریخچه و نشست میان کلاینت‌ها</span>
          <span>2 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>40%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "40%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [گپ تحت وب](/fa/web/webchat)، [Android](/fa/platforms/android)، [مسیریابی کانال](/fa/channels/channel-routing)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">عیب‌یابی، نگه‌داری و بازیابی</span>
          <span>3 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>40%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "40%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [عیب‌یابی](/fa/gateway/diagnostics)، [Compaction مدیریت نشست](/fa/reference/session-management-compaction)، [پرچم‌ها](/fa/diagnostics/flags)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">پرامپت‌ها و زمینهٔ هسته</span>
          <span>2 قابلیت / با پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>38%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "38%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [زمینه](/fa/concepts/context)، [پاکیزگی رونوشت](/fa/reference/transcript-hygiene)، [Discord](/fa/channels/discord)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">حافظه</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>46%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "46%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [پیکربندی حافظه](/fa/reference/memory-config)، [Qmd حافظه](/fa/concepts/memory-qmd)، [حافظه](/fa/concepts/memory)، [Discord](/fa/channels/discord)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">مسیریابی نشست</span>
          <span>2 قابلیت / با پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>25%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "25%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [نشست](/fa/concepts/session)، [مسیریابی کانال](/fa/channels/channel-routing)، [Discord](/fa/channels/discord)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">ماندگاری رونوشت</span>
          <span>2 قابلیت / با پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Compaction مدیریت نشست](/fa/reference/session-management-compaction)، [بهداشت رونوشت](/fa/reference/transcript-hygiene)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="چارچوب کانال - M3 بتا - 8 حوزه">
    <a id="channel-framework" />

    بسیاری از کانال‌ها قراردادهای تحویل و مسیریابی Gateway را به‌اشتراک می‌گذارند، اما رفتار کانال بر اساس API بالادستی و محدودیت‌های سیاست حساب متفاوت است.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 13%</span><span>کیفیت بتا - 76%</span><span>کامل‌بودن بتا - 79%</span><span><span className="maturity-lts maturity-lts-partial">جزئی - 5</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">فرمان‌ها و تأییدهای کنش‌های کانال</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [گروه‌ها](/fa/channels/groups)، [Discord](/fa/channels/discord)، [Google Chat](/fa/channels/googlechat)، [Signal](/fa/channels/signal)، [Matrix](/fa/channels/matrix)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">راه‌اندازی کانال</span>
          <span>5 قابلیت / با پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>14%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "14%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [نمایه](/fa/channels/index)، [جفت‌سازی](/fa/channels/pairing)، [عیب‌یابی](/fa/channels/troubleshooting)، [Pluginهای کانال SDK](/fa/plugins/sdk-channel-plugins)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">رفتار رشته‌گفت‌وگوی گروهی و اتاق محیطی</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>36%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "36%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [گروه‌ها](/fa/channels/groups)، [پیام‌های گروهی](/fa/channels/group-messages)، [رویدادهای اتاق محیطی](/fa/channels/ambient-room-events)، [گروه‌های پخش همگانی](/fa/channels/broadcast-groups)، [Discord](/fa/channels/discord)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">درگاه‌های دسترسی ورودی و هویت</span>
          <span>5 قابلیت / با پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [گروه‌های دسترسی](/fa/channels/access-groups)، [گروه‌ها](/fa/channels/groups)، [Discord](/fa/channels/discord)، [Line](/fa/channels/line)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">پیوست‌های رسانه‌ای و داده‌های غنی کانال</span>
          <span>4 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Line](/fa/channels/line)، [Signal](/fa/channels/signal)، [Google Chat](/fa/channels/googlechat)، [Matrix](/fa/channels/matrix)، [Discord](/fa/channels/discord)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">تحویل خروجی و پایپ‌لاین پاسخ</span>
          <span>4 قابلیت / با پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>38%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "38%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [گروه‌ها](/fa/channels/groups)، [رویدادهای اتاق محیطی](/fa/channels/ambient-room-events)، [Discord](/fa/channels/discord)، [Matrix](/fa/channels/matrix)، [پیکربندی کانال‌ها](/fa/gateway/config-channels)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">مسیریابی و تحویل مکالمه</span>
          <span>10 قابلیت / با پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>19%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "19%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [مسیریابی کانال](/fa/channels/channel-routing)، [گروه‌ها](/fa/channels/groups)، [Discord](/fa/channels/discord)، [Matrix](/fa/channels/matrix)، [عیب‌یابی](/fa/channels/troubleshooting)، [مرجع پیکربندی](/fa/gateway/configuration-reference)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">سلامت وضعیت و کنترل‌های اپراتور</span>
          <span>4 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [سلامت](/fa/gateway/health)، [مرجع پیکربندی](/fa/gateway/configuration-reference)، [عیب‌یابی](/fa/channels/troubleshooting)، [Discord](/fa/channels/discord)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="مشاهده‌پذیری - M3 بتا - 5 حوزه">
    <a id="observability" />

    مستندات OTel، Prometheus، ثبت گزارش و تشخیص مشکلات موجود است. به یک مرحله بلوغ عمومی درباره «اپراتورها ابتدا باید چه مواردی را بررسی کنند» نیاز دارد.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 18%</span><span>کیفیت بتا - 75%</span><span>کامل‌بودن بتا - 79%</span><span><span className="maturity-lts maturity-lts-partial">جزئی - 3</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">سلامت و ترمیم</span>
          <span>12 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>28%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "28%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [سلامت](/fa/gateway/health)، [Telegram](/fa/channels/telegram)، [پزشک](/fa/cli/doctor)، [پزشک](/fa/gateway/doctor)، [زیرمسیرهای SDK](/fa/plugins/sdk-subpaths)، [سلامت](/fa/cli/health)، [پروتکل](/fa/gateway/protocol)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">ثبت گزارش</span>
          <span>5 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [ثبت گزارش](/fa/logging)، [ثبت گزارش](/fa/gateway/logging)، [گزارش‌ها](/fa/cli/logs)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">گردآوری اطلاعات تشخیصی</span>
          <span>8 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>30%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "30%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [عیب‌یابی](/fa/gateway/diagnostics)، [سلامت](/fa/gateway/health)، [هارنس Codex](/fa/plugins/codex-harness)، [پروتکل](/fa/gateway/protocol)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">صدور داده‌های تله‌متری</span>
          <span>13 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>33%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "33%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [هوک‌ها](/fa/plugins/hooks)، [Opentelemetry](/fa/gateway/opentelemetry)، [ثبت گزارش](/fa/logging)، [زیرمسیرهای SDK](/fa/plugins/sdk-subpaths)، [عیب‌یابی Otel](/fa/plugins/reference/diagnostics-otel)، [Prometheus](/fa/gateway/prometheus)، [عیب‌یابی Prometheus](/fa/plugins/reference/diagnostics-prometheus)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">عیب‌یابی نشست</span>
          <span>4 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Opentelemetry](/fa/gateway/opentelemetry)، [Prometheus](/fa/gateway/prometheus)، [عیب‌یابی](/fa/gateway/diagnostics)، [پروتکل](/fa/gateway/protocol)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="برنامهٔ وب Gateway - بتای M3 - 6 حوزه">
    <a id="gateway-web-app" />

    رابط کاربری وب همراه با جریان‌های جفت‌سازی، گفت‌وگو، PWA، مکالمه، اعلان‌های پوش و Gateway راه دور مستند شده است. پس از تکمیل کارنامه‌های ارزیابی چندمرورگری و PWA موبایل، آن را ارتقا دهید.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 4%</span><span>کیفیت بتا - 74%</span><span>کامل‌بودن بتا - 79%</span><span><span className="maturity-lts maturity-lts-none">هیچ‌کدام</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">گفت‌وگوی بلادرنگ مرورگر</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [رابط کاربری کنترل](/fa/web/control-ui)، [پروتکل](/fa/gateway/protocol)، [گفت‌وگو](/fa/nodes/talk)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">دسترسی و اعتماد مرورگر</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [رابط کاربری کنترل](/fa/web/control-ui)، [داشبورد](/fa/web/dashboard)، [Tailscale](/fa/gateway/tailscale)، [دسترسی راه‌دور](/fa/gateway/remote)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">پیکربندی</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [رابط کاربری کنترل](/fa/web/control-ui)، [پیکربندی](/fa/gateway/configuration)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">رابط کاربری مرورگر</span>
          <span>10 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>8%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "8%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [رابط کاربری کنترل](/fa/web/control-ui)، [نمایه](/fa/web/index)، [داشبورد](/fa/web/dashboard)، [پروتکل](/fa/gateway/protocol)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">مکالمات WebChat</span>
          <span>15 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>10%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "10%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [رابط کاربری کنترل](/fa/web/control-ui)، [گفت‌وگوی وب](/fa/web/webchat)، [شروع کار](/fa/start/getting-started)، [مسیریابی کانال](/fa/channels/channel-routing)، [عملیات امن فایل](/fa/gateway/security/secure-file-operations)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">کنسول اپراتور</span>
          <span>10 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>8%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "8%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [رابط کاربری کنترل](/fa/web/control-ui)، [سلامت](/fa/gateway/health)، [پروتکل](/fa/gateway/protocol)، [داشبورد](/fa/web/dashboard)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="Pluginها - M3 بتا - 9 حوزه">
    <a id="plugins" />

    مستندات گسترده و شواهد داخلی قدرتمندی از محیط اجرا در زمینهٔ مانیفست‌ها، کشف، بارگذاری، معماری ارائه‌دهنده/ابزار و مرزهای تأیید وجود دارد. این ردیف را تا زمانی که شواهد API/زیرمسیرهای SDK عمومی و توزیع خارجی قوی‌تر شوند، در سطح بتا نگه دارید.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 12%</span><span>کیفیت بتا - 72%</span><span>کامل‌بودن بتا - 79%</span><span><span className="maturity-lts maturity-lts-partial">جزئی - 7</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">ساخت و بسته‌بندی Pluginها</span>
          <span>8 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [ساخت Pluginها](/fa/plugins/building-plugins)، [نمای کلی SDK](/fa/plugins/sdk-overview)، [نقاط ورود SDK](/fa/plugins/sdk-entrypoints)، [زیرمسیرهای SDK](/fa/plugins/sdk-subpaths)، [مانیفست](/fa/plugins/manifest)، [مرجع](/fa/plugins/reference)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">Pluginهای همراه</span>
          <span>5 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [فهرست Pluginها](/fa/plugins/plugin-inventory)، [Pluginها](/fa/cli/plugins)، [جزئیات داخلی معماری](/fa/plugins/architecture-internals)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">Plugin بوم</span>
          <span>6 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [بوم](/fa/plugins/reference/canvas)، [بوم](/fa/refactor/canvas)، [مرجع پیکربندی](/fa/gateway/configuration-reference)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">نصب و اجرای Pluginها</span>
          <span>6 قابلیت / با پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>35%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "35%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [معماری](/fa/plugins/architecture)، [جزئیات داخلی معماری](/fa/plugins/architecture-internals)، [Pluginها](/fa/cli/plugins)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">Pluginهای کانال</span>
          <span>5 قابلیت / با پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Pluginهای کانال SDK](/fa/plugins/sdk-channel-plugins)، [ورودی کانال SDK](/fa/plugins/sdk-channel-inbound)، [خروجی کانال SDK](/fa/plugins/sdk-channel-outbound)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">Pluginهای ارائه‌دهنده و ابزار</span>
          <span>6 قابلیت / با پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>43%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "43%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Pluginهای ارائه‌دهنده SDK](/fa/plugins/sdk-provider-plugins)، [Pluginهای ابزار](/fa/plugins/tool-plugins)، [افزودن قابلیت‌ها](/fa/plugins/adding-capabilities)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">تأییدیه‌های Plugin</span>
          <span>6 قابلیت / با پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [درخواست‌های مجوز Plugin](/fa/plugins/plugin-permission-requests)، [تأییدهای اجرا](/fa/tools/exec-approvals)، [Pluginهای کانال SDK](/fa/plugins/sdk-channel-plugins)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">انتشار Pluginها</span>
          <span>6 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Pluginها](/fa/cli/plugins)، [سازگاری](/fa/plugins/compatibility)، [انتشار](/fa/clawhub/publishing)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">آزمایش Pluginها</span>
          <span>6 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>27%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "27%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [آزمایش SDK](/fa/plugins/sdk-testing)، [راه‌اندازی SDK](/fa/plugins/sdk-setup)، [هارنس Codex](/fa/plugins/codex-harness)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="امنیت، احراز هویت، جفت‌سازی و اسرار - M3 بتا - 6 حوزه">
    <a id="security-auth-pairing-and-secrets" />

    مستندات مناسب و سطوح مقاوم‌سازی وجود دارند. پس از آن‌که اجرای منظم سناریوهای ارتقا و امنیت ثابت کند هیچ پس‌رفتی در راه‌اندازی وجود ندارد، ارتقا دهید.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 16%</span><span>کیفیت بتا - 72%</span><span>کامل‌بودن بتا - 79%</span><span><span className="maturity-lts maturity-lts-partial">جزئی - 5</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">سیاست تأیید و محافظ‌های ابزار</span>
          <span>2 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>50%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "50%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [تأییدهای اجرا](/fa/tools/exec-approvals)، [تأییدها](/fa/cli/approvals)، [درخواست‌های مجوز Plugin](/fa/plugins/plugin-permission-requests)، [بررسی‌های ممیزی](/fa/gateway/security/audit-checks)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">احراز هویت Gateway و دسترسی از راه دور</span>
          <span>9 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [نمایه](/fa/gateway/security/index)، [راهنمای عملیاتی مواجهه](/fa/gateway/security/exposure-runbook)، [احراز هویت پروکسی مورد اعتماد](/fa/gateway/trusted-proxy-auth)، [Tailscale](/fa/gateway/tailscale)، [دسترسی از راه دور](/fa/gateway/remote)، [مرجع پیکربندی](/fa/gateway/configuration-reference)، [Gateway](/fa/cli/gateway)، [پزشک](/fa/cli/doctor)، [رابط کاربری کنترل](/fa/web/control-ui)، [کنترل مرورگر](/fa/tools/browser-control)، [بررسی‌های ممیزی](/fa/gateway/security/audit-checks)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">کنترل دسترسی کانال</span>
          <span>3 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [جفت‌سازی](/fa/channels/pairing)، [Telegram](/fa/channels/telegram)، [گروه‌های دسترسی](/fa/channels/access-groups)، [بررسی‌های ممیزی](/fa/gateway/security/audit-checks)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">جفت‌سازی دستگاه و Node</span>
          <span>11 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [پروتکل](/fa/gateway/protocol)، [دستگاه‌ها](/fa/cli/devices)، [جفت‌سازی](/fa/channels/pairing)، [جفت‌سازی](/fa/gateway/pairing)، [دامنه‌های عملگر](/fa/gateway/operator-scopes)، [رابط کاربری کنترل](/fa/web/control-ui)، [گفت‌وگوی وب](/fa/web/webchat)، [تأییدها](/fa/cli/approvals)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">اعتماد به Plugin</span>
          <span>2 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [مانیفست](/fa/plugins/manifest)، [درخواست‌های مجوز Plugin](/fa/plugins/plugin-permission-requests)، [مدیریت Pluginها](/fa/plugins/manage-plugins)، [بررسی‌های ممیزی](/fa/gateway/security/audit-checks)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">بهداشت اعتبارنامه‌ها و اسرار</span>
          <span>5 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>46%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "46%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [احراز هویت](/fa/gateway/authentication)، [مدل‌ها](/fa/cli/models)، [OpenAI](/fa/providers/openai)، [OAuth](/fa/concepts/oauth)، [اسرار](/fa/gateway/secrets)، [اسرار](/fa/cli/secrets)، [سطح اعتبارنامه Secretref](/fa/reference/secretref-credential-surface)، [بررسی‌های ممیزی](/fa/gateway/security/audit-checks)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="خودکارسازی: Cron، هوک‌ها، وظایف، نظرسنجی - M3 بتا - 6 حوزه">
    <a id="automation-cron-hooks-tasks-polling" />

    مستندسازی‌شده و قابل استفاده است، اما اثبات سناریو باید تحویل بدون نظارت، تلاش‌های مجدد و مشاهده‌پذیری خرابی را پوشش دهد.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 2%</span><span>کیفیت بتا - 72%</span><span>کامل‌بودن بتا - 79%</span><span><span className="maturity-lts maturity-lts-none">هیچ‌کدام</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">کارهای Cron</span>
          <span>15 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [کارهای Cron](/fa/automation/cron-jobs)، [Cron](/fa/cli/cron)، [پروتکل](/fa/gateway/protocol)، [وظایف](/fa/automation/tasks)، [Discord](/fa/channels/discord)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">ورود رویدادها</span>
          <span>15 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Telegram](/fa/channels/telegram)، [Zalo](/fa/channels/zalo)، [عیب‌یابی](/fa/channels/troubleshooting)، [iMessage از BlueBubbles](/fa/channels/imessage-from-bluebubbles)، [یکپارچه‌سازی Gmail Pub/Sub](/fa/automation/cron-jobs#gmail-pubsub-integration)، [Gmail Pub/Sub](/fa/automation/cron-jobs)، [Webhookها](/fa/cli/webhooks)، [Webhookها](/fa/automation/cron-jobs#webhooks)، [Webhook](/fa/automation/cron-jobs)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">هوک‌های خودکارسازی</span>
          <span>11 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [هوک‌ها](/fa/automation/hooks)، [هوک‌ها](/fa/cli/hooks)، [هوک‌ها](/fa/plugins/hooks)، [درخواست‌های مجوز Plugin](/fa/plugins/plugin-permission-requests)، [زیرمسیرهای SDK](/fa/plugins/sdk-subpaths)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">وظایف و جریان‌های پس‌زمینه</span>
          <span>10 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [وظایف](/fa/automation/tasks)، [نمایه](/fa/automation/index)، [وظایف](/fa/cli/tasks)، [TaskFlow](/fa/automation/taskflow)، [زمان اجرای SDK](/fa/plugins/sdk-runtime)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">Heartbeat</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>14%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "14%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [نمایه](/fa/automation/index)، [Heartbeat](/fa/gateway/heartbeat)، [تعهدات](/fa/concepts/commitments)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">کنترل‌های نظرسنجی</span>
          <span>10 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [نظرسنجی](/fa/cli/message)، [پیام](/fa/cli/message)، [Telegram](/fa/channels/telegram)، [Microsoft Teams](/fa/channels/msteams)، [فرایند پس‌زمینه](/fa/gateway/background-process)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="درک و تولید رسانه - M2 آلفا - 6 حوزه">
    <a id="media-understanding-and-media-generation" />

    سطح گسترده‌ای از قابلیت‌ها وجود دارد، اما تفاوت میان ارائه‌دهندگان، محدودیت‌های فایل و هم‌ترازی Node/برنامه باعث می‌شود هنوز پایدار نباشد.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 2%</span><span>کیفیت آلفا - 64%</span><span>کامل‌بودن آلفا - 68%</span><span><span className="maturity-lts maturity-lts-none">هیچ‌کدام</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">دریافت و دسترسی به رسانه</span>
          <span>8 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>61%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "61%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [نمای کلی رسانه](/fa/tools/media-overview)، [درک رسانه](/fa/nodes/media-understanding)، [عملیات امن فایل](/fa/gateway/security/secure-file-operations)، [PDF](/fa/tools/pdf)، [تولید تصویر](/fa/tools/image-generation)، [QR](/fa/cli/qr)، [LINE](/fa/channels/line)، [WhatsApp](/fa/channels/whatsapp)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">مدیریت رسانه در کانال</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>61%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "61%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [تصاویر](/fa/nodes/images)، [نمای کلی رسانه](/fa/tools/media-overview)، [Discord](/fa/channels/discord)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">پیکربندی رسانه</span>
          <span>1 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>61%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "61%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [نمای کلی رسانه](/fa/tools/media-overview)، [تولید تصویر](/fa/tools/image-generation)، [مانیفست](/fa/plugins/manifest)، [مهار Codex](/fa/plugins/codex-harness)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">تحویل تبدیل متن به گفتار</span>
          <span>2 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>61%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "61%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [تبدیل متن به گفتار](/fa/tools/tts)، [نمای کلی رسانه](/fa/tools/media-overview)، [Discord](/fa/channels/discord)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">درک رسانه</span>
          <span>12 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>7%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "7%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>69%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "69%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>69%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "69%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [صدا](/fa/nodes/audio)، [درک رسانه](/fa/nodes/media-understanding)، [نمای کلی رسانه](/fa/tools/media-overview)، [WhatsApp](/fa/channels/whatsapp)، [تصاویر](/fa/nodes/images)، [استنتاج](/fa/cli/infer)، [PDF](/fa/tools/pdf)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">تولید رسانه</span>
          <span>17 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>5%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "5%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>69%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "69%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>69%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "69%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [تولید تصویر](/fa/tools/image-generation)، [نمای کلی رسانه](/fa/tools/media-overview)، [Skills](/fa/tools/skills)، [تولید موسیقی](/fa/tools/music-generation)، [تولید ویدئو](/fa/tools/video-generation)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="صدا و گفت‌وگوی بی‌درنگ - آلفای M2 - 6 حوزه">
    <a id="voice-and-realtime-talk" />

    پیاده‌سازی‌های متعددی در رابط کاربری کنترل، برنامه‌ها و ارائه‌دهندگان وجود دارد. پیش از نسخه بتا، به کارنامه‌های تأخیر، حالت‌های خرابی و راه‌اندازی نیاز است.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 0%</span><span>کیفیت آلفا - 61%</span><span>کامل‌بودن آلفا - 68%</span><span><span className="maturity-lts maturity-lts-none">هیچ‌کدام</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">ارائه‌دهندگان گفت‌وگو</span>
          <span>7 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>61%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "61%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [OpenAI](/fa/providers/openai)، [Google](/fa/providers/google)، [Pluginهای ارائه‌دهنده SDK](/fa/plugins/sdk-provider-plugins)، [گفت‌وگو](/fa/nodes/talk)، [رابط کاربری کنترل](/fa/web/control-ui)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">جلسه‌های گفت‌وگوی بی‌درنگ</span>
          <span>11 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>61%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "61%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [گفت‌وگو](/fa/nodes/talk)، [رابط کاربری کنترل](/fa/web/control-ui)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">گفتار و رونویسی</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>61%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "61%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [گفت‌وگو](/fa/nodes/talk)، [OpenAI](/fa/providers/openai)، [Google](/fa/providers/google)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">گفت‌وگو در برنامه بومی</span>
          <span>4 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>61%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "61%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [گفت‌وگو](/fa/nodes/talk)، [بیدارباش صوتی](/fa/platforms/mac/voicewake)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">بیدارباش صوتی و مسیریابی</span>
          <span>4 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>61%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "61%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [بیدارباش صوتی](/fa/nodes/voicewake)، [بیدارباش صوتی](/fa/platforms/mac/voicewake)، [هم‌پوشان صوتی](/fa/platforms/mac/voice-overlay)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">مشاهده‌پذیری گفت‌وگو</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>61%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "61%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [رابط کاربری کنترل](/fa/web/control-ui)، [هم‌پوشان صوتی](/fa/platforms/mac/voice-overlay)، [گفت‌وگو](/fa/nodes/talk)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="TUI - M2 آلفا - 5 حوزه">
    <a id="tui" />

    در مستندات و کد منبع وجود دارد، اما به‌عنوان گردش‌کار اصلی کاربر کمتر دیده می‌شود. به تعریف صریح سناریو نیاز دارد.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 0%</span><span>کیفیت آلفا - 59%</span><span>کامل‌بودن آلفا - 66%</span><span><span className="maturity-lts maturity-lts-none">هیچ‌کدام</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">حالت‌های زمان اجرا</span>
          <span>14 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>59%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "59%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [TUI](/fa/cli/tui)، [TUI](/fa/web/tui)، [نمایه](/fa/cli/index)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">ورودی و فرمان‌ها</span>
          <span>8 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>59%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "59%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [TUI](/fa/web/tui)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">مدیریت جلسه</span>
          <span>3 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>59%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "59%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [TUI](/fa/web/tui)، [جلسه‌ها](/fa/cli/sessions)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">اجرای پوستهٔ محلی</span>
          <span>4 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>59%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "59%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [TUI](/fa/web/tui)، [TUI](/fa/cli/tui)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">ایمنی رندر و خروجی</span>
          <span>4 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>59%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "59%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [TUI](/fa/web/tui)، [QR](/fa/cli/qr)، [گزارش‌ها](/fa/cli/logs)، [تکمیل](/fa/cli/completion)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="ClawHub - آلفای M2 - 4 حوزه">
    <a id="clawhub" />

    مستندات عمومی و مفهوم اکوسیستم وجود دارند. به کارت‌های امتیاز نصب، اعتماد، به‌روزرسانی، بازگردانی و سازگاری نیاز دارد.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 0%</span><span>کیفیت آلفا - 58%</span><span>کامل‌بودن آلفا - 62%</span><span><span className="maturity-lts maturity-lts-none">هیچ‌کدام</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">انتشار</span>
          <span>7 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>54%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "54%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>55%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "55%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [انتشار](/fa/clawhub/publishing)، [ایجاد Skills](/fa/tools/creating-skills)، [جامعه](/fa/plugins/community)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">کشف کاتالوگ</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>61%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "61%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Plugin](/fa/tools/plugin)، [Pluginها](/fa/cli/plugins)، [Skills](/fa/cli/skills)، [Skills](/fa/tools/skills)، [جامعه](/fa/plugins/community)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">سازگاری و اعتماد</span>
          <span>12 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>55%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "55%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>56%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "56%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Plugin](/fa/tools/plugin)، [Pluginها](/fa/cli/plugins)، [سازگاری](/fa/plugins/compatibility)، [موجودی Pluginها](/fa/plugins/plugin-inventory)، [انتشار](/fa/clawhub/publishing)، [Skills](/fa/tools/skills)، [پیکربندی Skills](/fa/tools/skills-config)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">چرخهٔ عمر و سلامت Plugin</span>
          <span>26 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>61%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "61%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Plugin](/fa/tools/plugin)، [Pluginها](/fa/cli/plugins)، [Skills](/fa/cli/skills)، [Skills](/fa/tools/skills)، [پروتکل](/fa/gateway/protocol)، [بسته‌ها](/fa/plugins/bundles)، [تفکیک وابستگی‌ها](/fa/plugins/dependency-resolution)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="SDK برنامهٔ OpenClaw - آلفای M2 - 6 حوزه">
    <a id="openclaw-app-sdk" />

    SDK برنامهٔ OpenClaw یک قرارداد متمایز برای برنامه‌های خارجی است که از زمان اجرای Gateway و SDK مربوط به Plugin جداست. امتیازدهی کنونی یک مسیر واقعی `@openclaw/sdk` را نشان می‌دهد که در زمینهٔ بسته‌بندی عمومی، کشف خودکار، تأییدها، ابزارهای کمکی و سازگاری شکاف‌هایی دارد.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 3%</span><span>کیفیت آلفا - 54%</span><span>کامل‌بودن آلفا - 53%</span><span><span className="maturity-lts maturity-lts-none">هیچ‌کدام</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">API کارخواه</span>
          <span>4 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>51%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "51%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>50%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "50%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [SDK ‏OpenClaw](/fa/gateway/external-apps)، [طراحی API ‏SDK ‏OpenClaw](/fa/gateway/external-apps)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">دسترسی به Gateway</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>53%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "53%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>54%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "54%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [SDK ‏OpenClaw](/fa/gateway/external-apps)، [طراحی API ‏SDK ‏OpenClaw](/fa/gateway/external-apps)، [پروتکل](/fa/gateway/protocol)، [نمایه](/fa/gateway/security/index)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">گفت‌وگوهای عامل</span>
          <span>6 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>52%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "52%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>52%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "52%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [SDK ‏OpenClaw](/fa/gateway/external-apps)، [طراحی API ‏SDK ‏OpenClaw](/fa/gateway/external-apps)، [پروتکل](/fa/gateway/protocol)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">رویدادها و تأییدها</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>52%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "52%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>52%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "52%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [SDK ‏OpenClaw](/fa/gateway/external-apps)، [طراحی API ‏SDK ‏OpenClaw](/fa/gateway/external-apps)، [پروتکل](/fa/gateway/protocol)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">ابزارهای کمکی منابع</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>17%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "17%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>62%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "62%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>53%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "53%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [SDK ‏OpenClaw](/fa/gateway/external-apps)، [طراحی API ‏SDK ‏OpenClaw](/fa/gateway/external-apps)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">سازگاری</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>54%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "54%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>55%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "55%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [طراحی API ‏SDK ‏OpenClaw](/fa/gateway/external-apps)، [Typebox](/fa/concepts/typebox)، [پروتکل](/fa/gateway/protocol)

    </div>
      </div>
    </div>

  </Accordion>

</AccordionGroup>

### پلتفرم

<AccordionGroup>
  <Accordion title="میزبان Gateway لینوکس - M4 پایدار - 5 حوزه">
    <a id="linux-gateway-host" />

    استفاده از محیط اجرای Node توصیه می‌شود، سرویس کاربر systemd مستندسازی شده است و راهنمای VPS/کانتینر دامنه گسترده‌ای دارد.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 0%</span><span>کیفیت بتا - 75%</span><span>کامل‌بودن پایدار - 89%</span><span><span className="maturity-lts maturity-lts-partial">جزئی - 4</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">راه‌اندازی و به‌روزرسانی‌های میزبان</span>
          <span>4 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>75%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "75%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>89%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "89%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [نمایه](/fa/install/index)، [به‌روزرسانی](/fa/install/updating)، [لینوکس](/fa/platforms/linux)، [نمایه](/fa/platforms/index)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">محیط اجرای Gateway و کنترل سرویس</span>
          <span>6 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>75%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "75%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>89%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "89%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [نمایه](/fa/gateway/index)، [Gateway](/fa/cli/gateway)، [لینوکس](/fa/platforms/linux)، [VPS](/fa/vps)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">دسترسی از راه دور و امنیت</span>
          <span>6 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>75%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "75%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>89%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "89%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [راه دور](/fa/gateway/remote)، [Tailscale](/fa/gateway/tailscale)، [راهنمای عملیاتی در معرض قرارگیری](/fa/gateway/security/exposure-runbook)، [احراز هویت](/fa/gateway/authentication)، [اسرار](/fa/gateway/secrets)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">عیب‌یابی و تعمیر</span>
          <span>4 قابلیت / با پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>75%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "75%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>89%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "89%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [وضعیت](/fa/cli/status)، [گزارش‌ها](/fa/cli/logs)، [پزشک](/fa/cli/doctor)، [عیب‌یابی](/fa/gateway/diagnostics)، [نمایه](/fa/gateway/index)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">مقصدهای استقرار</span>
          <span>3 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>75%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "75%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>89%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "89%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [VPS](/fa/vps)، [Docker](/fa/install/docker)، [Hetzner](/fa/install/hetzner)، [DigitalOcean](/fa/install/digitalocean)، [Kubernetes](/fa/install/kubernetes)، [Podman](/fa/install/podman)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="میزبان Gateway در macOS - M4 پایدار - 7 حوزه">
    <a id="macos-gateway-host" />

    مسیر سرویس LaunchAgent، حالت‌های محلی/راه‌دور Gateway، نصب CLI و یکپارچه‌سازی برنامه مستند شده‌اند.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 0%</span><span>کیفیت بتا - 74%</span><span>کامل‌بودن پایدار - 88%</span><span><span className="maturity-lts maturity-lts-none">هیچ‌کدام</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">راه‌اندازی CLI</span>
          <span>4 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>74%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "74%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>88%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "88%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [مک‌اواس](/fa/platforms/macos)، [Gateway همراه](/fa/platforms/mac/bundled-gateway)، [نصب‌کننده](/fa/install/installer)، [Node](/fa/install/node)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">یکپارچه‌سازی Gateway محلی</span>
          <span>9 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>74%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "74%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>88%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "88%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [مک‌اواس](/fa/platforms/macos)، [Gateway همراه](/fa/platforms/mac/bundled-gateway)، [راه دور](/fa/platforms/mac/remote)، [نمایه](/fa/gateway/index)، [Gateway](/fa/cli/gateway)، [Bonjour](/fa/gateway/bonjour)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">حالت Gateway راه دور</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>74%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "74%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>88%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "88%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [راه دور](/fa/platforms/mac/remote)، [راه دور](/fa/gateway/remote)، [Tailscale](/fa/gateway/tailscale)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">چرخهٔ عمر سرویس Gateway</span>
          <span>10 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>74%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "74%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>88%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "88%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [مک‌اواس](/fa/platforms/macos)، [Gateway همراه](/fa/platforms/mac/bundled-gateway)، [Gateway](/fa/cli/gateway)، [نمایه](/fa/gateway/index)، [به‌روزرسانی](/fa/cli/update)، [به‌روزرسانی](/fa/install/updating)، [حذف نصب](/fa/install/uninstall)، [عیب‌یابی](/fa/gateway/troubleshooting)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">تشخیص و مشاهده‌پذیری</span>
          <span>4 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>74%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "74%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>88%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "88%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Gateway همراه](/fa/platforms/mac/bundled-gateway)، [مک‌اواس](/fa/platforms/macos)، [Gateway](/fa/cli/gateway)، [پزشک](/fa/gateway/doctor)، [عیب‌یابی](/fa/gateway/troubleshooting)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">مجوزها و قابلیت‌های بومی</span>
          <span>4 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>74%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "74%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>88%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "88%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [مک‌اواس](/fa/platforms/macos)، [راه دور](/fa/platforms/mac/remote)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">پروفایل‌ها و جداسازی</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>74%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "74%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>88%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "88%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [چند Gateway](/fa/gateway/multiple-gateways)، [نمایه](/fa/gateway/index)، [Gateway](/fa/cli/gateway)

    </div>
      </div>
    </div>

  </Accordion>
  <Accordion title="برنامه Android - M4 پایدار - 7 حوزه">
    <a id="android-app" />

    توزیع رسمی در Google Play موجود است، مستندات ساخت و اجرای کد منبع نگهداری می‌شوند و برنامه Android به‌عنوان یک Node همراه عادی برای کاربران مستند شده است.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 0%</span><span>کیفیت پایدار - 80%</span><span>کامل‌بودن پایدار - 80%</span><span><span className="maturity-lts maturity-lts-none">هیچ‌کدام</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">ضبط رسانه</span>
          <span>1 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>80%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "80%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>80%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "80%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Android](/fa/platforms/android)، [دوربین](/fa/nodes/camera)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">گفت‌وگوی موبایل</span>
          <span>1 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>80%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "80%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>80%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "80%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Android](/fa/platforms/android)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">راه‌اندازی اتصال</span>
          <span>1 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>80%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "80%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>80%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "80%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Android](/fa/platforms/android)، [Bonjour](/fa/gateway/bonjour)، [جفت‌سازی](/fa/gateway/pairing)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">توزیع</span>
          <span>3 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>80%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "80%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>80%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "80%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [اندروید](/fa/platforms/android)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">تنظیمات</span>
          <span>1 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>80%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "80%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>80%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "80%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [اندروید](/fa/platforms/android)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">صدا</span>
          <span>1 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>80%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "80%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>80%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "80%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [اندروید](/fa/platforms/android)، [گفت‌وگو](/fa/nodes/talk)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">محیط اجرای دستگاه</span>
          <span>2 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>80%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "80%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>80%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "80%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [اندروید](/fa/platforms/android)، [عیب‌یابی](/fa/nodes/troubleshooting)، [پروتکل](/fa/gateway/protocol)

    </div>
      </div>
    </div>

  </Accordion>
  <Accordion title="برنامه iOS - M4 پایدار - 8 حوزه">
    <a id="ios-app" />

    توزیع رسمی از طریق App Store وجود دارد، ارسال اعلان با پشتیبانی رله مستند شده است و برنامه iOS به‌عنوان یک Node همراه معمول برای کاربران مستند شده است.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 0%</span><span>کیفیت پایدار - 80%</span><span>کامل‌بودن پایدار - 80%</span><span><span className="maturity-lts maturity-lts-none">هیچ‌کدام</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">رسانه و اشتراک‌گذاری</span>
          <span>1 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>80%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "80%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>80%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "80%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [iOS](/fa/platforms/ios)، [دوربین](/fa/nodes/camera)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">بوم و صفحه‌نمایش</span>
          <span>1 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>80%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "80%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>80%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "80%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [iOS](/fa/platforms/ios)، [بوم](/fa/plugins/reference/canvas)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">گفت‌وگو و نشست‌ها</span>
          <span>1 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>80%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "80%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>80%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "80%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [iOS](/fa/platforms/ios)، [گفت‌وگوی وب](/fa/web/webchat)، [پروتکل](/fa/gateway/protocol)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">راه‌اندازی و عیب‌یابی Gateway</span>
          <span>7 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>80%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "80%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>80%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "80%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [iOS](/fa/platforms/ios)، [جفت‌سازی](/fa/channels/pairing)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">توزیع</span>
          <span>1 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>80%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "80%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>80%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "80%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [iOS](/fa/platforms/ios)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">فرمان‌های دستگاه</span>
          <span>2 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>80%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "80%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>80%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "80%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [iOS](/fa/platforms/ios)، [پروتکل](/fa/gateway/protocol)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">اعلان‌ها و اجرای پس‌زمینه</span>
          <span>1 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>80%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "80%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>80%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "80%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [iOS](/fa/platforms/ios)، [پیکربندی](/fa/gateway/configuration)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">صدا</span>
          <span>1 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>80%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "80%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>80%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "80%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [iOS](/fa/platforms/ios)، [گفتار](/fa/nodes/talk)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="میزبانی Docker و Podman - بتای M3 - 4 حوزه">
    <a id="docker-and-podman-hosting" />

    مستندات نصب موجودند و این روش‌ها از مسیرهای رایج استقرار هستند. پس از آنکه آزمون‌های دود دوره‌ای انتشار، رفتار ارتقا و ولوم را ثبت کردند، آن را ارتقا دهید.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 7%</span><span>کیفیت بتا - 71%</span><span>کامل‌بودن بتا - 79%</span><span><span className="maturity-lts maturity-lts-none">هیچ‌کدام</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">راه‌اندازی کانتینر</span>
          <span>6 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Docker](/fa/install/docker)، [Podman](/fa/install/podman)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">عملیات کانتینر</span>
          <span>11 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Podman](/fa/install/podman)، [محیط اجرای ماشین مجازی Docker](/fa/install/docker-vm-runtime)، [Docker](/fa/install/docker)، [Hetzner](/fa/install/hetzner)، [Hostinger](/fa/install/hostinger)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">انتشار و اعتبارسنجی ایمیج</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>29%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "29%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Docker](/fa/install/docker)، [محیط اجرای ماشین مجازی Docker](/fa/install/docker-vm-runtime)، [اعتبارسنجی کامل انتشار](/fa/reference/full-release-validation)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">محیط ایزوله و ابزارهای عامل</span>
          <span>3 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Docker](/fa/install/docker)، [محیط اجرای ماشین مجازی Docker](/fa/install/docker-vm-runtime)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="ویندوز از طریق WSL2 - بتای M3 - 6 حوزه">
    <a id="windows-via-wsl2" />

    مسیر پیشنهادی ویندوز همراه با راهنمای systemd/سرویس کاربر و مستندات زنجیره راه‌اندازی. پس از ثبت مکرر کارنامه‌های نصب/به‌روزرسانی، ارتقا یابد.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 6%</span><span>کیفیت آلفا - 69%</span><span>کامل‌بودن بتا - 79%</span><span><span className="maturity-lts maturity-lts-partial">جزئی - 5</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">راه‌اندازی WSL</span>
          <span>6 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>67%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "67%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [ویندوز](/fa/platforms/windows)، [شروع به کار](/fa/start/getting-started)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">CLI</span>
          <span>8 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>67%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "67%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [ویندوز](/fa/platforms/windows)، [شروع به کار](/fa/start/getting-started)، [به‌روزرسانی](/fa/install/updating)، [راه‌اندازی اولیه](/fa/cli/onboard)، [عیب‌یابی](/fa/cli/doctor)، [وضعیت](/fa/cli/status)، [گزارش‌ها](/fa/cli/logs)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">چرخه عمر سرویس Gateway</span>
          <span>10 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>67%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "67%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [ویندوز](/fa/platforms/windows)، [نمایه](/fa/gateway/index)، [عیب‌یابی](/fa/gateway/doctor)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">دسترسی و در معرض قرارگیری Gateway</span>
          <span>11 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>67%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "67%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [احراز هویت](/fa/gateway/authentication)، [اسرار](/fa/gateway/secrets)، [راه دور](/fa/gateway/remote)، [راهنمای عملیاتی در معرض قرارگیری](/fa/gateway/security/exposure-runbook)، [ویندوز](/fa/platforms/windows)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">عیب‌یابی و تعمیر</span>
          <span>6 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>38%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "38%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Windows](/fa/platforms/windows)، [وضعیت](/fa/cli/status)، [گزارش‌ها](/fa/cli/logs)، [پزشک](/fa/cli/doctor)، [پزشک](/fa/gateway/doctor)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">مرورگر و رابط کاربری کنترل</span>
          <span>6 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>67%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "67%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [عیب‌یابی CDP راه‌دور مرورگر در WSL2 ویندوز](/fa/tools/browser-wsl2-windows-remote-cdp-troubleshooting)، [مرورگر](/fa/tools/browser)، [رابط کاربری کنترل](/fa/web/control-ui)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="Raspberry Pi و دستگاه‌های کوچک Linux - بتای M3 - 4 حوزه">
    <a id="raspberry-pi-and-small-linux-devices" />

    مستندات پلتفرم موجود است و مسیر Gateway مبتنی بر Linux است. برای ارتقا به سطح بالاتر، به مدرک آزمون دود انتشار مختص سخت‌افزار نیاز دارد.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 0%</span><span>کیفیت آلفا - 67%</span><span>کامل‌بودن بتا - 79%</span><span><span className="maturity-lts maturity-lts-none">هیچ‌کدام</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">راه‌اندازی و سازگاری</span>
          <span>12 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>67%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "67%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Raspberry Pi](/fa/install/raspberry-pi)، [نمایه](/fa/install/index)، [پرسش‌های متداول اجرای نخست](/fa/help/faq-first-run)، [پرسش‌های متداول](/fa/help/faq)، [Linux](/fa/platforms/linux)، [نصب‌کننده](/fa/install/installer)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">دسترسی راه‌دور و احراز هویت</span>
          <span>9 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>67%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "67%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Raspberry Pi](/fa/install/raspberry-pi)، [احراز هویت](/fa/gateway/authentication)، [اسرار](/fa/gateway/secrets)، [جفت‌سازی](/fa/gateway/pairing)، [دستگاه‌ها](/fa/cli/devices)، [راه‌دور](/fa/gateway/remote)، [Tailscale](/fa/gateway/tailscale)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">زمان اجرای Gateway</span>
          <span>10 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>67%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "67%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [نمایه](/fa/gateway/index)، [Gateway](/fa/cli/gateway)، [Raspberry Pi](/fa/install/raspberry-pi)، [Linux](/fa/platforms/linux)، [VPS](/fa/vps)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">عملکرد و عیب‌یابی</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>67%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "67%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Raspberry Pi](/fa/install/raspberry-pi)، [Linux](/fa/platforms/linux)، [سلامت](/fa/gateway/health)، [عیب‌یابی](/fa/gateway/diagnostics)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="برنامه همراه macOS - بتای M3 - 8 حوزه">
    <a id="macos-companion-app" />

    برنامه‌ای غنی برای نوار منو، مجوزها، حالت Node، Canvas، بیدارباش صوتی، WebChat و حالت راه‌دور موجود است. تغییرات آن هنوز آن‌قدر سریع است که از سطح پایدار اجتناب شود.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 0%</span><span>کیفیت آلفا - 66%</span><span>کامل‌بودن بتا - 78%</span><span><span className="maturity-lts maturity-lts-none">هیچ‌کدام</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">Canvas</span>
          <span>4 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Canvas](/fa/platforms/mac/canvas)، [macOS](/fa/platforms/macos)، [WebChat](/fa/web/webchat)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">راه‌اندازی محلی</span>
          <span>7 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Gateway همراه](/fa/platforms/mac/bundled-gateway)، [macOS](/fa/platforms/macos)، [فرایند فرزند](/fa/platforms/mac/child-process)، [راه‌اندازی توسعه](/fa/platforms/mac/dev-setup)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">وضعیت و تنظیمات</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [نوار منو](/fa/platforms/mac/menu-bar)، [آیکون](/fa/platforms/mac/icon)، [macOS](/fa/platforms/macos)، [سلامت](/fa/platforms/mac/health)، [ثبت گزارش](/fa/platforms/mac/logging)، [راه دور](/fa/platforms/mac/remote)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">قابلیت‌های بومی</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [macOS](/fa/platforms/macos)، [XPC](/fa/platforms/mac/xpc)، [مجوزها](/fa/platforms/mac/permissions)، [امضاکردن](/fa/platforms/mac/signing)، [Peekaboo](/fa/platforms/mac/peekaboo)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">اتصال‌های راه دور</span>
          <span>3 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [راه دور](/fa/platforms/mac/remote)، [macOS](/fa/platforms/macos)، [راه دور](/fa/gateway/remote)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">صدا و مکالمه</span>
          <span>3 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [بیدارباش صوتی](/fa/platforms/mac/voicewake)، [پوشش صوتی](/fa/platforms/mac/voice-overlay)، [مکالمه](/fa/nodes/talk)، [macOS](/fa/platforms/macos)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">WebChat</span>
          <span>3 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Webchat](/fa/platforms/mac/webchat)، [Macos](/fa/platforms/macos)، [Webchat](/fa/web/webchat)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">WebChat راه‌دور</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Webchat](/fa/platforms/mac/webchat)، [راه‌دور](/fa/gateway/remote)، [راه‌دور](/fa/platforms/mac/remote)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="Windows بومی - آلفای M2 - 4 حوزه">
    <a id="native-windows" />

    جریان‌های اصلی CLI/Gateway کار می‌کنند، اما مستندات همچنان برای بهره‌مندی از تجربهٔ کامل WSL2 را توصیه می‌کنند و ملاحظات نسخهٔ بومی را برمی‌شمارند.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 0%</span><span>کیفیت آلفا - 58%</span><span>کامل‌بودن آلفا - 66%</span><span><span className="maturity-lts maturity-lts-partial">جزئی - 1</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">CLI</span>
          <span>9 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>54%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "54%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>64%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "64%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [نمایه](/fa/install/index)، [نصب‌کننده](/fa/install/installer)، [Windows](/fa/platforms/windows)، [شروع به کار](/fa/start/getting-started)، [راه‌اندازی اولیه](/fa/cli/onboard)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">مدیریت Gateway</span>
          <span>11 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>59%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "59%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Windows](/fa/platforms/windows)، [نمایه](/fa/gateway/index)، [Gateway](/fa/cli/gateway)، [Doctor](/fa/cli/doctor)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">شبکه</span>
          <span>4 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>59%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "59%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Windows](/fa/platforms/windows)، [نمایه](/fa/gateway/index)، [Gateway](/fa/cli/gateway)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">به‌روزرسانی‌ها</span>
          <span>4 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>59%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "59%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [به‌روزرسانی](/fa/install/updating)، [CI](/fa/ci)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="میزبانی Kubernetes - M2 آلفا - 4 حوزه">
    <a id="kubernetes-hosting" />

    میزبانی Kubernetes یک مسیر مجزای استقرار کلاستر مبتنی بر Kustomize است. امتیازدهی کنونی وجود یک مسیر واقعی و حداقلی برای استقرار را نشان می‌دهد که در زمینه‌های CI ویژه Kubernetes، بسته‌بندی ingress/TLS/NetworkPolicy، پشتیبان‌گیری/بازیابی و مقاوم‌سازی ارائه در محیط عملیاتی کاستی‌هایی دارد.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 0%</span><span>کیفیت آلفا - 55%</span><span>کامل‌بودن آلفا - 61%</span><span><span className="maturity-lts maturity-lts-none">هیچ‌کدام</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">راه‌اندازی استقرار</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>55%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "55%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>61%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "61%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Kubernetes](/fa/install/kubernetes)، [نمایه](/fa/install/index)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">پیکربندی و اسرار</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>55%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "55%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>61%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "61%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Kubernetes](/fa/install/kubernetes)، [اسرار](/fa/gateway/secrets)، [محیط](/fa/help/environment)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">دسترسی و ارائه</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>55%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "55%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>61%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "61%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Kubernetes](/fa/install/kubernetes)، [احراز هویت](/fa/gateway/authentication)، [دسترسی از راه دور](/fa/gateway/remote)، [راهنمای عملیاتی ارائه](/fa/gateway/security/exposure-runbook)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">چرخهٔ عمر کلاستر</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>55%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "55%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>61%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "61%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Kubernetes](/fa/install/kubernetes)، [نمایه](/fa/gateway/index)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="مسیر نصب Nix - M1 آزمایشی - 5 حوزه">
    <a id="nix-install-path" />

    جریان نصب اختیاری است. پیش از ارتقا به آلفا/بتا، به تعهد پشتیبانی روشن‌تری نیاز دارد.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 0%</span><span>کیفیت آزمایشی - 41%</span><span>کامل‌بودن آزمایشی - 44%</span><span><span className="maturity-lts maturity-lts-none">هیچ‌کدام</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">تحویل نصب</span>
          <span>4 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>41%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "41%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>44%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "44%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Nix](/fa/install/nix)، [فهرست](/fa/install/index)، [دایرکتوری مستندات](/fa/start/docs-directory)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">چرخهٔ حیات Plugin</span>
          <span>4 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>41%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "41%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>44%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "44%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [مدیریت Pluginها](/fa/plugins/manage-plugins)، [Plugin](/fa/tools/plugin)، [Nix](/fa/install/nix)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">فعال‌سازی و تجربهٔ کاربری برنامه</span>
          <span>7 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>41%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "41%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>44%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "44%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Nix](/fa/install/nix)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">پیکربندی و وضعیت</span>
          <span>7 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>41%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "41%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>44%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "44%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Nix](/fa/install/nix)، [راه‌اندازی](/fa/cli/setup)، [محیط](/fa/help/environment)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">زمان اجرای سرویس و محافظ‌ها</span>
          <span>8 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>41%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "41%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>44%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "44%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Nix](/fa/install/nix)، [راه‌اندازی](/fa/cli/setup)، [عیب‌یابی](/fa/cli/doctor)، [به‌روزرسانی](/fa/cli/update)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="سطوح همراه watchOS - M1 آزمایشی - 5 حوزه">
    <a id="watchos-companion-surfaces" />

    منبع دارای سطوح برنامه/افزونهٔ Watch است؛ مستندات عمومی هنوز این را به‌عنوان قابلیتی برای کاربر ارائه نمی‌کنند.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 0%</span><span>کیفیت آزمایشی - 41%</span><span>کامل‌بودن آزمایشی - 44%</span><span><span className="maturity-lts maturity-lts-none">هیچ‌کدام</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">تحویل و بازیابی</span>
          <span>7 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>41%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "41%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>44%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "44%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [iOS](/fa/platforms/ios)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">تأییدهای اجرا</span>
          <span>3 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>41%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "41%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>44%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "44%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [تأییدهای اجرا](/fa/tools/exec-approvals)، [iOS](/fa/platforms/ios)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">توزیع و پشتیبانی</span>
          <span>6 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>41%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "41%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>44%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "44%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [iOS](/fa/platforms/ios)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">اعلان‌ها و پاسخ‌ها</span>
          <span>7 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>41%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "41%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>44%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "44%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [iOS](/fa/platforms/ios)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">رابط کاربری برنامه ساعت</span>
          <span>3 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>41%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "41%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>44%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "44%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [iOS](/fa/platforms/ios)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="برنامه همراه Linux - M0 برنامه‌ریزی‌شده - 5 حوزه">
    <a id="linux-companion-app" />

    مستندات می‌گویند برنامه‌های همراه بومی Linux برنامه‌ریزی شده‌اند؛ در حال حاضر Gateway مسیر پشتیبانی‌شده برای Linux است.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 0%</span><span>کیفیت آزمایشی - 19%</span><span>کامل‌بودن آزمایشی - 21%</span><span><span className="maturity-lts maturity-lts-none">هیچ‌کدام</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">توزیع برنامه</span>
          <span>3 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>19%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "19%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>21%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "21%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Linux](/fa/platforms/linux)، [نمایه](/fa/platforms/index)، [نمایه](/fa/install/index)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">اتصال Gateway</span>
          <span>4 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>19%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "19%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>21%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "21%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Linux](/fa/platforms/linux)، [نمایه](/fa/gateway/index)، [جفت‌سازی](/fa/gateway/pairing)، [راه دور](/fa/gateway/remote)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">گفت‌وگو و نشست‌ها</span>
          <span>3 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>19%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "19%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>21%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "21%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Linux](/fa/platforms/linux)، [پروتکل](/fa/gateway/protocol)، [گفت‌وگوی وب](/fa/web/webchat)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">قابلیت‌های دسکتاپ</span>
          <span>9 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>19%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "19%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>21%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "21%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Linux](/fa/platforms/linux)، [تأییدهای اجرا](/fa/tools/exec-approvals)، [اسرار](/fa/gateway/secrets)، [نمایه](/fa/nodes/index)، [اجرا](/fa/tools/exec)، [گفت‌وگو](/fa/nodes/talk)، [دوربین](/fa/nodes/camera)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">وضعیت و عیب‌یابی</span>
          <span>7 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>19%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "19%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>21%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "21%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Linux](/fa/platforms/linux)، [OpenClaw](/fa/start/openclaw)، [Doctor](/fa/gateway/doctor)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="برنامه همراه بومی Windows - M0 برنامه‌ریزی‌شده - 5 حوزه">
    <a id="native-windows-companion-app" />

    فقط برنامه‌ریزی شده است.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 0%</span><span>کیفیت آزمایشی - 19%</span><span>کامل‌بودن آزمایشی - 21%</span><span><span className="maturity-lts maturity-lts-none">هیچ‌کدام</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">نصب و به‌روزرسانی‌ها</span>
          <span>4 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>19%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "19%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>21%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "21%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Windows](/fa/platforms/windows)، [نمایه](/fa/install/index)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">اتصال Gateway</span>
          <span>3 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>19%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "19%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>21%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "21%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Windows](/fa/platforms/windows)، [نمایه](/fa/gateway/index)، [جفت‌سازی](/fa/gateway/pairing)، [راه دور](/fa/gateway/remote)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">نشست‌های گفت‌وگو</span>
          <span>2 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>19%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "19%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>21%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "21%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Windows](/fa/platforms/windows)، [پروتکل](/fa/gateway/protocol)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">وضعیت و تعمیر</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>19%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "19%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>21%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "21%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Windows](/fa/platforms/windows)، [عیب‌یاب](/fa/gateway/doctor)، [نمایه](/fa/gateway/index)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">ابزارها و مجوزهای دسکتاپ</span>
          <span>10 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>19%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "19%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>21%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "21%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Windows](/fa/platforms/windows)، [نمایه](/fa/nodes/index)، [اجرا](/fa/tools/exec)، [تأییدیه‌های اجرا](/fa/tools/exec-approvals)، [نمایه](/fa/gateway/security/index)

    </div>
      </div>
    </div>

  </Accordion>

</AccordionGroup>

### کانال

<AccordionGroup>
  <Accordion title="Discord - M4 پایدار - 6 حوزه">
    <a id="discord" />

    مستندات عمیق و پوشش گستردهٔ قابلیت‌ها. مسیرهای صوتی/واگذاری باید جداگانه در سطح بتا/آلفا امتیازدهی شوند.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 0%</span><span>کیفیت بتا - 73%</span><span>کامل‌بودن پایدار - 87%</span><span><span className="maturity-lts maturity-lts-partial">جزئی - 4</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">راه‌اندازی و عملیات کانال</span>
          <span>10 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>73%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "73%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>87%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "87%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Discord](/fa/channels/discord), [Discord](/fa/plugins/reference/discord), [Fly](/fa/install/fly), [دستورهای اسلش](/fa/tools/slash-commands), [سلامت](/fa/gateway/health), [کانال‌ها](/fa/cli/channels), [پیکربندی کانال‌ها](/fa/gateway/config-channels)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">دسترسی و هویت</span>
          <span>6 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>73%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "73%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>87%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "87%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Discord](/fa/channels/discord), [جفت‌سازی](/fa/channels/pairing), [گروه‌های دسترسی](/fa/channels/access-groups), [گروه‌ها](/fa/channels/groups)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">مسیریابی و تحویل مکالمه</span>
          <span>12 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>73%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "73%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>87%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "87%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Discord](/fa/channels/discord)، [مسیریابی کانال](/fa/channels/channel-routing)، [گروه‌ها](/fa/channels/groups)، [گروه‌های دسترسی](/fa/channels/access-groups)، [عامل‌های ACP](/fa/tools/acp-agents)، [زیرعامل‌ها](/fa/tools/subagents)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">رسانه و محتوای غنی</span>
          <span>1 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>73%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "73%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>87%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "87%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Discord](/fa/channels/discord)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">کنترل‌ها و تأییدهای بومی</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>73%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "73%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>87%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "87%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Discord](/fa/channels/discord)، [دستورهای اسلش](/fa/tools/slash-commands)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">صدا و تماس‌های بی‌درنگ</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>73%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "73%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-stable"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-stable">پایدار</span><span>87%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "87%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Discord](/fa/channels/discord)، [OpenAI](/fa/providers/openai)، [Elevenlabs](/fa/providers/elevenlabs)، [اتوماسیون سرتاسری QA](/fa/concepts/qa-e2e-automation)، [پیکربندی کانال‌ها](/fa/gateway/config-channels)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="Telegram - M3 بتا - 5 حوزه">
    <a id="telegram" />

    کانال اصلی برای استفاده منظم به‌اندازه کافی بالغ است، اما تجربه کاربری بسیار متغیر و موارد مرزی رسانه‌ای به اثبات مکرر سناریوها نیاز دارند.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 0%</span><span>کیفیت آلفا - 68%</span><span>کامل‌بودن بتا - 78%</span><span><span className="maturity-lts maturity-lts-full">کامل - 5</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">راه‌اندازی و عملیات کانال</span>
          <span>10 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Telegram](/fa/channels/telegram)، [پیکربندی کانال‌ها](/fa/gateway/config-channels)، [کانال‌ها](/fa/cli/channels)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">دسترسی و هویت</span>
          <span>10 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Telegram](/fa/channels/telegram)، [جفت‌سازی](/fa/channels/pairing)، [گروه‌های دسترسی](/fa/channels/access-groups)، [گروه‌ها](/fa/channels/groups)، [چندعاملی](/fa/concepts/multi-agent)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">مسیریابی و تحویل مکالمه</span>
          <span>1 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Telegram](/fa/channels/telegram)، [گروه‌ها](/fa/channels/groups)، [چندعاملی](/fa/concepts/multi-agent)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">رسانه و محتوای غنی</span>
          <span>1 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Telegram](/fa/channels/telegram)، [موقعیت مکانی](/fa/channels/location)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">کنترل‌ها و تأییدهای بومی</span>
          <span>9 قابلیت / پشتیبانی‌شده در LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>77%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "77%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Telegram](/fa/channels/telegram)، [تأییدهای اجرا](/fa/tools/exec-approvals)، [واکنش‌ها](/fa/tools/reactions)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="Slack - بتای M3 - ۵ حوزه">
    <a id="slack" />

    مستندات سطح‌اول کانال و سطح مسیریابی. به کارت‌های امتیاز سناریوهای نصب و مدیریت فضای کاری نیاز دارد.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 0%</span><span>کیفیت آلفا - 66%</span><span>کامل‌بودن بتا - 78%</span><span><span className="maturity-lts maturity-lts-full">کامل - 5</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">راه‌اندازی و عملیات کانال</span>
          <span>10 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Slack](/fa/channels/slack)، [Slack](/fa/plugins/reference/slack)، [اسرار](/fa/gateway/secrets)، [خودکارسازی QA سرتاسری](/fa/concepts/qa-e2e-automation)، [عیب‌یابی](/fa/channels/troubleshooting)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">دسترسی و هویت</span>
          <span>1 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Slack](/fa/channels/slack)، [جفت‌سازی](/fa/channels/pairing)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">مسیریابی و تحویل مکالمه</span>
          <span>5 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Slack](/fa/channels/slack)، [محافظت در برابر حلقه ربات](/fa/channels/bot-loop-protection)، [جفت‌سازی](/fa/channels/pairing)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">رسانه و محتوای غنی</span>
          <span>1 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Slack](/fa/channels/slack)، [خودکارسازی QA سرتاسری](/fa/concepts/qa-e2e-automation)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">کنترل‌ها و تأییدهای بومی</span>
          <span>8 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Slack](/fa/channels/slack)، [دستورهای اسلش](/fa/tools/slash-commands)، [تأییدهای اجرا](/fa/tools/exec-approvals)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="iMessage و BlueBubbles - بتای M3 - 5 حوزه">
    <a id="imessage-and-bluebubbles" />

    iMessage پشتیبانی‌شده از طریق imsg روی میزبان macOS Messages که به حساب وارد شده است اجرا می‌شود؛ پیکربندی‌های قدیمی BlueBubbles نیازمند مهاجرت‌اند. مجوزهای macOS، پوشش SSH، API خصوصی/SIP و ملاحظات مهاجرت را آشکار نگه دارید.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 0%</span><span>کیفیت آلفا - 66%</span><span>کامل‌بودن بتا - 78%</span><span><span className="maturity-lts maturity-lts-none">هیچ‌کدام</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">راه‌اندازی و عملیات کانال</span>
          <span>11 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [iMessage در BlueBubbles](/fa/announcements/bluebubbles-imessage)، [مهاجرت iMessage از BlueBubbles](/fa/channels/imessage-from-bluebubbles)، [پیکربندی کانال‌ها](/fa/gateway/config-channels)، [iMessage](/fa/channels/imessage)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">دسترسی و هویت</span>
          <span>6 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [iMessage](/fa/channels/imessage)، [مهاجرت iMessage از BlueBubbles](/fa/channels/imessage-from-bluebubbles)، [پیکربندی کانال‌ها](/fa/gateway/config-channels)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">مسیریابی و تحویل مکالمه</span>
          <span>4 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [iMessage](/fa/channels/imessage)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">رسانه و محتوای غنی</span>
          <span>7 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [iMessage](/fa/channels/imessage)، [iMessage از BlueBubbles](/fa/channels/imessage-from-bluebubbles)، [پیکربندی کانال‌ها](/fa/gateway/config-channels)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">کنترل‌ها و تأییدهای بومی</span>
          <span>3 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [iMessage](/fa/channels/imessage)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="WhatsApp - M3 بتا - 5 حوزه">
    <a id="whatsapp" />

    مسیر اصلی مهم و مستند است؛ ناپایداری بالادستی Baileys/نشست، آن را پایین‌تر از سطح پایدار نگه می‌دارد.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 0%</span><span>کیفیت آلفا - 66%</span><span>کامل‌بودن بتا - 78%</span><span><span className="maturity-lts maturity-lts-none">هیچ‌کدام</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">راه‌اندازی و عملیات کانال</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [WhatsApp](/fa/channels/whatsapp)، [پیکربندی کانال‌ها](/fa/gateway/config-channels)، [WhatsApp](/fa/plugins/reference/whatsapp)، [خودکارسازی QA سرتاسری](/fa/concepts/qa-e2e-automation)، [Doctor](/fa/gateway/doctor)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">دسترسی و هویت</span>
          <span>7 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [WhatsApp](/fa/channels/whatsapp)، [کانال‌های پیکربندی](/fa/gateway/config-channels)، [خودکارسازی سرتاسری تضمین کیفیت](/fa/concepts/qa-e2e-automation)، [جفت‌سازی](/fa/channels/pairing)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">مسیریابی و تحویل مکالمه</span>
          <span>4 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [WhatsApp](/fa/channels/whatsapp)، [پیام‌های گروهی](/fa/channels/group-messages)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">رسانه و محتوای غنی</span>
          <span>2 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [WhatsApp](/fa/channels/whatsapp)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">کنترل‌ها و تأییدهای بومی</span>
          <span>2 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [WhatsApp](/fa/channels/whatsapp)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="Matrix - M2 آلفا - 6 حوزه">
    <a id="matrix" />

    از طریق Plugin همراه پشتیبانی می‌شود. به کارت‌های امتیازدهی پل، احراز هویت و چرخه عمر اتاق نیاز دارد.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 0%</span><span>کیفیت آلفا - 60%</span><span>کامل‌بودن آلفا - 67%</span><span><span className="maturity-lts maturity-lts-none">هیچ‌کدام</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">راه‌اندازی و عملیات کانال</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>60%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "60%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>67%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "67%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Matrix](/fa/channels/matrix)، [مهاجرت Matrix](/fa/channels/matrix-migration)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">دسترسی و هویت</span>
          <span>7 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>60%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "60%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>67%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "67%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Matrix](/fa/channels/matrix)، [گروه‌ها](/fa/channels/groups)، [محافظت در برابر حلقه ربات](/fa/channels/bot-loop-protection)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">مسیریابی و تحویل مکالمه</span>
          <span>1 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>60%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "60%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>67%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "67%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Matrix](/fa/channels/matrix)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">رسانه و محتوای غنی</span>
          <span>1 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>60%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "60%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>67%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "67%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Matrix](/fa/channels/matrix)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">کنترل‌ها و تأییدهای بومی</span>
          <span>6 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>60%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "60%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>67%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "67%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Matrix](/fa/channels/matrix)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">رمزنگاری و راستی‌آزمایی</span>
          <span>3 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>60%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "60%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>67%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "67%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Matrix](/fa/channels/matrix)، [مهاجرت Matrix](/fa/channels/matrix-migration)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="Google Chat - M2 آلفا - 5 حوزه">
    <a id="google-chat" />

    کانالی مستندشده است، اما راه‌اندازی سازمانی/مدیریتی خطر بلوغ را افزایش می‌دهد.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 0%</span><span>کیفیت آلفا - 59%</span><span>کامل‌بودن آلفا - 66%</span><span><span className="maturity-lts maturity-lts-none">هیچ‌کدام</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">راه‌اندازی و عملیات کانال</span>
          <span>16 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>59%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "59%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Google Chat](/fa/channels/googlechat)، [Google Chat](/fa/plugins/reference/googlechat)، [پیکربندی کانال‌ها](/fa/gateway/config-channels)، [مرجع CLI راه‌انداز](/fa/start/wizard-cli-reference)، [اسرار](/fa/gateway/secrets)، [سطح اعتبارنامه Secretref](/fa/reference/secretref-credential-surface)، [سلامت](/fa/gateway/health)، [فهرست Pluginها](/fa/plugins/plugin-inventory)، [نمایه](/fa/channels/index)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">دسترسی و هویت</span>
          <span>11 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>59%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "59%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Google Chat](/fa/channels/googlechat)، [جفت‌سازی](/fa/channels/pairing)، [گروه‌های دسترسی](/fa/channels/access-groups)، [پیکربندی کانال‌ها](/fa/gateway/config-channels)، [محافظت در برابر حلقهٔ ربات](/fa/channels/bot-loop-protection)، [مسیریابی کانال](/fa/channels/channel-routing)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">مسیریابی و تحویل مکالمه</span>
          <span>1 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>59%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "59%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Google Chat](/fa/channels/googlechat)، [محافظت در برابر حلقه ربات](/fa/channels/bot-loop-protection)، [گروه‌های دسترسی](/fa/channels/access-groups)، [مسیریابی کانال](/fa/channels/channel-routing)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">رسانه و محتوای غنی</span>
          <span>1 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>59%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "59%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Google Chat](/fa/channels/googlechat)، [پیام](/fa/cli/message)، [درک رسانه](/fa/nodes/media-understanding)، [سطح اعتبارنامه Secretref](/fa/reference/secretref-credential-surface)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">کنترل‌های بومی و تأییدها</span>
          <span>16 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>59%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "59%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Google Chat](/fa/channels/googlechat)، [پیام](/fa/cli/message)، [درک رسانه](/fa/nodes/media-understanding)، [سطح اعتبارنامه Secretref](/fa/reference/secretref-credential-surface)، [واکنش‌ها](/fa/tools/reactions)، [دستورهای اسلش](/fa/tools/slash-commands)، [پیکربندی عامل‌ها](/fa/gateway/config-agents)، [بازآرایی چرخه عمر پیام](/fa/concepts/message-lifecycle-refactor)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="Microsoft Teams - آلفای M2 - 5 حوزه">
    <a id="microsoft-teams" />

    جریان‌های احراز هویت و مدیریت سازمانی به اثبات صریح سناریو نیاز دارند.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 0%</span><span>کیفیت آلفا - 59%</span><span>کامل‌بودن آلفا - 66%</span><span><span className="maturity-lts maturity-lts-none">هیچ‌کدام</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">راه‌اندازی و عملیات کانال</span>
          <span>9 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>59%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "59%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Microsoft Teams](/fa/channels/msteams)، [Microsoft Teams](/fa/plugins/reference/msteams)، [کانال‌های پیکربندی](/fa/gateway/config-channels)، [سلامت](/fa/gateway/health)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">دسترسی و هویت</span>
          <span>9 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>59%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "59%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Microsoft Teams](/fa/channels/msteams)، [جفت‌سازی](/fa/channels/pairing)، [گروه‌های دسترسی](/fa/channels/access-groups)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">مسیریابی و تحویل مکالمه</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>59%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "59%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Microsoft Teams](/fa/channels/msteams)، [گروه‌ها](/fa/channels/groups)، [مسیریابی کانال](/fa/channels/channel-routing)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">رسانه و محتوای غنی</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>59%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "59%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Microsoft Teams](/fa/channels/msteams)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">کنترل‌ها و تأییدهای بومی</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>59%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "59%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Microsoft Teams](/fa/channels/msteams)، [تأییدهای پیشرفته اجرای فرمان](/fa/tools/exec-approvals-advanced)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="Signal - آلفای M2 - 5 حوزه">
    <a id="signal" />

    مستندات کانال پشتیبانی‌شده موجود است؛ به شواهد قوی‌تری برای نصب و اتصال مجدد نیاز دارد.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 0%</span><span>کیفیت آلفا - 59%</span><span>کامل‌بودن آلفا - 66%</span><span><span className="maturity-lts maturity-lts-none">هیچ‌کدام</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">راه‌اندازی و عملیات کانال</span>
          <span>7 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>59%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "59%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Signal](/fa/channels/signal), [Signal](/fa/plugins/reference/signal)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">دسترسی و هویت</span>
          <span>6 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>59%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "59%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Signal](/fa/channels/signal)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">مسیریابی و تحویل مکالمه</span>
          <span>1 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>59%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "59%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Signal](/fa/channels/signal)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">رسانه و محتوای غنی</span>
          <span>7 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>59%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "59%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Signal](/fa/channels/signal)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">کنترل‌ها و تأییدهای بومی</span>
          <span>3 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>59%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "59%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Signal](/fa/channels/signal)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="Feishu، QQ Bot، WeChat، Yuanbao، Zalo، Zalo Personal، کانال‌های منطقه‌ای - آلفای M2 - 4 حوزه">
    <a id="feishu-qq-bot-wechat-yuanbao-zalo-zalo-personal-regional-channels" />

    پوشش منطقه‌ای مهم است، اما سطح پشتیبانی عمومی باید برای هر نوع حساب، تأیید بالادستی و شواهد نگه‌دارنده به‌طور جداگانه تنظیم شود.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 0%</span><span>کیفیت آلفا - 55%</span><span>کامل‌بودن آلفا - 58%</span><span><span className="maturity-lts maturity-lts-none">هیچ‌کدام</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">راه‌اندازی و عملیات کانال</span>
          <span>6 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>61%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "61%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [نمایه](/fa/channels/index)، [جفت‌سازی](/fa/channels/pairing)، [Feishu](/fa/plugins/reference/feishu)، [جزئیات داخلی معماری](/fa/plugins/architecture-internals)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">دسترسی و هویت</span>
          <span>1 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>53%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "53%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>54%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "54%" }} /></span></span></div>
        <div className="maturity-category-docs">

    مستند پیوندشده‌ای وجود ندارد

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">مسیریابی و تحویل مکالمه</span>
          <span>1 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>53%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "53%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>54%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "54%" }} /></span></span></div>
        <div className="maturity-category-docs">

    مستند پیوندشده‌ای وجود ندارد

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">رسانه و محتوای غنی</span>
          <span>1 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>53%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "53%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>54%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "54%" }} /></span></span></div>
        <div className="maturity-category-docs">

    مستندات پیوندشده‌ای وجود ندارد

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="Mattermost، LINE، IRC، Nextcloud Talk، Nostr، Twitch، Tlon، Synology Chat - M2 آلفا - 4 حوزه">
    <a id="mattermost-line-irc-nextcloud-talk-nostr-twitch-tlon-synology-chat" />

    سطوح پشتیبانی‌شده وجود دارند، اما سطح بلوغ احتمالاً بسته به پوشش بالادستی و نگه‌دارندگان متفاوت است. بعداً هرکدام را جداگانه امتیازدهی کنید.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 0%</span><span>کیفیت آلفا - 53%</span><span>کامل‌بودن آلفا - 54%</span><span><span className="maturity-lts maturity-lts-none">هیچ‌کدام</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">راه‌اندازی و عملیات کانال</span>
          <span>1 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>53%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "53%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>54%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "54%" }} /></span></span></div>
        <div className="maturity-category-docs">

    مستندات پیوندشده‌ای وجود ندارد

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">دسترسی و هویت</span>
          <span>1 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>53%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "53%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>54%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "54%" }} /></span></span></div>
        <div className="maturity-category-docs">

    مستندات پیوندشده‌ای وجود ندارد

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">مسیریابی و تحویل مکالمه</span>
          <span>1 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>53%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "53%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>54%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "54%" }} /></span></span></div>
        <div className="maturity-category-docs">

    مستندات پیوندشده‌ای وجود ندارد

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">رسانه و محتوای غنی</span>
          <span>1 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>53%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "53%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>54%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "54%" }} /></span></span></div>
        <div className="maturity-category-docs">

    هیچ مستندی پیوند نشده است

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="کانال تماس صوتی - M1 آزمایشی - 5 حوزه">
    <a id="voice-call-channel" />

    مسیر اختیاری/Plugin با رفتار پیچیدهٔ بلادرنگ. پیش از بتای عمومی به کارت امتیاز سناریو نیاز دارد.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 0%</span><span>کیفیت آزمایشی - 41%</span><span>کامل‌بودن آزمایشی - 44%</span><span><span className="maturity-lts maturity-lts-none">هیچ‌کدام</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">راه‌اندازی و عملیات کانال</span>
          <span>2 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>41%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "41%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>44%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "44%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [تماس صوتی](/fa/cli/voicecall)، [تماس صوتی](/fa/plugins/voice-call)، [پروتکل](/fa/gateway/protocol)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">دسترسی و هویت</span>
          <span>1 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>41%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "41%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>44%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "44%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [تماس صوتی](/fa/plugins/voice-call)، [تماس صوتی](/fa/cli/voicecall)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">مسیریابی و تحویل مکالمه</span>
          <span>1 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>41%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "41%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>44%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "44%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [تماس صوتی](/fa/plugins/voice-call)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">رسانه و محتوای غنی</span>
          <span>2 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>41%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "41%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>44%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "44%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [تماس صوتی](/fa/plugins/voice-call)، [فهرست Pluginها](/fa/plugins/plugin-inventory)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">صدای بلادرنگ و تماس‌ها</span>
          <span>2 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>41%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "41%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>44%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "44%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [تماس صوتی](/fa/plugins/voice-call)

    </div>
      </div>
    </div>

  </Accordion>

</AccordionGroup>

### ارائه‌دهنده و ابزار

<AccordionGroup>
  <Accordion title="خودکارسازی مرورگر، اجرا و ابزارهای سندباکس - M3 بتا - 3 حوزه">
    <a id="browser-automation-exec-and-sandbox-tools" />

    ابزارهای اصلی مستند شده‌اند، اما امنیت میزبان و تجربه کاربری مجوزها باید همچنان در بازبینی فعال کارت امتیازی باشند.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 21%</span><span>کیفیت بتا - 75%</span><span>کامل‌بودن بتا - 79%</span><span><span className="maturity-lts maturity-lts-partial">جزئی - 2</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">خودکارسازی مرورگر</span>
          <span>8 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>13%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "13%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [کنترل مرورگر](/fa/tools/browser-control)، [آزمایش](/fa/help/testing)، [مرورگر](/fa/tools/browser)، [نمایه](/fa/gateway/security/index)، [بررسی‌های ممیزی](/fa/gateway/security/audit-checks)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">فراخوانی و اجرای ابزار</span>
          <span>6 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>50%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "50%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [اجرا](/fa/tools/exec)، [فرایند پس‌زمینه](/fa/gateway/background-process)، [فراخوانی API ‏HTTP ابزارها](/fa/gateway/tools-invoke-http-api)، [دامنه‌های عملگر](/fa/gateway/operator-scopes)، [پروتکل](/fa/gateway/protocol)، [تأییدهای اجرا](/fa/tools/exec-approvals)، [تأییدهای پیشرفته اجرا](/fa/tools/exec-approvals-advanced)، [ارتقایافته](/fa/tools/elevated)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">سیاست محیط ایزوله و ابزار</span>
          <span>6 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [ایزوله‌سازی](/fa/gateway/sandboxing)، [محیط ایزوله در برابر سیاست ابزار در برابر حالت ارتقایافته](/fa/gateway/sandbox-vs-tool-policy-vs-elevated)، [ابزارهای محیط ایزوله چندعاملی](/fa/tools/multi-agent-sandbox-tools)، [مرجع چارچوب Codex](/fa/plugins/codex-harness-reference)، [ابزارهای پیکربندی](/fa/gateway/config-tools)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="مسیر ارائه‌دهنده OpenAI و Codex - بتای M3 - 5 حوزه">
    <a id="openai-and-codex-provider-path" />

    مستندات عمیق، مسیر OAuth/اشتراک، صدای بلادرنگ، تصویر و رفتار سازگاری. تغییرات مکرر ارائه‌دهنده مانع از رسیدن این بخش به سطح پایدار بدون شواهد کارت امتیازی انتشار می‌شود.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 26%</span><span>کیفیت بتا - 74%</span><span>کامل‌بودن بتا - 79%</span><span><span className="maturity-lts maturity-lts-partial">جزئی - 3</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">مدل و احراز هویت</span>
          <span>6 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>44%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "44%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [OpenAI](/fa/providers/openai)، [چارچوب Codex](/fa/plugins/codex-harness)، [مدل‌ها](/fa/concepts/models)، [OAuth](/fa/concepts/oauth)، [مرجع چارچوب Codex](/fa/plugins/codex-harness-reference)، [پایش احراز هویت](/fa/gateway/authentication)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">پاسخ‌ها و سازگاری ابزار</span>
          <span>4 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>40%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "40%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [OpenAI](/fa/providers/openai)، [API ‏HTTP ‏OpenResponses](/fa/gateway/openresponses-http-api)، [API ‏HTTP ‏OpenAI](/fa/gateway/openai-http-api)، [Pluginهای بومی Codex](/fa/plugins/codex-native-plugins)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">هارنس بومی Codex</span>
          <span>2 قابلیت / تحت پشتیبانی LTS</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>44%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "44%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [هارنس Codex](/fa/plugins/codex-harness)، [زمان‌اجرای هارنس Codex](/fa/plugins/codex-harness-runtime)، [مرجع هارنس Codex](/fa/plugins/codex-harness-reference)، [Pluginهای بومی Codex](/fa/plugins/codex-native-plugins)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">ورودی تصویر و چندوجهی</span>
          <span>2 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>67%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "67%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [OpenAI](/fa/providers/openai)، [تولید تصویر](/fa/tools/image-generation)، [تصاویر](/fa/nodes/images)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">صدا و صوت بلادرنگ</span>
          <span>2 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>67%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "67%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [OpenAI](/fa/providers/openai)، [Discord](/fa/channels/discord)، [تماس صوتی](/fa/plugins/voice-call)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="ابزارهای جست‌وجوی وب - بتای M3 - 4 حوزه">
    <a id="web-search-tools" />

    چندین ارائه‌دهنده و مستندات موجود است. برای هر خانواده ارائه‌دهنده، به اثبات سهمیه/خطا/SSRF نیاز است.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 9%</span><span>کیفیت بتا - 74%</span><span>کامل‌بودن بتا - 79%</span><span><span className="maturity-lts maturity-lts-none">هیچ‌کدام</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">ارائه‌دهندگان جست‌وجو</span>
          <span>19 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>11%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "11%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [وب](/fa/tools/web)، [جست‌وجوی Brave](/fa/tools/brave-search)، [Tavily](/fa/tools/tavily)، [جست‌وجوی Exa](/fa/tools/exa-search)، [Firecrawl](/fa/tools/firecrawl)، [جست‌وجوی Perplexity](/fa/tools/perplexity-search)، [جست‌وجوی Duckduckgo](/fa/tools/duckduckgo-search)، [جست‌وجوی Searxng](/fa/tools/searxng-search)، [جست‌وجوی Gemini](/fa/tools/gemini-search)، [جست‌وجوی Grok](/fa/tools/grok-search)، [جست‌وجوی Kimi](/fa/tools/kimi-search)، [جست‌وجوی Minimax](/fa/tools/minimax-search)، [جست‌وجوی Ollama](/fa/tools/ollama-search)، [زیرمسیرهای SDK](/fa/plugins/sdk-subpaths)، [نمای کلی SDK](/fa/plugins/sdk-overview)، [مانیفست](/fa/plugins/manifest)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">راه‌اندازی و عیب‌یابی</span>
          <span>9 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [وب](/fa/tools/web)، [واکشی وب](/fa/tools/web-fetch)، [پرسش‌های متداول](/fa/help/faq)، [هزینه‌های استفاده از API](/fa/reference/api-usage-costs)، [جست‌وجوی Brave](/fa/tools/brave-search)، [جست‌وجوی Perplexity](/fa/tools/perplexity-search)، [Tavily](/fa/tools/tavily)، [Firecrawl](/fa/tools/firecrawl)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">ایمنی شبکه</span>
          <span>4 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [وب](/fa/tools/web)، [واکشی وب](/fa/tools/web-fetch)، [Firecrawl](/fa/tools/firecrawl)، [جست‌وجوی Searxng](/fa/tools/searxng-search)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">دسترس‌پذیری ابزار و واکشی</span>
          <span>11 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>25%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "25%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [ابزارهای پیکربندی](/fa/gateway/config-tools)، [واکشی وب](/fa/tools/web-fetch)، [وب](/fa/tools/web)، [پرسش‌های متداول](/fa/help/faq)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="مسیر ارائه‌دهنده Anthropic - بتای M3 - 5 حوزه">
    <a id="anthropic-provider-path" />

    ارائه‌دهنده مدل درجه‌یک. به اثبات دوره‌ای سناریوهای احراز هویت/کاتالوگ/فراخوانی ابزار نیاز دارد.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 0%</span><span>کیفیت بتا - 71%</span><span>کامل‌بودن بتا - 78%</span><span><span className="maturity-lts maturity-lts-none">هیچ‌کدام</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">احراز هویت ارائه‌دهنده و بازیابی</span>
          <span>9 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Anthropic](/fa/providers/anthropic)، [Doctor](/fa/gateway/doctor)، [نمونه‌های پیکربندی](/fa/gateway/configuration-examples)، [عیب‌یابی](/fa/gateway/troubleshooting)، [کش‌کردن پرامپت](/fa/reference/prompt-caching)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">انتخاب مدل و محیط اجرا</span>
          <span>10 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Anthropic](/fa/providers/anthropic)، [پیکربندی عامل‌ها](/fa/gateway/config-agents)، [مدل‌ها](/fa/concepts/models)، [بک‌اندهای CLI](/fa/gateway/cli-backends)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">انتقال درخواست و معناشناسی نوبت</span>
          <span>10 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>77%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "77%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>79%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "79%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Anthropic](/fa/providers/anthropic)، [کش‌کردن پرامپت](/fa/reference/prompt-caching)، [عیب‌یابی](/fa/gateway/troubleshooting)، [بک‌اندهای CLI](/fa/gateway/cli-backends)، [ارائه‌دهندگان مدل](/fa/concepts/model-providers)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">کش پرامپت و زمینه</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Anthropic](/fa/providers/anthropic)، [کش‌کردن پرامپت](/fa/reference/prompt-caching)، [عیب‌یابی](/fa/gateway/troubleshooting)، [Heartbeat](/fa/gateway/heartbeat)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">ورودی‌های رسانه‌ای</span>
          <span>4 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Anthropic](/fa/providers/anthropic)، [پیکربندی عامل‌ها](/fa/gateway/config-agents)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="مسیر ارائه‌دهنده Google - M3 بتا - 5 حوزه">
    <a id="google-provider-path" />

    ارائه‌دهنده‌ای درجه‌یک با سطوح مدل و بلادرنگ. به امتیازدهی جداگانه برای Live/Talk نیاز دارد.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 0%</span><span>کیفیت آلفا - 66%</span><span>کامل‌بودن بتا - 78%</span><span><span className="maturity-lts maturity-lts-none">هیچ‌کدام</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">راه‌اندازی ارائه‌دهنده و اعتبارنامه‌ها</span>
          <span>10 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Google](/fa/providers/google)، [ارائه‌دهندگان مدل](/fa/concepts/model-providers)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">مسیریابی مدل و نقاط پایانی</span>
          <span>10 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Google](/fa/providers/google)، [ارائه‌دهندگان مدل](/fa/concepts/model-providers)، [Google](/fa/plugins/reference/google)، [جست‌وجوی Gemini](/fa/tools/gemini-search)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">محیط اجرای مستقیم Gemini</span>
          <span>9 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Google](/fa/providers/google)، [ارائه‌دهندگان مدل](/fa/concepts/model-providers)، [پرسش‌های متداول مدل‌ها](/fa/help/faq-models)، [آزمایش زنده](/fa/help/testing-live)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">رسانه، جست‌وجو و بلادرنگ</span>
          <span>10 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Google](/fa/plugins/reference/google)، [Google](/fa/providers/google)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">کش‌کردن پرامپت</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [کش‌کردن پرامپت](/fa/reference/prompt-caching)، [Google](/fa/providers/google)، [ارائه‌دهندگان مدل](/fa/concepts/model-providers)، [مصرف توکن](/fa/reference/token-use)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="مسیر ارائه‌دهنده OpenRouter - بتای M3 - 4 حوزه">
    <a id="openrouter-provider-path" />

    مسیر یکپارچه ارائه‌دهنده مستندسازی شده و ارزشمند است، اما رفتار مختص هر مدل متفاوت است.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 0%</span><span>کیفیت آلفا - 66%</span><span>کامل‌بودن بتا - 78%</span><span><span className="maturity-lts maturity-lts-none">هیچ‌کدام</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">راه‌اندازی و احراز هویت ارائه‌دهنده</span>
          <span>14 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Openrouter](/fa/providers/openrouter)، [ارائه‌دهندگان مدل](/fa/concepts/model-providers)، [پیکربندی](/fa/cli/configure)، [احراز هویت](/fa/gateway/authentication)، [محیط](/fa/help/environment)، [مدل‌ها](/fa/cli/models)، [مدل‌ها](/fa/concepts/models)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">زمان اجرای چت و نرمال‌سازی</span>
          <span>15 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Openrouter](/fa/providers/openrouter)، [ارائه‌دهندگان مدل](/fa/concepts/model-providers)، [کش‌کردن پرامپت](/fa/reference/prompt-caching)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">بازیابی و عیب‌یابی ارائه‌دهنده</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [جایگزینی خودکار مدل](/fa/concepts/model-failover)، [Openrouter](/fa/providers/openrouter)، [مدل‌ها](/fa/cli/models)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">تولید رسانه و گفتار</span>
          <span>7 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>66%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "66%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-beta"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-beta">بتا</span><span>78%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "78%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Openrouter](/fa/providers/openrouter)، [تولید تصویر](/fa/tools/image-generation)، [تولید موسیقی](/fa/tools/music-generation)، [نمای کلی رسانه](/fa/tools/media-overview)، [تولید ویدئو](/fa/tools/video-generation)، [تبدیل متن به گفتار](/fa/tools/tts)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="ابزارهای تولید تصویر، ویدئو و موسیقی - M2 آلفا - 5 حوزه">
    <a id="image-video-and-music-generation-tools" />

    این قابلیت در میان ارائه‌دهندگان وجود دارد، اما کیفیت، تأخیر و سازگاری پارامترها بدون اثبات جداگانه برای هر ارائه‌دهنده، بیش از حد متغیر است که بتوان آن را بتا دانست.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 0%</span><span>کیفیت آلفا - 61%</span><span>کامل‌بودن آلفا - 68%</span><span><span className="maturity-lts maturity-lts-none">هیچ‌کدام</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">مسیریابی و کشف رسانه</span>
          <span>4 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>61%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "61%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [پیکربندی عامل‌ها](/fa/gateway/config-agents)، [تولید تصویر](/fa/tools/image-generation)، [تولید ویدئو](/fa/tools/video-generation)، [تولید موسیقی](/fa/tools/music-generation)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">چرخه عمر و تحویل وظیفه</span>
          <span>12 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>61%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "61%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [نمای کلی رسانه](/fa/tools/media-overview)، [تولید تصویر](/fa/tools/image-generation)، [تولید ویدئو](/fa/tools/video-generation)، [تولید موسیقی](/fa/tools/music-generation)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">تولید تصویر</span>
          <span>9 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>61%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "61%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [تولید تصویر](/fa/tools/image-generation)، [استنتاج](/fa/cli/infer)، [نمای کلی رسانه](/fa/tools/media-overview)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">تولید ویدئو</span>
          <span>11 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>61%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "61%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [تولید ویدئو](/fa/tools/video-generation)، [Runway](/fa/providers/runway)، [Pixverse](/fa/providers/pixverse)، [Fal](/fa/providers/fal)، [Openrouter](/fa/providers/openrouter)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">تولید موسیقی</span>
          <span>6 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>61%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "61%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [تولید موسیقی](/fa/tools/music-generation)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="ارائه‌دهندگان مدل محلی: Ollama، vLLM، SGLang، LM Studio - آلفای M2 - 5 حوزه">
    <a id="local-model-providers-ollama-vllm-sglang-lm-studio" />

    مفید و مستند است، اما تفاوت میان محیط‌ها زیاد است.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 0%</span><span>کیفیت آلفا - 61%</span><span>کامل‌بودن آلفا - 68%</span><span><span className="maturity-lts maturity-lts-none">هیچ‌کدام</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">راه‌اندازی، چرخهٔ حیات و عیب‌یابی ارائه‌دهنده</span>
          <span>12 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>61%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "61%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [مدل‌های محلی](/fa/gateway/local-models)، [Lmstudio](/fa/providers/lmstudio)، [Ollama](/fa/providers/ollama)، [Vllm](/fa/providers/vllm)، [سرویس‌های مدل محلی](/fa/gateway/local-model-services)، [پیکربندی عامل‌ها](/fa/gateway/config-agents)، [عیب‌یابی](/fa/gateway/troubleshooting)، [Doctor](/fa/gateway/doctor)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">Pluginهای بومی ارائه‌دهنده</span>
          <span>10 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>61%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "61%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Ollama](/fa/providers/ollama)، [Lmstudio](/fa/providers/lmstudio)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">سازگاری زمان اجرای سازگار با OpenAI</span>
          <span>8 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>61%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "61%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [Vllm](/fa/providers/vllm)، [Sglang](/fa/providers/sglang)، [مدل‌های محلی](/fa/gateway/local-models)، [Lmstudio](/fa/providers/lmstudio)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">حافظه و تعبیه‌سازی‌های محلی</span>
          <span>5 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>61%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "61%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [حافظه](/fa/concepts/memory)، [Doctor](/fa/gateway/doctor)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">ایمنی شبکه و کنترل‌های پرامپت</span>
          <span>2 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>61%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "61%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [نمایه](/fa/gateway/security/index)، [ابزارهای پیکربندی](/fa/gateway/config-tools)، [مدل‌های محلی](/fa/gateway/local-models)

    </div>
      </div>
    </div>

  </Accordion>

  <Accordion title="ارائه‌دهندگان میزبانی‌شده کم‌کاربرد - M2 آلفا - 3 حوزه">
    <a id="long-tail-hosted-providers" />

    صفحات مستندات/مرجع بسیاری وجود دارند؛ امتیاز باید از فراداده ارائه‌دهندگان به‌همراه پوشش آزمون عملی زنده تولید شود.

    <div className="maturity-surface-rollup"><span>پوشش آزمایشی - 0%</span><span>کیفیت آلفا - 61%</span><span>کامل‌بودن آلفا - 68%</span><span><span className="maturity-lts maturity-lts-none">هیچ‌کدام</span></span></div>

    <div className="maturity-category-list">
      <div className="maturity-category-row maturity-category-row-header"><span>حوزه</span><span>پوشش</span><span>کیفیت</span><span>کامل‌بودن</span><span>مستندات</span></div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">ارائه‌دهندگان میزبانی‌شده LLM</span>
          <span>12 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>61%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "61%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [نمایه](/fa/providers/index)، [ارائه‌دهندگان مدل](/fa/concepts/model-providers)، [آزمون زنده](/fa/help/testing-live)، [راه‌اندازی اولیه](/fa/cli/onboard)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">ارائه‌دهندگان میزبانی‌شده رسانه</span>
          <span>8 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>61%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "61%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [مانیفست](/fa/plugins/manifest)، [آزمون زنده](/fa/help/testing-live)، [نمایه](/fa/providers/index)

    </div>
      </div>
      <div className="maturity-category-row">
        <div className="maturity-category-area">
          <span className="maturity-category-title">عملیات ارائه‌دهنده</span>
          <span>12 قابلیت</span>
        </div>
        <div><span className="maturity-score maturity-score-experimental"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-experimental">آزمایشی</span><span>0%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "0%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>61%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "61%" }} /></span></span></div>
        <div><span className="maturity-score maturity-score-alpha"><span className="maturity-score-label"><span className="maturity-level-pill maturity-level-alpha">آلفا</span><span>68%</span></span><span className="maturity-meter" aria-hidden="true"><span style={{ width: "68%" }} /></span></span></div>
        <div className="maturity-category-docs">

    [نمایه](/fa/providers/index)، [ارائه‌دهندگان مدل](/fa/concepts/model-providers)، [مانیفست](/fa/plugins/manifest)، [آزمون زنده](/fa/help/testing-live)، [مدل‌ها](/fa/cli/models)

    </div>
      </div>
    </div>

  </Accordion>

</AccordionGroup>
