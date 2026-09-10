# QNX 移植深入：mio 后端设计分析

> 配套文档：[`porting-qnx.md`](./porting-qnx.md) §4.1（三大硬阻塞之首选）
> 分析基准：`mio 1.2.1`（`Cargo.lock` 实测）
> **接口签名警告**：本文的 Rust 签名基于 mio 1.x 架构整理，**未经本地源码核对**（分析环境无 cargo registry 缓存）。动手前请用 `cargo vendor` 或 crates.io 源码逐项比对 `src/sys/` 下的 trait 定义。凡标 **[核对]** 处即为此意。

---

## 1. 为什么 mio 是首选切入点

mio 是 tokio 的**唯一**平台适配层。一旦 mio 在 QNX 上工作：

| 依赖 mio 的层 | 是否自动可用 |
|--------------|-------------|
| `tokio`（IO driver） | ✅ |
| `hyper` / `reqwest` | ✅ |
| `tokio-tungstenite`（relay WebSocket） | ✅ |
| `tokio::net`（TCP/Unix socket） | ✅ |
| `tokio::time` | ✅ |
| `tokio::fs`（走 `spawn_blocking`） | ✅ |

即：**50 个依赖 tokio 的 crate 一次性解锁**。这也是整个移植中投资回报最高的单点工作。

---

## 2. 接入点：只需实现四个模块

mio 的后端接口面比想象中小，全部集中在 `src/sys/`：

```
src/sys/mod.rs                    ← cfg 链，加入 QNX 分支
src/sys/unix/selector/
  ├── selector.rs   (trait Selector)      ← 接口定义
  ├── epoll.rs      (Linux 参考实现)       ← 照抄结构
  └── poll_qnx.rs   ← 新增（核心工作）
src/sys/unix/waker/
  └── pipe.rs       ← 直接复用，无需新写
src/sys/unix/event/                       ← 平台事件结构（epoll::Event 的对应物）
src/sys/unix/sourcefd.rs                  ← SourceFd 与 selector 的桥接
```

必须满足的接口（**[核对]** 签名）：

```rust
// src/sys/unix/selector/selector.rs
pub(crate) trait Selector {
    fn select(&mut self, events: &mut Events, timeout: Option<Duration>) -> io::Result<()>;
    fn register(&mut self, fd: RawFd, token: Token, interests: Interest) -> io::Result<()>;
    fn reregister(&mut self, fd: RawFd, token: Token, interests: Interest) -> io::Result<()>;
    fn deregister(&mut self, fd: RawFd) -> io::Result<()>;
    // mio 1.x 为 kqueue 引入；确认 QNX 后端是否需要提供
    fn wake(&self, token: Token) -> io::Result<()>;
}
```

`sys/unix/mod.rs` 需要按 cfg 导出：

```rust
pub(crate) use self::selector::{event, Events, Selector};
pub(crate) use self::sourcefd::SourceFd;
pub(crate) use self::waker::Waker;
```

---

## 3. QNX 可用的机制

| 机制 | 类型 | 触发语义 | 说明 |
|------|------|---------|------|
| **`poll()`** | POSIX | **水平触发** | 完整支持，与 mio 语义天然匹配 |
| `select()` | POSIX | 水平触发 | `FD_SETSIZE` 限制，排除 |
| **`ionotify()`** | QNX 特有 | **边缘触发** | `ionotify(fd, action, flags, &sigevent)`，需重新 arm |
| 脉冲 + `MsgReceive` | QNX 原生 | 事件驱动 | `SIGEV_PULSE` 投递到 channel，性能最好但架构不匹配 mio |
| `epoll` 兼容层 | ？ | — | **最该先探的一项** |

---

## 4. 四个方案对比与决策路径

| 方案 | 机制 | 实现量 | 性能 | 风险 | 定位 |
|------|------|--------|------|------|------|
| **A3** | 若 QNX 提供 epoll → 复用 `epoll.rs` | ~20 行（改 cfg） | 最优 | 取决于平台 | **先探针** |
| **A1** | `poll()` + self-pipe waker | 800–1000 行 | O(n)/轮 | 低 | **主力方案** |
| A2 | `ionotify()` + sigevent | 1200+ 行 | 好 | 中高 | 后续优化 |
| B | 脉冲 + `MsgReceive` | 最多 | 最优 | 高 | 不推荐 |

**决策路径**：

```
阶段 0 探针：QNX 8.0 是否存在 epoll / epoll_* 兼容符号？
  ├── 有 → A3：复用 epoll.rs，改 cfg + 少量适配（1–2 天）
  └── 无 → A1：实现 poll 后端（3–6 周）
```

