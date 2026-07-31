---
read_when:
    - Uzaktan Mac denetimini ayarlama veya hata ayıklama
summary: Uzak bir OpenClaw Gateway'ini denetlemek için macOS uygulama akışı
title: Uzaktan kontrol
x-i18n:
    generated_at: "2026-07-27T00:05:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7e558c39fa173a77bf11270a8961c14c6e2350dfc4f458da3633532513b98bf6
    source_path: platforms/mac/remote.md
    workflow: 16
---

Bu akış, macOS uygulamasının başka bir ana makinede (masaüstü/sunucu) çalışan bir OpenClaw Gateway için tam uzaktan kumanda görevi görmesini sağlar. Uygulama, güvenilir LAN/Tailnet Gateway URL'lerine doğrudan bağlanır veya uzak Gateway yalnızca geri döngü üzerinden kullanılabiliyorsa bir SSH tünelini yönetir. Sistem durumu kontrolleri, Sesle Uyandırma yönlendirmesi ve Web Sohbeti, _Settings -> General_ bölümündeki aynı uzak yapılandırmayı yeniden kullanır.

## Modlar

- **Yerel (bu Mac)**: her şey dizüstü bilgisayarda çalışır; SSH kullanılmaz.
- **SSH üzerinden uzak (varsayılan)**: OpenClaw komutları uzak ana makinede çalışır. Uygulama; `-o BatchMode`, seçtiğiniz kimlik/anahtar ve yerel port yönlendirmesiyle bir SSH bağlantısı açar.
- **Doğrudan uzak (ws/wss)**: SSH tüneli kullanılmaz; uygulama Gateway URL'sine doğrudan bağlanır (LAN, Tailscale, Tailscale Serve veya genel bir HTTPS ters proxy'si).

## Uzak aktarım yöntemleri

- **SSH tüneli** (varsayılan): Gateway portunu localhost'a yönlendirmek için `ssh -N -L ...` kullanır. Tünel geri döngü üzerinden çalıştığı için Gateway, Node'un IP adresini `127.0.0.1` olarak görür.
- **Doğrudan (ws/wss)**: Gateway URL'sine doğrudan bağlanır. Gateway, gerçek istemci IP adresini görür.

Uygulama, seçilen diğer ad `ControlMaster` veya `ForkAfterAuthentication` özelliğini etkinleştirse bile ilgili işlemi tam olarak izleyip yeniden başlatabilmek için kendi SSH işlemlerinde SSH bağlantısı çoğullamasını ve kimlik doğrulama sonrası arka plana geçişi devre dışı bırakır.

Gateway kimlik bilgileri bu tünel üzerinden aktarıldığından SSH ana makine anahtarı doğrulaması varsayılan olarak katıdır. Yönetilen bir SSH diğer adının kendi güven davranışını kullanmayı seçmek için `openclaw-mac configure-remote` aracılığıyla `--ssh-host-key-policy openssh` ayarını yapın veya `gateway.remote.sshHostKeyPolicy` değerini doğrudan `"openssh"` olarak ayarlayın. Etkinleştirmeden önce diğer adı ve eşleşen `Host *` ya da sistem yapılandırmasını inceleyin. SSH hedefinin değiştirilmesi (uygulamada veya `configure-remote` aracılığıyla), yeni hedef için bu davranışı yeniden açıkça etkinleştirmediğiniz sürece politikayı tekrar `strict` değerine sıfırlar.

SSH tüneli modunda keşfedilen LAN/Tailnet ana makine adları `gateway.remote.sshTarget` olarak kaydedilir. Uygulama, CLI, Web Sohbeti ve yerel Node ana makine hizmetinin aynı geri döngü aktarımını kullanması için `gateway.remote.url` değerini yerel tünel uç noktasında (örneğin `ws://127.0.0.1:18789`) tutar. Keşif hem ham Tailnet IP'leri hem de kararlı ana makine adlarını döndürdüğünde uygulama, bağlantıların adres değişikliklerine karşı daha dayanıklı olması için Tailscale MagicDNS veya LAN adlarını tercih eder. Yerel tünel portu uzak Gateway portundan farklıysa `gateway.remote.remotePort` değerini uzak ana makinedeki porta ayarlayın.

Uzak moddaki tarayıcı otomasyonunun sahibi, yerel macOS uygulaması Node'u değil, CLI Node ana makinesidir. Uygulama, mümkün olduğunda kurulu Node ana makine hizmetini başlatır; söz konusu Mac'ten tarayıcı denetimini etkinleştirmek için hizmeti `openclaw node install ...` ve `openclaw node start` ile kurup başlatın (veya `openclaw node run ...` komutunu ön planda çalıştırın), ardından tarayıcı özelliğine sahip bu Node'u hedefleyin.

