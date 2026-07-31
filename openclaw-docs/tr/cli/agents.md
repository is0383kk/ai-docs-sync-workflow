---
read_when:
    - Birden fazla yalıtılmış agent istiyorsunuz (çalışma alanları + yönlendirme + kimlik doğrulama)
summary: '`openclaw agents` için CLI başvurusu (listeleme/ekleme/silme/bağlamalar/bağlama/bağlantıyı kaldırma/kimlik ayarlama)'
title: Ajanlar
x-i18n:
    generated_at: "2026-07-26T22:37:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 76a2e50462f6a52760dcb639405ed5f23857f2fa429469281e3acfa1eb61e974
    source_path: cli/agents.md
    workflow: 16
---

# `openclaw agents`

Yalıtılmış ajanları (çalışma alanları + kimlik doğrulama + yönlendirme) yönetin. Alt komut olmadan `openclaw agents` çalıştırmak, `openclaw agents list` çalıştırmaya eşdeğerdir.

İlgili:

- [Çok ajanlı yönlendirme](/tr/concepts/multi-agent)
- [Ajan çalışma alanı](/tr/concepts/agent-workspace)
- [Skills yapılandırması](/tr/tools/skills-config): skill görünürlüğü yapılandırması.

## Örnekler

```bash
openclaw agents list
openclaw agents list --bindings
openclaw agents add work --workspace ~/.openclaw/workspace-work
openclaw agents add work --workspace ~/.openclaw/workspace-work --bind telegram:*
openclaw agents add ops --workspace ~/.openclaw/workspace-ops --bind telegram:ops --non-interactive
openclaw agents bindings
openclaw agents bind --agent work --bind telegram:ops
openclaw agents unbind --agent work --bind telegram:ops
openclaw agents set-identity --workspace ~/.openclaw/workspace --from-identity
openclaw agents set-identity --agent main --avatar avatars/openclaw.png
openclaw agents delete work
```

## Komut yüzeyi

### `agents list`

Seçenekler: `--json`, `--bindings` (yalnızca ajan başına sayımları/özetleri değil, tüm yönlendirme kurallarını dahil eder).

### `agents add [name]`

Seçenekler: `--workspace <dir>`, `--model <id>`, `--agent-dir <dir>`, `--bind <channel[:accountId]>` (tekrarlanabilir), `--non-interactive`, `--json`.

- Herhangi bir açık ekleme bayrağının iletilmesi, komutu etkileşimsiz yola geçirir.
- Etkileşimsiz mod hem bir ajan adı hem de `--workspace` gerektirir.
- `main` ayrılmıştır ve yeni ajan kimliği olarak kullanılamaz.
- Etkileşimli mod, bir kimlik bilgisi `copyToAgents: false` ile kapsam dışında kalmayı seçmediği sürece yalnızca taşınabilir statik kimlik bilgilerini (`api_key` ve statik `token` profilleri) kopyalayarak kimlik doğrulamasını başlangıç durumuna getirir; OAuth yenileme belirteci profilleri, bir sağlayıcı `copyToAgents: true` ile dahil olmayı seçmediği sürece kopyalanmaz. Kopya olmadan OAuth, yalnızca gerçek `main` ajan deposundan okuma yoluyla devralma üzerinden kullanılabilir durumda kalır. Yapılandırılmış varsayılan ajan `main` değilse yeni ajandaki OAuth profilleri için ayrıca oturum açın.

### `agents bindings`

Seçenekler: `--agent <id>`, `--json`.

### `agents bind`

Seçenekler: `--agent <id>` (varsayılan olarak geçerli varsayılan ajan), `--bind <channel[:accountId]>` (tekrarlanabilir), `--json`.

### `agents unbind`

Seçenekler: `--agent <id>` (varsayılan olarak geçerli varsayılan ajan), `--bind <channel[:accountId]>` (tekrarlanabilir), `--all`, `--json`. `--all` veya bir ya da daha fazla `--bind` değeri kabul eder; ikisini birlikte kabul etmez.

### `agents set-identity`

