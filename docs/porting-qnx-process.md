# QNX 移植深入：`tokio::process` 与子进程层分析

> 配套文档：[`porting-qnx.md`](./porting-qnx.md)（总览）、[`porting-qnx-mio.md`](./porting-qnx-mio.md)（mio 后端）
> 分析基准：**tokio 1.52.3**、rustix 0.38.44、portable-pty 0.9.0、signal-hook-registry 1.4.8
> **核对警告**：tokio 的 process/signal 内部 cfg 分支随版本演进较快，本文标 **[核对]** 处请对照 tokio 1.52.3 源码确认。标 **[待实测]** 处需在真实 QNX SDP 8.0 上验证。

---

## 1. 结论与风险定位

process 是**仅次于 mio 的第二大风险面**，但与 mio 的"单点阻塞"不同，它是**散布式风险**：

- **46 个文件 / 14 个 crate** 依赖 spawn 子进程
- 其中夹着 `pre_exec` + `PR_SET_PDEATHSIG` 这类 **Linux 专属**写法，且恰好在 **PTY 核心路径**上

关键判断：**`tokio::process` 本身大概率能编译能跑**（QNX 会走非 Linux 的 SIGCHLD 路径，SIGCHLD 是 POSIX 基本要求）。**真正的坑在它下面两层** —— Rust std 的 `fork` vs `posix_spawn` 选择，以及 QNX 特有的进程语义（无 `prctl`）。

---

## 2. tokio 的两条路径与 QNX 走向

tokio 在 unix 上有两套进程实现：

| 路径 | 平台 | 机制 |
|------|------|------|
| **pidfd 路径** | Linux only（内核 ≥ 5.3） | `rustix` 的 `pidfd_open` + `AsyncFd` 轮询，**不碰全局 SIGCHLD** |
| **orphan queue 路径** | 其他所有 unix（macOS / BSD / **QNX**） | 全局 SIGCHLD handler（`signal-hook-registry`）+ `OrphanQueue` + `waitpid(WNOHANG)` |

**QNX 走 orphan queue 路径**（`cfg(target_os = "linux")` 不命中）。依赖链：

```
tokio::process
  → signal-hook-registry（sigaction + self-pipe）   ← POSIX，应可用
  → orphan::OrphanQueueImpl（Mutex + VecDeque）
  → waitpid(WNOHANG)                                ← POSIX，QNX 支持
  → 信号驱动 / mio waker                            ← 见 porting-qnx-mio.md
```

**这条路是为 macOS/BSD 设计的通用 POSIX 实现，不含 Linux 专属 syscall。**

### 2.1 三个前置条件

1. **必须启用 tokio 的 `signal` feature**，否则 SIGCHLD 驱动缺失 → 子进程退出无人回收 → **僵尸累积**。
   仓库现状：`Cargo.toml` 中 `tokio = { features = ["full"] }` → 已包含。
2. **SIGCHLD 已有多订阅者（已核实，不冲突）**：grok-build 自身也订阅了 tokio 的 SIGCHLD 流：

   ```rust
   // xai-grok-tools/src/computer/local/terminal.rs:124
   signal: tokio::signal::unix::signal(tokio::signal::unix::SignalKind::child()).ok(),
   ```

   tokio 的 signal 实现基于 `signal-hook-registry`，它注册**一个**底层 handler 再分发给所有订阅者，因此 tokio 内部 orphan queue 与用户代码的 `SignalKind::child()` 流**可以共存**。
   但要注意：**外部（非 tokio）用原生 `sigaction` 覆盖 SIGCHLD handler 会破坏它** —— 仓库对此已有认知（`xai-grok-tools/src/computer/local/terminal.rs:967` 与 `xai-tty-utils/src/child_wait.rs:16` 的注释引用了 GB-5008）。

   隐含要求：**tokio 的 signal driver 必须在 QNX 上工作**，而它依赖 mio 的 waker + self-pipe → **与 mio 后端直接耦合**（见 [`porting-qnx-mio.md`](./porting-qnx-mio.md)）。
3. **tokio 1.52.3 的具体 cfg 判定** **[核对]**：`build.rs` 的 pidfd 探测、`src/process/unix/{mod,orphan,reap}.rs` 的分支走向 —— 本文写作时的知识早于该版本。

