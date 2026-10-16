# DuckDB.NET 并发语义与批量入库吞吐（Spike 调研笔记）

> 对应 GitHub issue #2（wayfinder research 票）/ `design.MD` §12 `SD-EXT-100`。
> 实验工程：`/tmp/sd-research/duckdb-net/SdSpike`（未入库，可复现，见附录 A 命令）。
> 日期：2026-10-16。全部结论除非标注「来自文档」均为本机实测。

## 结论摘要

1. **同进程多连接前提成立**：`Data Source=<path>` 的连接串在进程内按路径复用**同一个原生 Database 实例**（DuckDB.NET 内部 `ConnectionManager` 缓存，见 §7.1）。写连接提交后，已存在的读连接**立即可见**；写方未提交写对读方不可见（无脏读）；读方显式事务的**快照在事务内首次读语句时建立**（`BEGIN` 之后、读之前的提交仍可见），事务内后续查询不受外部提交影响。⇒ `IConnectionManager` 可以按「单写作用域 + 多读并发」设计，**不需要**退化为单连接读写串行队列（`SD-STO-062` 的退化路径不必启用）。
2. **同一张表上的并发追加不冲突**已实测证实（4/8 个连接同时 Appender 同一张带主键的表，0 错误，行数正确）；`SD-STO-061` 的追加语义成立。
3. **吞吐（Release，osx-arm64，10 核 / 16 GB）**：宽表（4 个 VARCHAR 键 + 40 × `FLOAT`，含主键）单连接 Appender **556k die/s**；8 连接并行 Appender **1.44M die/s**；`INSERT ... SELECT`（进程内 staging）1.40M die/s；`read_parquet('shards/*.parquet') → INSERT` **798k die/s**。全部远超 `SD-SCL-023` 的 100k die/s。内存峰值 ≤ 1.94 GB（进程 RSS，含 staging 表），远低于 4 GB 预算（`SD-SCL-025` / `SD-SCL-075`）。
4. **长表（5 列主键，2.5M:1 的 VARCHAR 键）是唯一不达标项**：`SD-SCL-024` 要求 ≥ 1.5M 行/s。实测单连接 Appender 0.81M 行/s、`INSERT ... SELECT` 1.26M 行/s、Parquet 通道 1.21M 行/s，均不达标；**只有并行 Appender（4 连接 1.64M、8 连接 1.86M 行/s）达标**。⇒ 若保留 1.5M 行/s 目标，「单写连接串行」不够，写入通道必须支持**多个 Appender 连接并行灌同一张表**（每条连接各自 `BEGIN`/`COMMIT` 不冲突）。若不改写入模型，则应把该目标修订为 ~1.2M 行/s 或下调长表默认开启策略。
5. **Parquet 暂存通道端到端 ≈ 直写通道**：2.5M 行宽表，通道 B 全程（8 路并行 staging 1.08 s + `COPY TO parquet` 0.28 s + `INSERT FROM read_parquet` 3.13 s ≈ **4.49 s wall**）与直写单连接（4.49 s）打平；直写 8 路并行反而最快（1.74 s）。Parquet 写本身极快（382 MB 8 文件 **1.35 GB/s**）。⇒ **切片走直写通道**（实现小、结果立即可查、吞吐达标）；Parquet 通道的价值在「生成与单写通道解耦 / 中断可续」，不在吞吐，作为 ≥10 Lot 或需要 `--resume` 时的升级路径保留接口（`SD-STO-070` 两通道同语义的要求两通道都实测产出相同行数）。
6. **跨工作簿并发成立**：同进程依次/并行创建并释放 4 个独立 `.duckdb` 稳定（两轮并行创建 0.39–0.44 s，0 错误；4 个库可同时保持打开）；**4 个进程各写不同 `.duckdb` 互不阻塞**（4 进程并行 wall 1.73 s vs 单进程 solo 1.52 s）。
7. **同库跨进程互斥由 DuckDB 原生文件锁保证，错误文本可直接用于 `ERR_WORKBOOK_LOCKED (14)` 的判定**：第二个进程（rw 或 ro）打开被占用的库，都在 `DuckDBConnection.Open()` 抛出
   `DuckDB.NET.Data.DuckDBException: DuckDBOpen failed: IO Error: Could not set lock on file "<path>": Conflicting lock is held in /usr/local/share/dotnet/dotnet (PID nnnn) by user xxx. See also https://duckdb.org/docs/stable/connect/concurrency`
   ——注意**同机只读打开也会失败**（与 `SD-SYS-257` 的判断一致）。**两个只读进程可以同时打开同一个库**（实测）。持有者退出后（最后一条连接关闭）文件锁**立即释放**，另一进程可马上接力打开。
