# ADR-0001：架构铁律的强制与测试机制

- **状态**：已接受（2026-10-20）
- **相关需求**：`SD-GUI-011`、`SD-GUI-013`（需求册 §5）
- **相关设计约束**：`SD-SYS-170`、`SD-SYS-171`、`SD-SYS-180`、`SD-SYS-181`、`SD-SYS-183`、`SD-NFR-095`、`SD-NFR-097`（设计册沿用编号）
- **相关设计**：`doc/design.MD` §2.1、§2.2、§4.1、§11.1
- **来源**：issue #5

## 背景

三条铁律必须在 CI 中可执行地强制：

1. `SiliconDraft.Core` 不得依赖任何 UI / CLI / 插件实现（`SD-SYS-181`）。
2. `SiliconDraft.Gui` 不得直连数据库（`SD-GUI-011`）。
3. GUI 与 CLI 必须共用同一个 `IHost` 构建过程（`SD-SYS-183`）。

单一手段无法覆盖全部失败模式：

- **IL 层**只能看到实际编译进程序集的引用。项目引用了 `Avalonia` 包但一行未用，`AssemblyRef` 里根本不存在。
- **元数据层**（csproj / `deps.json`）只能看到依赖声明，看不到代码里实际依赖了哪些类型。
- **静态分析看不到运行时动态加载**（反射、`AssemblyLoadContext`），而 `SD-SYS-170` / `SD-SYS-171` 要求的恰恰是"CLI 进程内没有加载 Avalonia"。

此外，xUnit v3 的进程模型是**每个测试工程一个进程、工程内所有测试同进程**，官方明确不提供按测试隔离进程。因此"在测试进程内调用 `Main` 再查 `AppDomain.GetAssemblies()`"会被同工程其他测试污染，且 `Environment.Exit` 会直接杀掉测试宿主。

## 决策

### 1. 三层组合断言

| 层 | 工具 | 覆盖 |
| --- | --- | --- |
| IL | `TngTech.ArchUnitNET 0.13.4` + `TngTech.ArchUnitNET.xUnitV3 0.13.4` | 类型依赖、方法体调用、程序集 `ReferencedAssemblyNames` |
| 元数据 | 普通 xUnit + `XDocument` / `JsonDocument` | csproj 直接引用；**Core** 的 `deps.json` 传递依赖图（含未使用的包引用）。**Gui 只检查直接引用，不对 `deps.json` 查 DuckDB**——经 Core 传递必然存在 |
| 进程 | 独立子进程 + 环境变量触发的已加载程序集 dump | 运行时动态加载；CLI 路径的进程洁净性 |

三层互补，缺一不可。不允许只写几条 ArchUnitNET Fluent 规则就认为铁律已被强制。

使用要点：

- `ArchLoader` 必须显式加载目标程序集；优先用 `LoadFilteredDirectory`（Cecil 能在该目录内解析依赖），不要依赖 `LoadAssembliesIncludingDependencies` 的文件模式。
- 程序集级引用读取 `Architecture.Assemblies → Assembly.ReferencedAssemblyNames`。只被引用、未显式加载的 `Assembly` 该属性为 `null`。
- 类型级规则用 `NotDependOnAnyTypesThat().ResideInNamespaceMatching(...)` / `ResideInAssemblyMatching(...)`；**不要**用 `NotDependOnAny(IObjectProvider)` 指向未加载的目标程序集——它会真空通过。
- `ResideInAssembly(string)` 匹配的是程序集 **FullName**，不是简单名；用 `System.Reflection.Assembly` 重载或 `ResideInAssemblyMatching`。
- `.Check()` 扩展来自 `TngTech.ArchUnitNET.xUnitV3`，只装核心包编译不过。

### 2. `Architecture.Tests` 工程

新增 `tests/SiliconDraft.Architecture.Tests/`，引用全部产品程序集（用于构建顺序与输出复制）；**IL 与元数据断言**用 `LoadFilteredDirectory` 通过 Cecil 读取，不把目标程序集反射加载进测试进程。运行时共用性断言（§3）按正常引用加载 `App`，会连带加载 `Gui` / Avalonia，这是该断言的目的。工程职责：

- IL 层与元数据层断言；
- `HostBootstrap` 的运行时共用性断言（见下）；
- CLI 子进程洁净性探针。

### 3. 单一组合根与 JIT 边界

