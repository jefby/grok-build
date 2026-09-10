# QNX 移植深入：TLS 与加密层分析

> 配套文档：[`porting-qnx.md`](./porting-qnx.md)（总览）
> 分析基准：`rustls 0.23`、`ring 0.17`、`aws-lc-rs`、`rustls-native-certs 0.8`、`webpki-roots 0.26`、`rcgen 0.13`、`rsa 0.9`、`sha2 0.10 (force-soft)`、`hmac 0.12`、`jsonwebtoken 10 (rust_crypto)`
> 标 **[待实测]** 处需在真实 QNX SDP 8.0 验证。

---

## 1. 结论

**推荐方案：纯 Rust crypto provider + `ed25519-dalek`**，彻底消除 C/汇编依赖。

| 方案 | 说明 | 评价 |
|------|------|------|
| **A. 纯 Rust provider（推荐）** | rustls + RustCrypto（`aes-gcm`/`chacha20poly1305`/`sha2`/`p256`/`ed25519-dalek`）；社区已有 `rustls-rustcrypto` 可复用 | 零 C 依赖，交叉编译友好 |
| B. QNX OpenSSL | QNX SDP 若带 OpenSSL，用 `openssl` crate | 省事但引入 C/ABI 绑定 **[待实测]** |
| C. 移植 `ring` / `aws-lc-rs` | C + 汇编 + cmake | **不推荐**（高风险，收益低） |

**关键背景**：项目**同时编译了两套 provider**（`ring` 与 `aws-lc-rs`），当前靠"first install wins"兜住 —— 这是移植时必须先拆掉的隐含耦合。

---

## 2. 现状清点（实测矩阵）

| 用途 | 依赖 | 位置 | 纯 Rust 可替代 |
|------|------|------|---------------|
| **TLS provider 安装** | `aws-lc-rs` | `xai-grok-extra-ca/src/lib.rs:24` `ensure_default_crypto_provider()` | ✅ 改为可配置 |
| **HTTP 客户端 TLS** | `rustls`(aws-lc-rs) + `reqwest` | `xai-grok-http/src/lib.rs:654` | ✅ |
| **WebSocket TLS** | `tokio-tungstenite`(`rustls-tls-native-roots`) + `tokio-rustls`(**ring**) + `rustls`(**ring**) | `xai-grok-shell/Cargo.toml:88–98` | ✅ |
| **gRPC/OTLP TLS** | `opentelemetry-otlp`(`tls-aws-lc`) + `tonic` | `xai-grok-telemetry/Cargo.toml:81` | ⚠️ 需换 feature |
| **Ed25519 验签（生产）** | **`ring`** | `xai-grok-config/src/signed_policy.rs:271` | ✅ `ed25519-dalek` |
| **证书生成** | `rcgen 0.13`(pem) | —— | ✅ 跟随 provider |
| **系统根证书** | `rustls-native-certs 0.8` | `xai-grok-extra-ca`（**唯一接入点**） | ✅ `webpki-roots` 或 QNX 路径 |
| **内置根证书** | `webpki-roots 0.26` | 纯数据 | ✅ 无平台依赖 |
| **JWT（登录）** | `jsonwebtoken 10`(**`rust_crypto`**) | `xai-grok-login` | ✅ **已是纯 Rust** |
| 哈希 | `sha2 0.10` (**`force-soft`**) | —— | ✅ 已是纯软件实现 |
| HMAC | `hmac 0.12` | —— | ✅ 纯 Rust |
| RSA | `rsa 0.9` | —— | ✅ RustCrypto 纯 Rust |

**三个重要观察**：

1. **`sha2` 已启用 `force-soft`** —— 团队已在主动规避 SIMD/汇编，说明可移植性有基础。
2. **`jsonwebtoken` 已切 `rust_crypto`** —— 登录链路已是纯 Rust，无需动。
3. **`rustls-native-certs` 只有一个接入点**（`extra-ca`）—— 改造面很小。

### 2.1 双 provider 并存的隐含耦合

`xai-grok-extra-ca/src/lib.rs:20` 的注释直说了这个问题：

> *First install wins; without a default, `ClientConfig::builder()` panics when `ring` and `aws-lc-rs` are both compiled in.*

```rust
pub fn ensure_default_crypto_provider() {
    static ONCE: std::sync::Once = std::sync::Once::new();
    ONCE.call_once(|| {
        if rustls::crypto::aws_lc_rs::default_provider().install_default().is_err() {
            // 已被别的 provider 抢先安装：检查 P-521 支持并告警
        }
    });
}
```