Seçenekler: `--agent <id>`, `--workspace <dir>`, `--identity-file <path>`, `--from-identity`, `--name <name>`, `--theme <theme>`, `--emoji <emoji>`, `--avatar <value>`, `--json`. Aşağıdaki [Kimliği ayarlama](#set-identity) bölümüne bakın.

### `agents delete <id>`

Seçenekler: `--force`, `--json`.

- `main` silinemez.
- `--force` olmadan etkileşimli onay gerekir (TTY olmayan bir oturumda başarısız olur; `--force` ile yeniden çalıştırın).
- Çalışma alanı, ajan durumu ve oturum dökümü dizinleri kalıcı olarak silinmek yerine Çöp Kutusu'na taşınır. Çöp Kutusu kullanılamıyorsa ajan yapılandırmasının silinmesi yine de başarılı olur ve elle temizlenmesi gereken yollar bildirilir.
- Gateway'e erişilebildiğinde silme işlemi Gateway üzerinden yönlendirilir; böylece yapılandırma ve oturum deposu temizliği, çalışma zamanı trafiğiyle aynı yazıcıyı paylaşır. Gateway'e erişilemiyorsa CLI, çevrimdışı yerel yola geri döner.
- Başka bir ajanın çalışma alanı aynı yoldaysa, bu çalışma alanının içindeyse veya bu çalışma alanını içeriyorsa çalışma alanı korunur ve `--json`; `workspaceRetained`, `workspaceRetainedReason` ve `workspaceSharedWith` değerlerini bildirir.

## Yönlendirme bağlamaları

Gelen kanal trafiğini belirli bir ajana sabitlemek için yönlendirme bağlamalarını kullanın.

Ajan başına farklı görünür skill'ler de istiyorsanız `openclaw.json` içinde `agents.defaults.skills` ve `agents.entries.*.skills` yapılandırın. [Skills yapılandırması](/tr/tools/skills-config) ve [Yapılandırma başvurusu](/tr/gateway/config-agents#agentsdefaultsskills) bölümlerine bakın.

Bağlamaları listeleyin:

```bash
openclaw agents bindings
openclaw agents bindings --agent work
openclaw agents bindings --json
```

Bağlamalar ekleyin:

```bash
openclaw agents bind --agent work --bind telegram:ops --bind discord:guild-a
```

Bir ajan oluştururken de bağlamalar ekleyebilirsiniz:

```bash
openclaw agents add work --workspace ~/.openclaw/workspace-work --bind telegram:* --bind discord:*
```

`accountId` (`--bind <channel>`) belirtilmezse OpenClaw bunu Plugin kurulum kancalarından, zorunlu hesap bağlamasından veya kanalın yapılandırılmış hesap sayısından çözümler.

`bind` veya `unbind` için `--agent` belirtilmezse OpenClaw geçerli varsayılan ajanı hedefler.

### `--bind` biçimi

| Biçim                       | Anlamı                                                                                            |
| ---------------------------- | -------------------------------------------------------------------------------------------------- |
| `--bind <channel>:*`         | Kanaldaki tüm hesaplarla eşleşir.                                                                 |
| `--bind <channel>:<account>` | Tek bir hesapla eşleşir.                                                                                 |
| `--bind <channel>`           | CLI, Plugin'e özgü bir hesap kapsamını güvenli biçimde çözümleyemediği sürece yalnızca varsayılan hesapla eşleşir. |

### Bağlama kapsamı davranışı

- `accountId` içermeyen kayıtlı bir bağlama yalnızca kanalın varsayılan hesabıyla eşleşir.
- `accountId: "*"`, kanal genelindeki geri dönüş seçeneğidir (tüm hesaplar) ve açık bir hesap bağlamasından daha az özeldir.
- Aynı ajanda `accountId` olmadan eşleşen bir kanal bağlaması zaten varsa ve daha sonra açık veya çözümlenmiş bir `accountId` ile bağlama yaparsanız OpenClaw, yinelenen bir bağlama eklemek yerine mevcut bağlamayı yerinde yükseltir.

Örnekler:

```bash
# kanaldaki tüm hesaplarla eşleştir
openclaw agents bind --agent work --bind telegram:*

# belirli bir hesapla eşleştir
openclaw agents bind --agent work --bind telegram:ops

# başlangıçtaki yalnızca kanal bağlaması
openclaw agents bind --agent work --bind telegram

# daha sonra hesap kapsamlı bağlamaya yükselt
openclaw agents bind --agent work --bind telegram:alerts
```

Yükseltmeden sonra bu bağlamanın yönlendirmesi `telegram:alerts` ile kapsamlandırılır. Varsayılan hesap yönlendirmesini de istiyorsanız bunu açıkça ekleyin (örneğin `--bind telegram:default`).

Bağlamaları kaldırın:

```bash
openclaw agents unbind --agent work --bind telegram:ops
openclaw agents unbind --agent work --all
```

## Kimlik dosyaları

Her ajan çalışma alanı, çalışma alanı kökünde bir `IDENTITY.md` içerebilir:

- Örnek yol: `~/.openclaw/workspace/IDENTITY.md`
- `set-identity --from-identity`, çalışma alanı kökünden (veya açık bir `--identity-file` yolundan) okur.

Avatar yolları çalışma alanı köküne göre çözümlenir ve sembolik bağlantı üzerinden bile bunun dışına çıkamaz.

## Kimliği ayarlama

`set-identity`, `agents.entries.*.identity` içine şu alanları yazar: `name`, `theme`, `emoji`, `avatar` (çalışma alanına göre yol, http(s) URL'si veya veri URI'si).

- `--agent` veya `--workspace` hedef ajanı seçer. `--workspace` birden fazla ajanla eşleşirse komut başarısız olur ve `--agent` iletmenizi ister.
- Çalışma alanına göre belirtilen yerel avatar görüntüsü dosyaları 2 MB ile sınırlıdır. HTTP(S) URL'leri ve `data:` URI'leri yerel dosya boyutu sınırına göre denetlenmez.
- Açık kimlik alanları sağlanmadığında komut, kimlik verilerini `IDENTITY.md` konumundan okur.

`IDENTITY.md` kaynağından yükleyin:

```bash
openclaw agents set-identity --workspace ~/.openclaw/workspace --from-identity
```

Alanları açıkça geçersiz kılın:

```bash
openclaw agents set-identity --agent main --name "OpenClaw" --emoji "🦞" --avatar avatars/openclaw.png
```

Yapılandırma örneği:

```json5
{
  agents: {
    list: [
      {
        id: "main",
        identity: {
          name: "OpenClaw",
          theme: "uzay ıstakozu",
          emoji: "🦞",
          avatar: "avatars/openclaw.png",
        },
      },
    ],
  },
}
```

## İlgili

- [CLI başvurusu](/tr/cli)
- [Çok ajanlı yönlendirme](/tr/concepts/multi-agent)
- [Ajan çalışma alanı](/tr/concepts/agent-workspace)