## Uzak ana makinedeki ön koşullar

1. Node + pnpm'i kurun ve OpenClaw CLI'yi derleyin/kurun (`pnpm install && pnpm build && pnpm link --global`).
2. Etkileşimsiz kabuklarda `openclaw` öğesinin PATH üzerinde olduğundan emin olun (gerekirse `/usr/local/bin` veya `/opt/homebrew/bin` içine sembolik bağlantı oluşturun).
3. SSH aktarımı için: anahtar tabanlı SSH kimlik doğrulamasını ayarlayın. LAN dışından kararlı erişilebilirlik için Tailscale IP'leri önerilir.

## macOS uygulaması kurulumu

Uygulamayı karşılama akışını kullanmadan SSH üzerinden önceden yapılandırmak için:

```bash
openclaw-mac configure-remote \
  --ssh-target user@gateway-host \
  --local-port 18789 \
  --remote-port 18789 \
  --token "$OPENCLAW_GATEWAY_TOKEN"
```

Güvenilir bir LAN veya Tailnet üzerinden zaten erişilebilen bir Gateway için SSH'yi tamamen atlayabilirsiniz:

```bash
openclaw-mac configure-remote \
  --direct-url ws://192.168.0.202:18789 \
  --token "$OPENCLAW_GATEWAY_TOKEN"
```

`openclaw-mac connect`, `wizard` ve `configure-remote` etkin yapılandırmayı şu sırayla çözümler: `OPENCLAW_CONFIG_PATH`, ardından `$OPENCLAW_STATE_DIR/openclaw.json`, ardından `~/.openclaw/openclaw.json`. Her iki yapılandırma biçimi de bu etkin dosyaya yazar, ilk katılımı tamamlanmış olarak işaretler ve uygulamanın bir sonraki başlangıçta seçilen aktarım yöntemini yönetmesini sağlar. `--local-port`/`--remote-port` için varsayılan değer `18789` olur. Diğer bayraklar: `--password`, `--identity <path>`, `--ssh-host-key-policy <strict|openssh>`, `--project-root <path>`, `--cli-path <path>`, `--json`. Tam başvuru için `openclaw-mac configure-remote --help` komutunu çalıştırın.

Bunun yerine kullanıcı arayüzünden yapılandırmak için:

1. _Settings -> General_ bölümünü açın.
2. **OpenClaw runs** altında **Remote** seçeneğini belirleyin ve şunları ayarlayın:
   - **Transport**: **SSH tunnel** veya **Direct (ws/wss)**.
   - **SSH target**: `user@host` (isteğe bağlı `:port`). Gateway aynı LAN üzerindeyse ve Bonjour ile kendini duyuruyorsa bu alanı otomatik doldurmak için keşfedilenler listesinden Gateway'i seçin.
   - **Gateway URL** (yalnızca Direct): `wss://gateway.example.ts.net` (veya yerel/LAN için `ws://...`).
   - **Identity file** (gelişmiş): anahtarınızın yolu.
   - **Project root** (gelişmiş): komutlar için kullanılan uzak çalışma kopyasının yolu.
   - **CLI path** (gelişmiş): çalıştırılabilir bir `openclaw` giriş noktasına/ikili dosyasına giden isteğe bağlı yol (duyurulduğunda otomatik doldurulur).
3. **Test remote** düğmesine basın. Başarı, uzak `openclaw status --json` komutunun doğru çalıştığı anlamına gelir. Hatalar genellikle PATH/CLI sorunlarını gösterir; 127 çıkış kodu, CLI'nin uzak ana makinede bulunamadığı anlamına gelir.
4. Sistem durumu kontrolleri ve Web Sohbeti artık seçilen aktarım yöntemi üzerinden otomatik olarak çalışır.

## Web Sohbeti

- **SSH tüneli**: yönlendirilmiş WebSocket denetim portu (varsayılan 18789) üzerinden Gateway'e bağlanır.
- **Doğrudan (ws/wss)**: yapılandırılmış Gateway URL'sine doğrudan bağlanır.
- Ayrı bir Web Sohbeti HTTP sunucusu yoktur.

## İzinler

- Uzak ana makine, yerel ana makineyle aynı TCC onaylarına ihtiyaç duyar (Otomasyon, Erişilebilirlik, Ekran Kaydı, Mikrofon, Konuşma Tanıma, Bildirimler). Bu izinleri vermek için ilk katılımı söz konusu makinede bir kez çalıştırın.
- Aracıların nelerin kullanılabilir olduğunu bilmesi için Node'lar izin durumlarını `node.list` / `node.describe` aracılığıyla duyurur.

## Güvenlik notları

