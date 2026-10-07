# 06 — 分片与 MTU

> 状态：正文 | 权威源：[CONTEXT.md](../../CONTEXT.md) + `pqc_fragment.c` / `send.c` / `receive.c`  
> 主套件：`mlkem512-mlkem768-sm4gcm-sm3` | 预计篇幅：3–4 min 讲稿

## 1. 为什么主套件需要应用层分片

| 报文 | 尺寸 | 1400 B 阈值 | 主套件行为 |
|------|------|------------|-----------|
| Handshake Initiation | 2068 B | 超出 | 分 2 片 |
| Handshake Response | 1916 B | 超出 | 分 2 片 |
| Key Confirmation | 60 B | 不超出 | 单 UDP 包 |

典型以太网 MTU 1500 B，UDP 载荷约 1472 B；纯 ML-KEM-512/768 组合的握手 struct 远超此值，**必须**在应用层可控地拆分，避免依赖 IP 层分片。

主套件构建记录：`suite_meta.env` 中 `APP_LAYER_FRAGMENTATION=auto`、`APP_LAYER_FRAGMENTATION_EFFECTIVE=1`。

---

## 2. 两种分片（CONTEXT）

| 概念 | 机制 | 可控性 | 风险 |
|------|------|--------|------|
| **应用层分片** | `pqc_fragment.c` 在 UDP 载荷加 8 B 头，按 1400 B 切分 | 应用可控 | 需收发双方实现重组 |
| **IP 层分片** | 内核 IP 栈对超大 datagram 自动分片 | 应用不可控 | 中间防火墙/NAT 易丢弃 |

### "应用层"一词的含义辨析

此处"应用层"是相对 **IP 层** 而言的上一层约定——不是 OSI 七层模型中第 7 层（HTTP/DNS）那个绝对意义上的应用层。

```
┌────────────────────────────────────────────┐
│  应用层分片（站在 IP 层看）                │  ← pqc_fragment.c 在此切分
│  - WG/PQST 自己在 UDP 载荷内切段           │
│  - 8 字节 PQ 头：magic/msg_id/frag_index/frag_total/frag_len
│  - 收发两端自己重组                        │
├────────────────────────────────────────────┤
│  IP 层（基座）                             │  ← 默认 fallback
│  - 内核 IP 栈对超 MTU datagram 自动分片     │
│  - 中间防火墙/NAT/有状态设备易丢           │
├────────────────────────────────────────────┤
│  底层（IP 也不管）                         │
│  - 以太网帧 / 物理链路                      │
└────────────────────────────────────────────┘
```

这与本文档其他章节出现的"Noise 框架之上的应用层字段（mac1/mac2/TAI64N）"**是同一个相对用法，但基座不同**。两者关系是**正交的两条工程问题轨道**，互不干扰：

| 说法 | 基座 | 含义 | 文档位置 |
|------|------|------|---------|
| "Noise 框架之上的应用层字段" | Noise 状态机（`ck`/`h`）| mac1/mac2/TAI64N 不进 transcript，由 WG 自己定义语义 | `wireguard-background.md` §"应用层的含义澄清" |
| "应用层分片（`pqc_fragment`）" | IP 层 | WG/PQST 自己在 UDP 载荷里切段，不让 IP 层做 | 本节 |

二者在 PQ-SecTunnel 中**同时存在**但互不干扰：

```
握手消息内容（数据平面）
├─ mac1 / mac2 / TAI64N            ← Noise 之上的"应用层字段"
├─ encrypted_ciphertext (KEM 大包) ← PQ 改造引入的握手载荷
└─ 对超大的整条 UDP 报文做切分      ← IP 之上的"应用层分片"
```

- 前者是**字段**层面的应用层约定——决定"握手包里有什么内容"。
- 后者是**传输**层面的应用层约定——决定"这条已经构造好的握手 UDP 报文怎么经过链路"。

而且二者还有一个**强制顺序**约束：必须**先**做 mac1/mac2（Noise 应用层字段）覆盖整段原始握手缓冲，**再**做应用层分片切分——否则破解分片后的 mac1 验证无法对原始 transcript 做 WireGuard 兼容的 MAC1 校验（详见下节"设计约束：MAC 先于分片"）。

### WireGuard 在哪一层？分片是不是“传输层分片”？

WireGuard **不是**“专属于 IP 层、由 IP 栈完成的协议”。更准确：

| 视角 | WireGuard / PQ-SecTunnel 做什么 |
|------|--------------------------------|
| **隧道内容（内层）** | 加密封装的是内层 **IP 包** → 常称 L3 VPN |
| **外层承载** | 握手与加密数据都放在 **UDP** 里发出（如 `:51820`） |

