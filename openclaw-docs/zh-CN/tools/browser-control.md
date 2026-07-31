---
read_when:
    - 通过本地控制 API 编写脚本或调试智能体浏览器
    - 正在查找 `openclaw browser` CLI 参考文档
    - 添加使用快照和引用的自定义浏览器自动化
summary: OpenClaw 浏览器控制 API、CLI 参考和脚本操作
title: 浏览器控制 API
x-i18n:
    generated_at: "2026-07-26T07:01:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 812358a5ad366e419413b78507d3620ea9f3981224bc8cc62fb512b87eaadd9b
    source_path: tools/browser-control.md
    workflow: 16
---

有关设置、配置和故障排除，请参阅[浏览器](/zh-CN/tools/browser)。
本页是本地控制 HTTP API、`openclaw browser`
CLI 和脚本编写模式（快照、引用、等待、调试流程）的参考文档。

## 控制 API（可选）

Gateway 网关仅为本地集成公开一个小型 loopback HTTP API。
这个独立服务器需要主动启用——在 Gateway 网关服务环境中设置环境变量
`OPENCLAW_EAGER_BROWSER_CONTROL_SERVER=1`
并重启 Gateway 网关，HTTP 端点才会可用。如果没有
此变量，浏览器控制运行时仍可通过 CLI 和
智能体工具运行，但 loopback 控制端口上不会有任何监听服务。

- 状态/启动/停止：`GET /`、`GET /doctor`、`POST /start`、`POST /stop`、`POST /reset-profile`
- 配置文件：`GET /profiles`、`POST /profiles/create`、`DELETE /profiles/:name`
- 标签页：`GET /tabs`、`POST /tabs/open`、`POST /tabs/focus`、`DELETE /tabs/:targetId`、`POST /tabs/action`
- 快照/屏幕截图：`GET /snapshot`、`POST /screenshot`
- 操作：`POST /navigate`、`POST /act`
- 钩子：`POST /hooks/file-chooser`、`POST /hooks/dialog`
- 下载：`POST /download`、`POST /wait/download`
- 权限：`POST /permissions/grant`
- 调试：`GET /console`、`POST /pdf`
- 调试：`GET /errors`、`GET /requests`、`GET /dialogs`、`POST /trace/start`、`POST /trace/stop`、`POST /highlight`
- 网络：`POST /response/body`
- 状态：`GET /cookies`、`POST /cookies/set`、`POST /cookies/clear`
- 状态：`GET /storage/:kind`、`POST /storage/:kind/set`、`POST /storage/:kind/clear`
- 设置：`POST /set/offline`、`POST /set/headers`、`POST /set/credentials`、`POST /set/geolocation`、`POST /set/media`、`POST /set/timezone`、`POST /set/locale`、`POST /set/device`

`POST /tabs/action` 是 CLI 内部用于
`browser tab` 子命令（`{"action":"new"|"label"|"select"|"close"|"list", ...}`）的批处理形式；
直接编写脚本时，优先使用上面用途单一的标签页路由。

所有端点都接受 `?profile=<name>`。`POST /start?headless=true` 会为本地托管配置文件请求
一次性无头启动，而不会更改持久化的
浏览器配置；仅附加、远程 CDP 和现有会话配置文件会拒绝
此覆盖，因为 OpenClaw 不会启动这些浏览器进程。

对于标签页端点，`targetId` 是兼容性字段名。优先传入来自
`GET /tabs` 或 `POST /tabs/open` 的 `suggestedTargetId`；也接受标签和 `tabId`
句柄，例如 `t1`。原始 CDP 目标 ID 和唯一的原始
目标 ID 前缀仍然有效，但它们是易变的诊断句柄。

如果配置了共享密钥 Gateway 网关身份验证，浏览器 HTTP 路由也需要身份验证：

- `Authorization: Bearer <gateway token>`
- `x-openclaw-password: <gateway password>`，或使用该密码的 HTTP Basic 身份验证

注意：

- 这个独立的 loopback 浏览器 API **不会**使用受信任代理或
  Tailscale Serve 身份标头。
- 如果 `gateway.auth.mode` 为 `none` 或 `trusted-proxy`，这些 loopback 浏览器
  路由不会继承这些携带身份信息的模式；请确保它们仅用于 loopback。

### `/act` 错误约定

`POST /act` 对路由级验证和
策略失败使用结构化错误响应：

