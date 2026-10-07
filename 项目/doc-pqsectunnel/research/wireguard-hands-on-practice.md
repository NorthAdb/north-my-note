# WireGuard 动手实践笔记

> 本文档整理自 PQ-SecTunnel 开发过程中的 WireGuard 原生学习实验。  
> 目标：理解标准 WireGuard 的配置流程，为对照和调整 PQ-SecTunnel（`pqst` / `pqst-quick`）提供基线。

---

## 1. 实验目标与架构

在本机用 **network namespace（netns）** 模拟两台机器，完成标准 WireGuard 端到端隧道。

```
┌─ wg-server-t1 ─────────────────────┐     ┌─ wg-client-t1 ─────────────────────┐
│  veth-t1s: 172.30.0.1  (底层网络)  │     │  veth-t1c: 172.30.0.2  (底层网络)  │
│  server-t1: 10.30.0.1  (隧道内网)  │◄───►│  client-t1: 10.30.0.2  (隧道内网)  │
│  lo: 127.0.0.1        (回环)       │ UDP │  lo: 127.0.0.1        (回环)       │
└────────────────────────────────────┘     └────────────────────────────────────┘
```

### IP 规划（避免与 PQ 实验的 `10.20.0.0/24` 冲突）

| 角色 | 底层网卡 | 底层 IP | 隧道网卡 | 隧道 IP |
|------|----------|---------|----------|---------|
| Server | `veth-t1s` | `172.30.0.1/24` | `server-t1`* | `10.30.0.1/24` |
| Client | `veth-t1c` | `172.30.0.2/24` | `client-t1`* | `10.30.0.2/24` |

\* 接口名由 `wg-quick` 配置文件名决定（`server-t1.conf` → 接口 `server-t1`）。纯命令行方式可自定名为 `wg0`。

### 每个 netns 中的三张网卡

| 网卡 | 来源 | 作用 |
|------|------|------|
| `lo` | 系统自带 | 回环，每个 namespace 都有 |
| `veth-t1s` / `veth-t1c` | 手动创建 | 底层网络，UDP 握手包经此收发 |
| `server-t1` / `client-t1` | WireGuard | 隧道内网，加密流量走此接口 |

---

## 2. 前置检查

### 2.1 确认 WireGuard 已安装

```bash
which wg wg-quick
wg --version
```

Ubuntu 典型安装：`wireguard`（元包）+ `wireguard-tools`（`wg` / `wg-quick`）。

### 2.2 加载内核模块并验证

```bash
sudo modprobe wireguard

# 功能验证：能创建 wireguard 类型网卡即 OK
sudo ip link add test-wg type wireguard
sudo ip link del test-wg
```

| 组件 | 工具/模块 | 作用 |
|------|-----------|------|
| 用户态 | `wg`、`wg-quick`（wireguard-tools） | 配置、查看状态 |
| 内核态 | `wireguard.ko` | 实现隧道网卡、握手、加解密 |
| 通信机制 | netlink | 用户态与内核的配置通道 |

**说明**：`ip link add ... type wireguard` 走 RTNETLINK；`wg set` 走 WireGuard 的 Generic Netlink。二者配合完成隧道搭建。VPN 数据本身走 UDP socket，不走 netlink。

**常见报错**：

```
Error: Unknown device type.
Unable to access interface: Protocol not supported
```

原因：`wireguard.ko` 未加载。先 `sudo modprobe wireguard`。注意内置模块不会出现在 `lsmod` 中，应以能否创建接口为准。

---

## 3. 密钥生成

```bash
mkdir -p ~/wg-learn && cd ~/wg-learn

# Server
wg genkey | tee server.key | wg pubkey > server.pub

# Client
wg genkey | tee client.key | wg pubkey > client.pub

chmod 600 server.key client.key
```

| 文件 | 内容 |
|------|------|
| `*.key` | 私钥，绝不外传 |
| `*.pub` | 公钥，写入对端 `[Peer]` 段 |

管道数据流：

```
wg genkey → tee（写 server.key + 转发）→ wg pubkey → server.pub
```

---

