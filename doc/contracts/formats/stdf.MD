# SiliconDraft 对外契约 · STDF 输出规格（CP）

> **契约性质**：STDF 是交付给 YMS / Yield Viewer 的**输出契约**。本文件定义记录集合、字段映射与 profile 语义；**规格冻结前不得变更**。
> **规格来源**：SEMI **E10**。
> **本文档不是需求**：需求（CP 导出必须符合 E10、必须支持 wafer map 可重建）见 `doc/requirements-spec.MD` §6.5；写入算法实现见 `doc/design.MD`。


**SD-FMT-010**（MUST）：STDF 记录集合必须包含以下记录（SEMI E10）：

| 记录 | 必需性 | 说明 |
| --- | --- | --- |
| `HDR` | **必需（必须唯一，必须在文件首）** | 版本、创建时间、记录计数 |
| `SMR` | 必需 | 产品、客户、作业、测试机、程序 |
| `PIR` | 必需 | 每 Wafer 一条 Part Information |
| `HBR` | 必需 | 测试机 / 探针台配置 |
| `WIR` | 必需 | Wafer 信息（wafer_id、lot_id、wafer 计数） |
| `PCR` | SHOULD | Pin 配置 |
| `RDR` | SHOULD | 复测信息 |
| `MRR` | **必需（必须唯一）** | 多结果汇总（含 Retest） |
| `PTR` | **必需（核心）** | 参数测试结果；每个 Die × 每个测试项一条 |
| `GDR` | SHOULD | 通用数据（WAT 空间统计；CP 场景下的扩展元数据） |
| `DIE` | SHOULD（默认开启） | 部分 YMS 需要 part 级摘要 |
| `TCR` / `TSR` | 可选 | 测试计时汇总 |
| `PMR` | 可选 | 多工位引脚复用 |
| `HIL` | 可选 | 硬件实例 |

**SD-FMT-011**（MUST）：必须严格按 E10 规范实现**字段顺序与类型**：

| PTR 字段 | 来源 | 备注 |
| --- | --- | --- |
| `TEST_NUM` | `TestItemDef.test_num` | **必须稳定**（同一 program 多次导出数值一致） |
| `HEAD_NUM` | 固定 `1` | — |
| `SITE_NUM` | `DieRecord.site_id` | — |
| `TEST_FLG` | 判定 / 告警位（bit0=alarm、bit1=invalid、bit3=not executed、bit4=result invalid、bit6=fail） | 必须正确使用 bit 语义 |
| `PARM_FLG` | 参数标志（超限、异常等） | — |
| `RESULT` | `DieRecord.values[item]` | 按 `data_type` 转换 |
| `TEST_TXT` | `TestItemDef.item_id`（或按 `naming_profile` 输出别名） | 测试项名 |
| `ALARM_ID` | 空 | — |
| `OPT_FLAG` | 空 | — |
| `RES_SCAL` | 由 `precision` 推导（数量级缩放） | 必须与 `LLM/HLM` 一致 |
| `LLM` / `HLM` | `lsl` / `usl` | 按 `RES_SCAL` 缩放 |
| `LO_SPEC` / `HI_SPEC` | `lsl_warn` / `usl_warn` | 预警限 |
| `UNITS` | `TestItemDef.unit` | — |

**SD-FMT-012**（MUST）：**XY 坐标扩展** —— 标准 PTR 不含 Die XY 坐标。必须按以下策略之一实现 Wafer Map 可重建，通过 `profile` 配置选择：

| profile | 策略 | 兼容性 |
| --- | --- | --- |
| `strict`（默认） | 只写标准记录；Die 坐标通过 `PIR` 的 `PART_ID` + `X_COORD/Y_COORD` 字段（若 YMS 支持）或由 `WIR` + Die 序号隐式恢复 | 最强兼容，但部分 YMS 无法直接出 Map |
| `yms_extended` | 在标准记录之外，追加自定义 **`GDR`** 记录承载 `DIE_XY` 表（`TEST_TXT` 定义为 `SD_DIE_XY`） | 兼容性略低但可被自定义解析器读取 |

**SD-FMT-013**（MUST）：`profile: yms_extended` 时必须保证：自定义记录**不破坏标准记录的字段顺序与长度**（`GDR` 是标准记录，仅语义自定义），且解析器忽略未知 `GDR` 仍可正常读取标准部分。

**SD-FMT-014**（MUST）：**数值与舍入**必须统一：

| 项 | 要求 |
| --- | --- |
| 内部计算 | 全程 `f64` |
| 序列化 | 按 `data_type`（`R4`/`R8`/`U4`/`I2`）转换；默认 `R4` |
| 舍入 | 按 `TestItemDef.precision` 舍入（避免跨平台差异导致不可复现） |
| 字节序 | 默认 `little_endian`；**必须使用 `BinaryPrimitives` 而非 `BitConverter`**（后者依赖主机字节序，会导致跨 RID 字节不一致） |
| 单位与刻度 | `RES_SCAL` 必须与 `LLM/HLM/LO_SPEC/HI_SPEC` 保持一致 |
| 非法值 | 未测 / 无效必须使用 `TEST_FLG` 标记，**不得写入 `NaN`**（STDF 无 NaN 语义） |

**SD-FMT-015**（MUST）：**`HDR` 一致性** —— `HDR` 中的各类记录计数（`D_CNT`/`H_CNT`/`P_CNT`/`R_CNT`/`T_CNT` 等）必须与实际写入的记录数一致。实现要求：**写文件头时预留位置，`Finalize()` 时回填**（两遍写或随机访问回填）；必须处理"输出流不可 Seek"的情况（若目标为管道 / 网络流，则必须先在临时文件完成再搬运）。

**SD-FMT-016**（MUST）：**字节序与压缩** —— 默认小端；`compression: true` 时按 E10 压缩规则（各数值字段前带 1 字节长度）实现，且必须与 `false` 模式产出**语义等价**的可解析文件。

**SD-FMT-017**（MUST）：**文件名与内容一致性** —— STDF 内部的 `WIR.WAFER_ID` / `SMR.LOT_ID` 必须与文件名模板渲染出的 `lot_id`/`wafer_id` 一致（防止人工改名后内容不匹配）。

**SD-FMT-018**（MUST）：**拆分粒度** —— 支持 `per_wafer`（默认，一片一个 STDF）与 `per_lot`（一个 Lot 一个 STDF，含多 Wafer 的 `WIR`/`PIR` 序列）两种模式；两种模式的**记录链必须都合法**。

**SD-FMT-019**（MUST）：**性能与实现要求**：

| 要求 | 说明 |
| --- | --- |
| 缓冲区 | 必须使用 `Span<byte>` / `ArrayPool<byte>` / 预分配缓冲区，避免逐字段装箱与 GC 压力 |
| 吞吐 | 见 §10.2 的 STDF 写入吞吐指标 |