8. **容量模型有实质偏差**：主键 ART 索引占用 **≈ 40 B/行**（宽表 2.5M 行实测：有 PK 404 MB vs 无 PK 305 MB，差 99 MB；长表 5M 行：有 PK 246 MB vs 无 PK 35.5 MB，差 211 MB）。`design.MD` §5 容量估算把「索引 + 元数据 + 视图」整体按 ~60 MB（100 Lot）计，**严重低估**：若开启长表（100M 行），仅 5 列 PK 索引就 ~4 GB。另：噪声主导的合成数据实测压缩比只有 ~1.2×（宽表 169.5 B/行），远低于文档假设的 4–8×；无 PK 的长表则压缩极好（7.4 B/行）。
9. **读写混合**：写进行时读查询可用（不阻塞、不报错），但同一实例内**平均延迟从 2.3 ms 升到 5.0 ms（2.2×）**，超过 `SD-SCL-034` 的「≤ +50%」；绝对延迟仍然很低（p95 ≈ 9.8 ms）。
10. **调试器下 `AccessViolationException` 未复现**——本环境全部为无调试器运行，无法验证；上游 issue #178 记录该现象（仅调试器下、参数化语句、Rider/Ubuntu），维护者未能复现，issue 已关闭无定论。属「来自文档，未实测」。
11. **三个必须写进实现的 API 事实**（实测踩坑）：① Appender 的 row 对象**不能跨行复用**，每行必须 `appender.CreateRow()`；② `DuckDBConnection.Duplicate()` **只支持内存库**，磁盘库抛 `NotSupportedException`——读连接要用同一连接串 `new DuckDBConnection(...)`；③ 磁盘库**在已有 rw 连接时，随后用 `AccessMode=ReadOnly` 打开的连接并不只读**（`access_mode` 只在路径首次打开时生效）⇒ 只读语义必须由应用层强制。

---

## 1. 实验环境与工程

| 项 | 值 |
| --- | --- |
| 机器 | macOS Darwin 25.6.0，Apple Silicon（osx-arm64），10 CPU / 16 GB / SSD |
| .NET SDK | 10.0.402（运行时 `System.Environment.Version` = 10.0.13） |
| DuckDB.NET.Data.Full | **1.5.6.1**（2026-10-15，依赖 DuckDB.NET.Bindings.Full + Apache.Arrow 23.0.0） |
| DuckDB 引擎 | `v1.5.6`（`SELECT version()` 实测） |
| 工程 | `/tmp/sd-research/duckdb-net/SdSpike`，`dotnet new console --framework net10.0`，无解决方案、无 Avalonia |
| 配置 | `SET threads=5`（= `min(8, cpu/2)`，`SD-STO-022` 默认）、`memory_limit='4GB'`、`temp_directory` 显式设置、`checkpoint_threshold='256MB'`、`preserve_insertion_order=false`；WAL 默认开启 |
| 测量方式 | 计时 `Stopwatch`（含 JIT 预热，先跑 2–5 万行热身）；内存为后台线程每 20 ms 采样进程 RSS（`Process.WorkingSet64`，含原生内存）取峰值；每次跑完 `CHECKPOINT` 后量文件大小 |
| 数据模型 | `sd_cp_result_wide`（design.MD 附录 C 同款：4 个 VARCHAR 键 + p01..p40 `FLOAT` + 4 列主键）；`sd_die`（附录 C 原样，含 PK + `(x,y)` UNIQUE）；`sd_cp_result_long`（4 键 + `item_id` VARCHAR + value FLOAT + flag VARCHAR，5 列主键）。键值形如 `wb_01JSPIKE-r1` / `LOT001` / `W01` / `D0001` / `CP_ITEM_01`，浮点值 = 空间梯度 + 噪声（模拟真实参数分布，噪声使压缩比偏保守） |
| 规模 | 宽表 2.5M 行 = 100 Lot × 25 Wafer × 1000 Die（与 `SD-STO-090` 的 100 Lot 基准一致）；长表 5M 行（125k Die × 40 项）、并行场景 10M 行 |
| 构建配置 | 吞吐数字以 **Release** 为准（Debug 也跑过一轮对比，见各表备注）；本项目 `dotnet run` 默认 Debug，务必用 `-c Release` 复现 |

