# SiliconDraft 对外契约 · STDF 输出规格（CP）

> **契约性质**：STDF 是交付给 YMS / Yield Viewer 的**输出契约**。本文件定义记录集合、字段映射与结构不变式；**规格冻结前不得变更**。
> **规格来源**：**STDF V4**（Teradyne 规范，1985 年起；2010 年起由 SEMI 托管；现行版 V4-2007）。v1 写经典 V4（`FAR.STDF_VER=4`、`FAR.CPU_TYPE=2`）。
> **不是 SEMI E10**：SEMI E10 是设备 RAM（可靠性 / 可用性 / 可维护性）标准，与 STDF 无关；`E5` 是 SECS-II，`X8` / `X14` 无规范依据。
> **本文档不是需求**：需求（CP 导出必须符合 STDF V4、必须支持 wafer map 可重建）见 `doc/requirements-spec.MD` §6.6 / §6.7；写入算法实现见 `doc/design.MD`。


**SD-FMT-010**（MUST）：**记录集合**。v1 只能写下列记录；括号内为规范的 `(REC_TYP, REC_SUB)`：

| 记录 | 规范编号 | 必需性 | 说明 |
| --- | --- | --- | --- |
| `FAR` | (0,10) | **必需，唯一，首条** | `CPU_TYPE=2`（小端）、`STDF_VER=4` |
| `MIR` | (1,10) | **必需，唯一** | 紧随 `FAR`；承载 lot / product / tester / job 信息 |
| `WCR` | (2,30) | 默认开启（可选） | 每文件至多一条；wafer 几何与朝向；**仅 `per_wafer` 模式写** |
| `WIR` | (2,10) | **必需（CP）** | 每 wafer 一条，标记 wafer 数据块开始 |
| `PIR` | (5,10) | **必需** | **每 die 一条**（不是每 wafer 一条），标记 part 数据块开始 |
| `PTR` | (15,10) | **必需（核心）** | 每 die × 每测试项一条 |
| `PRR` | (5,20) | **必需** | 每 die 一条；承载 `HARD_BIN` / `SOFT_BIN` / `X_COORD` / `Y_COORD` |
| `WRR` | (2,20) | **必需（CP）** | 每 wafer 一条，标记 wafer 数据块结束并承载该片计数 |
| `TSR` | (10,30) | 默认开启（可选） | 每测试项一条执行 / 失败 / 告警计数 |
| `HBR` | (1,40) | 默认开启（可选） | 每 hard bin 一条计数 |
| `SBR` | (1,50) | 默认关闭（可选） | 每 soft bin 一条计数 |
| `PCR` | (1,30) | **必需（≥1 条）** | 文件级 part 统计 |
| `MRR` | (1,20) | **必需，唯一，末条** | 文件结束 |
| `RDR` | (1,70) | 默认关闭（可选） | 仅重测场景；若启用必须紧随 `MIR` |
| `SDR` | (1,80) | 默认关闭（可选） | 仅多 site 配置描述；v1 单 site 不写 |
| `GDR` | (50,1) | **v1 不写** | 仅自定义扩展用；strict 档不使用 |

> **不存在的记录（禁止出现在配置、代码与文档中）**：`HDR`、`SMR`、`DIE`、`TCR`、`HIL`、`CPS`。STDF V4 / V4-2007 中没有这些记录，也没有文件头计数（`D_CNT` / `H_CNT` / …）。
> **不存在的机制**：记录级压缩、压缩标志、"各数值字段前带 1 字节长度"的编码、`NEXT_REC` 链指针——STDF 均无。
> **v1 不写的记录**：`ATR`、`MPR`、`FTR`、`PMR`、`PGR`、`PLR`、`BPS`、`EPS`、`DTR`、`VUR`。

**SD-FMT-011**（MUST）：**PTR 字段映射**（字段顺序即写入顺序；类型由规范固定）：

