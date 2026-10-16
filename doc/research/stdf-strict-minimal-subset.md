# STDF strict 最小子集的记录与字段要求（调研笔记）

> 对应 GitHub issue **#4**（wayfinder research 票）/ `doc/contracts/formats/stdf.MD`、`doc/requirements-spec.MD` §6.6、§6.7、§14（SD-Q-002/013/014/015）、`doc/design.MD` §6 与 §12 `SD-EXT-103`。
> 日期：2026-10-16。实测脚本：`/tmp/sd-research/{make_minimal_stdf.py, parse_check.py}`（未入库，可复现，见附录 B）。
> 信源分级：**[规范]** 公开规范镜像原文；**[源码]** 解析器/工具源码；**[实测]** 本机 Python 3.9.6 实测；**[未证]** 未找到权威来源（只给采信逻辑，不编造条文）。
> 版权说明：本笔记只引用公开可得的字段定义与结构（短句引文），不含任何第三方机密材料。

**信源清单**

| 编号 | 信源 | 说明 |
| --- | --- | --- |
| S1 | STDF Specification V4（Teradyne） | 公开镜像：`github.com/lantianjialiang/stdf_spec`（`Stdf-V4-spec.pdf`）、freestdf SVN `svn.code.sf.net/p/freestdf/svn/docs/stdf-spec.pdf`、NI 论坛附件。本笔记页码均指该规范 |
| S2 | STDF V4-2007 | 同上镜像仓库 `stdf_V4-2007_spec.pdf` |
| S3 | Semi-ATE/STDF 库（`semi-ate-stdf`） | PyPI 0.1.28（2022-06-21，MIT）；仓库 `github.com/Semi-ATE/STDF`（MIT，main 最近提交 2024-12-20，75★）。源码：`STDFFile.py`、`STDR.py`、FAR/PTR/PRR/… 各 record 类 |
| S4 | pystdf | PyPI 1.4.0（2026-08-20，`requires-python >=3.8`）；仓库 `github.com/cmars/pystdf`（GPL-2.0，最近 push 2026-08-20，184★） |
| S5 | pystdf4 | PyPI 0.1.2（2025-11-17，MIT）；仓库 `github.com/Asachoo/pystdf4`（3★），README 声明 reader 开发中 |
| S6 | stdfast | PyPI 0.2.1（2026-04-01，BSD-2-Clause，`requires-python >=3.10`）；仓库 `github.com/gluek/stdfast` |
| S7 | Marklio.Stdf（C#） | 仓库 `github.com/marklio/Marklio.Stdf`（MIT，.NET 10，最近 push 2026-04-17，未发布 NuGet） |
| S8 | libstdf / FreeSTDF | `freestdf.sourceforge.io`（C 库；`is_valid_stdf.c` 校验样例；规范镜像） |
| S9 | Nornion STDF Workshop 手册 | 公开工具手册（nornion.com）；其「检查完整性、自动补齐缺失记录」的描述用于旁证消费端预期（低可信度来源） |
| S10 | SEMI 官方商店 | `store-us.semi.org`：E10 = 设备 RAM 标准；G91 = STDF Memory Fail Datalog（Inactive） |
| S11 | SemiEngineering 报道 | STDF 治理史（Teradyne 2010 授权 SEMI 管理；2019 STDF 任务组转向 RITdb） |
| S12 | TofuPilot 导入文档 | 公开消费端声明（文件以 FAR 开始、以 MRR 结束） |
| S13 | 本仓库契约文档 | `doc/contracts/formats/stdf.MD`（对照对象，未修改） |
| S14 | 本机实测 | 2026-10-16，macOS，Python 3.9.6，`semi-ate-stdf 0.1.28` + `pystdf 1.4.0` 双解析器对拍 |

---

## 结论摘要

### 最小子集记录清单

「是否必须」分两列理解：**规范级**（不写就不是合法 STDF 文件）与**实务级**（本产品 CP/WAT 输出的推荐档，置信度见备注）。

