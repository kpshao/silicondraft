# SiliconDraft 对外契约 · CLI

> **契约性质**：本文件定义外部系统（CI 流水线、调度平台、外部 YMS 测试框架）**直接依赖**的 CLI 接口。**变更属破坏性变更**：必须在下述位置同时公告，并保留至少一个大版本的兼容期。
> **公告位置**：`CHANGELOG`、`silicondraft --help`、本文件。
> **本文档不是需求**：需求的"系统必须提供什么能力"见 `doc/requirements-spec.MD` §7；实现方式见 `doc/design.MD` §1.4。

**目录**

1. 命令集
2. 参数规范
3. 退出码规范
4. 日志规范
5. 命令速查（速查表）
6. 环境变量表

---

## 1. 命令集


**SD-CLI-020**（MUST）：必须提供以下命令：

| 编号 | 命令 | 用途 |
| --- | --- | --- |
| **SD-CLI-020a** | `silicondraft init [--dir] [--template]` | 初始化项目目录、生成配置模板与测试项库模板 |
| **SD-CLI-020b** | `silicondraft validate -c <config>` | 校验配置文件（语法 + Schema + 语义 + 引用完整性），不产生数据 |
| **SD-CLI-020c** | `silicondraft generate --workbook <path> [-c <config>]` | 生成数据并写入该工作簿的数据库（**不导出文件**） |
| **SD-CLI-020d** | `silicondraft export --workbook <path> --format <plugin> [选择集]` | 从工作簿导出为指定格式文件 |
| **SD-CLI-020e** | `silicondraft query --workbook <path> [--sql \| --named <query>] [--output …]` | 查询（受控 SQL / 预定义查询） |
| **SD-CLI-020f** | `silicondraft stats --workbook <path> [--level lot\|wafer\|die\|item]` | 输出统计结果（良率 / Bin / 参数统计 / 相关性） |
| **SD-CLI-020g** | `silicondraft verify --file <path> [--format stdf\|ad5]` | 校验导出文件的格式正确性与可解析性 |
| **SD-CLI-020h** | `silicondraft preview -c <config> --out preview.png` | 仅生成 Die Map / WAT 布局预览图，不产生数据（配置试错） |
| **SD-CLI-020i** | `silicondraft version [--output json]` | 版本信息（核心引擎、Schema、插件 API 版本） |
| **SD-CLI-020j** | `silicondraft healthcheck [--output json]` | 健康检查，供 K8s liveness/readiness 使用 |
| **SD-CLI-020k** | `silicondraft diff --workbook-a <a> --workbook-b <b>` | ⚠️ **降级为"仅元信息差异"**：由于 D-12，**不做跨库统计对比**；该命令仅支持输出两个工作簿的**元信息并列展示**（名称、预置、配置哈希、节点包版本），便于用户判断是否值得手工对比。**不输出任何数据级对比结论** |
| **SD-CLI-020l** | `silicondraft presets list \| show <name>`；`silicondraft nodes list \| show <node>` | 预设与工艺节点查看 |
| **SD-CLI-020m** | `silicondraft items export\|import\|sync\|diff [--node <node>]` | 测试项库管理（§7.2） |
| **SD-CLI-020n** | `silicondraft db init \| migrate \| info \| compact \| vacuum \| backup \| restore` | 数据库运维（作用于**当前工作簿**） |

**SD-CLI-021**（MUST）：**工作簿命令域** `workbook`：

