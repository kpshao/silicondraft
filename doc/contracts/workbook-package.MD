# SiliconDraft 对外契约 · 工作簿目录包与清单

> **契约性质**：工作簿会被用户**手工拷贝、跨机器分发、长期归档**。目录内文件布局与清单文件字段一旦变更，会直接影响用户既有数据，属**破坏性变更**。
> **本文档不是需求**：需求（工作簿必须自包含、可重定位、可跨平台打开、可重放）见 `doc/requirements-spec.MD` §4；表结构与 DDL 见 `doc/design.MD` §5 与附录 C。

## 1. 目录包结构


**SD-SYS-210**（MUST）：`<name>.sdworkbook/` 目录结构必须如下（★ = 必需，○ = 按需创建）：

```
<name>.sdworkbook/
├── workbook.json          ★ 清单
├── workbook.duckdb        ★ 数据（含 §4.2 全部 sd_* 表/视图/索引）
├── workbook.lock          ○ 运行时独占锁（关闭时删除；崩溃遗留由失效检测处理）
├── logs/                  ○ 日志（JSON Lines，轮转）
├── exports/               ○ 导出产物根（默认输出目录，见 §5.2）
├── staging/               ○ Parquet 暂存（运行时；入库成功后清空，中断时保留供 --resume）
├── reports/               ○ DQ 自检报告 HTML、图表 PNG/SVG
├── preview/               ○ 缩略图（Wafer Map 概览 / Bin 分布小图）
├── nodes/                 ○ 固化本次使用的节点包快照（保证可重放）
└── extra/                 ○ 用户自由放置（不参与任何校验，不进入 INV-5 检查）
```

**SD-SYS-211**（MUST）：`logs/` `exports/` `staging/` `reports/` `preview/` 的路径必须**可重定向**（配置项），默认指向工作簿内；**禁止**把 staging 放在系统临时目录（否则中断后无法 `--resume`）。

**SD-SYS-212**（MUST）：`nodes/` 中必须固化本次生成所用的**节点包快照**。理由：全局 `nodes/` 目录可能被升级或删除，而工作簿必须能独立重放。存储成本低（YAML 文本）。

**条目存在性（实现规则）**：上表 `○` 表示**按需创建**，不要求工作簿创建时即存在。`INV-5` 只禁止**未声明的**顶层条目，**不要求**已声明的条目全部存在；`workbook verify` 不得因缺少 `logs/`、`exports/` 等按需目录而报错。


### 节点包（nodepack）的交付形态

> 工作簿内 `nodes/` 固化的是本次生成所用的**节点包快照**，其原始交付形态规定如下（原 `SD-LIB-069`）。

| **SD-LIB-069** | MUST | 节点包（nodepack）的交付形态为：**一个目录 / 一个 zip**，含 `nodepack.json`（清单：`nodeId` / `version` / `parentNode` / `itemsFile` / `programsFile` / `binsFile`）+ YAML 文件。**不采用程序集包形态**（节点包是纯数据）。安装 = 解压到 `nodes/` 目录 |

---

## 2. 清单文件 `workbook.json` 规范


字段顺序固定（由 `System.Text.Json` 源生成器保证），**路径一律使用 `/`**（保证跨 RID 可拷贝）。

```json
{
  "schemaVersion": 1,
  "workbookId": "wb_01JQ7X3K9M2P4R6T8V0W1XYZAB",
  "name": "centerhigh_25lot_20261014",
  "state": "Ready",
  "isTemporary": false,
  "createdAt": "2026-10-14T11:30:12+08:00",
  "modifiedAt": "2026-10-14T12:05:31+08:00",
  "appVersion": "1.0.0",
  "generator": {
    "engineVersion": "1.0.0",
    "duckdbVersion": "1.5.5",
    "pluginApiVersion": "1.0"
  },
  "originPath": null,
  "derivedFrom": null,
  "preset": "P-CENTER-HIGH",
  "product": "SOC_X",
  "node": "28nm",
  "nodepackRef": "nodepack-28nm@1.0.0",
  "configHash": "sha256:9f2c…",
  "seedRoot": 20261014,
  "summary": {
    "lotCount": 25,
    "waferCount": 625,
    "dieCount": 625000,
    "overallYield": 0.9612,
    "bin1Count": 24250,
    "dbSizeBytes": 73400320,
    "totalSizeBytes": 91226112,
    "dqStatus": "PASS",
    "cachedAt": "2026-10-14T12:05:31+08:00"
  },
  "counts": {
    "testItemsCp": 40,
    "testItemsWat": 25,
    "watStructures": 60000,
    "exportJobs": 2,
    "exportFiles": 50
  },
  "defectsApplied": [
    { "defectId": "D-CENTER-HIGH", "shape": "circle",
      "geometry": { "radiusMm": 60 },
      "effects": [{ "quantity": "vt_n", "delta": 15 }] }
  ],
  "externalExports": [],
  "tags": [],
  "favorite": false,
  "notes": null,
  "checksums": {
    "workbook.duckdb": "sha256:1b7d…"
  }
}
```

| 字段组 | 说明 |
| --- | --- |
| 标识 | `schemaVersion` / `workbookId`（ULID，全局唯一）/ `name` / `state` / `isTemporary` |
| 时间与版本 | `createdAt` / `modifiedAt` / `appVersion` / `generator.*`（用于复现与兼容判定） |
| 归属 | `originPath`（临时→持久化 / 解包的来源） / `derivedFrom`（派生链） |
| 生成语义 | `preset` / `product` / `node` / `nodepackRef` / `configHash` / `seedRoot` |
| `summary` | **缓存**（可由 `workbook rebuild-manifest` 从库重建）；`cachedAt` 为缓存时间 |
| `counts` / `defectsApplied` | 便于**不打开库**即可读取（MRU 与 `workbook list` 的性能关键） |
| `externalExports` | 记录落在工作簿之外的导出产物，供 `workbook verify` 报告 |
| `checksums` | `workbook.duckdb` 的校验和；**仅在 `pack` / `save-as` / 显式 `workbook seal` 时写入**（日常生成不重算，避免每次关闭都付大文件哈希开销） |

**清单的读写兼容规则**：每次写入必须输出**完整字段形状**——未赋值的字段写 `null`、空集合写 `[]`，禁止省略字段；读取必须忽略未知字段，缺失字段取默认值。`schemaVersion` 高于当前实现支持的版本时必须拒绝打开（`ERR_WORKBOOK_INVALID`），不得按默认值猜测。这样旧版本读新清单不会因缺字段失败，新版本读旧清单也能取得完整默认值。

**摘要缓存的失效与刷新**：`modifiedAt > cachedAt` 或 `state ∈ {Dirty, Incomplete}` 时，**任何展示该工作簿摘要的地方**（起始页 MRU 条目、`workbook info`、`workbook list --dir` 的输出）**必须显示"统计待刷新"标记而非展示可能错误的数字**；`workbook verify` 必须校验 `summary.dieCount` 与库内 `COUNT(*)` 是否一致并对不一致给出 WARN；必须提供 `workbook rebuild-manifest <path>` 从库重建摘要。
**MRU 与 `workbook list` 的展示必须只依赖 `workbook.json` 这一处，且必须容忍其缺失或过期，不得因此报错**（读不到清单时只显示"名称 + 路径"，不显示统计）。