```json
{ "error": "<message>", "code": "ACT_*" }
```

当前的 `code` 值：

- `ACT_KIND_REQUIRED`（HTTP 400）：`kind` 缺失或无法识别。
- `ACT_INVALID_REQUEST`（HTTP 400）：操作载荷未通过规范化或验证。
- `ACT_SELECTOR_UNSUPPORTED`（HTTP 400）：`selector` 与不受支持的操作类型一起使用。
- `ACT_EVALUATE_DISABLED`（HTTP 403）：`evaluate`（或 `wait --fn`）已被配置禁用。
- `ACT_TARGET_ID_MISMATCH`（HTTP 403）：顶层或批处理的 `targetId` 与请求目标冲突。
- `ACT_EXISTING_SESSION_UNSUPPORTED`（HTTP 501）：现有会话配置文件不支持此操作。

其他运行时失败仍可能返回不含
`code` 字段的 `{ "error": "<message>" }`。

### Playwright 要求

某些功能（导航/操作/AI 快照/角色快照、元素屏幕截图、
PDF）需要 Playwright。如果未安装 Playwright，这些端点会返回
明确的 501 错误。

没有 Playwright 时仍可使用：

- ARIA 快照
- 当每个标签页的 CDP WebSocket 可用时，可使用角色样式的无障碍快照（`--interactive`、`--compact`、
  `--depth`、`--efficient`）。这是用于检查和发现引用的
  后备方案；Playwright 仍是主要的操作引擎。
- 当每个标签页的 CDP
  WebSocket 可用时，可为托管的 `openclaw` 浏览器生成页面屏幕截图
- 为 `existing-session` / Chrome MCP 配置文件生成页面屏幕截图
- 从快照输出生成基于 `existing-session` 引用的屏幕截图（`--ref`）

仍需要 Playwright 的功能：

- `navigate`
- `act`
- 依赖 Playwright 原生 AI 快照格式的 AI 快照
- CSS 选择器元素屏幕截图（`--element`）
- 完整的浏览器 PDF 导出

元素屏幕截图也会拒绝 `--full-page`；该路由会返回 `fullPage is
not supported for element screenshots`。

如果看到 `Playwright is not available in this gateway build`，则打包的
Gateway 网关缺少核心浏览器运行时依赖项。请重新安装或更新
OpenClaw，然后重启 Gateway 网关。对于 Docker，还需按下方所示安装 Chromium
浏览器二进制文件。

#### Docker Playwright 安装

如果 Gateway 网关在 Docker 中运行，请避免使用 `npx playwright`（npm 覆盖冲突）。
对于自定义镜像，请将 Chromium 预装到镜像中：

```bash
OPENCLAW_INSTALL_BROWSER=1 ./scripts/docker/setup.sh
```

对于现有镜像，请改为通过内置 CLI 安装：

```bash
docker compose run --rm openclaw-cli \
  node /app/node_modules/playwright-core/cli.js install chromium
```

要持久化浏览器下载，请设置 `PLAYWRIGHT_BROWSERS_PATH`（例如
`/home/node/.cache/ms-playwright`），并确保通过
`OPENCLAW_HOME_VOLUME` 或绑定挂载持久化 `/home/node`。OpenClaw 会在 Linux 上自动检测持久化的
Chromium。请参阅 [Docker](/zh-CN/install/docker)。

## 工作原理（内部）

一个小型 loopback 控制服务器接收 HTTP 请求，并通过 CDP 连接到基于 Chromium 的浏览器。高级操作（点击/输入/快照/PDF）通过 CDP 之上的 Playwright 执行；缺少 Playwright 时，只能使用不依赖 Playwright 的操作。底层的本地/远程浏览器和配置文件可以自由切换，而智能体始终看到一个稳定的接口。

## CLI 快速参考

所有命令都接受 `--browser-profile <name>` 以指定特定配置文件，并接受 `--json` 以输出机器可读结果。

<AccordionGroup>

<Accordion title="基础：状态、标签页、打开/聚焦/关闭">

```bash
openclaw browser status
openclaw browser doctor
openclaw browser doctor --deep    # 添加实时快照探测
openclaw browser start
openclaw browser start --headless # 一次性启动本地托管无头浏览器
openclaw browser stop            # 还会清除仅附加/远程 CDP 上的模拟设置
openclaw browser reset-profile   # 将配置文件的浏览器数据移至废纸篓
openclaw browser tabs
openclaw browser tab             # 当前标签页的快捷命令
openclaw browser tab new
openclaw browser tab new --label research
openclaw browser tab label abcd1234 research
openclaw browser tab select 2
openclaw browser tab close 2
openclaw browser open https://example.com
openclaw browser focus abcd1234
openclaw browser close abcd1234
```