| 记录 | 是否必须 | 理由 | 出处 |
| --- | --- | --- | --- |
| `FAR` (0,10) | **必须**（规范） | 每文件恰好一条，且必须是首记录；携带 `CPU_TYPE`（读方据此判定字节序）与 `STDF_VER` | [S1] p.14「Initial Sequence」、p.82「Required Records」、FAR 节；[S3] STDR.py 记录表 |
| `MIR` (1,10) | **必须**（规范） | 每文件恰好一条，紧随 FAR（及 ATR）之后；承载 lot/product/job/机台等全局信息 | [S1] 同上、MIR 节 |
| `PCR` (1,30) | **必须**（规范，≥1 条） | 每文件至少一条：`HEAD_NUM=255` 汇总或每 head/site 一条或两者；位于 initial sequence 之后、MRR 之前 | [S1] p.82、PCR 节 |
| `MRR` (1,20) | **必须**（规范） | 每文件恰好一条，且必须是文件最后一条记录 | [S1] p.82、MRR 节 |
| `WIR` (2,10) | 规范可选；CP/WAT **实务必须** | 每片 wafer 一条，标记该片数据开始；WIR/WRR 夹住整片数据；缺它无法把 die 数据归到 wafer | [S1] WIR 节（Frequency: One per wafer tested，仅 wafer probe 场景）；[S3] WIR.py docstring「Obligatory for Wafer sort」 |
| `WRR` (2,20) | 规范可选；CP/WAT **实务必须** | 每片 wafer 一条；承载该片 PART/RTST/ABRT/GOOD/FUNC_CNT 与 WAFER_ID（规范原文：WAFER_ID 在 WRR 中 strongly recommended） | [S1] WRR 节 |
| `PIR` (5,10) | 规范可选；有 part 数据时**实务必须** | 每个被测 part/die 一条；每个 PIR 必须有配对 PRR（同 HEAD/SITE）；PTR/MPR/FTR 必须落在这对 PIR/PRR 之间 | [S1] PIR 节；[S8] `is_valid_stdf.c` 校验逻辑 |
| `PTR` (15,10) | 规范可选；**本产品核心必须** | 参数测试结果载体（CP/WAT 数值主体）；每条位于对应 PIR/PRR 对之间 | [S1] PTR 节；[S13] SD-FMT-010 |
| `PRR` (5,20) | 规范可选；**实务必须** | 每个 die 一条；承载 `HARD_BIN`/`SOFT_BIN`/`X_COORD`/`Y_COORD`/`PART_ID`——bin 分配与 wafer map 的事实载体 | [S1] PRR 节 |
| `WCR` (2,30) | 可选（**推荐**） | 每文件一条；提供 wafer 直径、die 尺寸、方向（WF_FLAT/POS_X/POS_Y）——wafer map 的几何与朝向 | [S1] WCR 节 |
| `TSR` (10,30) | 可选（**推荐**） | 每个测试一条执行/失败/告警计数；规范原文称 EXEC_CNT/FAIL_CNT「optional, but strongly recommended」；支撑良率/DPPM 汇总 | [S1] TSR 节 |
| `HBR` (1,40) | 可选（**推荐**） | 每个硬件 bin 一条计数（HBIN_NUM/HBIN_CNT/HBIN_PF）；YMS 良率表首选来源 | [S1] HBR 节 |
| `SBR` (1,50) | 可选 | 软件 bin 计数；无软 bin 语义可省 | [S1] SBR 节 |
| `RDR` (1,70) | 可选 | 仅重测（retest）数据文件需要；若使用必须紧随 MIR | [S1] p.14、RDR 节（V4 原文「Optional」；[S3] 的 V4-2007 文案为「Obligatory if a lot is retested」——按版本引用） |
| `SDR` (1,80) | 可选 | 每 site/site 组一条配置描述；单 site 不需要 | [S1] SDR 节 |
| `ATR` (0,20) | 可选 | 过滤/变换审计记录；本产品不需要 | [S1] ATR 节 |
| `MPR` (15,15) | 可选 | 向量/多点结果的标准载体（`RTN_RSLT` + `RSLT_CNT`）；WAT 多点曲线可用（消费端支持度未逐一验证） | [S1] MPR 节 |
| `FTR`/`BPS`/`EPS`/`DTR`/`PMR`/`PGR`/`PLR` | 可选 | 功能测试/程序段/文本/引脚映射；本产品不需要 | [S1] 各节 |
| `GDR` (50,1) | 可选 | 通用数据记录（自定义语义的标准外壳） | [S1] GDR 节 |
| `VUR` (0,30) | **仅 V4-2007 必须** | V4-2007 规定每文件必须含一条 VUR，位于最后一条 ATR 之后、MIR 之前；写经典 V4（`STDF_VER=4`）则不需要 | [S2]「A note on Initial Record Sequences」 |
| `HDR` / `SMR` / `DIE` / `TCR` / `HIL` | **记录不存在** | V4 / V4-2007 / ATDF 三份规范全文均无这些记录；V4 首记录是 FAR 而不是 HDR；die 级摘要由 PIR/PRR 承载（无独立 DIE 记录）；规范只有 `TSR` 没有 TCR | [S1] [S2] 全文检索（0 命中）；[S3] STDR.py 记录表；[S7] README「All 32 V4/V4-2007 record types」无 HDR/TCR |
| `CPS` / 记录级压缩 | **不存在** | 规范无任何压缩记录、压缩标志或「字段前带长度字节」的编码；实际世界的压缩是**整文件** gzip/bz2/lzma | [S1] [S2] 全文检索（compress/gzip/zip 0 命中）；[S3] README/STDFFile.py（文件名+magic 嗅探）；[S4] stdf2text（按扩展名）；[S7] README |

### 五条主结论（要点）

1. **规范级必需恰好 4 条：FAR、MIR、PCR(≥1)、MRR**（置信度：高）。规范原文（V4 p.14）：「Every file must contain one FAR, one MIR, one or more PCR, and one MRR. All other records are optional.」p.82 重复该结论并逐条给出条件。规范同时声明：required/optional **只定义「合法文件」**，分析软件可要求规范里标记为可选的字段（p.2「Teradyne's Use of the STDF Specification」）——所以「规范最小」≠「被 YMS 接受」。
2. **本产品（CP/WAT）推荐 strict 档 = FAR → MIR → WCR → WIR → (PIR → PTR\* → PRR)\* → WRR → TSR\* → HBR\* → PCR → MRR**（置信度：高，顺序直接取自规范 p.70 的 wafer-probe 示例）。WCR/HBR/TSR 为推荐可选；SBR/SDR/RDR 按需。
3. **契约中的 `HDR`/`SMR`/`DIE`/`TCR`/`HIL`、`HDR` 计数回填（D_CNT/H_CNT/…）、E10 记录级压缩均无规范依据**（置信度：高，见 §3、§4、§7）。按现契约实现会产出无法被任何标准解析器读取的文件。
4. **双解析器实测**：以上推荐集（含去掉 WCR/TSR/HBR/SBR 的极端最小版、gzip 版、RDR 版）被两个互相独立的解析器 **完全一致** 地解码；而首记录非 FAR 的伪文件：`semi-ate-stdf` 抛 `IndexError`（响亮失败），`pystdf` **静默返回 0 条记录、不报错**——CI 交叉校验必须断言记录计数与关键字段，不能只断言「解析没抛异常」。
5. **PTR 的「省略默认字段尾部」在消费端不生效**：规范允许后续 PTR 省略 OPT_FLAG 之后的字段（沿用该 TEST_NUM 首条 PTR 的默认值），但实测两个解析器都**不做跨记录默认值继承**——省略字段被解为「无效」（semi-ate: OPT_FLAG 全位=无效、限值 0、UNITS 空；pystdf: 全部 `None`）。⇒ **写出策略：每条 PTR 都写满尾部字段**，不要依赖默认值机制（详见 §2.4）。

---

## 1.（票面 1）哪些记录是必须的

### 1.1 规范事实（[S1]，置信度：高）

- 必需集 = FAR / MIR / PCR / MRR，出处 p.14 与 p.82（两处独立成文）。
- 合法 initial sequence 只有 8 种：`FAR–MIR`、`FAR–ATRs–MIR`、`FAR–MIR–RDR`、`FAR–ATRs–MIR–RDR`，以及各自带 `–SDRs` 尾的 4 种（p.14）。ATR 必须在 FAR 之后立即出现；RDR 必须在 MIR 之后立即出现；SDR 紧随 MIR/RDR 之后。
- 各记录 Frequency/Location（规范各记录节）：
  - WIR/WRR：one per wafer tested（仅 wafer probe 场景使用）；
  - PIR/PRR：one per part tested；PRR 定义「after the corresponding PIR and before the MRR」；
  - WCR：one per STDF file（wafer 测试时）；
  - PTR：one per parametric test execution；TSR：one for each test executed；
  - HBR/SBR：one per bin（per site 或汇总）；RDR：Optional（重测场景见 [S3] 的 V4-2007 文案「Obligatory if a lot is retested」）；
  - ATR：Optional；SDR：one per differently-configured site group；PCR：at least one（HEAD_NUM=255 汇总或逐 head/site 或两者）。