| 子命令 | 说明 | 关键参数 |
| --- | --- | --- |
| `workbook new` | 创建工作簿 | `--name` / `--path`（默认 `<workbook.default_dir>/<auto>`）/ `--preset` / `--temp` / `--config` / `--from <existing>` |
| `workbook generate` | 在现有工作簿内生成数据 | `-c` / 覆盖参数 / `--resume`；**若该工作簿已有完成的数据集，必须拒绝并要求新建工作簿**（除非 `--force-recreate`） |
| `workbook info <path>` | 输出元信息与摘要统计 | `--output json` |
| `workbook current` | 输出当前会话打开的工作簿 | `--output json` |
| `workbook list` | **枚举**指定范围内的工作簿 | `--dir <目录>`（必需）/ `--glob <模式>` / `--filter preset=…` / `--sort` / `--output json`；**一次枚举，结果不缓存** |
| `workbook save-as` | 另存为（临时 → 持久化） | `<new-path>` / `--move` / `--copy` |
| `workbook duplicate` | 复制为变体实验基线 | `<new-path>` |
| `workbook rename` | 重命名 | `<new-name>` |
| `workbook delete` | 删除工作簿 | `--to-trash`（默认）/ `--purge`；**必须二次确认**（输入名称） |
| `workbook verify` | 校验不变量（只读打开） | `--output json` |
| `workbook repair` | 修复 | `--what lock,manifest,state,staging` |
| `workbook unlock` | 清理残留锁 | `--force` / `--yes`；跨主机锁禁止自动清理 |
| `workbook pack` | 打包为单文件 | `-o <file>.sdwb` / `--exclude logs,staging` / `--out-dir`（批量） |
| `workbook unpack` | 解包 | `-o <dir>` / `--verify` |
| `workbook migrate` | Schema 迁移 | `--to <schemaVersion>` / `--dry-run`；迁移前自动备份 |
| `workbook temp list` | 列出临时工作簿 | `--output json` |
| `workbook temp clean` | 清理临时工作簿 | `--older-than 7d` / `--to-trash` / `--yes` |
| `workbook recover` | 崩溃恢复 | `--list` / `--all` / `<path>` |
| ~~`workbook compare`~~ | 🚫 **已撤销**（D-12 完全隔离） | — |
| ~~`workspace …`~~ | 🚫 **已撤销**：原 `workspace list/create/use/rename/remove/rescan/stats/paths` 整体删除 | — |

**SD-CLI-022**（MUST）：**批量命令域** `batch`，用于"对一组**显式指定**的工作簿逐个执行单文件操作"：

| 子命令 | 说明 |
| --- | --- |
| `batch generate` | 按 `--workbook-per lot\|lot_group\|all_in_one` 策略批量创建 + 生成；支持 `--out-dir`、`--parallel`、`--resume`、`--continue-on-error`。**每个产物是独立工作簿，彼此无关联** |
| `batch init` | 从 CSV/YAML 清单批量定义工作簿（每行的 Lot 范围、预设、缺陷配置、目标路径） |
| `batch export` | 对一批工作簿执行导出；支持选择集与并行 |
| `batch verify` | 对一批工作簿执行校验，输出失败清单 |
| `batch pack` | 批量打包（用于整批归档 / 交付） |
| `batch clean` | 批量清理（默认进回收站，需确认） |
| ~~`batch stats`~~ | 🚫 **已撤销** —— 跨工作簿汇总分析属于"跨工作簿能力"，与 D-12 冲突 |

> **⚠️ 边界说明（重要）**：`batch` **不属于"跨工作簿能力"**，因此不违反 D-12。理由是它**只在"逐个执行单文件操作"的意义上工作** —— 每个目标工作簿都是独立处理的，产物之间不产生任何数据关联、不写入任何清单或索引、不改变任何工作簿的内容。它的价值仅仅是"省去把同一条命令敲 N 遍"。相比之下 `batch stats`（跨库汇总**分析**）才是真正的跨工作簿能力，已删除。**若认为连批量也应当去掉**，则需同时放弃 §10.1 的"日生成 100 Lot"目标（`SD-SCL-021`/`SD-SCL-022`）—— 该取舍登记为 **SD-Q-060**。

**SD-CLI-023**（MUST）：`batch` 部分失败必须返回 **`ERR_PARTIAL (9)`**，并在结果中输出"成功清单 + 失败清单 + 各自失败原因 + 是否可重试"，且必须支持 `batch generate --resume` 只重试失败项（**依赖失败工作簿自身 `workbook.json.state` 的判定，不依赖任何外部索引**）。

**SD-CLI-024**（MUST）：批量命令必须支持"一条命令生成 N 个工作簿"：

```bash
# 单工作簿：一个任务含 25 个 Lot
silicondraft workbook new --name lot2026w42 --preset P-CENTER-HIGH -c cfg.yaml
silicondraft generate --workbook ~/Documents/SiliconDraft/lot2026w42.sdworkbook --resume
silicondraft export   --workbook ~/Documents/SiliconDraft/lot2026w42.sdworkbook --format stdf

# 批量：按策略拆成多个工作簿（100 个 Lot → 默认每 5 个 Lot 一个工作簿）
silicondraft batch generate --preset P-EDGE-HIGH --lots LOT_A01..LOT_A100 \
  --workbook-per lot_group --lot-group-size 5 \
  --out-dir ~/Documents/SiliconDraft --parallel 8 --continue-on-error

# 枚举某个目录下有哪些工作簿（一次性枚举，不建索引）
silicondraft workbook list --dir ~/Documents/SiliconDraft --filter preset=P-EDGE-HIGH

# 交付：打包发给同事（主要的跨机器交换方式）
silicondraft workbook pack ~/Documents/SiliconDraft/lot2026w42.sdworkbook -o lot2026w42.sdwb
```


