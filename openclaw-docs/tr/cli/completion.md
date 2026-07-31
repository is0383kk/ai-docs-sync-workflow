---
read_when:
    - zsh/bash/fish/PowerShell için kabuk otomatik tamamlamaları istiyorsunuz
    - Tamamlama betiklerini OpenClaw durumunda önbelleğe almanız gerekir
summary: '`openclaw completion` için CLI başvurusu (kabuk tamamlama betiklerini oluşturma/yükleme)'
title: Tamamlama
x-i18n:
    generated_at: "2026-07-26T23:14:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 67cb52a47036745150887c752d18e2dfa84fab2722c27c696142d23080bb2efd
    source_path: cli/completion.md
    workflow: 16
---

# `openclaw completion`

Kabuk tamamlama betiklerini oluşturun, bunları OpenClaw durumu altında önbelleğe alın ve isteğe bağlı olarak kabuk profilinize yükleyin.

## Kullanım

```bash
openclaw completion                          # zsh betiğini stdout'a yazdır
openclaw completion --shell fish             # fish betiğini yazdır
openclaw completion --write-state            # tüm kabukların betiklerini önbelleğe al
openclaw completion --write-state --install  # önbelleğe al, ardından tek adımda yükle
openclaw completion --shell bash --write-state
```

## Seçenekler

- `-s, --shell <shell>`: hedef kabuk (`zsh`, `bash`, `powershell`, `fish`; varsayılan: `zsh`)
- `-i, --install`: önbelleğe alınmış betik için kabuk profilinize bir source satırı ekleyerek tamamlamayı yükler
- `--write-state`: stdout'a yazdırmadan tamamlama betiklerini `$OPENCLAW_STATE_DIR/completions` konumuna yazar (varsayılan `~/.openclaw/completions`); `--shell` ile yalnızca belirtilen kabuğu, aksi takdirde dördünü de yazar
- `-y, --yes`: yükleme onay istemlerini atlar (etkileşimsiz)

## Yükleme akışı

`--install`, profilinizi önbelleğe alınmış betiğe yönlendirir; bu nedenle önce önbelleğin mevcut olması gerekir: mevcut değilse komut başarısız olur ve `openclaw completion --write-state` komutunu çalıştırmanızı söyler. Her ikisini tek adımda yapmak için `--write-state --install` ile birleştirin. `--shell` olmadan `--install`, kabuğu `$SHELL` üzerinden algılar (algılanamazsa zsh kullanılır).

Yükleme, kabuk profilinize küçük bir `# OpenClaw Completion` bloğu yazar ve eski, yavaş `source <(openclaw completion ...)` satırlarını önbelleğe alınmış source satırıyla değiştirir:

| Kabuk      | Profil                                                                                                                                                                                    |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| bash       | `~/.bashrc` (`~/.bashrc` mevcut olmadığında `~/.bash_profile` kullanılır)                                                                                                                  |
| fish       | `~/.config/fish/config.fish`                                                                                                                                                               |
| powershell | `~/.config/powershell/Microsoft.PowerShell_profile.ps1` (Windows'ta: `Documents/PowerShell/Microsoft.PowerShell_profile.ps1` veya Windows PowerShell için `Documents/WindowsPowerShell/...`) |
| zsh        | `~/.zshrc`                                                                                                                                                                                 |

## Notlar

- `--install` veya `--write-state` olmadan komut, betiği stdout'a yazdırır.
- Tamamlama oluşturma işlemi, iç içe alt komutların da dahil edilmesi için Plugin CLI komutları da dahil olmak üzere tam komut ağacını önceden yükler.
- `openclaw update`, başarılı bir güncellemeden sonra tamamlama önbelleğini otomatik olarak yeniler; `openclaw doctor`, eksik veya eski tamamlama kurulumlarını onarabilir.

## İlgili

- [CLI başvurusu](/tr/cli)