| 字段 | 类型 | 来源 / 取值 | 备注 |
| --- | --- | --- | --- |
| `TEST_NUM` | `U4` | `TestItemDef.test_num` | 同一 program 多次导出必须稳定 |
| `HEAD_NUM` | `U1` | 固定 `1` | 单 head |
| `SITE_NUM` | `U1` | `DieRecord.site_id` | 与 `PIR` / `PRR` 一致 |
| `TEST_FLG` | `B1` | 见下方位义 | — |
| `PARM_FLG` | `B1` | 见下方位义 | — |
| `RESULT` | `R4` | `DieRecord.values[item]` | 存储值归一化到 `UNITS` 的基础单位 |
| `TEST_TXT` | `C*n` | `TestItemDef.item_id`（或 `naming_profile` 别名） | — |
| `ALARM_ID` | `C*n` | 无告警时为空串 | — |
| `OPT_FLAG` | `B1` | 全字段有效时 `0x02` | 见下方位义 |
| `RES_SCAL` | `I1` | `0` | v1 固定 0 |
| `LLM_SCAL` | `I1` | `0` | v1 固定 0 |
| `HLM_SCAL` | `I1` | `0` | v1 固定 0 |
| `LO_LIMIT` | `R4` | `lsl` | 测试下限 |
| `HI_LIMIT` | `R4` | `usl` | 测试上限 |
| `UNITS` | `C*n` | `TestItemDef.unit` 的基础单位 | **不得写前缀形式**（`AMPS`，不是 `uAMPS`） |
| `C_RESFMT` | `C*n` | 由 `precision` 生成（如 `%7.3f`） | 显示精度 |
| `C_LLMFMT` | `C*n` | 同上 | 显示精度 |
| `C_HLMFMT` | `C*n` | 同上 | 显示精度 |
| `LO_SPEC` | `R4` | `lsl_warn` | 预警下限；无限值时写 `0.0` 并置 `OPT_FLAG` bit2 |
| `HI_SPEC` | `R4` | `usl_warn` | 预警上限；无限值时写 `0.0` 并置 `OPT_FLAG` bit3 |

**标志位义**（低位为 bit 0）：

| 字段 | 位 | 含义 |
| --- | --- | --- |
| `TEST_FLG` | 0 | 测试中检到告警 |
| | 1 | `RESULT` 无效（测试执行了但无数值） |
| | 2 | 结果不可靠 |
| | 3 | 超时 |
| | 4 | 测试未执行 |
| | 5 | 测试中止 |
| | 6 | 无 pass/fail 指示（此时 bit7 无意义） |
| | 7 | 失败（`0` = 通过；仅 bit6=0 时有效） |
| `PARM_FLG` | 0 | 缩放错误 |
| | 1 | 漂移 |
| | 2 | 振荡 |
| | 3 | 高于高限 |
| | 4 | 低于低限 |
| | 5 | 通过备用限 |
| | 6 | 结果恰等于低限（`0`=fail，`1`=pass） |
| | 7 | 结果恰等于高限（`0`=fail，`1`=pass） |
| `OPT_FLAG` | 0 | `RES_SCAL` 无效（沿用该 `TEST_NUM` 首条默认值） |
| | 1 | 保留，**必须为 1** |
| | 2 | 无 `LO_SPEC` |
| | 3 | 无 `HI_SPEC` |
| | 4 | `LO_LIMIT` / `LLM_SCAL` 无效 |
| | 5 | `HI_LIMIT` / `HLM_SCAL` 无效 |
| | 6 | 本测试无低限 |
| | 7 | 本测试无高限 |

**写出策略**：

- pass：`TEST_FLG=0x00`；fail：`TEST_FLG=0x80`（bit7=1、bit6=0）；未执行：bit4=1。
- 结果高于高限置 `PARM_FLG` bit3，低于低限置 bit4。
- 无低限时置 `OPT_FLAG` bit4 + bit6；无高限时置 bit5 + bit7；无限值字段写 `0.0`。
- 规范允许同一 `TEST_NUM` 的后续 `PTR` 省略 `OPT_FLAG` 之后的字段并沿用首条默认值，但实测主流解析器（`semi-ate-stdf`、`pystdf`）都不做跨记录继承。**每条 `PTR` 必须写满 `OPT_FLAG…HI_SPEC` 全段**。

