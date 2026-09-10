# QNX 移植深入：PTY 与终端层分析

> 配套文档：[`porting-qnx.md`](./porting-qnx.md)（总览）、[`porting-qnx-mio.md`](./porting-qnx-mio.md)、[`porting-qnx-process.md`](./porting-qnx-process.md)
> 分析基准：`portable-pty 0.9.0`、`alacritty_terminal 0.26.0`、`crossterm 0.28`、`ratatui 0.29`、`vte 0.15.0`
> 标 **[待实测]** 处需在真实 QNX SDP 8.0 验证；标 **[核对]** 处需对照依赖源码。

---

## 1. 结论

终端层要**分成两个独立目标**看，否则会严重高估工作量：

| 目标 | 内容 | 依赖 | 必要性 |
|------|------|------|--------|
| **A. 命令执行（PTY 执行）** | `run_terminal_cmd`、后台任务、子 Agent 执行命令 | `portable-pty` + `libc` | **必需**（核心工具） |
| **B. 交互 TUI** | 全屏界面、输入、渲染 | `crossterm` + `ratatui` + **mio** | 可选（服务形态不需要） |

**关键判断**：如果目标形态是 headless / stdio / ACP 服务，**只需要打通 A**，B 可以后置甚至不做。而 A 的难点集中在 `portable-pty` 能否在 QNX 编译 + `devc-pty` 是否运行，**不涉及 mio**。

---

## 2. 依赖与使用面（实测）

```
portable-pty = "0.9"          ← PTY 创建（unix: libc openpty/posix_openpt）
alacritty_terminal = "0.26.0" ← 终端模拟（ptyctl 使用）
crossterm = "0.28"            ← TUI 输入/输出（event-stream feature → mio）
ratatui = "0.29"              ← TUI 渲染（纯 Rust）
vte = "0.15.0"                ← 转义序列解析（纯 Rust）
```

**PTY 使用点**：

```
crates/codegen/ptyctl/src/pty.rs:96              pty_system.openpty(pty_size)   ← portable-pty
crates/codegen/xai-grok-shell-terminal/src/pty_session.rs:260  .openpty(size)      ← portable-pty
crates/codegen/xai-grok-pager-pty-harness/src/pty_spawn.rs    自研 fork/exec（因 portable-pty 无 pre_exec）
```

`ptyctl` 的定位（`lib.rs` 头部注释）：**headless PTY 控制器**，基于 `alacritty_terminal`，提供"spawn 进程 → 发送按键 → 读屏内容（text/styled/HTML）→ HTTP REST API"。

**终端属性使用点**：

```
xai-crash-handler/src/handler.rs:192   libc::termios + tcgetattr(0)  崩溃时恢复终端
```

---

## 3. 平台能力映射

| 能力 | 现状依赖 | QNX 8.0 | 说明 |
|------|---------|---------|------|
| PTY 创建 | `libc::openpty` / `posix_openpt` | ✅ 需 `devc-pty` 运行 | **[待实测]** libc 绑定 + 设备节点 |
| 设为控制终端 | `TIOCSCTTY` | ✅ ioctl | **[待实测]** |
| 窗口大小 | `TIOCGWINSZ` / `TIOCSWINSZ` | ✅ ioctl | **[待实测]** |
| termios | `tcgetattr` / `tcsetattr` | ✅ POSIX | 应可用 |
| 转义序列解析 | `vte`（纯 Rust） | ✅ | 无平台依赖 |
| TUI 渲染 | `ratatui`（纯 Rust） | ✅ | 无平台依赖 |
| 键盘/鼠标事件 | `crossterm` | ⚠️ `event-stream` 依赖 **mio** | 见 §5 |
| terminfo / TERM | 环境变量 + 数据库 | ⚠️ QNX 需提供 terminfo | `TERM=xterm-256color` 已由代码设置 |

QNX 侧要点：