汇总命令：`cd /tmp/sd-research/duckdb-net/SdSpike && dotnet run -c Release -- <mode> <dir>`（mode 列表见附录 A）。

## 2. 同进程多连接：可见性与快照（`vis` 模式）

场景：进程内先开写连接 **W**（autocommit），随后用**同一路径的新连接串**开读连接 **R**（`new DuckDBConnection("Data Source=...")`）。

| # | 步骤 | 实测结果 | 判读 |
| --- | --- | --- | --- |
| 1 | W 建表 + 插 1 行（提交） | — | — |
| 2 | 再开 R（同路径新连接） | Open 成功 | 进程内按路径共享实例；**不会**报「lock」 |
| 3 | R 读 | `count=1` | 已提交数据可见 |
| 4 | W autocommit 再插 10 行，**R 不重开**直接读 | `count=11`（立即） | 写提交后对既有读连接立即可见（同实例 MVCC） |
| 5–6 | R `BEGIN`；W 提交 +100 行；R 事务内读 | `11`（事务内第二次读仍是 `11`） | 读事务内快照稳定（快照隔离），外部提交不影响 |
| 7 | R `COMMIT` 后读 | `111` | 事务结束后看到最新已提交 |
| 8–9 | R `BEGIN`（尚未读）；W 插 1 行（112→113）；R **首次**读 | `113` | **快照在事务内首次读语句时建立**，`BEGIN` 后的提交仍可见 |
| 10 | W `BEGIN` + 插 500 行未提交；R 读 | `113`（不变） | 无脏读 |
| 11 | 第二个写连接 W2 在 W 写事务未提交时 INSERT | **成功**（提交为独立事务） | 追加不与未提交事务冲突；`SD-STO-061` 成立 |
| 12 | W 提交后 | `count=614` | 两侧写入都落库（613+1），行数正确 |
| 13 | `W.Duplicate()` | `NotSupportedException: Duplication of the connection is only supported for in-memory connections.` | **磁盘库不能用 `Duplicate()`** |
| 14 | `W.Close()` 后 R 继续读 | `614` | 实例由仍开着的连接保持 |
| 15 | 重开写连接 W3 插入 1 行 | R（旧连接）读到 `615` | 关闭再打开不换实例，可见性规则不变 |

**结论**：`SD-STO-062` 要验证的语义全部满足——写提交 ⇒ 既有读连接立即可见；显式事务快照边界清晰（首次读时建立）；无脏读；追加不冲突。`IConnectionManager` 的「写作用域全局唯一串行 + 读连接可并发」前提**成立**，无需启用退化路径。

## 3. 批量入库吞吐（Release，`bench` 模式）

目标：`SD-SCL-023` 宽表 ≥ 100k die/s；`SD-SCL-024` 长表 ≥ 1.5M 行/s；`SD-SCL-025/075` 内存 ≤ 4 GB。

### 3.1 宽表 2.5M 行（4 VARCHAR 键 + 40 × FLOAT + PK）

| 路径 | wall | 速率 | 峰值 RSS | 文件 |
| --- | --- | --- | --- | --- |
| ① 单连接 `Appender` → 最终表（含 PK） | 4.49 s | **556k die/s** | 888 MB | 420 MB |
| ② `INSERT INTO ... SELECT` ← 进程内 staging 表（无 PK） | 1.79 s | **1.40M die/s** | 1.62 GB（base 650 MB） | 721 MB |
| ③ `INSERT INTO ... SELECT` ← `read_parquet('8 个 shard/*.parquet')` | 3.13 s | **798k die/s** | 1.94 GB（base 1.45 GB） | 790 MB |
| ④ 4 连接并行 `Appender` → 同一张表 | 2.05 s | **1.22M die/s** | — | — |
| ⑤ 8 连接并行 `Appender` → 同一张表 | 1.74 s | **1.44M die/s** | — | — |

- Debug 构建同口径：①483k、②1.38M、③822k、④1.07M die/s——与 Release 相差 ~10–15%，结论不受影响。
- 热身/复跑：①重复跑 4.49 s / 5.15 s（Release/Debug 各两轮偏差 <3%）。
- ④⑤无任何异常或冲突（`SD-STO-061`「追加永不冲突」在 C# 侧同样成立）。
- `sd_die` 真实 DDL（23 列、PK + `(x,y)` UNIQUE）：单连接 Appender 2.5M 行 **5.36 s = 466k die/s**，峰值 805 MB。