**SD-FMT-012**（MUST）：**Die XY 与 wafer map 重建**：

- Die 坐标只存在于 `PRR.X_COORD` / `PRR.Y_COORD`（`I2`，有符号）。`PIR` **没有** XY，也没有 `PART_ID`。
- 每片 wafer 的数据由一对 `WIR` / `WRR` 界定；`WIR.WAFER_ID` 标识该片。
- 每个 die 必须恰好有一对 `PIR` / `PRR`，同一片内 `(X_COORD, Y_COORD)` 必须唯一。
- 仅凭 `FAR` / `MIR` / `WIR` / `PIR` / `PTR` / `PRR` / `WRR`（以及可选的 `WCR`）必须能重建 wafer map。
- **v1 不存在 `yms_extended` profile、`profile` 配置项与 `include_xy_in_ptr`**；不写自定义 `GDR`。

**SD-FMT-013**（MUST）：**记录顺序与配对不变式**。合法记录链（v1 默认集合）：

```
FAR MIR [WCR] ( WIR ( PIR PTR* PRR )* WRR )* [TSR*] [HBR*] [SBR*] PCR MRR
```

- `FAR` 必须是首条；`MIR` 紧随其后；`MRR` 必须是末条。
- v1 不写 `ATR`。若将来启用 `RDR` / `SDR`：`RDR` 必须紧随 `MIR`，`SDR` 紧随 `MIR` / `RDR`。
- `WIR` / `WRR` 必须成对，且 `HEAD_NUM` / `SITE_GRP` 一致。
- `PIR` / `PRR` 必须成对，且 `HEAD_NUM` / `SITE_NUM` 一致。
- `PTR` 必须位于其所属 `PIR` 与 `PRR` 之间；不得出现在 `WIR` / `WRR` 之外。
- `TSR` / `HBR` / `SBR` / `PCR` 必须位于所有 wafer 块之后、`MRR` 之前。
- `WCR` 在 `per_wafer` 模式下位于 initial sequence（`MIR` / `RDR` / `SDR`）之后、首个 `WIR` 之前；`per_lot` 模式下不写。

**SD-FMT-014**（MUST）：**数值、类型与字节序**：

| 项 | 要求 |
| --- | --- |
| 字段类型 | 严格按规范固定（`PTR.RESULT` / 限值一律 `R4`，标志 `B1`，`TEST_NUM` `U4`，XY `I2`）；**不提供 `data_type` 配置** |
| 字节序 | 小端；`FAR.CPU_TYPE=2`；必须使用 `BinaryPrimitives`（不得用依赖主机字节序的 `BitConverter`） |
| 量化 | 按 `TestItemDef.precision` 舍入后再转 `R4`；显示精度由 `C_RESFMT` / `C_LLMFMT` / `C_HLMFMT` 承担 |
| 缩放 | `RES_SCAL = LLM_SCAL = HLM_SCAL = 0`；存储值使用 `UNITS` 的基础单位 |
| 非法值 | 未测 / 无效必须用 `TEST_FLG` 标记；**不得写 `NaN` / `Inf`** |
| `TEST_NUM` | 同一 program 多次导出必须稳定 |

**SD-FMT-015**（MUST）：**记录长度与计数一致性**：