> 这一步只需半天，却可能把 3–6 周降到 1–2 天 —— 投入产出比最高。

---

## 5. `poll()` 后端的实现难点（八项）

### 难点 1：poll 无状态 → 必须用户态维护注册表

epoll 的注册状态在内核；`poll()` 每轮都要传完整 fd 集合。结构差异：

```rust
pub(crate) struct PollSelector {
    /// 权威注册表（poll 没有内核状态，这是唯一真相）
    entries: HashMap<RawFd, (Token, Interest)>,
    /// 复用的 pollfd 缓冲，避免每轮分配
    pollfds: Vec<PollFd>,
    /// 仅在注册/注销/兴趣变化时重建 pollfds
    pollfds_dirty: bool,
    /// 产出事件
    events: Events,
}
```

`reregister` / `deregister` 语义完全由自己维护 —— 这是与 epoll 后端最大的结构差异。

### 难点 2：`POLLNVAL` —— 最经典的坑

`SourceFd` 在 drop 时**不会**自动 deregister（mio 设计上依赖内核清理）。epoll 随 fd 关闭自动移除条目；**poll 后端的用户态表不会**。已关闭的 fd 会**每轮都返回 `POLLNVAL`** → 无限空转、CPU 打满。

必须在解码阶段静默清理：

```rust
// select() 事件解码阶段
if revents.contains(PollFlags::POLLNVAL) {
    self.entries.remove(&fd);     // fd 已关闭：清理
    self.pollfds_dirty = true;
    continue;                      // 不向用户投递（关闭 fd 是合法操作）
}
```

> 漏掉这段，客户端断连后整个运行时开始 100% CPU 空转。这是 poll 后端最典型的线上事故。

### 难点 3：兴趣映射很直白（poll 的核心优势）

```rust
fn to_poll_events(i: Interest) -> PollFlags {
    let mut f = PollFlags::empty();
    if i.is_readable() { f |= PollFlags::POLLIN;  }
    if i.is_writable() { f |= PollFlags::POLLOUT; }
    f
}
```

**mio 明确只用水平触发（LT）**，而 `poll()` 天生 LT —— **语义天然匹配，这是选它的最大理由**。（`ionotify` 是边缘触发，需要额外做"一次性唤醒重放"才能对齐语义。）

### 难点 4：超时换算（亚毫秒必须向上取整）

```rust
fn to_timeout_ms(t: Option<Duration>) -> c_int {
    match t {
        None => -1,                                    // 无限等待
        Some(d) if d.is_zero() => 0,                   // 立即返回
        // 关键：非零但 <1ms 必须 round up，否则退化为忙轮询
        Some(d) => d.as_millis().max(1).min(i32::MAX as u128) as c_int,
    }
}
```

tokio 的 timer 常传极短超时；少了 `.max(1)` 就是 CPU 打满。

### 难点 5：waker 用 self-pipe

```rust
// QNX 是否提供 pipe2 需确认；没有则 pipe() + fcntl()
let (r, w) = pipe()?;
fcntl(r, F_SETFL, O_NONBLOCK)?;
fcntl(w, F_SETFL, O_NONBLOCK)?;
// read 端注册到 selector，绑定保留 token
// wake():  write(w, 1 byte)；EAGAIN 表示已处于唤醒态，忽略
// reset(): 读空至 EAGAIN
```

**优先复用 mio 现有的 `sys/unix/waker/pipe.rs`**，不要新写。

### 难点 6：waker token 识别

tokio 需要判断"本次唤醒是否来自内部 waker"以避免虚假唤醒。pipe waker 用一个 `AtomicBool` 标记（`is_waker_ready`）即可 —— mio 现有实现就是这个套路。

### 难点 7：`Events` 容量配合

`poll()` 不会像 epoll 那样报告溢出；事件数上限 = 注册 fd 数。需保证 `Events` 内部缓冲足够，必要时扩容，避免丢事件。

### 难点 8：性能取舍

每轮重建 `pollfd` 数组是 O(n)。Grok Build 作为 agent 服务 fd 通常 < 100，**完全可接受**。优化手段是 `pollfds_dirty` 脏标记（仅在注册变化时重建）。**不要过早优化成 ionotify。**

---

## 6. tokio 侧需单独验证的点

mio 通了 ≠ 全通，以下 tokio 子系统有平台分支：