- 规范 p.67 的合法组合示例：① packaged lot 数据流（FAR/MIR/(PIR/PTR/PRR)\*/TSR\*/HBR\*/SBR\*/PCR/MRR）；② 只要 summary 的文件；③ wafer probe（FAR/MIR/WCR/(WIR/(PIR/PTR/PRR)\*/WRR)\*/TSR\*/HBR\*/SBR\*/PCR/MRR）。第③种即 CP/WAT 的直接模板。
- 结构约束（p.82 与各节）：PTR/MPR/FTR 必须位于某对 PIR/PRR 之间；WIR/WRR 成对且同 HEAD_NUM/SITE_GRP 值；MRR 最后。
- 独立的公开校验实现（[S8] `is_valid_stdf.c`，置信度：中高）检查的正是：首记录必须 FAR、MIR 必须存在、PIR 必须有配对 PRR、PTR/MPR/FTR 必须位于正确的 PIR/PRR 对之间——与本笔记的集合判断一致。

### 1.2 主流解析器实测（[S14]，置信度：高——本机实测）

对拍文件与结果见附录 A。要点：

- v1（含 WCR/TSR/HBR/SBR 的全集，2 wafer/6 die/12 PTR）与 v2（**去掉 WCR/TSR/HBR/SBR 的最小版**）、v3（无 wafer 结构、final-test 风格）、v5（额外 RDR）、v1 的 gzip 版：两个解析器记录直方图**逐一相等**、关键字段解码值一致。
- 结论：**主流解析器不强制 HBR/SBR/TSR/WCR/RDR**（规范怎么写就怎么收）。YMS 侧是否接受则无公开证据（见 1.3）。
- 反例（x1）：把首记录换成自造的「HDR」= 非标准 (0,1) 记录 → `semi-ate-stdf` 直接抛异常；`pystdf` 把整文件当垃圾、**返回 0 条记录且不报错**。⇒ 一旦产出非标记录，消费者表现从「报错」到「静默丢数据」都有，必须靠断言兜底。

### 1.3 YMS 侧要求：**未找到权威来源**

对 PDF Solutions Exensio、yieldWerx、KLA Klarity、Synopsys Yield Explorer / Silicon.da、Galaxy Semiconductor（Examinator）、Onto Innovation、Siemens Opcenter/Tessent 的公开材料（产品页、datasheet、SEC 文件、公开会议材料）逐一排查：**没有任何一家公开说明导入 STDF 需要哪些记录**；公开表述只到格式级（「支持 STDF V4-2007」「支持 STDF + 120 种格式」）。用户手册普遍登录墙（Exensio/SolvNet/Siemens 支持站）。旁证仅两条（置信度：低—中）：
- TofuPilot 公开导入文档（[S12]）：文件「以 FAR 开始、以 MRR 结束」，以 PIR/PRR 映射其 run 模型；
- Nornion STDF Workshop 手册（[S9]）：把「缺失 MIR/MRR/PCR 类记录」视作不完整文件并可自动补齐。

⇒ **判断**：YMS 的硬性要求无法从公开信息核定，SD-Q-001（目标 YMS 是哪家）不落地前，「strict 档」只能按「规范必需 + 通行惯例（WIR/WRR/PIR/PTR/PRR/HBR/TSR）」保守设计；因此保留 `records` 配置开关是合理的，但默认集合必须换成合法记录名（见 §7.7）。

### 1.4 推荐的 strict 档记录集（CP 与 WAT）

按规范 p.70 模板：

```
CP（per_wafer 或 per_lot）:
FAR MIR [WCR] ( WIR ( PIR PTR* PRR )* WRR )* [TSR*] [HBR*] [SBR*] PCR MRR

WAT（结构级、多点）:
同一骨架；wafer 上的 PCM/测试结构按「part」处理：
  PIR/PRR 每结构一条（X/Y = 结构坐标，HARD_BIN 记结构 pass/fail）；
  单点值用 PTR；一条曲线/多点用 MPR（RTN_RSLT+RSLT_CNT）或拆成多条 PTR（最保守）。
```

- WAT 注意（置信度：中，工程判断）：WAT 原始数据在业界大量以 CSV/PCM 格式存在，若客户要求 STDF，以上骨架仍成立；「多点」若用 MPR，需在 SD-Q 里与目标 YMS 核实其 MPR 支持（本笔记未实测 MPR 路径）。
- CP 注意：`WCR` 强烈建议写入（die 尺寸+方向决定 wafer map 的物理还原），否则 map 只能按格点还原、无比例信息。

---

## 2.（票面 2）PTR 字段语义

（本节除标注外，均出自 [S1] PTR 节 p.45–50，置信度：高；实测见 [S14]）

### 2.1 字段表（顺序即写入顺序）

| 字段 | 类型 | 说明 / 缺失语义 |
| --- | --- | --- |
| `TEST_NUM` | U\*4 | 测试编号（本产品要求跨导出稳定，是产品约定而非规范要求） |
| `HEAD_NUM` | U\*1 | 单机台无并行时应写 `1`（规范原文 should be set to 1） |
| `SITE_NUM` | U\*1 | 同上；并行测试时与 PIR/PRR 配对 |
| `TEST_FLG` | B\*1 | 见 2.2（**契约中的 bit 映射有误**） |
| `PARM_FLG` | B\*1 | 见 2.2 |
| `RESULT` | R\*4 | IEEE754 单精度；缺失/无效标志 = TEST_FLG bit1 = 1 |
| `TEST_TXT` | C\*n | 测试项名；空 = 长度字节 0 |
| `ALARM_ID` | C\*n | 告警名；无告警可空 |
| `OPT_FLAG` | B\*1 | **后续字段的「有效性开关」；写入任何尾部字段时必须存在**（仅当它是记录最后一个字段才可省略） |
| `RES_SCAL` | I\*1 | 结果缩放指数（显示用，见 2.3）；无效标志 = OPT_FLAG bit0 |
| `LLM_SCAL` | I\*1 | 低限缩放指数；无效标志 = OPT_FLAG bit4 或 bit6 |
| `HLM_SCAL` | I\*1 | 高限缩放指数；无效标志 = OPT_FLAG bit5 或 bit7 |
| `LO_LIMIT` | R\*4 | 低测试限；无效标志 = OPT_FLAG bit4 或 bit6 |
| `HI_LIMIT` | R\*4 | 高测试限；无效标志 = OPT_FLAG bit5 或 bit7 |
| `UNITS` | C\*n | **只允许基础单位**（"AMPS"/"VOLTS"，绝不允许 "uAMPS"/"mVOLTS"，规范原文） |
| `C_RESFMT` / `C_LLMFMT` / `C_HLMFMT` | C\*n | ANSI C 格式串（如 `%7.2f`），显示精度由它们承担 |
| `LO_SPEC` | R\*4 | 低规格限（预警限）；无效标志 = OPT_FLAG bit2 |
| `HI_SPEC` | R\*4 | 高规格限；无效标志 = OPT_FLAG bit3 |