### 2.2 现成的跨平台抽象（好消息）

代码里已经有一个跨平台的子进程退出抽象，QNX 可直接落在 unix 分支：

```rust
// xai-grok-tools/src/computer/local/terminal.rs:110
/// Wakes the sweep when a child of this actor exits. Unix listens to the
/// shared SIGCHLD; Windows registers one handle wait per spawned child.
struct ChildExitWake {
    #[cfg(unix)]      signal: Option<tokio::signal::unix::Signal>,
    #[cfg(not(unix))] notify: std::sync::Arc<tokio::sync::Notify>,
}
```

即：**QNX 会自动落到 SIGCHLD 分支**，无需新增平台代码；这也说明团队对子进程退出同步已有抽象意识 —— 移植时应沿用这个模式，而不是另起炉灶。

---

## 3. grok-build 的实际进程使用面（实测）

```
tokio::process 使用：46 个文件 / 14 个 crate
分布：xai-grok-shell-terminal、xai-grok-workspace、xai-grok-tools、
     xai-grok-hooks、xai-grok-mcp、xai-grok-shell、xai-grok-workspace-daemon、
     xai-grok-sandbox、xai-grok-update、xai-grok-login、xai-tty-utils 等
```

对应的**核心功能**（缺一不可）：

| 功能 | 依赖方式 |
|------|---------|
| `run_terminal_cmd`（核心工具） | `xai-grok-shell-terminal` 的 runner |
| PTY 会话 | `portable-pty 0.9` + `xai-grok-shell-terminal/pty_session.rs` |
| `ripgrep` 内容搜索 | spawn `rg` |
| Hooks（command runner） | spawn hook 脚本 |
| MCP 服务器 | spawn server 进程 |
| 子 Agent / `task` 工具 | 经由 terminal backend 执行 |
| `xai-fast-worktree` | spawn `git` |
| 自动更新 | spawn updater |

### 3.1 Linux 专属写法（必须替换）

**`PR_SET_PDEATHSIG`（QNX 无 `prctl`）**

```
xai-grok-pager/src/app/mermaid_worker.rs:454   libc::prctl(PR_SET_PDEATHSIG, SIGKILL)
xai-grok-pager-pty-harness/src/pty_spawn.rs    pty_session_pre_exec()（注释明写 Linux PDEATHSIG）
```

**`pre_exec` 使用点**（强制走 fork 路径）

```
xai-grok-pager-pty-harness/src/pty_spawn.rs:200    cmd.pre_exec(pty_session_pre_exec)
xai-grok-pager/src/wrap_cmd.rs:181                 CommandExt
xai-grok-pager/src/app/screen_mode_relaunch.rs:213 CommandExt
xai-grok-mermaid/src/mmdc.rs:89                    setsid / console detach
```

两者**绑定在一起**：`pre_exec` 闭包内既做 `setsid` / `TIOCSCTTY`，又 arm PDEATHSIG。

### 3.2 `/proc`、cgroup、seccomp 依赖

```
xai-fast-worktree/src/api/gc/process_scan.rs:67   read_dir("/proc")          进程扫描
xai-fast-worktree/src/mount_info.rs:50           /proc/self/mountinfo       mount 检测
xai-fast-worktree/src/btrfs/detect.rs:134        /proc/self/mountinfo       btrfs 检测
xai-fast-worktree/bin/worktree_lifecycle_bench/runtime.rs:43  /proc/self/cgroup + /sys/fs/cgroup
xai-fast-worktree/bin/worktree_lifecycle_bench/runtime.rs:752 /proc/self/ns/mnt
xai-grok-sandbox/src/child_net.rs:142            seccomp + PR_SET_NO_NEW_PRIVS
```

QNX 的 `/proc` 存在但**结构不同**（主要面向调试器）：无 `mountinfo`、无 `cgroup`、无 `ns/mnt`。

---

## 4. 三层风险详解

### L1 — tokio 层（风险：低）

走 orphan queue 路径，机制纯 POSIX。**代码改动为零**，只需验证。风险点：SIGCHLD 投递可靠性 + handler 独占性。

### L2 — Rust std 层（风险：中，最易被忽略）

Rust 的 `std::process::Command` 在 unix 上有两条内部路径：

