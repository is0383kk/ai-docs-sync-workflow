---
read_when:
    - 你想從命令列介面編輯 exec 核准設定
    - 你需要在閘道或節點主機上管理允許清單
    - 你需要在沒有聊天介面的情況下列出或處理待核准項目
summary: '`openclaw approvals` 和 `openclaw exec-policy` 的命令列介面參考資料'
title: 核准
x-i18n:
    generated_at: "2026-07-26T07:35:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f8b6f198af718d7b058498dbb960a1eb68ced601e1cd9205070b7199688552d2
    source_path: cli/approvals.md
    workflow: 16
---

# `openclaw approvals`

管理**本機主機**、**閘道主機**或**節點主機**的 exec 核准。未指定目標旗標時，命令會讀寫磁碟上的本機核准檔案。使用 `--gateway` 指定閘道，或使用 `--node <id|name|ip>` 指定特定節點。

別名：`openclaw exec-approvals`

相關內容：[Exec 核准](/zh-TW/tools/exec-approvals)、[節點](/zh-TW/nodes)

## `openclaw exec-policy`

`openclaw exec-policy` 是**僅限本機**的便利命令，可在單一步驟中讓要求的 `tools.exec.*` 設定與本機主機核准檔案保持同步：

```bash
openclaw exec-policy show
openclaw exec-policy show --json

openclaw exec-policy preset yolo
openclaw exec-policy preset cautious --json

openclaw exec-policy set --host gateway --security full --ask off --ask-fallback full
```

預設集（`yolo`、`cautious`、`deny-all`）會一起套用 `host`、`security`、`ask` 和 `askFallback`。`set` 只會套用你傳入的旗標；每個接受的值都會經過驗證（`--host auto|sandbox|gateway|node`、`--security deny|allowlist|full`、`--ask off|on-miss|always`、`--ask-fallback deny|allowlist|full`）。

範圍：

- 同時更新本機設定檔和本機核准檔案；不會將原則推送至閘道或節點主機。
- `--host node` 會遭到拒絕：節點 exec 核准是在執行階段從節點擷取，因此本機 `exec-policy` 無法將其同步。請改用 `openclaw approvals set --node <id|name|ip>`。
- `exec-policy show` 會在執行階段將 `host=node` 範圍標記為由節點管理，而不是從本機核准檔案推導有效原則。

若要管理遠端主機核准，請直接使用 `openclaw approvals set --gateway` 或 `openclaw approvals set --node <id|name|ip>`。

## 常用命令

```bash
openclaw approvals get
openclaw approvals get --node <id|name|ip>
openclaw approvals get --gateway
openclaw approvals pending
openclaw approvals resolve <id> <allow-once|allow-always|deny>
```

`get` 會顯示目標的有效 exec 原則：要求的 `tools.exec` 原則、主機核准檔案原則，以及合併後的有效結果。具有主機原生原則的節點（例如 Windows 伴隨應用程式）會直接顯示該原則，而不套用 OpenClaw 核准檔案的原則計算方式。

對於以檔案為基礎的節點，合併檢視需要由主機解析的原則快照。較舊的節點會將有效原則顯示為無法取得，而不會假設閘道要求的原則也適用於主機。

<Note>
不包含每個工作階段的 `/exec` 覆寫。請在相關工作階段中執行 `/exec`，以檢查其目前的預設值。
</Note>

優先順序：

- 主機核准檔案是可強制執行的單一真實來源。
- 要求的 `tools.exec` 原則可以縮小或擴大意圖範圍，但有效結果是從主機規則推導而來。
- `--node` 會將節點主機核准檔案與閘道 `tools.exec` 原則結合（兩者都會在執行階段套用）。
- 如果無法取得閘道設定，命令列介面會退回使用節點核准快照，並註明無法計算最終執行階段原則。

## 待處理的核准

列出閘道中待處理的 exec、外掛和 OpenClaw 系統代理程式核准：

```bash
openclaw approvals pending
openclaw approvals pending --json
```

完整列舉以及相符的全操作員 `resolve` 流程會使用 `operator.admin`，因為核准記錄在其他情況下會保留要求者／審查者篩選。解析時也會要求專用的 `operator.approvals` 範圍。標準命令列介面操作員授權包含這兩個範圍；受限的第三方用戶端不應只為了模擬此命令而要求管理員權限。

人類可讀輸出會顯示核准種類、代理程式／工作階段歸屬、要求經過時間、距離到期的時間、縮短的命令或摘要，以及與 shell 無關的 `id64_<base64url>` ID 權杖。精簡表格後一律會接著顯示 `Full request text` 區塊，其中包含每個完整權杖和以無損方式逸出的要求，因此終端機寬度造成的縮短不會隱藏尾碼或解析所需的權杖。請將完整權杖複製到 `resolve`。其他欄位中的不安全終端機字元會顯示為可見的 Unicode 逸出序列。JSON 輸出會在 `approvals` 下傳回正規化項目，並為指令碼保留原始的 `id`、`summary`、`createdAtMs` 和 `expiresAtMs`；除非原始 ID 使用保留的 `id64_` 顯示權杖前綴，否則 `resolve` 仍接受原始 ID。