### 2.2 三个标志字段（规范原文位义）

`TEST_FLG`（bit0 = 最低位，规范 p.10 明确 low order bit = bit 0）：

| bit | 含义 |
| --- | --- |
| 0 | 1 = 测试中检到告警 |
| 1 | 1 = RESULT 无效（测试执行了但无数值；此时要看 bit6/7 判 pass/fail） |
| 2 | 1 = 结果不可靠 |
| 3 | 1 = 超时 |
| 4 | 1 = 测试未执行 |
| 5 | 1 = 测试中止 |
| 6 | 1 = 无 pass/fail 指示（即 bit7 无效）；0 = bit7 有效 |
| 7 | 1 = 失败，0 = 通过（仅在 bit6=0 时有意义） |

`PARM_FLG`：bit0 缩放错误；bit1 漂移；bit2 振荡；bit3 高于高限；bit4 低于低限；bit5 通过备用限；bit6 = 结果恰等于低限时（0=fail，1=pass）；bit7 = 恰等于高限时（0=fail，1=pass）。

`OPT_FLAG`：bit0=1 表示 RES_SCAL 无效（沿用该 TEST_NUM 首条的默认）；**bit1 保留且必须为 1**；bit2=1 无低规格限；bit3=1 无高规格限；bit4=1 表示 LO_LIMIT/LLM_SCAL 无效；bit5=1 表示 HI_LIMIT/HLM_SCAL 无效；bit6=1 本测试无低限；bit7=1 本测试无高限。⇒ 全字段有效时典型值 `0x02`；契约里 OPT_FLAG 写「空」在「后面还写字段」的前提下是非法组合。

`RESULT` 可用性判据（规范原文归纳）：TEST_FLG 的 bit0–bit5 全 0 且 PARM_FLG 的 bit0–bit2 全 0，才认为 RESULT 有用。

### 2.3 缩放与单位（规范 p.65「Storing and Displaying Parametric Test Data」）

- **存储值一律归一化到 `UNITS` 的基础单位**；`_SCAL` 只影响「显示」。
- 显示值 = `RESULT × 10**RES_SCAL`、`LO_LIMIT × 10**LLM_SCAL`、`HI_LIMIT × 10**HLM_SCAL`。
- `_SCAL` 同时是前缀代码：15=f(10⁻¹⁵)、12=p、9=n、6=u、3=m、2=%(10⁻²)、0、−3=K、−6=M、−9=G、−12=T。规范例子：UNITS=AMPS、RES_SCAL=6 → 显示 µAMPS。
- 规范明确 `RES_SCAL, LLM_SCAL, HLM_SCAL, UNITS, C_RESFMT, C_LLMFMT, C_HLMFMT` **互相依赖**；覆盖其中一个时须同步调整其余。
- 精度由 `C_RESFMT` 等格式串表达；契约里「RES_SCAL 由 precision 推导」是产品自定策略，规范没有这个映射关系（置信度：高，规范文本）。

### 2.4 默认值机制与实测（本节是写实现的关键）

规范机制（[S1] p.46「Default Data」）：每个 TEST_NUM 的**第一条** PTR 写满 OPT_FLAG 之后的字段作为默认值；后续 PTR 可省略，省略即沿用默认；覆盖仅对该条生效。规范还说：为省空间，除非覆盖默认值，否则**应该省略**这些字段。

**实测（[S14] v6 变体，2026-10-16）**：构造「test 1003 首条 PTR 写全（LO/HI=1.05/1.15、UNITS=V、RES_SCAL=0），第二条省略尾部」的合法样本：

| 解析器 | 第二条 PTR 的省略字段解码结果 |
| --- | --- |
| semi-ate-stdf 0.1.28 | `OPT_FLAG` = 全位 1（全部无效）；`LO_LIMIT/HI_LIMIT/LO_SPEC/HI_SPEC` = 0.0；`UNITS` = 空串。**没有回填默认值** |
| pystdf 1.4.0 | `OPT_FLAG/RES_SCAL/…/HI_SPEC` 全部 = `None`。**没有回填默认值** |

⇒ 两个主流解析器都按「逐记录」解码，不做跨记录默认值继承（semi-ate 甚至把「缺失的 OPT_FLAG」解为「全部无效」，这在语义上是错误的方向——缺失 ≠ 无效）。**写出策略：每条 PTR 都写满 `OPT_FLAG…HI_SPEC` 全段**；代价是文件略大，换来任何消费者上语义一致。

### 2.5 给实现的写法建议（工程判断，置信度：中高）

1. `OPT_FLAG` 恒写：全字段有效时 `0x02`；无 LO_SPEC / HI_SPEC 时置 bit2/bit3；某测试整体无低/高限时置 bit6/bit7（不要用 bit4/5 的「沿用默认」语义，因为 §2.4 已证消费端不继承）。
2. 缩放：最稳做法 `RES_SCAL = LLM_SCAL = HLM_SCAL = 0` + `UNITS` 写基础单位 + `C_*FMT` 管精度；如需 mm/µm 之类显示前缀再启用一致的正负指数（三者同步）。
3. `TEST_FLG`：pass = 0x00；fail = 0x80（bit7=1、bit6=0）；未执行 = 0x10（bit4）；数值无效 = bit1=1（不要写 NaN——规范未定义 NaN 语义，「不写 NaN 用 TEST_FLG」与规范精神一致，置信度：高）。
4. `PARM_FLG`：结果高于高限时建议置 bit3、低于低限置 bit4（analyses 工具会读）。
5. 浮点：`RESULT`/限值都是 R\*4 单精度（0.1 类十进制值会带舍入尾差，实测样本 0.5/1.2/0.4/0.6 中 0.4/0.6 解出 0.4000000059/0.6000000238）；如需十进制直读一致，用 `precision` 舍入 + `C_RESFMT` 显示，或评估在 R\*4 内选可精确表示的舍入值。
6. `HEAD_NUM/SITE_NUM` 单 site 写 `1`；`TEST_TXT` 用稳定 item_id（产品约定）。

