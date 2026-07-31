---
read_when: You are managing sandbox runtimes or debugging sandbox/tool-policy behavior.
status: active
summary: Sandbox çalışma zamanlarını yönetin ve geçerli sandbox politikasını inceleyin
title: Sandbox CLI'si
x-i18n:
    generated_at: "2026-07-26T23:52:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ea8311de7702222295f3ba8753304e30f6ed21958e2843f62db5d064f06e24ae
    source_path: cli/sandbox.md
    workflow: 16
---

Yalıtılmış ajan yürütmesi için sandbox çalışma ortamlarını yönetin: Docker kapsayıcıları, SSH hedefleri veya OpenShell arka uçları.

## Komutlar

### `openclaw sandbox list`

Sandbox çalışma ortamlarını durum, arka uç, yapılandırma eşleşmesi, yaş, boşta kalma süresi ve ilişkili oturum/ajan bilgileriyle listeleyin.

```bash
openclaw sandbox list
openclaw sandbox list --browser  # yalnızca tarayıcı kapsayıcıları
openclaw sandbox list --json
```

### `openclaw sandbox recreate`

Geçerli yapılandırmayla yeniden oluşturulmalarını zorlamak için sandbox çalışma ortamlarını kaldırın. Çalışma ortamları, ajan bir sonraki kez kullanıldığında otomatik olarak yeniden oluşturulur.

```bash
openclaw sandbox recreate --all
openclaw sandbox recreate --agent mybot        # agent:mybot:* alt oturumlarını içerir
openclaw sandbox recreate --session "agent:main:main"
openclaw sandbox recreate --browser --all      # yalnızca tarayıcı kapsayıcıları
openclaw sandbox recreate --all --force        # onayı atla
```

Seçenekler:

- `--all`: tüm sandbox kapsayıcılarını yeniden oluşturur
- `--session <key>`: çalışma ortamını tam olarak bu kapsam anahtarıyla yeniden oluşturur (`sandbox list` tarafından gösterildiği gibi); kısa ad genişletmesi yapılmaz
- `--agent <id>`: tek bir ajan için çalışma ortamlarını yeniden oluşturur (`agent:<id>` ve `agent:<id>:*` ile eşleşir)
- `--browser`: yalnızca tarayıcı kapsayıcılarını etkiler
- `--force`: onay istemini atlar

`--all`, `--session` veya `--agent` seçeneklerinden tam olarak birini iletin.

`ssh` ve OpenShell `remote` için yeniden oluşturma, Docker'a kıyasla daha önemlidir: ilk hazırlamadan sonra uzak çalışma alanı kurallı kaynaktır, `recreate` seçilen kapsamın bu kurallı uzak çalışma alanını siler ve sonraki çalıştırma, geçerli yerel çalışma alanından yeniden hazırlar.

### `openclaw sandbox explain`

Etkin sandbox modu/kapsamı/çalışma alanı erişimini, sandbox araç politikasını ve yükseltilmiş araç geçitlerini (düzeltme yapılandırma anahtarı yollarıyla birlikte) inceleyin.

Rapor, `workspaceRoot` değerini yapılandırılmış sandbox kökü olarak tutar ve etkin ana makine çalışma alanını, arka uç çalışma ortamı çalışma dizinini ve Docker bağlama tablosunu ayrıca gösterir. `workspaceAccess: "rw"` için etkin ana makine çalışma alanı, `workspaceRoot` altındaki bir dizin yerine ajan çalışma alanıdır.

```bash
openclaw sandbox explain
openclaw sandbox explain --session agent:main:main
openclaw sandbox explain --agent work
openclaw sandbox explain --json
```

`recreate --session` aksine bu, kısa oturum adlarını (örneğin `main`) kabul eder ve bunları çözümlenen ajana göre genişletir.

## Yeniden oluşturma neden gereklidir?