</Accordion>

<Accordion title="配置文件：列出、创建、删除">

```bash
openclaw browser profiles
openclaw browser create-profile --name research --color "#0066CC"
openclaw browser create-profile --name attach --driver existing-session --cdp-url http://127.0.0.1:9222
openclaw browser delete-profile --name research
```

</Accordion>

<Accordion title="检查：屏幕截图、快照、控制台、错误、请求">

```bash
openclaw browser screenshot
openclaw browser screenshot --full-page
openclaw browser screenshot --ref 12        # 或使用 --ref e12
openclaw browser screenshot --labels
openclaw browser snapshot
openclaw browser snapshot --format aria --limit 200
openclaw browser snapshot --interactive --compact --depth 6
openclaw browser snapshot --efficient
openclaw browser snapshot --labels
openclaw browser snapshot --urls
openclaw browser snapshot --selector "#main" --interactive
openclaw browser snapshot --frame "iframe#main" --interactive
openclaw browser snapshot --out snapshot.txt
openclaw browser console --level error
openclaw browser errors --clear
openclaw browser requests --filter api --clear
openclaw browser pdf
openclaw browser responsebody "**/api" --max-chars 5000
```

</Accordion>

<Accordion title="操作：导航、点击、输入、拖动、等待、求值">

```bash
openclaw browser navigate https://example.com
openclaw browser resize 1280 720
openclaw browser click 12 --double           # 角色引用也可使用 e12
openclaw browser click-coords 120 340        # 视口坐标
openclaw browser type 23 "hello" --submit
openclaw browser press Enter
openclaw browser hover 44
openclaw browser scrollintoview e12
openclaw browser drag 10 11
openclaw browser select 9 OptionA OptionB
openclaw browser download e12 report.pdf
openclaw browser waitfordownload report.pdf
openclaw browser upload /tmp/openclaw/uploads/file.pdf
openclaw browser upload /tmp/openclaw/uploads/file.pdf --ref e12
openclaw browser upload media://inbound/file.pdf
openclaw browser fill --fields '[{"ref":"1","type":"text","value":"Ada"}]'
openclaw browser dialog --accept
openclaw browser dialog --dismiss --dialog-id d1
openclaw browser wait --text "Done"
openclaw browser wait "#main" --url "**/dash" --load networkidle --fn "window.ready===true"
openclaw browser evaluate --fn '(el) => el.textContent' --ref 7
openclaw browser evaluate --fn 'const title = document.title; return title;'
openclaw browser evaluate --timeout-ms 30000 --fn 'async () => { await window.ready; return true; }'
openclaw browser highlight e12
openclaw browser trace start
openclaw browser trace stop
```

</Accordion>

<Accordion title="状态：Cookie、存储、离线、标头、地理位置、设备">

```bash
openclaw browser cookies
openclaw browser cookies set session abc123 --url "https://example.com"
openclaw browser cookies clear
openclaw browser storage local get
openclaw browser storage local set theme dark
openclaw browser storage session clear
openclaw browser set offline on
openclaw browser set headers --headers-json '{"X-Debug":"1"}'
openclaw browser set credentials user pass            # 使用 --clear 移除
openclaw browser set geo 37.7749 -122.4194 --origin "https://example.com"
openclaw browser set media dark
openclaw browser set timezone America/New_York
openclaw browser set locale en-US
openclaw browser set device "iPhone 14"
```

</Accordion>

</AccordionGroup>

注意：

- 面向智能体的 `browser` 工具提供 `action=download`（必需的 `ref` 和
  `path`）以及 `action=waitfordownload`（可选的 `path`）。两者都会返回已保存的
  下载 URL、建议的文件名和受保护的本地路径。托管的 Playwright 配置文件支持显式下载
  拦截；现有会话配置文件会返回不支持操作错误。