## 4. 搭建 netns 底层网络

### 4.1 创建 namespace

```bash
sudo ip netns add wg-server-t1
sudo ip netns add wg-client-t1
```

命名空间在 `/run/netns/<名字>` 注册。若报 `File exists`，说明已存在，可 `sudo ip netns del <名字>` 后重建，或直接复用。

### 4.2 用 veth 连接两端

```bash
sudo ip link add veth-t1s type veth peer name veth-t1c
sudo ip link set veth-t1s netns wg-server-t1
sudo ip link set veth-t1c netns wg-client-t1
```

可分两步移动并改名，也可一步完成：

```bash
sudo ip link set veth-t1s netns wg-server-t1 name eth0   # 若 eth0 已存在会报 File exists
```

### 4.3 配置底层 IP

```bash
sudo ip netns exec wg-server-t1 ip addr add 172.30.0.1/24 dev veth-t1s
sudo ip netns exec wg-server-t1 ip link set veth-t1s up
sudo ip netns exec wg-server-t1 ip link set lo up

sudo ip netns exec wg-client-t1 ip addr add 172.30.0.2/24 dev veth-t1c
sudo ip netns exec wg-client-t1 ip link set veth-t1c up
sudo ip netns exec wg-client-t1 ip link set lo up

# 验证底层连通
sudo ip netns exec wg-client-t1 ping -c 2 172.30.0.1
```

---

## 5. 方案 A：纯命令行配置（`wg set` + `ip`）

配置写入**内核内存**，重启或 `ip link del` 后消失，**不会自动生成配置文件**。

### 5.1 Server 端

```bash
cd ~/wg-learn

sudo ip netns exec wg-server-t1 ip link add wg0 type wireguard

sudo ip netns exec wg-server-t1 wg set wg0 \
    private-key ./server.key \
    listen-port 51820

sudo ip netns exec wg-server-t1 wg set wg0 \
    peer $(cat client.pub) \
    allowed-ips 10.30.0.2/32

sudo ip netns exec wg-server-t1 ip addr add 10.30.0.1/24 dev wg0
sudo ip netns exec wg-server-t1 ip link set wg0 up
```

### 5.2 Client 端

```bash
cd ~/wg-learn

sudo ip netns exec wg-client-t1 ip link add wg0 type wireguard

sudo ip netns exec wg-client-t1 wg set wg0 \
    private-key ./client.key

sudo ip netns exec wg-client-t1 wg set wg0 \
    peer $(cat server.pub) \
    endpoint 172.30.0.1:51820 \
    allowed-ips 10.30.0.0/24 \
    persistent-keepalive 25

sudo ip netns exec wg-client-t1 ip addr add 10.30.0.2/24 dev wg0
sudo ip netns exec wg-client-t1 ip link set wg0 up
```

### 5.3 各命令职责

| 命令 | 工具 | 作用 |
|------|------|------|
| `ip link add wg0 type wireguard` | iproute2（RTNETLINK） | 创建隧道网卡壳子 |
| `wg set ... private-key / peer / endpoint` | wireguard-tools（genl） | 写密钥、对端、路由策略 |
| `ip addr add ... dev wg0` | iproute2 | 配隧道 IP |
| `ip link set wg0 up` | iproute2 | 启用接口 |

### 5.4 从内核导出配置（可选）

```bash
sudo ip netns exec wg-server-t1 wg showconf wg0 > server-export.conf
```

`wg showconf` 不含 `Address` 字段（Address 由 `ip addr` 或 `wg-quick` 单独配置）。

---

## 6. 方案 B：`wg-quick` + 配置文件（推荐用于可复现部署）

**持久化的是磁盘上的 `.conf` 文件**；`wg-quick` 负责读文件并写入内核。

### 6.1 编写配置文件

**`~/wg-learn/server-t1.conf`**（Server）：

```ini
[Interface]
PrivateKey = <server.key 内容>
Address = 10.30.0.1/24
ListenPort = 51820

[Peer]
PublicKey = <client.pub 内容>
AllowedIPs = 10.30.0.2/32
```