| 模块 | 情况 | 验证点 |
|------|------|--------|
| IO driver | 直接用 mio | 无改动 |
| `tokio::net` | socket + mio | 应直接可用 |
| `tokio::time` | timer 走 select 超时 | 验证亚毫秒超时路径 |
| **`tokio::process`** | Linux 走 **pidfd**，QNX 无 | **确认走 signal-based fallback**（`SIGCHLD` + self-pipe） |
| `tokio::signal` | `signal-hook-registry` | 纯 POSIX，验证 QNX 信号语义 |
| `tokio::fs` | `spawn_blocking` | 应可用 |

> **`tokio::process` 是风险最高的一项** —— 它决定 `run_terminal_cmd` 这类核心工具能否工作，必须单独测。

---

## 7. 工作量分解

| 工作项 | 规模 | 时间 |
|--------|------|------|
| `poll_qnx.rs`（Selector 实现） | ~500–700 行 | 1–2 周 |
| 平台 `event` 结构 + 转换 | ~150 行 | 2–3 天 |
| waker（复用 `pipe.rs`） | ~100 行 | 2–3 天 |
| `sys/mod.rs` cfg 接线 + `SourceFd` 适配 | ~50 行 | 1–2 天 |
| mio 自带测试套件在 QNX 跑通 | — | 1–2 周 |
| tokio 集成验证（net/time/process/signal） | — | 1–2 周 |
| **合计** | **~1000 行** | **3–6 周** |

---

## 8. 阶段 0 探针

```rust
// QNX aarch64 最小验证：TCP echo + timer + signal
#[tokio::main]
async fn main() -> std::io::Result<()> {
    let listener = tokio::net::TcpListener::bind("0.0.0.0:9999").await?;
    let mut sig = tokio::signal::unix::signal(
        tokio::signal::unix::SignalKind::terminate())?;
    loop {
        tokio::select! {
            Ok((mut s, _)) = listener.accept() => {
                tokio::spawn(async move {
                    let (mut r, mut w) = s.split();
                    let _ = tokio::io::copy(&mut r, &mut w).await;
                });
            }
            _ = sig.recv() => break,
        }
    }
    Ok(())
}
```

**这一个程序同时压到四项**：Selector（accept/read/write）、Waker（`select!` 的跨任务唤醒）、Events、信号。四项全过即判定 go。再加一条 `Command::new("ls")` 覆盖 `tokio::process`，基本盘就稳了。

---

## 9. 附录：`poll_qnx.rs` 实现骨架（草案）

> ⚠️ 以下为**设计草案**，非可编译代码；签名与 `Poll`/`Event` 内部结构 **[核对]** mio 1.2.1 源码后使用。

