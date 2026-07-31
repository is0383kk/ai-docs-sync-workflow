---
read_when:
    - 在執行階段重新解析祕密參照
    - 稽核純文字殘留與未解析的參照
    - 設定 SecretRefs 並套用單向清除變更
summary: '`openclaw secrets` 的命令列介面參考（重新載入、稽核、設定、套用）'
title: 密鑰
x-i18n:
    generated_at: "2026-07-26T08:28:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 61f6f81e358ca2e6a97ac9498186b32f7a74d16052d226c398dad0030d47211e
    source_path: cli/secrets.md
    workflow: 16
---

# `openclaw secrets`

管理 SecretRef，並維持作用中執行階段快照的健康狀態。

| 命令        | 角色                                                                                                                                                                                                 |
| ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `reload`    | 閘道 RPC（`secrets.reload`）：重新解析參照，並以不可分割方式發布可識別擁有者的執行階段快照（不寫入設定）；符合條件的擁有者失敗可發布為冷啟動或過期警告 |
| `audit`     | 以唯讀方式掃描設定／驗證／產生的模型儲存區與舊版殘留項目，檢查明文、未解析的參照及優先順序偏移（除非使用 `--allow-exec`，否則會略過 exec 參照） |
| `configure` | 用於提供者設定、目標對應及預檢的互動式規劃工具（需要 TTY）                                                                                                                                            |
| `apply`     | 執行已儲存的計畫（`--dry-run` 僅驗證，且預設略過 exec 檢查；除非使用 `--allow-exec`，否則寫入模式會拒絕包含 exec 的計畫），接著清除指定目標中的明文殘留項目 |

建議的操作員流程：

```bash
openclaw secrets audit --check
openclaw secrets configure
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json
openclaw secrets audit --check
openclaw secrets reload
```

如果計畫包含 `exec` SecretRef／提供者，請在試執行與寫入的 `apply` 命令中都傳入 `--allow-exec`。

CI／閘門的結束代碼：

- `audit --check` 發現問題時會傳回 `1`。
- 未解析的參照會傳回 `2`（無論是否使用 `--check`）。

相關內容：[機密資料管理](/zh-TW/gateway/secrets) · [SecretRef 認證資訊介面](/zh-TW/reference/secretref-credential-surface) · [安全性](/zh-TW/gateway/security)

## 重新載入執行階段快照

```bash
openclaw secrets reload
openclaw secrets reload --json
openclaw secrets reload --url ws://127.0.0.1:18789 --token <token>
```

使用閘道 RPC 方法 `secrets.reload`。健康的擁有者會各自重新整理。只有在參照識別資訊、提供者定義及完整的非機密擁有者合約均未變更時，符合條件但失敗的擁有者才會變為過期狀態；新增或已變更的失敗會變為冷啟動狀態。這種降級啟用會成功，並回報 `warningCount`。嚴格模式或未對應的失敗會傳回錯誤，並保留先前作用中的快照。

選項：`--url <url>`、`--token <token>`、`--timeout <ms>`、`--json`。

## 稽核

掃描 OpenClaw 狀態，檢查：

- 以明文儲存機密資料
- 未解析的參照
- 優先順序偏移（`auth-profiles.json` 認證資訊遮蔽 `openclaw.json` 參照）
- 產生的 `agents/*/agent/models.json` 殘留項目（提供者 `apiKey` 值與敏感的提供者標頭）
- 舊版殘留項目（舊版驗證儲存區項目、OAuth 提醒）

`.env` 掃描涵蓋有效的狀態目錄，以及包含作用中設定的目錄。當兩個路徑指向同一個檔案時，只會掃描一次。

敏感提供者標頭的偵測以名稱啟發法為基礎：名稱符合常見驗證／認證資訊片段（`authorization`、`x-api-key`、`token`、`secret`、`password`、`credential`）的標頭會被標記。

```bash
openclaw secrets audit
openclaw secrets audit --check
openclaw secrets audit --json
openclaw secrets audit --allow-exec
```

報告格式：

- `status`：`clean | findings | unresolved`
- `resolution`：`refsChecked`、`skippedExecRefs`、`resolvabilityComplete`
- `summary`：`plaintextCount`、`unresolvedRefCount`、`shadowedRefCount`、`legacyResidueCount`
- 發現項目代碼：`PLAINTEXT_FOUND`、`REF_UNRESOLVED`、`REF_SHADOWED`、`LEGACY_RESIDUE`