**`~/wg-learn/client-t1.conf`**（Client）：

```ini
[Interface]
PrivateKey = <client.key 内容>
Address = 10.30.0.2/24

[Peer]
PublicKey = <server.pub 内容>
Endpoint = 172.30.0.1:51820
AllowedIPs = 10.30.0.0/24
PersistentKeepalive = 25
```

用命令生成（避免手抄密钥）：

```bash
cd ~/wg-learn

cat > server-t1.conf << EOF
[Interface]
PrivateKey = $(cat server.key)
Address = 10.30.0.1/24
ListenPort = 51820

[Peer]
PublicKey = $(cat client.pub)
AllowedIPs = 10.30.0.2/32
EOF

cat > client-t1.conf << EOF
[Interface]
PrivateKey = $(cat client.key)
Address = 10.30.0.2/24

[Peer]
PublicKey = $(cat server.pub)
Endpoint = 172.30.0.1:51820
AllowedIPs = 10.30.0.0/24
PersistentKeepalive = 25
EOF

chmod 600 server-t1.conf client-t1.conf
```

### 6.2 启动与停止

```bash
cd ~/wg-learn

# 若已有手动创建的 wg0，先删除
sudo ip netns exec wg-server-t1 ip link del wg0 2>/dev/null
sudo ip netns exec wg-client-t1 ip link del wg0 2>/dev/null

# 启动（接口名 = 配置文件名去掉 .conf）
sudo ip netns exec wg-server-t1 wg-quick up ./server-t1.conf
sudo ip netns exec wg-client-t1 wg-quick up ./client-t1.conf

# 停止（.conf 文件保留）
sudo ip netns exec wg-server-t1 wg-quick down ./server-t1.conf
sudo ip netns exec wg-client-t1 wg-quick down ./client-t1.conf
```

`wg-quick up` 等价于：`ip link add` → `wg set` → `ip addr add` → `ip link set up`。

### 6.3 配置存储对比

| 方式 | 配置真源 | 重启后 | 查看配置 |
|------|----------|--------|----------|
| 纯命令行 `wg set` | 内核内存 | 消失 | `wg show` / `wg showconf` |
| `wg-quick` + `.conf` | 磁盘文件 | 文件仍在，需再 `up` | `cat xxx.conf` |

| 配置项 | 归属 |
|--------|------|
| 私钥、peer、endpoint、allowed-ips | `[Interface]` / `[Peer]`（WireGuard） |
| 隧道 IP（Address） | `[Interface]`，由 wg-quick 用 `ip addr` 应用 |
| 底层 veth IP | 独立 `ip addr`，不在 .conf 中 |

---

## 7. 握手：自动触发，无需手动命令

WireGuard **没有**单独的「开始握手」命令。满足条件后内核自动完成 Noise IK 握手（1-RTT）。

### 7.1 触发条件

| 动作 | 是否触发握手 |
|------|-------------|
| Client `wg set ... endpoint ...` | **立刻触发** |
| 接口 up 后有待发送流量 | 触发 |
| `ping` 隧道 IP | 若尚未握手，先握手再传数据 |
| Server 仅 `listen-port`，无 endpoint | 被动等待，不主动发起 |

### 7.2 角色分工

```
Server（被动方）：listen-port 51820，不设 endpoint → 等客
Client（主动方）：endpoint 172.30.0.1:51820 → 主动发 Handshake Initiation
```

### 7.3 时序

```
Client                              Server
  │── Handshake Initiation (UDP) ──►│
  │◄── Handshake Response (UDP) ────│
  │  双方派生会话密钥，隧道就绪         │
```

配置完成 ≠ 仅「就绪」；Client 写上 `endpoint` 后握手即自动完成。

---

## 8. `wg show` 使用

### 8.1 基本查看

```bash
sudo ip netns exec wg-server-t1 wg show
sudo ip netns exec wg-client-t1 wg show

# 指定接口（wg-quick 创建的接口名可能不是 wg0）
sudo ip netns exec wg-server-t1 wg show server-t1
```

### 8.2 输出字段解读

