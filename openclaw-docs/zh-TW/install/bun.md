---
read_when:
    - 你想要使用 Bun 安裝相依套件或執行套件指令碼
    - 你遇到 Bun 安裝、修補或生命週期指令碼問題
summary: 使用 Bun 進行安裝與套件指令碼工作流程；執行階段需要 Node
title: Bun
x-i18n:
    generated_at: "2026-07-26T07:56:39Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b822f700123b91c785eb881ebf28a63e77915b46dfd44beb9dbf63fb71aaa0d2
    source_path: install/bun.md
    workflow: 16
---

<Warning>
Bun 無法執行 OpenClaw 命令列介面或閘道，因為它未提供必要的 `node:sqlite` API。請安裝支援的 Node 版本，以執行所有 OpenClaw 執行階段命令。
</Warning>

Bun 仍可作為選用的相依套件安裝工具和套件指令碼執行工具。預設套件管理器仍為 `pnpm`，其受到完整支援，並由文件工具使用。Bun 無法使用 `pnpm-lock.yaml`，且會忽略它。

## 安裝

<Steps>
  <Step title="安裝相依套件">
    ```sh
    bun install
    ```

    `bun.lock` / `bun.lockb` 已列入 gitignore，因此不會造成儲存庫變動。若要完全略過鎖定檔寫入：

    ```sh
    bun install --no-save
    ```

  </Step>
  <Step title="建置與測試">
    ```sh
    bun run build
    bun run vitest run
    ```

    啟動 OpenClaw 本身的命令仍必須透過 Node 執行。

  </Step>
</Steps>

## 生命週期指令碼

除非明確信任，否則 Bun 會封鎖相依套件的生命週期指令碼。對此儲存庫而言，通常遭封鎖的指令碼並非必要：

- `baileys` `preinstall`：檢查 Node 主要版本是否 >= 20（OpenClaw 需要 Node 22.22.3+、24.15+ 或 25.9+，建議使用 Node 24）
- `protobufjs` `postinstall`：針對不相容的版本配置方式發出警告（不產生建置成品）

如果遇到需要這些指令碼的執行階段問題，請明確信任它們：

```sh
bun pm trust baileys protobufjs
```

## 注意事項

有些套件指令碼在內部將 `pnpm` 寫死（例如 `check:docs`、`ui:*`、`protocol:check`）。透過 `bun run` 執行它們時，仍會啟動殼層並呼叫 `pnpm`，因此請直接透過 `pnpm` 執行這些指令碼。

## 相關內容

- [安裝概覽](/zh-TW/install)
- [Node.js](/zh-TW/install/node)
- [更新](/zh-TW/install/updating)
