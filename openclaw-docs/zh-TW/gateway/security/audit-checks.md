---
read_when:
    - 你在 `openclaw security audit` 輸出中看到特定的 `checkId`，並想知道它代表什麼意思
    - 你需要特定發現項目的修正鍵／路徑
    - 你正在對一次安全稽核執行中的嚴重程度進行分級處理
summary: openclaw security audit 發出的 checkId 參考目錄
title: 安全性稽核檢查
x-i18n:
    generated_at: "2026-07-26T08:34:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6de8cd97fdae79de2a29434a5e05521fb3e9af9805173df1ea48cf41e85a91b3
    source_path: gateway/security/audit-checks.md
    workflow: 16
---

`openclaw security audit` 會依 `checkId` 鍵值輸出結構化的發現項目。本
頁是這些 ID 的參考目錄。如需高階威脅模型
與強化指南，請參閱[安全性](/zh-TW/gateway/security)。

部分檢查僅會搭配 `openclaw security audit --deep` 執行：外掛／技能程式碼
掃描（`plugins.code_safety*`、`skills.code_safety*`）及即時閘道探測
檢查（`gateway.probe_*`）。此表中的其他所有項目皆會在一般的
`openclaw security audit` 上執行。

像 `warn/critical` 這樣的嚴重性，表示同一個 `checkId` 可能會依
設定而以任一層級輸出（例如閘道是否對遠端
開放）。在實際部署中最可能看到的高訊號值如下（並非
完整清單）：

