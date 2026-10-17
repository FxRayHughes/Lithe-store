# 为语言插件接入构建、运行与测试

LSP 服务器负责语言分析，构建工具负责项目导入、编译、依赖解析与产物。
一个实用的语言插件应把这两条链路分别验证，再验证它们在保存、切换项目和停止时能配合工作。
先完成[LSP 插件生命周期教程](lsp-plugin-lifecycle.md)，再按本篇增加执行能力。
本篇的可操作示例以 macOS 声明式插件宿主为范围；Windows 仍有独立扩展实现，
不能凭 Core 校验通过推断已有相同的运行/测试宿主能力。

## 1. 先选择当前能发布的接入方式

本文于 2026-10-17 对照 Lithe 主仓库及 Lua 模板核对。API 3 的构建系统接口仍在开发；
下面区分已支持的声明与 Core 中尚未形成产品闭环的接口。发布前还要核实目标宿主版本。

| 需求 | 当前接入方式 | 能做什么、边界在哪里 |
| --- | --- | --- |
| 补全、跳转、诊断 | API 1/2 `languageServers` | 按 LSP 提供能力，不自动声明编译和测试 |
| 运行文件或固定项目命令 | API 1/2 `executionProviders[].run` | 命令与 argv 模板，工作目录为工作区根；不提供通用目标选择或构建模型 |
| 运行整个工程、文件所在范围或指定测试 | API 1/2 `executionProviders[].test` | 测试项目标记、文件发现和命令规划；不是完整语义测试发现与结构化结果协议 |
| 官方 Java 项目的运行/测试/调试 | 官方 Java 插件的 API 2 `hostIntegration: jdt` | 宿主已有 JDT 集成，不能复制为社区通用扩展点 |
| 通用 Build 工具窗口、目标与依赖模型、增量同步 | API 3 `buildSystems`，开发中 | Core 验证/规划接口预览，尚不能作为市场可安装的社区构建插件发布 |

当需求只是 `lua <当前文件>`、`go test ./...` 这类固定命令，可以使用现有执行提供者。
当需求需要 Maven/Gradle/Cargo 的模块、profile、target、classpath、依赖树或任务动态发现，
应使用成熟构建后端和完整模型；不要把一个 `run` 命令包装成已经完成工程集成。

## 2. 给 Lua 模板加上运行能力

在生命周期教程的最小清单中，替换下面三个顶层字段，其他字段保留：

```json
{
  "permissions": ["languageServer.launch", "process.run"],
  "activationEvents": ["onLanguage:lua", "onRun:lua"],
  "executionProviders": [
    {
      "languageID": "lua",
      "run": {
        "executable": { "command": "lua" },
        "arguments": ["${file}", "${userArguments}"]
      }
    }
  ]
}
```