- 优先使用原子式选择器上传：在上传时传入触发器 `--ref`，这样 OpenClaw 就能在一个请求中完成预备和点击。如果有意稍后触发，仍支持仅含路径的 `upload`。使用 `--input-ref` 或 `--element` 可直接设置文件输入。`dialog` 是预备调用；应在触发对话框的点击/按键操作之前运行。如果某个操作打开模态框，操作响应会包含 `blockedByDialog` 和 `browserState.dialogs.pending`；传入该 `dialogId` 可直接响应。在 OpenClaw 之外处理的对话框会显示在 `browserState.dialogs.recent` 下。
- `click`/`type`/等操作需要来自 `snapshot` 的 `ref`（数字 `12`、角色引用 `e12` 或可操作的 ARIA 引用 `ax12`）。操作有意不支持 CSS 选择器。当可见视口位置是唯一可靠的目标时，请使用 `click-coords`。
- 下载和跟踪路径仅限于 OpenClaw 临时根目录：`/tmp/openclaw{,/downloads}`（回退：`${os.tmpdir()}/openclaw/...`）。
- `upload` 接受来自 OpenClaw 临时上传根目录和
  OpenClaw 管理的入站媒体中的文件。托管入站媒体可通过
  `media://inbound/<id>`、沙箱相对路径 `media/inbound/<id>`，或托管入站媒体
  目录内的已解析路径引用。嵌套媒体引用、
  路径遍历、符号链接、硬链接和任意本地路径仍会被拒绝。
- `upload` 也可以通过 `--input-ref` 或 `--element` 直接设置文件输入。

当 OpenClaw 能够确认替换后的标签页时，稳定的标签页 ID 和标签可在 Chromium 原始目标替换后继续保留，
例如同一 URL 存在唯一的旧/新标签页对，或提交表单后
一个旧标签页变为一个新标签页。存在歧义的
重复 URL 替换会获得新的句柄。原始目标 ID 仍然
不稳定；在脚本中优先使用来自 `tabs` 的 `suggestedTargetId`。

快照标志一览：