## 設定（互動式輔助工具）

以互動方式建立提供者與 SecretRef 變更、執行預檢，並選擇性套用：

```bash
openclaw secrets configure
openclaw secrets configure --plan-out /tmp/openclaw-secrets-plan.json
openclaw secrets configure --apply --yes
openclaw secrets configure --providers-only
openclaw secrets configure --skip-provider-setup
openclaw secrets configure --agent ops
openclaw secrets configure --json
```

流程：先設定提供者（新增／編輯／移除 `secrets.providers` 別名），再對應認證資訊（選取欄位、指派 `{source, provider, id}` 參照），接著執行預檢，並選擇性套用。

旗標：

- `--providers-only`：僅設定 `secrets.providers`，略過認證資訊對應
- `--skip-provider-setup`：略過提供者設定，將認證資訊對應至現有提供者
- `--agent <id>`：將 `auth-profiles.json` 目標探索與寫入範圍限定為單一代理程式儲存區
- `--allow-exec`：允許在預檢／套用期間執行 exec SecretRef 檢查（可能會執行提供者命令）

`--providers-only` 與 `--skip-provider-setup` 無法併用。

注意事項：

- 需要互動式 TTY。
- 以 `openclaw.json` 中含有機密資料的欄位，以及所選代理程式範圍的 `auth-profiles.json` 為目標；標準支援介面：[SecretRef 認證資訊介面](/zh-TW/reference/secretref-credential-surface)。
- 支援直接在選取器流程中建立新的 `auth-profiles.json` 對應。
- 在套用前執行預檢解析。
- 產生的計畫預設會啟用清除選項（`scrubEnv`、`scrubAuthProfilesForProviderTargets`、`scrubLegacyAuthJson`）。已清除的明文值無法透過套用還原。
- `--plan-out` 會拒絕建立 UTF-8 序列化形式超過 16 MiB（16,777,216 bytes）的計畫，與 `apply --from` 的輸入限制一致。
- 若未使用 `--apply`，命令列介面仍會在預檢後提示 `Apply this plan now?`。
- 使用 `--apply`（且未使用 `--yes`）時，命令列介面會額外提示不可逆移轉確認。
- `--json` 會印出計畫與預檢報告，但仍需要互動式 TTY。

### Exec 提供者安全性

Homebrew 安裝通常會在 `/opt/homebrew/bin/*` 下公開符號連結的二進位檔。只有受信任的套件管理器路徑需要時，才設定 `allowSymlinkCommand: true`，並搭配 `trustedDirs`（例如 `["/opt/homebrew"]`）。在 Windows 上，如果無法驗證提供者路徑的 ACL，OpenClaw 會採取失敗關閉；僅針對受信任的路徑，可在該提供者上設定 `allowInsecurePath: true`，以略過路徑安全性檢查。

## 套用已儲存的計畫

```bash
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --json
```

`--dry-run` 會驗證預檢而不寫入檔案；試執行預設會略過 exec SecretRef 檢查。除非使用 `--allow-exec`，否則寫入模式會拒絕包含 exec SecretRef／提供者的計畫。使用 `--allow-exec` 可在任一模式中選擇啟用 exec 提供者檢查／執行。

`--from` 必須指向不超過 16 MiB（16,777,216 bytes）的普通檔案。位元組限制適用於完整的序列化檔案，包括空白字元。

`apply` 可能更新的項目：

- `openclaw.json`（SecretRef 目標與提供者更新插入／刪除）
- `auth-profiles.json`（清除提供者目標）
- 舊版 `auth.json` 殘留項目
- 有效狀態與作用中設定目錄中的 `.env` 檔案，針對值已移轉的已知機密金鑰

計畫合約詳細資料（允許的目標路徑、驗證規則、失敗語意）：[機密資料套用計畫合約](/zh-TW/gateway/secrets-plan-contract)。

### 為何沒有復原備份

`secrets apply` 刻意不寫入包含舊明文值的復原備份。安全性來自嚴格的預檢與近似不可分割的套用，並在失敗時盡力於記憶體中還原。

## 範例

```bash
openclaw secrets audit --check
openclaw secrets configure
openclaw secrets audit --check
```

如果 `audit --check` 仍回報明文發現項目，請更新其餘回報的目標路徑，然後重新執行稽核。

## 相關內容

- [命令列介面參考](/zh-TW/cli)
- [機密資料管理](/zh-TW/gateway/secrets)
- [Vault SecretRef](/zh-TW/plugins/vault)