如果提供的 `id64_` 值同時符合某個常值原始 ID，以及另一項核准經解碼後的顯示權杖，命令列介面會因其含意不明而拒絕該值，以免誤解析錯誤的要求。

使用完整 ID 解析一項核准：

```bash
openclaw approvals resolve <id> allow-once
openclaw approvals resolve <id> allow-always
openclaw approvals resolve <id> deny --reason "Not expected during maintenance"
```

命令列介面會讀取統一核准記錄以選取其種類、依照記錄允許的決定檢查要求的決定，然後呼叫統一解析器。第一次成功決定會以 `0` 結束。重複已記錄的決定也會以 `0` 結束，並回報 `already resolved (same decision)`。互相衝突的決定、缺少的核准、已過期的核准，或該核准種類不支援的決定，都會輸出清楚的錯誤並以非零狀態結束。

`--reason` 會在命令列介面確認訊息中新增本機備註。目前的閘道核准記錄沒有自由文字的解析原因欄位，因此此備註不會持久保存，也不會傳送至其他核准介面。

## 從檔案取代核准

```bash
openclaw approvals set --file ./exec-approvals.json
openclaw approvals set --stdin <<'EOF'
{ version: 1, defaults: { security: "full", ask: "off", askFallback: "full" } }
EOF
openclaw approvals set --node <id|name|ip> --file ./exec-approvals.json
openclaw approvals set --gateway --file ./exec-approvals.json
```

`set` 接受 JSON5，不限於嚴格 JSON。請使用 `--file` 或 `--stdin`，不要同時使用兩者。

主機原生 Windows 節點使用自己的原則格式：

```bash
openclaw approvals set --node <id|name|ip> --stdin <<'EOF'
{
  defaultAction: "deny",
  rules: [{ pattern: "hostname", action: "allow" }]
}
EOF
```

命令列介面會先讀取節點目前的雜湊值，並在更新時一併傳送，因此同時進行的本機編輯會遭到拒絕，而不會被覆寫。由於此操作會取代節點的完整規則清單，因此必須提供 `rules`；`defaultAction` 則為選用。如果節點回報其原生原則已停用，就無法從遠端設定；請先在該主機上啟用或設定原則。主機原生原則不支援 `allowlist add|remove` 輔助命令。

## 「永不提示」／YOLO 範例

若主機不應因 exec 核准而停止，請將其主機核准預設值設為 `full` + `off`：

```bash
openclaw approvals set --stdin <<'EOF'
{
  version: 1,
  defaults: {
    security: "full",
    ask: "off",
    askFallback: "full"
  }
}
EOF
```

對於公開 OpenClaw 核准檔案的節點，請搭配 `openclaw approvals set --node <id|name|ip> --stdin` 使用相同內容。主機原生節點需要使用上方所示的擁有者專屬格式。

這只會變更**主機核准檔案**。若要讓要求的 OpenClaw 原則保持一致，也請設定：

```bash
openclaw config set tools.exec.host gateway
openclaw config set tools.exec.mode full
```

此處明確指定 `tools.exec.host=gateway`，因為 `host=auto` 仍表示「可用時使用沙箱，否則使用閘道」：YOLO 關乎核准，而非路由。即使已設定沙箱，若仍要使用主機 exec，請使用 `gateway`（或 `/exec host=gateway`）。

省略 `askFallback` 時，預設為 `deny`。升級應維持永不提示行為的無 UI 主機時，請明確設定 `askFallback: "full"`。

僅在本機上表達相同意圖的本機捷徑：

```bash
openclaw exec-policy preset yolo
```

## 允許清單輔助命令

```bash
openclaw approvals allowlist add "~/Projects/**/bin/rg"
openclaw approvals allowlist add --agent main --node <id|name|ip> "/usr/bin/uptime"
openclaw approvals allowlist add --agent "*" "/usr/bin/uname"

openclaw approvals allowlist remove "~/Projects/**/bin/rg"
```

## 常用選項

`get`、`set` 和 `allowlist add|remove` 均支援：

- `--node <id|name|ip>`（解析 ID、名稱、IP 或 ID 前綴；使用與 `openclaw nodes` 相同的解析器）
- `--gateway`
- 共用節點 RPC 選項：`--url`、`--token`、`--timeout`、`--json`

未指定目標旗標時，表示磁碟上的本機核准檔案。

`allowlist add|remove` 也支援 `--agent <id>`（預設為 `"*"`，套用至所有代理程式）。

`pending` 和 `resolve` 一律使用閘道，因為待處理要求是即時閘道狀態。它們支援共用的閘道連線選項 `--url`、`--token` 和 `--timeout`；`pending` 也支援 `--json`。

## 注意事項

- 節點主機必須公告 `system.execApprovals.get/set`（macOS 應用程式、無頭節點主機或 Windows 伴隨應用程式）。
- 核准檔案會依主機儲存在 OpenClaw 狀態目錄中：`$OPENCLAW_STATE_DIR/exec-approvals.json`；若未設定該變數，則為 `~/.openclaw/exec-approvals.json`。

## 相關內容

- [命令列介面參考](/zh-TW/cli)
- [Exec 核准](/zh-TW/tools/exec-approvals)