---

## 3.（票面 3）「HDR 计数回填」的真实情况

**结论：STDF V4/V4-2007 中不存在 `HDR` 记录，也不存在文件头记录计数；`D_CNT/H_CNT/P_CNT/R_CNT/T_CNT` 等字段在任何已发布版本中都不存在。**（置信度：高）

- 首记录是 FAR (0,10)，字段只有 `CPU_TYPE` 与 `STDF_VER` 两个字节（[S1] FAR 节表格）。规范把「头」定义为**每条记录自己的 4 字节头**：`REC_LEN`(U\*2, = 头之后的字节数) + `REC_TYP`(U\*1) + `REC_SUB`(U\*1)（[S1]「STDF Record Header」p.6–7）。这 4 字节头的 `REC_LEN` 才是唯一与「长度/数量」有关的头字段，且必须逐条正确。
- 全套规范（V4、V4-2007、ATDF）与 `semi-ate-stdf` 记录表均无 HDR；无任何公开实现使用 D_CNT 等字段。
- 带「计数」的字段只存在于汇总记录里（此处也才是「finalize 一致性」的真实工作内容）：

| 记录 | 计数字段 | 写出时机（规范） |
| --- | --- | --- |
| `PCR` | `PART_CNT/RTST_CNT/ABRT_CNT/GOOD_CNT/FUNC_CNT` | initial sequence 之后、MRR 之前（实时录制时通常靠近文件尾） |
| `WRR` | 同上五类 | 对应 WIR 之后（每片结束时） |
| `HBR`/`SBR` | `HBIN_CNT`/`SBIN_CNT` | MRR 之前（通常文件尾） |
| `TSR` | `EXEC_CNT/FAIL_CNT/ALRM_CNT` | MRR 之前（通常文件尾） |

- ⇒ 对实现的含义（建议修订 SD-FMT-015，置信度：高）：
  1. **不需要两遍写/随机访问回填**。所有计数记录都位于它所统计的数据**之后**，单遍流式写即可：每片 WRR 在该片最后一条 PRR 之后写；PCR/TSR/HBR/SBR 在 MRR 前写——写出时刻计数已天然可算。这直接消除了 SD-FMT-015 里「不可 Seek 的输出流」的复杂度。
  2. 真正要保证的不变式是：每条记录的 `REC_LEN` == 实际负载字节数；`PCR.PART_CNT/GOOD_CNT` == 实际 PRR 统计；`WRR.*_CNT` == 该片统计；`HBR/SBR.HBIN_CNT/SBIN_CNT` == 各 bin 实际计数；`TSR.EXEC_CNT/FAIL_CNT` == 实际执行/失败数。这些应在 `Finalize()`（或写入时）校验并作为验收断言（与 §6 交叉校验方案联动）。
  3. 若将来确实需要「文件级总量」元数据，规范内的做法是 `GDR`（50,1）自定义文本或自定义号段（REC_TYP/REC_SUB **>200 为自定义保留**，<200 保留给 Teradyne，[S1] p.7）——但这会破坏 strict 档，默认不做。

---

## 4.（票面 4）字节序与压缩

### 4.1 字节序

- **规范不强制字节序**：writer 用自身 CPU 的原生表示，并把 CPU 类型写入 `FAR.CPU_TYPE`；reader 读数时按该字段做转换（[S1] p.10「Note on Data Representation」；FAR 节 CPU_TYPE 取值表：0=DEC、1=Sun、2=Sun 386i/IBM PC、3–127 Teradyne 保留、128–255 客户保留）。
- 契约中「E10 小端约定」的提法**未找到权威来源**——小端只是「CPU_TYPE=2（x86）」的当代惯例，不是规范条文（置信度：高；已查三份规范全文）。
- **建议：小端 + `CPU_TYPE=2`**。实测两个解析器都以 CPU_TYPE 判字节序：`semi-ate-stdf` 对 CPU_TYPE∈{1,2} 均可读（`'>'`/`'<'`），其他值直接拒绝；`pystdf` 只在 `CPU_TYPE==2` 时用小端，**其余一律按大端**（含 0/3…）——写 `2` 是唯一两个解析器都走「小端」分支的值。
- 位序：规范「low order bit = bit 0」（[S1] p.10），即低位在前、LSB 编号 0——flag 位表按此解读。

### 4.2 压缩

- **规范无任何记录级压缩、无压缩标志、无 `CPS` 记录**（三份规范全文 0 命中；置信度：高）。契约 SD-FMT-016 的「E10 压缩规则（各数值字段前带 1 字节长度）」**查无此规则**；按其实现产出的将不是 STDF（任何解析器无法读取）。
- 现实世界只有**整文件压缩**，按约定/实现支持度排序：gzip（最普及）、bz2、lzma/xz（置信度：中高，来自实现而非规范）：
  - `semi-ate-stdf`：读支持 gzip/lzma/bz2（要求文件名匹配 `stdf?.gz` 等 **且** magic 正确），写支持 gzip；
  - `pystdf` CLI/解析器：`.gz`/`.bz2` 按扩展名；
  - `Marklio.Stdf`（C#）：读自动识别 gzip/bzip2；`rust-stdf`：gz/bz2；`libstdf`：有 gzip/zip/bzip2 等 file-format 枚举。
- 实测：v1 的 gzip 版被两个解析器正常解析，计数与未压缩版完全一致。
- **建议（可修订 SD-Q-015 默认值）**：strict 档默认 **不压缩**（最大兼容；无法从公开信息确认目标 YMS 是否接受 .gz，未找到权威来源）；保留 `compression: gzip` 开关，命名 `.stdf.gz`；CI 对压缩/未压缩两个变体都跑交叉校验（语义等价断言）。

