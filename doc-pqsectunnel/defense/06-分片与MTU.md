# 06 — 分片与 MTU

> 状态：正文 | 权威源：[CONTEXT.md](../../CONTEXT.md) + `pqc_fragment.c` / `send.c` / `receive.c`  
> 主套件：`mlkem512-mlkem768-sm4gcm-sm3` | 预计篇幅：2 min 讲稿

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

### 答辩话术

被追问"你说的应用层分片中的'应用层'是不是 OSI 第 7 层"时，应答：

> 两者都是"相对某层基座而言的上一层"，这个用法一致，但**基座不同**。Noise 应用层字段是站在 Noise 状态机角度看 WG 加给的 mac1/mac2/TAI64N——这些字段不进 ck/h。应用层分片是站在 IP 层角度看 WG/PQST 自己在 UDP 载荷内做的分片——不依赖 IP 分片。两者是正交的工程问题：前者是字段级外挂，后者是传输层外挂。在我们项目里，前者完全继承自 WireGuard，后者是新加的——但两条轨道不互相影响。

PQ-SecTunnel 主套件编译启用 **`PQST_APP_LAYER_FRAGMENTATION`**，Initiation/Response 走应用层路径；未启用宏的小包实验（如 McEliece 默认套件）可单包直发。

---

## 3. 设计约束：MAC 先于分片

KE.md 与 CONTEXT **应用层分片** 均要求：

1. **完整握手 struct** 先由 `wg_cookie_add_mac_to_packet()` 写入 mac₁/mac₂
2. **再**对含 MAC 的完整缓冲调用 `pqc_fragment_send()`

接收侧：`pqc_fragment_recv()` 重组出**完整握手包** → `wg_cookie_validate_packet()` 验 MAC → `noise.c` consume。

若先分片再算 MAC，则无法对原始 transcript 做 WireGuard 兼容的 MAC1 验证。

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

## 4. 分片头格式

定义：`pqc_fragment.h`

```c
struct pqc_frag_hdr {          // 8 字节，__packed
    __le16 magic;              // 0x5051 ("PQ")
    __le16 msg_id;             // 随机 handshake id
    __u8   frag_index;         // 0-based
    __u8   frag_total;         // 总片数
    __le16 frag_len;           // 本片载荷字节数
};
```

| 常量 | 值 | 含义 |
|------|-----|------|
| `PQC_FRAG_MAGIC` | 0x5051 | 识别 PQ 分片 |
| `PQC_FRAG_PAYLOAD_SIZE` | 1400 | 每片最大握手字节（不含 8 B 头） |
| `PQC_MAX_REASM_SLOTS` | 16 | 并发重组槽 |
| `PQC_FRAG_MAX_FRAGS` | 256 | 单消息最大片数 |

分片 ACK 使用 `frag_total=0xFF, frag_index=0xFF` 编码（`PQC_FRAG_ACK_*`），与普通分片头不冲突。

---

## 5. 发送逻辑（pqc_fragment_send）

`pqc_fragment.c` L450–492：

1. 若 `len ≤ 1400`：单包直发（`wg_socket_send_buffer_to_peer`）
2. 若 `len > 1400`：计算 `frag_total = ceil(len / 1400)`
3. 每片：写 `pqc_frag_hdr` + 复制 `frag_len` 字节载荷
4. 记录 pending 状态，支持 ACK 与选择性重传（`FRAG_ACK_TIMEOUT_MS=100`，最多 3 次）

主套件 Initiation 2068 B → **2 片**（1400 + 668）；Response 1916 B → **2 片**（1400 + 516）。

---

## 6. 接收逻辑（pqc_fragment_recv）

入口：`wg_packet_receive()` L607–627

1. 检查 skb 前 2 字节是否为 `magic == 0x5051`
2. 调用 `pqc_fragment_recv(wg, skb)` 写入重组槽
3. 片未到齐 → 释放分片 skb，等待后续片
4. 到齐 → 分配新 skb，拷贝完整握手缓冲（**不含**分片头），返回给上层
5. 上层按**普通握手包**走 MAC 验证与 noise consume

重组完成后 `skb->len` 等于原始握手 struct 长度（如 2068）。

---

## 7. 主套件 dmesg 实证

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

## 8. 编译开关

| 变量 | 效果 |
|------|------|
| `APP_LAYER_FRAGMENTATION=auto` | struct >1400B 时定义 `PQST_APP_LAYER_FRAGMENTATION` |
| `APP_LAYER_FRAGMENTATION=1` | 强制启用 |
| `APP_LAYER_FRAGMENTATION=0` | 禁用；超大包依赖 IP 分片（不推荐） |

注入路径：`build.sh` → `APP_LAYER_FRAG_MACRO` → `make KCFLAGS`。

---

## 9. 答辩要点

- **为何分片**：主套件 Initiation 2068 B > MTU 安全 UDP 载荷
- **为何应用层**：可控、防火墙友好；MAC 覆盖完整包后再切
- **谁分片**：Initiation + Response；Key Confirm 不分
- **如何验证**：dmesg 中 `fragmented into 2 parts` + `reassembled ... 2068/1916`

---

## 10. 交叉引用

| 主题 | 文档 |
|------|------|
| 报文尺寸 | [04](04-密码原语与套件.md) |
| send/receive 路径 | [05](05-内核实现路径.md) |
| 构建与 suite_meta | [07](07-工程架构与实验验证.md) |
