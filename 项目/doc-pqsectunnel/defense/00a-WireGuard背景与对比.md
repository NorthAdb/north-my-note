# 00a — WireGuard 背景与 PQ-SecTunnel 对比

> 状态：正文 | 权威源：WireGuard 官方文档 + 白皮书 + 本仓库源码  
> 建议：**先读本章再读 [01](01-项目定位与改造动机.md)** | 调研笔记：[../research/wireguard-background.md](wireguard-background.md)

---

## 1. 本章要回答什么

答辩评委常问：「你们和 WireGuard 什么关系？改了什么、保留了什么？」  
本章从 **WireGuard 原版** 讲起，再对照 **PQ-SecTunnel**，避免只讲改造点而说不清基线。

---

## 2. WireGuard 是什么

**WireGuard** 是 Jason Donenfeld 设计的 **Layer 3 VPN**：Linux 内核虚拟网卡（`wg0`）+ **UDP** 承载加密 IP 包，目标是用更简单、可审计的实现替代 IPsec / OpenVPN 等方案。


| 维度  | 要点                                                      |
| --- | ------------------------------------------------------- |
| 层次  | 仅 **L3**（IP in IP），不做 L2 桥接                             |
| 实现  | 内核模块 + 用户态 `wg` / `wg-quick`                            |
| 管理  | 配置静态密钥与 peer 后，**握手/重连/漫游自动完成**                         |
| 哲学  | **密码学不协商**——固定 Curve25519 + ChaCha20-Poly1305 + BLAKE2s |