端口到纯 Rust provider 时，这里是**第一个要改的地方** —— 并保留 P-521 支持检查（企业代理证书）。

---

## 3. 需要的算法覆盖

改造 provider 时必须覆盖这些（否则 TLS 握手在特定服务器上失败）：

| 类别 | 算法 | RustCrypto crate |
|------|------|-----------------|
| 对称加密 | AES-128-GCM、AES-256-GCM | `aes-gcm` |
| 对称加密 | ChaCha20-Poly1305 | `chacha20poly1305` |
| 哈希 | SHA-256、SHA-384、SHA-512 | `sha2` |
| 密钥交换 | X25519 | `x25519-dalek` |
| 密钥交换 | P-256 ECDHE | `p256` |
| 签名验证 | ECDSA P-256 / P-384 | `p256` / `p384` |
| 签名验证 | **ECDSA P-521** | `p521`（较新，**需确认**） |
| 签名验证 | Ed25519（TLS + 策略验签） | `ed25519-dalek` |
| 签名验证 | RSA-PSS / PKCS#1 v1.5 | `rsa` |
| 随机数 | OS entropy | `rand` + **QNX `/dev/urandom`** **[待实测]** |

> **P-521 与 RSA 是容易被忽略的两项**：`extra-ca` 的告警逻辑专门提到 P-521（企业代理），cert 链里也常见 RSA。

---

## 4. 方案 A 实施路径（推荐）

### 4.1 用 `rustls-rustcrypto`（优先）

社区已有 **`rustls-rustcrypto`** 项目，提供基于 RustCrypto 的 rustls `CryptoProvider`。优先评估它，避免从零实现 `CryptoProvider`（`cipher_suites` / `kx_groups` / `signature_verification_algorithms` / `secure_random` / `key_provider` 五部分）。

评估要点：

- 是否覆盖上表全部算法（尤其 **P-521**）
- 与 `rustls 0.23` 的版本对齐
- 与 `rcgen` 的签名路径兼容

### 4.2 若需要自实现 `CryptoProvider`

```rust
// 伪代码：provider 装配点
let provider = rustls::crypto::CryptoProvider {
    cipher_suites: vec![ /* TLS13_AES_256_GCM_SHA384, TLS13_CHACHA20..., TLS12_... */ ],
    kx_groups: vec![ /* X25519, SECP256R1 */ ],
    signature_verification_algorithms: WebPkiSupportedAlgorithms { /* ECDSA, Ed25519, RSA */ },
    secure_random: &/* RustCrypto 的 OsRng */,
    key_provider: &/* 对应 provider */,
};
rustls::crypto::CryptoProvider::install_default(provider)?;
```

### 4.3 替换 Ed25519 验签

```rust
// 现状：xai-grok-config/src/signed_policy.rs:271
ring::signature::UnparsedPublicKey::new(&ring::signature::ED25519, public_key)
    .verify(payload, signature)

// 改为 ed25519-dalek（纯 Rust）：
use ed25519_dalek::{Signature, VerifyingKey, Verifier};
let vk = VerifyingKey::from_bytes(&public_key)?;
vk.verify(payload, &Signature::from_slice(signature)?)
```

注意 `signed_policy` 的语义要求：**`key_id` 选择必须限定在受信集合内、且必须在验签前完成**（见源码注释）。替换实现时保持这个顺序不变。

### 4.4 根证书来源

- `rustls-native-certs` 读 Linux 系统证书库 → QNX 路径不同
- 改为：**`webpki-roots`（内置 Mozilla 根）** 为主，`GROK_EXTRA_CA_BUNDLE` / `SSL_CERT_FILE` 额外根保留（这两个逻辑在 `extra-ca` 中已实现，是纯文件读取，可移植）

### 4.5 gRPC/OTLP 的 TLS feature

`xai-grok-telemetry` 用 `opentelemetry-otlp` 的 `tls-aws-lc`。选项：

1. 换 `tls-ring`（同样要移植 ring —— 不解决根问题）
2. 换 native-tls（依赖系统 OpenSSL —— 见方案 B）
3. 若 OTLP 导出非必需 → **在 QNX 构建中关闭该 feature**（最省事）

---

## 5. 方案 B：QNX OpenSSL（备选）