---

## 5.（票面 5）交叉校验工具

### 5.1 先纠正票面里的 `pylibstdf`

**`pylibstdf` 不存在**（置信度：高）：PyPI 404、GitHub 无对应仓库、无 conda 包。与它描述最接近的真实库是 **Semi-ATE 的 `semi-ate-stdf`**（即 `github.com/Semi-ATE/STDF`；旧名 Semi-ATE-STDF 会重定向到它）。若仓库其它文档引用了 `pylibstdf`，应统一改名为 `semi-ate-stdf` 并锁版本。

### 5.2 候选工具清单（版本/仓库/最近提交/许可证/3.9.6 可用性/实测）

| 工具 | 版本 | 仓库 · 最近提交 | 许可证 | Python 3.9.6 | 实测（2026-10-16） |
| --- | --- | --- | --- | --- | --- |
| **semi-ate-stdf** | PyPI 0.1.28（2022-06-21）；GitHub release 已到 **0.1.33**（2024-12-20，未发 PyPI） | `github.com/Semi-ATE/STDF` · main 最近提交 2024-12-20（repo pushed_at 2025-12-14，来自其他分支） · 75★ | MIT（PyPI + 仓库 LICENSE；注：源码 `STDR.py` docstring 误写「License : GPL」，与仓库元数据矛盾，建议法务复核——置信度：高（矛盾事实），中（以谁为准）） | ✅（classifiers 3.7–3.9；纯 Python，sdist 可直接构建；PyPI 依赖仅 `xlsxwriter`，核心解析只用标准库；无 CLI） | **全部合法变体（v1–v6 与 gz 版）解析 OK**；含 gz；对非法首记录抛 `IndexError`；另：对 pystdf 官方 demo 文件（4.56 MB，真实世界数据）与 pystdf 计数完全一致（59,890 条） |
| **pystdf** | 1.4.0（PyPI 2026-08-20，requires-python ≥3.8） | `github.com/cmars/pystdf` · 2026-08-20 · 184★ | GPL-2.0（PyPI classifier 另有「Free for non-commercial use」措辞） | ✅（wheel，纯 Python） | **全部合法变体（v1–v6 与 gz 版）解析 OK**；含 gz；对非法输入**静默 0 条**。CLI：`stdf2text/stdf2excel/stdf2xml/stdf_slice/rec_index`（注意：`rec_index` 只计数不打印，别拿它做 CI 计数） |
| stdf-convert | 0.3.0（2026-10-08）；**py3.9 需钉 0.1.0**（cp39-abi3 wheel；0.2.0+ 要求 ≥3.11） | `github.com/davidmathers/stdf-convert` · 2026-10-13 | MIT | ⚠️（仅 0.1.0 可装，实测 --help 可用） | 未做对拍；CLI 输出**每记录一行 JSON**（含 sequence_number 与 byte_offset），gz/bz2/zip 直读——适合用 `jq/wc` 做「记录数/字节偏移差异」第三意见 |
| pystdf4 | 0.1.2（2025-11-17） | `github.com/Asachoo/pystdf4` · 2025-11-17 · 3★ | MIT | ✅（≥3.8，零依赖） | 未用于解析：README 自述 reader 仍在开发（MPR/FTR/GDR 未完成）；写侧完整 |
| stdfast | 0.2.1（2026-04-01） | `github.com/gluek/stdfast` · 2026-04-13 | BSD-2-Clause | ❌ `requires-python >=3.10`，**3.9.6 装不上**（实测报错） | 未测 |
| **STDFParser4Net（C#）** | NuGet 0.1.1（2026-07-23） | `github.com/jzh19930420/STDFParser4Net` | MIT | —（.NET Standard 2.1，`dotnet add package` 即用） | 未做对拍；26 种标准 V4 记录（无 VUR/PSR/STR）、流式解析、感知字节序——**C# 测试工程内做第三方交叉校验的最省事选项** |
| Marklio.Stdf（C#） | 无发布 | `github.com/marklio/Marklio.Stdf` · 2026-04-17 · 未上 NuGet | MIT | —（.NET 10） | 未测；README 宣称字节级 round-trip、全部 32 种 V4/V4-2007 记录、gzip/bzip2、双字节序 |
| libstdf / FreeSTDF（C） | SourceForge | `freestdf.sourceforge.io` | 未核验（中；SourceForge 本次不可达） | —（C） | 未测；`is_valid_stdf.c` 的校验清单可当「规范合规检查表」参考 |

其他生态（已核验元数据，详表见被引调研；此处只列结论）：**Java 无 Maven Central 构件**（`stdf/stdf4j/jstdf` 检索 0 命中，仅 GitHub 上两个 2022 年未维护仓库）；**Go 无 STDF 解析器**；Node 生态有 `@wafertools/testdata-parser`（MIT，2026-10-13，WASM，核心记录可读、MPR 明确不读）与 `stdf-parser`（npm 2.0.2，**无 license 字段，法务风险**）、`stdfjs`（2018 年停更）等；C++ 最近有 `nieffed/stdfcpp`（MIT，2025-10）。MATLAB File Exchange 与 tsmap/stdf.io 等查看器不构成校验器。所有候选均**不发布「必需记录」判据**，也无任何一方实现 `HDR/TCR/DIE`（记录清单里首记录一律是 FAR、part 结果是 PRR）。

### 5.3 选型结论

1. **主用 `semi-ate-stdf==0.1.28`**（PyPI 钉版本；MIT、纯 Python、3.9.6 原生可用、对坏数据响亮失败、无 CLI 依赖）。已知唯一瑕疵：PyPI 落后于 GitHub release（0.1.33），如需 0.1.29+ 需从 GitHub 安装——**CI 用 PyPI 0.1.28 即可**。
2. **辅用 `pystdf==1.4.0`**（血统独立——不同作者、不同年代、不同代码结构，作第二意见；GPL-2.0 ⇒ **只在 CI/内部测试运行，不链接进产品、不随产品分发**；使用前建议与法务确认）。
3. **C# 测试工程内再加一个第三方实现（推荐）**：`STDFParser4Net`（NuGet 0.1.1，MIT，`dotnet add package` 即用，流式、感知字节序）——这是「SD-FMT-041 外部第三方解析器」在 .NET 侧成本最低的落点；`Marklio.Stdf` 备选（未发 NuGet，需从源码引）。
4. 可选第三意见：`stdf-convert==0.1.0`（py3.9 唯一可装版本；JSONL 输出适合记录数/字节偏移差异对比；0.2.0+ 需 ≥3.11）。
5. 排除：`pylibstdf`（不存在）、`stdfast`（3.9.6 装不上）、`pystdf4`（读侧未完成）、npm `stdf-parser`（无 license，法务风险）、Java/Go 生态（无可用构件）。