### 3.2 长表（5 列 PK，VARCHAR 键 + FLOAT + VARCHAR flag）

| 路径 | 行数 | wall | 速率 | 峰值 RSS | 备注 |
| --- | --- | --- | --- | --- | --- |
| ① 单连接 `Appender` → 最终表 | 5M | 6.18 s | **0.81M 行/s** | 716 MB | ❌ 未达 1.5M |
| ② `INSERT..SELECT` ← 进程内 staging | 5M | 3.96 s | **1.26M 行/s** | 1.26 GB | ❌ 未达 1.5M |
| ③ `INSERT..SELECT` ← parquet glob | 5M | 4.12 s | **1.21M 行/s** | 1.50 GB | ❌ 未达 1.5M |
| ④ `INSERT..SELECT` → **无 PK** 目标表（对照） | 5M | 1.04 s | **4.81M 行/s** | 541 MB | Debug；证明瓶颈是 PK 索引维护 |
| ⑤ 4 连接并行 `Appender`（键区间互斥） | 5M | 3.05 s | **1.64M 行/s** | — | ✅ 达标，0 错误 |
| ⑥ 8 连接并行 `Appender` | 5M | 2.68 s | **1.86M 行/s** | — | ✅ 达标，0 错误 |
| ⑦ 10M 行 `INSERT..SELECT`（threads=5） | 10M | 8.03 s | 1.25M 行/s | — | 规模翻倍速率不变 |
| ⑧ 10M 行 `INSERT..SELECT`（threads=10） | 10M | 7.26 s | 1.38M 行/s | — | 提高 threads 有一点帮助但不达标 |

**判读**：长表达不到 1.5M 行/s 的原因不是 I/O、不是并发语义，而是 **5 列 PRIMARY KEY 的 ART 索引在单写线程里逐行维护**（④ 对照：去掉 PK 后 4.81M 行/s，相差近 4×）。要达到目标必须**多连接并行灌同一张表**（⑤⑥），或把目标修订为 ~1.2–1.3M 行/s，或评估「先无 PK 批量入库、最后 `CREATE INDEX`/`ALTER TABLE ADD PRIMARY KEY` 重建索引」的变体（**未测**，需另行验证 DuckDB 对已有数据加 PK 的成本）。

### 3.3 staging → Parquet → 最终表（通道 B 全链路，宽表 2.5M 行）

| 阶段 | wall | 速率 | 备注 |
| --- | --- | --- | --- |
| 分片并行 staging（8 连接 Appender → 8 张无 PK staging 表） | 1.08 s | **2.32M 行/s** | 并行追加再次证明无冲突 |
| `COPY st_wide_s TO 'shard_s.parquet'`（8 个并行） | 0.28 s | **1.35 GB/s**（382 MB） | SNAPPY |
| `INSERT INTO sd_cp_result_wide SELECT * FROM read_parquet('shards/*.parquet', union_by_name=true)` | 3.13 s | 798k die/s | 行数校验 2.5M == 2.5M ✅ |
| **通道 B 全程（wall 合计）** | **4.49 s** | — | 与通道 A 单连接 4.49 s 打平；A 的 8 路并行 1.74 s 更快 |

长表（5M 行）对应数字：并行 staging + COPY 共 1.50 s（3.34M 行/s，parquet 仅 19.8 MB），`INSERT FROM read_parquet` 4.12 s（1.21M 行/s）；两通道行数一致。

> 说明：本实验的「分片 worker」是「8 个并行 Appender 写 staging 表 + `COPY TO`」，而不是 worker 直接用 Arrow/自管理写出 Parquet —— `COPY TO parquet` 部分与真实做法等价（同样的 DuckDB 写 parquet 路径），staging 端速度（2.3M 行/s）是并行 Appender 的能力上限。

## 4. 跨工作簿并发（`multidb` / `xproc` 模式）

### 4.1 同进程依次 / 并行创建并释放多个独立 `.duckdb`

| 场景 | 结果 |
| --- | --- |
| 4 个库并行创建（各建 schema + 写 100k 行） | 两轮均成功：0.39 s / 0.44 s，**0 错误** |
| 依次 reopen → 追加 → 关闭（每库各 1 次） | 全部成功（0.19–0.25 s/库），行数正确 |
| 4 个库同进程同时保持打开 | OK（打开耗时 0.01 s） |

