---
read_when:
    - 你正在为插件添加设置向导
    - 你需要理解 `setup-entry.ts` 与 `index.ts` 之间的区别
    - 你正在定义插件配置 schema 或 package.json 中的 openclaw 元数据
sidebarTitle: Setup and config
summary: 设置向导、setup-entry.ts、配置架构和 package.json 元数据
title: 插件设置和配置
x-i18n:
    generated_at: "2026-07-26T06:53:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b07e3fa365939fa9c0885b31b7894f5e734313a7deef2297e316956063d97e45
    source_path: plugins/sdk-setup.md
    workflow: 16
---

插件打包（`package.json` 元数据）、清单（`openclaw.plugin.json`）、设置入口和配置架构的参考。

<Tip>
**想查看分步指南？** 操作指南结合具体场景介绍了打包：[渠道插件](/plugins/sdk-channel-plugins#step-1-package-and-manifest)和[提供商插件](/zh-CN/plugins/sdk-provider-plugins#step-1-package-and-manifest)。
</Tip>

## 软件包元数据

你的 `package.json` 需要一个 `openclaw` 字段，用于告知插件系统你的插件提供了什么：

<Tabs>
  <Tab title="渠道插件">
    ```json
    {
      "name": "@myorg/openclaw-my-channel",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "setupEntry": "./setup-entry.ts",
        "channel": {
          "id": "my-channel",
          "label": "My Channel",
          "blurb": "Short description of the channel."
        }
      }
    }
    ```
  </Tab>
  <Tab title="提供商插件 / ClawHub 基线">
    ```json openclaw-clawhub-package.json
    {
      "name": "@myorg/openclaw-my-plugin",
      "version": "1.0.0",
      "type": "module",
      "dependencies": {
        "typebox": "1.1.39"
      },
      "peerDependencies": {
        "openclaw": ">=2026.3.24-beta.2"
      },
      "openclaw": {
        "extensions": ["./index.ts"],
        "compat": {
          "pluginApi": ">=2026.3.24-beta.2",
          "minGatewayVersion": "2026.3.24-beta.2"
        },
        "build": {
          "openclawVersion": "2026.3.24-beta.2",
          "pluginSdkVersion": "2026.3.24-beta.2"
        }
      }
    }
    ```
  </Tab>
</Tabs>

<Note>
在 ClawHub 上对外发布需要 `compat` 和 `build`。规范发布代码片段位于 `docs/snippets/plugin-publish/`。
</Note>

### `openclaw` 字段

<ParamField path="extensions" type="string[]">
  入口点文件（相对于软件包根目录）。适用于工作区和 Git 检出开发的有效源码入口。
</ParamField>
<ParamField path="runtimeExtensions" type="string[]">
  `extensions` 的已构建 JavaScript 对应文件；OpenClaw 加载已安装的 npm 软件包时优先使用。有关源码/构建产物的解析顺序，请参阅 [SDK 入口点](/zh-CN/plugins/sdk-entrypoints)。
</ParamField>
<ParamField path="setupEntry" type="string">
  仅用于设置的轻量入口（可选）。
</ParamField>
<ParamField path="runtimeSetupEntry" type="string">
  `setupEntry` 的已构建 JavaScript 对应文件。还必须设置 `setupEntry`。
</ParamField>
<ParamField path="plugin" type="object">
  `{ id, label }` 后备插件标识，用于插件没有可据以派生 ID 或标签的渠道/提供商元数据时。
</ParamField>
<ParamField path="channel" type="object">
  用于设置、选择器、快速开始和状态界面的渠道目录元数据。
</ParamField>
<ParamField path="install" type="object">
  安装提示：`npmSpec`、`localPath`、`defaultChoice`、`minHostVersion`、`expectedIntegrity`、`allowInvalidConfigRecovery`、`requiredPlatformPackages`。
</ParamField>
<ParamField path="startup" type="object">
  启动行为标志。
</ParamField>
<ParamField path="compat" type="object">
  此插件支持的 `pluginApi` 版本范围。对外发布到 ClawHub 时必填。
</ParamField>

<Note>
提供商 ID（`providers: string[]`）是清单元数据，而不是软件包元数据。请在 `openclaw.plugin.json` 中声明，而不是在这里声明——请参阅[插件清单](/zh-CN/plugins/manifest)。
</Note>

### `openclaw.channel`

`openclaw.channel` 是一种低成本的软件包元数据，用于在运行时加载前支持渠道发现和设置界面。

### 渠道自有设置字段

渠道插件应使用 `defineChannelSetupContract(...)` 在运行时代码中定义一次设置字段，并在 `openclaw.channel.setup.fields` 下发布匹配的可序列化投影。运行时定义会推断插件本地输入类型，解析引导式和非交互式值，并将渠道专用键排除在核心类型之外。软件包元数据使 `openclaw channels add <channel-id> --help` 和 `openclaw channels add --channel <channel-id> --help` 无需加载插件即可仅发现所选渠道的选项。

```ts
import { defineChannelSetupContract } from "openclaw/plugin-sdk/channel-setup";

export const setupContract = defineChannelSetupContract({
  fields: {
    endpoint: {
      kind: "string",
      cli: { flags: "--endpoint <url>", description: "服务端点" },
    },
    transport: {
      kind: "choice",
      choices: ["native", "container"],
      cli: { flags: "--transport <kind>", description: "传输所有者" },
    },
  },
  adapter: {
    applyAccountConfig: ({ cfg, input }) => ({
      ...cfg,
      channels: { ...cfg.channels, example: input },
    }),
  },
});
```

```json
{
  "openclaw": {
    "channel": {
      "id": "example",
      "setup": {
        "fields": [
          {
            "key": "endpoint",
            "kind": "string",
            "cli": { "flags": "--endpoint <url>", "description": "服务端点" }
          },
          {
            "key": "transport",
            "kind": "choice",
            "choices": ["native", "container"],
            "cli": { "flags": "--transport <kind>", "description": "传输所有者" }
          }
        ]
      }
    }
  }
}
```

支持的字段种类包括 `string`、`boolean`、`integer`、`string-list` 和 `choice`。凭据请使用 `sensitive: true`。每个字段键必须等于其长 CLI 标志的驼峰式属性名，包括任何否定形式，例如 `--api-token` 对应 `apiToken`。当同时需要肯定形式和 `--no-*` 形式时，布尔字段可以添加 `cli.negatedFlags`。`channel`、`account` 和账户显示 `name` 仍属于共享控制封装。

已发布的 `setup`/`ChannelSetupInput` 适配器仍可供现有外部插件使用。新插件应公开 `setupContract`；两者同时存在时，OpenClaw 始终优先使用它。

| 字段                                   | 类型       | 含义                                                                          |
| -------------------------------------- | ---------- | ----------------------------------------------------------------------------- |
| `id`                    | `string`   | 规范渠道 ID。                                                         |
| `label`                    | `string`   | 主要渠道标签。                                                        |
| `selectionLabel`                    | `string`   | 需要与 `label` 不同时使用的选择器/设置标签。               |
| `detailLabel`                    | `string`   | 用于内容更丰富的渠道目录和状态界面的次要详细信息标签。                |
| `docsPath`                    | `string`   | 用于设置和选择链接的文档路径。                                        |
| `docsLabel`                    | `string`   | 需要与渠道 ID 不同时用于文档链接的覆盖标签。                          |
| `blurb`                    | `string`   | 简短的新手引导/目录描述。                                              |
| `order`                    | `number`   | 渠道目录中的排序顺序。                                                 |
| `aliases`                    | `string[]` | 用于选择渠道的额外查找别名。                                           |
| `preferOver`                    | `string[]` | 此渠道应优先于的较低优先级插件/渠道 ID。                               |
| `systemImage`                    | `string`   | 渠道 UI 目录的可选图标/系统图像名称。                                  |
| `selectionDocsPrefix`                    | `string`   | 选择界面中文档链接前的前缀文本。                                       |
| `selectionDocsOmitLabel`                    | `boolean`  | 在选择文案中直接显示文档路径，而不是带标签的文档链接。                 |
| `selectionExtras`                    | `string[]` | 附加到选择文案中的额外短字符串。                                       |
| `markdownCapable`                    | `boolean`  | 将渠道标记为支持 Markdown，以便做出站格式决策。                         |
| `exposure`                    | `object`   | 针对设置、已配置列表和文档界面的渠道可见性控制。                       |
| `quickstartAllowFrom`                    | `boolean`  | 让此渠道加入标准快速开始 `allowFrom` 设置流程。                  |
| `forceAccountBinding`                    | `boolean`  | 即使仅存在一个账户，也要求显式绑定账户。                               |
| `preferSessionLookupForAnnounceTarget`                    | `boolean`  | 为此渠道解析公告目标时优先进行会话查找。                               |
| `setup`                    | `object`   | 用于延迟发现 CLI 选项的可序列化渠道自有设置字段。                      |

示例：

```json
{
  "openclaw": {
    "channel": {
      "id": "my-channel",
      "label": "My Channel",
      "selectionLabel": "My Channel (self-hosted)",
      "detailLabel": "My Channel Bot",
      "docsPath": "/channels/my-channel",
      "docsLabel": "my-channel",
      "blurb": "Webhook-based self-hosted chat integration.",
      "order": 80,
      "aliases": ["mc"],
      "preferOver": ["my-channel-legacy"],
      "selectionDocsPrefix": "Guide:",
      "selectionExtras": ["Markdown"],
      "markdownCapable": true,
      "exposure": {
        "configured": true,
        "setup": true,
        "docs": true
      },
      "quickstartAllowFrom": true
    }
  }
}
```

`exposure` 支持：

- `configured`：在已配置/状态类列表界面中包含该渠道
- `setup`：在交互式设置/配置选择器中包含该渠道
- `docs`：在文档/导航界面中将该渠道标记为面向公众

### `openclaw.install`

`openclaw.install` 是软件包元数据，而不是清单元数据。

| 字段                        | 类型                                | 含义                                                                     |
| ---------------------------- | ----------------------------------- | --------------------------------------------------------------------------------- |
| `clawhubSpec`                | `string`                            | 用于安装/更新和新手引导按需安装流程的规范 ClawHub 规约。 |
| `npmSpec`                    | `string`                            | 用于安装/更新回退流程的规范 npm 规约。                             |
| `localPath`                  | `string`                            | 本地开发或内置安装路径。                                        |
| `defaultChoice`              | `"clawhub"` \| `"npm"` \| `"local"` | 有多个来源可用时的首选安装来源。                     |
| `minHostVersion`             | `string`                            | 支持的最低 OpenClaw 版本，`>=x.y.z` 或 `>=x.y.z-prerelease`。            |
| `expectedIntegrity`          | `string`                            | 固定版本安装预期的 npm dist 完整性字符串，通常为 `sha512-...`。    |
| `allowInvalidConfigRecovery` | `boolean`                           | 允许内置插件重新安装流程从特定的陈旧配置故障中恢复。  |
| `requiredPlatformPackages`   | `string[]`                          | npm 安装期间验证的必需平台特定 npm 别名。               |

<AccordionGroup>
  <Accordion title="新手引导行为">
    交互式新手引导使用 `openclaw.install` 提供按需安装界面：如果插件在运行时加载前公开提供商身份验证选项或渠道设置/目录元数据，新手引导可以提示使用 ClawHub、npm 或本地安装，安装或启用插件，然后继续所选流程。ClawHub 选项使用 `clawhubSpec`，存在时优先使用；npm 选项需要具有注册表 `npmSpec` 的可信目录元数据（精确版本和 `expectedIntegrity` 是可选的固定值，设置后会在安装/更新时强制执行）。将“显示什么”保留在 `openclaw.plugin.json` 中，将“如何安装”保留在 `package.json` 中。
  </Accordion>
  <Accordion title="minHostVersion 强制执行">
    如果设置了 `minHostVersion`，安装和非内置清单注册表加载都会强制执行该值。较旧的主机会跳过外部插件；无效的版本字符串会被拒绝。假定内置源插件与主机检出版本一致。
  </Accordion>
  <Accordion title="固定版本的 npm 安装">
    对于固定版本的 npm 安装，请在 `npmSpec` 中保留精确版本，并添加预期的工件完整性：

    ```json
    {
      "openclaw": {
        "install": {
          "npmSpec": "@wecom/wecom-openclaw-plugin@1.2.3",
          "expectedIntegrity": "sha512-REPLACE_WITH_NPM_DIST_INTEGRITY",
          "defaultChoice": "npm"
        }
      }
    }
    ```

  </Accordion>
  <Accordion title="allowInvalidConfigRecovery 的作用域">
    `allowInvalidConfigRecovery` 并不是绕过损坏配置的通用机制。它仅用于范围有限的内置插件恢复，允许重新安装/设置修复已知的升级残留问题，例如缺少内置插件路径，或同一插件存在陈旧的 `channels.<id>` 条目。如果配置因无关原因损坏，安装仍会以关闭方式失败，并提示操作员运行 `openclaw doctor --fix`。
  </Accordion>
</AccordionGroup>

### 延迟完整加载

渠道插件可通过以下配置选择延迟加载：

```json
{
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "startup": {
      "deferConfiguredChannelFullLoadUntilAfterListen": true
    }
  }
}
```

启用后，即使对于已配置的渠道，OpenClaw 在开始监听前的启动阶段也只加载 `setupEntry`。完整入口会在 Gateway 网关开始监听后加载。

<Warning>
仅当 `setupEntry` 注册了 Gateway 网关在开始监听前所需的一切内容（渠道注册、HTTP 路由、Gateway 网关方法）时，才启用延迟加载。如果完整入口拥有必需的启动能力，请保留默认行为。
</Warning>

如果设置/完整入口注册了 Gateway RPC 方法，请将其保留在插件专用前缀下。保留的核心管理命名空间（`config.*`、`exec.approvals.*`、`wizard.*`、`update.*`）仍归核心所有，并始终规范化为 `operator.admin`。

## 插件清单

每个原生插件都必须在包根目录中附带一个 `openclaw.plugin.json`。OpenClaw 使用它在不执行插件代码的情况下验证配置。

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "description": "为 OpenClaw 添加 My Plugin 能力",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "webhookSecret": {
        "type": "string",
        "description": "Webhook 验证密钥"
      }
    }
  }
}
```

对于渠道插件，请添加 `channels`（提供商插件则添加 `providers`）：

```json
{
  "id": "my-channel",
  "channels": ["my-channel"],
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

即使插件没有配置，也必须附带架构。空架构是有效的：

```json
{
  "id": "my-plugin",
  "configSchema": {
    "type": "object",
    "additionalProperties": false
  }
}
```

完整架构参考请参阅[插件清单](/zh-CN/plugins/manifest)。

## ClawHub 发布

Skills 和插件包使用不同的 ClawHub 发布命令。对于插件包，请使用包专用命令：

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

<Note>
`clawhub skill publish <path>` 是用于发布技能文件夹的另一条命令，不用于发布插件包。请参阅[在 ClawHub 上发布](/zh-CN/clawhub/publishing)。
</Note>

## 设置入口

`setup-entry.ts` 是 `index.ts` 的轻量替代方案，OpenClaw 仅需要设置界面（新手引导、配置修复、已禁用渠道检查）时会加载它：

```typescript
// setup-entry.ts
import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
import { myChannelPlugin } from "./src/channel.js";

export default defineSetupPluginEntry(myChannelPlugin);
```

这样可避免在设置流程期间加载繁重的运行时代码（加密库、CLI 注册、后台服务）。

将设置安全导出保留在配套模块中的内置工作区渠道，可以使用 `openclaw/plugin-sdk/channel-entry-contract` 中的 `defineBundledChannelSetupEntry(...)`，而不是 `defineSetupPluginEntry(...)`。该内置契约还支持可选的 `runtime` 导出，使设置时的运行时接线保持轻量且明确。

<AccordionGroup>
  <Accordion title="OpenClaw 何时使用 setupEntry 而不是完整入口">
    - 渠道已禁用，但需要设置/新手引导界面。
    - 渠道已启用但尚未配置。
    - 已启用延迟加载（`deferConfiguredChannelFullLoadUntilAfterListen`）。

  </Accordion>
  <Accordion title="setupEntry 必须注册的内容">
    - 渠道插件对象（通过 `defineSetupPluginEntry`）。
    - Gateway 网关监听前所需的所有 HTTP 路由。
    - 启动期间所需的所有 Gateway 网关方法。

    这些启动 Gateway 网关方法仍应避开保留的核心管理命名空间，例如 `config.*` 或 `update.*`。

  </Accordion>
  <Accordion title="setupEntry 不应包含的内容">
    - CLI 注册。
    - 后台服务。
    - 繁重的运行时导入（加密库、SDK）。
    - 仅在启动后需要的 Gateway 网关方法。

  </Accordion>
</AccordionGroup>

### 精简的设置辅助工具导入

对于高频且仅用于设置的路径，如果只需要部分设置界面，请优先使用精简的设置辅助工具接口，而不是更宽泛的 `plugin-sdk/setup` 聚合接口：

| 导入路径                | 用途                                                                                | 主要导出                                                                                                                                                                                                                                                                                                           |
| -------------------------- | ----------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/setup-runtime` | 在 `setupEntry` / 延迟渠道启动期间仍可使用的设置时运行时辅助工具 | `createSetupTranslator`、`createPatchedAccountSetupAdapter`、`createEnvPatchedAccountSetupAdapter`、`createSetupInputPresenceValidator`、`noteChannelLookupFailure`、`noteChannelLookupSummary`、`promptResolvedAllowFrom`、`splitSetupEntries`、`createAllowlistSetupWizardProxy`、`createDelegatedSetupWizardProxy` |
| `plugin-sdk/setup-tools`   | 设置/安装 CLI/归档/文档辅助工具                                                    | `formatCliCommand`、`detectBinary`、`extractArchive`、`resolveBrewExecutable`、`formatDocsLink`、`CONFIG_DIR`                                                                                                                                                                                                         |

需要完整的共享设置工具箱（包括 `moveSingleAccountChannelSectionToDefaultAccount(...)` 等配置补丁辅助工具）时，请使用更宽泛的 `plugin-sdk/setup` 接口。

使用 `createSetupTranslator(...)` 获取固定的设置向导文案。它按顺序使用 `OPENCLAW_LOCALE`、`LC_ALL`、`LC_MESSAGES` 和 `LANG` 中第一个非空白值，然后回退到英语。设置 `OPENCLAW_LOCALE=en` 可显式覆盖英语文本。将插件专用设置文本保留在插件自有代码中，并且仅将共享目录键用于通用设置标签、状态文本和官方内置插件设置文案。

设置补丁适配器在导入时保持高频路径安全。其内置单账户提升契约界面查找采用惰性方式，因此导入 `plugin-sdk/setup-runtime` 不会在实际使用适配器之前提前加载内置契约界面发现功能。

### 渠道自有的设置输入字段

`ChannelSetupInput` 是设置调用方和渠道插件共享的通用封装。其永久类型化字段包括 `name`、`token`、`tokenFile`、`useEnv`、`allowFrom` 和 `defaultTo`。运行时输入对象仍可包含由插件拥有的其他键，但共享类型不声明索引签名。每个插件都必须声明并收窄自己的设置字段，或在适配器边界使用插件自有架构进行验证：

```typescript
import type { ChannelSetupAdapter, ChannelSetupInput } from "openclaw/plugin-sdk/channel-setup";

type AcmeSetupInput = ChannelSetupInput & {
  workspaceId?: string;
  webhookUrl?: string;
};

export const acmeSetupAdapter: ChannelSetupAdapter = {
  applyAccountConfig: ({ cfg, input }) => {
    const setupInput = input as AcmeSetupInput;
    return {
      ...cfg,
      channels: {
        ...cfg.channels,
        acme: {
          token: setupInput.token,
          workspaceId: setupInput.workspaceId,
          webhookUrl: setupInput.webhookUrl,
        },
      },
    };
  },
};
```

以前直接在渠道上声明的渠道专属字段
`ChannelSetupInput` 暂时仍保留类型定义，以兼容外部源代码。
这些字段已弃用。2026-07-22 对 426 个已发布的树外
渠道插件进行了注册表扫描，移除了 21 个没有读取方的字段，并保留了 22 个存在已知
读取方的字段。每个保留字段都会在没有任何已发布插件读取它后立即删除；
无需版本边界。新的和内置插件不得依赖此
层级；应在本地声明其拥有的字段。

### 渠道自有的单账户提升

当渠道从单账户顶层配置升级到 `channels.<id>.accounts.*` 时，默认共享行为会将提升的账户范围值移入 `accounts.default`。

每个渠道插件都可以通过其设置适配器扩展或缩小该提升范围：

- `singleAccountKeysToMove`：应移入提升后账户的额外顶层键
- `namedAccountPromotionKeys`：当命名账户已存在时，只有这些键会移入提升后的账户；共享策略/投递键保留在渠道根级别
- `resolveSingleAccountPromotionTarget(...)`：选择由哪个现有账户接收提升的值

`singleAccountKeysToMove` 的存在表示提升契约已完整声明。即使该字段为空数组，也要声明它，以选择不执行旧键提升。省略该字段的适配器会为已经发布的插件保留一个由读取方支持的预声明提升层级。2026-07-22 的注册表扫描移除了 23 个没有已发布依赖方的键，并保留了六个常用键以及仅用于设置的 `rooms` 键。每个保留键都会在其已发布读取方迁移到声明后立即删除；无需版本边界。

当 Doctor 必须从轻量级内置设置工件加载这些声明时，请在插件包清单中声明 `openclaw.setupFeatures.configPromotion: true`。仅设置插件表面与完整渠道插件必须公开相同的声明。

使用已解析的插件调用 `moveSingleAccountChannelSectionToDefaultAccount(...)` 时，将其设置适配器作为 `setupSurface` 传入。调用方提供的设置表面优先于已加载和内置的查找结果，从而使限定范围或仅设置插件保持独立，不依赖全局注册。

<Note>
Matrix 是当前的内置示例。如果恰好已有一个命名的 Matrix 账户，或者 `defaultAccount` 指向现有的非规范键（例如 `Ops`），提升会保留该账户，而不是创建新的 `accounts.default` 条目。
</Note>

## 配置架构

插件配置会依据清单中的 JSON Schema 进行验证。用户通过以下方式配置插件：

```json5
{
  plugins: {
    entries: {
      "my-plugin": {
        config: {
          webhookSecret: "abc123",
        },
      },
    },
  },
}
```

注册期间，插件会以 `api.pluginConfig` 接收此配置。

对于渠道专属配置，请改用渠道配置部分：

```json5
{
  channels: {
    "my-channel": {
      token: "bot-token",
      allowFrom: ["user1", "user2"],
    },
  },
}
```

### 构建渠道配置架构

使用 `buildChannelConfigSchema` 将 Zod 架构转换为插件自有配置工件所使用的 `ChannelConfigSchema` 包装器：

```typescript
import { z } from "zod";
import { buildChannelConfigSchema } from "openclaw/plugin-sdk/channel-config-schema";

const accountSchema = z.object({
  token: z.string().optional(),
  allowFrom: z.array(z.string()).optional(),
  accounts: z.object({}).catchall(z.any()).optional(),
  defaultAccount: z.string().optional(),
});

const configSchema = buildChannelConfigSchema(accountSchema);
```

如果已使用 JSON Schema 或 TypeBox 编写契约，请使用直接辅助函数，这样 OpenClaw 就能在元数据路径中跳过 Zod 到 JSON Schema 的转换：

```typescript
import { Type } from "typebox";
import { buildJsonChannelConfigSchema } from "openclaw/plugin-sdk/channel-config-schema";

const configSchema = buildJsonChannelConfigSchema(
  Type.Object({
    token: Type.Optional(Type.String()),
    allowFrom: Type.Optional(Type.Array(Type.String())),
  }),
);
```

对于第三方插件，冷路径契约仍是插件清单：将生成的 JSON Schema 镜像到 `openclaw.plugin.json#channelConfigs`，以便配置架构、设置和 UI 表面无需加载运行时代码即可检查 `channels.<id>`。

## 设置向导

渠道插件可以为 `openclaw onboard` 提供交互式设置向导。该向导是 `ChannelPlugin` 上的一个 `ChannelSetupWizard` 对象：

```typescript
import type { ChannelSetupWizard } from "openclaw/plugin-sdk/channel-setup";

const setupWizard: ChannelSetupWizard = {
  channel: "my-channel",
  status: {
    configuredLabel: "Connected",
    unconfiguredLabel: "Not configured",
    resolveConfigured: ({ cfg }) => Boolean((cfg.channels as any)?.["my-channel"]?.token),
  },
  credentials: [
    {
      inputKey: "token",
      providerHint: "my-channel",
      credentialLabel: "Bot token",
      preferredEnvVar: "MY_CHANNEL_BOT_TOKEN",
      envPrompt: "Use MY_CHANNEL_BOT_TOKEN from environment?",
      keepPrompt: "Keep current token?",
      inputPrompt: "Enter your bot token:",
      inspect: ({ cfg, accountId }) => {
        const token = (cfg.channels as any)?.["my-channel"]?.token;
        return {
          accountConfigured: Boolean(token),
          hasConfiguredValue: Boolean(token),
        };
      },
    },
  ],
};
```

`ChannelSetupWizard` 还支持 `textInputs`、`dmPolicy`、`allowFrom`、`groupAccess`、`prepare`、`finalize` 等。完整的内置示例请参阅 Discord 插件的 `src/setup-core.ts`。

<AccordionGroup>
  <Accordion title="共享 allowFrom 提示">
    对于只需要标准 `note -> prompt -> parse -> merge -> patch` 流程的私信允许列表提示，优先使用 `openclaw/plugin-sdk/setup` 中的共享设置辅助函数：`createPromptParsedAllowFromForAccount(...)` 和 `createTopLevelChannelParsedAllowFromPrompt(...)`。
  </Accordion>
  <Accordion title="标准渠道设置状态">
    对于仅标签、分数和可选附加行有所不同的渠道设置状态块，优先使用 `openclaw/plugin-sdk/setup` 中的 `createStandardChannelSetupStatus(...)`，而不是在每个插件中手工构建相同的 `status` 对象。
  </Accordion>
  <Accordion title="可选渠道设置表面">
    对于只应在特定上下文中显示的可选设置表面，请使用 `openclaw/plugin-sdk/channel-setup` 中的 `createOptionalChannelSetupSurface`：

    ```typescript
    import { createOptionalChannelSetupSurface } from "openclaw/plugin-sdk/channel-setup";

    const setupSurface = createOptionalChannelSetupSurface({
      channel: "my-channel",
      label: "My Channel",
      npmSpec: "@myorg/openclaw-my-channel",
      docsPath: "/channels/my-channel",
    });
    // Returns { setupAdapter, setupWizard }
    ```

    当只需要该可选安装表面的一半时，`plugin-sdk/channel-setup` 还会公开更底层的 `createOptionalChannelSetupAdapter(...)` 和 `createOptionalChannelSetupWizard(...)` 构建器。

    生成的可选适配器/向导会对实际配置写入采取失败关闭策略。它们会在 `validateInput`、`applyAccountConfig` 和 `finalize` 中复用同一条“需要安装”消息，并在设置了 `docsPath` 时附加文档链接。

  </Accordion>
  <Accordion title="二进制程序支持的设置辅助函数">
    对于由二进制程序支持的设置 UI，优先使用共享委托辅助函数，而不是将相同的二进制程序/状态粘合代码复制到每个渠道：

    - `createDetectedBinaryStatus(...)`：用于仅标签、提示、分数和二进制程序检测有所不同的状态块
    - `createCliPathTextInput(...)`：用于基于路径的文本输入
    - `createDelegatedSetupWizardProxy(...)`：当 `setupEntry` 需要将状态、准备或完成行为惰性转发给更重的完整向导时使用
    - `createDelegatedTextInputShouldPrompt(...)`：当 `setupEntry` 只需要委托一个 `textInputs[*].shouldPrompt` 决策时使用

  </Accordion>
</AccordionGroup>

## 发布和安装

**外部插件：**发布到 [ClawHub](/clawhub)，然后安装：

<Tabs>
  <Tab title="npm">
    ```bash
    openclaw plugins install @myorg/openclaw-my-plugin
    ```

    在启动切换期间，裸包说明会从 npm 安装；但如果名称与内置或官方插件 ID 匹配，OpenClaw 会改用相应的本地/官方副本。使用 `clawhub:`、`npm:`、`git:` 或 `npm-pack:` 可确定性选择来源——请参阅[管理插件](/zh-CN/plugins/manage-plugins)。

  </Tab>
  <Tab title="仅 ClawHub">
    ```bash
    openclaw plugins install clawhub:@myorg/openclaw-my-plugin
    ```
  </Tab>
  <Tab title="npm 包说明">
    当包尚未迁移到 ClawHub，或迁移期间需要
    直接的 npm 安装路径时，请使用 npm：

    ```bash
    openclaw plugins install npm:@myorg/openclaw-my-plugin
    ```

  </Tab>
</Tabs>

**仓库内插件：**放置在内置插件工作区树下；构建期间会自动发现这些插件。

<Info>
对于来源为 npm 的安装，`openclaw plugins install` 会将包安装到 `~/.openclaw/npm/projects` 下每个插件独立的项目中，并禁用生命周期脚本（`--ignore-scripts`）。请确保插件依赖树仅包含纯 JS/TS，并避免使用需要 `postinstall` 构建的包。
</Info>

<Note>
Gateway 网关启动时不会安装插件依赖项。npm/git/ClawHub 安装流程负责依赖收敛；本地插件必须已经安装其依赖项。
</Note>

内置包元数据是显式声明的，不会在 Gateway 网关启动时根据已构建的 JavaScript 推断。运行时依赖项应属于拥有它们的插件包；打包后的 OpenClaw 在启动时绝不会修复或镜像插件依赖项。

## 相关内容

- [Building Plugins](/zh-CN/plugins/building-plugins) — 分步入门指南
- [插件清单](/zh-CN/plugins/manifest) — 完整的清单架构参考
- [SDK 入口点](/zh-CN/plugins/sdk-entrypoints) — `definePluginEntry` 和 `defineChannelPluginEntry`
