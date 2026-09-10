# QNX 移植深入：文件系统事件层分析

> 配套文档：[`porting-qnx.md`](./porting-qnx.md)（总览）
> 分析基准：`notify 8`、`notify-debouncer-full 0.5`、`notify-debouncer-mini`
> 标 **[待实测]** 处需在真实 QNX SDP 8.0 验证。

---

## 1. 结论

这是三大硬阻塞中**最容易降级解决**的一个。

**核心判断**：文件事件对延迟不敏感（记忆索引、配置热重载、skills 发现都容忍亚秒到秒级延迟），因此 **"轮询兜底"是完全可接受的方案** —— 它把这一项从"硬阻塞"降级为**性能取舍**。

| 方案 | 复杂度 | 延迟 | 建议 |
|------|--------|------|------|
| **轮询（mtime + 指纹）** | 低 | 200ms–2s | **兜底首选**，先跑通 |
| `ionotify()` + `sigevent` | 中高 | 毫秒级 | 后续优化 |
| QNX inotify 兼容层（若有） | 低 | 毫秒级 | **[待实测]** 先探 |

---

## 2. 现状与使用面（实测）

```
notify = "8"                      ← 核心库（RecommendedWatcher）
notify-debouncer-full = "0.5"     ← 去抖（xai-fsnotify 用）
notify-debouncer-mini             ← 另一个去抖实现（xai-grok-shell 用）
```

### 2.1 依赖 `notify` 的四个 crate

| crate | 用途 | 依赖形式 |
|-------|------|---------|
| `xai-fsnotify` | **主封装**：工作区文件事件源 | `notify` + `notify-debouncer-full` |
| `xai-grok-memory` | 记忆文件监听（索引同步） | `notify` |
| `xai-grok-shell` | 会话内文件监听 | `notify` + `notify-debouncer-mini` + `xai-fsnotify` |
| `xai-grok-pager-render` | 渲染侧（**optional**） | `notify`（optional） |

> **注意**：不是单一接入点，共有 **4 处**。但 `xai-fsnotify` 是主要抽象层，优先改它。

### 2.2 `xai-fsnotify` 的内部结构

```
src/
├── source.rs       FsEventSource（对外入口：start / start_on）
├── watcher.rs      watcher 线程：建 debouncer、arm watch、服务命令直到关闭
├── install.rs      arm/prune 子树 watch、顶层 watch 协调（ARM_SYNC_MAX 分块）
├── selection.rs    watch 目录选择（MAX_TOP_LEVEL_FANOUT 上限、按目录扇出）
├── merge.rs        原始事件合并（RawFsEvent → merge_events）
├── event.rs        FsEvent / FsEventKind / GitMetaKind（平台无关的事件模型）
├── registry.rs     统计指标（FsWatcherStats）
├── checkout.rs     watch 根覆盖判断
├── state.rs        SETTLE_MS 等常量
└── vcs.rs / paths.rs / handle.rs / error.rs
```

**关键观察**：事件模型（`event.rs`）与 watch 策略（`selection.rs`/`merge.rs`/`install.rs`）**都是平台无关的**，只有 `watcher.rs` + `install.rs` 里对 `notify` 的调用是平台相关的。**这是理想的接入点结构** —— 替换掉 `notify` 的使用即可，上游消费者（`xai-grok-shell/src/session/fs_watch.rs`）通过 `FsEventSource`/`FsEvent` 通信，无需改动。

### 2.3 语义要求（替换实现必须保持）

从代码结构可以读出这些必须保持的语义：

| 语义 | 来源 | 说明 |
|------|------|------|
| **去抖/静默期** | `SETTLE_MS`（`state.rs`） | 事件突发后等待稳定再投递 |
| **批量合并** | `merge.rs` | 同一路径的多次变更合并为一个事件 |
| **递归 watch** | `install.rs` + `RecursiveMode` | 子树 watch 的 arm/prune |
| **watch 上限** | `MAX_TOP_LEVEL_FANOUT`（`selection.rs`） | 顶层 watch 数量受控（fd 预算） |
| **事件过滤** | `event.rs` | `Access`/`Any`/`Other` 等 kind 被丢弃 |
| **工作区边界** | `checkout.rs` / `paths.rs` | 超出边界的路径不算 |
| **VCS 元数据区分** | `GitMetaKind` | git 内部文件变更单独分类 |

