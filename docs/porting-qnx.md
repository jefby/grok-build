# Grok Build 移植到 QNX 8.0（aarch64）分析

> 目标平台：**QNX 8.0 / aarch64**（Rust target `aarch64-unknown-nto-qnx800`）
> 分析基准：本仓库 `upstream/main` @ `37949780`（源 rev `eb4a894`）
>
> ⚠️ **目标形态先行**：若目标是**后台 agent 进程**（`grok agent stdio|serve|leader|headless`），可跳过 TUI / PTY / 剪贴板 / 音频，工作量从 4–6 个月降至 **2.5–4 个月**。详见 [`porting-qnx-index.md`](./porting-qnx-index.md) §0（含“命令执行不需 PTY”的代码依据）。
> 说明：文中"实测数据"来自对 `crates/` 源码与 `Cargo.{toml,lock}` 的统计；标注 **[待实测]** 的条目必须在真实 QNX SDP 8.0 环境上验证后再决策。
> 架构背景见 [`architecture.md`](./architecture.md)。
> **移植专题文档**（建议从索引入）：[`porting-qnx-index.md`](./porting-qnx-index.md) —— mio 后端 [`porting-qnx-mio.md`](./porting-qnx-mio.md)、子进程 [`porting-qnx-process.md`](./porting-qnx-process.md)、PTY/终端 [`porting-qnx-pty.md`](./porting-qnx-pty.md)、文件事件 [`porting-qnx-fsevents.md`](./porting-qnx-fsevents.md)、TLS/加密 [`porting-qnx-tls.md`](./porting-qnx-tls.md)。

---

## 1. 结论摘要

**可行，但属于"运行时级"移植，不是交叉编译一遍就能通过。** 核心矛盾：Grok Build 是一个 **96 个 workspace 成员、其中 50 个依赖 tokio** 的桌面级应用，深度绑定 Linux syscall 语义；而 QNX 8.0 虽然 POSIX 认证完善，但**没有 epoll、没有 inotify、没有 Landlock、没有 D-Bus**。

三个真正的硬阻塞（按严重程度）：

| # | 阻塞 | 代码事实 | 影响面 |
|---|------|---------|--------|
| 1 | **tokio/mio 无 QNX 后端** | `mio 1.2.1`（tokio 传递依赖）；mio 仅支持 epoll / kqueue / IOCP / event ports，QNX 都不属于 | 全局（50 个 crate） |
| 2 | **`notify 8` 不支持 QNX** | `xai-fsnotify` 依赖 `notify = "8"` + `notify-debouncer-full` | 文件监听、记忆索引、配置热重载 |
| 3 | **TLS crypto provider 无 QNX 支持** | `ring 0.17` + `rustls`（`aws-lc-rs` feature）+ `rustls-native-certs` | 全部 HTTPS / MCP / 认证 |

**最重要的前提决策**：先确定目标形态。若只需 **headless / stdio / ACP 服务形态**（车载、工业、宿主嵌入的典型需求），可跳过 TUI / PTY / 剪贴板 / 音频 / 自动更新，工作量减少一半以上。

---

## 2. 平台与工具链现状

### 2.1 Rust 侧

- Target：`aarch64-unknown-nto-qnx800`，属 Rust **Tier 3**（无官方预编译 std）
- 需要 `-Z build-std=std,panic_abort`（nightly），或使用 QNX 侧提供的工具链
- 需要 QNX SDP 8.0 交叉工具链（`ntoaarch64-gcc`），并设置 `QNX_TARGET`、`QNX_HOST`，将工具链加入 `PATH`
- `libc` crate 含 QNX 支持，`compiler_builtins` 可构建 —— 基础链路基本可用

### 2.2 QNX 8.0 能力对照（关键差异）

| Linux / macOS 惯用 | QNX 8.0 | 备注 |
|---|---|---|
| `epoll` / `kqueue` | ❌ 不提供 | 有 `poll()`、`select()`、`ionotify()`、`sigevent` |
| `inotify` / `FSEvent` | ❌ | 有 `ionotify()`（DIAG 风格） |
| Landlock / Seatbelt | ❌ | 有 **ability**（`procmgr_ability()`，`PROCMGR_AID_*`） |
| systemd-inhibit / IOKit / D-Bus | ❌（无 D-Bus） | QNX 自有 IPC（`MsgSend`/`MsgReceive`、channels） |
| X11 / Wayland 剪贴板 | ❌ | 图形层是 QNX Screen |
| ALSA / CoreAudio | ❌ | QNX Sound Architecture（`io-audio`） |
| POSIX 进程 / 文件 / socket | ✅ 完整 | fork/exec、setsid、flock、Unix socket、`O_NOFOLLOW` |
| PTY | ✅ | `devc-pty` + `devctl()` |