```
内层：  业务 IP 包（AllowedIPs / 隧道流量）
          ↓ AEAD 加密
外层：  WG / PQST 协议报文（握手 struct 或 DATA）
          ↓
        UDP
          ↓
        IP（Underlay，如实验里的 10.20.0.x）
```

因此：

- **不是** IP 层在替你分握手包；
- **也不是** UDP/TCP 协议自带的分片（UDP 本身不分片；TCP 是字节流，另说）；
- **是** 把 UDP 当 datagram 用的 **WG/PQST 协议自己** 定义的分段/重组（UDP 之上的自定义分片）。

一句话：相对 IP 叫“应用层分片”；相对 OSI 更接近 **“UDP 载荷里的自定义分段”**。

### 发出去长什么样（直观结论）

分片 = 对**完整握手报文**切分 → 每段前加 **8 B PQ 分片头** → 每一段各自再套 **独立的 UDP 头 + IP 头** 发出：

```
完整握手 2068 B（已含 mac1/mac2）
    ├─ 片0: [PQ头 8B | 握手[0..1400)]    → UDP → IP
    └─ 片1: [PQ头 8B | 握手[1400..2068)] → UDP → IP
```

对端收齐后去掉各片 PQ 头，拼回原来的 2068 B，再验 MAC、进 `noise.c`。

### 答辩话术

被追问"你说的应用层分片中的'应用层'是不是 OSI 第 7 层"时，应答：

> 两者都是"相对某层基座而言的上一层"，这个用法一致，但**基座不同**。Noise 应用层字段是站在 Noise 状态机角度看 WG 加给的 mac1/mac2/TAI64N——这些字段不进 ck/h。应用层分片是站在 IP 层角度看 WG/PQST 自己在 UDP 载荷内做的分片——不依赖 IP 分片。两者是正交的工程问题：前者是字段级外挂，后者是传输层外挂。在我们项目里，前者完全继承自 WireGuard，后者是新加的——但两条轨道不互相影响。

被追问"WG 不是 IP 层的吗，怎么做应用层分片"时，应答：

> WG 管的是**隧道里的 IP**；握手大包走的是 **UDP**。应用层分片是我们在进 socket 之前把完整握手切成多段，每段加 PQ 头后各自发成一个小 UDP 包，刻意避开 IP 分片——不是传输层协议替你分的片。

PQ-SecTunnel 主套件编译启用 **`PQST_APP_LAYER_FRAGMENTATION`**，Initiation/Response 走应用层路径；未启用宏的小包实验（如 McEliece 默认套件）可单包直发。

---

## 3. 设计总览：切在哪一层、切什么

分片对象是 **已经组好、且已写入 mac₁/mac₂ 的完整握手 struct**（`messages.h` 里那一整段字节），不是对 KEM 密文单独切。  
每一片 = **一个独立 UDP datagram**；UDP 载荷开头是 8 B `pqc_frag_hdr`，后面跟本片握手字节。

```
┌──────────── noise 组包 ────────────┐
│  Initiation / Response 完整 struct │  ← 含 type…payload…mac1/mac2
└─────────────────┬──────────────────┘
                  │ MAC 已覆盖完整缓冲（必须先于此）
                  ▼
┌──────────── pqc_fragment_send ─────┐
│  按 1400 B 切片，每片前缀 8 B 头   │
│  → 多次 wg_socket_send_buffer…   │  ← 每片一个 UDP
└─────────────────┬──────────────────┘
                  │ 网络
                  ▼
┌──────────── pqc_fragment_recv ─────┐
│  按 msg_id 进重组槽，片齐后拼回    │
│  → 向上交付「无分片头」的完整包    │
└─────────────────┬──────────────────┘
                  ▼
         先验 MAC → 再进 noise.c
```

**设计要点（答辩可背）：**

1. **MAC 包住完整握手，再切分**（见 §4）——验 MAC 时看到的仍是原始 WireGuard/PQ 握手布局。  
2. **切分粒度固定 1400 B 载荷**——给 IP/UDP 头留余量，避免再触发 IP 分片。  
3. **接收侧对上层透明**——重组后 `skb->data` 从握手 struct 开头起，后续路径与未分片相同。  
4. **Key Confirm（60 B）不进分片路径**——`send.c` 直接 `wg_socket_send_buffer_to_peer`。

---

## 4. 设计约束：MAC 先于分片

KE.md 与 CONTEXT **应用层分片** 均要求：

1. **完整握手 struct** 先由 `wg_cookie_add_mac_to_packet()` 写入 mac₁/mac₂  
2. **再**对含 MAC 的完整缓冲调用 `pqc_fragment_send()`  

接收侧：`pqc_fragment_recv()` 重组出**完整握手包** → `wg_cookie_validate_packet()` 验 MAC → `noise.c` consume。