结论：`batch` 场景下「同进程依次/并行创建多个独立工作簿」稳定，不需要退化为串行创建。

### 4.2 跨进程：不同 `.duckdb`

| 场景 | 结果 |
| --- | --- |
| 单进程写 500k 行（solo 基线） | 1.52 s wall（子进程内 1.20 s = 418k 行/s） |
| **2 进程并行**、各自独立的库 | wall **1.39 s**（两子进程各 1.19 s） |
| **4 进程并行**、各自独立的库 | wall **1.73 s**（四个子进程各 1.32 s） |

无阻塞、无相互干扰，且各子进程吞吐几乎不变（418k → 376–380k 行/s，系统级共享资源导致的轻微下降）。`SD-SCL-031` 的「≥4 个独立工作簿同时生成」在本机成立（跨进程路径）。

### 4.3 跨进程：同一个 `.duckdb`

进程 A 以 rw 打开并持有 8 s，期间：

| 尝试 | 结果 |
| --- | --- |
| 进程 B 以 **rw** 打开 | `Open()` 抛 `DuckDBException`：`DuckDBOpen failed: IO Error: Could not set lock on file "<path>": Conflicting lock is held in /usr/local/share/dotnet/dotnet (PID nnnn) by user shaokp. See also https://duckdb.org/docs/stable/connect/concurrency` |
| 进程 B 以 **read-only** 打开 | **同样失败**，错误文本相同（对应 `SD-SYS-257`「本机已有读写打开时同机只读会失败」） |
| 进程 A 退出后，进程 B 以 rw 打开 | 成功，读到 1000 行 |
| 进程 A 退出后,进程 B 完整「打开→追加 200k 行→关闭」 | 成功（0.59 s，行数 201000 正确） |
| **两个进程都以 read-only 打开**同一个库 | **同时成功**（第二个 ro 在第一个 ro 持有期间可打开并查询） ✅ |

实现层面：DuckDB 原生文件锁在 `DuckDBConnection.Open()` 时就生效（不是延迟到查询），异常类型 `DuckDBException`、消息含持有者 PID/用户，**可直接解析用于 `ERR_WORKBOOK_LOCKED (14)`**；不必只依赖 `workbook.lock` 锁文件。只读并发共享可行、只读与读写不可共存 ⇒ `SD-STO-080`「只读 Parquet 归档多实例共享」是唯一无锁共享路径。

### 4.4 锁的生命周期（`lockrel` 模式）

进程内 `using` 释放（`Close()`）后：① 另一个**新进程**可立即以 rw 打开同一文件（成功）；② 本进程内重新打开也成功；③ 再次释放后子进程仍可打开。⇒ **文件锁随最后一个连接关闭立即释放**（DuckDB.NET 引用计数到 0 时 `duckdb_close`），工作簿「关闭即释放、可接力」成立，不存在「锁滞留到进程退出」的问题。

## 5. 读写混合与内存峰值（`readmix` 模式）

固定 2.5M 行表 A 上跑 `SELECT count(*), avg(p01), sum(p40)`（全表扫描类查询），另一连接同时向**另一张表** Appender 灌 2.5M 行：

| 状态 | 平均 | p95 | max | 样本 |
| --- | --- | --- | --- | --- |
| 空闲 | 2.3 ms | 3.8 ms | — | n=10 |
| 写进行中 | 5.0 ms | 9.8 ms | 41.6 ms | n=45 |
| 比值 | **2.16×** | | | |

读查询全程可用、无阻塞、无错误（「可用」成立），但**「延迟增加 ≤ 50%」不成立**（绝对延迟仍 ≤ 10 ms/95 分位）。注意这是「扫描型查询 + 单线程饱和写入」的组合；点查/小范围查询（`wafer_id` 谓词）受影响会小得多——**未测**。

内存峰值（Release，本节即上文各表峰值）都在 4 GB 预算内：单连接宽表 Appender 888 MB、`INSERT..SELECT` 1.62 GB、Parquet 通道 1.94 GB、长表 1.2–1.5 GB。Parquet/INSERT 路径的峰值包含 staging 表本身。

## 6. 容量与索引开销（`sizeprobe` 模式，Release）

同一份 Parquet（2.5M 行宽表 / 5M 行长表）灌进全新库，`CHECKPOINT` 后量文件：