- QNX 的 PTY 由 **`devc-pty`** 资源管理器提供；设备节点形如 `/dev/ptyp*`、`/dev/ttyp*`（或 POSIX `posix_openpt` 路径）**[待实测]**
- 若 `devc-pty` 未运行，`openpty`/`posix_openpt` 会失败 → 部署清单里必须加上"启动 `devc-pty`"

---

## 4. A 路：命令执行（必需）

### 4.1 `portable-pty 0.9` 的风险

`portable-pty`（wezterm 项目）的 unix 实现在 `src/unix.rs`，依赖：

- `libc::openpty`（或 `posix_openpt` + `grantpt` + `unlockpt` + `ptsname`）
- `libc::ioctl(TIOCSWINSZ)` 等
- `std::os::unix::process::CommandExt` 的 `pre_exec`（grok-build 的 PTY harness 就是绕开它自己实现）

**风险点**：`portable-pty` 的 `[target.'cfg(unix)']` 依赖与 `libc` 绑定是否覆盖 QNX **[待实测]**。QNX 是 `cfg(unix)`，所以会尝试编译 —— 若其内部用了 Linux/macOS 专有常量，编译失败。

**三个应对**：

| 方案 | 说明 | 代价 |
|------|------|------|
| 1. 直接编译通过 | 最理想 | 0 |
| 2. 打补丁适配 QNX | 改 `portable-pty` 的 unix 分支（fork + `[patch.crates-io]`） | 1–2 周 |
| 3. 自研 PTY 后端 | 用 `libc` 直接 `posix_openpt`/`openpty` + `ioctl` | 1–2 周 |

> 仓库里 `ptyctl` 已经把 PTY 操作封装成自己的 `pty.rs`（`PtyConfig` + `spawn`），**方案 3 的沉没成本比看起来低** —— 只需要替换它内部对 `portable-pty` 的调用。

### 4.2 与 process 层的交叉

PTY 子进程的 spawn 走 `pre_exec`（`setsid` + `TIOCSCTTY` + PDEATHSIG），因此：

- **受 [`porting-qnx-process.md`](./porting-qnx-process.md) §4 L2 影响**：`pre_exec` 强制 `fork` 路径，QNX 上应改为 `posix_spawn` 的 `POSIX_SPAWN_SETSID` 等属性
- PDEATHSIG 需替换为 channel 断连守卫（同 §5）

---

## 5. B 路：交互 TUI（可选）

### 5.1 crossterm 的隐藏依赖：mio

`crossterm 0.28` 的 **`event-stream`** feature 在 unix 上通过 `mio` 做事件轮询（用于 `EventStream` 异步接口）。仓库里启用了该 feature：

```toml
# crates/codegen/xai-grok-pager-minimal/Cargo.toml:26
crossterm = { workspace = true, features = ["event-stream", "bracketed-paste"] }
```

**推论**：TUI 路径也依赖 mio → **B 路完全受 [`porting-qnx-mio.md`](./porting-qnx-mio.md) 阻塞**。若 mio 通了，crossterm 大概率直接可用；若不通，退路是改用 crossterm 的阻塞式 `event::read()`（不用 event-stream），但会牺牲响应性/与其他 async 任务的整合。

**[核对]**：确认 `crossterm 0.28` 的 `event-stream` 在 unix 上的实现是否确实经 `mio`（读其 `Cargo.toml` 与 `src/event/source/unix/`）。

### 5.2 其他 B 路考虑

- `ratatui 0.29` 与 `vte 0.15` **纯 Rust，无平台依赖** —— 渲染不是问题
- `alacritty_terminal 0.26` 的 `tty` 模块有自己的平台分支（Linux/macOS/Windows），QNX 可能不命中 **[待实测]**；但 `ptyctl` 只用它的**终端模拟（vte 解析）**部分，PTY 由 `portable-pty` 提供 —— 这部分是纯 Rust，风险低
- terminfo：需在 QNX 镜像里提供 `xterm-256color` 定义（或依赖 crossterm 的内置转义序列）