- 每条记录的 `REC_LEN` 必须等于头部之后的实际负载字节数。
- `PRR.NUM_TEST` 必须等于该 die 对应的 `PTR` 条数。
- `PRR.HARD_BIN` 使用产品 hard bin；未配置 soft bin 时 `PRR.SOFT_BIN` 写 `65535`（规范的"无软 bin"缺失值）。
- `PRR.PART_FLG`：pass 写 `0x00`，fail 写 `0x08`（bit3=1、bit4=0）；正常完成时 bit2=0；bit4 只在"测试完成但无 pass/fail 指示"时置 1。
- `PCR`（每文件）与 `WRR`（每 wafer）的计数必须与实际数据一致：

| 字段 | v1 取值 |
| --- | --- |
| `PART_CNT` | 该范围内 `PIR` 数（= `PRR` 数） |
| `GOOD_CNT` | `HARD_BIN` 为 pass bin 的 `PRR` 数 |
| `RTST_CNT` | `0`（v1 不生成重测） |
| `ABRT_CNT` | `0` |
| `FUNC_CNT` | `0`（v1 只做参数测试） |

- `HBR.HBIN_CNT` / `SBR.SBIN_CNT` 必须等于对应 bin 的 `PRR` 数；`HBIN_PF` / `SBIN_PF` 写 `P`（pass bin）或 `F`（fail bin）。
- `TSR` 每测试项一条：`EXEC_CNT` = 该测试项的 `PTR` 数，`FAIL_CNT` = 其中 `TEST_FLG` bit7=1 的条数，`ALRM_CNT` = bit0=1 的条数。
- **不需要两遍写或随机访问回填**：所有计数记录都位于被统计数据之后，writer 必须单遍流式写出；`Finalize()` 只做校验，不修改已写字节。
- 校验失败必须报错并使导出失败，**不得产出计数不符的文件**。

**SD-FMT-016**（MUST）：**压缩**：

- 默认 `compression: none`。
- `compression: gzip` 时对**整个文件**做 gzip（.NET `GZipStream`），文件名扩展为 `.stdf.gz`。
- 压缩与未压缩产物必须语义等价（记录直方图与关键字段一致）。
- **不存在记录级压缩**；v1 不支持 bz2 / xz。

**SD-FMT-017**（MUST）：**文件名与内容一致性**：

- `WIR.WAFER_ID` 必须与文件名模板渲染出的 `wafer_id` 一致。
- `MIR.LOT_ID` 必须与文件名模板渲染出的 `lot_id` 一致。
- 人工改名导致不一致时，`verify` 必须报错。

**SD-FMT-018**（MUST）：**拆分粒度**：

| 模式 | 结构 | 默认 |
| --- | --- | --- |
| `per_wafer` | 一片一个文件：`FAR MIR WCR WIR (PIR PTR* PRR)* WRR [汇总] PCR MRR` | ✅ |
| `per_lot` | 一个 Lot 一个文件：`FAR MIR ( WIR (PIR PTR* PRR)* WRR )* [汇总] PCR MRR` | — |

- `per_lot` 要求同一 Lot 内所有 wafer 的 die 几何 / 朝向一致；不一致时导出必须失败并提示改用 `per_wafer`。
- `per_lot` 不写 `WCR`：`WCR` 的 Frequency 是每文件一条且只承载单个 `WAFER_ID`，多 wafer 文件写它会误导消费者；配置中的 `WCR` 被忽略并记录 WARN。
- 两种模式的记录链都必须通过 `SD-FMT-013` 的不变式校验。

**SD-FMT-019**（MUST）：**性能与实现要求**：

| 要求 | 说明 |
| --- | --- |
| 缓冲区 | 必须使用 `Span<byte>` / `ArrayPool<byte>` / 预分配缓冲区，避免逐字段装箱与 GC 压力 |
| 吞吐 | 见 §10.2 的 STDF 写入吞吐指标 |
| 流式 | writer 必须单遍流式，不依赖可 Seek 的输出流 |

**SD-FMT-020**（MUST）：**插件配置键**（`export.plugins.stdf`）：