| `checkId`                                                       | 嚴重性           | 重要性                                                                                  | 主要修正鍵值／路徑                                                                                      | 自動修正 |
| --------------------------------------------------------------- | ------------------ | --------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- | -------- |
| `fs.state_dir.perms_world_writable`                             | 嚴重           | 其他使用者／程序可修改完整的 OpenClaw 狀態                                    | `~/.openclaw` 的檔案系統權限                                                                       | 是      |
| `fs.state_dir.perms_group_writable`                             | 警告               | 群組使用者可修改完整的 OpenClaw 狀態                                              | `~/.openclaw` 的檔案系統權限                                                                       | 是      |
| `fs.state_dir.perms_readable`                                   | 警告               | 其他人可讀取狀態目錄                                                         | `~/.openclaw` 的檔案系統權限                                                                       | 是      |
| `fs.state_dir.symlink`                                          | 警告               | 狀態目錄的目標位置會成為另一個信任邊界                                         | 狀態目錄的檔案系統配置                                                                             | 否       |
| `fs.config.perms_writable`                                      | 嚴重           | 其他人可變更驗證／工具政策／設定                                               | `~/.openclaw/openclaw.json` 的檔案系統權限                                                         | 是      |
| `fs.config.symlink`                                             | 警告               | 不支援寫入符號連結的設定檔，且這會增加另一個信任邊界        | 替換為一般設定檔，或將 `OPENCLAW_CONFIG_PATH` 指向實際檔案                     | 否       |
| `fs.config.perms_group_readable`                                | 警告               | 群組使用者可讀取設定中的權杖／設定                                             | 設定檔的檔案系統權限                                                                         | 是      |
| `fs.config.perms_world_readable`                                | 嚴重           | 設定可能暴露權杖／設定                                                       | 設定檔的檔案系統權限                                                                         | 是      |
| `fs.config_include.perms_writable`                              | 嚴重           | 其他人可修改設定所納入的檔案                                           | `openclaw.json` 所參照納入檔案的權限                                                      | 是      |
| `fs.config_include.perms_group_readable`                        | 警告               | 群組使用者可讀取納入的機密／設定                                          | `openclaw.json` 所參照納入檔案的權限                                                      | 是      |
| `fs.config_include.perms_world_readable`                        | 嚴重           | 所有使用者皆可讀取納入的機密／設定                                            | `openclaw.json` 所參照納入檔案的權限                                                      | 是      |
| `fs.auth_profiles.perms_writable`                               | 嚴重           | 其他人可注入或替換已儲存的模型認證資訊                                   | `agents/<agentId>/agent/auth-profiles.json` 權限                                                       | 是      |
| `fs.auth_profiles.perms_readable`                               | 警告               | 其他人可讀取 API 金鑰和 OAuth 權杖                                               | `agents/<agentId>/agent/auth-profiles.json` 權限                                                       | 是      |
| `fs.credentials_dir.perms_writable`                             | 嚴重           | 其他人可修改頻道配對／認證資訊狀態                                      | `~/.openclaw/credentials` 的檔案系統權限                                                           | 是      |
| `fs.credentials_dir.perms_readable`                             | 警告               | 其他人可讀取頻道認證資訊狀態                                                | `~/.openclaw/credentials` 的檔案系統權限                                                           | 是      |
| `fs.sessions_store.perms_readable`                              | 警告               | 其他人可讀取工作階段逐字稿／中繼資料                                            | 工作階段儲存區權限                                                                                     | 是      |
| `fs.log_file.perms_readable`                                    | 警告               | 其他人可讀取已遮蔽但仍屬敏感的日誌                                       | 閘道日誌檔案權限                                                                                  | 是      |
| `fs.synced_dir`                                                 | 警告               | iCloud／Dropbox／Drive 中的狀態／設定會擴大權杖／逐字稿的暴露範圍                 | 將設定／狀態移出同步資料夾                                                                    | 否       |
| `gateway.bind_no_auth`                                          | 嚴重           | 遠端繫結未使用共用密鑰                                                       | `gateway.bind`、`gateway.auth.*`                                                                        | 否       |
| `gateway.loopback_no_auth`                                      | 嚴重           | 經反向代理的回送位址可能變為未經驗證                                     | `gateway.auth.*`、代理設定                                                                           | 否       |
| `gateway.trusted_proxies_missing`                               | 警告               | 存在反向代理標頭，但未受信任                                       | `gateway.trustedProxies`                                                                                | 否       |
| `gateway.http.no_auth`                                          | 警告／嚴重      | 可透過 `auth.mode="none"` 存取閘道 HTTP API                                     | `gateway.auth.mode`、`gateway.http.endpoints.*`、`plugins.entries.admin-http-rpc`                       | 否       |
| `gateway.http.session_key_override_enabled`                     | 資訊               | HTTP API 呼叫端可覆寫 `sessionKey`                                              | `gateway.http.allowSessionKeyOverride`                                                                  | 否       |
| `gateway.tools_invoke_http.dangerous_allow`                     | 警告／嚴重      | 為擁有者／管理員呼叫端重新啟用透過 HTTP API 使用的危險工具                        | `gateway.tools.allow`                                                                                   | 否       |
| `gateway.nodes.allow_commands_dangerous`                        | 警告／嚴重      | 啟用高影響力的節點命令（桌面輸入／相機／螢幕／聯絡人／行事曆／SMS）   | `gateway.nodes.commands.allow`                                                                          | 否       |
| `gateway.nodes.deny_commands_ineffective`                       | 警告               | 類似模式的拒絕項目不會比對 Shell 文字或群組                             | `gateway.nodes.commands.deny`                                                                           | 否       |
| `gateway.tailscale_funnel`                                      | 嚴重           | 暴露於公用網際網路                                                                | `gateway.tailscale.mode`                                                                                | 否       |
| `gateway.tailscale_serve`                                       | 資訊               | 已透過 Serve 啟用 Tailnet 暴露                                                   | `gateway.tailscale.mode`                                                                                | 否       |
| `gateway.control_ui.allowed_origins_required`                   | 嚴重           | 非回送位址的控制介面未明確設定瀏覽器來源允許清單                       | `gateway.controlUi.allowedOrigins`                                                                      | 否       |
| `gateway.control_ui.allowed_origins_wildcard`                   | 警告／嚴重      | `allowedOrigins=["*"]` 會停用瀏覽器來源允許清單                             | `gateway.controlUi.allowedOrigins`                                                                      | 否       |
| `gateway.control_ui.host_header_origin_fallback`                | 警告／嚴重      | 啟用 Host 標頭來源後援機制（降低 DNS 重新繫結防護）                 | `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback`                                            | 否       |
| `gateway.control_ui.insecure_auth`                              | 警告               | 已啟用不安全驗證相容性切換                                              | `gateway.controlUi.allowInsecureAuth`                                                                   | 否       |
| `gateway.control_ui.device_auth_disabled`                       | 嚴重           | 已淘汰的裝置驗證略過遷移仍允許修復存取                   | 完成 **Secure this browser** 配對（`gateway.controlUi.deviceAuthMigration` 狀態）                | 否       |
| `gateway.real_ip_fallback_enabled`                              | 警告／嚴重      | 信任 `X-Real-IP` 後援機制，可能因代理設定錯誤而導致來源 IP 偽造         | `gateway.allowRealIpFallback`、`gateway.trustedProxies`                                                 | 否       |
| `gateway.token_too_short`                                       | 警告               | 較短的共用權杖更容易遭暴力破解                                             | `gateway.auth.token`                                                                                    | 否       |
| `gateway.auth_no_rate_limit`                                    | 警告               | 暴露的驗證機制若未限制速率，會提高暴力破解風險                           | `gateway.auth.rateLimit`                                                                                | 否       |
| `gateway.trusted_proxy_auth`                                    | 嚴重           | 代理身分現在會成為驗證邊界                                            | `gateway.auth.mode="trusted-proxy"`                                                                     | 否       |
| `gateway.trusted_proxy_no_proxies`                              | 嚴重           | 受信任代理驗證若未設定受信任的代理 IP，便不安全                                  | `gateway.trustedProxies`                                                                                | 否       |
| `gateway.trusted_proxy_no_user_header`                          | 嚴重           | 受信任代理驗證無法安全地解析使用者身分                                  | `gateway.auth.trustedProxy.userHeader`                                                                  | 否       |
| `gateway.trusted_proxy_no_allowlist`                            | 警告               | 受信任代理驗證接受上游任何已驗證的使用者                              | `gateway.auth.trustedProxy.allowUsers`                                                                  | 否       |
| `gateway.trusted_proxy_allow_loopback`                          | 警告               | 受信任代理驗證接受明確允許的回送代理來源                    | `gateway.auth.trustedProxy.allowLoopback`                                                               | 否       |
| `gateway.probe_auth_secretref_unavailable`                      | 警告               | 深度探查無法在此命令路徑中解析驗證 SecretRef                       | 深度探查驗證來源／SecretRef 可用性                                                         | 否       |
| `gateway.probe_failed`                                          | 警告               | 即時閘道探查失敗（僅 `--deep`）                                               | 閘道連線能力／驗證                                                                               | 否       |
| `discovery.mdns_full_mode`                                      | 警告／嚴重      | mDNS 完整模式會在區域網路上公告 `cliPath`/`sshPort` 中繼資料                 | `discovery.mdns.mode`, `gateway.bind`                                                                   | 否       |
| `config.insecure_or_dangerous_flags`                            | 警告               | 已啟用一個不安全／危險的偵錯旗標                                            | 調查結果詳細資料中指定的鍵                                                                             | 否       |
| `security.audit.suppressions.active`                            | 資訊               | 稽核輸出已設定抑制項目，可能會遭到篩選                            | `security.audit.suppressions`                                                                           | 否       |
| `config.secrets.gateway_password_in_config`                     | 警告               | 閘道密碼直接儲存在設定中                                           | `gateway.auth.password`                                                                                 | 否       |
| `config.secrets.hooks_token_in_config`                          | 警告               | 鉤子持有者權杖直接儲存在設定中                                          | `hooks.token`                                                                                           | 否       |
| `hooks.token_reuse_gateway_token`                               | 嚴重           | 鉤子輸入權杖也能解鎖閘道驗證                                            | `hooks.token`, `gateway.auth.token`, `gateway.auth.password`                                            | 否       |
| `hooks.token_too_short`                                         | 警告               | 鉤子輸入更容易遭到暴力破解                                                      | `hooks.token`                                                                                           | 否       |
| `hooks.default_session_key_unset`                               | 警告               | 鉤子代理程式執行會分散到針對每個請求產生的工作階段                             | `hooks.defaultSessionKey`                                                                               | 否       |
| `hooks.allowed_agent_ids_unrestricted`                          | 警告／嚴重      | 已驗證的鉤子呼叫端可將請求路由至任何已設定的代理程式                            | `hooks.allowedAgentIds`                                                                                 | 否       |
| `hooks.request_session_key_enabled`                             | 警告／嚴重      | 外部呼叫端可選擇 sessionKey                                                   | `hooks.allowRequestSessionKey`                                                                          | 否       |
| `hooks.request_session_key_prefixes_missing`                    | 警告／嚴重      | 外部工作階段金鑰的格式不受限制                                                 | `hooks.allowedSessionKeyPrefixes`                                                                       | 否       |
| `hooks.path_root`                                               | 嚴重           | 鉤子路徑為 `/`，使輸入更容易發生衝突或錯誤路由                          | `hooks.path`                                                                                            | 否       |
| `hooks.installs_unpinned_npm_specs`                             | 警告               | 鉤子安裝記錄未固定至不可變的 npm 規格                              | 鉤子安裝中繼資料                                                                                   | 否       |
| `hooks.installs_missing_integrity`                              | 警告               | 鉤子安裝記錄缺少完整性中繼資料                                            | 鉤子安裝中繼資料                                                                                   | 否       |
| `hooks.installs_version_drift`                                  | 警告               | 鉤子安裝記錄與已安裝的套件不一致                                      | 鉤子安裝中繼資料                                                                                   | 否       |
| `logging.redact_off`                                            | 警告               | 敏感值洩漏至記錄／狀態                                                    | `logging.redactSensitive`                                                                               | 是      |
| `browser.control_invalid_config`                                | 警告               | 瀏覽器控制設定在執行階段前即為無效                                        | `browser.*`                                                                                             | 否       |
| `browser.control_no_auth`                                       | 嚴重           | 瀏覽器控制在沒有權杖／密碼驗證的情況下公開                                     | `gateway.auth.*`                                                                                        | 否       |
| `browser.remote_cdp_http`                                       | 警告               | 透過純 HTTP 的遠端 CDP 缺少傳輸加密                                   | 瀏覽器設定檔 `cdpUrl`                                                                                | 否       |
| `browser.remote_cdp_private_host`                               | 警告               | 遠端 CDP 的目標是私人／內部主機                                              | 瀏覽器設定檔 `cdpUrl`, `browser.ssrfPolicy.*`                                                        | 否       |
| `sandbox.docker_config_mode_off`                                | 警告               | 沙箱 Docker 設定存在但未啟用                                              | `agents.*.sandbox.mode`                                                                                 | 否       |
| `sandbox.bind_mount_non_absolute`                               | 警告               | 相對繫結掛載的解析結果可能無法預測                                          | `agents.*.sandbox.docker.binds[]`                                                                       | 否       |
| `sandbox.dangerous_bind_mount`                                  | 嚴重           | 沙箱繫結掛載的目標是遭封鎖的系統、認證資訊或 Docker 通訊端路徑           | `agents.*.sandbox.docker.binds[]`                                                                       | 否       |
| `sandbox.dangerous_network_mode`                                | 嚴重           | 沙箱 Docker 網路使用 `host` 或 `container:*` 命名空間加入模式                 | `agents.*.sandbox.docker.network`                                                                       | 否       |
| `sandbox.dangerous_seccomp_profile`                             | 嚴重           | 沙箱 seccomp 設定檔削弱容器隔離                                     | `agents.*.sandbox.docker.securityOpt`                                                                   | 否       |
| `sandbox.dangerous_apparmor_profile`                            | 嚴重           | 沙箱 AppArmor 設定檔削弱容器隔離                                    | `agents.*.sandbox.docker.securityOpt`                                                                   | 否       |
| `sandbox.browser_cdp_bridge_unrestricted`                       | 警告               | 沙箱瀏覽器橋接在未限制來源範圍的情況下公開                      | `sandbox.browser.cdpSourceRange`                                                                        | 否       |
| `sandbox.browser_container.non_loopback_publish`                | 嚴重           | 現有瀏覽器容器將 CDP 發布於非迴路介面                     | 瀏覽器沙箱容器發布設定                                                                | 否       |
| `sandbox.browser_container.hash_label_missing`                  | 警告               | 現有瀏覽器容器早於目前的設定雜湊標籤                          | `openclaw sandbox recreate --browser --all`                                                             | 否       |
| `sandbox.browser_container.hash_epoch_stale`                    | 警告               | 現有瀏覽器容器早於目前的瀏覽器設定時期                        | `openclaw sandbox recreate --browser --all`                                                             | 否       |
| `sandbox.browser_container.docker_probe_timeout`                | 警告               | 瀏覽器容器的 Docker 標籤探查逾時                                  | Docker 常駐程式連線能力                                                                              | 否       |
| `tools.exec.host_sandbox_no_sandbox_defaults`                   | 警告               | 沙箱關閉時，`exec host=sandbox` 會以封閉方式失敗                                    | `tools.exec.host`, `agents.defaults.sandbox.mode`                                                       | 否       |
| `tools.exec.host_sandbox_no_sandbox_agents`                     | 警告               | 沙箱關閉時，每個代理程式的 `exec host=sandbox` 會以封閉方式失敗                          | `agents.entries.*.tools.exec.host`, `agents.entries.*.sandbox.mode`                                     | 否       |
| `tools.exec.security_full_configured`                           | 警告／嚴重      | 主機執行正以 `security="full"` 運作                                             | `tools.exec.security`, `agents.entries.*.tools.exec.security`                                           | 否       |
| `tools.exec.agent_skill_mcp_boundary_drift`                     | 警告               | 當主機執行可存取 MCP 用戶端／登錄檔時，存在代理程式技能允許清單     | `agents.entries.*.tools.exec.*`、沙箱／作業系統隔離、MCP 伺服器認證資訊                           | 否       |
| `tools.exec.fs_tools_disabled_but_exec_enabled`                 | 警告               | 檔案系統工具原則無法使殼層執行成為唯讀                          | `tools.deny`, `agents.entries.*.tools.deny`, `agents.*.sandbox.workspaceAccess`                         | 否       |
| `tools.exec.auto_allow_skills_enabled`                          | 警告               | 執行核准會隱含信任技能二進位檔                                              | 主機核准檔案                                                                                     | 否       |
| `tools.exec.allowlist_interpreter_without_strict_inline_eval`   | 警告               | 直譯器允許清單允許內嵌求值，且不會強制重新核准                     | `tools.exec.strictInlineEval`, `agents.entries.*.tools.exec.strictInlineEval`、執行核准允許清單 | 否       |
| `tools.exec.safe_bins_interpreter_unprofiled`                   | 警告               | `safeBins` 中缺少明確設定檔的直譯器／執行階段二進位檔會擴大執行風險      | `tools.exec.safeBins`, `tools.exec.safeBinProfiles`, `agents.entries.*.tools.exec.*`                    | 否       |
| `tools.exec.safe_bins_broad_behavior`                           | 警告               | `safeBins` 中行為範圍廣泛的工具會削弱低風險 stdin 篩選信任模型         | `tools.exec.safeBins`, `agents.entries.*.tools.exec.safeBins`                                           | 否       |
| `tools.exec.safe_bin_trusted_dirs_risky`                        | 警告               | `safeBinTrustedDirs` 包含可變或有風險的目錄                              | `tools.exec.safeBinTrustedDirs`, `agents.entries.*.tools.exec.safeBinTrustedDirs`                       | 否       |
| `tools.elevated.allowFrom.<provider>.wildcard`                  | 嚴重           | `tools.elevated.allowFrom.<provider>` 包含 `"*"`，因而核准所有傳送者            | `tools.elevated.allowFrom.<provider>`                                                                   | 否       |
| `tools.elevated.allowFrom.<provider>.large`                     | 警告               | `<provider>` 的提升權限允許清單超過 25 個項目                            | `tools.elevated.allowFrom.<provider>`                                                                   | 否       |
| `skills.workspace.symlink_escape`                               | 警告               | 工作區 `skills/**/SKILL.md` 解析至工作區根目錄之外（符號連結鏈偏移）    | 工作區 `skills/**` 的檔案系統狀態                                                                  | 否       |
| `skills.workspace.scan_truncated`                               | 警告               | 工作區 Skill 掃描在完成前已達目錄巡訪上限                       | 扁平化／簡化工作區 `skills/` 的目錄樹                                                 | 否       |
| `plugins.extensions_no_allowlist`                               | 警告               | 安裝外掛時未明確設定外掛允許清單                              | `plugins.allowlist`                                                                                     | 否       |
| `plugins.allow_phantom_entries`                                 | 警告               | `plugins.allow` 列出的 ID 沒有相符的已安裝外掛                           | `plugins.allow`                                                                                         | 否       |
| `plugins.installs_unpinned_npm_specs`                           | 警告               | 外掛索引記錄未固定至不可變的 npm 規格                              | 外掛安裝中繼資料                                                                                 | 否       |
| `plugins.installs_missing_integrity`                            | 警告               | 外掛索引記錄缺少完整性中繼資料                                            | 外掛安裝中繼資料                                                                                 | 否       |
| `plugins.installs_version_drift`                                | 警告               | 外掛索引記錄與已安裝套件不一致                                      | 外掛安裝中繼資料                                                                                 | 否       |
| `plugins.code_safety`                                           | 警告／重大      | 外掛程式碼掃描發現可疑或危險模式（僅限 `--deep`）                 | 外掛程式碼／安裝來源                                                                            | 否       |
| `plugins.code_safety.entry_path`                                | 警告               | 外掛進入點路徑指向隱藏或 `node_modules` 位置                        | 外掛資訊清單 `entry`                                                                                 | 否       |
| `plugins.code_safety.entry_escape`                              | 重大           | 外掛進入點逸出外掛目錄                                               | 外掛資訊清單 `entry`                                                                                 | 否       |
| `plugins.code_safety.manifest_parse_error`                      | 警告               | 在程式碼安全掃描期間無法剖析外掛資訊清單                         | 外掛資訊清單檔案                                                                                    | 否       |
| `plugins.code_safety.scan_failed`                               | 警告               | 外掛程式碼掃描無法完成（僅限 `--deep`）                                     | 外掛路徑／掃描環境                                                                          | 否       |
| `plugins.<pluginId>.security_audit_failed`                      | 警告               | 外掛所擁有的安全稽核收集器擲回錯誤                                  | 該外掛的安全稽核收集器                                                                  | 否       |
| `skills.code_safety`                                            | 警告／重大      | Skill 安裝程式中繼資料／程式碼包含可疑或危險模式（僅限 `--deep`） | Skill 安裝來源                                                                                    | 否       |
| `skills.code_safety.scan_failed`                                | 警告               | Skill 程式碼掃描無法完成（僅限 `--deep`）                                      | Skill 掃描環境                                                                                  | 否       |
| `channels.discord.allowlisted_groups.broad_members`             | 警告               | 允許清單中的 Discord 伺服器／頻道目標沒有成員或角色限制            | `channels.discord.guilds.*.users/roles`、各頻道的 `users/roles`                                      | 否       |
| `security.exposure.open_channels_with_exec`                     | 警告／重大      | 共用／公開聊天室可存取啟用執行功能的代理                                       | `channels.*.dmPolicy`、`channels.*.groupPolicy`、`tools.exec.*`、`agents.entries.*.tools.exec.*`        | 否       |
| `security.exposure.open_groups_with_elevated`                   | 重大           | 開放的私訊／群組加上提升權限工具，會形成高影響的提示注入路徑              | 頂層或巢狀私訊政策路徑、帳號覆寫、`channels.*.groupPolicy`                        | 否       |
| `security.exposure.open_groups_with_runtime_or_fs`              | 重大／警告      | 開放的私訊／群組可在沒有沙箱／工作區防護的情況下存取命令／檔案工具           | 私訊／群組政策路徑、`tools.profile/deny`、`tools.fs.workspaceOnly`、`agents.*.sandbox.mode`          | 否       |
| `security.exposure.open_groups_with_control_plane_tools`        | 重大           | 開放的私訊／群組可存取閘道／排程控制平面工具                              | 私訊／群組政策路徑、`tools.allow`、`tools.alsoAllow`、`tools.profile`、`gateway`、`cron`             | 否       |
| `security.trust_model.multi_user_heuristic`                     | 警告               | 設定看似為多使用者，但閘道信任模型是個人助理                 | 分隔信任邊界，或強化共用使用者安全性（`sandbox.mode`、工具拒絕／工作區範圍限制）          | 否       |
| `tools.profile_minimal_overridden`                              | 警告               | 代理覆寫會略過全域最小設定檔                                           | `agents.entries.*.tools.profile`                                                                        | 否       |
| `plugins.tools_reachable_permissive_policy`                     | 警告               | 在寬鬆的情境中可存取擴充功能工具                                        | `tools.profile` + 工具允許／拒絕                                                                       | 否       |
| `models.legacy`                                                 | 警告               | 仍設定了舊版模型系列                                              | 模型選擇                                                                                         | 否       |
| `models.weak_tier`                                              | 警告               | 已設定的模型低於目前建議的層級                                   | 模型選擇                                                                                         | 否       |
| `models.small_params`                                           | 重大／資訊      | 小型模型加上不安全的工具介面會提高注入風險                                | 模型選擇 + 沙箱／工具政策                                                                      | 否       |
| `channels.<provider>.dm.open`                                   | 重大           | `<provider>` 私訊政策為 `"open"`；任何人都能私訊機器人                               | `channels.<provider>.dmPolicy`、`.allowFrom`                                                            | 否       |
| `channels.<provider>.dm.open_invalid`                           | 警告               | `allowFrom` 中有 `dmPolicy="open"` 卻沒有 `"*"`，設定不一致                          | `channels.<provider>.allowFrom`                                                                         | 否       |
| `channels.<provider>.dm.scope_main_multiuser`                   | 警告               | 多個私訊傳送者目前共用主要工作階段                                    | `session.dmScope`                                                                                       | 否       |
| `channels.<provider>.allowFrom.dangerous_name_matching_enabled` | 資訊               | `dangerouslyAllowNameMatching` 會重新啟用可變的名稱／電子郵件／標籤傳送者比對        | 停用 `dangerouslyAllowNameMatching`，改用穩定的傳送者 ID                                           | 否       |
| `channels.<provider>.account.read_only_resolution`              | 警告               | 無法完整解析頻道帳號以進行稽核（缺少密鑰／閘道）        | 確保可解析所參照的密鑰，或針對即時閘道快照執行                        | 否       |
| `channels.<provider>.warning.<n>`                               | 資訊／警告／重大 | 依據外掛的自由格式文字分類之供應商特定安全警告               | 請參閱發現項目詳細資料                                                                                      | 否       |
| `summary.attack_surface`                                        | 資訊               | 認證、頻道、工具及暴露狀態的彙總摘要                            | 多個鍵（請參閱發現項目詳細資料）                                                                      | 否       |

`channels.<provider>.*` 和 `tools.elevated.allowFrom.<provider>.*` checkId 會依每個已設定的頻道／提供者產生，因此在實際輸出中，`<provider>` 是真正的頻道 ID
（例如 `telegram`、`discord`），而不是常值字串。

## 相關內容

- [安全性](/zh-TW/gateway/security)
- [設定](/zh-TW/gateway/configuration)
- [受信任的 Proxy 驗證](/zh-TW/gateway/trusted-proxy-auth)