---

## 6. 改动清单

| 文件 / 依赖 | 问题 | 改法 |
|------------|------|------|
| `portable-pty 0.9` | QNX 编译未知 | 试编 → 打补丁 或 自研后端 |
| `ptyctl/src/pty.rs` | 依赖 `portable-pty` 的 `openpty` | 若打补丁/自研，替换此处 |
| `xai-grok-shell-terminal/src/pty_session.rs` | `openpty` + `cfg(unix)` 分支 | 同上 |
| `xai-grok-pager-pty-harness/src/pty_spawn.rs` | `pre_exec` + PDEATHSIG | 见 process 文档 §6 |
| `xai-crash-handler/src/handler.rs` | `termios` 恢复 | QNX 验证 `tcgetattr(0)` 在无 TTY 时的行为 |
| `crossterm` (`event-stream`) | 依赖 mio | 受 mio 后端影响；退路用阻塞 `event::read()` |
| 部署 | `devc-pty` 未运行 | 启动脚本加入 `devc-pty` |

---

## 7. 探针计划

```rust
// T1: PTY 创建（最小）
let pair = native_pty_system().openpty(PtySize { rows: 24, cols: 80, ..Default::default() })?;
// 失败 → 检查 devc-pty 是否运行

// T2: PTY 里执行命令 + 读输出
let mut cmd = CommandBuilder::new("echo");
cmd.arg("hello");
let mut child = pair.slave.spawn_command(cmd)?;
// 读 master 端，应看到 "hello\r\n"

// T3: 窗口大小
pair.master.resize(PtySize { rows: 40, cols: 120, ..Default::default() })?;

// T4: 交互（发送按键 + 读屏）
// 通过 ptyctl: spawn → send keys → read screen as text

// T5: 崩溃处理器恢复终端（TUI 场景）
// 触发 panic → 验证 termios 被恢复
```

**T1 是最关键的 30 分钟验证**：`devc-pty` + `openpty` 通不通，决定 A 路是"零成本"还是"1–2 周"。

---

## 8. 工作量估算

| 项 | 时间 |
|----|------|
| T1–T3 PTY 基础验证 | 2–3 天 |
| `portable-pty` 打补丁（若需要） | 1–2 周 |
| 或自研 PTY 后端（若需要） | 1–2 周 |
| PTY 会话（`ptyctl` / `pty_session`）适配 | 3–5 天 |
| terminfo / TERM 环境 | 1–2 天 |
| **A 路合计** | **1–3 周** |
| B 路（TUI）：依赖 mio + crossterm 验证 | 受 mio 阻塞，另计 1–2 周 |

---

## 9. 待核对 / 待实测

| # | 项 | 方式 |
|---|----|------|
| 1 | `libc` crate 是否给 QNX 暴露 `openpty`/`posix_openpt`/`TIOCSCTTY` **[待实测]** | 读 `libc` 的 qnx 模块 |
| 2 | `portable-pty 0.9` 在 QNX 能否编译 **[待实测]** | 直接试编 |
| 3 | QNX `devc-pty` 设备节点与 `posix_openpt` 路径 **[待实测]** | 查 QNX 文档 + `ls /dev` |
| 4 | `TIOCSCTTY` / `TIOCSWINSZ` 在 QNX 的可用性 **[待实测]** | C 小程序 |
| 5 | `crossterm 0.28` `event-stream` 是否经 mio **[核对]** | 读其源码 |
| 6 | `alacritty_terminal 0.26` 在 QNX 的编译情况 **[待实测]** | 直接试编 |
| 7 | QNX terminfo 数据库是否含 `xterm-256color` **[待实测]** | `infocmp xterm-256color` |

---

*文档生成时间：2026-09-09（基于 `upstream/main` @ `37949780`）*