```
interface: server-t1
  public key: kkwhGzuvpAa/...          # 本机公钥
  private key: (hidden)
  listening port: 51820                 # UDP 监听端口

peer: yy9VxersE54y...                   # 对端公钥
  endpoint: 172.30.0.2:37989            # 看到的对端 UDP 地址（含源端口）
  allowed ips: 10.30.0.2/32              # 允许通过的隧道 IP
  latest handshake: 26 seconds ago      # ★ 有值 = 握手已成功
  transfer: 3.20 KiB received, 920 B sent # 加密流量统计
  persistent keepalive: every 25 seconds # Client 侧 NAT 保活
```

| 字段 | 含义 |
|------|------|
| `latest handshake` | 上次 Noise 握手完成时间；`(none)` 表示尚未握手 |
| `endpoint` | 对端 UDP 地址；Server 侧是动态学到的 Client 源地址 |
| `transfer` | 该 peer 的加解密字节计数 |
| `listening port` | 本机 UDP 端口；Client 侧常为随机高端口 |

### 8.3 验证隧道

```bash
sudo ip netns exec wg-client-t1 ping -c 3 10.30.0.1
sudo ip netns exec wg-server-t1 ping -c 3 10.30.0.2
```

`ping` 走隧道 IP → 经 `wg0`/`server-t1` 加密；`ping 172.30.0.x` 走底层 veth，不加密。

---

## 9. 查看 netns 中的全套网卡

```bash
# 简要一览（推荐）
sudo ip netns exec wg-server-t1 ip -br addr
sudo ip netns exec wg-client-t1 ip -br addr

# 链路层详情
sudo ip netns exec wg-server-t1 ip link show

# 路由表
sudo ip netns exec wg-server-t1 ip route show

# 两端汇总
for ns in wg-server-t1 wg-client-t1; do
  echo "========== $ns =========="
  sudo ip netns exec $ns ip -br addr
  sudo ip netns exec $ns ip route
  sudo ip netns exec $ns wg show 2>/dev/null
  echo
done
```

---

## 10. systemd 服务：真实场景 vs netns 实验

### 10.1 真实单机部署

配置文件放在 `/etc/wireguard/`：

```bash
sudo cp ~/wg-learn/server-t1.conf /etc/wireguard/wg0.conf
sudo chmod 600 /etc/wireguard/wg0.conf

sudo wg-quick up wg0
sudo wg-quick down wg0

# 开机自启
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
sudo systemctl status wg-quick@wg0
sudo journalctl -u wg-quick@wg0
```

此时可在主机上直接查看服务状态、日志、开机自启。

### 10.2 netns 实验环境

默认 **不能** 在 netns 内直接 `systemctl start wg-quick@wg0`——systemd 服务跑在主机 init namespace，看不到 netns 内的接口。

实验中的做法（与 PQ-SecTunnel `setup_ns.sh` 一致）：

```bash
sudo ip netns exec wg-server-t1 wg-quick up ./server-t1.conf
```

状态查看用 `wg show` / `ip -br addr`，而非 `systemctl`。

### 10.3 若一定要在 netns 中服务化管理

在主机上创建 wrapper unit，内部用 `ip netns exec`：

```ini
# /etc/systemd/system/wg-quick-server-t1.service
[Unit]
Description=WireGuard server-t1 in netns wg-server-t1
After=network.target

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/home/northadb/wg-learn
ExecStart=/usr/bin/ip netns exec wg-server-t1 /usr/bin/wg-quick up /home/northadb/wg-learn/server-t1.conf
ExecStop=/usr/bin/ip netns exec wg-server-t1 /usr/bin/wg-quick down /home/northadb/wg-learn/server-t1.conf

[Install]
WantedBy=multi-user.target
```

较新 systemd（248+）也可用 `NetworkNamespacePath=/run/netns/wg-server-t1`。

---

## 11. 与 PQ-SecTunnel 对照