---

## 2. 参数规范


**SD-CLI-030**（MUST）：CLI 必须遵循统一的参数约定。

**全局参数（所有子命令可用）**

| 参数 | 说明 | 默认 |
| --- | --- | --- |
| `--config, -c <path>` | 配置文件路径（YAML/JSON） | 自动查找 `./silicondraft.yaml`、`~/.silicondraft/config.yaml` |
| `--workbook <path>` | 工作簿路径（`.sdworkbook` 目录或 `.sdwb` 包）。**只接受路径**，不按名称解析 | `app-settings.json` 中的"当前工作簿"；无则报 `ERR_NO_WORKBOOK (16)` |
| `--set KEY=VALUE` | 覆盖任意配置项（点号路径 + 数组下标，可重复） | — |
| `--output, -o <mode>` | stdout 输出格式：`text` \| `json` \| `ndjson` \| `table` \| `csv` | `text` |
| `--log-level <lvl>` | `trace\|debug\|info\|warn\|error\|fatal` | `info` |
| `--log-file <path>` | 日志文件路径（开启后同时写文件） | 不写文件 |
| `--log-format <fmt>` | `human` \| `json`（JSON Lines） | `human`（TTY）/ `json`（非 TTY） |
| `--progress <mode>` | `auto` \| `bar` \| `ndjson` \| `none` | `auto`（TTY=bar，非 TTY=ndjson） |
| `--quiet, -q` | 仅输出错误 | — |
| `--verbose, -v` | 提升日志级别（可叠加） | — |
| `--non-interactive` | 禁止任何交互提示；需要确认时直接失败 | 非 TTY 时自动开启 |
| `--yes, -y` | 对所有确认提示回答"是" | — |
| `--parallel, -j <n>` | 并发度（**全局配额**：进程内分片并行度 × 跨工作簿并行度不得超过此值） | CPU 核数的一半，上限 16 |
| `--read-only` | 以只读方式打开工作簿 | — |
| `--on-lock <mode>` | `fail` \| `wait` | `fail` |
| `--task-id <id>` | 幂等键；重复运行相同 task-id 直接返回已有结果 | 由 `config_hash + scope` 派生 |
| `--force` | 忽略幂等，强制重新生成 / 覆盖 | — |
| `--dry-run` | 只做规划与校验，不写数据库、不写文件 | — |
| `--timeout <sec>` | 整体超时 | 0（不超时） |
| ~~`--db <path>`~~ | ❌ **作废**（全局单库概念取消）；保留为隐藏兼容别名并给出迁移提示 | — |

**生成命令专有参数**

| 参数 | 说明 |
| --- | --- |
| `--preset <name>` | 使用内置预设（与 `-c` 互斥） |
| `--lots <n>` / `--lot-ids A,B,C` | Lot 范围 |
| `--wafers <n>` / `--wafer-slots 1-25` | Wafer 范围 |
| `--seed <int>` | 覆盖随机种子 |
| `--test-type cp\|wat\|both` | 生成范围 |
| `--no-aggregate` | 跳过聚合统计（仅原始数据，加速） |
| `--shard-by lot\|wafer` | 分片粒度 |
| `--shard-index i` / `--shard-total n` | 跨机 / 跨进程分片（见 §4.4 `SD-STO-064`） |
| `--resume` | 从中断处续跑 |

**导出命令专有参数**

| 参数 | 说明 |
| --- | --- |
| `--format, -f <plugin>` | 导出插件名（`stdf` / `ad5` / `csv`） |
| `--select-lots` / `--select-wafers` / `--select-items` / `--select-bins` | 子集选择 |
| `--select-test-type cp\|wat\|both` | 测试类型 |
| `--select-die-state tested\|all` | Die 状态筛选 |
| `--out <dir>` | 输出根目录（默认 `{workbook_exports}`） |
| `--path-template <tpl>` | 路径命名模板（§5.2） |
| `--on-conflict <strategy>` | `overwrite` \| `skip` \| `rename` \| `fail` |
| `--retry <n>` / `--retry-delay <dur>` / `--retry-backoff <linear\|exponential>` | 失败重试策略 |
| `--resume <job-id>` | 断点续传指定导出任务 |
| `--split-per-wafer` / `--split-per-lot` | 每片 / 每 Lot 一个文件 |
| `--manifest <path>` | 导出清单输出路径 |
| `--checksum <algo>` | `none` / `sha256` / `md5`，默认 `sha256` |
| `--max-files <n>` | 限额保护，默认 10,000 |
| `--verify-after-export` | 导出后自动校验格式 |

