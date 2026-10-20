# ADR-0002：工作簿锁采用应用层锁文件 + DuckDB 原生锁兜底

- **状态**：已接受（2026-10-20）
- **相关需求**：`SD-SYS-250`、`SD-SYS-255`、`SD-SYS-256`、`SD-SYS-260`、`SD-SYS-262`、`SD-EXT-101`
- **相关设计**：`doc/design.MD` §1.5、§3.3、§8、§12
- **来源**：issue #6

## 背景

工作簿必须同时满足两条要求：

1. **同一工作簿同一时刻只允许一个读写进程**（`SD-SYS-250`），且被占用时第二个进程必须**立即失败**（不得等待）。
2. 被占用时必须能**读取持有者信息**（PID、主机、进程启动时间）并给出可操作提示（`SD-SYS-256`、`SD-SYS-262`）。

最初设计采用 `FileStream` + `FileShare.None` 作为进程间独占锁。.NET 的实现事实推翻了这一方案：

- .NET 在 Unix 上把 `FileShare.None` 映射为 `flock(LOCK_EX)`，**其余所有 `FileShare` 取值（含 `Read`）都映射为 `LOCK_SH`**。因此 `FileAccess.ReadWrite + FileShare.Read` 在 macOS / Linux 上两个进程都能成功打开，**没有写互斥**；同一代码在 Windows 上却因共享冲突而互斥。这正是 `SD-EXT-101` 警告的平台差异。
- 若改用 `FileShare.None` 获得真正的独占，竞争进程**无法打开锁文件**，也就无法读取持有者信息。
- `FileStream.Lock`（字节范围锁）在 macOS 上被标注 `[UnsupportedOSPlatform("macos")]`，调用直接抛 `PlatformNotSupportedException`；POSIX `fcntl` 锁在同一进程内可重入，且关闭任一文件描述符即释放，不适合作为会话级锁。

切片只在 `osx-arm64` 上运行，第一条事实足以否决原方案。

## 决策

### 1. 应用层锁文件是唯一仲裁者

- `workbook.lock` 以 `FileMode.CreateNew`（`O_CREAT | O_EXCL`）原子创建。创建成功即取得锁；文件存在即被占用。
- 持有者信息以 JSON 写入锁文件，写入后关闭句柄；互斥语义由**文件存在性**承担，不依赖句柄或 OS 锁。
- 正常关闭时删除锁文件；删除失败只 WARN 并退避重试。
- Unix 上以 `0600` 创建；读写时使用 `FileShare.ReadWrite | FileShare.Delete`，保证持有者信息始终可读、可清理。

### 2. DuckDB 原生文件锁作为兜底与清理串行化原语

打开 `workbook.duckdb` 时由 DuckDB 获取原生文件锁。它承担两个职责：

- **失效锁清理的串行化**：判定锁文件失效后，必须先以读写打开数据库；打开成功证明不存在活着的写者，此时才允许删除旧锁并重新创建。两个进程同时清理时，DuckDB 只放行一个。
- **防御外部 DuckDB 客户端**：若有进程绕过 SiliconDraft 直接以读写打开数据库，由 DuckDB 打开失败暴露；异常消息含锁冲突特征时映射为 `ERR_WORKBOOK_LOCKED (14)`。

崩溃时内核自动释放 DuckDB 锁，残留的 `workbook.lock` 由下一次打开按上述流程清理。

### 3. 失效判定用 PID + 进程启动时间，跨主机不自动清理

- 同主机：PID 不存在或启动时间不匹配（PID 复用）→ 失效锁，允许清理并 WARN。
- 同主机：PID 存在但启动时间读取失败（权限等）→ **未知**，按存活处理，报 14。
- 跨主机：**禁止自动清理**，报 14 并提示手工处理。
- 心跳不参与锁仲裁；`heartbeatAt` 字段保留，写锁时等于 `startedAt`。

### 4. 进程内互斥

会话单例 + 进程级规范化路径注册表；同进程第二次打开同一工作簿返回 `ERR_LOCKED (13)`。

## 后果

- 行为在三平台一致，不依赖 .NET / OS 文件锁语义；代价是互斥依赖所有写者都遵守同一协议（外部 DuckDB 客户端由数据库自身的锁兜底）。
- 锁文件在"创建后、写入前"存在一个极短的空文件窗口；读取方遇到空 / 半写内容时重试至多 200 ms。
- 锁文件删除失败会留下"持有者 PID 仍存活"的假占用；下一次打开可凭 DuckDB 打开成功安全接管，但需要 WARN。
- 心跳、崩溃恢复页与 `recovery.json` 属产品层（`SD-SYS-232`），不阻塞切片；实现时不得改变本 ADR 的仲裁语义。
- `SD-EXT-101` 在发布前仍需验证 DuckDB 原生锁在网络盘上的行为、`File.Move` 三平台原子替换与回收站降级；本 ADR 已将"平台文件锁差异"从风险变为已知事实。

## 依据

- [dotnet/runtime#24432](https://github.com/dotnet/runtime/issues/24432)：Unix 上 `FileShare` 映射为 flock 共享锁。
- [.NET 6 兼容性说明：FileStream 在 Unix 上使用共享锁](https://learn.microsoft.com/dotnet/core/compatibility/core-libraries/6.0/filestream-file-locks-unix)。
- [dotnet/corefx `FileStream.OSX.cs`](https://github.com/dotnet/corefx/blob/master/src/Common/src/CoreLib/System/IO/FileStream.OSX.cs)：macOS 上 `Lock` / `Unlock` 抛 `PlatformNotSupportedException`。