| 标准 WireGuard | PQ-SecTunnel |
|----------------|--------------|
| `wireguard.ko` | `pqsectunnel.ko` |
| `wg` / `wg-quick` | `pqst` / `pqst-quick` |
| `PrivateKey` / `PublicKey` | `InitiatorPrivateKey` / `ResponderPublicKey` 等 |
| 2 轮握手（Init + Response） | 3 轮（+ Key Confirmation） |
| X25519 ECDH | ML-KEM 封装/解封装 |
| netns 名（学习用）`wg-server-t1` | 测试框架 `srv0_ns` / `cli0_ns` |
| 底层网段（学习用）`172.30.0.0/24` | 测试框架 `10.20.0.0/24`（index=0） |
| 配置文件 `*.conf` | `pqst0.conf` / `server0.pqst0.conf` |
| 触发机制 | 同样自动；`endpoint` 触发主动方 |

PQ 实验目录典型用法：

```bash
cd experiments/mlkem512-mlkem768-sm4gcm-sm3/
sudo insmod pqsectunnel.ko
sudo ./setup_ns.sh
sudo ip netns exec srv0_ns ./pqst show
```

---

## 12. 清理命令

```bash
# 停止隧道
cd ~/wg-learn
sudo ip netns exec wg-server-t1 wg-quick down ./server-t1.conf 2>/dev/null
sudo ip netns exec wg-client-t1 wg-quick down ./client-t1.conf 2>/dev/null

# 删除 netns（会先清理内部网卡）
sudo ip netns del wg-server-t1
sudo ip netns del wg-client-t1

# 查看残留
sudo ip netns list
ls -la /run/netns/
```

---

## 13. 两种方式选择建议

| 场景 | 推荐 |
|------|------|
| 理解内核配置路径、对照源码 | 方案 A：纯 `wg set` |
| 可复现实验、对照 PQ 配置文件 | 方案 B：`wg-quick` + `.conf` |
| 本机 netns 学习 | `ip netns exec ... wg-quick up` |
| 生产部署 | `/etc/wireguard/wg0.conf` + `systemctl enable wg-quick@wg0` |

---

## 14. 相关文档

- [wireguard-background.md](wireguard-background.md) — WireGuard 协议与密码学背景调研
- [../../docs/zh/handshake.md](../../docs/zh/handshake.md) — PQ-SecTunnel 握手流程
- [../../KE.md](../../KE.md) — Noise_IKpsk1kem 协议规范
- [../../CONTEXT.md](../../CONTEXT.md) — PQ-SecTunnel 领域词汇表

---

## 15. 漫游（Roaming）实验

WireGuard **内置漫游**：当对端从新的 UDP 源地址发来**已认证**的数据包时，本机自动更新该 peer 的 `endpoint`，无需改配置文件、无需重启接口。

### 15.1 漫游机制简述

| 概念 | 说明 |
|------|------|
| 配置里的 `Endpoint` | 只是**初始/_hint** 地址，主动方用来发起第一次通信 |
| 运行时的 `endpoint` | `wg show` 里看到的**动态**地址，可被漫游更新 |
| 更新触发 | 收到对端合法加密包时，用包的源 IP:port 覆盖 peer endpoint |
| 双向 | Server、Client 都会学习对端新地址，不是单向 |

典型场景：手机从 WiFi 切到 4G，公网 IP 变了；笔记本换网络；NAT 重新分配端口。

### 15.2 实验前提

- 隧道已用 `wg-quick` 跑通（`latest handshake` 有值）
- 使用本文档的 `wg-server-t1` / `wg-client-t1` 环境
- 开一个辅助终端实时监控 endpoint：

```bash
# 终端 A：盯 Server 侧 peer endpoint
watch -n 1 'sudo ip netns exec wg-server-t1 wg show server-t1'

# 终端 B：盯 Client 侧 peer endpoint
watch -n 1 'sudo ip netns exec wg-client-t1 wg show client-t1'
```

### 15.3 实验一：Client 底层 IP 变化（最常见）

模拟 Client「换了个网络，公网 IP 变了」，Server 自动学到新地址。

**① 记录当前 endpoint**

```bash
sudo ip netns exec wg-server-t1 wg show server-t1
# peer 的 endpoint 应为 172.30.0.2:<随机端口>
```