```rust
//! QNX selector backend built on POSIX `poll()`.
//!
//! `poll()` is level-triggered, matching mio's LT-only contract, so interest
//! mapping is direct. The cost is that poll has no kernel-side registration:
//! this selector owns the fd -> (token, interest) table.

use std::collections::HashMap;
use std::io;
use std::os::unix::io::RawFd;
use std::time::Duration;

use crate::{event::Source, Interest, Token};
use crate::sys::unix::selector::event::{Event, Events};

pub(crate) struct PollSelector {
    entries: HashMap<RawFd, (Token, Interest)>,
    pollfds: Vec<PollFd>,
    dirty: bool,
    events: Events,
    /// Set when `wake()` was called; consumed by `is_waker_ready`.
    waker_ready: std::sync::atomic::AtomicBool,
    waker_token: Token,
}

impl PollSelector {
    pub fn new(events_capacity: usize, waker_token: Token) -> io::Result<Self> {
        Ok(Self {
            entries: HashMap::new(),
            pollfds: Vec::new(),
            dirty: true,
            events: Events::with_capacity(events_capacity),
            waker_ready: std::sync::atomic::AtomicBool::new(false),
            waker_token,
        })
    }

    fn rebuild(&mut self) {
        if !self.dirty { return; }
        self.pollfds.clear();
        self.pollfds.reserve(self.entries.len());
        for (&fd, &(_, interest)) in &self.entries {
            self.pollfds.push(PollFd {
                fd,
                events: to_poll_events(interest),
                revents: PollFlags::empty(),
            });
        }
        self.dirty = false;
    }

    fn to_timeout_ms(t: Option<Duration>) -> libc::c_int {
        match t {
            None => -1,
            Some(d) if d.is_zero() => 0,
            Some(d) => d.as_millis().max(1).min(i32::MAX as u128) as libc::c_int,
        }
    }
}

impl Selector for PollSelector {
    fn select(&mut self, events: &mut Events, timeout: Option<Duration>) -> io::Result<()> {
        self.rebuild();
        events.clear();

        let n = loop {
            let rc = unsafe {
                libc::poll(self.pollfds.as_mut_ptr().cast(),
                           self.pollfds.len() as libc::nfds_t,
                           Self::to_timeout_ms(timeout))
            };
            if rc >= 0 { break rc as usize; }
            let err = io::Error::last_os_error();
            if err.kind() == io::ErrorKind::Interrupted { continue; } // EINTR
            return Err(err);
        };
        if n == 0 { return Ok(()); } // timeout

        // Decode. Note: iterate by index; POLLNVAL arm mutates `entries`.
        let mut closed = Vec::new();
        for i in 0..self.pollfds.len() {
            let revents = self.pollfds[i].revents;
            if revents.is_empty() { continue; }
            let fd = self.pollfds[i].fd;

            if revents.contains(PollFlags::POLLNVAL) {
                // fd closed without deregister (SourceFd drop does not deregister).
                // Drop the entry or this fd reports forever and spins the loop.
                closed.push(fd);
                continue;
            }

            let Some(&(token, interest)) = self.entries.get(&fd) else { continue };
            let readable  = revents.intersects(PollFlags::POLLIN  | PollFlags::POLLHUP | PollFlags::POLLERR);
            let writable  = revents.intersects(PollFlags::POLLOUT | PollFlags::POLLHUP | PollFlags::POLLERR);
            if token == self.waker_token {
                self.waker_ready.store(true, std::sync::atomic::Ordering::Release);
            }
            events.push(Event::new(token, readable, writable));
        }
        if !closed.is_empty() {
            for fd in closed { self.entries.remove(&fd); }
            self.dirty = true;
        }
        Ok(())
    }

    fn register(&mut self, fd: RawFd, token: Token, interests: Interest) -> io::Result<()> {
        if self.entries.contains_key(&fd) {
            return Err(io::Error::from(io::ErrorKind::AlreadyExists));
        }
        self.entries.insert(fd, (token, interests));
        self.dirty = true;
        Ok(())
    }

    fn reregister(&mut self, fd: RawFd, token: Token, interests: Interest) -> io::Result<()> {
        let slot = self.entries.get_mut(&fd)
            .ok_or_else(|| io::Error::from(io::ErrorKind::NotFound))?;
        *slot = (token, interests);
        self.dirty = true;
        Ok(())
    }

    fn deregister(&mut self, fd: RawFd) -> io::Result<()> {
        self.entries.remove(&fd);
        self.dirty = true;
        Ok(())
    }

    fn wake(&self, _token: Token) -> io::Result<()> {
        // self-pipe waker (reuse mio's sys/unix/waker/pipe.rs)
        unreachable!("delegated to the pipe waker")
    }
}
```

**接线要点**：

1. `src/sys/mod.rs` cfg 链加入 QNX 分支选中 `poll_qnx`
2. 平台 `event::Event` 提供 `is_readable()` / `is_writable()` / `is_error()` / `token()` 与 `PollFlags` 的转换
3. waker 复用 `pipe.rs`：`wake()` 写 1 字节，`reset()` 读空，`is_waker_ready()` 读 `AtomicBool`
4. `Cargo.toml` 用 `[patch.crates-io]` 指向 mio fork（仓库已有同类先例，如 `async-openai` 的 patch）

---

## 10. 动手前核对清单

| # | 待核对项 | 方式 |
|---|---------|------|
| 1 | `Selector` trait 的完整方法集与签名（是否含 `wake`） | 读 mio 1.2.1 `src/sys/unix/selector/selector.rs` |
| 2 | `sys/unix/mod.rs` 期望后端导出的符号 | 读 mio 1.2.1 `src/sys/unix/mod.rs` |
| 3 | 平台 `event::Event` 需实现的构造/查询方法 | 读 mio 1.2.1 `src/sys/unix/selector/epoll.rs` 的 `Event` |
| 4 | `SourceFd` 如何调用 register/deregister | 读 mio 1.2.1 `src/sys/unix/sourcefd.rs` |
| 5 | **QNX 是否提供 epoll 兼容** [待实测] | 在 QNX SDP 查 `<sys/epoll.h>` 与 `epoll_create1` 符号 |
| 6 | QNX 是否有 `pipe2` [待实测] | 查 `<unistd.h>`；无则退回 `pipe` + `fcntl` |
| 7 | QNX `poll()` 的 `POLLNVAL`/`POLLHUP` 语义 [待实测] | 写小 C 程序验证 |
| 8 | tokio `process` 是否走 SIGCHLD fallback [待实测] | 读 tokio `src/process/unix/` 的 cfg，并在 QNX 实测 |

---

*文档生成时间：2026-09-09（基于 `upstream/main` @ `37949780`，mio 1.2.1）*