---

## 6. 交叉校验方案（选定做法）

**原则**（对齐 SD-FMT-040/041/042/043）：第三方解析器只进 CI 与测试，绝不成为运行时依赖；校验脚本独立于写入实现（不复用产品代码路径）。Python 双解析器为主；C# 测试工程可另加 `STDFParser4Net` 作同语言第三方检查。

**运行方式**（现状可直接执行；未来固化为 CI step）：

```bash
python3 -m venv .venv-check
.venv-check/bin/pip install "semi-ate-stdf==0.1.28" "pystdf==1.4.0"   # 均可 --no-deps 安装（核心解析路径只用标准库）
.venv-check/bin/python tools/check_stdf.py <golden.stdf> [more.stdf ...]
```

**断言强度（对每个 golden 文件、每个解析器分别执行，再做双解析器对拍）**：

1. **解析必须成功**（异常即 CI 红）——但仅此不够（§1.2 的静默失败案例）。
2. **记录直方图断言**：FAR==1 且首条、MIR==1、PCR≥1、MRR==1 且末条；PIR==PRR；PTR==Σ(每 die 测试项数)；WIR==WRR==wafer 数；若启用 TSR/HBR/SBR 则数量 == 测试项数/出现过的 bin 数。
3. **结构不变式**：PTR/MPR/FTR 全部位于某 PIR/PRR 对之间；WIR/WRR 成对；RDR（若用）紧随 MIR。
4. **关键字段探针**：`FAR.CPU_TYPE==2`、`STDF_VER==4`；`WIR/WRR.WAFER_ID` == 文件名渲染出的 wafer_id（SD-FMT-017 同款）；`PRR` 的 XY/HARD_BIN == 期望 die；含一个「已知失败」die：`PTR.TEST_FLG` bit7=1（且 PARM_FLG bit3/4 符合设定）、PRR.PART_FLG bit3=1；`PCR.PART_CNT/GOOD_CNT` == 实际 PRR 统计；`TSR.EXEC_CNT/FAIL_CNT` == 实际计数。
5. **双解析器一致性**：两者解出的「记录直方图 + 探针字段值」必须完全相等（本笔记实测的全部合法变体均达到此标准；真实世界参照：pystdf 官方 4.56 MB demo 文件上两库计数一致，59,890 条——见 §5.2）。
6. **负例必须红**：截断到半条记录、REC_LEN 篡改、首记录替换为伪记录各造一个样本，校验脚本必须断言失败。注意：不要依赖 pystdf 对「伪记录」报错——它静默返回 0 条，必须靠第 2 条的计数断言抓住。
7. **压缩变体**：对同一逻辑文件的 `.stdf` 与 `.stdf.gz` 各跑一遍，断言语义等价（记录直方图+探针一致）。

**不做**：跨实现的**字节级**相等断言（不同 writer 舍入/字段省略策略不同）；字节级回归由自家 golden 文件 SHA-256（SD-FMT-043）负责。

**断言到什么程度（结论口径）**：记录级完备性（集合、数量、顺序不变式）+ 关键字段值 + 双解析器一致 + 负例必红。这能把「写坏文件」「漏记录」「字段语义错位」「静默丢数据」四类故障全部挡住。

---

## 7. 与本地契约文档的差异清单（建议修订，未改动任何文件）

> 对照 [S13] `doc/contracts/formats/stdf.MD`（以及 requirements/design 中的引用）。逐条给规范依据与建议改法。

1. **「规格来源：SEMI E10」错误**（置信度：高，[S10][S11]）：SEMI E10 是**设备 RAM（可靠性/可用性/维护性）标准**，与 STDF 无关；SEMI 官方商店检索也没有「E10-1101」。STDF V4 是 Teradyne 的规范（1985 起），2010 年由 Teradyne 授权 SEMI 托管，现行版本 V4-2007（SEMI 以许可方式分发）；SEMI 自家的 STDF 家族文档是 G91（STDF Memory Fail Datalog，已 Inactive）。**未找到 V4/V4-2007 本身的 SEMI 文档号**。建议公开表述改为：「STDF V4（Teradyne 规范；2010 年起由 SEMI 托管；现行版 V4-2007）」。
2. **SD-FMT-010 记录表**：`HDR`/`SMR`/`DIE`/`TCR`/`HIL` 均非标准记录（§1.1/§3）；`PCR` 标 SHOULD 实为规范必需（≥1 条）；`HBR` 标「必需」并描述为「测试机/探针台配置」——HBR 实为硬件 bin 计数、规范可选；`PIR` 描述「每 Wafer 一条」错误（PIR 是 per part/die）；表内缺 `FAR`、`PRR`、`WRR`（CP 的核心记录），缺 `SBR`。建议整表替换为 §结论摘要的记录表。
3. **SD-FMT-011 PTR 表**：`TEST_FLG` bit 映射错误（规范：bit0 告警、bit1 结果无效、bit3 超时、bit4 未执行、bit6=P/F 有效性、bit7=失败）；`OPT_FLAG`「空」非法（写尾部字段时必填；bit1 必须为 1，全有效时 0x02）；`RES_SCAL/LLM/HLM` 语义错误（LO_LIMIT 用 `LLM_SCAL`、HI_LIMIT 用 `HLM_SCAL`，不是「按 RES_SCAL 缩放」；精度由 `C_*FMT` 承担）。建议按 §2 重写。
4. **SD-FMT-012**：PIR **没有** PART_ID/X_COORD/Y_COORD；这三者是 `PRR` 字段（规范 PRR 定义即用于 wafer map）。strict 档可直接用标准 PRR 承载 die XY，**无需 GDR**——这直接回答 SD-Q-002 的一半（「strict 能否出 map」：能，且是规范内机制）。
5. **SD-FMT-015**：`HDR` 计数回填不存在（§3）。建议改为：「每条记录 `REC_LEN` 正确 + PCR/WRR/HBR/SBR/TSR 计数与实际数据一致；实现可单遍流式，不需要回填/两遍/临时文件」。
6. **SD-FMT-016 压缩**：不存在「E10 记录级压缩规则」；按原文实现会产出不可解析文件。建议改为：「默认不压缩；`compression: gzip` 时输出整文件 gzip（`.stdf.gz`），语义与未压缩一致（CI 双变体校验）」。
7. **SD-FMT-017**：`SMR.LOT_ID` → 规范里没有 SMR；lot 信息在 `MIR.LOT_ID`（MIR 字段确有 LOT_ID）。`WIR.WAFER_ID` 正确。
8. **design.MD §6 配置示例** `records: [HDR, SMR, PIR, HBR, WIR, PCR, RDR, MRR, GDR, PTR, DIE]`（design.MD:1106–1109、1895–1900）与 `version: E10`：均含非标准名，缺 FAR/PRR/WRR；`version: E10` 应改为 `stdf_version: 4`（或 `V4-2007`，后者启用须写 VUR）。SD-EXT-103（design.MD:1634）的「`HDR` 回填」同步修订。
9. **SD-Q-013/015 的默认值**：SD-Q-013 默认「含 RDR 与 DIE」——`DIE` 不存在，建议默认改为「含 WCR/HBR/TSR，不含 RDR/SBR/SDR」；SD-Q-015「compression 默认开启」建议改为默认关闭（§4.2）。
10. **SD-PLG-018a**（requirements:1436）「SEMI E10」措辞同上修订。