来源：[WireGuard 白皮书](https://www.wireguard.com/papers/wireguard.pdf) §1；[Protocol 页](https://www.wireguard.com/protocol/)。

---



## 3. 核心设计：Cryptokey Routing

WireGuard 最根本的规则（白皮书 §2）：

> **Peer = 静态公钥**；每个 peer 绑定一组 **AllowedIPs**（允许使用的源 IP）。

```
出站：目的 IP → 查表 → 选 peer 公钥 → 加密发送
入站：解密 → 源 IP 必须匹配该 peer 的 AllowedIPs → 否则丢弃
```

设备上有一张 **AllowedIPs 路由表**（实现上是前缀 Trie）：


| **表里存什么**              | **含义**    |
| ---------------------- | --------- |
| IP 前缀（如 `10.0.8.2/32`） | 内层地址范围    |
| → 指向某个 **peer**        | 该范围归哪个公钥管 |


配置里每个 peer 的 `AllowedIPs` 合在一起，就是这张表。

代码里：`wg->peer_allowedips`（`allowedips.c`）。

---

## **谁负责查表**

**内核**（`pqsectunnel.ko` / WireGuard），用户态只负责往表里**填**，不负责运行时查。


| **时机** | **谁查**          | **查什么**               | **API**                    |
| ------ | --------------- | --------------------- | -------------------------- |
| **出站** | `device.c` 发包路径 | 内层**目的 IP** → peer    | `wg_allowedips_lookup_dst` |
| **入站** | `receive.c` 解密后 | 内层**源 IP** 是否属于该 peer | `wg_allowedips_lookup_src` |


**填表**：用户态 `pqst set` / Netlink 配 `AllowedIPs` → 内核 `wg_allowedips_insert_`*。

效果：

- 防火墙可把 `wg0` 上某源 IP 视为**密码学认证过的身份**
- 公钥即身份，无需 x509 用户名（密钥分发走 SSH 式 out-of-band）

**PQ-SecTunnel 保留**：`.conf` 里仍是 `PublicKey` + `AllowedIPs` + 可选 `Endpoint`；只是公钥从 32B Curve25519 变为 **ML-KEM 静态公钥（800B）**。

---



## 4. 功能与用法（原版）



### 4.1 典型部署步骤

```bash
ip link add dev wg0 type wireguard
ip address add dev wg0 10.0.0.1/24 dev wg0
wg setconf wg0 peer.conf    # 或 wg set / wg-quick up
ip link set wg0 up
ping 10.0.0.2
```



### 4.2 配置里有什么


| 字段                         | 作用                             |
| -------------------------- | ------------------------------ |
| `PrivateKey` / `PublicKey` | 长期 Curve25519 身份（Base64 44 字符） |
| `ListenPort`               | UDP 监听（默认常 51820）              |
| `Peer.PublicKey`           | 对端身份                           |
| `AllowedIPs`               | 路由 + 入站源 IP 校验                 |
| `Endpoint`                 | 可选初始 `IP:Port`                 |
| `PresharedKey`             | 可选 PSK，混入 Noise 链              |




### 4.3 漫游（Roaming）

未固定 Endpoint 时：收到**合法**加密包 → 用外层 UDP 源地址更新 peer endpoint。  
手机换网、NAT 映射变化时无需改配置。

**PQ 保留**：endpoint 学习与定时器逻辑仍在 `receive.c` / `timers.c`。

### 4.4 「无状态」用户体验

WireGuard 内核维护定时器（重握手、Keepalive、重密钥、超时丢弃），管理员**不用手动 reconnect**。  
白皮书 §6 + protocol 页详述 `REKEY_TIMEOUT`、`KEEPALIVE` 等。

**PQ 保留**：`messages.h` 中同名常量与状态机框架。

---



## 5. WireGuard 的 Noise 是怎么设计的



### 5.1 选用的模式：Noise_IKpsk2

```
Noise_IKpsk2_25519_ChaChaPoly_BLAKE2s
WireGuard v1 zx2c4 Jason@zx2c4.com
```


| 片段      | 含义                                      |
| ------- | --------------------------------------- |
| **IK**  | Initiator **预先知道** Responder 静态公钥（配置文件） |
| **psk** | 可选 32B 预共享密钥混入链（`PresharedKey`）         |
| **2**   | Noise 内部版本号                             |


来源：[protocol 页](https://www.wireguard.com/protocol/) CONSTRUCTION / IDENTIFIER。

### 5.2 两次握手，1-RTT

```
Initiator                          Responder
    │──── Handshake Initiation ────►│
    │◄─── Handshake Response ───────│
    │──── Transport Data (可空) ───►│  ← 隐式密钥确认
    │◄─── Transport Data ───────────│
```


| 报文             | 大小约    | 核心字段                                                     |
| -------------- | ------ | -------------------------------------------------------- |
| **Initiation** | ~148 B | 明文 ephemeral 32B；AEAD(static)；AEAD(TAI64N 时间戳)；mac1/mac2 |
| **Response**   | ~92 B  | 明文 ephemeral 32B；AEAD(空)；mac1/mac2                       |


**密码学操作**（ECDH）：

- Initiation：`DH(e_i, S_r)` + 加密暴露 `S_i` 再 `DH(S_i, S_r)`
- Response：`DH(e_r, e_i)` + `DH(e_r, S_i)` + **mix_psk**



### 5.3 密钥确认：WireGuard 的做法（重要）

WireGuard **没有第三条握手报文**。

官方说明（[protocol 页 Connection-less Protocol](https://www.wireguard.com/protocol/)）：

> Response 之后 Initiator 可发加密数据；**Responder 必须等到收到 Initiator 的第一条加密数据**才启用新会话；若无业务数据，Initiator 应发**空数据包**作确认。

这是 **用数据面第一包作隐式 Key Confirmation**。

### 5.4 数据面

握手后派生 `sending_key` / `receiving_key`，Transport 报文（type=4）：

```
type | reserved | receiver_index | counter | AEAD(内层 IP 包)
```

ChaCha20-Poly1305，64-bit counter，滑动窗口防重放。

### 5.5 DoS：MAC1 / MAC2 / Cookie


| 机制               | 作用                                                  |
| ---------------- | --------------------------------------------------- |
| **MAC1**         | `BLAKE2s("mac1----" ‖ 接收方静态公钥)` 认证握手主体；**不进 C/H 链** |
| **MAC2**         | 负载高时要求 Cookie MAC，证明 IP 所有权                         |
| **Cookie Reply** | type=3，加密回传 cookie                                  |
| **静默**           | mac1 无效 → **不回复、不建状态**（Silence is a Virtue）         |


**MAC1 ≠ PSK**——PSK 只进 Noise mix_psk；与 PQ 相同。

### 5.6 可选 PSK 与「后量子」

WireGuard 白皮书 §5.2：PSK 是为「录流量 + 将来破解 Curve25519」的**经典混合**方案，**不是**完整后量子替换。

PQ-SecTunnel 则把 **静态身份本身**换成 ML-KEM，PSK 仍可选混入（默认由双方 static 公钥异或派生，见 KE.md）。

---



## 6. WireGuard 报文类型（原版）


| type | 名称                   |
| ---- | -------------------- |
| 1    | Handshake Initiation |
| 2    | Handshake Response   |
| 3    | Cookie Reply         |
| 4    | Transport Data       |


---



## 7. PQ-SecTunnel 改了什么



### 7.1 一句话

> 保留 WireGuard 的 **VPN 形态**（TUN、Cryptokey Routing、UDP、Cookie、定时器、漫游）；把 **Noise_IKpsk2 + X25519** 换成 **Noise_IKpsk1kem + 双 ML-KEM**，并增加 **显式 Key Confirmation** 第三轮。



### 7.2 对照总表


| 维度            | WireGuard                               | PQ-SecTunnel（主套件）                          |
| ------------- | --------------------------------------- | ------------------------------------------ |
| Noise 名       | `Noise_IKpsk2_25519_ChaChaPoly_BLAKE2s` | `Noise_IKpsk1kem_SM4GCM_SM3_EphemMLKEM768` |
| 握手报文数         | **2**                                   | **3**（+ Key Confirmation）                  |
| RTT           | **1-RTT**                               | **1.5-RTT**                                |
| 密钥确认          | 第一条**数据包**（可空）                          | 独立 **Key Confirm** 报文                      |
| 长期身份          | Curve25519 32B                          | **Static ML-KEM-512** 800B                 |
| 临时密钥          | Curve25519 32B                          | **Ephemeral ML-KEM-768** 1184B             |
| 共享秘密          | ECDH × 多次                               | **KEM ct₁/ct₂/ct₃** 混链                     |
| Hash/AEAD     | BLAKE2s / ChaCha20-Poly1305             | **SM3 / SM4-GCM**                          |
| Initiation 大小 | ~148 B                                  | **2068 B**                                 |
| Response 大小   | ~92 B                                   | **1916 B**                                 |
| MTU           | 通常单包                                    | **应用层分片**（2 片）                             |
| 用户工具          | `wg` / `wg-quick`                       | `pqst` **/** `pqst-quick`                  |
| 模块名           | `wireguard.ko`                          | `pqsectunnel.ko`                           |
| 报文 type       | Cookie=3, Data=4                        | **KeyConfirm=3, Cookie=4, Data=5**         |




### 7.3 三次握手（PQ）

```
Initiator                          Responder
    │──── Initiation (2068B) ──────►│
    │◄─── Response (1916B) ─────────│
    │──── Key Confirmation (60B) ──►│  ← 显式确认，用 TK 加密
    │◄─── Transport Data ───────────│
```

Initiator 验 Response 密码学成功即知 Responder 完成；Responder 需 **Key Confirm** 才 `begin_session`（见 [02](02-协议总览.md)）。

### 7.4 Initiation 结构变化（概念）


| WireGuard Initiation              | PQ Initiation               |
| --------------------------------- | --------------------------- |
| `unencrypted_ephemeral[32]`       | `E_i^pub`（1184B，ML-KEM-768） |
| （无）                               | `ct₁` 对 S_r 封装（768B）        |
| `encrypted_static` = AEAD(32B 公钥) | AEAD(**Hash(S_i^pub)**)     |
| `encrypted_timestamp`             | 保留 TAI64N 语义                |


Response：WireGuard 仅 `encrypted_nothing`；PQ 为 **ct₂ + ct₃ + empty**（见 [03](03-握手协议逐步详解.md)）。

---



## 8. 保留了哪些 WireGuard 功能


| 功能                                | 说明                           |
| --------------------------------- | ---------------------------- |
| **TUN 虚拟接口**                      | 仍是 L3 VPN，非用户态 OpenVPN 式     |
| **Cryptokey Routing**             | PublicKey ↔ AllowedIPs 语义不变  |
| **UDP 传输**                        | 51820 等端口模型不变                |
| **Roaming**                       | 从合法包学习 endpoint              |
| **MAC1/MAC2/Cookie**              | DoS 框架保留；label 仍为 `mac1----` |
| **TAI64N 时间戳**                    | Initiation 防重放               |
| **sender_index / receiver_index** | index 表                      |
| **定时器**                           | REKEY、KEEPALIVE、HANDSHAKE 重试 |
| **PSK mix**                       | 可选；PQ 默认派生规则见 KE.md          |
| **数据面**                           | counter + AEAD 封装 IP         |
| **Netlink 配置**                    | 结构同构，改名 PQST                 |


**管理员视角**：仍是「生成密钥 → 写 conf → quick up → ping 通」，实验见 [07](07-工程架构与实验验证.md)。

---



## 9. 主要不同（答辩必讲三点）



### ① 密码学内核换血

经典 ECDH → **后量子 KEM**；国际算法 → 答辩套件 **SM3/SM4-GCM**。  
编译时固定，**无运行时协商**（延续 WireGuard「不 agility」哲学，但套件由 build.sh 选）。

### ② 握手从 2 到 3

不仅多一轮 RTT，更是 **隐式确认 → 显式 Key Confirmation**。  
KEM 握手下 Responder 更需要 Initiator 证明「我也算出了 TK」。

### ③ 工程后果：大包 + 分片

公钥/密文从几十字节到 **KB 级** → Initiation/Response 超 MTU → `pqc_fragment` **应用层分片**（MAC 后切）。WireGuard 无此需求。

---



## 10. 兼容性说明


| 问题                                 | 答案                            |
| ---------------------------------- | ----------------------------- |
| pqsectunnel 能否与官方 wireguard.ko 互通？ | **不能**——报文格式、type 编号、密码原语均不同  |
| 能否用 `wg` 命令配 PQ 接口？                | 实验用 `pqst`（wg 分支重命名）          |
| 配置语法像吗？                            | **像** WireGuard ini；字段长度与算法不同 |


---



## 11. 答辩 1 分钟版

> WireGuard 是 Layer3 内核 VPN，核心是 Cryptokey Routing：公钥绑定 AllowedIPs，Noise IKpsk2 两次握手、1-RTT，Curve25519 和 ChaCha20。我们 PQ-SecTunnel fork 其模块架构，保留 TUN、UDP、Cookie、漫游和配置模型，把密钥交换换成双 ML-KEM，握手加到 Initiation、Response 和 Key Confirmation 三轮，算法换成 SM3 和 SM4-GCM。握手包从约 150 字节涨到 2000 字节，所以加了应用层分片。功能上还是 VPN，密码内核和报文格式是后量子版。

---



## 12. 交叉引用


| 主题          | 文档                                                                         |
| ----------- | -------------------------------------------------------------------------- |
| PQ 改造动机（简版） | [01](01-项目定位与改造动机.md)                                                      |
| 1.5-RTT 时序  | [02](02-协议总览.md)                                                           |
| 逐步握手        | [03](03-握手协议逐步详解.md)                                                       |
| 分片          | [06](06-分片与MTU.md)                                                         |
| 调研笔记（带 URL） | [../research/wireguard-background.md](wireguard-background.md) |




## 13. 外部链接

- [WireGuard Protocol & Cryptography](https://www.wireguard.com/protocol/)
- [WireGuard Whitepaper (PDF)](https://www.wireguard.com/papers/wireguard.pdf)
- [Noise Protocol Framework](https://noiseprotocol.org/noise.html)