| 条件 | 使用路径 |
|------|---------|
| 无 `pre_exec`，文件操作可用标准 file_actions 表达 | **`posix_spawn`**（高效，QNX 推荐） |
| 有 `pre_exec`（需在 fork 与 exec 之间执行闭包） | **`fork` + `exec`** |

**grok-build 的 PTY / mermaid / wrap 路径都带 `pre_exec` → 强制走 `fork`。**

QNX 的 `fork()` 支持（POSIX 要求），但：

- QNX 官方**明确推荐 `posix_spawn()` / `spawn()`**，fork 被视为昂贵路径
- 多线程进程内 fork 是 POSIX 通病（子进程仅保留调用线程），QNX 上更易撞上
- QNX 的 `vfork()` 在 exec 场景更高效

**建议**：把 `pre_exec` 的工作改写为 `posix_spawn` 的 file_actions / spawnattrs 可表达的形式，或改用 QNX 原生 `spawn()`。这是**明确的改造工作量**。

### L3 — OS 层：QNX 进程语义（风险：中高）

| 项 | QNX 8.0 情况 | 影响 |
|----|-------------|------|
| `posix_spawn` | ✅ 支持 | 主路径可用 |
| QNX 原生 `spawn()` | ✅ | 可选优化（属性更丰富） |
| `fork()` | ⚠️ 支持但慢、多线程受限 | `pre_exec` 路径退化 |
| `waitpid(WNOHANG)` | ✅ | orphan queue 可用 |
| `SIGCHLD` | ✅ POSIX 要求 | 需实测投递行为 **[待实测]** |
| 进程组 / `setsid` / `setpgid` | ✅ | PTY 会话管理可用 |
| PTY（`openpty` / `posix_openpt`） | ✅ 需 `devc-pty` 运行 | **需确认 libc / portable-pty 绑定** **[待实测]** |
| `TIOCSCTTY` / `TIOCSWINSZ` | ✅（ioctl） | **[待实测]** |
| `prctl` / PDEATHSIG | ❌ **无** | **必须替代** |
| `seccomp` | ❌（改用 ability） | 禁用 |
| cgroup / namespace | ❌ | 禁用 |

---

## 5. PDEATHSIG 的 QNX 替代（设计题）

> ⚠️ **后台 agent 形态下本节的必要性大幅降低**：两处 `PDEATHSIG` 调用点都在 **TUI 相关路径**（`mermaid_worker` 渲染、`pty_spawn` PTY 会话）。若只做 `grok agent stdio|serve|leader|headless`，可先不替换，把它归入 TUI 工作包（见 [`porting-qnx-index.md`](./porting-qnx-index.md) §0）。

`PR_SET_PDEATHSIG(SIGKILL)` 解决的是：**父进程被 SIGKILL（无 Drop 机会）时，子进程不能泄漏**。

QNX 无等价物，三个方案：

| 方案 | 机制 | 覆盖 SIGKILL | 适用性 |
|------|------|-------------|--------|
| **1. Channel 断连检测（推荐）** | 父进程创建 channel 并把连接 ID 传给子进程；子进程 `ionotify()` 监听该 connection 断开 | ✅ | 库内实现，最接近原语义 |
| 2. `umbrella` / `waitfor` 进程监控 | QNX 自带守护进程，声明"某进程退出则杀掉进程组" | ✅ | 产品化部署（启动脚本声明），非库内 |
| 3. 父进程 signal handler 里 kill 进程组 | 覆盖普通退出与可控信号 | ❌ | 降级方案 |

> **建议**：方案 1 做成平台抽象（`xai-platform-qnx` 的 `child_lifetime_guard`），替换 PTY 与 mermaid 两处调用点。

---

## 6. 代码改动清单