| 表 | 有 PK | 无 PK | PK 索引差 | 有 PK B/行 | 无 PK B/行 |
| --- | --- | --- | --- | --- | --- |
| 宽表 2.5M 行（4 VARCHAR 键 + 40 FLOAT） | 404.0 MB | 305.3 MB | **≈ 99 MB（40 B/行）** | 169.5 | 128.0 |
| 长表 5M 行（5 列 PK） | 246.3 MB | 35.5 MB | **≈ 211 MB（42 B/行）** | 51.6 | 7.4 |

- **PK 索引 ≈ 40 B/行**（两张表一致）：`design.MD` §5 把「索引 + 元数据 + 视图」按 ~60 MB/100 Lot 计——宽表主键一项实测就已 ~99 MB，长表若按 100M 行推算仅索引 ~4 GB。**容量模型必须单列索引项**（对照 `SD-STO-094` 的验证门槛）。
- 压缩比：本实验浮点是「梯度 + 噪声」的合成值，宽表实测仅 ~1.2×（噪声不可压）；真实生成数据若按 `precision` 舍入（十进制可表达），压缩比会显著更好——**4–8× 的文档假设未获证实也未被证伪，必须用真实生成数据复测**（这是「测不准」项，见 §8）。
- 无 PK 长表 7.4 B/行（VARCHAR 字典/FSST 压缩极好），远优于文档 21 B/行的原始估算；长表的体积问题不在数据而在索引。
- 附带观察：同样 5M 行长表，Parquet 落盘 19.8 MB（4 B/行），比 DuckDB 原生文件（无 PK 35.5 MB）还小。

## 7. API 陷阱与实现注意（实测 + 上游源码）

1. **Appender row 对象不能复用**：`var row = app.CreateRow(); ... row.EndRow();` 之后继续用同一个 `row` 追加第二行会抛 `IndexOutOfRangeException: The table X has N columns but you are trying to append value for column N+1`。**每行都必须重新 `CreateRow()`**（`AppendRow(callback)` 重载同样可用）。这会带来每行一次对象分配，是单连接 Appender 吞吐的一部分成本。
2. **`DuckDBConnection.Duplicate()` 只支持内存库**，磁盘库抛 `NotSupportedException`。读连接并发应使用**相同连接串的新 `DuckDBConnection`**（即实验中的做法，天然共享实例）。
3. **连接串配置「首次打开生效」**：DuckDB.NET 内部 `ConnectionManager` 以 `connectionString.DataSource` 为键缓存 `FileReference`，只在**首次**为某路径创建原生数据库时应用连接串中的配置（`duckdb_create_config`/`duckdb_open`），后续同路径连接复用已打开的实例、配置被忽略（来源：DuckDB.NET 源码 `DuckDB.NET.Data/Connection/ConnectionManager.cs`，见 §附录 B；实测表现为 §7.4 的只读被忽略）。⇒ 配置（`memory_limit` / `temp_directory` / `threads`…）应通过**首个打开连接**带上，或用 `SET` 语句在实例上设置（本实验用 `SET`，两次连接都生效）。
4. **连接串里 `AccessMode=ReadOnly` 在「同路径已有 rw 连接」时被静默忽略**：实测第二个「只读」连接仍可 INSERT 成功（`count` 变化）。单独一个只读连接打开整个库时，DuckDB 会明确拒绝写：`DuckDBException: Invalid Input Error: Cannot execute statement of type "INSERT" on database "romode" which is attached in read-only mode!`。⇒ `--read-only`（`SD-SYS-257`）必须**应用层强制禁止写入**，不能只依赖打开模式。
5. **文件锁在 `Open()` 抛错**、错误文本含持有进程 PID/用户（见 §4.3），行为同 DuckDB 官方文档的「File locks」条目。
6. **锁随最后一条连接关闭释放**（引用计数归零时关闭原生库），工作簿关闭 → 其他进程可立即接力（`SD-SYS-260` 的释放顺序不受影响）。

## 8. 没测出来 / 测不准（不要据此猜测）