- `SiliconDraft.App.Hosting.HostBootstrap` 是**唯一**引用 `HostApplicationBuilder` / `HostBuilder` 的类型；`Cli`、`Gui` 程序集中不得出现此类引用。
- 签名：`public static IHost Build(string[] args, bool includeGui)`。
- `ConfigureShared(...)` 与 GUI 专属注册（`ConfigureGuiServices`）分属独立类型 / 方法；`Program`、`CliEntry` 与共享构建方法**不得出现任何 Avalonia 类型 token**。
- 理由：.NET 的 JIT 在编译方法时会解析方法体内出现的类型引用，即使对应分支未执行。把 GUI 类型放进同一个方法体的 `if` 分支，CLI 路径仍会加载 Avalonia。
- `Cli` / `Gui` 不得引用 `IServiceCollection`：服务注册只发生在组合根。

运行时断言：分别以 `includeGui: false/true` 构建 `IHost`，断言 7 个门面服务（`IGenerationService` / `IQueryService` / `IExportService` / `ILibraryService` / `IStatsService` / `IWorkbookService` / `IRecentFilesService`）在两种模式下解析到**相同实现类型**；GUI 专属服务只在 `includeGui: true` 时存在。

### 4. CLI 进程洁净性探针

- 生产代码在 `Main` 第一行调用 `LoadedAssemblyDump.RegisterIfRequested()`；仅当环境变量 `SILICONDRAFT_DIAG_DUMP_ASSEMBLIES` 指向一个路径时才注册 `ProcessExit` 处理器，把 `AppDomain.CurrentDomain.GetAssemblies()` 的程序集名写入该文件。未设置时完全惰性、无副作用。
- 测试工程用 `Process` 启动 `dotnet <App 输出目录>/SiliconDraft.App.dll`，至少覆盖 `--help` 与一个真实子命令（具体命令由 issue #9 定）。
- 断言：退出码 0；dump 文件存在；**哨兵**（dump 中必须出现 `SiliconDraft.App`，防止钩子失效导致假通过）；不含 `Avalonia*` / `SkiaSharp*` / `HarfBuzzSharp*` / `SiliconDraft.Gui`。
- 测试侧显式移除 `DOTNET_STARTUP_HOOKS`、`CORECLR_PROFILER*` 等可能把 Avalonia 注入子进程的环境变量。
- `Architecture.Tests` 正常引用 `App`（运行时共用性断言需要 `HostBootstrap`）；子进程探针的路径由 MSBuild 生成的 `AssemblyMetadata` 指向 **App 自己的 bin 目录**，不使用测试输出中的副本。

### 5. 测试运行器与坏插件夹具

- 测试统一用 `xunit.v3 4.x` + **Microsoft.Testing.Platform**；`global.json` 中设置 `"test": { "runner": "Microsoft.Testing.Platform" }`，不引 `Microsoft.NET.Test.Sdk` / `xunit.runner.visualstudio`。
- `tests/fixtures/SiliconDraft.BadPlugin/` 是一个只用于构建的夹具工程：引用 `Core` 并实现一个假的 `IExportPlugin`，供 `Plugins.Tests` 断言 `SD-NFR-097`（引用 Core 的插件被拒绝加载）。

## 后果

- `SiliconDraft.App` 增加约 20 行仅在测试环境激活的诊断代码；这是为可观测的进程级铁律付出的最小代价。
- 架构测试需要额外构建一次 App 并启动子进程；相对整个测试套件可忽略。
- 新增铁律或依赖方向规则时，必须同时补 IL 与元数据两层断言（以及必要时的进程断言），不能只加一条 Fluent 规则。
- `Architecture.Tests` 与 `BadPlugin` 夹具已回写设计册 §2.1。该节是**终态**工程清单：测试工程从 5 个变为 6 个 + 1 个构建夹具；**切片**实际建 5 个测试工程（不含 `Benchmarks`），完整清单见 issue #5。

## 已否决的备选

- **只用 ArchUnitNET**：看不见未使用的包引用与运行时动态加载。
- **测试进程内调用 `Main` 再查 `AppDomain`**：xUnit v3 无按测试隔离进程，污染与 `Environment.Exit` 风险不可接受。
- **`DOTNET_STARTUP_HOOKS` 注入探针**：零产品代码，但需要额外构建 hook 程序集并管理路径；环境变量 dump 更简单且已实测。若将来要求产品代码零诊断接缝，可切换到该方案。
- **自研 Cecil 遍历器**：重复 ArchUnitNET 已提供的类型依赖图能力，维护成本更高。
