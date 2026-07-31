---
read_when:
    - zsh/bash/fish/PowerShell 用のシェル補完が必要な場合
    - OpenClaw の状態領域に補完スクリプトをキャッシュする必要があります
summary: '`openclaw completion` の CLI リファレンス（シェル補完スクリプトの生成/インストール）'
title: 完了
x-i18n:
    generated_at: "2026-07-26T09:28:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 67cb52a47036745150887c752d18e2dfa84fab2722c27c696142d23080bb2efd
    source_path: cli/completion.md
    workflow: 16
---

# `openclaw completion`

シェル補完スクリプトを生成して OpenClaw の状態領域にキャッシュし、必要に応じてシェルプロファイルにインストールします。

## 使用方法

```bash
openclaw completion                          # zsh スクリプトを標準出力に出力
openclaw completion --shell fish             # fish スクリプトを出力
openclaw completion --write-state            # すべてのシェル用スクリプトをキャッシュ
openclaw completion --write-state --install  # キャッシュしてから一度にインストール
openclaw completion --shell bash --write-state
```

## オプション

- `-s, --shell <shell>`: 対象シェル（`zsh`、`bash`、`powershell`、`fish`。デフォルト: `zsh`）
- `-i, --install`: キャッシュ済みスクリプトを読み込む行をシェルプロファイルに追加して補完をインストール
- `--write-state`: 標準出力には出力せず、補完スクリプトを `$OPENCLAW_STATE_DIR/completions`（デフォルトは `~/.openclaw/completions`）に書き込む。`--shell` を指定した場合はそのシェルのみ、指定しない場合は 4 つすべてを書き込む
- `-y, --yes`: インストール確認プロンプトを省略（非対話型）

## インストールの流れ

`--install` はプロファイルからキャッシュ済みスクリプトを参照するようにするため、先にキャッシュが存在している必要があります。キャッシュがない場合、コマンドは失敗し、`openclaw completion --write-state` を実行するよう案内します。`--write-state --install` を組み合わせると、両方を一度に実行できます。`--shell` を指定しない場合、`--install` は `$SHELL` からシェルを検出します（検出できない場合は zsh）。

インストールでは、シェルプロファイルに小さな `# OpenClaw Completion` ブロックを書き込み、古くて低速な `source <(openclaw completion ...)` 行があれば、キャッシュ済みスクリプトを読み込む行に置き換えます。

| シェル     | プロファイル                                                                                                                                                                               |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| bash       | `~/.bashrc`（`~/.bashrc` がない場合は `~/.bash_profile` を使用）                                                                                                           |
| fish       | `~/.config/fish/config.fish`                                                                                                                                                                        |
| powershell | `~/.config/powershell/Microsoft.PowerShell_profile.ps1`（Windows の場合: `Documents/PowerShell/Microsoft.PowerShell_profile.ps1`、または Windows PowerShell では `Documents/WindowsPowerShell/...`）                                                                                |
| zsh        | `~/.zshrc`                                                                                                                                                                        |

## 注意事項

- `--install` または `--write-state` を指定しない場合、コマンドはスクリプトを標準出力に出力します。
- 補完の生成では、Plugin の CLI コマンドを含むコマンドツリー全体が事前に読み込まれるため、ネストされたサブコマンドも含まれます。
- `openclaw update` は更新に成功すると補完キャッシュを自動的に更新します。`openclaw doctor` は欠落または古くなった補完設定を修復できます。

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