- QNX SDP 8.0 是否附带 OpenSSL **[待实测]**
- 若附带：`openssl` crate + QNX 的 `libssl`/`libcrypto`，可绕开纯 Rust provider 的实现成本
- 代价：C 依赖、版本/ABI 绑定、与 `rustls` 的 feature 需重新梳理（`tonic`/`reqwest` 都要切 native-tls 后端）

---

## 6. 改动清单

| 文件 / 依赖 | 改动 |
|------------|------|
| `xai-grok-extra-ca/src/lib.rs` | `ensure_default_crypto_provider()` 改为可配置 provider；`rustls-native-certs` → `webpki-roots`；保留 P-521 检查 |
| `xai-grok-http/src/lib.rs:654` | 去掉硬编码 `aws_lc_rs::default_provider()` |
| `xai-grok-shell/Cargo.toml:88–98` | `tokio-rustls`/`rustls` 的 `ring` feature → 纯 Rust provider |
| `xai-grok-config/src/signed_policy.rs:271` | `ring::signature::ED25519` → `ed25519-dalek` |
| `xai-grok-config/Cargo.toml` | `ring` → `ed25519-dalek` |
| `xai-grok-telemetry/Cargo.toml:81` | `tls-aws-lc` → 关闭或换后端 |
| `Cargo.toml` | `rustls` 的 `aws-lc-rs` feature → 纯 Rust provider feature |
| 全局 | 确认 `ring`/`aws-lc-rs` 不再进入依赖树（`cargo tree`） |

---

## 7. 探针计划

```rust
// C1: 纯 Rust provider 能否构建（最小）
let provider = my_rustcrypto_provider();
rustls::crypto::CryptoProvider::install_default(provider)?;
let _cfg = rustls::ClientConfig::builder().with_root_certificates(roots).with_no_client_auth();
// 不 panic 即 provider 装配成功

// C2: 真实 HTTPS 握手（覆盖 AES-GCM / ChaCha20 / ECDHE）
//     GET https://example.com

// C3: TLS 1.2 与 1.3 各测一次
//     （注意 shell 的 rustls feature 里显式开了 "tls12"）

// C4: Ed25519 验签（signed_policy 回归测试）
//     cargo test -p xai-grok-config signed_policy

// C5: 企业代理场景 —— P-521 / RSA 证书链
//     若失败，检查 provider 的 signature_verification_algorithms

// C6: OS entropy —— QNX /dev/urandom 可用性
//     读取 32 字节随机数并校验
```

**C6 容易被忽略但很关键**：`ring`/RustCrypto 的 `OsRng` 需要 OS 熵源；QNX 的 `/dev/urandom` 可用性 **[待实测]**。

---

## 8. 工作量估算

| 项 | 时间 |
|----|------|
| C1–C3 provider 装配与握手验证 | 3–5 天 |
| `rustls-rustcrypto` 评估/接入（或自实现 provider） | 1–2 周 |
| Ed25519 替换（`signed_policy`） | 2–3 天 |
| 根证书策略（`webpki-roots` + QNX 路径） | 1–2 天 |
| OTLP/gRPC TLS 处置 | 2–3 天 |
| 依赖树清理（剔除 ring/aws-lc） | 2–3 天 |
| 回归（策略验签、企业代理、relay WS） | 3–5 天 |
| **合计** | **3–5 周** |

---

## 9. 待核对 / 待实测

| # | 项 | 方式 |
|---|----|------|
| 1 | `rustls-rustcrypto` 是否覆盖 P-521 与 RSA **[待实测]** | 读其文档/源码 |
| 2 | QNX `/dev/urandom` 可用性 **[待实测]** | `dd if=/dev/urandom bs=32 count=1` |
| 3 | QNX SDP 是否带 OpenSSL **[待实测]** | 查 SDP 目录 |
| 4 | `ring` 是否真的还在依赖树中（除 config/shell） | `cargo tree -i ring` **[核对]** |
| 5 | `opentelemetry-otlp` 关闭 TLS feature 后 OTLP 是否仍可用（明文/无导出） | 读 telemetry 代码 |
| 6 | `rcgen 0.13` 与所选 provider 的签名路径兼容性 **[待实测]** | 试编 + 测试 |
| 7 | 企业代理证书链是否含 P-521（决定 provider 必须支持它） | 与部署环境确认 |

---

*文档生成时间：2026-09-09（基于 `upstream/main` @ `37949780`）*