**SD-CLI-031**（MUST）：参数优先级必须为 **CLI 参数 > 环境变量 > 配置文件 > 内置默认值**，且该规则必须写入 `--help` 文档。

**SD-CLI-032**（MUST）：环境变量统一使用 `SILICONDRAFT_` 前缀（完整清单见 §16.8）。

**SD-CLI-033**（MUST）：`silicondraft <cmd> --help` 必须提供完整帮助；`silicondraft --help` 列出全部命令与一句话说明。


> 配置文件的字段与格式规范见 `doc/contracts/config.MD`。

---

## 3. 退出码规范

> **稳定性承诺**：退出码语义在版本间保持稳定（新增编码不改变既有语义）。
> **唯一一次有意的语义调整**：把 `13` 收窄为"同进程内资源占用"并新增 `14`（工作簿跨进程占用）。该调整必须在 `CHANGELOG`、`--help` 与本文件中同时公告，并保留至少一个大版本的双判兼容期。
> **流水线迁移提示**：脚本若依赖"13 = 库被占用"的旧判定，必须改为判定 **13 或 14**（推荐只判 14）。

**SD-CLI-050**（MUST）：CLI 必须使用以下退出码。**语义在版本间保持稳定**（新增编码不改变既有语义）—— **唯一一次有意的语义调整**是把 `13` 收窄并新增 `14`（原因见需求册 §4.5 `SD-SYS-262`），该调整必须在 `CHANGELOG`、`--help` 与文档中同时公告，并保留至少一个大版本的双判兼容期。

| 码 | 常量 | 含义 | 流水线建议处理 |
| --- | --- | --- | --- |
| **0** | `OK` | 成功（含"无操作需要执行"的幂等命中） | 继续 |
| **1** | `ERR_GENERAL` | 未分类的通用错误 | 上报缺陷 |
| **2** | `ERR_USAGE` | 命令行用法错误 | 修脚本 |
| **3** | `ERR_CONFIG` | 配置错误 | 修配置 |
| **4** | `ERR_GENERATE` | 数据生成失败 | 上报缺陷 |
| **5** | `ERR_STORAGE` | 存储层错误 | 检查磁盘 / 权限 / 迁移 |
| **6** | `ERR_EXPORT` | 导出失败（全部失败） | 检查路径 / 权限 / 空间 |
| **7** | `ERR_PLUGIN` | 插件错误 | 检查插件版本 |
| **8** | `ERR_IO` | 文件系统 / 权限错误 | 检查路径 |
| **9** | `ERR_PARTIAL` | **部分成功** | 读 JSON 取失败清单后 `--resume` |
| **10** | `ERR_QUALITY` | 数据质量自检未通过 | 查看 DQ 报告；必要时 `--skip-quality-gate` |
| **11** | `ERR_TIMEOUT` | 超时终止 | 调大 `--timeout` 或分片 |
| **12** | `ERR_VERIFY` | 格式校验未通过 | 上报缺陷 |
| **13** | `ERR_LOCKED` | **仅表示同进程内的资源占用**（任务级互斥、文件句柄冲突） | 等待后重试 |
| **14** | `ERR_WORKBOOK_LOCKED` | 目标工作簿被其他进程以读写方式占用 | 等待 / 换工作簿 / 只读打开 |
| **15** | `ERR_WORKBOOK_NOT_FOUND` | 指定路径不存在，或需先解包 `.sdwb` | 检查路径 |
| **16** | `ERR_NO_WORKBOOK` | 未指定工作簿且无默认工作簿 | 加 `--workbook` 或 `workbook new` |
| **17** | `ERR_WORKBOOK_INVALID` | 工作簿不变量校验失败 / `schemaVersion` 不兼容 | 运行 `workbook verify` / `migrate` |
| **130** | `ERR_INTERRUPTED` | 用户中断（SIGINT） | 用 `--resume` 续跑 |
| **143** | `ERR_TERMINATED` | 被 SIGTERM 终止 | 用 `--resume` 续跑 |

**SD-CLI-051**（MUST）：退出码必须可通过 `--output json` 在结果对象中读取（`{"exit_code": N, "status": "…", "errors": […]}`），便于流水线无需解析 stderr。

