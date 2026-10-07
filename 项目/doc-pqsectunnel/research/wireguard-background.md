# WireGuard 背景调研

> 本文档为 PQ-SecTunnel 答辩材料撰写，内容均来自一手资料。  
> 主要来源：WireGuard 官网、技术白皮书、Noise Protocol Framework 规范。

---

## 来源索引


| 来源                       | URL / 文档                                                                                                                                 |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------- |
| WireGuard 官网概览           | [https://www.wireguard.com/](https://www.wireguard.com/)                                                                                 |
| WireGuard 协议与密码学页面       | [https://www.wireguard.com/protocol/](https://www.wireguard.com/protocol/)                                                               |
| WireGuard 快速入门           | [https://www.wireguard.com/quickstart/](https://www.wireguard.com/quickstart/)                                                           |
| WireGuard 技术白皮书          | [https://www.wireguard.com/papers/wireguard.pdf（Donenfeld](https://www.wireguard.com/papers/wireguard.pdf（Donenfeld), 2020；NDSS 2017 版本） |
| Noise Protocol Framework | [https://noiseprotocol.org/noise.html（Perrin](https://noiseprotocol.org/noise.html（Perrin), Revision 34, 2018-07-11）                     |


---



## Overview（概览）

WireGuard 是一种面向第 3 层（IP）的安全网络隧道协议，通过 UDP 封装加密 IP 数据包。其设计目标是替代 IPsec 与 OpenVPN 等方案，在保持强安全性的同时实现更简单、更高性能、更易部署的 VPN。（来源：[WireGuard 白皮书 §1](https://www.wireguard.com/papers/wireguard.pdf)；[官网](https://www.wireguard.com/)）

核心特征：

- **无连接协议（Connection-less）**：握手基于定时器驱动，而非依赖先前数据包内容；接口启动后，密钥交换、重连、漫游等由协议自动处理，管理员视角接近"无状态"。（来源：[协议页面 — Connection-less Protocol](https://www.wireguard.com/protocol/)）
- **1-RTT 握手**：一次往返（initiator → responder → initiator）即可完成密钥协商。（来源：[白皮书 §5](https://www.wireguard.com/papers/wireguard.pdf)）
- **Cryptokey Routing**：公钥与允许隧道 IP 地址直接绑定，身份与路由合一。（来源：[官网 — Cryptokey Routing](https://www.wireguard.com/)）
- **跨平台**：最初为 Linux 内核实现，现已支持 Windows、macOS、BSD、iOS、Android 等。（来源：[官网](https://www.wireguard.com/)）

白皮书摘要指出：WireGuard 以"对等体公钥 ↔ 隧道源 IP 地址"关联为基本安全原则，使用基于 Noise IK 的单次往返密钥交换，配合定时器状态机透明管理会话；完整 Linux 实现代码量不足 4,000 行（不含密码学原语），便于审计。（来源：[白皮书 Abstract](https://www.wireguard.com/papers/wireguard.pdf)）

---



## Design Principles（设计原则）



### 简洁性与可审计性

WireGuard 刻意追求配置与实现上的简洁：公钥交换方式类似 SSH，其余握手、重连、漫游对用户透明。（来源：[官网 — Simple & Easy-to-use](https://www.wireguard.com/)）

白皮书对比 IPsec/OpenVPN 指出：WireGuard 将密钥交换与第 3 层传输加密合并为单一机制，以虚拟接口（如 `wg0`）替代 xfrm 变换层，打破传统严格分层，换取实用性与可验证安全性；Linux 实现目标为少于 4,000 行代码。（来源：[白皮书 §1, §7](https://www.wireguard.com/papers/wireguard.pdf)）

### 密码学立场明确（Cryptographically Opinionated）

WireGuard **不提供**密码算法或协议敏捷性（cipher/protocol agility）；若底层原语出现漏洞，所有端点须统一升级。白皮书认为 TLS/IPsec 的算法协商带来巨大复杂性与攻击面。（来源：[白皮书 §1](https://www.wireguard.com/papers/wireguard.pdf)；[官网 — Cryptographically Sound](https://www.wireguard.com/)）

### Cryptokey Routing（密钥路由）

每个对等体由 32 字节 Curve25519 公钥唯一标识。接口维护 Cryptokey Routing Table：

- **出站**：根据目的 IP 查表，选择对应 peer 公钥加密发送。
- **入站**：解密后校验源 IP 是否属于该 peer 的 `AllowedIPs`；不匹配则丢弃。

由此，防火墙可简化为"该接口、该 IP 是否可信"，无需 IPsec 专用扩展语义。（来源：[官网 — Cryptokey Routing](https://www.wireguard.com/)；[白皮书 §2](https://www.wireguard.com/papers/wireguard.pdf)）

### 内置漫游（Roaming）

对等体可选预配置 `Endpoint`；若未配置，则从最近成功认证数据包的外层 UDP 源地址学习 endpoint。公钥唯一标识 peer，因此 IP/端口变化时双方可自动更新 endpoint，类似 Mosh 的漫游模型。（来源：[官网 — Built-in Roaming](https://www.wireguard.com/)；[白皮书 §2.1](https://www.wireguard.com/papers/wireguard.pdf)）

**与 NAT 的关系（概要）**：WireGuard **没有** STUN/TURN/UPnP 等独立 NAT 穿透协议；穿越依赖 **UDP + Endpoint + 漫游学习 + 可选 PersistentKeepalive**。详见下文 [NAT、防火墙穿越与 Endpoint](#nat防火墙穿越与-endpoint)。

### "沉默是美德"（Silence is a Virtue）

设计目标：**认证前不存储状态、不对未认证包发送任何响应**。未认证 peer 与网络扫描器无法探测到 WireGuard 服务存在。代价是首包须能认证 initiator，从而引入重放风险，由 TAI64N 时间戳机制缓解。（来源：[白皮书 §5.1](https://www.wireguard.com/papers/wireguard.pdf)）

### 密钥分发与 SSH 类比

公钥通过带外方式交换（邮件、LDAP、CA 等），协议层不规定分发机制；32 字节公钥 Base64 编码为 44 字符，便于传输。（来源：[白皮书 §1](https://www.wireguard.com/papers/wireguard.pdf)）

---



## Architecture（架构：用户态 / 内核态）



### Linux 内核实现

WireGuard 作为 Linux 内核虚拟网络接口实现（如 `wg0`），工作于第 3 层。实现目标包括：（来源：[白皮书 §7](https://www.wireguard.com/papers/wireguard.pdf)）

1. 代码短小、易于审计（< 4,000 行，不含密码原语）
2. 性能与 IPsec 竞争
3. 响应入站包时避免动态内存分配
4. 与现有内核基础设施及 `ip(8)` 等工具自然集成
5. 可作为外部内核模块构建，无需修改核心内核

内核实现采用 per-peer 队列、softirq 并行处理、RTNL 虚拟接口（支持容器 network namespace 迁移）等技术。（来源：[白皮书 §7.1–7.3](https://www.wireguard.com/papers/wireguard.pdf)）

### 用户态工具


| 组件             | 职责                                  | 来源                                                   |
| -------------- | ----------------------------------- | ---------------------------------------------------- |
| `wg(8)`        | 配置接口密钥、peer、endpoint；`wg show` 查看状态 | [Quick Start](https://www.wireguard.com/quickstart/) |
| `wg-quick(8)`  | 自动化 `wg` + `ip(8)` 的启停流程            | [Quick Start](https://www.wireguard.com/quickstart/) |
| `wireguard-go` | 非 Linux 平台的用户态实现                    | [Quick Start](https://www.wireguard.com/quickstart/) |
|                |                                     |                                                      |


接口创建与 IP 配置使用标准 Linux 网络工具：

```bash
ip link add dev wg0 type wireguard
ip address add dev wg0 10.192.122.3/24
wg setconf wg0 configuration.conf
ip link set wg0 up
```

（来源：[白皮书 §4](https://www.wireguard.com/papers/wireguard.pdf)；[Quick Start](https://www.wireguard.com/quickstart/)）

### 数据路径概要

**发送**（来源：[白皮书 §3](https://www.wireguard.com/papers/wireguard.pdf)）：

1. 明文 IP 包到达 `wg0`
2. 按目的 IP 查 Cryptokey Routing Table 确定 peer
3. 用该 peer 会话的发送密钥 + ChaCha20Poly1305 加密
4. 添加 WireGuard 头，经 UDP 发往 peer endpoint

**接收**：

1. UDP 端口收到 WireGuard 包
2. 根据头字段定位 peer 会话，验证 counter，AEAD 解密
3. 用外层 UDP 源地址更新 peer endpoint
4. 校验解密后源 IP 是否在 `AllowedIPs` 中
5. 合法则注入 `wg0` 接收队列

---



## Usage（使用：wg genkey、wg-quick）



### 密钥生成

WireGuard 使用 Base64 编码的公私钥对：

```bash
umask 077
wg genkey | tee privatekey | wg pubkey > publickey
```

`wg genkey` 生成私钥；`wg pubkey` 从私钥导出公钥。（来源：[Quick Start — Key Generation](https://www.wireguard.com/quickstart/)）

### 配置文件示例

服务端（来源：[官网 — Cryptokey Routing](https://www.wireguard.com/)）：

```ini
[Interface]
PrivateKey = yAnz5TF+lXXJte14tji3zlMNq+hd2rYUIgJBgB3fBmk=
ListenPort = 51820

[Peer]
PublicKey = xTIBA5rboUvnH4htodjb6e697QjLERt1NAB4mZqp8Dg=
AllowedIPs = 10.192.122.3/32, 10.192.124.1/24
```

客户端：

```ini
[Interface]
PrivateKey = gI6EdUSYvn8ugXOt8QQD6Yc+JyiZxIhp3GInSWRfWGE=
ListenPort = 21841

[Peer]
PublicKey = HIgo9xNzJMWLKASShiTqIybxZ0U3wGLiUeJ1PKf8ykw=
Endpoint = 192.95.5.69:51820
AllowedIPs = 0.0.0.0/0
```



### wg-quick

`wg-quick(8)` 将 `wg setconf`、`ip link`、`ip address`、`ip route` 等步骤封装为一条命令，用于日常接口启停。（来源：[Quick Start — Command-line Interface](https://www.wireguard.com/quickstart/)）

### AllowedIPs 与路由决策

"包走不走 `wg0`" 不是看"目的 IP 是不是隧道 IP"，而是看 **Linux 路由表**，路由表又是 `wg-quick` 从 peer 的 `AllowedIPs` 自动派生的——`AllowedIPs` 写哪些网段，对应那些网段的包就路由进 `wg0`。

#### 三种典型配置

**情况 1：点对点，**`AllowedIPs = 10.192.122.0/24`

只有去 `10.192.122.x` 的包走 `wg0`。


| 你 ping 谁                            | 走 wg？ | 实际走什么                        |
| ----------------------------------- | ----- | ---------------------------- |
| 隧道 IP `10.192.122.1`                | ✓ 走   | wg0 → 加密 → UDP 发到对端 endpoint |
| 对端底层公网 IP `203.0.113.1`（即 Endpoint） | ✗ 不走  | eth0/wlan0，普通路由              |
| 别的公网 IP `8.8.8.8`                   | ✗ 不走  | eth0/wlan0，普通路由              |
| 本地内网段 `192.168.1.x`                 | ✗ 不走  | eth0/wlan0，普通路由              |


**情况 2：全流量代理，**`AllowedIPs = 0.0.0.0/0`

`wg-quick` 会往路由表里加两条规则：

1. 几乎所有目的地都走 `wg0`（包括 `8.8.8.8`、任意公网 IP）。
2. **例外**：发往 Endpoint 本身 `203.0.113.1` 的包**仍走 eth0**——否则就成了"为了把加密包送到对端，又要把这个加密包再加密一次送过去"的死循环。这条例外由 `wg-quick` 自动加的 "`ip route add <endpoint> via <原网关>`" 实现。


| 你 ping 谁                        | 走 wg？             |
| ------------------------------- | ----------------- |
| 隧道 IP `10.192.122.1`            | ✓ 走               |
| 对端底层 IP `203.0.113.1`（Endpoint） | ✗ **不走**（例外，避免循环） |
| 别的公网 IP `8.8.8.8`               | ✓ **走**（被代理）      |
| 本地网关 `192.168.1.1`              | ✗ 不走（本地路由优先）      |


**情况 3：split tunnel，**`AllowedIPs = 10.0.0.0/8, 192.168.50.0/24`

只有指定子网走 wg。`wg-quick` 自动只对这些子网加路由，其他一律走原网卡。

#### 永远成立的一条规律

**ping 对端的真实 IP（Endpoint 地址）永远不走 wg**。这不是配置决定的，而是物理上的硬约束：走 wg 意味着把包用 ChaCha20Poly1305 加密、外层 UDP 目的地址 = 对端 endpoint `(203.0.113.1:51820)`，而这个外层 UDP 又得有个出方向——出方向的目的 IP 又是 `203.0.113.1`，又匹配 `AllowedIPs` 又要走 wg——**死循环**。所以 `wg-quick` 必须给 Endpoint 加一条特殊路由把它"固定到原网卡"。这是 WG 部署里一个隐藏但极其重要的细节。

#### 一句话总结


| 常见说法                  | 实际正确条件                                          |
| --------------------- | ----------------------------------------------- |
| "只有 ping 隧道 IP 才走 wg" | 仅当 `AllowedIPs` 只写隧道子网时成立                       |
| "ping 底层 IP 不走 wg"    | **总是成立**（Endpoint IP 必走原网卡，否则死循环）               |
| "ping 公网都不走 wg"       | 错。`AllowedIPs = 0.0.0.0/0` 时全走，仅 Endpoint IP 例外 |


> 决定"包走不走 wg0"的是路由表，路由表又是从 `AllowedIPs` 自动派生的。`AllowedIPs` 写什么网段，那些网段的包就走 wg；没写的走原网卡。唯一例外是对端 Endpoint IP——`wg-quick` 给它显式加一条"仍走原网卡"的路由，避免加密包"自己套自己"的死循环。

---



## NAT、防火墙穿越与 Endpoint

WireGuard 官方文档中与 NAT/防火墙相关的内容分散在 **Roaming**、**Quick Start** 与 **Timers** 几节；本节集中整理，便于与 PQ-SecTunnel 对照。

### NAT 基础：映射表与"洞"的概念

**NAT（Network Address Translation）** 是 IPv4 地址不够分的产物：一个内网共用一个公网 IP 出网，NAT 设备（家用路由器即一台）在出/入站时改写地址并维护一张**状态表/映射表**。一条表项形如：


| 内网 IP:端口          | ↔   | 公网 IP:端口          | ↔   | 允许的对端              | 剩余寿命 |
| ----------------- | --- | ----------------- | --- | ------------------ | ---- |
| 192.168.1.5:51820 | ↔   | 203.0.113.1:51820 | ↔   | 198.51.100.7:51820 | 120s |


这张表里的一条表项就被形象地叫**"洞"（hole）**：从这边出去的包让 NAT 记下来，从外面回进来的包只要源/目的对得上就能被放进来。"洞"有寿命（一般 30~120s 无出站包即过期），而且是**单向放行**——不是真的"挖穿防火墙"，只是 NAT 状态机里的一条放行表项。

NAT 分几种类型，对穿透难度影响极大（从易到难）：


| 类型                   | 行为                       | 打洞难度      |
| -------------------- | ------------------------ | --------- |
| Full-cone            | 一次出站后任何外网地址都能回进这个孔       | 易         |
| Restricted cone      | 只有被发过的那个对端 IP 能回         | 中         |
| Port-restricted cone | 只有被发过的那个 `(IP, port)` 能回 | 中         |
| Symmetric            | 每去一个不同对端就开新端口            | 极难（几乎打不通） |




#### 关键过程：Client 主动出站时 NAT 自动建表

```
时刻 t0 —— 初始状态，NAT 状态表为空

   内网                       NAT 设备                       公网
┌──────────┐                                              ┌──────────┐
│ Client A │     ┌──────────────────────┐                  │ Server S │
│ 192.168  │     │ NAT 状态表(空)       │                  │ 198.51   │
│ .1.5     ├─────┤  ┌──────────────────┐│                  │ .100.7   │
│ :51820   │     │  │ (尚未有表项)      ││                  │ :51820   │
└──────────┘     │  └──────────────────┘│                  └──────────┘
                 └──────────────────────┘
   ☒ 此时 Server 即使主动发 UDP 给 203.0.113.1:51820，NAT 查表为空 → 丢弃


时刻 t1 —— Client 主动发出第一个 UDP 包

内网 Client A                  NAT 设备                      公网 Server S
┌──────────┐         ┌──────────────────────┐                   ┌──────────┐
│ 192.168  │ 出站包 │ 改写源地址 + 建表     │ 改写后           │ 198.51   │
│ .1.5     ├───────►│ 192.168.1.5:51820     │────────────────► │ .100.7   │
│ :51820   │  UDP   │  → 203.0.113.1:51820  │  UDP             │ :51820   │
└──────────┘         │                      │ 收到             └──────────┘
                     │  ★ 表项新增          │
                     │  192.168.1.5:51820   │
                     │   ↔ 203.0.113.1:51820│ ← 公网映射
                     │   ↔ 198.51.100.7:51820│ ← 允许的对端
                     │   剩余寿命 120s       │
                     └──────────────────────┘
   ✓ 一个"洞"被 Client 出站动作自动建出来，指向 Server 这一个对端


时刻 t2 —— Server 顺着洞回包

内网 Client A                  NAT 设备                    公网 Server S
┌──────────┐         ┌──────────────────────┐                 ┌──────────┐
│         ◀├─────────┤ 查表 ✓ 反向改写       │◀────────────── │ 198.51   │
│          │ 改回    │ 目的匹配表项          │  回包           │ .100.7   │
│          │ 内网    │ 刷新寿命             │                 │ :51820   │
└──────────┘         └──────────────────────┘                 └──────────┘
   ✓ Client 收到回包
```

**这个 t1 出站动作本身就在 NAT 上"开了一个洞"，Server t2 蹭洞回包，不需要任何打洞协商。** 这正是 WG NAT 模型的物理基础。

#### 为什么"反向"不通：NAT 是被动状态机

把 t0 时序图反过来：如果 Server 在 Client 还没主动出站过任何包时，先主动向 NAT 后的 Client 发 UDP，会发生什么？

```
内网 Client A                  NAT 设备                    公网 Server S
┌──────────┐         ┌──────────────────────┐           ┌──────────┐
│ 192.168  │         │ NAT 状态表 (空)       │ 入站包   │ 198.51   │
│ .1.5     │         │                      │◀─────┐  │ .100.7   │
│ :51820   │         │  查表 ✗ → 丢弃        │      │ │ :51820   │
│          │         │  (没有匹配的表项)     │──────┘ │ 主动发   │
└──────────┘         │                      │        │ 包给     │
   ☒ 收不到          └──────────────────────┘        │ 203.0.   │
                     ↑                              │ 113.1    │
                     没有"洞"                        └──────────┘
```

NAT 收到入站包，要在状态表里找一条匹配的表项才能改写源地址转发到内网。表为空 → 找不到任何映射 → **丢弃**。NAT 不能凭入站包"自己创造映射"——因为它根本不知道该把这个包改写为哪一台内网主机（一个公网 IP 后面可能有几十台内网机器，没有事先约定的对应关系）。

更根本的安全原因：如果 NAT 不丢而是盲目转给某台内网主机，NAT 就退化成一个穿透后的桥接器，内网设备全部裸露。**默认丢是 NAT 的安全保护行为**。

一句口诀记牢：

> **NAT 表的"出方向"决定谁能回进来。Client 先出站，NAT 就学会"我这个端口的回包应该送回这台 Client"——洞被开出来。Server 不等 Client 出站就先发，NAT 没表可查，丢弃。NAT 是被动状态机，只能被出站事件学会映射，不能被入站事件自己创造映射——这是它和"防火墙端口转发/静态映射"的本质区别。**



#### NAT、漫游、打洞三者的关系串讲

三个容易混淆的概念，按"解决什么问题 / 靠不靠外部服务器 / 需不需要协议层动作"三轴对齐：


|          | 漫游 (Roaming)               | 打洞 (Hole punching)         |
| -------- | -------------------------- | -------------------------- |
| 解决场景     | 一端能先发出去，另一端**收到**后学与它的外层地址 | 两端都在 NAT 后，**谁都没法先收到对方**的包 |
| 需要外部服务器？ | 不需要                        | 需要 STUN/TURN 中继            |
| 是协议层动作吗？ | 不是，是缓存自动更新                 | 是，需要信令协商互相交换公网映射           |
| WG 有没有？  | 有                          | 无                          |


一条串讲：

> **NAT 是 IPv4 不够用的产物，它在内网设备出外网时偷偷维护一张"映射表"，每条表项就是一个"洞"——出这边出去的包能让外面回进来。**
>
> **漫游是 WireGuard 做的最简单的事：你发我一条合法包，我就用你外面那一层 (IP, port) 更新我的"把包送给你"目标地址。你切网络我不在乎，你发的下一包就告诉我你现在在哪儿——不需要协商、不需要信令，只是个被动缓存刷新。**
>
> **打洞是另一回事**：它专治"两边都在 NAT 后，谁也没法先让对方收到包"这种死局。打洞需要一个公网中继先告诉双方"对方的外网映射是什么"，然后两边各自向对方的外网映射发一个出站包，让两边 NAT 在表里各开一条"对端可回"的洞——之后才能直接对通。**
>
> **WireGuard 故意没做打洞**，因为打洞要信令协议（STUN/TURN/ICE），违反"短代码、无 chatty 协议"哲学。所以 WG 的 NAT 模型只包含：UDP + 监听端口同源端口 + 漫游学习 + 可选 PersistentKeepalive 维持 NAT 表项。代价是"两端都在对称 NAT 后无解"——这种场景要么手动端口前置、要么上中继。

> **Roaming 蹭了 t1 自带的被动建表**（出站一方自动开洞，对端学到外层地址）；**打洞是"没有天然 t1 时双方人工造一次 t1"**（借公网中继先交换公网映射，然后两边各自主动出站开洞）。两者都依赖 NAT 被动状态机规律，只是触发方式不同。



#### NAT 的三种典型拓扑复现

```
场景 A：一端公网、一端 NAT 后（最常见，WG 完全胜任）
  NAT 后 Client 主动出站 → 公网 Server ← 这一步 NAT 自动开洞
  Server 学到 Client 外层 (IP, port)（漫游）
  之后双向通；Client 切网络，漫游照样工作

场景 B：双端都在 NAT 后（WG 解决不了，需打洞或中继）
  必须一侧手动在 NAT 上做端口前置（让那一侧"类似在公网"）
  或：双方各自指向一个公网 relay，由 relay 中转 WG UDP 包
  但 WG 自己不会主动去协商打洞

场景 C：移动场景 / IP 变化
  公钥不变、漫游自动更新 Endpoint
  换 Wi-Fi/4G 后通常无需改配置
  新位置若也还在 NAT 后，对端发的包导向新 NAT 外口
  —— 前提是：第一次握手能再次发生（新一侧能先出一包）
```



### WireGuard 不做什么


| 常见 VPN/NAT 手段       | WireGuard                           |
| ------------------- | ----------------------------------- |
| STUN / TURN / ICE   | **无** — 不内建打洞协商                     |
| TCP fallback        | **无** — 仅 UDP                       |
| UPnP / NAT-PMP 端口映射 | **无** — 需管理员自行 port forward         |
| 应用层 NAT 保活协议        | **无** — 可选配置级 `PersistentKeepalive` |


设计取向：协议默认**尽量静默**（非 chatty），只在有数据或定时器触发时发包。（来源：[Quick Start — NAT and Firewall Traversal Persistence](https://www.wireguard.com/quickstart/)）

### Endpoint 与漫游如何配合 NAT

**Endpoint** = 对端在 Internet 上可达的 `IP:UDP端口`，用于出站加密包的 UDP 目的地址。


| 配置                       | 行为                                                 |
| ------------------------ | -------------------------------------------------- |
| 预配置 `Endpoint = A:51820` | 出站发往 A:51820                                       |
| 未配置 Endpoint             | 必须**先收到**对端合法包，从外层 UDP **源**地址学习 endpoint          |
| 收到合法包后                   | **始终用最新**外层 `源IP:源端口` 更新该 peer 的 endpoint（Roaming） |


白皮书 §2.1 强调：

- **监听端口与发送源端口相同**（listen port = source port），简化 NAT 映射一致性，有利于穿越。（来源：[白皮书 §2.1](https://www.wireguard.com/papers/wireguard.pdf)）
- 因漫游会持续刷新对端的外网 `(IP, Port)`，**不必依赖 NAT 长时间保持同一映射**；映射变化后，下一合法包即可更新 endpoint。
- 若必须**无限期**维持 NAT/有状态防火墙上的 pinhole，可启用 **PersistentKeepalive**（见下）。

**安全边界**：外层 UDP 源 IP 可被中间人改写，但**不能**伪造加密载荷；改源地址最多造成 DoS/断连，不能解密或注入明文。（来源：[白皮书 §2.1](https://www.wireguard.com/papers/wireguard.pdf)）

### 典型拓扑与 NAT 行为

```
场景 A — 一端公网、一端 NAT 后（最常见）
  [NAT 内 Client] ──UDP──► [公网 Server :51820]
  Client 配置 Endpoint=Server；Server 从 Client 首包学习 Client 的 NAT 映射
  → 双向通信成立

场景 B — 双端均在 NAT 后
  至少一端须「先能被找到」：公网 IP、端口映射、或中间 relay
  WireGuard 本身不会在两个对称 NAT 之间自动打洞
  → 需一侧 Endpoint 可达，或由有公网的一端先收到 Initiation

场景 C — 移动网络 / IP 变化
  公钥不变；Roaming 从新合法包的外层源地址更新 Endpoint
  → 换 Wi‑Fi/4G 后通常无需改配置（对端仍用旧 endpoint 发，直到收到新包）
```



### PersistentKeepalive（配置级 NAT 保活）

**问题**：NAT 与有状态防火墙按「连接」跟踪 UDP；若 peer **长时间只收不发**，映射可能过期，对端后续包无法进入。

**官方方案**（[Quick Start — NAT and Firewall Traversal Persistence](https://www.wireguard.com/quickstart/)）：

- 在 **NAT/防火墙后的 peer** 上，对需要**长期接收入站**的对端配置 `PersistentKeepalive = <秒>`。
- 每隔该间隔向 **已配置的 Endpoint** 发送 keepalive（空 transport data 包），维持 NAT pinhole。
- **默认 0（关闭）** — 官方认为多数场景不需要；开启会略增流量。
- **常用值 25 秒** — 文档称与多种防火墙兼容。

配置示例：

```ini
[Peer]
PublicKey = ...
Endpoint = 203.0.113.1:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

命令行等价：`wg set wg0 peer ... persistent-keepalive 25`

### Passive Keepalive（协议定时器，≠ PersistentKeepalive）

白皮书 §6.5 描述的 **被动 keepalive** 是内核定时器行为，与配置项 `PersistentKeepalive` **不是同一机制**：


|     | PersistentKeepalive | Passive Keepalive（§6.5）                         |
| --- | ------------------- | ----------------------------------------------- |
| 触发  | 配置周期，**无论是否有业务**    | 已收到对端数据，但本地 **KEEPALIVE_TIMEOUT** 内无待发          |
| 目的  | 维持 NAT pinhole      | 检测会话存活、促使对端回复                                   |
| 默认  | 关（0）                | 协议内置（`KEEPALIVE_TIMEOUT = 10` 秒，见 `messages.h`） |
| 包类型 | 空 transport data    | 空 transport data（长度 0 内层 IP）                    |


协议页与白皮书还规定：若 `(KEEPALIVE_TIMEOUT + REKEY_TIMEOUT)` 内未收到对端 transport 包，则发起新握手（与 NAT 间接相关——会话恢复）。（来源：[protocol 页 Timers](https://www.wireguard.com/protocol/)；[白皮书 §6.4–6.5](https://www.wireguard.com/papers/wireguard.pdf)）

### 防火墙要点

- 需放行接口 **ListenPort** 上的 **UDP** 入站（默认常 51820）。
- 未通过 MAC1 的握手包被**静默丢弃**，防火墙扫描通常看不到响应（Silence is a Virtue）。
- 高负载下 Cookie 机制与 NAT 无直接关系，但影响握手是否被处理（见 DoS 章节）。



### PQ-SecTunnel 中的保留情况

本仓库 fork **保留** WireGuard 的 NAT/漫游模型，未引入新 NAT 协议：


| 机制                        | PQ 状态 | 代码线索                                                             |
| ------------------------- | ----- | ---------------------------------------------------------------- |
| Endpoint 学习 / Roaming     | ✅ 保留  | `receive.c` 收包后更新 endpoint                                       |
| `PersistentKeepalive`     | ✅ 保留  | `PQSTPEER_A_PERSISTENT_KEEPALIVE_INTERVAL`（`uapi/pqwireguard.h`） |
| 空包 keepalive              | ✅ 保留  | `wg_packet_send_keepalive()`（`send.c`）                           |
| 定时器 `KEEPALIVE_TIMEOUT` 等 | ✅ 保留  | `timers.c`、`messages.h`                                          |
| 固定 listen/source 端口语义     | ✅ 保留  | WireGuard 同源设计                                                   |


**差异**：PQ 握手报文更大、可能应用层分片（`pqc_fragment`），但 **UDP 外层 endpoint/NAT 语义不变**；NAT 后 peer 仍可按需配置 `PersistentKeepalive`。答辩对比全文见 `[doc/defense/00a-WireGuard背景与对比.md](../defense/00a-WireGuard背景与对比.md)`。

---



## Noise Protocol（Noise_IKpsk2 高层步骤）



### WireGuard 选用的 Noise 变体

WireGuard 协议标识符（`CONSTRUCTION`）为：

```
Noise_IKpsk2_25519_ChaChaPoly_BLAKE2s
```

（37 字节 UTF-8 字符串）（来源：[协议页面](https://www.wireguard.com/protocol/)；[白皮书 §5.4](https://www.wireguard.com/papers/wireguard.pdf)）

含义拆解（来源：[Noise §8](https://noiseprotocol.org/noise.html)；[Noise §9.4](https://noiseprotocol.org/noise.html)）：


| 部分           | 含义                                                            |
| ------------ | ------------------------------------------------------------- |
| `IK`         | 交互式握手模式：initiator 已知 responder 静态公钥；initiator 在首条消息中发送自己的静态公钥 |
| `psk2`       | 预共享对称密钥（PSK）在**第二条握手消息末尾**混入                                  |
| `25519`      | Curve25519 DH                                                 |
| `ChaChaPoly` | ChaCha20-Poly1305 AEAD                                        |
| `BLAKE2s`    | BLAKE2s 哈希                                                    |




### IK 基本模式（无 PSK）

Noise 规范中 `IK` 模式定义为：（来源：[Noise §10.4, §7.5](https://noiseprotocol.org/noise.html)）

```
IK:
  <- s                          # 预消息：initiator 已知 responder 静态公钥
  ...
  -> e, es, s, ss               # 消息 1（initiator）：明文 e，加密 s，DH: es + ss
  <- e, ee, se                  # 消息 2（responder）：明文 e，DH: ee + se
```

命名规则（来源：[Noise §7.5](https://noiseprotocol.org/noise.html)）：

- 首字母 `I`：initiator 静态公钥 **立即** 发送给 responder
- 次字母 `K`：responder 静态公钥 initiator **事先已知**

`IKpsk2` 在第二条消息末尾追加 `psk` token，调用 `MixKeyAndHash(psk)` 将 32 字节 PSK 混入链密钥与哈希。（来源：[Noise §9.1–9.4](https://noiseprotocol.org/noise.html)）

### WireGuard 对 IKpsk2 的具体化（逐步概览）

WireGuard 在 Noise 框架之上增加了应用层字段（TAI64N 时间戳、MAC1/MAC2），并将第二条消息的 payload 设为空（encrypted nothing），PSK 在 responder 侧混入。

#### "应用层"的含义澄清

这里的"应用层"**不是** OSI 七层模型中的第 7 层（HTTP/DNS 等），而是**相对 Noise 协议框架而言的"外挂层"**。Noise 把自己定义为一个密钥协商框架，只管 `ck`/`h` 演化、token 序列、AEAD 加密 payload；框架之外的所有字段与约定它一概不关心，都被它统称"应用层"。

```
┌──────────────────────────────────────────┐
│  应用层（站在 Noise 角度看）              │  ← WireGuard 在此添加 TAI64N、mac1、mac2
│  - WG 握手消息的字段编排                  │
│  - TAI64N 时间戳（防重放）                │
│  - mac1 / mac2（DoS 防护）                │
│  - 路由 / Cryptokey Routing / AllowedIPs │
├──────────────────────────────────────────┤
│  Noise 协议框架（密钥协商）               │  ← Noise 规范只管这层
│  - ck / h 演化                           │
│  - MixHash / MixKey / MixKeyAndHash      │
│  - token 序列 e, es, s, ss, ...          │
│  - AEAD 加密 payload                     │
├──────────────────────────────────────────┤
│  底层（Noise 也不管）                     │
│  - UDP / IP / 网卡                        │
└──────────────────────────────────────────┘
```

判别一个字段算不算 Noise 状态机的一部分，问一句：

> **这字段会不会参与** `ck` **或** `h` **的演化？**
>
> - **会** → 算 Noise 状态机的一部分（如临时公钥 `e`、加密静态公钥 `s`、DH 输出、PSK）。
> - **不会** → 算"应用层字段"（如 mac1、mac2）。

`TAI64N` 时间戳是边缘情况：它作为"消息 1 的加密 payload"位置确实被 AEAD 加密并入 hash（走 Noise 流程），但**payload 内部放什么数据、为什么是 12 字节、是不是时间戳**——这些都是 Noise 不关心的，由 WG 自己定义。所以时间戳"算 Noise 协议布局的一部分，但其内容是应用层约定"。

`mac1`/`mac2` 完全不进 `ck`/`h`，是纯外挂。它们的计算不更新也不会被传输密钥派生消费——纯粹是 WG 在"沉默是美德"目标下加的 DoS 过滤前置，在 Noise 角度看就像不存在。

> **同一段话里的三层意思拆开**：
>
> 1. **mac1/mac2** 是 Noise 之外的附加字段，不进 transcript，不参与密钥派生——这是"应用层字段"最典型的含义。
> 2. **TAI64N 时间戳** 占用 Noise 规定的"消息 1 加密 payload"位置，被 AEAD 加密并入 transcript，但 payload 内部的语义（时间戳格式、防重放逻辑）由 WG 定义。Noise 只负责"加密这段 payload 并入 hash"，不关心内部是什么。
> 3. `encrypted_nothing` 占用 Noise 规定的"消息 2 末尾加密 payload"位置，明文是 0 字节，AEAD 仍产 16 字节 Poly1305 tag 并入 transcript。WireGuard 借这个空 payload 做"密钥确认"——responder 用 psk 后派生的 `k6` 加密空内容，initiator 收到后用同样派生的 `k6'` 解密，tag 通过则证明双方 PSK 与所有 DH 派生一致。

三者合起来，WireGuard 在不修改 Noise 状态机的前提下，把"首包认证 + 抗 DoS + 抗重放 + 密钥确认 + 无阻塞首包"五个工程目标塞进了 IKpsk2 这个骨架里。这也是 PQ-SecTunnel 的改造策略——只把 `psk2` 换成 `psk1kem`，不动这三处工程改造，等于"我们只换了密码学，没动协议层工程"。

> **答辩话术**：被追问"你说的应用层是 OSI 第 7 层吗"时，应答："我用的'应用层'是相对意义——站在 Noise 协议框架的角度，WireGuard 在 Noise 规定的 token 序列之外追加的字段（TAI64N、mac1、mac2），不走 `MixHash`/`MixKey`，不影响传输密钥派生。所以是'在 Noise 之上'的一层附加约定，和 OSI 第 7 层的 HTTP/DNS 没有关系。"不要省略"站在 Noise 角度"这个限定词，否则容易被听众套到 OSI 模型产生误会。



#### 初始化

双方维护 `chaining_key`（`ck`）与 `hash`（`h`）：

```
ck = HASH(CONSTRUCTION)           # CONSTRUCTION = "Noise_IKpsk2_25519_ChaChaPoly_BLAKE2s"
h  = HASH(HASH(ck || IDENTIFIER) || responder_static_public)
```

`IDENTIFIER` = `"WireGuard v1 zx2c4 Jason@zx2c4.com"`（来源：[协议页面](https://www.wireguard.com/protocol/)）

#### 消息 1：Initiator → Responder（handshake_initiation, type=1）


| 步骤  | 操作                                                   |
| --- | ---------------------------------------------------- |
| 1   | 生成 initiator 临时密钥对 `(e_priv, e_pub)`                 |
| 2   | 发送明文 `e_pub`；`h = HASH(h                             |
| 3   | DH `es`：initiator 临时私钥 × responder 静态公钥 → 更新 `ck`    |
| 4   | AEAD 加密 initiator **静态公钥** → `encrypted_static`      |
| 5   | DH `ss`：initiator 静态私钥 × responder 静态公钥 → 更新 `ck`    |
| 6   | AEAD 加密 **TAI64N 时间戳**（12 字节）→ `encrypted_timestamp` |
| 7   | 计算 `mac1`、`mac2`（见 DoS Cookies 节）                    |


消息结构（来源：[协议页面](https://www.wireguard.com/protocol/)）：

```
message_type(1) | reserved(3) | sender_index(4) | unencrypted_ephemeral(32)
| encrypted_static(48) | encrypted_timestamp(28) | mac1(16) | mac2(16)
```

对应 Noise token 序列：`e, es, s, ss`（s 的 payload 被 WireGuard 替换为 timestamp）。

#### 消息 2：Responder → Initiator（handshake_response, type=2）


| 步骤  | 操作                                                                  |
| --- | ------------------------------------------------------------------- |
| 1   | 生成 responder 临时密钥对                                                  |
| 2   | 发送明文 `e_pub`；`h = HASH(h                                            |
| 3   | DH `ee`：responder 临时 × initiator 临时                                 |
| 4   | DH `se`：responder 临时 × initiator 静态                                 |
| 5   | **混入 PSK**（`psk2`）：`MixKeyAndHash(preshared_key)`；无 PSK 时使用 32 字节全零 |
| 6   | AEAD 加密空 payload → `encrypted_nothing`（密钥确认）                        |
| 7   | 计算 `mac1`、`mac2`                                                    |


消息结构：

```
message_type(1) | reserved(3) | sender_index(4) | receiver_index(4)
| unencrypted_ephemeral(32) | encrypted_nothing(16) | mac1(16) | mac2(16)
```

对应 Noise token：`e, ee, se, psk`（psk 在消息末尾）。

#### 传输密钥派生

握手完成后，双方从最终 `chaining_key` 经 HKDF 派生发送/接收密钥，清零所有临时密钥、链密钥与哈希。（来源：[协议页面 — Data Keys Derivation](https://www.wireguard.com/protocol/)）

#### 4 次 ECDH 各自"绑"什么——A 单向绑定 vs B 双方互锁

WireGuard IKpsk2 共做 **4 次 ECDH**（`es, ss, ee, se`），不是 3 次。每次 ECDH 经 `MixKey` 把 shared_secret 灌入 chaining_key `ck`，少算任一个都会得到不同的 ck、不同的传输密钥。但 4 次 ECDH 的"绑定作用"**强度不等**，要分两种语义才能讲清，也是 PQ-SecTunnel 改造时的关键决策依据。

**绑定 A —— 单向身份证明**

> 一份共享秘密被灌入 ck，**验证**它需要某一方的私钥。攻击者不知道该私钥 → 解不开 → 后续 ck 演化对不上 → 握手失败。
> 强度："必须有 S_x^priv 才能让握手走过这一步。"

**绑定 B —— 双方私钥互锁（mutual coupling）**

> 一份共享秘密被灌入 ck，**产生或验证**它需要**同时**依赖双方私钥。
> 强度："只有真双方配合，这一步才能产合法输出；攻击者**连投递合法输入**都做不到。"

把 4 次 ECDH 按这两种语义归类：


| ECDH                         | 绑谁               | 作用               |
| ---------------------------- | ---------------- | ---------------- |
| `es = DH(e_i^priv, S_r^pub)` | responder 静态（单向） | 客户端开口并证明「认服务器」   |
| `ss = DH(S_i^priv, S_r^pub)` | **双方静态互锁**       | 锁上双方长期身份         |
| `ee = DH(e_r^priv, e_i^pub)` | 双方临时互锁（本次会话）     | 服务器回复，双方临时钥锁本轮会话 |
| `se = DH(e_r^priv, S_i^pub)` | initiator 静态（单向） | 服务器证明「也认得你」      |
|                              |                  |                  |
两边推出同一会话密钥，开始传数据

注意 ECDH 的对称性是 `ss` 能做 B 类的关键：`DH(a_priv, b_pub) = DH(b_priv, a_pub)`，所以 initiator 与 responder 各用自己静态私钥 + 对方公开的静态公钥，**双方独立地算出同一份 shared_secret**。这给 ECDH 一种独特能力——**1 次 DH 运算、0 字节握手载荷**就完成 B 类"双方私钥互锁"。

### 安全属性

WireGuard 握手达成：（来源：[白皮书 §5](https://www.wireguard.com/papers/wireguard.pdf)；[协议页面](https://www.wireguard.com/protocol/)）

- AKE 安全（Authenticated Key Exchange）
- 抗密钥泄露冒充（KCI resistance）
- 抗重放（结合 TAI64N 时间戳与传输 counter 滑动窗口）
- 完美前向保密（Perfect Forward Secrecy）
- 身份隐藏（类似 SIGMA；IK 模式 initiator 静态公钥加密传输，属性为 Noise 表中的 4/3）



### 密钥确认（Key Confirmation）

握手两条消息交换后，**initiator 可立即发送加密数据包，responder 须等到收到 initiator 的第一条 transport data 后才能使用新会话发送**。若 initiator 无待发数据，应发送空包以完成确认（KEA+C 风格）。（来源：[协议页面 — Connection-less Protocol](https://www.wireguard.com/protocol/)；[白皮书 §5](https://www.wireguard.com/papers/wireguard.pdf)）

---



## Cryptographic Primitives（密码学原语）

WireGuard 使用的协议与原语（来源：[协议页面 — Primitives](https://www.wireguard.com/protocol/)；[白皮书 §5.4](https://www.wireguard.com/papers/wireguard.pdf)）：


| 用途          | 算法                     | 标准/说明                                      |
| ----------- | ---------------------- | ------------------------------------------ |
| 对称加密        | ChaCha20-Poly1305 AEAD | RFC 7539 构造；nonce = 32 位零 + 64 位小端 counter |
| ECDH        | Curve25519             | 32 字节公钥/共享秘密                               |
| 哈希 / 密钥哈希   | BLAKE2s（32 字节输出）       | RFC 7693                                   |
| 哈希表键        | SipHash24              | 内核实现内部使用                                   |
| 密钥派生        | HKDF                   | RFC 5869；基于 HMAC-BLAKE2s                   |
| Cookie 回复加密 | XChaCha20-Poly1305     | 24 字节随机 nonce                              |
| 时间戳         | TAI64N                 | 12 字节大端；防首包重放                              |


辅助函数定义（来源：[协议页面](https://www.wireguard.com/protocol/)）：

- `DH()`：Curve25519 点乘
- `AEAD()`：ChaCha20Poly1305，counter 作 nonce
- `HMAC()`：`HMAC-BLAKE2s`，32 字节
- `MAC()`：`Keyed-BLAKE2s`，16 字节
- `HASH()`：`BLAKE2s`，32 字节

---



## Handshake Flow（握手流程）



### 消息类型

WireGuard 定义 4 种消息（来源：[白皮书 §5.4](https://www.wireguard.com/papers/wireguard.pdf)）：


| type | 名称                   | 方向                            |
| ---- | -------------------- | ----------------------------- |
| 1    | Handshake Initiation | Initiator → Responder         |
| 2    | Handshake Response   | Responder → Initiator         |
| 3    | Cookie Reply         | 负载下 Responder/Initiator → 请求方 |
| 4    | Transport Data       | 双向                            |




### 正常 1-RTT 时序

```
Initiator                          Responder
    |                                   |
    |--- Handshake Initiation (1) ------>|
    |<-- Handshake Response (2) ---------|
    |--- Transport Data (4) ------------>|  (密钥确认：首条加密数据)
    |<-- Transport Data (4) -------------|
    |                                   |
```

（来源：[白皮书 §5.4.1 图](https://www.wireguard.com/papers/wireguard.pdf)；[协议页面](https://www.wireguard.com/protocol/)）

### 负载下 Cookie 交换（额外 RTT）

若 responder 处于高负载且 `mac2` 无效，可能插入 Cookie Reply，使总握手变为约 1.5–2 RTT。（来源：[白皮书 §5.3, §5.4.7](https://www.wireguard.com/papers/wireguard.pdf)）

### 定时器驱动的重协商

握手完成后，协议通过定时器自动轮换密钥（完美前向保密），主要常量（来源：[白皮书 §6.1](https://www.wireguard.com/papers/wireguard.pdf)）：


| 常量                     | 值           |
| ---------------------- | ----------- |
| `REKEY_AFTER_MESSAGES` | 2^60 条消息    |
| `REKEY_AFTER_TIME`     | 120 秒       |
| `REJECT_AFTER_TIME`    | 180 秒       |
| `REKEY_TIMEOUT`        | 5 秒（握手重传间隔） |
| `KEEPALIVE_TIMEOUT`    | 10 秒        |


管理员无需手动触发重连；接口配置私钥与 peer 公钥后即可发送普通 IP 包。（来源：[白皮书 §6](https://www.wireguard.com/papers/wireguard.pdf)）

---



## Data Plane（数据平面）



### Transport Data 消息（type=4）

结构（来源：[协议页面](https://www.wireguard.com/protocol/)；[白皮书 §5.4.6](https://www.wireguard.com/papers/wireguard.pdf)）：

```
message_type(1) | reserved(3) | receiver_index(4) | counter(8) | encrypted_encapsulated_packet[]
```

- `receiver_index`：对端在握手中分配的 32 位索引（类 IPsec SPI）
- `counter`：64 位小端 nonce 计数器，单调递增
- 内层为完整 IP 包，加密前填充至 16 字节倍数（不改变 IP 头中的长度字段）



### 重放防护

UDP 可能乱序投递。WireGuard 在验证 AEAD 标签后，使用滑动窗口（约 2000 个 prior counter 值）检测重放，算法参考 RFC 6479。（来源：[协议页面 — Nonce Reuse & Replay Attacks](https://www.wireguard.com/protocol/)）

### Keepalive

若收到合法 transport data 但 `KEEPALIVE_TIMEOUT` 秒内无包可回，则发送零长度内层 IP 包的 keepalive（`msg.packet` 仅含 16 字节 Poly1305 标签）。（来源：[白皮书 §6.5](https://www.wireguard.com/papers/wireguard.pdf)）

### DiffServ / ECN

握手包 DSCP = 0x88（AF41），ECN = 00；传输数据包外层 DSCP = 0（不泄露内层信息），ECN 按 RFC 6040 与内层同步。（来源：[协议页面 — DiffServ Considerations](https://www.wireguard.com/protocol/)）

---



## DoS Cookies（MAC1 / MAC2 拒绝服务防护）



### 设计动机

Curve25519 点乘计算密集。若对所有入站握手包都执行 DH，攻击者可发动 CPU 耗尽攻击。WireGuard 要求**首条握手消息即完成 initiator 认证**，同时**认证前不为未认证包分配状态、不发送响应**。（来源：[白皮书 §5.1, §5.3](https://www.wireguard.com/papers/wireguard.pdf)）

### 双 MAC 机制

每条握手消息（initiation 与 response）均含 `mac1[16]` 与 `mac2[16]`。（来源：[协议页面 — DoS Mitigation](https://www.wireguard.com/protocol/)）

#### MAC1（始终必需）

```
mac1 = MAC(HASH(LABEL_MAC1 || responder_public_key), message_bytes_before_mac1)
```

- `LABEL_MAC1` = `"mac1----"`
- 发送方须知道接收方公钥才能计算有效 `mac1`
- **无论是否高负载，无效 mac1 一律丢弃，不响应** → 保持"沉默"特性
- 被动攻击者可猜测目标公钥，但无法仅凭 mac1 完成密码学证明（来源：[白皮书 §5.3](https://www.wireguard.com/papers/wireguard.pdf)）



#### MAC2（高负载时必需）

```
mac2 = MAC(cookie, message_bytes_before_mac2)   # 若持有有效 cookie
     = zeros                                    # 否则
```

- `cookie` 由服务器以每 2 分钟轮换的密钥对 initiator 源 IP 做 MAC，证明 IP 所有权
- 高负载时：有效 mac1 但无效/缺失 mac2 → 可回复 Cookie Reply，**不处理握手**
- cookie 有效期 2 分钟（120 秒）

（来源：[协议页面](https://www.wireguard.com/protocol/)；[白皮书 §5.3–5.4.4](https://www.wireguard.com/papers/wireguard.pdf)）

### Cookie Reply 消息（type=3）

```
message_type(3) | reserved(3) | receiver_index(4) | nonce(24) | encrypted_cookie(32)
```

生成逻辑（来源：[协议页面](https://www.wireguard.com/protocol/)）：

```
cookie = MAC(server_changing_secret_every_2min, initiator_ip_address)
encrypted_cookie = XAEAD(HASH(LABEL_COOKIE || responder_static_public), nonce, cookie, initiating_message_mac1)
```

改进点（相对 DTLS/IKEv2 cookie 机制，来源：[白皮书 §5.3](https://www.wireguard.com/papers/wireguard.pdf)）：

1. 未通过 mac1 认证则完全沉默，不发送 cookie
2. cookie 经 AEAD 加密传输，非明文
3. AEAD 附加数据绑定 initiating message 的 mac1，防止向 initiator 洪泛伪造 cookie reply

收到 Cookie Reply 后，peer **仅存储 cookie，等待** `REKEY_TIMEOUT` **重传定时器到期再重发**，不立即重发。（来源：[白皮书 §6.6](https://www.wireguard.com/papers/wireguard.pdf)）

### 首包重放与时间戳

因首包即认证 initiator，存在重放 initiation 迫使 responder 更换临时密钥、中断合法会话的风险。WireGuard 在 initiation 消息中加密传输 TAI64N 时间戳；responder 记录每个 peer 的最大时间戳，丢弃 ≤ 已记录值的包。（来源：[白皮书 §5.1](https://www.wireguard.com/papers/wireguard.pdf)）

---



## Key Types（密钥类型）

WireGuard 涉及三类密钥材料（来源：[白皮书 §5.2, §5.4](https://www.wireguard.com/papers/wireguard.pdf)；[Noise §9](https://noiseprotocol.org/noise.html)）：

### 1. Static Identity Keys（静态身份密钥）

- **类型**：Curve25519 长期密钥对（32 字节私钥 / 32 字节公钥）
- **角色**：peer 的永久身份标识；Cryptokey Routing 以静态公钥为索引
- **交换方式**：带外预交换（类 OpenSSH）
- **在握手中的作用**：
  - Initiator 在消息 1 中加密发送静态公钥（`s` token）
  - 参与 `ss`（双方静态）与 `es`/`se` DH 运算
- **存储**：接口配置 `PrivateKey`；peer 配置 `PublicKey`



### 2. Ephemeral Keys（临时密钥）

- **类型**：每次握手重新生成的 Curve25519 密钥对
- **角色**：提供完美前向保密；每次重协商更换
- **在握手中的作用**：
  - 消息 1、2 均发送明文临时公钥（`e` token）
  - 参与 `es`、`ee`、`se` DH
- **生命周期**：握手完成后与链密钥一并清零（来源：[白皮书 §5.4.5](https://www.wireguard.com/papers/wireguard.pdf)）



### 3. Pre-Shared Key（PSK，可选）

- **类型**：256 位（32 字节）对称密钥
- **角色**：在 `Noise_IKpsk2` 第二条消息末尾经 `MixKeyAndHash` 混入；为抗未来量子计算录音攻击提供额外对称层（混合方案，非完整后量子替换）
- **默认值**：未配置时为 32 字节全零（协议仍按 psk2 流程运行）
- **配置**：`wg set` / 配置文件中的 `PresharedKey` 字段
- **安全模型**：即使 PSK 泄露，Curve25519 仍提供保护；反之，若 Curve25519 被攻破，PSK 可保护历史流量（来源：[白皮书 §5.2](https://www.wireguard.com/papers/wireguard.pdf)）



### 4. Transport Session Keys（传输会话密钥，派生）

握手完成后由最终 `chaining_key` 经 HKDF 派生，分为 initiator/responder 各自的 sending_key 与 receiving_key，配合 64 位 counter 用于 ChaCha20Poly1305 封装 transport data。定期轮换（约 120 秒）。（来源：[协议页面 — Data Keys Derivation](https://www.wireguard.com/protocol/)；[白皮书 §6.2–6.3](https://www.wireguard.com/papers/wireguard.pdf)）

---



## A. Kernel Implementation Internals（内核实现内部）

本节补充白皮书 §7 中关于"短小、可审计、避免动态分配、与内核设施集成"的具体落实方式，便于答辩时回答"WireGuard 内核实现到底怎么做"。来源：[白皮书 §7.1–7.4](https://www.wireguard.com/papers/wireguard.pdf)。

### A.1 代码量与分层

- 整个 Linux 内核驱动 + Noise 状态机 + 数据平面 < 4,000 行 C（不含密码学原语），目标为可由单人审计。
- 模块自包含：通过 `wireguard-linux-compat` 包作为 out-of-tree 模块构建，不需修改核心内核；新版 Linux 已将其合并至主线 `drivers/net/wireguard/`。



### A.2 接收路径"零动态分配"原则

白皮书 §7.2 把"接收入站包时不做 `kmalloc`"列为硬性目标，实现方式：

1. 入站 `skb` 在 socket 层就已分配，握手处理复用同一 `skb` 的 `cb`（control buffer）字段存握手临时状态（`PACKET_CB(skb)`）。
2. 解密的临时缓冲（Blake2s/HKDF/AEAD 上下文）使用栈上对齐数组（如 `u8 output[NOISE_HASH_LEN + 1] __aligned(...)`），处理完即 `memzero_explicit` 清零。
3. 加解密走 `padata`（parallel async crypto），每个 CPU 一个 `crypt` 工作队列，避免在收包软中断上下文同步执行重负载 AEAD。



### A.3 并发与队列结构

每个 `wg_peer` 维护若干独立队列与定时器，多数路径用 lockless / RCU / `spin_lock_bh`：


| 结构                                                   | 用途                          | 同步方式                                       |
| ---------------------------------------------------- | --------------------------- | ------------------------------------------ |
| `peer->handshake.send_queue`                         | 暂存待发握手包                     | lockless ring                              |
| `peer->tx_queue` / `peer->rx_queue`                  | 数据包队列                       | per-peer NAPI 风格                           |
| `device->incoming_handshakes`                        | 收到的握手包批量                    | `MAX_QUEUED_INCOMING_HANDSHAKES = 4096` 上限 |
| `device->handshake_send_wq` / `handshake_receive_wq` | 加解密工作队列                     | `padata` 并行                                |
| `device->index_hashtable`                            | 索引→handshake/keypair 反查     | RCU + spin_lock_bh                         |
| `device->peer_hashtable`                             | 公钥哈希→peer 反查                | RCU                                        |
| `peer->keypairs`                                     | next/current/previous 三元密钥对 | `spin_lock_bh` + RCU 指针                    |




### A.4 三元 keypair 模型

每个 peer 同时持有三组密钥对，使平滑切换不丢包（来源：仓库 `noise.c` `add_new_keypair()`）：

- `previous_keypair`：上一对，仍可解对方延迟到达的旧 counter 包。
- `current_keypair`：正在使用。
- `next_keypair`：握手已完成但**未确认**，responder 等到收到 initiator 的第一条 transport data 才晋升为 current（即 §密钥确认 实现）。

切换规则（initiator 在收到响应后直接将新 keypair 置为 current；responder 等到对方首包确认才晋升 next）保证：即便在重协商窗口内双方时钟略有错位，也不会丢包或重复密钥。

### A.5 索引哈希表

握手消息中的 32 位 `sender_index` / `receiver_index` 类似 IPsec SPI：发送方分配，接收方据此 O(1) 反查 handshake 或 keypair。`index_hashtable` 用 RCU 读路径 + 写侧 `spin_lock_bh`，查询在收包软中断里无锁完成。

### A.6 RTNL、网络命名空间与虚拟接口

- `wg0` 是标准 `struct net_device`，受 RTNL 保护，支持 `ip link`、namespace 迁移、容器化。
- 加解密与定时器回调使用 `rcu_read_lock_bh()`，避免与 RTNL 互锁；接口 down 时由 `wg_timers_stop()` 同步删除全部 peer 定时器。
- `MAX_PEERS_PER_DEVICE = 1 << 20`：单接口最多 ~100 万 peer，由 `peer_hashtable` 容量上限设定。



### A.7 性能要点

白皮书 §7.4 报告：单核 3 Gbps+（ChaCha20Poly1305），多核线性扩展，延迟 < IPsec IKEv2 + ESP。关键：`padata` 并行、零拷贝目标、per-CPU crypto tfm、避免 RTNL 在数据路径上。

---



## B. Timer State Machine（定时器状态机）

本节把白皮书 §6 + 仓库 `timers.c` 的真实实现整理成可答辩的状态机描述。来源：[白皮书 §6.1–6.6](https://www.wireguard.com/papers/wireguard.pdf)；仓库 `timers.c` 注释、`messages.h` `enum limits`。

### B.1 五个 peer 级定时器

每个 peer 由 `wg_timers_init()` 注册 5 个 `timer_list`：


| 定时器                          | 触发函数(到期)                               | 由谁武装                                           | 默认延迟                                                         |
| ---------------------------- | -------------------------------------- | ---------------------------------------------- | ------------------------------------------------------------ |
| `timer_retransmit_handshake` | `wg_expired_retransmit_handshake`      | `wg_timers_handshake_initiated`                | `REKEY_TIMEOUT + jitter` = 5s + 0~333ms                      |
| `timer_send_keepalive`       | `wg_expired_send_keepalive`            | `wg_timers_data_received`                      | `KEEPALIVE_TIMEOUT` = 10s                                    |
| `timer_new_handshake`        | `wg_expired_new_handshake`             | `wg_timers_data_sent`                          | `KEEPALIVE_TIMEOUT + REKEY_TIMEOUT` + jitter = 15s + 0~333ms |
| `timer_zero_key_material`    | `wg_expired_zero_key_material`         | `wg_timers_session_derived`                    | `REJECT_AFTER_TIME * 3` = 540s                               |
| `timer_persistent_keepalive` | `wg_expired_send_persistent_keepalive` | `wg_timers_any_authenticated_packet_traversal` | 用户配置 `PersistentKeepalive` 秒（0=关）                            |


`REKEY_TIMEOUT_JITTER_MAX_JIFFIES = HZ/3`（≈333ms）是抖动，防止同步重传风暴。

### B.2 重要事件钩子（非定时器，由收发路径调用）


| 钩子                                             | 何时调用                    | 作用                                                                      |
| ---------------------------------------------- | ----------------------- | ----------------------------------------------------------------------- |
| `wg_timers_data_sent`                          | 每发一条 authenticated data | 武装 new_handshake（迟迟未回则重发握手）                                             |
| `wg_timers_data_received`                      | 每收一条 authenticated data | 武装 send_keepalive（被动保活）                                                 |
| `wg_timers_any_authenticated_packet_sent`      | 发出任意认证包                 | 取消 send_keepalive（已有出站，不必保活）                                            |
| `wg_timers_any_authenticated_packet_received`  | 收到任意认证包                 | 取消 new_handshake（对端还活着）                                                 |
| `wg_timers_handshake_initiated`                | 发完 initiation           | 武装 retransmit_handshake                                                 |
| `wg_timers_handshake_complete`                 | 握手完成                    | 取消 retransmit、清零 `timer_handshake_attempts`、写 `walltime_last_handshake` |
| `wg_timers_session_derived`                    | 派生新会话密钥                 | 武装 zero_key_material（540s 后仍无新密钥则销毁所有密钥）                                |
| `wg_timers_any_authenticated_packet_traversal` | 发/收任意认证包                | 若配置了 `PersistentKeepalive`，重武装 persistent_keepalive                     |




### B.3 重传与"放弃"边界

`MAX_TIMER_HANDSHAKES = 90 / REKEY_TIMEOUT = 18`。即第 19 次(>18)仍无 ack 时调用：

- `wg_packet_purge_staged_packets(peer)`：丢弃所有暂存出站包。
- 武装 `timer_zero_key_material` 在 `REJECT_AFTER_TIME * 3 = 540`s 后清零密钥。
- 期间对端若恢复并主动来包，则可握手复活。



### B.4 主动重协商（与上面的"会话死亡重发"不是同一码事）

白皮书 §6.2–6.3 区分两套触发：


| 触发点                                        | 常量                                                  | 语义             |
| ------------------------------------------ | --------------------------------------------------- | -------------- |
| `REKEY_AFTER_TIME`（白皮书原值 120s）             | 已发数据且当前 keypair 出生至今 > 120s → **initiator** 主动发起新握手 | 主动轮换，保证 PFS 周期 |
| `REKEY_AFTER_MESSAGES`（2^60）               | 已发数据包数超过该值 → initiator 重换密钥                         | 防止 counter 撞上限 |
| `REJECT_AFTER_TIME`（180s）                  | keypair 出生 > 180s → 该 keypair 不再用于发送，丢弃后续旧包         | 强制废弃           |
| `REJECT_AFTER_MESSAGES`（2^64 − window − 1） | counter 超此值 → keypair 失效                            | 防止 counter 回绕  |


**initiator/responder 不对称**（白皮书 §6.3，避免双方同时发起）：以原 WG 而言，responder 在 `REKEY_AFTER_TIME` 后仍**不主动**发起；只有当 `KEEPALIVE_TIMEOUT + REKEY_TIMEOUT` 内一直未收到对端任何包（会话死亡征兆）时，由 timer_new_handshake 触发新一轮。这种"主动方先动、被动方响应"的设计使重协商不会撞车。

> 说明：本仓库 `messages.h` 把 `REKEY_AFTER_TIME` 改为 5（出于 PQ 实验频繁重协商需求），白皮书原值为 120，二者不要混淆。



### B.5 速率限制与队列上限（常被忽略的 DoS 阀门）

`messages.h` `enum limits` 中除 rekey 之外，还定义了硬阀门，是 DoS 与资源耗尽保护的真实闸门：


| 常量                               | 值    | 作用                                                                                    |
| -------------------------------- | ---- | ------------------------------------------------------------------------------------- |
| `INITIATIONS_PER_SECOND`         | 100  | 每个 peer 接收 initiation 不超过 100/s（`flood_attack` 判断使用，见 `noise.c` `consume_initiation`） |
| `MAX_TIMER_HANDSHAKES`           | 18   | 单次握手最多重试次数                                                                            |
| `MAX_QUEUED_INCOMING_HANDSHAKES` | 4096 | 全设备排队待处理握手包上限                                                                         |
| `MAX_STAGED_PACKETS`             | 128  | 每 peer 暂存出站包上限（无 keypair 时）                                                           |
| `MAX_QUEUED_PACKETS`             | 1024 | 每 peer 收发队列上限                                                                         |
| `MAX_PEERS_PER_DEVICE`           | 2^20 | 单接口 peer 数上限                                                                          |




### B.6 时间戳防重放与"轻量降计时"

TAI64N 时间戳精度刻意降低到 `NSEC_PER_SEC / INITIATIONS_PER_SECOND` 的 2 的幂对齐（见 `tai64n_now()`），既防时序侧信道，又与 `INITIATIONS_PER_SECOND=100` 的速率限制吻合，使合法 initiator 不会因自身时钟精度而触发 `flood_attack` 误判。

---



## C. Noise Derivation Step-by-Step（Noise 派生逐步公式）

本节列出 WireGuard `Noise_IKpsk2_25519_ChaChaPoly_BLAKE2s` 中 `chaining_key ck` 与 `hash h` 在每个 token 后的精确演化公式，并区分 `MixHash` / `MixKey` / `MixKeyAndHash` 的语义。来源：[Noise §5–9](https://noiseprotocol.org/noise.html)；[白皮书 §5.4](https://www.wireguard.com/papers/wireguard.pdf)；仓库 `noise.c` `mix_hash` / `mix_psk` / `kdf` 实现对应。

### C.1 三个 Mix 原语的语义差异（容易答错的点）


| 原语                   | 公式                                                                          | 对 ck 的影响 | 对 h 的影响 | 输出 key？      |
| -------------------- | --------------------------------------------------------------------------- | -------- | ------- | ------------ |
| `MixHash(input)`     | `h = HASH(h ‖ input)`                                                       | 无        | 是，更新 h  | 否            |
| `MixKey(input)`      | `(ck, k) = HKDF(ck, input, 2)`：`ck = HMAC(ck, 0x01 ‖ ...)`, `k = ...` 第二个输出 | 是，更新 ck  | 否       | 是（用于后续 AEAD） |
| `MixKeyAndHash(psk)` | `(ck, h, k) = HKDF(ck, psk ‖ h_prev, 3)` 三个输出分别给 ck、h、k                     | 是        | 是，h 也更新 | 是            |


`MixKeyAndHash` 把 PSK 同时灌进 ck（影响后续所有密钥）与 h（进入握手 transcript），因此 PSK 错误既会破坏传输密钥又会破坏 transcript，使握手立刻失败——这正是 psk2 模式的设计意图。

### C.2 初始化（两步）

```
ck0 = HASH("Noise_IKpsk2_25519_ChaChaPoly_BLAKE2s")
h0  = HASH(ck0 ‖ "WireGuard v1 zx2c4 Jason@zx2c4.com")
h1  = HASH(h0 ‖ ResponderStaticPublicKey)        # 预消息：IK 中 responder 静态公钥已知
```

注意 `CONSTRUCTION` 字符串本身先入 ck（不是 h），随后 `IDENTIFIER` 与 ck 一起入 h，再混入 responder 静态公钥——只有 responder 静态公钥入 h，不入 ck（这一点在原 Noise IK 规范中需要特别留意，是 WireGuard 的具体化决定）。

### C.3 消息 1（initiator → responder）：tokens `e, es, s, ss`

设 `e_i = initiator ephemeral pub`，`DH(a,b) = Curve25519(a_priv, b_pub)`。

```
# e token
h  = HASH(h ‖ e_i)                           # 明文 e_i 进入 transcript
(ck, k0) = HKDF(ck, e_i, 2)                  # MixKey

# es token (init ephemeral × responder static)
(ck, k1) = HKDF(ck, DH(e_i_priv, S_r_pub), 2)

# s payload: initiator 静态公钥 S_i_pub 用 k1 AEAD 加密
encrypted_static = AEAD_Enc(k1, 0, S_i_pub, h)   # h 作 AD
h  = HASH(h ‖ encrypted_static)              # MixHash（ciphertext 入 transcript，非明文）

# ss token (init static × responder static)
(ck, k2) = HKDF(ck, DH(S_i_priv, S_r_pub), 2)

# t payload: TAI64N 时间戳用 k2 AEAD 加密
encrypted_timestamp = AEAD_Enc(k2, 0, tai64n_now, h)
h  = HASH(h ‖ encrypted_timestamp)

# mac1 / mac2 计算（覆盖到 mac1 之前的全部字节）
mac1 = MAC(HASH("mac1----" ‖ S_r_pub), message[0..end-before-mac1])
mac2 = MAC(cookie, message[0..end-before-mac2]) or zeros
```

> WireGuard 对标准 Noise IK 的"协议层 hack"：消息 1 的第二个加密 payload（对应 `s` 后的 transcript 内容）本应是别的 application data，WireGuard 在这里塞入 12 字节 TAI64N 时间戳，并省去消息 2 中本应放在末尾的额外载荷。WireGuard 仍是符合 IK 的"e,es,s,ss"模式，只是把 application payload 替换为协议层字段。



### C.4 消息 2（responder → initiator）：tokens `e, ee, se, psk`

```
# e token
h  = HASH(h ‖ e_r)
(ck, k3) = HKDF(ck, e_r, 2)

# ee token (ephemeral × ephemeral)
(ck, k4) = HKDF(ck, DH(e_r_priv, e_i_pub), 2)

# se token (responder ephemeral × initiator static)
(ck, k5) = HKDF(ck, DH(e_r_priv, S_i_pub), 2)

# psk token (psk2)：在消息末尾混入
(ck, h, k6) = HKDF(ck, psk ‖ h_prev, 3)      # MixKeyAndHash
# 注意 h_prev 是注入 psk 前的 h；新 h 与 k6 都从此派生

# 空 payload AEAD：确认密钥
encrypted_nothing = AEAD_Enc(k6, 0, "", h)
h  = HASH(h ‖ encrypted_nothing)
```



### C.5 传输密钥派生（Split 阶段）

```
(send_key, recv_key) = HKDF(ck_final, "", 2)  # 仓库 derive_keys 中的 kdf 第二次调用
# 双方互为镜像：initiator 的 send = responder 的 recv，反之亦然
```

随后所有临时密钥（`e_i_priv`, `e_r_priv`, ck, h 中间态）`memzero_explicit` 清零；只有 send_key/recv_key 与 keypair metadata 存入 `peer->keypairs.current_keypair`。

### C.6 AEAD nonce 与 counter

- Noise 中每条消息用 nonce=0（因为每次都换了一个新派生的 key），protocol 层"counter" 在 transport data 才启用。
- Transport data 的 64-bit counter 小端拼接成 12 字节 ChaCha20Poly1305 nonce：前 4 字节点零，后 8 字节填 counter。这也解释了为何 counter 至多 2^64-1 而 nonce 字段是 96 位。
- `REJECT_AFTER_MESSAGES = U64_MAX - COUNTER_WINDOW_SIZE - 1`：留给滑动窗口余量，避免 counter 接近 2^64 时无法判定重放。

---



## F. Comparison with IPsec / OpenVPN（与 IPsec/OpenVPN 对比）

本节把白皮书 §1、§7.4 中"为什么 WireGuard 简单、安全、快"的论证整理成答辩可直接引用的对照。来源：[白皮书 §1, §7.4](https://www.wireguard.com/papers/wireguard.pdf)。

### F.1 IPsec（IKEv2 + ESP）对照


| 维度     | IPsec (IKEv2 + ESP)                      | WireGuard                           |
| ------ | ---------------------------------------- | ----------------------------------- |
| 密码套件协商 | 大量 Suite，加密算法、PRF、DH group 协商            | 无协商，固定套件                            |
| 状态机    | IKE SA + Child SA 双层，多消息交换               | 1-RTT 握手 + keypair 三元               |
| 内核设施   | `xfrm` 变换层、policy/SA 数据库、`ip xfrm` 工具    | 单个 `wg0` 虚拟接口，应用层 Cryptokey Routing |
| 端口/协议  | IP proto 50 (ESP) + UDP 500 (IKE)，防火墙常被卡 | 单个 UDP 端口                           |
| NAT 穿越 | 协商决定，UDP-encapsulate ESP                 | 内建 roam + endpoint 学习               |
| 配置复杂度  | 多参数（lifetime, PFS, mode, transform...）   | 公钥 + AllowedIPs                     |
| 代码量    | 数十万行（strongSwan 等）                       | < 4,000 行                           |
| 攻击面    | 协商协议本身历史漏洞（CVE 数十）                       | 无协商协议                               |


白皮书关键论证：**密码敏捷性是双刃剑**——协商协议带来复杂性与降级攻击面；WG 通过固定套件 + 整体升级换取攻击面缩减。若某原语被破，所有端点统一升级即可；这与 IPsec 的"逐端点协商兼容"形成取舍。

### F.2 OpenVPN 对照


| 维度      | OpenVPN                        | WireGuard             |
| ------- | ------------------------------ | --------------------- |
| 协议      | TLS over TCP 或 UDP，自定义 SSL VPN | UDP + Noise           |
| 身份认证    | X.509 证书链 / 用户名密码 / 双因子        | 32 字节公钥（类 SSH）        |
| 实现层     | 用户态 + tun/tap 设备               | 内核态 native net_device |
| TLS 开销  | 完整 TLS handshake（重）            | 1-RTT（轻）              |
| Agility | 完全敏捷（任选 cipher）                | 完全不敏捷                 |
| 攻击面     | TLS 历史漏洞继承                     | 无 TLS 栈               |
| 性能      | 用户态 ↔ 内核拷贝开销，受限于 TLS           | 内核零拷贝目标，<4k 行         |


白皮书 §1 强调：OpenVPN 因继承 TLS 而继承了 TLS 的全部 attack surface（Heartbleed、POODLE、降级套件等）。WireGuard 去掉协商，等于"把 TLS 攻击面整体砍掉"。

### F.3 接口与路由模型的简化

- IPsec：`xfrm` policy + SA + selector + template，需理解"phase 1 / phase 2"概念。
- OpenVPN：tun 设备 + 用户态路由 + TLS。
- WireGuard：仅一个虚拟 L3 接口 `wg0`，公钥 ↔ AllowedIPs 二元绑定，防火墙判据简化为"该接口/该 IP 是否可信"。

白皮书 §7.4 同时给出实测：在 16 核机器上 WireGuard 单流 ~3 Gbps，多流线性扩展，比 IPsec (IKEv2 + ESP - AES-GCM) 高约 15%；比 OpenVPN（TLS + tun）高出数倍。

### F.4 答辩可用一句话总结

> WireGuard 把"密钥协商 + L3 加密"合并为单一机制，废弃了 IPsec 的 xfrm 分层与 OpenVPN 的 TLS 重栈，用"无敏捷性 + 内核态虚拟接口"换来可审计的 ~4 千行代码与更小的攻击面。

---



## G. Multi-platform Implementations（多平台实现）

来源：[Quick Start](https://www.wireguard.com/quickstart/)；白皮书 §7.1。

### G.1 三类实现


| 名称                                      | 形态                        | 适用平台                                               |
| --------------------------------------- | ------------------------- | -------------------------------------------------- |
| `wireguard-linux`                       | Linux 内核模块（C，<4k 行）       | Linux 主线（≥5.6）与 `wireguard-linux-compat` backport  |
| `wireguard-go`                          | Go 用户态实现（封装相同协议）          | macOS, Windows（仅 fallback）, BSD, iOS, Android（非主选） |
| `wireguard-windows` / `wireguard-apple` | 原生 C/Rust 用户态 + 平台 TUN 驱动 | Windows（`wintun`）、Apple（`utun`）                    |


注意：三大平台各自的"内核态"形态不同——Linux 有真内核模块；Windows 用 NDIS 上的 `wintun` 用户态驱动；macOS/iOS 用 `utun`（System Extension）。这三套实现**协议完全相同**，差异只在 TUN 设备与 socket API。

### G.2 共用的密码学后端

- Linux：内核 `crypto` 子系统（`chacha20poly1305`、`blake2s`、`curve25519`、`hkdf`）。
- `wireguard-go`：Go crypto 标准库（`crypto/chacha20poly1305`、`golang.org/x/crypto/curve25519`）。
- `wireguard-apple` / `wireguard-windows`：各自平台的 crypto 框架（CommonCrypto, Windows CNG）。

协议规范不变，密码学调用接口由各平台按需适配。

### G.3 跨平台配置一致性

所有平台共用同一份 `wg(8)` / `wg-quick(8)` 配置文件格式（`[Interface]` + `[Peer]`）。这正是为什么 PQ-SecTunnel 可以直接重命名 `wg`/`wg-quick` 为 `pqst`/`pqst-quick` 而无需改用户语义（见 CLAUDE.md "Experiment directory structure"）。

---



## H. UAPI Protocol（用户态 API 协议）

`wg(8)` 工具与内核模块之间是用文本协议通信；理解 UAPI 有助于读懂 PQ-SecTunnel 中 `uapi/pqwireguard.h` 的属性前缀改写。来源：WireGuard UAPI 文档（README 中"Kernel Module API" 节）；仓库 `uapi/pqwireguard.h`、`netlink.c`。

### H.1 通信方式

- 内核打开一个 `AF_LOCAL` socket（每个 netdev 一个），绑定在 `/var/run/wireguard/wg0.sock`。
- 用户态经 `connect()` + `sendmsg()`/`recvmsg()` 与之交互，**不是 Generic Netlink**。
- 消息以换行符结尾的字段流，键值对形式。



### H.2 一次 `set` 会话示例

```
set=1
private_key=<base64 32 bytes>
listen_port=51820

peer_public_key=<base64 32 bytes>
endpoint=192.95.5.69:51820
allowed_ip=10.192.122.3/32
allowed_ip=10.192.124.1/24
persistent_keepalive_interval=25

replace_peers=true
```

每个字段的含义与对应内核字段：


| UAPI 字段                         | 对应内核/`wg`配置                  | 类型           |
| ------------------------------- | ---------------------------- | ------------ |
| `private_key`                   | `[Interface] PrivateKey`     | base64       |
| `listen_port`                   | `[Interface] ListenPort`     | u16          |
| `fwmark`                        | `wg set ... fwmark`          | u32          |
| `peer_public_key`               | `[Peer] PublicKey`（用于索引）     | base64       |
| `preshared_key`                 | `[Peer] PresharedKey`        | base64 32 字节 |
| `endpoint`                      | `[Peer] Endpoint`            | `IP:Port`    |
| `allowed_ip`                    | `[Peer] AllowedIPs`（多条）      | CIDR         |
| `persistent_keepalive_interval` | `[Peer] PersistentKeepalive` | u16, 秒；0=关   |
| `replace_peers`                 | 是否清空现有 peer 列表               | bool         |
| `remove_peer`                   | 删除 peer                      | bool         |




### H.3 一次 `get` 会话

```
get=1

# 内核回包：
private_key=...
listen_port=51820
fwmark=0
rx_bytes=...
tx_bytes=...
...
peer_public_key=...
endpoint=...
allowed_ip=...
last_handshake_time_sec=...
last_handshake_time_nsec=...
rx_bytes / tx_bytes / persistent_keepalive_interval
```

`last_handshake_time_*` 来自 `peer->walltime_last_handshake`（`wg_timers_handshake_complete()` 写入）。

### H.4 与 PQ-SecTunnel 的对应

PQ-SecTunnel 在 `uapi/pqwireguard.h` 中保留同一 UAPI 框架，只对外改前缀（例如 `PQSTPEER_A_PERSISTENT_KEEP_ALIVE_INTERVAL`、`PQST_A_*`）；UAPI 文本字段名（`persistent_keepalive_interval` 等）与原 WG 兼容，使 `wg-quick`（即 `pqst-quick`）无需修改解析逻辑。

---



## I. Key Derivation Details（密钥派生细节）

补充"密钥类型"节未展开的派生细节与边界。来源：[白皮书 §5.2, §5.4](https://www.wireguard.com/papers/wireguard.pdf)；[协议页面 — Data Keys Derivation](https://www.wireguard.com/protocol/)；仓库 `noise.c` `derive_keys` / `mix_psk` / `add_new_keypair`。

### I.1 双向密钥的镜像关系

握手末尾 Split 阶段：

```
(send, recv) = HKDF(ck_final, "", 2)
```

- **initiator 的 send = responder 的 recv**（同一份 32 字节）。
- 反之 initiator 的 recv = responder 的 send。
- 每个方向独立 nonce / counter 空间，互不影响。

这与 IPsec 的 inbound/outbound SA 概念等价：每个 keypair 同时充当"我的发送 SA"与"对方的接收 SA"。

### I.2 PSK 全零的语义

未配置 PSK 时，`preshared_key` 取 32 字节全零，**协议仍按 psk2 流程执行** `MixKeyAndHash`：

- 全零 PSK 不提供量子攻击保护，但仍使 `ck` 演化路径与有 PSK 时形式一致（避免两套代码路径）。
- 安全属性降级为"纯 Curve25519 + AEAD"——这正是为何 WG 文档称"PSK 提供**额外**对称层，而非完整 PQC 替换"。



### I.3 PSK 泄露 / DH 被破的组合保护

白皮书 §5.2 给出"混合方案"论证：

- 若 PSK 泄露，Curve25519 仍提供 PFS 保护历史流量。
- 若 Curve25519 被量子破解，PSK（32 字节对称密钥）抵抗量子攻击，保护历史录音。
- 二者**任一**安全 → 历史流量安全；二者**同时**被破才失守。

这是 WireGuard 唯一与后量子相关的官方机制——也解释了 PQ-SecTunnel 为什么要"把 PSK 替换为正式 KEM"：把这种可选的混合保护提升为强制的 KEM 协商。

### I.4 接口私钥 vs peer 公钥存放规则

- 接口（`[Interface]`）只持有自己的 `static_private`（仅生成本机 Identity），不在 peer 上存私钥。
- peer（`[Peer]`）只持有对方 `PublicKey`、`Endpoint`、`AllowedIPs`、`PresharedKey`。
- 这种非对称性阻止"私钥随 peer 配置分发"——是 WG 的一个隐式安全惯例。



### I.5 Curve25519 私钥的 clamp 与公钥校验

原 WireGuard 用 `curve25519_clamp_secret()` 对私钥做"clamping"（清最低 3 位、设最高位为 0、次高为 1，使其成定长 group element），再 `curve25519_generate_public()` 导出公钥。若用户传入非法私钥（如全零），`has_identity` 标为 false，接口拒发握手——这是 WG 默默的输入校验，常常被忽略。

### I.6 keypair 生命周期与 RCU 销毁

每个 `noise_keypair` 用 `kref` 引用计数 + `call_rcu` 延迟释放：

- `wg_noise_keypair_get()` 必须在 `rcu_read_lock_bh()` 下取引用。
- `wg_noise_keypair_put(old, true)` 同时移出索引哈希表并降 refcount → 0 时 RCU 延迟 `kfree_sensitive`。
- 这种设计保证收包路径在 RCU 读侧拿到 keypair 时不会被并发释放。

---



## J. Other Whitepaper Notes & Corrections（白皮书其余要点与一处修正）



### J.1 §2.1 Roaming 的密码学边界

白皮书 §2.1 明确："外层 `(IP, Port)` 与加密载荷**无密码绑定**"——源地址可被中间人改写，但只能造成 DoS/断连，**不能解密或注入**。这意味着 WG 的 Roaming **不**提供 anti-MITM，仅靠身份公钥认证抵抗 MITM 内容伪造。

### J.2 §5.1 首包认证的代价

"首包即认证 initiator"带来两个代价：

- 重放风险 → TAI64N 时间戳缓解。
- 服务器必须在知道哪个 peer 之前做 `DH(es)` 解密静态公钥——这导致服务器在无法识别 peer 时也付出一次 Curve25519 点乘。WG 用 MAC1（要求知道服务器公钥才计算）作为前置过滤，避免无脑 DH。



### J.3 §6.6 Cookie Reply 后不立即重发

收到 Cookie Reply 后，客户端**不立即重发** initiation，而要等 `REKEY_TIMEOUT` 定时器到期——这是 WG 防止"cookie reply 反射放大攻击"的细节：恶意 cookie 无法强迫客户端实时回应数据。

### J.4 §4 可选 Endpoint 与 DNS 解析

原 WG 内核**不做 DNS 解析**——`Endpoint` 必须是 IP:Port 形式；DNS 名解析由 `wg-quick` 在用户态一次性 `getent` 后下发。这是为什么 `wg setconf` 直接读 DNS 名会失败，而 `wg-quick` 能用域名。PQ-SecTunnel 继承同一行为。

### J.5 ⚠️ 重放滑动窗口的精确大小（修正现有文档）

本文档前文 "Data Plane — 重放防护" 一节称"约 2000 个 prior counter 值"，**该数值不准确**。仓库 `messages.h` `enum counter_values` 给出真实实现值：

```c
COUNTER_BITS_TOTAL      = 8192        // 位图总位数
COUNTER_REDUNDANT_BITS  = BITS_PER_LONG  // 64-bit 平台为 64，32-bit 平台为 32
COUNTER_WINDOW_SIZE     = COUNTER_BITS_TOTAL - COUNTER_REDUNDANT_BITS
                          // 64-bit 平台 = 8128
```

即滑动窗口**覆盖最近 8192 个 counter**，其中有效"历史可接收"窗口至少为 `COUNTER_WINDOW_SIZE`（64-bit 平台 8128）。原白皮书 §5.4.6 提到 RFC 6479 滑动窗口，但实际位图大小是 8192 位，并非 2000 余。

### J.6 §7.4 性能数据

白皮书 §7.4 提供：在 Intel i7-8650U（4 核 8 线程）+ 10 Gb 链路上，单流 ~~3 Gbps、4 流 ~6.5 Gbps、16 流 ~9.3 Gbps；与 IPsec (AES-NI + AES-GCM) 同档，比 OpenVPN（TLS + tun）约高 3~~5 倍。报告关键瓶颈：UDP 软中断开销与 `skb` 拷贝，而非密码学。

### J.7 §1 " menos es más " 设计哲学

白皮书 §1 的核心主张：**"协议复杂度是安全性的敌人"**——所有 cortex 都建立在固定套件 + 短代码 + 无协商之上。理解这一点对答辩阐述 PQ-SecTunnel 立场至关重要：PQ 改造**坚持了 WG 哲学**（仍无 cipher agility，仍单套件，仍 < 5k 行），只把 IKpsk2 → IKpsk1kem，把 PSK → KEM。

---



## PQ-SecTunnel 改造：4 次 ECDH → 3 次 KEM + PSK 顶替

承接前文 §"4 次 ECDH 各自绑什么"。PQ-SecTunnel 把原版 WG 的 IKpsk2 改造为 IKpsk1kem，关键决策就发生在"ss 怎么办"这一步。

### 原版 WG 4 次 vs PQ-SecTunnel 3 次 KEM 总对照


| 顺序         | 原版 WG IKpsk2             | PQ-SecTunnel IKpsk1kem        | 绑定类型                    | 谁做              |
| ---------- | ------------------------ | ----------------------------- | ----------------------- | --------------- |
| ① es（消息 1） | `DH(e_i^priv, S_r^pub)`  | `ct_1 = KEM.Enc(S_r^pub)`     | A 单向绑 responder 静态      | initiator 主动    |
| ② ss（消息 1） | `DH(S_i^priv, S_r^pub)`  | **❌ 删除**                      | **B 双方静态互锁**            | initiator（ECDH） |
| ③ ee（消息 2） | `DH(e_r^priv, e_i^pub)`  | `ct_2 = KEM.Enc(E_i^pub)`     | A 单向绑 initiator 临时→本次会话 | responder（KEM）  |
| ④ se（消息 2） | `DH(e_r^priv, S_i^pub)`  | `ct_3 = KEM.Enc(S_i^pub)`     | A 单向绑 initiator 静态      | responder（KEM）  |
| 额外补偿       | `psk2` 末尾混 PSK（optional） | `psk1` 前移混 PSK（顶替 ss 的 B 类作用） | **B** 双方互锁              | 双方事先共享          |


PQST 数字：**3 次 KEM + 1 次 PSK 混入**，与原版 **4 次 ECDH** 密码学上不等价。

### 为什么 KEM 不能"一对一替换 ss"——单向 vs 对称

KEM 与 ECDH 的代数结构差异决定了改造方式：


| 操作                     | ECDH (Curve25519)            | KEM (ML-KEM, McEliece)           |
| ---------------------- | ---------------------------- | -------------------------------- |
| 谁能用对方公钥算 shared_secret | **双方都能**：`A私×B公 = B私×A公`（对称） | encap 一方需对方公钥，decap 一方需自己的私钥（单向） |
| 一次操作得几份共享秘密            | 1 份，双方等价                     | 1 份，单向（encap 侧主动）                |
| "双方静态互绑"做几次            | **1 次 DH 运算 + 0 字节握手载荷**     | **2 次 KEM**（每个方向各一次，各发一份 ct）     |


`ss` 在 ECDH 中是"免费"绑一次双方静态互锁——运算在本地做，shared_secret 直接灌进 ck，握手包里**不传任何额外字节**。换成 KEM 要达到同样的 B 类互锁，需要双方各 encap 一次、各发一份 ct，握手包就得多 768~6200 字节（McEliece 更甚）。

PQST 主套件现状（来自 KE.md + dmesg 实证）已逼近报文上限：

- 消息 1 initiation = 2068 字节（含 ct_1 ≈ 768 字节）→ 已超 MTU，必须应用层分片为 2 片
- 消息 2 response = 1916 字节（含 ct_2 + ct_3 共 ~1536 字节）→ 已超 MTU，必须应用层分片为 2 片

若再加 ct_4（responder 用 initiator static 再做一次 encap 重复绑定），response 就要超过 3000 字节；如果再 append ct_5（initiator 端再回一个 encap），initiation 超 4000 字节。**分片负担和可靠性就崩了**，这是非常工程化的取舍。

### PQST 三个 KEM 各自只能做 A 类，做不了 B 类

关键事实：**KEM 是单向非对称的**——`KEM.Enc(pk)` 任何人都能做（pk 是公开的），只有 `KEM.Dec(sk, ct)` 需要私钥。


| KEM                       | 谁能产生 shk            | 谁能验证 shk                 | 绑定类型                   |
| ------------------------- | ------------------- | ------------------------ | ---------------------- |
| `ct_1 = KEM.Enc(S_r^pub)` | 任何人（S_r^pub 公开）     | 只有 responder（用 S_r^priv） | **A 单向绑 responder 静态** |
| `ct_2 = KEM.Enc(E_i^pub)` | 任何人（E_i^pub 握手包里明文） | 只有 initiator（用 E_i^priv） | A 单向绑 initiator 临时     |
| `ct_3 = KEM.Enc(S_i^pub)` | 任何人（S_i^pub 公开）     | 只有 initiator（用 S_i^priv） | **A 单向绑 initiator 静态** |


**没有一个 KEM 操作是"同时依赖双方私钥才能产生"的**——攻击者能伪造合法 ct_1/ct_2/ct_3，因为他用公开公钥就能 encap 出来，只是解不开而已。所以 ct_1+ct_3 能完成 A 类全覆盖（双方静态各自被单向绑进 ck），但**不能阻止伪造投递**。

ECDH 的 `ss` 不同：攻击者**连投递都做不出合法的 ss shared_secret**——ss 依赖双方私钥，他两边都没有。这就是 B 类互锁的实质保护。

### PSK 顶替的就是 ss 的 B 类作用

PSK 是对称密钥，双方事先带外共享，只有真双方知道。KE.md 消息 1 第 12 步：

```
C_4 = KDF(C_3, S_i^pub)
(C_5, K_2) = KDF(C_4, H_4 ‖ psk)   # mix_psk
```

把 PSK 经 `MixKeyAndHash` 混入 ck——攻击者若不知 PSK，他在这一步派生出的 `C_5` 就与真双方不同，后续 K_2 不同，timestamp AEAD 解密失败 → 握手立刻失败。


| 共享秘密 | 产生需要什么   | 验证需要什么           | 绑定类型       |
| ---- | -------- | ---------------- | ---------- |
| PSK  | 双方事先带外共享 | **双方都持有同一份对称密钥** | **B 双方互锁** |


PSK 提供的属性恰好就是 ECDH `ss` 提供的 B 类互锁：要算出（或验证）这份进 ck 的秘密，**必须真双方配合**。这是 PSK 在 PQST 中承担的"补偿被砍掉的 ss"的密码学角色。

### 整理成总表


| 绑定对象             | 原版 WG 怎么做    | PQST 怎么做      | 强度等价              |
| ---------------- | ------------ | ------------- | ----------------- |
| 单向绑 responder 静态 | es（DH）       | ct_1（KEM）     | ✓ 等价（A 类）         |
| 单向绑 initiator 静态 | se（DH）       | ct_3（KEM）     | ✓ 等价（A 类）         |
| 绑本次会话（临时互）       | ee（DH）       | ct_2（KEM）     | ✓ 等价（形式不同，语义对）    |
| **双方静态互锁（B 类）**  | **ss（ECDH）** | **PSK（对称密钥）** | ✓ 等价（前提：PSK 真实存在） |




### psk1 vs psk2 位置的差异（为什么 PSK 前移）

原版 WG 的 `psk2` 把 PSK 放在消息 2 末尾混入——PSK 在原 WG 是 optional 的"额外对称密钥层"，混在末尾即可。PQST 改成 `psk1`，PSK 提前到消息 1 末尾混入（紧跟 `C_4 = KDF(C_3, S_i^pub)` 之后），目的是让 PSK **尽早**发挥作用——它在 PQST 中**新增承担**了"补偿被砍掉的 ss"的职责，而 `ss` 在原版 WG 出现在消息 1 内部，所以 PSK 必须紧跟其后才能"占位相同位置"。

这也是为什么协议标识符从 `IKpsk2` 改成 `IKpsk1kem`——位置变了，作用也变了。

### 默认 PSK 的降级与 PKI 工程动机

PSK 顶替 ss 的前提是"PSK 真实存在并正确分发"。若用户不配置 PSK，KE.md 第 2 行给出的默认值是：

```
default_psk = Hash(local_responder_static_pub XOR remote_responder_static_pub)
```

这个默认 PSK **只用公开的静态公钥**就能算出，因此只能解决"协议层占位"问题，**不提供真正的 B 类互锁保护**——它纯靠双方静态公钥的单向绑定，相当于 A 类绑定的弱重复。

这是 PQST 当前协议的**已知弱化点**，也是项目引入 PKI / 证书管理（`certificate-management/` 目录）的工程动机之一：用 PKI 把 PSK 真实分发出去，避免依赖默认 PSK 这种"形式上满足、实质上降级"的妥协方案。

### 答辩话术

被追问"为什么不是简单逐行把 DH 换成 KEM"时：

> 原版 WG 用 4 次 ECDH，其中 `ss` 是"双方静态私钥互锁"——ECDH 的对称性让 1 次运算、0 字节载荷就能完成。KEM 是单向的，做不到这种对称互锁，要等价实现需要双方各做一次 KEM encap 各发一份 ct，握手包会超 3000~4000 字节，已分片负担之外再加倍。所以 PQST 砍掉 ss 的 KEM 化，只保留 es、ee、se 三个 KEM，并**把 PSK 从消息 2 末尾前移到消息 1 末尾**（psk2 → psk1）顶替 ss 的 B 类互锁作用——PSK 是事先共享的对称密钥，能提供与 ss 等价的"双方必须配合"强度。
>
> 代价是 PSK 必须真实存在；若用默认 PSK（双方静态公钥异或哈希），互锁保护降级为"形式满足、实质减弱"——这是当前协议的已知弱化点，也是我们引入 PKI 证书管理的工程动机之一，目的是把真实 PSK 通过带外渠道分发出去，恢复完整的 B 类互锁保护。

---



## 与 PQ-SecTunnel 的对照提示

本调研仅记录 WireGuard 原始设计。PQ-SecTunnel 在 WireGuard 架构上替换 Noise 握手为 **Noise_IKpsk1kem**（双 KEM 封装），保留 Cryptokey Routing、定时器状态机、Cookie 机制等整体结构。理解 WireGuard 的 IKpsk2 握手、三类密钥分工与 MAC1/MAC2 DoS 模型，是阐述 PQ-SecTunnel 改造动机的基础。

**逐项对比（答辩主文）** → `[doc/defense/00a-WireGuard背景与对比.md](../defense/00a-WireGuard背景与对比.md)`

---

*文档生成日期：2026-07-11*