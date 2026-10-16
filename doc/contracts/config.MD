# SiliconDraft 对外契约 · 配置

> **契约性质**：配置文件是用户与脚本向 SiliconDraft 传递意图的**唯一输入契约**。字段名、类型、取值域与默认值变更属**破坏性变更**。
> **完整示例**：见 `doc/design.MD` 附录 A。**Schema 是唯一权威**（用于 YAML 校验、生成配置模板、生成 GUI 配置表单）。
> **本文档不是需求**：需求见 `doc/requirements-spec.MD` §3.1；Schema 生成与校验的实现见 `doc/design.MD`。

## 1. 配置文件规范


| 编号 | 强度 | 需求 |
| --- | --- | --- |
| **SD-CLI-040** | MUST | 支持 YAML 与 JSON 两种配置格式（内容结构一致） |
| **SD-CLI-041** | MUST | 配置必须通过 **JSON Schema 强校验**，校验失败时输出精确的字段路径 + 期望类型 + 实际值 + 修复建议（见 §8.2.1 `SD-CLI-011`） |
| **SD-CLI-042** | MUST | 支持 `extends` 引入基础配置（单继承或多继承，后者需显式声明合并顺序） |
| **SD-CLI-043** | MUST | 支持 YAML Anchor / 别名，避免大配置重复 |
| **SD-CLI-044** | MUST | 支持配置中的**引用**：`test_items: [{ file: ./items/x.yaml }]`、`preset: P-CENTER-HIGH`、`node: 28nm`；引用缺失时给出明确错误；循环引用必须检测并报错 |
| **SD-CLI-045** | MUST | 配置文件必须支持注释（YAML 原生），并随 `init` 生成带完整中文注释的模板 |
| **SD-CLI-046** | MUST | 生成时把**解析后的完整配置**（含所有继承与默认值）冻结写入 `sd_config` 表，保证可复现 |
| **SD-CLI-047** | SHOULD | 提供 `silicondraft validate --explain` 输出配置的"解析后结果 + 生效来源（哪个文件 / 哪层覆盖）" |
| **SD-CLI-048** | MUST | 配置文件必须支持**版本标记** `config_version: 1`，Schema 升级时提供自动迁移或明确不兼容提示 |


---

## 2. 配置节总览

> 完整示例见 `doc/design.MD` 附录 A；下表给出各配置节的作用与归属。

| 配置节 | 作用 | 归属 |
| --- | --- | --- |
| `run` | 任务名、预设、种子、并发度 | 生成 |
| `production` | Lot / Wafer / 产品 / 程序清单 | 生成 |
| `process` | 工艺节点、产品、良率与缺陷密度参数 | 生成 |
| `wafer` | 晶圆规格与 Die 排布 | 生成 |
| `test_items` | 测试项定义文件引用 | 生成 |
| `wat` | WAT 布局模型、环数与结构类型、冗余结构 | 生成 |
| `linkage` | CP ↔ WAT 联动强度与映射模式 | 生成 |
| `physics` | 空间场成分开关与参数 | 生成 |
| `defects` | 缺陷注入清单与影响映射 | 生成 |
| `execution` | CP / WAT 执行模拟参数 | 生成 |
| `storage` | 存储后端、线程/内存、入库通道、归档 | 存储 |
| `home` / `workbook` | 应用数据目录、工作簿默认目录、最近打开、锁与临时保留策略 | 工作簿 |
| `batch` | 批量生成的默认拆分策略 | 批量 |
| `export` | 导出路径模板、冲突策略、重试、校验和、插件私有配置 | 导出 |
| `library` | 节点包、测试程序、测试项覆盖 | 测试项库 |
| `plugins` | 插件搜索路径、禁用与优先级、隔离 | 插件 |
| `quality` | DQ 门禁开关与阈值 | 质量 |
| `output` | 输出格式清单与图表开关 | 导出 |
| `ipc` | 本地 IPC 服务（可选） | 集成 |

## 3. 配置项详表

> 各字段的类型、默认值、取值范围与约束以 **JSON Schema** 为唯一权威来源；Schema 随 `silicondraft init --template` 生成，并驱动 GUI 的配置编辑表单。
> 字段级说明采用**由 Schema 自动生成的注释**，避免模板与代码漂移；完整字段清单见 `doc/design.MD` 附录 A 的带注释示例。