**SD-CLI-052**（MUST）：`ERR_PARTIAL` 必须在结果中列出**失败清单**（哪些 Lot / Wafer / 文件失败、原因、是否可重试）以及**成功清单**。

**SD-CLI-053**（MUST）：`--dry-run` 时若校验通过必须返回 `0`。


---

## 4. 日志规范


**SD-CLI-060**（MUST）：CLI 必须严格区分两个输出流：

| 流 | 内容 | 要求 |
| --- | --- | --- |
| **stdout** | **仅业务结果**（查询结果、统计输出、文件清单、JSON 结果对象） | 必须可被管道直接消费；**不得输出任何日志、进度条、警告** |
| **stderr** | **日志、进度、警告、错误详情** | 人类可读；非 TTY 时自动切换为结构化格式 |

> 这是流水线可用的硬性前提：`silicondraft stats … --output csv > stats.csv` 必须得到纯净 CSV。

**SD-CLI-061**（MUST）：日志必须包含以下字段（human 模式为紧凑文本，json 模式为字段）：

| 字段 | 说明 |
| --- | --- |
| `ts` | ISO 8601 带时区，毫秒精度 |
| `level` | `TRACE/DEBUG/INFO/WARN/ERROR/FATAL` |
| `logger` | 模块名（如 `storage.duckdb`、`export.scheduler`、`plugin.stdf`） |
| `msg` | 消息文本 |
| `task_id` | 任务 ID（有则必填） |
| `workbook_id` / `dataset_id` | 工作簿 / 轮次 ID（有则必填） |
| `lot_id` / `wafer_id` | 当前处理对象（有则必填） |
| `plugin` | 插件名（导出相关日志必填） |
| `duration_ms` | 耗时（操作完成日志必填） |
| `error_code` | 错误码 / 异常类名（错误日志必填） |
| `detail` | 结构化附加信息（对象） |

**SD-CLI-062**（MUST）：日志级别语义必须明确：

| 级别 | 使用场景 |
| --- | --- |
| `TRACE` | 逐 Die / 逐记录的极细粒度追踪（默认关闭） |
| `DEBUG` | 分片边界、SQL 语句、插件调用时序 |
| `INFO` | 任务开始 / 结束、每 Lot 完成、导出文件写出 |
| `WARN` | 可恢复异常、降级行为（如跳过损坏记录）、DQ WARN |
| `ERROR` | 操作失败但进程可继续（单个 Wafer 失败） |
| `FATAL` | 进程必须终止的错误 |

**SD-CLI-063**（MUST）：`--log-file` 写出时必须启用**轮转**（按大小，默认 100 MB × 5 份），且文件内容为 JSON Lines（便于日志采集）。工作簿内的 `logs/` 同样适用。

**SD-CLI-064**（MUST）：日志中**必须脱敏** —— 不得输出完整数据库连接串中的凭据；导出路径按原样输出但需过滤 `..` 逃逸痕迹的警告。

**SD-CLI-065**（MUST）：必须支持 `--progress ndjson`，以 **NDJSON 事件流**输出到 stderr：

```json
{"event":"task_start","task_id":"...","total_units":25,"unit":"wafer","ts":"..."}
{"event":"unit_start","task_id":"...","unit_id":"LOT_A01/wafer_01","ts":"..."}
{"event":"progress","task_id":"...","done":12,"total":25,"percent":48.0,"rate_per_s":1.8,"eta_s":7.2}
{"event":"unit_done","task_id":"...","unit_id":"...","duration_ms":1180,"rows":1000}
{"event":"warning","code":"DQ_WARN","msg":"..."}
{"event":"task_done","task_id":"...","status":"ok","exit_code":0,"summary":{...}}
```

**SD-CLI-066**（MUST）：进度事件必须**节流**（默认 ≥ 500 ms 一条，可配置），避免日志洪泛。


---

## 5. 命令速查（速查表）