| 项 | 状态 |
| --- | --- |
| 调试器下 `AccessViolationException` | **未复现**——实验全部无调试器运行。上游 issue #178 记录「仅调试器下、参数化语句」现象，维护者未能复现后关闭（来自文档/上游，未实测） |
| Windows / Linux 行为 | 未测（仅 osx-arm64）。文件锁差异、`File.Move`、只读打开行为需按 `SD-EXT-101` 在三大平台分别验证 |
| ≥16 并行分片任务 | 最高测到 8 个并行 Appender（成功）；16 路未测，按「追加不冲突」推断可行但未验证 |
| 长表「无 PK 入库 + 事后加 PK」变体 | 未测——若要用它绕开 1.5M 行/s 目标，需先验证 `ALTER TABLE ... ADD PRIMARY KEY` 在 5M–100M 行上的成本 |
| 真实 CP/WAT 数据的压缩比与容量 | **测不准**——本实验浮点噪声含量高，压缩比是悲观下界；4–8× 假设需用真实生成输出复测（`SD-STO-094`） |
| 100 Lot 全链路（几何/物理/缺陷计算 + 入库） | 未测——本票只覆盖表级写入，生成管线本身的开销不在范围内 |
| 长表 100M 行（500 Lot 极限） | 未测——只测到 10M 行 |
| 多写并发下的读延迟 / 点查延迟 | 未测——readmix 只测了单写 + 扫描型查询 |
| Parquet shard worker「直接写出 parquet」（不经营 staging 表） | 未测——用 `COPY TO parquet` 等价的代理路径代替；若 worker 用自管 ParquetWriter，需另测其吞吐 |

## 9. 对切片的影响

**① 切片该走直写还是 Parquet 暂存通道？ → 走直写（通道 A）。**
- 宽表直写 556k die/s 是 100k die/s 目标的 5.6 倍，且 8 路并行可到 1.44M die/s；实现只需要 `IConnectionManager` 的写作用域 + Appender，结果立即可查，无 staging 目录清理与行数校验的额外复杂度。
- Parquet 通道端到端（4.49 s）与直写（4.49 s）打平，没有吞吐上的理由让它成为默认路径；`design.MD` §5 说它「推荐用于 ≥10 Lot」的依据是「规避长事务」——实测并未观察到长事务问题（2.5M 行单事务写入无异常）。因此**切换点应由「中断可续 / 生成与入库解耦」需求驱动**，而不是规模阈值。
- 但通道 B 的接口（`SD-STO-070`）仍应在切片里留出：`shard_*.parquet` 写出 + `INSERT FROM read_parquet` 已验证可行，行数一致可通过 `count(*)` 校验。

**② `IConnectionManager` 按「单写串行 + 多读并发」还是退化为单连接串行队列？ → 保留「单写作用域 + 多读并发」。**
- `SD-STO-062` 的两个前提（提交立即可见、事务快照边界）全部满足，退化路径不需要启用；多读连接只要用相同连接串新建 `DuckDBConnection` 即可（**不要**用 `Duplicate()`）。
- 但有一个**必须修改的点**：`SD-STO-063` 的「写作用域：会话内唯一，串行使用」如果实现成「整个会话只有一条写连接、所有写入排队」，则**长表永远达不到 `SD-SCL-024` 的 1.5M 行/s**（实测单连接 0.81–1.26M）。要达标，写作用域必须允许多条 Appender 连接**并行**灌入（键区间互斥或同表追加，实测 4/8 路可达 1.64/1.86M 行/s，0 冲突）。建议把写作用域定义为「**逻辑上唯一（存在排他窗口），物理上可持有多个 Appender 连接**」，或者显式下调 `SD-SCL-024` 目标。
- 只读打开（`--read-only`）必须由应用层拒绝写操作（§7.4）；同时按 `SD-SYS-257` 的既有判断，同机已有 rw 打开时只读打开会失败，降级建议指向 Parquet 归档。
- `ERR_WORKBOOK_LOCKED (14)` 的判定可以直接利用原生锁错误的出现（`Open()` 抛 `DuckDBException` 且消息含 `Could not set lock on file`），与 `workbook.lock` 锁文件形成双保险（`SD-SYS-250` 的双层设计成立）。

**③ 其它应写回设计文档的结论**：容量模型补索引项（≈40 B/行/主键）；压缩比按真实数据复测；读写混合的「+50% 延迟」目标在扫描型查询 + 饱和写入下不成立（实测 2.2×，绝对仍低）。

## 10. 与文档假设的偏差一览