**② 改 Client 底层 IP（不动 WireGuard 配置、不 restart）**

```bash
sudo ip netns exec wg-client-t1 ip addr del 172.30.0.2/24 dev veth-t1c
sudo ip netns exec wg-client-t1 ip addr add 172.30.0.22/24 dev veth-t1c
```

**③ 从 Client 发隧道流量**

```bash
sudo ip netns exec wg-client-t1 ping -c 3 10.30.0.1
```

**④ 再看 Server 的 wg show**

```bash
sudo ip netns exec wg-server-t1 wg show server-t1
# endpoint 应变为 172.30.0.22:<端口>  ← 漫游成功
```

**⑤ 反向验证：Server 无需改配置即可回 ping**

```bash
sudo ip netns exec wg-server-t1 ping -c 3 10.30.0.2
```

全程**没有**改 `client-t1.conf` / `server-t1.conf`，也没有 `wg-quick down/up`。

### 15.4 实验二：Server 底层 IP 变化

模拟 Server 地址变了，Client 通过对端发来的包学习新地址。

**① 改 Server 底层 IP**

```bash
sudo ip netns exec wg-server-t1 ip addr del 172.30.0.1/24 dev veth-t1s
sudo ip netns exec wg-server-t1 ip addr add 172.30.0.11/24 dev veth-t1s
```

此时 Client 配置的 `Endpoint = 172.30.0.1:51820` 已失效：

```bash
sudo ip netns exec wg-client-t1 ping -c 2 10.30.0.1   # 可能失败
```

**② 从 Server 主动 ping Client（关键一步）**

```bash
sudo ip netns exec wg-server-t1 ping -c 3 10.30.0.2
```

Server 从**新地址** `172.30.0.11` 发出已认证包 → Client 收到后更新 peer endpoint。

**③ 检查 Client 是否学到新 Server 地址**

```bash
sudo ip netns exec wg-client-t1 wg show client-t1
# peer endpoint 应变为 172.30.0.11:51820
```

**④ Client 再 ping，应恢复**

```bash
sudo ip netns exec wg-client-t1 ping -c 3 10.30.0.1
```

要点：被动方 IP 变了之后，需要**对端先发来一个合法包**，本机才能完成 endpoint 更新。Client 侧的 `PersistentKeepalive = 25` 在真实 NAT 场景下有助于维持/触发这类更新。

### 15.5 实验三：观察 NAT 端口变化（可选进阶）

用 `iptables SNAT` 模拟 NAT 端口改写：

```bash
# 在 Client netns 对发出的 UDP 做 SNAT（需 root）
sudo ip netns exec wg-client-t1 iptables -t nat -A POSTROUTING \
    -o veth-t1c -p udp -j SNAT --to-source 172.30.0.2:60000
```

再从 Client ping 隧道 IP，观察 Server 侧 `endpoint` 端口是否变为 `60000`（或你指定的值）。

### 15.6 判断漫游成功的标准

| 检查项 | 漫游成功 |
|--------|----------|
| `wg show` 中 peer `endpoint` 变为新 IP:port | ✓ |
| 未修改 `.conf` 文件 | ✓ |
| 未执行 `wg-quick down/up` | ✓ |
| 隧道 IP `ping` 恢复 | ✓ |
| `latest handshake` 更新 | ✓ |

### 15.7 与配置文件的关系

- `.conf` 里的 `Endpoint` **不会**被漫游自动写回磁盘
- 漫游只更新**内核运行时状态**
- 重启 `wg-quick up` 后仍从 `.conf` 读初始 `Endpoint`
- 导出当前运行态：`wg showconf`（仍可能不含漫游后的 endpoint，视版本而定；**以 `wg show` 为准**）

### 15.8 与 PQ-SecTunnel 的关系

PQ-SecTunnel fork 自 WireGuard，数据面与 endpoint 学习逻辑同源。若 PQ 隧道已跑通，可用同样方法在 `srv0_ns` / `cli0_ns` 上改底层 veth IP，观察 `pqst show` 中 endpoint 是否自动更新——可作为 PQ 改造后的回归验证项。