- `--format ai`（使用 Playwright 时的默认值）：包含数字引用（`aria-ref="<n>"`）的 AI 快照。
- `--format aria`：包含 `axN` 引用的无障碍树。当 Playwright 可用时，OpenClaw 会使用后端 DOM ID 将引用绑定到实时页面，使后续操作可以使用它们；否则，该输出仅供检查。
- `--efficient`（或 `--mode efficient`）：紧凑角色快照预设。设置 `browser.snapshotDefaults.mode: "efficient"` 可将其设为默认值（请参阅 [Gateway 配置](/zh-CN/gateway/configuration-reference#browser)）。
- `--interactive`、`--compact`、`--depth`、`--selector` 会强制生成包含 `ref=e12` 引用的角色快照。`--frame "<iframe>"` 将角色快照限定到一个 iframe。
- 使用 Playwright 时，`--labels` 会添加带叠加引用标签的屏幕截图
  （输出 `MEDIA:<path>`），以及一个 `annotations` 数组，其中包含每个引用的边界
  框。在 `screenshot` 上，Playwright 支持的标签可与 `--full-page`、
  `--ref` 和 `--element` 配合使用；在 `snapshot` 上，随附的屏幕截图仍然
  仅限视口。现有会话/chrome-mcp 配置文件会在
  页面屏幕截图上呈现叠加标签，但不会返回 `annotations`，也不会使用 Playwright
  整页/引用/元素投影辅助程序。如果没有 Playwright 或 chrome-mcp，
  则无法使用带标签的屏幕截图。
- `--urls` 会将发现的链接目标附加到 AI 快照中。

## 快照和引用

OpenClaw 支持两种“快照”样式：

- **AI 快照（数字引用）**：`openclaw browser snapshot`（默认值；`--format ai`）
  - 输出：包含数字引用的文本快照。
  - 操作：`openclaw browser click 12`、`openclaw browser type 23 "hello"`。
  - 在内部，该引用通过 Playwright 的 `aria-ref` 解析。

- **角色快照（类似 `e12` 的角色引用）**：`openclaw browser snapshot --interactive`（或 `--compact`、`--depth`、`--selector`、`--frame`）
  - 输出：包含 `[ref=e12]`（以及可选的 `[nth=1]`）的基于角色的列表/树。
  - 操作：`openclaw browser click e12`、`openclaw browser highlight e12`。
  - 在内部，该引用通过 `getByRole(...)`（重复项还会使用 `nth()`）解析。
  - 添加 `--labels` 可包含带叠加 `e12` 标签的屏幕截图。在
    Playwright 支持的配置文件上，这还会返回每个引用的边界框元数据
    （`annotations[]`）。
  - 当链接文本存在歧义且智能体需要明确的
    导航目标时，添加 `--urls`。

- **ARIA 快照（类似 `ax12` 的 ARIA 引用）**：`openclaw browser snapshot --format aria`
  - 输出：作为结构化节点的无障碍树。
  - 操作：当快照路径可通过 Playwright 和 Chrome 后端 DOM ID
    绑定引用时，`openclaw browser click ax12` 可用。
- 如果 Playwright 不可用，ARIA 快照仍可用于
  检查，但引用可能无法操作。需要操作引用时，请使用 `--format ai`
  或 `--interactive` 重新生成快照。
- 原始 CDP 回退路径的 Docker 验证：`pnpm test:docker:browser-cdp-snapshot`
  使用 CDP 启动 Chromium，运行 `browser doctor --deep`，并验证角色
  快照包含链接 URL、由光标提升为可点击项的元素以及 iframe 元数据。

引用行为：

- 引用在导航之间**不稳定**；如果操作失败，请重新运行 `snapshot` 并使用新的引用。
- 当能够确认替换后的标签页时，`/act` 会在操作触发替换后返回当前原始 `targetId`。
  后续命令应继续使用稳定的标签页 ID/标签。
- 如果角色快照是使用 `--frame` 获取的，则角色引用会限定到该 iframe，直至获取下一次角色快照。
- 未知或过期的 `axN` 引用会快速失败，而不会回退到
  Playwright 的 `aria-ref` 选择器。发生这种情况时，请在同一标签页上重新生成快照。

## 浏览器批处理 CLI

`openclaw browser batch` 在一次 `/act` 调用中运行一组嵌套的 `/act` 操作
（即通过智能体工具访问的同一 `kind="batch"` 运行时），因此 CLI
用户和脚本可以将 `wait`、`click`、`type` 和
`evaluate` 等操作组合成一个可重放的计划，无需为每个操作往返请求。`actions[]` 中的每个
条目都是一个 `BrowserActRequest`，即 `/act`
路由接受的封闭联合（`click`、`clickCoords`、`type`、`press`、`hover`、
`scrollIntoView`、`drag`、`select`、`fill`、`resize`、`wait`、`evaluate`、
`close`、`batch`），而不是任意 `openclaw browser` 子命令。`batch`
不支持 `profile="user"` 和其他现有会话（chrome-mcp）
配置文件；在这些配置文件上应逐个发送操作。

- CLI：使用 `openclaw browser batch --actions '<json>'`、`openclaw browser batch
--actions-file plan.json` 或 `openclaw browser batch --actions-file -` 可
  从 stdin 读取 JSON 数组。`--continue` 设置 `stopOnError=false`；
  默认行为是在首次出错时停止。`--target-id` 将整个批处理限定到
  一个标签页。
- 引用生命周期：引用来自批处理之前运行的 `snapshot`（快照不是
  嵌套操作）。更改页面状态的嵌套操作（例如触发导航的
  `click`，或修改 DOM 的 `evaluate`）可能会
  使批处理中后续操作使用的早期引用失效。应将更改状态的操作
  放在前面，或重新生成快照后拆分为后续批处理。导航和
  重新生成快照在批处理之外进行（`openclaw browser navigate` /
  `snapshot`），因为 `open`、`navigate` 和 `snapshot` 不是 `/act` 类型。
- 目标 ID 冲突：嵌套操作可以省略 `targetId`，或重复
  请求级 `targetId`；如果显式嵌套的 `targetId` 解析到
  不同的标签页，则会在任何操作运行前以 `ACT_TARGET_ID_MISMATCH`
  拒绝。批处理操作按设计共享请求的标签页。
- 错误摘要：响应为 `{ "results": [{ "ok": true }, { "ok": false,
"error": "<message>" }, ...] }`，按顺序为每个操作提供一个条目。当
  `stopOnError` 为默认值时，数组在首次失败处结束；使用
  `--continue` 时，数组会涵盖每个操作。任何失败条目都会使 CLI 以
  非零状态退出；传入 `--json` 可为脚本保留完整的有序响应。

## 增强的等待功能

除了时间/文本，还可以等待更多条件：

- 等待 URL（支持 Playwright glob）：
  - `openclaw browser wait --url "**/dash"`
- 等待加载状态：
  - `openclaw browser wait --load networkidle`
  - 托管的 `openclaw` 和原始/远程 CDP 配置文件支持此功能。使用 `existing-session` 驱动程序的配置文件（包括默认的 `user` 配置文件）会拒绝 `networkidle`；在这些配置文件上请使用 `--url`、`--text`、选择器或 `--fn` 等待。
- 等待 JS 谓词：
  - `openclaw browser wait --fn "window.ready===true"`
- 等待选择器变为可见：
  - `openclaw browser wait "#main"`

这些条件可以组合使用：

```bash
openclaw browser wait "#main" \
  --url "**/dash" \
  --load networkidle \
  --fn "window.ready===true" \
  --timeout-ms 15000
```

## 调试工作流

当操作失败时（例如“不可见”“严格模式冲突”“被覆盖”）：

1. `openclaw browser snapshot --interactive`
2. 使用 `click <ref>` / `type <ref>`（交互模式下优先使用角色引用）
3. 如果仍然失败：使用 `openclaw browser highlight <ref>` 查看 Playwright 的目标
4. 如果页面行为异常：
   - `openclaw browser errors --clear`
   - `openclaw browser requests --filter api --clear`
5. 如需深入调试，请记录跟踪：
   - `openclaw browser trace start`
   - 复现问题
   - `openclaw browser trace stop`（输出 `TRACE:<path>`）

## JSON 输出

`--json` 用于脚本和结构化工具。

示例：

```bash
openclaw browser --json status
openclaw browser --json snapshot --interactive
openclaw browser --json requests --filter api
openclaw browser --json cookies
```

JSON 中的角色快照包含 `refs`，以及一个小型 `stats` 块（行数/字符数/引用数/交互项数），以便工具判断有效负载的大小和密度。

## 状态和环境调节选项

这些选项适用于“让网站表现得像 X”之类的工作流：

- Cookie：`cookies`、`cookies set`、`cookies clear`
- 存储：`storage local|session get|set|clear`
- 离线：`set offline on|off`
- 请求头：`set headers --headers-json '{"X-Debug":"1"}'`（或位置参数形式 `set headers '{"X-Debug":"1"}'`）
- HTTP 基本身份验证：`set credentials user pass`（或 `--clear`）
- 地理位置：`set geo <lat> <lon> --origin "https://example.com"`（或 `--clear`）
- 媒体：`set media dark|light|no-preference|none`
- 时区/区域设置：`set timezone ...`、`set locale ...`
- 设备/视口：
  - `set device "iPhone 14"`（Playwright 设备预设）
  - `set viewport 1280 720`

## 安全和隐私

- openclaw 浏览器配置文件可能包含已登录的会话；请将其视为敏感信息。
- `browser act kind=evaluate` / `openclaw browser evaluate` 和 `wait --fn`
  会在页面上下文中执行任意 JavaScript。提示词注入可能会操控
  此行为。如果不需要此功能，请使用 `browser.evaluateEnabled=false` 将其禁用。
- `openclaw browser evaluate --fn` 接受函数源代码、表达式或
  语句体。语句体会封装为异步函数，因此请使用
  `return` 返回所需的值。当页面端函数所需时间可能超过默认求值超时时间时，请使用
  `--timeout-ms <ms>`。
- 有关登录和反机器人注意事项（X/Twitter 等），请参阅[浏览器登录 + X/Twitter 发帖](/zh-CN/tools/browser-login)。
- 确保 Gateway 网关/节点主机不对外公开（仅限环回地址或 tailnet）。
- 远程 CDP 端点权限强大；请通过隧道访问并加以保护。

严格模式示例（默认阻止访问私有/内部目标）：

```json5
{
  browser: {
    ssrfPolicy: {
      dangerouslyAllowPrivateNetwork: false,
      hostnameAllowlist: ["*.example.com", "example.com"],
      allowedHostnames: ["localhost"], // 可选的精确允许项
    },
  },
}
```

## 相关内容

- [浏览器](/zh-CN/tools/browser) - 概览、配置、配置文件、安全性
- [浏览器登录](/zh-CN/tools/browser-login) - 登录网站
- [浏览器 Linux 故障排查](/zh-CN/tools/browser-linux-troubleshooting)
- [浏览器 WSL2 故障排查](/zh-CN/tools/browser-wsl2-windows-remote-cdp-troubleshooting)
