---
read_when: Connecting the macOS app to a remote gateway over SSH
summary: Uzak bir Gateway'e bağlanan OpenClaw.app için SSH tüneli kurulumu
title: Uzak Gateway kurulumu
x-i18n:
    generated_at: "2026-07-27T00:00:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 842578eb74e99d115b04abff5e9673a6454fa6d2cf7905d056999469e1c6b66d
    source_path: gateway/remote-gateway-readme.md
    workflow: 16
---

<Note>
Bu içerik artık [Uzaktan Erişim](/tr/gateway/remote#macos-persistent-ssh-tunnel-via-launchagent) sayfasında yer alıyor. Güncel kılavuz için o sayfayı kullanın; bu sayfa yönlendirme hedefi olarak kalır.
</Note>

# OpenClaw.app'i Uzak Bir Gateway ile Çalıştırma

OpenClaw.app, bir SSH tüneli üzerinden uzak bir Gateway'e erişir: bir SSH `LocalForward`, yerel bir bağlantı noktasını uzak ana makinedeki Gateway WebSocket bağlantı noktasına eşler.

```mermaid
flowchart TB
    subgraph Client["İstemci Makinesi"]
        direction TB
        A["OpenClaw.app"]
        B["ws://127.0.0.1:18789\n(yerel bağlantı noktası)"]
        T["SSH Tüneli"]

        A --> B
        B --> T
    end
    subgraph Remote["Uzak Makine"]
        direction TB
        C["Gateway WebSocket"]
        D["ws://127.0.0.1:18789"]

        C --> D
    end
    T --> C
```

## Kurulum

1. `LocalForward 18789 127.0.0.1:18789` ile bir SSH yapılandırma girdisi ekleyin (tam yapılandırma bloğu için [Uzaktan Erişim](/tr/gateway/remote#macos-persistent-ssh-tunnel-via-launchagent) sayfasına bakın).
2. SSH anahtarınızı `ssh-copy-id` ile uzak ana makineye kopyalayın.
3. `gateway.remote.token` (veya `gateway.remote.password`) değerini `openclaw config set gateway.remote.token "<your-token>"` aracılığıyla ayarlayın.
4. Tüneli başlatın: `ssh -N remote-gateway &`.
5. OpenClaw.app'ten çıkın ve uygulamayı yeniden açın.

Yeniden başlatmalardan sonra çalışmaya devam eden ve bağlantıyı otomatik olarak yeniden kuran bir tünel için manuel `ssh -N` yerine [Uzaktan Erişim](/tr/gateway/remote#macos-persistent-ssh-tunnel-via-launchagent) sayfasındaki LaunchAgent kurulumunu kullanın.

## Nasıl çalışır?

| Bileşen                            | İşlevi                                                  |
| ------------------------------------ | ------------------------------------------------------------- |
| `LocalForward 18789 127.0.0.1:18789` | Yerel 18789 bağlantı noktasını uzak 18789 bağlantı noktasına yönlendirir                |
| `ssh -N`                             | Uzak komutları yürütmeden SSH bağlantısı kurar (yalnızca bağlantı noktası yönlendirme)  |
| `KeepAlive`                          | Tünel çökerse otomatik olarak yeniden başlatır (LaunchAgent) |
| `RunAtLoad`                          | LaunchAgent yüklendiğinde tüneli başlatır (LaunchAgent)    |

OpenClaw.app, istemcideki `ws://127.0.0.1:18789` adresine bağlanır. Tünel, bu bağlantıyı Gateway'in çalıştığı uzak ana makinedeki 18789 bağlantı noktasına yönlendirir.

## İlgili

- [Uzaktan erişim](/tr/gateway/remote)
- [Tailscale](/tr/gateway/tailscale)
