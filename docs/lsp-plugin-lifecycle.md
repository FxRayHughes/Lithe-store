# 从零构建一个完整生命周期的 LSP 插件

这篇教程面向想给 Lithe 增加语言支持的插件作者。我们用可运行的
[Lua Support 模板](https://github.com/Lithe-IDEA/lithe-plugin-lua)走完选型、清单、语言服务器打包、
安装与激活、日常运行、停止、升级和卸载；构建、运行与测试的接入另见
[构建工具接入指南](build-tool-integration.md)。发布步骤见[发布社区插件](publishing.md)。

本文的安装与验收路径以 **macOS 的社区插件市场（Lithe 0.6.1 起）**为基线。
API 2 示例需要所用宿主明确支持 API 2；不要只凭应用版本号推断支持情况。
索引能描述 Windows/Linux 的包，不代表对应宿主已经实现相同的插件安装与运行流程。
Lua 模板目前只提供 macOS arm64、x64 两个包。

## 1. 先确定插件与宿主各自负责什么

LSP（Language Server Protocol，语言服务器协议）让编辑器通过独立进程获得补全、诊断、
跳转等能力。Lithe 的声明式插件用 JSON 描述要启动的程序；宿主读取清单、发现运行时并管理
进程及 LSP 会话。你不需要编写宿主内的 Swift/JavaScript 插件入口。

```text
开发与构建：固定上游版本 → 下载并校验 → 组装各平台插件包 → 发布 zip → 索引收录
用户安装：下载 zip → 校验摘要与清单 → 确认程序及权限 → 安装并记录完整性摘要
正常运行：启用 → 打开对应语言的文件 → 启动服务器 → 初始化 LSP → 编辑与诊断
结束使用：禁用/关闭工作区 → 宿主停止会话和进程；更新/卸载 → 替换或移除包
```

| 对象 | 插件作者负责 | Lithe 负责 |
| --- | --- | --- |
| 语言识别 | 文件扩展名、完整文件名、项目标记 | 匹配文件并选择提供者 |
| 语言服务器 | 上游选型、许可、固定版本、启动入口、正确参数 | 校验声明、按需启动、LSP 通信与停止 |
| 运行时 | 声明依赖，或在允许的 API 中随包提供 | 按声明发现、检查并启动运行时 |
| 运行与测试 | 声明命令、参数和测试范围 | 生成计划、执行与展示结果 |
| 可写数据 | 指定上游使用的日志、索引和状态路径 | 为 API 2 状态模板提供平台可写目录 |
| 发布与维护 | 各平台产物、版本升级、兼容与回退验证 | 下载校验、安装审查、插件启停 |

先选成熟且许可允许分发的语言服务器。项目导入、依赖解析、符号和模块模型应尽量由该语言的
成熟后端提供。不要为了“接上 LSP”再复制一套工程模型。LSP 支持的具体功能仍取决于服务器
能力与宿主支持；安装服务器不会自动带来构建、测试、DAP 调试或自定义界面。

## 2. 跑通原版模板，再改成自己的插件

本地需要 Git、Node.js 22+、`tar`、`zip` 和 `unzip`。打包工具使用 Node.js，
不代表目标语言服务器需要 Node.js；LuaLS 是平台相关的原生程序。

```bash
git clone https://github.com/Lithe-IDEA/lithe-plugin-lua.git
cd lithe-plugin-lua
node scripts/package.mjs
unzip -l dist/lua-support-0.3.0-darwin-arm64.zip
shasum -a 256 dist/lua-support-0.3.0-darwin-arm64.zip
```

这里的 `0.3.0` 对应本教程使用的模板版本；上游更新后，产物名以模板清单和脚本输出为准。
脚本会重新生成 `dist/`，下载两个 macOS 架构的 LuaLS、核对固定 SHA-256，并输出每个 zip
的 SHA-256。Linux CI 可以组装 macOS 包，但这只证明打包成功，不能替代在目标 Mac 上运行。

模板源码与产物的关系：

```text
源码仓库
├── io.github.lithe-idea.lua-support/
│   ├── plugin.json
│   └── bin/lithe-lua-language-server    # 保留可执行权限的启动器
├── language-server.json                # 构建输入：版本、各平台 URL 与 SHA-256
├── scripts/package.mjs                 # 下载、校验、解包、组装 zip
└── .github/workflows/                  # PR 校验与标签发布

产物 lua-support-0.3.0-darwin-arm64.zip
└── io.github.lithe-idea.lua-support/
    ├── plugin.json
    ├── bin/lithe-lua-language-server
    └── language-server/                # 完整上游资源，不只是可执行文件
        ├── bin/lua-language-server
        └── …                          # 上游依赖、脚本和许可文件
```

派生自己的仓库后，同步修改以下位置：

1. 包目录名、`plugin.json` 的 `id`、显示名、版本与 `vendor`，使用你自己的稳定反向域名。
2. `scripts/package.mjs` 的 `PACKAGE_DIR`、`ASSET_PREFIX`，以及与新服务器对应的资源检查。
3. `language-server.json` 的版本、各平台 URL、摘要和许可信息。它是模板脚本自己的输入，
   不是宿主自动解释的统一下载 API；更换清单结构时也要修改打包脚本。
4. 启动器中的包内入口、参数和缓存命名空间。不能继续写入 Lua 模板的缓存目录。
5. CI 中读取清单的目录名、Release 说明、README、许可证与第三方声明。

不要只换包目录名：模板 CI 中也有原插件 ID，启动器中也有状态路径。

## 3. 写一个最小、完整的 LSP 清单

下面是 API 1 的纯 LSP 示例。它沿用模板的目录布局，先只接入 LuaLS；如要运行 Lua 程序，
再按[构建工具指南](build-tool-integration.md)添加运行提供者。示例中的作者和 ID 要换成自己的。

```json
{
  "schemaVersion": 2,
  "id": "io.github.alice.lua-support",
  "displayName": "Lua Support",
  "version": "0.1.0",
  "pluginAPIVersion": 1,
  "vendor": {
    "id": "io.github.alice",
    "displayName": "Alice",
    "signatureRequirement": "none"
  },
  "permissions": ["languageServer.launch"],
  "activationEvents": ["onLanguage:lua"],
  "languages": [
    {
      "id": "lua",
      "displayName": "Lua",
      "fileExtensions": ["lua"],
      "projectFileNames": [".luarc.json", ".luarc.jsonc"]
    }
  ],
  "languageServers": [
    {
      "languageID": "lua",
      "languageIdentifier": "lua",
      "launch": {
        "kind": "packaged",
        "path": "bin/lithe-lua-language-server"
      },
      "arguments": [],
      "validationArguments": []
    }
  ]
}
```

三个版本不要混淆：`schemaVersion: 2` 是清单格式，`pluginAPIVersion: 1` 是宿主需要支持的
插件 API 等级，`version: "0.1.0"` 是你的插件版本。上游 LuaLS 的版本另记在构建输入中。
只声明实际需要的最低 API；API 1 的服务器参数是字面量，不展开 API 2 模板。

`languages[].id`、`languageServers[].languageID` 和 `onLanguage:<id>` 必须对应。
声明服务器需要 `languageServer.launch` 权限及激活事件；每个语言最多一个服务器。
事件集合必须与实际提供者精确对应，不能随意增加未使用的事件。权限是能力声明，
不是限制外部程序磁盘或网络访问的操作系统沙箱。
启用只表示允许激活，打开项目中相应语言的文档才触发服务器启动。
文件后缀不等于 LSP 的文档标识，必要时单独设置 `languageIdentifier`。

### 选择启动方式

| 方式 | 适用场景 | 作者要做的事 |
| --- | --- | --- |
| API 1 `packaged` | 随包提供原生服务器或启动器 | 声明相对路径，打包全部资源并保留可执行权限 |
| API 1 `systemCommand` | 明确要求用户自行安装的服务器 | 声明裸命令候选和正确探测参数，记录用户安装要求 |
| API 2 `runtime` | 包内 JS/JAR 由 Node.js/Java 启动 | 声明包内入口及发现的运行时最低版本，或声明随包运行时 |

例如，下面是 `languageServers[]` 中一个服务器声明的替代方案，要求用户预先安装 `gopls`，
完整清单还必须声明 `go` 语言及相应权限、事件：

```json
{
  "languageID": "go",
  "launch": { "kind": "systemCommand", "commands": ["gopls"] },
  "arguments": ["serve"],
  "validationArguments": ["version"],
  "runtimeDependencies": [{ "tool": "go" }]
}
```

探测参数必须能自行以退出码 0 结束，不能用会持续运行的 LSP 启动参数。若上游没有合适探测
方式，可以像 Lua 模板一样用空数组，让 LSP 握手决定是否可用。通过探测仍不代表补全已经正常。

随包分发时优先使用固定、可复验的服务器。`systemCommand` 不会替用户安装工具，也不能作为
启动时下载服务器的变通方案。运行时依赖与主入口是两回事：例如随包服务器调用 Go 时，
`runtimeDependencies` 让宿主发现 Go 并传递环境；API 1 支持 `node`、`go`，API 2 增加 `java`。

## 4. 在构建阶段准备完整语言服务器

沿用模板的处理顺序：

1. 固定上游发行版本和每个平台/架构的 HTTPS 归档地址，核实许可证及再分发条件。
2. 将归档 SHA-256 写入版本控制。摘要应通过上游校验资料或经过审查的下载确认，
   构建时不能现下载、现算一个摘要就当作预期值。
3. 下载后先核对摘要，再解压到临时组装目录。缓存命中也必须重新校验；不匹配立即失败。
4. 复制服务器所需的全部文件：二进制、脚本、资源、动态库、许可证等；检查声明入口存在。
5. 检查归档没有符号链接、越界路径和开发机绝对路径，保留启动器与二进制的可执行权限。
6. 按平台生成 zip，再计算最终 zip 的 SHA-256，交给市场索引固定。

上游归档摘要与最终 zip 摘要用途不同：前者保证构建输入，后者保证用户拿到同一个发布包。
更换语言服务器时不能只替换下载地址，必须同步检查解压布局、入口、依赖、探测方式及许可。
模板中的归档处理适配 LuaLS 的 tar.gz；其他格式需要在自己的打包脚本中明确实现。

社区插件使用自己的 ID 和 `vendor.signatureRequirement: "none"`，通过第三方市场源的
摘要校验、用户确认和加载时内容复查安装。不要伪造 Lithe 发布者签名，也不要使用保留的
`dev.lithe.*` 命名空间。官方包的发布者签名是另一条信任路径，签名必须在完整组装后完成。

## 5. 将可写状态与安装文件分开

Lithe 会在加载时复查社区包内容。服务器如果将日志、索引、锁或生成文件写回插件目录，
下次加载就可能失败；写入应用安装包还会破坏应用签名与增量更新基线。

| 数据 | 放在哪里 | 需要注意的生命周期 |
| --- | --- | --- |
| 服务器、运行时、默认配置、许可证 | 只读插件包 | 随插件版本安装和替换 |
| 日志、可重建缓存 | 平台用户缓存目录 | 设置容量/轮转策略，记录清理方法 |
| 项目索引、锁、项目相关状态 | 按工作区隔离的可写目录 | 不同项目、工作树不能共用同一份可变索引 |
| 用户源码与构建产物 | 用户工作区或构建工具指定目录 | 卸载插件不应误删用户工程 |

API 1 的 Lua 模板通过启动器设置缓存目录，并使用 `exec` 将启动器替换为真实服务器。
不要把服务器放到后台、`nohup` 或自行创建常驻守护进程，否则宿主难以完整停止它。
标准输出只承载 LSP 的 JSON-RPC 帧，启动器提示与日志应写标准错误或日志文件。

模板中的自定义缓存路径属于启动器自己的约定，**不能假定宿主卸载时自动清理**。
需要说明残留内容及安全删除方法。若自己的服务器维护工作区索引，还要增加稳定的工作区隔离，
不要直接复用模板的单一缓存目录保存所有项目索引。

### API 2：让宿主提供状态目录

目标宿主支持 API 2 时，可以将上面的 Lua 示例升到 `pluginAPIVersion: 2`，
把 `languageServers` 替换为以下值，直接运行包内 LuaLS 并传入宿主状态目录：

```json
[
  {
    "languageID": "lua",
    "languageIdentifier": "lua",
    "launch": {
      "kind": "packaged",
      "path": "language-server/bin/lua-language-server"
    },
    "arguments": [
      "--logpath=${stateDir}",
      "--metapath=${workspaceStateDir}"
    ],
    "validationArguments": []
  }
]
```

`${stateDir}` 是宿主为该插件服务器管理的可写状态目录，`${workspaceStateDir}` 进一步
按工作区隔离。`${pluginDir}` 是只读包根路径；`${arch}` 的值是 `arm64` 或 `x86_64`，
与 Release 资产里的 `x64` 拼写不同。状态模板不能放进 `validationArguments`。
不要将这些模板写进 API 1 的参数，宿主会按原样传给进程。

如果服务器需要 Node.js/Java，API 2 的 `runtime` 可免去启动器。以下是服务器声明中的
`launch` 片段，入口要替换成自己的真实资源，服务器专用参数仍放在 `arguments`：

```json
{
  "kind": "runtime",
  "tool": "node",
  "minimumVersion": "18.17",
  "entry": "language-server/server.js"
}
```

宿主负责发现并检查运行时；不要再在 `runtimeDependencies` 中重复声明同一个启动工具。
`runtime` 不声明 `validationArguments`，运行时探测由宿主执行。若改为 `packagedRuntime`，
该字段与 `minimumVersion` 互斥，插件要随包提供运行时、依赖和许可，并分别验证目标架构。

API 2 也支持 `initializationOptions` 和声明式 `settings`。只传上游真实支持的选项；
`${setting:<id>}` 只能引用已声明设置，值必须符合类型和范围。设置是插件级、跨工作区共享，
修改后宿主重启相应服务器。详细格式见[正式契约](https://github.com/Lithe-IDEA/Lithe-IDEA/blob/preview/shared/contracts/plugins/v1/README.md#plugin-settings-api-2)。

## 6. 本地验证、安装与激活

先运行模板现有 CI 中的两层检查：`node scripts/package.mjs` 组装产物，随后用 Lithe Store
的 `readPackageManifests` 和 `checkManifest` 检查包路径、ID、版本和社区信任声明。
可参考模板的 [CI 工作流](https://github.com/Lithe-IDEA/lithe-plugin-lua/blob/main/.github/workflows/ci.yml)。
这些检查不会启动服务器，也不等同于完整的 Core 清单语义校验。
在插件仓库根目录可以按 CI 的方式本地检查产物；下面的相邻目录应使用一个尚未占用的名字：

```bash
git clone --depth 1 https://github.com/Lithe-IDEA/Lithe-store.git ../lithe-store-validation
node --input-type=module <<'JS'
import { readdirSync, readFileSync } from "node:fs";
import { readPackageManifests, checkManifest } from "../lithe-store-validation/scripts/validate-index.mjs";
const manifest = JSON.parse(readFileSync("io.github.lithe-idea.lua-support/plugin.json", "utf8"));
for (const name of readdirSync("dist")) {
  const { manifests, unsafe } = readPackageManifests("dist/" + name, manifest.id);
  const problems = [...unsafe, ...checkManifest(manifests, { id: manifest.id }, manifest)];
  if (problems.length) throw new Error(name + ": " + problems.join("; "));
  console.log("validated " + name);
}
JS
```

派生模板后，把这里读取 `plugin.json` 的目录名也换成自己的 ID。
如果错误来自跨引用、模板上下文或 API 等级，还需对照契约并通过目标宿主的安装校验。

要在正式版 Lithe 中验收社区包，使用[发布流程](publishing.md)在自己的公开 GitHub 仓库
发布 zip，再准备自己仓库根目录的 `lithe-plugin-index.json`。索引格式与 Store 相同，
下载 URL 指向该仓库的 Release 资产，摘要固定最终 zip；不必先申请进入公共 Store。
在 Lithe 的“管理插件仓库…”中加入你的仓库 URL，即可通过自己的源安装测试。
不要把正式版的“从磁盘安装”当作绕过社区来源与确认流程的开发入口；`none` 社区包走市场源。
开发宿主的本地包例外属于单独信任路径，不能替代社区安装验收。

安装前先禁用与你的插件声明同一语言的其他插件，避免提供者冲突。然后：

1. 在插件市场选择你的源和版本，确认显示的程序、权限、来源都符合预期。
   安装或更新后按界面提示**重启 Lithe**，让新版本进入本次启动的插件图。
2. 打开一个小项目中的 `.lua` 文件。应当出现补全、悬停、跳转和诊断；
   不要只检查进程“已运行”。
3. 保存修改，确认诊断更新；关闭并重开文件，确认没有重复服务器和重复诊断。
4. 从 Dock 启动 Lithe 再试一次，避免依赖终端启动时偶然继承的 PATH。
5. 若增加运行/测试，分别验证；LuaLS 已正常工作并不代表用户的 `lua` 已安装。

启动后的协议顺序也应能解释清楚：宿主发送 `initialize`，服务器返回能力后接收
`initialized`，随后按协商能力通过 `didOpen`、`didChange`、`didSave`、`didClose` 同步文档，
补全等请求和诊断沿同一会话传输。正常停止先 `shutdown`，再 `exit`；初始化未完成或关闭
超时时，宿主可以强停进程。当前 macOS 默认初始化/请求超时为 30 秒、关闭超时为 2 秒。
取消请求通过 `$/cancelRequest` 通知服务器，服务器仍可能完成计算，不能将取消当作进程已退出。

可用这个文件进行最小交互验收：

```lua
---@param name string
local function greet(name)
  return "Hello, " .. name
end

print(greet("Lithe"))
```

悬停查看 `greet` 的签名，跳到定义，再将调用参数改成数字，观察类型诊断。
具体提示随 LuaLS 版本和项目配置变化，验收时应记录所用版本和配置。

## 7. 把停止、失败和卸载也当作功能验收

声明式插件没有让作者注册任意 `activate()` / `deactivate()` 回调的机制。
宿主管理会话、进程和停用；作者负责让服务器保持前台运行、遵守 LSP 生命周期，
并把所有子进程和状态设计成可以停止、重启和隔离。当前 Core 的强停路径面向直接子进程，
不能假定会自动终止服务器任意派生的整棵进程树。下面是发布前需要执行的验收，
不是声明后即可免测的保证。

| 操作或故障 | 要检查的结果 |
| --- | --- |
| 仅安装/启用，不打开该语言 | 不因安装就提前启动语言服务器 |
| 编辑、保存、关闭/重开文件 | 文档同步和诊断正常，没有重复会话 |
| 快速切换两个项目/工作区 | 旧结果不串到新项目，索引与锁不互相占用 |
| 服务器异常退出或初始化失败 | 宿主能显示可定位的失败；恢复操作不会留下重复进程 |
| 缺少 Node.js/Go/Java 或版本不足 | 给出可操作的运行时错误，不在启动时自行下载 |
| 修改 API 2 设置 | 新会话使用新值，旧进程退出 |
| 禁用插件、关闭工作区、退出 Lithe | 相应会话与进程被停止，启动器/辅助子进程不残留 |
| 卸载与重新安装 | 卸载先停用，重启后删除包和宿主管理的插件状态；自定义缓存有明确说明 |
| 篡改安装包内一个文件后重新加载 | 内容复查拒绝加载，不能靠服务器自行修改清单“修复” |

已在本次启动中注册的声明式插件可即时启用/禁用；本次启动后新装的插件要等重启。
安装、更新、回滚及卸载的文件/版本转换按界面提示在下次启动完成，
卸载请求会先停用已注册插件。验收需要覆盖重启前和重启后的状态。

验收完成后关闭测试启动的应用、服务器和子进程。不要把关闭编辑器标签等同于卸载服务器，
也不要假定崩溃后一定会自动重试或所有平台都有相同恢复策略。发现生命周期问题时，
在插件仓库记录日志、宿主版本、包版本、架构和复现步骤；宿主进程管理问题再提交到 Lithe。

## 8. 发布、更新与回退演练

按[发布社区插件](publishing.md)递增插件版本、发布正式 Release 并首次提交收录。
模板的 Release 工作流会校验 `v<版本>` 与清单一致；原版示例是
`node scripts/package.mjs --expect-tag v0.3.0`，派生后使用自己的版本。

每次更新服务器或运行时，都要更新固定输入和许可，重建对应平台包，走完交互及生命周期验收。
发布后的 zip 不可替换，同一版本不能重用；修复要发布新版本。新增可运行程序会触发新的
安装确认，具体规则见[版本追踪与更新](versioning-and-updates.md)。

发布前至少演练一次“旧版 → 新版 → 恢复旧版逻辑”：

- 保留旧版本资产和索引记录，确保旧宿主仍有兼容版本可装。
- 新旧服务器使用不兼容索引格式时，隔离状态或提供安全重建；不能让旧版读损坏状态。
- macOS 的回滚功能使用安装记录中保留的上一版本，重新验证该包并在重启后切换，
  不提供任意历史版本选择，也不会恢复服务器索引或用户数据。应实际演练该路径。
- 当前版本有问题时，发布更高版本的修复包，必要时在新包里恢复上一个稳定服务器；
  同时申请撤回问题版本，不能只依赖用户机器恰好保留上一版本。
- 卸载重装也要验证。源码、用户构建产物与自定义缓存的处理必须说明，避免误删用户数据。

## 9. 按失败阶段排查

| 现象 | 优先检查 |
| --- | --- |
| 市场看不到或不可安装 | 源地址、当前平台资产、`pluginAPIVersion` 是否被宿主支持 |
| `plugin.archive.checksumMismatch` | 是否替换过 Release 资产；索引固定的是否为最终 zip 摘要 |
| `plugin.manifest.invalid` | 未知字段、schema/API 混淆、ID/引用、API 2 字段误放到 API 1 |
| `plugin.permission.undeclared` / `plugin.activation.invalid` | 提供者、权限与事件是否一一对应 |
| `plugin.launch.missing` | 相对路径、大小写、打包完整性与可执行权限 |
| `plugin.runtime.missing` | 运行时发现、最低版本、Dock 启动环境 |
| 有进程但无法补全 | stdio 是否被日志污染、LSP 握手、文档语言标识与上游配置 |
| 用过一次后拒绝加载 | 服务器是否向包内写了日志、索引、锁或生成文件 |
| 禁用后仍有进程 | 启动器是否使用 `exec`，是否后台化或产生未清理辅助进程 |
| 能补全但无法构建/测试 | 是否实际声明并验证了独立的运行/测试或构建集成 |

字段全集、API 等级和稳定错误码以 Lithe 的
[插件契约](https://github.com/Lithe-IDEA/Lithe-IDEA/blob/preview/shared/contracts/plugins/v1/README.md)及
[清单 Schema](https://github.com/Lithe-IDEA/Lithe-IDEA/blob/preview/shared/contracts/plugins/v1/plugin-manifest.schema.json)为准。
跟随 preview 阅读时要区分已发布能力与开发中的契约，发布前以目标宿主实际支持为准。