若先分片再算 MAC，则无法对原始 transcript 做 WireGuard 兼容的 MAC1 验证（MAC 密钥覆盖的是整包正文，不是单片）。

### 发送顺序（send.c）

```c
wg_noise_handshake_create_*(&packet, ...);      // 1. noise 组包（无 MAC）
wg_cookie_add_mac_to_packet(&packet, sizeof(packet), peer);  // 2. MAC
#if defined(PQST_APP_LAYER_FRAGMENTATION)
    pqc_fragment_send(peer, &packet, sizeof(packet), HANDSHAKE_DSCP);  // 3. 分片
#else
    wg_socket_send_buffer_to_peer(...);         // 3. 直发
#endif
```

Initiation：`send.c` L63–74；Response：L129–140。Key Confirmation（60 B）**不分片**，直发 L164–166。

---

## 5. 线上格式：分片头 + 载荷

定义：`pqc_fragment.h` — **固定 8 字节**，小端字段，`__packed`。

```c
struct pqc_frag_hdr {
    __le16 magic;      /* 0x5051 ("PQ") —— 与普通握手 type 区分 */
    __le16 msg_id;     /* 随机，同一握手多片共用 */
    __u8   frag_index; /* 0-based 片序号 */
    __u8   frag_total; /* 本消息总片数 */
    __le16 frag_len;   /* 本片载荷字节数（握手数据，不含本头） */
};
```

| 常量 | 值 | 含义 |
|------|-----|------|
| `PQC_FRAG_MAGIC` | 0x5051 | 识别 PQ 分片 / ACK |
| `PQC_FRAG_PAYLOAD_SIZE` | 1400 | 每片最大握手字节（不含 8 B 头） |
| `PQC_MAX_REASM_SLOTS` | 16 | 并发重组槽（按 msg_id） |
| `PQC_FRAG_MAX_FRAGS` | 256 | 单消息最大片数 |
| `PQC_FRAG_PENDING_SLOTS` | 4 | 发送侧待 ACK 的并发分片发送 |

### 单片在 UDP 上的样子

```
UDP payload:
┌────────── 8 B ──────────┬──────────── frag_len ────────────┐
│ magic|msg_id|idx|total|len │  握手 struct 的一段连续字节     │
└────────────────────────────┴───────────────────────────────┘
  ≈ 每片 UDP 载荷 ≤ 8 + 1400 = 1408 B（远小于 ~1472 安全上限）
```

### 主套件 Initiation（2068 B）切法示例

```
原始握手缓冲（含 MAC）长度 L = 2068
frag_total = ceil(2068 / 1400) = 2
msg_id     = 随机（两片相同）

片 0: frag_index=0, frag_len=1400  ← 缓冲 [0, 1400)
片 1: frag_index=1, frag_len=668   ← 缓冲 [1400, 2068)
```

Response 1916 B → 同理 **2 片**（1400 + 516）。

接收侧按 `frag_index * 1400` 把各片拷回重组缓冲，全部 `frag_seen` 后，`handshake_len = Σ frag_len`（如 2068），向上交付**去掉所有分片头**的完整 struct。

---

## 6. 发送逻辑（pqc_fragment_send）

实现：`pqc_fragment.c`（约 L450–521）。

| 步骤 | 行为 |
|------|------|
| 1 | `len == 0` → 失败；`len ≤ 1400` → **不切片**，直接 `wg_socket_send_buffer_to_peer` |
| 2 | `len > 1400`：`msg_id = get_random_u16()`；`frag_total = ceil(len / 1400)` |
| 3 | 对每个 `frag_index`：填 8 B 头 + 拷贝本片握手字节 → 独立 UDP 发出 |
| 4 | 拷贝整包到 `pending_frag_send` 槽，启动 **100 ms** 定时器，等待分片 ACK（见 §8） |

片间无额外握手语义；顺序发送但**重组不依赖到达顺序**（靠 `frag_index` 定位）。

上限：单消息不超过 `256 × 1400` 字节；主套件远小于此。

---

## 7. 接收逻辑（pqc_fragment_recv）

入口：`receive.c` `wg_packet_receive()`——先看 UDP 载荷是否 `magic == 0x5051`，是则进分片模块，否则按普通握手/数据解析。

| 步骤 | 行为 |
|------|------|
| 1 | 解析 `pqc_frag_hdr`；若为 ACK 编码（见 §8）则更新发送侧 bitmap，返回 `NULL` |
| 2 | 校验 `frag_total / frag_index / frag_len` |
| 3 | 按 `msg_id + frag_total` 找/建重组槽（最多 16 槽）；缺则丢弃 |
| 4 | 已见过的 `frag_index` → 丢弃重复片 |
| 5 | 载荷写入 `data[frag_index * 1400]`；刷新约 **1 s** 超时定时器 |
| 6 | 片未齐 → 返回 `NULL`（本 skb 消费掉，等后续片） |
| 7 | 片齐 → 拼出完整握手 skb（**无**分片头前缀）→ `slot` 复位 → 交给上层 |