---

## 8. 未找到权威来源的点（诚实的空白清单）

1. **STDF V4/V4-2007 的 SEMI 文档号**，以及「SEMI E10 是 STDF 规范来源」的任何依据——不存在（[S10][S11]）。
2. **任何记录级压缩方案 / `CPS` 记录 / 「字段前带长度字节」**——不存在（规范全文 0 命中）。
3. **`HDR`/`TCR`/`DIE`/`SMR`/`HIL` 记录**及 `D_CNT/H_CNT/P_CNT/R_CNT/T_CNT` 字段——不存在（三份规范 + 多实现记录表）。
4. **任一主流 YMS 公开的导入必需记录清单**——不存在（Exensio/yieldWerx/KLA/Synopsys/Galaxy/Onto/Siemens 全部只到格式级；用户手册登录墙）。
5. **目标 YMS 对 `.stdf.gz`、对 MPR、对 GDR 扩展的接受度**——无公开信息，属 SD-Q-001 依赖。
6. **「TCR」一词的来源**：(S9) Nornion 手册出现过「TCR 和 PCR」标题但正文实为 TSR 内容——疑似笔误传播，不构成依据。
7. `pylibstdf`——不存在（§5.1）。
8. （旁注）「Semi-ATE STDF web validator」——未检索到公开页面，不采信。
9. `semi-ate-stdf` 的 MIT vs 源码 docstring 的 GPL 矛盾——上游不一致，采信仓库 LICENSE/PyPI 元数据（MIT），但建议复核。

---

## 附录 A：实测明细（2026-10-16，macOS，Python 3.9.6）

环境：`semi-ate-stdf 0.1.28`、`pystdf 1.4.0`（venv `/tmp/sd-research/venv-parsers`）；样本由手写生成器产出（`/tmp/sd-research/files/`），无产品代码参与。

| 文件 | 内容 | semi-ate-stdf | pystdf |
| --- | --- | --- | --- |
| `v1_cp_full.stdf` | FAR MIR WCR WIR (PIR PTR×2 PRR)×3 WRR ×2 + TSR×2 HBR SBR PCR MRR | OK：FAR1 MIR1 WCR1 WIR2 PIR6 PTR12 PRR6 WRR2 TSR2 HBR1 SBR1 PCR1 MRR1 | 同上，一致 |
| `v2_cp_minimal.stdf` | 去掉 WCR/TSR/HBR/SBR 的最小 CP | OK：FAR1 MIR1 WIR1 PIR3 PTR6 PRR3 WRR1 PCR1 MRR1 | 一致 |
| `v3_ft_minimal.stdf` | final-test 风格（无 wafer 记录），含 1 个失败 die | OK（PIR2 PTR2 PRR2） | 一致 |
| `v4_cp_ptr_defaults.stdf` | 对 test 1002 全条省略 OPT_FLAG 后字段 | OK；省略字段解为「无效」（OPT_FLAG 全位、限值 0、UNITS 空） | OK；省略字段解为 `None` |
| `v5_cp_rdr.stdf` | v2 + RDR（紧随 MIR） | OK | 一致 |
| `v6_defaults_inherit.stdf` | test 1003：首条 PTR 写全默认值、第二条省略尾部（规范允许的最省写法） | 第二条**不回填默认**：OPT_FLAG=全无效、限值 0、UNITS 空 | 第二条**不回填默认**：全部 `None` |
| `v1_cp_full.stdf.gz` | v1 的 gzip | OK，与未压缩一致 | 一致 |
| `x1_hdr_first.stdf` | 首记录换成自造「HDR」 | **FAIL：IndexError**（响亮） | **OK()，0 条记录**（静默） |

字段解码对拍样点（两个解析器一致）：`PTR{1001, RESULT=0.5, LO=0.4, HI=0.6, UNITS=V}`、`PRR{HARD_BIN=1, X/Y=(0,0), PART_ID}`、`PCR/TSR/HBR/WRR` 计数与构造值一致。

## 附录 B：复现命令

```bash
# 1) 生成样本（纯标准库）
python3 /tmp/sd-research/make_minimal_stdf.py /tmp/sd-research/files
# 2) 双解析器对拍
/tmp/sd-research/venv-parsers/bin/python3 /tmp/sd-research/parse_check.py \
  /tmp/sd-research/files/v1_cp_full.stdf /tmp/sd-research/files/v2_cp_minimal.stdf \
  /tmp/sd-research/files/v1_cp_full.stdf.gz /tmp/sd-research/files/x1_hdr_first.stdf
# 3) 建 venv（若需重建）
python3 -m venv /tmp/sd-research/venv-parsers
/tmp/sd-research/venv-parsers/bin/pip install --no-deps --no-cache-dir semi-ate-stdf==0.1.28 pystdf==1.4.0
```

> 说明：脚本在 `/tmp` 下随本机回收；若要让 CI 长期复用，应由实现票把「生成样本 + 断言脚本」迁入仓库测试目录（本笔记不越权新增仓库文件）。