| 文件 | 问题 | 改法 |
|------|------|------|
| `xai-grok-pager-pty-harness/src/pty_spawn.rs` | `pre_exec` + PDEATHSIG | 拆为 posix_spawn 属性 + channel 断连守卫 |
| `xai-grok-pager/src/app/mermaid_worker.rs` | `prctl(PDEATHSIG)` | 同上守卫 |
| `xai-grok-pager/src/wrap_cmd.rs`、`app/screen_mode_relaunch.rs` | `CommandExt` | 确认用途，尽量改 posix_spawn |
| `xai-grok-shell-terminal/src/pty_session.rs` | `portable-pty` + `cfg(unix)` 分支 | 验证 QNX 编译；必要时换自研 PTY（`ptyctl` 已有基础） |
| `xai-fast-worktree/src/mount_info.rs`、`btrfs/detect.rs` | `/proc/self/mountinfo` | QNX 走"无 mount 信息"分支 |
| `xai-fast-worktree/src/api/gc/process_scan.rs` | `read_dir("/proc")` | QNX 无等价 → 禁用该 GC 策略 |
| `xai-grok-sandbox/src/child_net.rs` | seccomp + `no_new_privs` | `cfg(target_os = "linux")` 门控 |
| `Cargo.toml` | `portable-pty` 等 unix 依赖 | 收紧为精确平台条件 |

---

## 7. 验证计划（分层探针，各约半小时）

```rust
// P1: tokio::process 基础
let out = tokio::process::Command::new("ls").output().await?;

// P2: SIGCHLD / orphan queue —— 最关键
let mut child = tokio::process::Command::new("sleep").arg("1").spawn()?;
let status = child.wait().await?;   // 卡住 = orphan queue 坏了
// 再确认 ps 中无 <defunct> 累积

// P3: kill_on_drop / kill()
child.kill().await?;

// P4: pre_exec 路径（触发 fork+exec，验证 QNX 多线程下可用）
unsafe { cmd.pre_exec(|| { libc::setsid(); Ok(()) }); }

// P5: PTY —— openpty + 回显 + TIOCSWINSZ（需 devc-pty 运行）

// P6: PDEATHSIG 替代 —— 父进程被 SIGKILL 后子进程是否消失
```

**P2 与 P6 最关键**：

- **P2** 决定 agent 长时间运行会不会僵尸泄漏
- **P6** 决定 PTY / mermaid 异常退出后会不会残留进程

---

## 8. 工作量估算

| 工作项 | 时间 |
|--------|------|
| L1 验证（tokio process + signal） | 2–3 天 |
| L2 改造（`pre_exec` → `posix_spawn` / QNX `spawn`） | 1–1.5 周 |
| PDEATHSIG 替代（channel 断连守卫）+ 替换 2 处调用点 | 1 周 |
| PTY 层验证 / 适配（`portable-pty` 或自研） | 1–2 周 |
| `/proc`、cgroup、seccomp 的 fallback | 3–5 天 |
| **合计** | **4–6 周** |

与 mio 后端的 3–6 周**部分可并行**（两者都依赖 `porting-qnx.md` §7 阶段 0 探针通过）。

---

## 9. 待核对 / 待实测清单

| # | 项 | 方式 |
|---|----|------|
| 1 | tokio 1.52.3 在 QNX 上的 process 分支走向 **[核对]** | 读 `tokio/src/process/unix/`、`build.rs` |
| 2 | tokio `signal` feature 是否已启用 | 读仓库 `Cargo.toml`（现状 `features = ["full"]`） |
| 3 | SIGCHLD handler 是否冲突 | **已核实不冲突**：grok-build 用 tokio 的 `SignalKind::child()`（`xai-grok-tools/.../terminal.rs:124`），与 tokio 内部 orphan queue 共享 `signal-hook-registry` 注册；仅原生 `sigaction` 覆盖才会破坏 |
| 4 | QNX `SIGCHLD` 投递行为 **[待实测]** | 小 C 程序 + tokio P2 探针 |
| 5 | QNX `posix_spawn` 支持哪些 spawnattr **[待实测]** | 查 `<spawn.h>` |
| 6 | QNX 多线程进程 `fork()` 限制 **[待实测]** | 查 QNX 文档 + P4 探针 |
| 7 | `libc` crate 对 QNX 的 `openpty`/`posix_openpt`/`TIOCSCTTY` 绑定 **[待实测]** | 读 `libc` 的 qnx 模块 |
| 8 | `portable-pty 0.9` 在 QNX 是否可编译 **[待实测]** | 直接试编 |
| 9 | QNX `devc-pty` 是否运行、设备节点路径 **[待实测]** | `ls /dev/ptyp*` / `posix_openpt` |
| 10 | QNX channel 断连检测 API 细节 **[待实测]** | 查 `ionotify` + connection 文档 |

---

*文档生成时间：2026-09-09（基于 `upstream/main` @ `37949780`）*