**意外的好消息**：QNX 的 POSIX 进程/文件/socket 层很完整。以下组件**几乎可原样运行**：

- `xai-grok-workspace-daemon` —— double-fork + `setsid` + pidfile + `O_NOFOLLOW`/`0600` 姿态
- `xai-grok-shell` 的 **Leader/Follower Unix socket** IPC
- `xai-chat-state` / `xai-grok-session-events` 的 **JSONL 持久化**（`updates.jsonl` 等）
- `xai-grok-sandbox`、`xai-grok-hooks`、`xai-grok-memory` 的**优雅降级默认语义**（见 §6.4）

> 📖 **子进程层深入阅读**：`tokio::process` 在 QNX 上的走向（pidfd vs orphan queue）、`pre_exec` + `PR_SET_PDEATHSIG` 这对 Linux 专属写法、QNX 无 `prctl` 的替代方案、PTY 与 `/proc` 依赖、分层探针计划，见 **[`porting-qnx-process.md`](./porting-qnx-process.md)**。

---

## 3. 代码库平台耦合实测

```
cfg(target_os = "linux")     370 处
cfg(target_os = "macos")      88 处
cfg(target_os = "windows")    27 处
cfg(unix)                    976 处   ← QNX 会走这条路
"其他 unix" 兜底分支          19 处
```

Linux 特定代码的集中位置：

| crate | Linux 分支数 | 说明 |
|-------|-------------|------|
| `xai-fast-worktree` | 15 | `nix`（fs feature）、`FICLONE`/reflink 类操作 |
| `xai-grok-sandbox` | 9 | Landlock（`nono`） |
| `xai-grok-shell` | 7 | PTY、进程、信号 |
| `xai-grok-pager` | 6 | 终端、事件循环 |
| `xai-tty-utils` | 4 | 终端 ioctl |
| `xai-grok-tools` | 4 | cgroup 内存统计等 |

### 3.1 关键陷阱：`cfg(unix)` 依赖会被强制编译

QNX 属于 `cfg(unix)`，因此所有 `[target.'cfg(unix)'.dependencies]` 中的依赖**都会参与编译**，包括：

- **`nono`**（sandbox 的 Landlock/Seatbelt 封装，`xai-grok-sandbox/Cargo.toml`）→ QNX 上必然失败
- `libc`、`nix`（`fs`/`mount` features）→ 部分 API 在 QNX 不存在

必须把平台专属依赖的门控从 `cfg(unix)` 收紧为 `cfg(any(target_os = "linux", target_os = "macos"))` 之类的精确条件。

---

## 4. 三大硬阻塞与解法

### 4.1 mio 的 QNX 后端（决定性）

这是**唯一必须动手的底层工作**，也是投资回报最高的一件事：一旦 mio 通了，**tokio、hyper、reqwest、tokio-tungstenite 全部自动可用**（它们都只是 mio 的消费者）。

**方案 A：给 mio 写 QNX backend（推荐）**

- 实现 `Selector` / `Waker` / `Events` 三个接口
- QNX 无 epoll，用 **`poll()` 水平触发** + **自管道（self-pipe）waker** —— 这是最成熟省事的做法
- 需要边缘触发或更高性能时，再用 `ionotify()` + `sigevent`（`SIGEV_*`）补充 **[待实测：ionotify 语义细节]**
- 规模估计：1000–2000 行 + 调试，约 **3–6 周**
- 维护方式：fork mio 打 patch，同时向上游提 PR

**方案 B：裁剪 tokio features + 自建 IO 抽象（不推荐）**

- tokio 不开 `net` 即不引入 mio；但代码中大量 `tokio::net` / `tokio::process` 需改写为阻塞 IO + 线程池
- 改动散布在 50 个 crate，成本**远大于**方案 A

> **决策建议**：A 为主。先用 spike 验证可行性 —— 在 QNX aarch64 上跑通一个最小 tokio TCP echo server，这是整个项目的"探针"。

> 📖 **深入阅读**：mio 后端的接入点（四个模块 / `Selector` trait）、四个方案对比、`poll()` 后端的**八个实现难点**（`POLLNVAL` 清理、亚毫秒超时取整、self-pipe waker 等）、tokio 侧验证点、`poll_qnx.rs` 实现骨架与核对清单，见 **[`porting-qnx-mio.md`](./porting-qnx-mio.md)**。

### 4.2 文件事件 → `ionotify` 或轮询

`xai-fsnotify` 是薄封装（`watcher.rs` + `registry.rs` + `install.rs`），改造点集中：

