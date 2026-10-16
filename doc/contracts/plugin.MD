# SiliconDraft 对外契约 · 插件

> **契约性质**：插件由**第三方**开发与分发。插件清单字段、目录形态、宿主 API 版本兼容规则属**契约**；破坏性变更必须升主版本并在诊断命令中明确报错。
> **本文档不是需求**：扩展性目标见 `doc/requirements-spec.MD` §6.1；插件机制实现见 `doc/design.MD` §4.3、§4.4、§6。
> **重要约束**：插件**只能引用 `SiliconDraft.PluginApi`**，不得引用 `SiliconDraft.Core`。

## 1. 插件目录形态与发现


**SD-PLG-040**（MUST）：宿主必须按以下**优先级顺序**发现插件（高优先级先注册；同名时高优先级**覆盖**低优先级并记录 WARN）：

| 优先级 | 来源 | 形式 | 加载方式 |
| --- | --- | --- | --- |
| 1 | **内置插件** | `SiliconDraft.Plugins.Stdf` / `.Ad5` / `.Csv` | 普通引用，DI 注册 |
| 2 | **项目本地插件** | `<project>/plugins/<name>/` | `AssemblyLoadContext` 隔离加载 |
| 3 | **用户级插件目录** | `%SILICONDRAFT_HOME%/plugins/<name>/` | `AssemblyLoadContext` 隔离加载 |
| 4 | **CLI/GUI 显式指定** | `--plugin-path <dir>` / 环境变量 `SILICONDRAFT_PLUGIN_PATH` | `AssemblyLoadContext` 隔离加载 |
| 5 | **启用 / 禁用与优先级覆盖** | `%SILICONDRAFT_HOME%/plugins.json` 中的 `disabled` / `priority` | 影响注册表装配 |

**SD-PLG-041**（MUST）：插件目录形态必须为：

```
plugins/
└── stdf_custom/                 # 目录名 = 插件名（约定）
    ├── plugin.json              # ★ 清单（必需）
    ├── StdfCustom.dll           # 程序集（必需；可多个）
    └── *.dll / *.json           # 依赖（可多个，含插件自带的 deps.json）
```

**SD-PLG-042**（MUST）：`plugin.json` 清单必须至少包含：`name`、`version`、`apiVersion`、`assembly`（相对路径）、`entryType`（实现 `IExportPlugin` 的完整类型名）、`displayName`、`testTypes`、`fileExtension`、`enabled`、`priority`、`author`、`description`、`dependencies`（可选）、`minHostVersion`（可选）。**完整规范见 §16.5。**

**SD-PLG-043**（MUST）：发现流程必须：

1. 扫描全部来源，构建"候选插件（清单路径 + 元信息）"列表
2. 按优先级与同名覆盖规则得到最终候选集合
3. 对每个候选：创建**独立可卸载的 `AssemblyLoadContext`** → 加载程序集（含依赖解析）→ 反射找到 `entryType` → 校验实现了 `IExportPlugin` 且**无参可构造** → 校验 `ApiVersion` 与宿主 semver 兼容 → 校验必需元信息完整 → 实例化
4. **任何一步失败**：只记录到 `sd_plugin` 表（`status=error`、`loadError`），**不得影响主程序启动**，并在 `plugins list` / `plugins doctor` 中显式呈现
5. 成功的插件注册到 `PluginRegistry`

**SD-PLG-044**（MUST）：必须使用 `AssemblyLoadContext`（可回收）实现插件隔离：

| 要求 | 说明 |
| --- | --- |
| 隔离加载 | 每个插件目录一个独立 ALC（或在检测到依赖冲突时按插件分组），避免插件间依赖版本互相污染 |
| **`PluginApi` 共享** | 插件对 `SiliconDraft.PluginApi` 的引用**必须由默认 ALC 提供**（通过 `Resolving` 事件重定向），保证 `IExportPlugin` 类型标识一致（否则无法转型） |
| 依赖解析 | 优先使用插件目录内的依赖；缺失时回退到宿主已加载的依赖（仅限 `PluginApi` 与 BCL） |
| 卸载 | 必须支持 `plugins reload` 卸载并重新加载指定插件（`AssemblyLoadContext.Unload()` + GC 确认），用于开发迭代 |
| 异常边界 | 插件抛出的异常必须在宿主侧捕获、包装为带插件名的错误、记录日志，并按"该文件失败 / 该任务失败"处理 |

**SD-PLG-045**（MUST）：必须提供以下诊断能力：

| 命令 | 要求 |
| --- | --- |
| `plugins list [--verbose]` | 输出 `name / displayName / version / apiVersion / testTypes / extension / source(builtin\|path) / location / enabled / status / loadError` |
| `plugins doctor` | 诊断：插件目录是否存在且可读、各插件清单是否合法、程序集是否可加载、`PluginApi` 版本是否兼容、依赖是否齐全、`OptionsSchema` 是否合法、`entryType` 是否可实例化 |
| `plugins test <name>` | 用内置构造数据驱动插件，校验：可生成文件、文件可被 `verify` 读取、行数与输入一致 |
| `plugins reload <name>` | 卸载并重新加载指定插件 |

**SD-PLG-046**（MUST）：**禁止**从在线源安装插件（离线与供应链安全考虑）；插件必须由用户手工放置到插件目录。是否支持在线安装列为 MAY（见 SD-Q-038）。


---

## 2. 插件清单文件 `plugin.json` 规范


```json
{
  "name": "stdf_custom",
  "version": "1.2.0",
  "apiVersion": "1.0",
  "displayName": "STDF 定制版（某 YMS 专用）",
  "assembly": "StdfCustom.dll",
  "entryType": "Acme.StdfCustom.StdfCustomPlugin",
  "testTypes": ["CP"],
  "fileExtension": "stdf",
  "enabled": true,
  "priority": 100,
  "author": "Acme Yield Team",
  "description": "针对某 YMS 的 STDF 输出扩展（含 XY 坐标自定义 GDR）",
  "dependencies": ["Acme.Yield.Common.dll"],
  "optionsSchemaFile": null,
  "minHostVersion": "1.0.0",
  "notes": "optionsSchemaFile 为 null 表示由 entryType 的 OptionsSchema 属性提供"
}
```

| 字段 | 必需 | 说明 |
| --- | --- | --- |
| `name` | ✔ | 唯一名；必须与目录名一致（约定） |
| `version` | ✔ | 插件 semver |
| `apiVersion` | ✔ | 依赖的 `PluginApi` 版本；宿主做 semver 兼容检查 |
| `assembly` | ✔ | 相对于插件目录的程序集路径 |
| `entryType` | ✔ | 实现 `IExportPlugin` 的完整类型名 |
| `testTypes` | ✔ | `CP` / `WAT` 的组合；宿主据此在导出向导中过滤可选插件 |
| `fileExtension` | ✔ | 扩展名（不含点） |
| `enabled` | ○ | 默认 `true`；`false` 表示不注册 |
| `priority` | ○ | 同名覆盖优先级，默认 `0`；> 内置插件优先级即可覆盖内置实现 |
| `minHostVersion` | ○ | 最低宿主版本；不满足则拒绝加载并给出明确提示 |
| `optionsSchemaFile` | ○ | 若指定，从该 JSON 文件读取 Schema（与 `OptionsSchema` 属性二选一，文件优先） |
| `dependencies` | ○ | 需要预加载的依赖 DLL 列表（按顺序加载，避免解析顺序问题） |