> 轮询实现**必须**保留去抖与合并语义，否则上层会出现事件风暴（例如编译产物目录的变化被逐文件上报）。

---

## 3. QNX 侧机制

| 机制 | 可用性 | 说明 |
|------|--------|------|
| `inotify` | **[待实测]** 未知 | QNX 8.0 为 Linux 兼容可能提供；先探 |
| `ionotify()` + `sigevent` | ✅ QNX 特有 | 面向 fd/资源管理器，语义与 inotify 不同，需重新 arm |
| 轮询（`stat` + 目录遍历） | ✅ 一定可用 | **兜底方案** |
| `FsysNotify` / pathmgr 事件 | **[待实测]** | QNX 传统机制，文档较少 |

**建议探测顺序**：`inotify` 兼容层 → `ionotify` → **轮询**。

---

## 4. 轮询方案设计（兜底）

### 4.1 基本形态

替换 `watcher.rs` 内部的 `notify` 使用，保留其命令循环与对外事件流：

```
watch 集合（复用 selection.rs / install.rs 的目录选择逻辑）
  ↓ 每 T 毫秒
遍历受 watch 的目录
  ↓ 对每个目录项取 (path, mtime_ns, size, inode?)
与上一轮快照 diff
  ↓ 新增/删除/修改 → RawFsEvent
merge_events（复用 merge.rs）
  ↓ SETTLE_MS 静默期
投递 FsEvent（复用 event.rs）
```

### 4.2 关键设计点

| 点 | 建议 |
|----|------|
| **轮询间隔** | 200ms–1s；工作区大时取 1s。作为配置项暴露 |
| **快照结构** | `HashMap<PathBuf, (mtime_ns, size, inode, mode)>`，避免读文件内容 |
| **目录遍历成本** | 跳过 `.git/objects`、`target/`、`node_modules/` 等（**复用现有的 gitignore 过滤**） |
| **递归深度** | 复用 `selection.rs` 的受限扇出策略，不要全树递归 |
| **首轮** | 只建快照不报事件（避免启动时事件风暴） |
| **去抖** | 保留 `SETTLE_MS`；轮询本身已聚合一轮的变化，去抖可适当缩短 |
| **CPU 占用** | 大仓库是主要成本；建议懒开启（只在真正需要监听的路径上轮询） |
| **原子替换检测** | 编辑器保存常用"写临时文件 + rename" → 需要同时监控目录项变化，不能只 stat 已知文件 |

### 4.3 何时可以接受

| 消费者 | 容忍延迟 | 结论 |
|--------|---------|------|
| 记忆索引同步（`xai-grok-memory`） | 秒级 | ✅ 完全可接受 |
| 配置热重载（`xai-grok-config`） | 亚秒–秒级 | ✅ 可接受 |
| Skills 发现 / 项目指令 | 秒级 | ✅ 可接受 |
| 工作区变更感知（`fs_watch.rs`） | 亚秒级理想 | ⚠️ 可接受，但体验下降 |

---

## 5. `ionotify` 方案（进阶）

若需要毫秒级延迟，用 QNX 原生机制：

- `ionotify(fd, action, flags, &sigevent)` 注册事件（`POLLIN` 等），通过 `sigevent` 投递（`SIGEV_SIGNAL` / `SIGEV_PULSE`）
- **边缘触发语义**：需要在每次投递后重新 arm
- 与 mio 的整合：`SIGEV_PULSE` 投到 channel → 用 `MsgReceive` 或信号的 waker 接入（与 mio QNX 后端共享事件循环）
- **难点**：`ionotify` 面向的是**单个 fd/资源管理器**，而 `notify` 的语义是**递归目录树**。要做目录树语义，需要自己维护"目录 → fd"映射并处理新增子目录的 arm 传播 —— 这正是 `install.rs` 已经做的事（可复用其逻辑骨架）