```yaml
stdf:
  stdf_version: 4                    # v1 唯一允许值
  compression: none                  # none | gzip
  split: per_wafer                   # per_wafer | per_lot
  optional_records: [WCR, HBR, TSR]  # 可选：WCR / HBR / TSR / SBR / SDR / RDR
  naming_profile: internal           # internal | foundry | customer
```

- `stdf_version` 在 v1 只接受 `4`；`E10` / `E5` / `X8` / `X14` 等值必须被 Schema 拒绝。
- `optional_records` 只接受可选记录名；把必需记录（`FAR` / `MIR` / `WIR` / `PIR` / `PTR` / `PRR` / `WRR` / `PCR` / `MRR`）写进去必须报错（防止拼写错误静默失效）。
- `naming_profile` 只影响 `PTR.TEST_TXT`，不影响任何结构字段。

**SD-FMT-021**（MUST）：**追溯字段**（配合 `doc/requirements-spec.MD` `SD-FMT-044`）：

- `MIR.JOB_NAM` 写入工作簿名；超出 `C*n` 实际上限时截断并记录 WARN。
- `MIR.JOB_REV` 写入 `configHash` 的前 8 位十六进制。
- `MIR.USER_TXT` 写入 `SYNTHETIC`（配合 `SD-NFR-110`）；不写 `workbookId` / `preset` / 生成时间 / 工具版本等追溯细节，这些只进 `export_manifest.json`。
- **strict 档只使用标准字段**：不使用 `GDR` 或任何自定义记录承载追溯信息。

**SD-FMT-022**（MUST）：**时间字段与可复现性**：

- 时间源：配置 `run.generated_at`（ISO 8601 UTC）。非空时，所有 STDF 时间字段必须由它确定性推导；为空时退回任务启动时间，并在 `export_manifest.json` 标记 `time_source: "wall_clock"`（此时不满足 `DQ-16` / `DQ-21`）。
- 记 `W(i)` = 第 `i` 片 wafer 的建模测试时长 = `Σ(PRR.TEST_T)` + `wafer_overhead_s`（来自 `execution` 配置，默认 0）。
- 映射：
  - `MIR.SETUP_T` = `MIR.START_T` = `run.generated_at` 的 epoch 秒；
  - `PRR.TEST_T` = `DieRecord.test_time_ms`；
  - `WIR.START_T(i)` = `run.generated_at` + `Σ_{j<i} W(j)`；
  - `WRR.FINISH_T(i)` = `WIR.START_T(i)` + `W(i)`；
  - `MRR.FINISH_T` = 本文件最后一个 wafer 的 `WRR.FINISH_T`。
- 同一全局时间线必须同时适用于 `per_wafer` 与 `per_lot`：同一 wafer 在两种拆分下的时间值必须一致；不得按文件重新从 `generated_at` 起算。
- `run.generated_at` 是配置快照的一部分；固定它时，相同配置 + 相同 seed ⇒ STDF 字节级一致。


## 修订记录

| 日期 | 修订内容 | 依据 |
| --- | --- | --- |
| 2026-10-16 | 初始契约导入 | — |
| 2026-10-20 | 删除不存在的 `HDR` / `SMR` / `DIE` / `TCR` / `HIL` / `CPS`；撤销 `HDR` 计数回填；更正 `PTR` 标志位与缩放语义；XY 移到 `PRR`；`PCR` 改必需、`HBR` 改可选；压缩改为整文件 gzip；规格来源改为 STDF V4 | `doc/research/stdf-strict-minimal-subset.md`（#4） |
| 2026-10-20 | 删除 `yms_extended` profile；v1 只写 V4；确定可选记录 / 压缩 / `per_lot` 默认值；追溯改走 `MIR.JOB_NAM` / `JOB_REV`；清理 `E5` / `X8` / `X14` 误称；`records` 改为 `optional_records` | #16 裁决 |
| 2026-10-20 | 新增 `SD-FMT-022`：STDF 时间字段由 `run.generated_at` + 确定性测试时长推导，保证同配置同 seed 的字节级可复现 | #16 裁决 |