Sandbox yapılandırmasını güncellemek, çalışan kapsayıcıları etkilemez: mevcut çalışma ortamları eski ayarlarını korur ve boşta olan çalışma ortamları yalnızca `prune.idleHours` sonrasında (varsayılan 24h) temizlenir. Düzenli kullanılan ajanlar, eski çalışma ortamlarını süresiz olarak etkin tutabilir. `openclaw sandbox recreate`, eski çalışma ortamını kaldırır; böylece sonraki kullanımda geçerli yapılandırmadan yeniden oluşturulur.

<Tip>
Arka uca özgü elle temizleme yerine `openclaw sandbox recreate` tercih edin. Bu, Gateway'in çalışma ortamı kayıt defterini kullanır ve kapsam veya oturum anahtarları değiştiğinde oluşabilecek uyumsuzlukları önler.
</Tip>

## Yaygın tetikleyiciler

| Değişiklik                                                                                                                                                     | Komut                                                               |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| Docker imajı güncellemesi (`agents.defaults.sandbox.docker.image`)                                                                                              | `openclaw sandbox recreate --all`                                   |
| Sandbox yapılandırması (`agents.defaults.sandbox.*`)                                                                                                              | `openclaw sandbox recreate --all`                                   |
| SSH hedefi/kimlik doğrulaması (`agents.defaults.sandbox.ssh.{target,workspaceRoot,identityFile,certificateFile,knownHostsFile,identityData,certificateData,knownHostsData}`) | `openclaw sandbox recreate --all`                                   |
| OpenShell kaynağı/politikası/modu (`plugins.entries.openshell.config.{from,mode,policy}`)                                                                      | `openclaw sandbox recreate --all`                                   |
| `setupCommand`                                                                                                                                             | `openclaw sandbox recreate --all` (veya tek bir ajan için `--agent <id>`) |

<Note>
Çalışma ortamları, ajan bir sonraki kez kullanıldığında otomatik olarak yeniden oluşturulur.
</Note>

## Kayıt defteri geçişi

Sandbox çalışma ortamı meta verileri, paylaşılan SQLite durum veritabanında bulunur. Eski kurulumlarda, normal okumaların artık yeniden yazmadığı eski kayıt defteri dosyaları bulunabilir:

- `~/.openclaw/sandbox/containers.json`
- `~/.openclaw/sandbox/browsers.json`
- `~/.openclaw/sandbox/containers/` veya `~/.openclaw/sandbox/browsers/` altında kapsayıcı/tarayıcı başına bir JSON parçası

Geçerli eski girdileri SQLite'a taşımak için `openclaw doctor --fix` çalıştırın. Bozuk eski bir kayıt defterinin geçerli çalışma ortamı girdilerini gizleyememesi için geçersiz eski dosyalar karantinaya alınır.

## Yapılandırma

Sandbox ayarları, `agents.defaults.sandbox` altındaki `~/.openclaw/openclaw.json` içinde bulunur (ajan başına geçersiz kılmalar `agents.entries.*.sandbox` içine eklenir):

```jsonc
{
  "agents": {
    "defaults": {
      "sandbox": {
        "mode": "all", // kapalı, ana olmayan, tümü
        "backend": "docker", // docker, ssh, openshell (plugin tarafından sağlanır)
        "scope": "agent", // oturum, ajan, paylaşılan
        "docker": {
          "image": "openclaw-sandbox:bookworm-slim",
          "containerPrefix": "openclaw-sbx-",
          // ... diğer Docker seçenekleri
        },
        "prune": {
          "idleHours": 24, // 24h boşta kaldıktan sonra otomatik temizle
          "maxAgeDays": 7, // 7 gün sonra otomatik temizle
        },
      },
    },
  },
}
```

## İlgili

- [CLI referansı](/tr/cli)
- [Sandbox kullanımı](/tr/gateway/sandboxing)
- [Ajan çalışma alanı](/tr/concepts/agent-workspace)
- [Doctor](/tr/gateway/doctor): sandbox kurulumunu denetler.