---

## 6. 改动清单

| 文件 / crate | 改动 |
|-------------|------|
| `xai-fsnotify/src/watcher.rs` | 替换 `notify_debouncer_full` 为 QNX 后端（轮询或 ionotify） |
| `xai-fsnotify/src/install.rs` | 保留；去掉对 `notify::RecursiveMode` 的直接依赖 |
| `xai-fsnotify/src/event.rs` / `merge.rs` / `selection.rs` / `checkout.rs` | **不改**（平台无关） |
| `xai-fsnotify/Cargo.toml` | `notify*` 依赖改平台门控 |
| `xai-grok-memory/Cargo.toml` + 监听代码 | 若直接用 `notify`，改走 `xai-fsnotify` 或独立轮询 |
| `xai-grok-shell/Cargo.toml` + 监听代码 | 同上（注意还用了 `notify-debouncer-mini`） |
| `xai-grok-pager-render`（optional） | 确认该 feature 在 QNX 构建里是否启用；不需要则关闭 |

> **架构建议**：把"文件事件源"提升为 `xai-platform-qnx` 的一个 trait（见 [`porting-qnx.md`](./porting-qnx.md) §6.1），让 4 个消费者统一走它，避免 4 份轮询实现。

---

## 7. 探针计划

```rust
// F1: notify 能否在 QNX 编译（先试，最快）
//     cargo build -p xai-fsnotify --target aarch64-unknown-nto-qnx800

// F2: QNX 是否有 inotify 兼容
//     ls /usr/include/sys/inotify.h  或  查 inotify_init 符号

// F3: ionotify 基础行为
//     对某 fd 注册 POLLIN，写入后验证 sigevent 投递

// F4: 轮询原型 —— 在真实仓库上测 CPU 与延迟
//     1000 目录 / 10 万文件规模下，200ms 间隔的 CPU 占用

// F5: 语义回归 —— 替换后端后跑 xai-fsnotify 现有测试
//     验证去抖 / 合并 / 边界 / VCS 元数据分类不变
```

**F5 是验收关键**：`xai-fsnotify` 自带的测试就是替换后端的回归基线。

---

## 8. 工作量估算

| 项 | 时间 |
|----|------|
| F1–F2 探测（notify 编译 / inotify 可用性） | 1–2 天 |
| 轮询后端实现（复用 merge/selection/settle 逻辑） | 1–1.5 周 |
| 4 处消费者接线统一 | 3–5 天 |
| 语义回归（对齐 `xai-fsnotify` 测试） | 3–5 天 |
| 大仓库性能调优（可选） | 3–5 天 |
| `ionotify` 后端（可选进阶） | 1–2 周 |
| **合计（轮询方案）** | **2–3 周** |

---

## 9. 待核对 / 待实测

| # | 项 | 方式 |
|---|----|------|
| 1 | `notify 8` 在 QNX 能否编译（其结果决定是否必须替换） **[待实测]** | 直接试编 |
| 2 | QNX 8.0 是否提供 `inotify` 兼容 **[待实测]** | 查头文件与符号 |
| 3 | `ionotify` 对**目录** fd 的语义 **[待实测]** | 查 QNX 文档 |
| 4 | `xai-grok-memory` / `xai-grok-shell` 是否可直接改用 `xai-fsnotify` | 读代码 |
| 5 | `xai-grok-pager-render` 的 `notify` optional feature 在 QNX 是否启用 | 读 feature 依赖树 |
| 6 | QNX 上大目录 `stat` 遍历性能 **[待实测]** | F4 原型 |

---

*文档生成时间：2026-09-09（基于 `upstream/main` @ `37949780`）*