```bash
# ── 初始化与校验 ──
silicondraft init [--dir <d>] [--template]
silicondraft validate -c cfg.yaml [--explain]

# ── 工作簿生命周期（只按路径操作） ──
silicondraft workbook new --name <n> [--path <p>] [--preset P-...] [--temp] [--config cfg.yaml]
silicondraft workbook info <path> | current | verify <path> | repair <path>
silicondraft workbook unlock <path> --force --yes
silicondraft workbook rename <path> <new-name> | delete <path> [--to-trash]
silicondraft workbook save-as <path> <new-path> [--move|--copy]
silicondraft workbook duplicate <path> <new-path>
silicondraft workbook pack <path> -o <f>.sdwb [--exclude logs,staging]
silicondraft workbook unpack <f>.sdwb -o <dir> [--verify]
silicondraft workbook migrate <path> [--to <v>] [--dry-run]
silicondraft workbook temp list | temp clean [--older-than 7d] [--yes]
silicondraft workbook recover [--list|--all|<path>]
# ── 一次性枚举（不建索引，结果不缓存） ──
silicondraft workbook list --dir ~/Documents/SiliconDraft [--glob '*.sdworkbook'] [--filter preset=P-...]

# ── 数据操作（须指定工作簿路径） ──
silicondraft generate --workbook <path> [-c cfg.yaml] [--resume] [-j 8]
silicondraft query    --workbook <path> --named yield_by_wafer --output csv
silicondraft stats    --workbook <path> --level wafer --output json
silicondraft export   --workbook <path> --format stdf [--out <dir>]
silicondraft preview  -c cfg.yaml --out preview.png
silicondraft verify   --file <path> [--format stdf|ad5]

# ── 批量（对一组显式路径逐个执行单文件操作，不建立关联） ──
silicondraft batch generate --preset P-EDGE-HIGH --lots A01..A100 \
  --workbook-per lot_group --lot-group-size 5 \
  --out-dir ~/Documents/SiliconDraft --parallel 8 --continue-on-error
silicondraft batch init -f plan.csv
silicondraft batch export | verify | pack | clean [--select ...]

# ── 库与节点 ──
silicondraft items export | import | sync | diff | import-naming [--node 28nm]
silicondraft nodes list | show <node> | new <name> --based-on <parent> | diff
silicondraft presets list | show <name>

# ── 插件 ──
silicondraft plugins list [--verbose] | doctor | test <name> | reload <name> | new <name>

# ── 数据库运维（作用于当前工作簿） ──
silicondraft db init | migrate | info | compact | vacuum | backup | restore

# ── 元信息 ──
silicondraft version [--output json]
silicondraft healthcheck [--output json]

# ── GUI ──
silicondraft                              # 起始页
silicondraft gui --workbook <path>        # 直接打开
silicondraft gui --open-latest            # 打开最近一项
silicondraft gui --read-only              # 只读打开
silicondraft ipc serve                    # 本地 IPC 服务（可选）
```


---

## 6. 环境变量表

> 优先级：**CLI 参数 > 环境变量 > 配置文件 > 内置默认值**。

统一使用 `SILICONDRAFT_` 前缀。**优先级：CLI 参数 > 环境变量 > 配置文件 > 内置默认值**。

| 变量 | 等价参数 | 说明 |
| --- | --- | --- |
| `SILICONDRAFT_HOME` | — | 应用数据根目录，默认 `~/.silicondraft`（配置 / 插件 / 临时 / 恢复登记；**不含工作簿**） |
| `SILICONDRAFT_DOCUMENTS` | — | 新建工作簿的默认目录，默认 `~/Documents/SiliconDraft` |
| `SILICONDRAFT_WORKBOOK` | `--workbook` | 默认工作簿**路径** |
| `SILICONDRAFT_CONFIG` | `--config` | 默认配置路径 |
| `SILICONDRAFT_TEMP_DIR` | — | 临时工作簿目录覆盖（**必须在 HOME 下**） |
| `SILICONDRAFT_ALLOW_TEMP` | — | 是否允许创建临时工作簿（流水线可设为 `0`，强制持久化以避免产出无主数据） |
| `SILICONDRAFT_PLUGIN_PATH` | `--plugin-path` | 额外插件目录（`: `/`;` 分隔） |
| `SILICONDRAFT_LOG_LEVEL` | `--log-level` | 日志级别 |
| `SILICONDRAFT_LOG_FORMAT` | `--log-format` | `human` / `json` |
| `SILICONDRAFT_OUTPUT` | `--output` | 默认输出格式 |
| `SILICONDRAFT_PROGRESS` | `--progress` | 进度模式 |
| `SILICONDRAFT_PARALLEL` | `--parallel` | 默认并发度 |
| `SILICONDRAFT_CACHE_DIR` | — | 缓存目录 |
| `SILICONDRAFT_NO_COLOR` | — | 禁用彩色输出 |
| ~~`SILICONDRAFT_WORKSPACE`~~ | — | 🚫 **已撤销**（工作区概念不存在） |