| 文档假设 | 出处 | 实测 | 结论 |
| --- | --- | --- | --- |
| 同进程内按路径复用同一 Database 实例，多连接安全 | `design.MD` §4.2 / `SD-STO-062` | 成立（DuckDB.NET `ConnectionManager` 缓存 + 15 步可见性矩阵） | ✅ 无需退化 |
| 追加永不冲突，进程内多线程并发追加安全 | `SD-STO-061` | 成立（同表 4/8 路并发 Appender，0 错误） | ✅ |
| 宽表写入 ≥ 100k die/s | `SD-SCL-023` | 556k（单连接）/1.44M（8 路） | ✅ 超 5–14× |
| 长表写入 ≥ 1.5M 行/s | `SD-SCL-024` | 0.81M（单连接 Appender）/1.26M（INSERT..SELECT）/1.21M（parquet）/1.64–1.86M（并行） | ❌ 单写串行达不到；需并行写或改目标 |
| 生成阶段内存峰值 ≤ 4 GB | `SD-SCL-025/075` | ≤ 1.94 GB（最重路径含 staging） | ✅ |
| 读写混合延迟增加 ≤ 50% | `SD-SCL-034` | 2.3 ms → 5.0 ms（2.16×），读始终可用 | ❌ 相对目标未达（绝对值低） |
| 进程独占由「DuckDB 原生文件锁」保护 | `SD-SYS-250` | `Open()` 即抛错，文本含 PID；只读并发可共享、rw+ro 不可共存 | ✅ 且可解析错误文本 |
| 只读打开在同机被 rw 占用时失败 | `SD-SYS-257` | 成立（ro 同样报 lock） | ✅ |
| 容量：宽表压缩后 4–8×（100 Lot ≈ 70–130 MB） | `design.MD` §5 | 合成噪声数据 ~1.2×（169.5 B/行，404 MB/2.5M 行） | ⚠️ 需用真实数据复测 |
| 容量：索引+元数据+视图 ≈ 60 MB | `design.MD` §5 | 单个宽表 PK ≈ 40 B/行（99 MB/2.5M 行）；长表 PK ≈ 42 B/行 | ❌ 严重低估，长表开启时更甚 |
| `Duplicate()` 可用于多连接 | （隐含实现） | 磁盘库不支持 | ❌ 改用同连接串新连接 |
| 调试器下 `AccessViolationException` | `SD-STO-023` | 未复现（无调试器）；上游 #178 | ⚠️ 未能证实/证伪 |

## 附录 A. 复现命令

```bash
mkdir -p /tmp/sd-research/duckdb-net && cd /tmp/sd-research/duckdb-net
dotnet new console -o SdSpike --framework net10.0
cd SdSpike && dotnet add package DuckDB.NET.Data.Full --version 1.5.6.1

# 语义/并发（Release 或 Debug 均可）
dotnet run -c Release -- info                     # 版本信息
dotnet run -c Release -- vis        v             # 可见性/快照矩阵
dotnet run -c Release -- multidb    m             # 同进程多库
dotnet run -c Release -- xproc      x             # 跨进程（不同库 / 同库 / ro+ro）
dotnet run -c Release -- lockrel    l             # 关闭后锁释放
dotnet run -c Release -- readmix    r             # 读写混合
dotnet run -c Release -- sizeprobe  sp            # 索引开销

# 吞吐（务必 -c Release）
for p in appender wide-insert parquet die long long-insert long-parquet parallel-append; do
  dotnet run -c Release -- bench b $p
done
SD_WORKERS=4 dotnet run -c Release -- bench b5 long-parallel-append
SD_WORKERS=8 dotnet run -c Release -- bench b5 long-parallel-append
dotnet run -c Release -- bench b5 wide-parallel-append-8
```

数据/日志目录：`/tmp/sd-research/duckdb-net/data/`、`/tmp/sd-research/duckdb-net/logs/`（本笔记全部原始输出在其中）。

## 附录 B. 出处

- DuckDB 官方并发文档（错误信息中引用的同一页）：<https://duckdb.org/docs/stable/connect/concurrency> / 现址 <https://duckdb.org/docs/current/connect/concurrency.html>——「Read-write mode: one process can both read and write… Read-only mode: multiple processes can read…」「Appends will never conflict, even on the same table.」「DuckDB handles concurrent database access requests using file locks.」
- DuckDB.NET 连接实例缓存实现（只读源码，用于解释实测语义）：<https://github.com/Giorgi/DuckDB.NET/blob/main/DuckDB.NET.Data/Connection/ConnectionManager.cs>
- DuckDB.NET issue #178（调试器下 AccessViolationException，已关闭未复现）：<https://github.com/Giorgi/DuckDB.NET/issues/178>
- 本地设计文档：`doc/design.MD` §4.2 / §5 / §8 / §12（`SD-EXT-100`）；`doc/requirements-spec.MD` §4.7 / §10