这也是[Lua 模板](https://github.com/Lithe-IDEA/lithe-plugin-lua)实际采用的方式。
用户需要另行安装 Lua 解释器；随包提供的 LuaLS 不等于解释器。
`process.run` 声明运行权限，`onRun:lua` 声明按运行操作激活，二者都不能省略。
测试声明同理需要 `process.test` 和 `onTest:<languageID>`。

`executable` 只能选一个：裸 `command` 交给宿主工具路径发现，或使用宿主已经支持的
项目 `toolchain` ID，例如 Go 的 `project-go`。插件不能发明一个 toolchain ID 就指望宿主实现。

所有参数都是独立 argv 元素，不经过 shell：

| 模板 | 含义与范围 |
| --- | --- |
| `${file}` | 工作区相对路径；以 `-` 开头时宿主加 `./`，适用于运行、文件测试和用例测试 |
| `${fileDirectory}` | 文件所在目录，根目录为 `.`，其他为 `./…` |
| `${userArguments}` | 仅用于运行，必须独占一个元素，展开为用户参数数组 |
| `${testName}` | 指定测试用例名，去除首尾空白 |
| `${testName:regex}` | 将用例名的正则特殊字符转义，适合测试筛选参数 |

不要在数组里写 `"lua ${file} && echo done"`、额外 shell 引号或依赖变量替换；
那只是一个普通参数。路径中的空格也不需要作者再加引号。API 1/2 执行提供者没有可自行
设置的 `cwd`、`${projectDir}` 或通用构建设置字段，工作目录固定是工作区根。

## 3. 用 Go 说明三个测试范围

下面是一个 **Go 语言插件**的顶层字段片段。它来自现有 Go 提供者的命令结构；
需要与声明 `go` 语言及其服务器的清单合并，不能直接粘贴到 Lua 插件中使用。

```json
{
  "permissions": ["languageServer.launch", "process.run", "process.test"],
  "activationEvents": ["onLanguage:go", "onRun:go", "onTest:go"],
  "executionProviders": [
    {
      "languageID": "go",
      "run": {
        "executable": { "toolchain": "project-go" },
        "arguments": ["run", "${file}", "${userArguments}"]
      },
      "test": {
        "frameworkID": "go",
        "workspaceLabel": "All Go Tests",
        "projectMarkers": ["go.mod", "go.work"],
        "testFilePatterns": ["*_test.go"],
        "executable": { "toolchain": "project-go" },
        "workspaceArguments": ["test", "./..."],
        "fileArguments": ["test", "${fileDirectory}"],
        "testCaseArguments": [
          "test",
          { "value": "${fileDirectory}", "when": "fileSelected" },
          { "value": "./...", "when": "noFileSelected" },
          "-run",
          "^${testName:regex}$"
        ]
      }
    }
  ]
}
```

- 工作区有任一 `projectMarkers` 才被识别为测试项目。`testFilePatterns` 是忽略大小写、
  只支持 `*` 的文件名模式，不是正则或任意 glob。
- 工作区范围运行 `go test ./...`；“文件”范围运行文件所在的包，符合 Go 的测试单位，
  不承诺只执行该文件。
- 用例范围有选中文件时限定包，没有文件时使用 `./...`；引用 `${file}` 或
  `${fileDirectory}` 的用例参数必须用 `when: "fileSelected"` 保护。
- 声明式发现返回工作区和匹配文件，不会仅凭上述字段解析任意框架的测试函数、
  测试结果树或调试入口。要得到这些能力，需要对应后端及宿主消费支持。

这里的 `go run <文件>` 也只运行选中文件，不能替代需要多个源码文件的包运行。
需求超出模板能力时，应如实说明边界，或等待完整构建模型接入。

## 4. 将项目事实留给成熟后端

先检查你选用的语言服务器或项目后端是否已经支持工程导入、构建命令和运行目标。
例如 Java 的模块、classpath、测试/调试目标应使用相应 Java 后端给出的事实，
不要由插件另读构建文件再拼一套容易漂移的 classpath。其他语言也应复用其成熟工具链。

一个完整的工程工作流需要确认以下顺序：

```text
保存文档 → 等待后端接收并同步工程变更 → 构建目标 → 检查构建结果
        → 取得本次构建后的运行路径/产物 → 启动所选入口
```

构建失败时不能继续运行旧产物；项目切换后不能用旧模型启动新项目；取消后不能把迟到结果
覆盖到新一次操作。上述职责需要真实后端与宿主配合，声明一条命令并不会自动完成。

构建工具正常解析项目依赖时，可能需要访问网络并写用户的工具缓存；这与启动插件时临时下载
语言服务器不同。语言服务器和随包运行时应在打包阶段准备好。项目依赖、构建产物、
Gradle daemon 状态、Maven 仓库或 Cargo target 等应由对应工具放到其用户缓存或工作区，
不能写入插件包或应用安装目录。每个后台进程都要有明确的停止与清理策略。

## 5. API 3 构建系统接口：目前能研究什么

**当前状态：开发中的 Core 接口，不是可发布的完整宿主功能。**
截至本次核对，已提交基线支持 API 1/2；开发工作区新增 API 3 的清单验证与纯规划模块。
macOS/Windows 尚无通用 Build 工具窗口对它的完整消费链路，市场可安装版本选择也暂时跳过
API 3。因此不要把自己的正式插件升到 API 3，再让用户通过安装市场获得尚未实现的能力。

如果你参与 Core/宿主开发，可以阅读开发版本中的 `buildSystems` 契约及 Cargo、Go、
Gradle、Maven fixtures，并用下列问题检查设计。它们是接口评估项，不能替代产品验收：

| 模型部分 | 需要回答的问题 |
| --- | --- |
| 检测 | 如何发现根工程、嵌套模块、wrapper，如何排除生成目录？ |
| 工具与运行时 | wrapper/系统命令如何选择，所需 JDK/其他运行时如何发现和验证？ |
| 选项 | profile、target、模式、平台等如何进入明确的参数与环境，而不是拼接 shell？ |
| 工程模型 | 模块、目标、依赖和运行入口来自哪个成熟后端，如何解码与校验？ |
| 任务 | 支持哪些范围、是否需要用户参数、是否执行项目代码或访问网络？ |
| 同步 | 哪些配置变更触发重导入，如何取消旧请求并拒绝过期结果？ |
| 测试与诊断 | 发现、执行、结果映射由谁提供，路径如何对应用户源码？ |
| 生命周期 | 谁启动进程、监督超时、终止任务并清理状态，宿主是否已经消费这些计划？ |

Core 的计划接口本身不启动进程、不监视文件、不持久化选择，也不会显示界面。
即使 fixture 能生成正确命令，宿主仍须实现执行、权限/确认、取消、状态、结果和工具窗口。
不要把内部 dispatcher 命令写成社区插件可以调用的宿主脚本 API。

`languageServers[].hostIntegration` 的 JDT 集成与开发中的构建 `hostIntegration` 是不同接口，
都不是社区作者任意命名即可扩展的机制。现有 JDT 路径限官方插件；
不要复制官方 ID 或修改 `vendor` 来绕过限制。未来正式开放的能力以目标版本契约和宿主为准。

## 6. 验证完整工作流

打包与清单校验完成后，还应在目标平台、实际宿主中完成：

1. 分别准备单文件、多文件、嵌套工程和带空格路径的项目，明确插件支持的范围。
2. 保存修改后运行/测试，确认使用当前文件和目标；将源码改错，确认错误能被看到，
   完整构建集成应阻止继续启动旧产物。
3. 覆盖工作区、文件和指定用例范围，核对实际命令、工作目录、输出与退出码。
4. 缺少解释器、工具链版本不符、wrapper 无执行权限或依赖下载失败时，错误能定位到原因。
5. 取消耗时任务，随后再运行；切换项目与关闭工作区，确认旧任务、锁与后端进程没有残留。
   若宿主或工具尚不能做到，应报告限制，不能只在文档里宣称已经支持。
6. 禁用/卸载语言插件后检查该语言的运行、测试与 LSP 都处于正确状态。
   插件声明的语言由插件负责，不能假定缺失能力会回退到宿主内置实现。
7. 清理验收启动的应用和辅助进程，再按[LSP 生命周期检查表](lsp-plugin-lifecycle.md#7-把停止失败和卸载也当作功能验收)
   验证重启、升级和卸载。

命令规划测试证明参数正确，进程冒烟验证证明工具能够启动，真实项目验收才能证明用户工作流
成立。发布说明应明确已经验证的平台、宿主、语言服务器及构建工具版本。

相关入口：[生命周期教程](lsp-plugin-lifecycle.md) · [发布社区插件](publishing.md) ·
[版本追踪](versioning-and-updates.md) ·
[Lithe 插件契约](https://github.com/Lithe-IDEA/Lithe-IDEA/blob/preview/shared/contracts/plugins/v1/README.md)。