- **短期**：轮询扫描 + mtime/内容指纹（部署场景对延迟通常不敏感，成本最低）
- **长期**：`ionotify()` + `sigevent`（`SIGEV_SIGNAL`）桥接到 mio waker

注意下游消费者：`xai-grok-memory` 的索引同步、`xai-grok-config` 的配置热重载。轮询模式必须保持**语义一致**（去抖、批量合并、事件顺序）。

### 4.3 TLS → 更换 crypto provider

`ring` / `aws-lc-rs` 均为 C/汇编，QNX 移植需改 build.rs，风险高。三个选择：

| 方案 | 优点 | 缺点 |
|------|------|------|
| **QNX 自带 OpenSSL** + `openssl` crate | 最省事 | 引入 C 依赖，版本/ABI 绑定 |
| **纯 Rust provider**（推荐） | 零 C 依赖 | 需给 rustls 实现 `CryptoProvider` |
| 移植 `ring` | —— | 不建议 |

纯 Rust provider 组成：`aes-gcm`、`chacha20poly1305`、`sha2`、`p256`（RustCrypto 全纯 Rust，QNX 无障碍）。

配套：`rustls-native-certs` 读取 Linux 的 `/etc/ssl/certs`，QNX 路径不同 → 改用 `webpki-roots`（内置根证书）或指向 QNX 证书目录。

---

## 5. 平台功能映射表

| 功能 | 现状依赖 | QNX 8.0 对应 | 工作量 |
|------|---------|-------------|--------|
| 异步运行时 | tokio / mio（epoll） | `poll()` + `ionotify`/`sigevent` | **大**（写 mio 后端） |
| 文件事件 | inotify / FSEvent（`notify`） | `ionotify()` 或轮询兜底 | 中 |
| 沙箱 | Landlock / Seatbelt（`nono`） | `procmgr_ability()` 或禁用 | 中 |
| PTY / 终端 | `openpty` / ioctl | `devc-pty` + `devctl()` | 中 |
| 电源管理 | systemd-inhibit / IOKit / `zbus` | 无 → stub | 小 |
| 剪贴板 | `arboard` | QNX Screen，无对应 → stub | 小 |
| 音频 / 语音 | `cpal` | QNX Sound Architecture | 大（或禁用） |
| 图形 / TUI | crossterm（依赖 mio） | 终端可用，事件循环依赖 mio | 中 |
| SQLite | `rusqlite`（bundled C） | gcc 可编 | 小-中 |
| tree-sitter | C 源码 | gcc 可编 | 小 |
| Git | `gix`（纯 Rust）+ `nix` | 基本可用，`FICLONE` 无 | 小-中 |
| 进程管理 | fork / setsid / flock | ✅ 完整支持 | **小** |
| Unix socket | ✅ | ✅ | 无 |

---

## 6. 分层改造方案

### 6.1 引入平台抽象层（PAL）

新建 `xai-platform-qnx`（或 `xai-platform`，含 `qnx` backend），把下列能力收敛到 trait，**用 `--features platform-qnx` 切换，而不是继续撒 `#[cfg]`**：

```rust
trait Sandbox          { fn available(&self) -> bool; /* ... */ }
trait FsEvents         { fn watch(&self, path: &Path, opts: WatchOpts) -> WatchHandle; }
trait PowerManagement  { fn prevent_sleep(&self) -> Guard; }
trait Clipboard        { fn get/set(&self, ...); }
trait AudioCapture     { /* voice 相关 */ }
```

长期收益最大的一步：阻止平台分支继续指数增长。

### 6.2 沙箱（`xai-grok-sandbox`）

现状已有运行时降级：

```rust
// xai-grok-sandbox/src/lib.rs:92
fn restrict_network_at_known_linux_launches(configured: bool) -> bool {
    configured && cfg!(target_os = "linux")
}
```

即非 Linux 上返回"不可用"而非编译失败。需要做的：

1. 把 `nono` 从 `cfg(unix)` 收紧为 `cfg(any(linux, macos))`
2. QNX 上默认走 `NoopSandbox`（`configured == false`）
3. 可选增强：用 QNX `procmgr_ability()` 做能力裁剪，映射 `PROCMGR_AID_*`
4. 应用层已有的 `CapabilityMode`（按 `ToolKind` 过滤）+ 权限管理器继续生效，作为主要防线

### 6.3 Worktree（`xai-fast-worktree`）

- `FICLONE` / reflink 类零拷贝操作在 QNX 不可用 → **fallback 到普通文件复制**（性能下降，语义正确）
- `nix`（`fs` feature）在 Linux 分支 → QNX 走通用路径或 `libc` 直调
- 注意 `crates/codegen/xai-fast-worktree/src/nfs/` 的实现依赖（client/liveness）**[待实测]**

### 6.4 善用已有的优雅降级