上层拿到完整包后与直发路径相同：**先验 MAC，再 noise consume**。

重组完成后 `skb->len` = 原始握手 struct 长度（主套件如 2068 / 1916）。

---

## 8. 分片 ACK 与丢包补发（设计细节）

丢片时若只靠上层握手重传整包成本高，因此在分片层加了 **选择性确认**：

**ACK 报文编码**（与普通分片头共用 `pqc_frag_hdr`，不另开 type）：

| 字段 | 取值 | 含义 |
|------|------|------|
| `frag_total` | `0xFF` | `PQC_FRAG_ACK_TOTAL` |
| `frag_index` | `0xFF` | `PQC_FRAG_ACK_INDEX` |
| `frag_len` | bitmap 字节数 | |
| 载荷 | `frag_seen` 位图 | 哪些片已收到 |

普通片中 `frag_index ∈ [0, total)` 且 `total ≤ 255`，故 `0xFF/0xFF` **不会**与数据分片冲突。

| 侧 | 行为 |
|----|------|
| **接收** | 非 0 号片到达而头片仍缺时，可立刻 `send_ack`；完整重组前/超时时也可 ACK |
| **发送** | `pending` 记录整包副本；`FRAG_ACK_TIMEOUT_MS=100`；据 bitmap **只重发缺失片**；最多 `FRAG_ACK_RETRY_MAX=3` |
| **会话建立后** | `pqc_fragment_session_established(peer)` 清掉该 peer 的 pending，避免握手已成功还重发分片 |

无 pending 槽时发送仍成功（依赖握手层超时重传），不因 ACK 机制失败。

---

## 9. 主套件 dmesg 实证

来源：`experiments/mlkem512-mlkem768-sm4gcm-sm3/dmesg.log`

```
pqc_fragment: fragmented into 2 parts, msg_id=0x6011, total_len=2068    # Initiation 发送
pqc_fragment: reassembled skb->len=2068 handshake_len=2068               # Responder 重组
...
pqc_fragment: fragmented into 2 parts, msg_id=0x7ee6, total_len=1916    # Response 发送
pqc_fragment: reassembled skb->len=1916 handshake_len=1916               # Initiator 重组
```

Key Confirmation（60 B）日志中无 `pqc_fragment` 行，与「小报文不分片」一致。

握手成功标志（同文件）：

- `server to [...] completed handshake in ... ns`（Response 侧）
- `client [...] completed handshake in ... ns`（Initiator 收 Response）
- `Receiving handshake key confirmation` → 会话建立

---

## 10. 编译开关

| 变量 | 效果 |
|------|------|
| `APP_LAYER_FRAGMENTATION=auto` | struct >1400B 时定义 `PQST_APP_LAYER_FRAGMENTATION` |
| `APP_LAYER_FRAGMENTATION=1` | 强制启用 |
| `APP_LAYER_FRAGMENTATION=0` | 禁用；超大包依赖 IP 分片（不推荐） |

注入路径：`build.sh` → `APP_LAYER_FRAG_MACRO` → `make KCFLAGS`。

---

## 11. 答辩要点

- **WG 分层**：内层封 IP（L3 VPN）；外层握手/数据走 **UDP**——不是“纯 IP 层协议”
- **分片含义**：切完整握手 → 每段加 8 B PQ 头 → **多个独立 UDP** 发出（不是 IP 分片，也不是 UDP 协议自带分片）
- **为何分片**：主套件 Initiation 2068 B / Response 1916 B > 安全 UDP 载荷
- **切什么**：整段握手 struct（含 MAC），不是单独切 KEM 字段
- **怎么切**：8 B 头（magic=`0x5051` / msg_id / index / total / len）+ 每片最多 1400 B；主套件各 2 片
- **顺序约束**：先 `wg_cookie_add_mac_to_packet`，再 `pqc_fragment_send`
- **对上层透明**：重组后无分片头，先验 MAC 再进 noise
- **丢片**：ACK 位图 + 100 ms 选择性重传（最多 3 次）
- **谁不分片**：Key Confirm 60 B 直发
- **如何验证**：dmesg `fragmented into 2 parts` + `reassembled ... 2068/1916`

---

## 12. 交叉引用

| 主题 | 文档 |
|------|------|
| 报文尺寸 | [04](04-密码原语与套件.md) |
| send/receive 路径 | [05](05-内核实现路径.md) |
| 构建与 suite_meta | [07](07-工程架构与实验验证.md) |