- Uzak ana makinede geri döngü bağlamalarını tercih edin ve SSH, Tailscale Serve veya güvenilir bir Tailnet/LAN doğrudan URL'si üzerinden bağlanın.
- SSH tünellemesi varsayılan olarak önceden güvenilen bir ana makine anahtarı gerektirir. Önce ana makine anahtarına güvenin (yapılandırılmış bilinen ana makineler dosyasına ekleyin) veya OpenSSH güven politikasını kabul ettiğiniz yönetilen bir diğer ad için `gateway.remote.sshHostKeyPolicy: "openssh"` değerini açıkça ayarlayın.
- Gateway'i geri döngü dışı bir arayüze bağlıyorsanız geçerli Gateway kimlik doğrulamasını zorunlu kılın: token, parola veya `gateway.auth.mode: "trusted-proxy"` kullanan kimlik farkındalığına sahip bir ters proxy.
- Doğrudan `wss://` bağlantıları hem operatör/denetim trafiğine hem de Mac yardımcı Node'una tek bir sertifika politikası uygular. Açık bir sabitleme için `gateway.remote.tlsFingerprint` değerini ayarlayın. Bu değer olmadan uygulama, yalnızca normal macOS güven doğrulaması başarılı olduktan sonra ilk kullanım sabitlemesini kaydeder.
- [Güvenlik](/tr/gateway/security) ve [Tailscale](/tr/gateway/tailscale) bölümlerine bakın.

## WhatsApp oturum açma akışı (uzak)

- `openclaw channels login --channel whatsapp --verbose` komutunu **uzak ana makinede** çalıştırın. QR kodunu telefonunuzdaki WhatsApp ile tarayın.
- Kimlik doğrulamasının süresi dolarsa oturum açma işlemini söz konusu ana makinede yeniden çalıştırın. Sistem durumu kontrolü bağlantı sorunlarını gösterir.

## Sorun giderme

| Belirti                                          | Neden / çözüm                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| ------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `exit 127` / bulunamadı                           | `openclaw`, oturum açma dışı kabuklar için PATH üzerinde değil. Bunu `/etc/paths` dosyasına veya kabuk rc dosyanıza ekleyin ya da `/usr/local/bin`/`/opt/homebrew/bin` içine sembolik bağlantı oluşturun.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Sağlık yoklaması başarısız oldu                              | SSH erişilebilirliğini, PATH'i ve Baileys (WhatsApp) oturumunun açık olduğunu (`openclaw status --json`) kontrol edin.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| Web Sohbeti takılı kaldı                                   | Gateway'in uzak ana bilgisayarda çalıştığını ve yönlendirilen bağlantı noktasının Gateway WS bağlantı noktasıyla eşleştiğini doğrulayın; kullanıcı arayüzü sağlıklı bir WS bağlantısı gerektirir.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| Node IP'si `127.0.0.1` gösteriyor                        | SSH tünelinde bu beklenen bir durumdur. Gateway'in gerçek istemci IP'sini görmesini istiyorsanız **Transport** seçeneğini **Direct (ws/wss)** olarak değiştirin.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Pano çalışıyor ancak Mac özellikleri çevrimdışı | Operatör/denetim bağlantısı sağlıklı, ancak eşlik eden Node bağlantısı bağlı değil veya komut yüzeyi eksik. Menü çubuğundaki aygıt bölümünü açın ve Mac'in `paired · disconnected` olup olmadığını kontrol edin. Doğrudan `wss://` operatör ve Node bağlantıları, yapılandırılmış veya depolanmış aynı sertifika politikasını kullanır. Güvenilir `wss://*.ts.net` Tailscale Serve uç noktalarında, geçerliliğini yitirmiş depolanmış yaprak sertifika sabitlemeleri sertifika rotasyonundan sonra değiştirilir ve bağlantı otomatik olarak yeniden denenir. Yapılandırılmış sabitlemeler hiçbir zaman otomatik olarak yenilenmez; yeni sertifikayı inceledikten sonra `gateway.remote.tlsFingerprint` değerini güncelleyin veya **Remote over SSH** seçeneğine geçin. |
| Sesle Uyandırma                                       | Tetikleyici ifadeler uzak modda otomatik olarak iletilir; ayrı bir iletici gerekmez.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |

## Bildirim sesleri

Her bildirim için betiklerden `openclaw nodes notify` ile ses seçin, örneğin:

```bash
openclaw nodes notify --node <id> --title "Ping" --body "Uzak Gateway hazır" --sound Glass
```

Uygulamada genel bir varsayılan ses anahtarı yoktur; çağıranlar her istek için bir ses seçer (veya hiç seçmez).

## İlgili

- [macOS uygulaması](/tr/platforms/macos)
- [Uzaktan erişim](/tr/gateway/remote)