这些默认行为对移植非常友好，**不要推翻**：

- `xai-grok-sandbox` 的"不可用"语义
- `xai-grok-hooks` 的 **fail-open** 默认（hook 失败不阻塞）
- `xai-grok-memory` **v2 隔离管道**（与 legacy `memory/` 树完全隔离，便于单独裁剪）
- 旁路调用（recap/btw）的 best-effort 语义

---

## 7. 分阶段路线图

> ⚠️ 下表为**全形态**路线图。若目标是**后台 agent 进程**（推荐，见 [`porting-qnx-index.md`](./porting-qnx-index.md) §0），可跳过 PTY/TUI 项，总计降至 **2.5–4 个月**。

| 阶段 | 内容 | 周期 | 验收标准 |
|------|------|------|---------|
| **0. 可行性探针** | QNX SDP 8.0 工具链 + `-Z build-std`；编译 `libc`/`tokio`（含 mio 后端 spike） | 1–2 周 | **最小 tokio echo server 在 QNX aarch64 跑通**（go/no-go 决策点） |
| **0.5 构建面裁剪**（推荐） | `xai-grok-pager-bin` 的 TUI feature 化；PTY 代码 feature 排除 | 3–5 天 | 能构建**不含** crossterm/portable-pty/ratatui 的服务二进制 |
| **1. 纯 Rust 叶子层** | `xai-grok-sampler`、`-sampling-types`、`xai-grok-compaction`、`xai-compaction-transcript`、`xai-prompt-queue`、`xai-workflow`（rhai）、`xai-token-estimation`、`xai-grok-secrets` | 2–3 周 | 建立 QNX target 下的**依赖裁剪清单** |
| **2. 平台抽象层** | `xai-platform-qnx`：sandbox / fs-events / TLS roots（后台 agent 形态无需 power / clipboard / audio / PTY） | 1–2 周 | 所有 `#[cfg]` 平台分支收敛为 PAL 注入 |
| **3. 服务形态跑通** | `grok agent stdio` / `serve` / `leader` / `headless`，**跳过 TUI** | 3–4 周 | 完成一轮真实对话 + 工具调用（读文件、跑命令） |
| **4. 工作区与沙箱语义** | worktree fallback、git 操作（`gix`）、权限模型叠加 | 3–5 周 | 文件编辑/回滚/检查点语义正确 |
| **5. PTY + TUI（仅交互场景）** | `portable-pty` 适配（`devc-pty` + terminfo）+ crossterm/ratatui | 4–8 周 | 全屏 TUI 可用 |

**总计**：后台 agent 形态 **2.5–4 个月**；含 TUI 全形态 **4–6 个月**（1–2 名熟悉 Rust + QNX 的工程师）。其中 **mio 后端是唯一的全局风险点**。

---

## 8. 风险登记与待实测项

| 风险 | 说明 | 缓解 |
|------|------|------|
| mio QNX 后端可行性 | 最大不确定性 | 阶段 0 探针；`poll()`+self-pipe 是低风险路径 |
| QNX 8.0 是否提供 `epoll` **[待实测]** | 若有兼容层可大幅简化 | 实测确认，勿依赖二手资料 |
| `ionotify()` 语义细节 **[待实测]** | 边缘/水平触发、事件合并行为 | 先用轮询兜底 |
| `ring` / `aws-lc` 交叉编译 **[待实测]** | C/汇编平台适配 | 改用纯 Rust provider |
| SQLite / tree-sitter 交叉编译 | C 代码需 QNX gcc 支持 | 阶段 1 验证 |
| QNX Tier 3 工具链稳定性 | 无官方 std，需 build-std | 固定 nightly 版本 + 自建工具链镜像 |
| `#[cfg]` 分支失控 | 现有 370 处 Linux 分支 | PAL 收敛（阶段 2） |

---

## 9. 附：验证方法

复现本文统计：

```bash
# 平台 cfg 分布
grep -rhoE 'cfg\(target_os = "[a-z]+"' crates/ --include="*.rs" | sort | uniq -c | sort -rn

# Linux 特化代码集中位置
grep -rl 'target_os = "linux"' crates/ --include="*.rs" | sed 's|/src/.*||' | sort | uniq -c | sort -rn

# 平台门控依赖
grep -rn "target\.'cfg" crates/*/*/Cargo.toml

# tokio 依赖面
grep -rl "tokio" crates/ --include="Cargo.toml" | wc -l

# 传递依赖版本
grep -A2 '^name = "mio"' Cargo.lock
```

**[待实测]** 项需在真实 QNX SDP 8.0（aarch64）环境上逐项确认后回填本文档。

---

*文档生成时间：2026-09-09（基于 `upstream/main` @ `37949780`）*
