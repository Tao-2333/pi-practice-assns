# 实践课 2 · 小组组织建仓与 Pi Agent 源码分析

## 任务一 · 小组组织仓库

本地仓库的 origin 指向 [hunnuOpenSourceSoftware2026Group001/pi-hunnu-2026](https://github.com/hunnuOpenSourceSoftware2026Group001/pi-hunnu-2026)。课程源仓库为 [hunnuOpenSourceSoftware2026/pi-hunnu-2026](https://github.com/hunnuOpenSourceSoftware2026/pi-hunnu-2026)。Fork 来源以组织仓库首页的 “forked from” 信息为准。

> **截图位置 ①｜组织仓库**  
> ![alt text](image.png)。

## 任务二 · 本地源码与版本

| 核对项 | 本地记录 |
|---|---|
| 源码目录 | `D:\桌面\doc\openSource\pi-hunnu-2026` |
| Clone 地址（Issue 中填写此 URL） | https://github.com/hunnuOpenSourceSoftware2026Group001/pi-hunnu-2026.git |
| 当前分支 | `main` |
| 完整提交编号 | `c3cc91632d0a1950fb122605b190233f2aa0006e` |
| 工作区状态 | 核对时 `git status --short` 显示 `?? report/`，表示报告目录尚未被 Git 跟踪。 |

```powershell
Get-Location
git remote -v
git branch --show-current
git rev-parse HEAD
git status --short
```

> **截图位置 ②｜S1 本地仓库信息**  
> ![alt text](image-1.png)。

> **截图位置 ③｜源码目录**  
> ![alt text](image-2.png)夹。

## 任务三 · 子模块功能与依赖关系

分析版本为 `c3cc91632`。根目录 `package.json:15` 依次构建下表的 11 个包，与题目一致。另有 `packages/evals/`，其 README 说明它用于编码助手的行为评测，包中没有 build 脚本，根构建也未调用它。

**本报告重点分析** `coding-agent`、`agent` 和 `ai`，用一次读取文件的请求追踪应用、工具执行与模型通信之间的分工。以下为源码与配置分析，产物按构建脚本推导，未运行完整构建。

### 模块功能与构建产物

表中路径相对于各自的 `packages/<子文件夹>/`。各行的共同依据为本包 README.md、package.json 的 scripts 和 dependencies，以及 tsconfig.build.json。内部包名按各自的 name 映射回目录名，例如 pi-agent-core 对应 agent。

**标准编译**表示执行 `tsc -p tsconfig.build.json`，把本包 src/ 编译到本包 dist/。继承的根配置 tsconfig.base.json 启用了类型声明和源映射，因此常见产物为 .js、.d.ts、.js.map 和 .d.ts.map。表中的“无”表示没有直接依赖仓库内的其他模块。

| 子文件夹 | 代码主要实现什么功能 | 本模块 build 执行什么、生成什么 | 直接依赖哪些内部模块 | 依据文件或代码位置 |
|---|---|---|---|---|
| `telemetry` | 提供追踪一次操作、跨进程传递追踪信息和记录指标的接口，同时提供内存记录器。 | 标准编译，生成追踪、指标及内存实现。 | 无 | src/index.ts、src/metrics.ts |
| `schema` | 用构造函数描述 JSON 数据的格式，再把规则编译为可重复使用的校验函数。 | 标准编译，生成规则构造器和校验器。 | 无 | src/index.ts（构造器与 compile） |
| `chord` | 用契约声明对象的状态和方法，使不同进程能访问对象、订阅变化，并支持插件热重载。 | 标准编译，生成对象运行库及 HTTP、worker 等子入口。 | `schema`、`telemetry` | src/index.ts、src/contracts/、src/store/store.ts |
| `tui` | 处理终端输入并绘制组件；比较前后画面，只更新发生变化的部分。 | 标准编译，生成终端界面和组件库。 | 无 | src/tui.ts、src/components/ |
| `codemode` | 在 QuickJS 沙箱中执行模型写的 JavaScript。脚本可循环调用宿主提供的工具，再返回汇总结果。 | 标准编译，生成沙箱执行及宿主通信代码。 | 无 | src/runtime/vm.ts、src/runtime/host.ts |
| `mcp` | 实现 MCP 客户端，通过标准输入输出或 HTTP 连接提供工具的服务，查询并调用工具。 | 标准编译，生成客户端和传输适配器。 | 无 | src/client.ts、src/index.ts |
| `ai` | 为不同供应商的模型提供统一调用方式，把消息和工具说明发给模型，再接收文本或工具调用请求。 | 运行 generate-models 生成模型目录和供应商数据，再由 build:offline 校验数据、编译并复制数据到 dist/providers/data/。 | `schema`、`telemetry` | catalog/generate.ts、src/api/、src/utils/validation.ts |
| `durable` | 把对话、模型轮次和工具调用进度保存到存储，使进程重启后能继续处理任务。 | 标准编译，生成会话、任务和文档运行库。 | `chord`、`ai`、`codemode`、`schema`、`telemetry` | src/index.ts、src/documents.ts、README.md 的 Persist and Resume |
| `env` | 为 durable 提供远程执行环境，可通过 SSH 连接远端守护进程，在远端运行工具。 | 标准编译，生成 JS 客户端；本命令未调用 Cargo 构建 daemon/ 中的 Rust 程序。 | `chord`、`durable` | src/connection.ts、src/remote-env.ts、daemon/ |
| `agent` | 维护对话状态；收到模型的工具请求后调用相应工具，把结果加入对话并继续请求模型。 | 标准编译，生成 Agent 类和对话循环库。 | `ai`、`schema` | src/agent.ts、src/agent-loop.ts（执行工具见第 835 行） |
| `coding-agent` | 把模型、对话循环、终端界面和读写文件等工具组装成可交互的 pi 编码助手，并管理会话。 | build:unbundled 先编译、设置 CLI 和 RPC 入口的执行权限并复制主题、图片、HTML 模板等资源；随后打包到 dist/bundle/，含 cli.js 和 rpc-entry.js。 | `agent`、`ai`、`codemode`、`mcp`、`tui` | src/core/sdk.ts:403、src/core/tools/read.ts、仓库根 scripts/build-coding-agent-bundle.mjs |

### 模块层级与依赖图

![alt text](module-dependencies.png)

箭头由使用方指向被依赖方。图中的 18 条边均来自各包 dependencies，未计入开发依赖和可选依赖。

以最长内部依赖路径为依据：没有内部依赖的包记作 L0，其余包的层级等于其直接依赖中的最高层级加一。由此得到 L0（telemetry、schema、tui、codemode、mcp），L1（chord、ai），L2（durable、agent），L3（env、coding-agent）。箭头允许跨层，例如 coding-agent 直接使用 L0 的 tui。

### 问题一 · 根目录与子模块的 build

npm run build 查找对应 package.json 的 scripts.build。根脚本用 cd 切换到各包目录，并用 && 串联该包的 npm run build；前一条成功才执行下一条。在子模块中执行时，npm 读取本包的脚本。例如在 packages/agent/ 中，npm 启动 tsc，随后由 tsc 读取 tsconfig.build.json，确定源码目录、输出目录及继承的编译选项。在 ai 和 coding-agent 中，脚本还显式安排了代码生成或打包等步骤。

### 问题二 · 构建顺序与依赖关系

本版本根脚本给出的顺序使每个被依赖包先于使用方构建。例如 agent 的 tsconfig.build.json 将 pi-ai 的类型入口指向 ../ai/dist/index.d.ts，因此全新检出时先构建 ai，才能为 agent 准备好声明文件。这个顺序由根脚本明确写出。相邻只说明执行时间相邻：telemetry 和 schema 排在前两位，两者都没有内部直接依赖。

### 问题三 · 沿一条依赖路径理解分工

**coding-agent → agent → ai**。假设用户要求“看看 hello.py”，这条路径可以从三个源码位置理解。

1. **coding-agent 提供工具并组装会话。** packages/coding-agent/src/core/sdk.ts 第 2 行导入 Agent，第 403 行创建实例。读文件的具体实现位于同包 src/core/tools/read.ts。它决定编码助手能对本地文件执行什么操作。
2. **agent 驱动对话和工具执行。** packages/agent/src/agent-loop.ts 使用 pi-ai 的消息等接口；当模型要求使用工具时，第 835 行附近调用 prepared.tool.execute。工具结果进入后续对话，模型据此生成解释。
3. **ai 处理模型通信。** packages/ai/src/api/ 包含不同供应商的请求与响应处理代码，将消息和工具说明交给模型，并把模型的响应整理成上层可以处理的形式。模型可以发出读取文件的请求，磁盘读取由第一步提供的工具完成。

这条路径只挑出两条直接依赖来说明分工。coding-agent 本身也直接声明了对 ai 的依赖，因此图中同时保留 coding-agent → ai。
