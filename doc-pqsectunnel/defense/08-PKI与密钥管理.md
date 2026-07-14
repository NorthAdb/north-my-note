# 08 — PKI 与密钥管理

> 状态：正文 | 权威源：CONTEXT.md + `certificate-management/` + `setup_configs.py`  
> 答辩主套件默认：**random 模式** | PKI 为可选能力

---

## 1. 先搞清：握手里有两类密钥

| 类型 | 存在哪里 | 谁生成 | 握手中的作用 |
|------|---------|--------|-------------|
| **Static KEM**（长期身份） | `.conf` / PKI 证书 | 部署前一次性生成或从证书提取 | ct₁、ct₃；Peer 互相识别 |
| **Ephemeral KEM**（临时） | 仅内核内存 | **每次握手**内核 `pqst_ephemeral_keygen` | E_i、ct₂；前向安全 |

**PKI 只解决 Static 身份从哪来**；Ephemeral 永远运行时随机，不在证书里（CONTEXT **PKI 模式**）。

---

## 2. 两种密钥来源（实验/部署）

| 模式 | 环境变量 | Static 密钥来源 | 本仓库默认 |
|------|---------|----------------|-----------|
| **random 模式** | `PQST_KEY_SOURCE=random`（默认） | `./pqst genkey` 或 `nist-mlkem512-genkey` | ✅ 主套件实验 |
| **PKI 模式** | `PQST_KEY_SOURCE=pki` | X.509 证书链 → `cert_key_store.txt` | 可选演示 |

```bash
# 默认（07 节 run.sh）
python3 ./setup_configs.py

# PKI 模式
sudo PQST_KEY_SOURCE=pki ./run.sh
```

---

## 3. certificate-management 是什么

独立 C 工具链，目录 `certificate-management/`，**不与 build.sh 链入** `.ko`；需单独 `make`。

| 能力 | 实现 | 作用 |
|------|------|------|
| 证书加载 | `cert_loader.c` | 读 PEM 证书 |
| 链验证 | `cert_verify.c` | Root → Intermediate → 终端实体 |
| 公钥提取 | `cert_extract_kem_public_key()` | 从 SPKI 提取 ML-KEM 裸公钥 |
| 落库 | `pubkey_store.c` | 写入 `cert_key_store.txt` |

**小白理解**：这是「发证机关后台」——验证证书真假，把里面的 ML-KEM 公钥抄到本地数据库，供后续 `setup_configs.py` 读取。

---

## 4. 证书链结构（本仓库示例）

```
Root CA (root.crt)
    │  ML-DSA-44 签名
    ▼
Intermediate CA (yunying.crt)
    │  ML-DSA-44 签名
    ├── user_enc.crt   ← 加密/ KEM 用途（握手 Static ML-KEM-512）
    └── user_sig.crt   ← 签名用途（ML-DSA-44，非握手 KEM）
```

| 证书 | 角色 | 握手是否使用 |
|------|------|-------------|
| `root.crt` | 根 CA | 仅验证链 |
| `yunying.crt` | 中间 CA | 仅验证链 |
| **`user_enc.crt`** | 用户**加密**证书，内嵌 ML-KEM-512 公钥 | ✅ Static KEM 身份来源 |
| `user_sig.crt` | 用户**签名**证书，内嵌 ML-DSA-44 | ❌ 证书签名用，不进 Noise 握手 |

**答辩必区分**（CONTEXT **ML-DSA-44** vs **ML-KEM-512**）：
- **ML-DSA-44**：证明「这张证书是 CA 签的」—— PKIX 信任
- **ML-KEM-512**：握手里真正做 KEM 封装的那把钥—— Noise 协议

---

## 5. cert_key_store.txt 格式

持久化文件：`certificate-management/certs/cert_key_store.txt`

每条记录一组 `key=value`，记录间用 `---` 分隔。关键字段：

| 字段 | 示例 | 含义 |
|------|------|------|
| `cert_name` | `user_enc.crt` | 源证书文件名 |
| `kem_alg` | `ML-KEM-512` | KEM 算法标识 |
| `cert_public_key_b64` | 800 字节解码后 | **写入 .conf 的 Static 公钥** |
| `cert_private_key_b64` | 1632 字节解码后 | 写入 .conf 的 Static 私钥（实验用） |

`user_enc.crt` 记录示例（节选）：

```
kem_alg=ML-KEM-512
cert_public_key_raw_len=800
cert_public_key_b64=VTUuUwRQKuWch7Q...
cert_private_key_raw_len=1632
```

生成方式（在 `certificate-management/` 目录）：

```bash
make clean && make
make test          # test_cert_loader.c 端到端：加载→验证→入库
# 或
./build/cert_parser certs/user_enc.crt
```

---

## 6. PKI 模式接入实验的流程

```
┌─────────────────────────────────────────────────────────────┐
│ 1. 离线：签发/准备 X.509 证书（user_enc.crt 等）               │
│ 2. certificate-management：验证链 + 提取公钥                  │
│    → cert_key_store.txt                                      │
└───────────────────────────┬─────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. run.sh：PQST_KEY_SOURCE=pki                               │
│    setup_configs.py → generate_pki_keys()                    │
│    读 cert_key_store 中 user_enc.crt 记录                     │
│    → pqst0.conf / server0.pqst0.conf（Static 钥来自证书）    │
└───────────────────────────┬─────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. pqst-quick up → Netlink 下发内核                          │
│ 5. pqsectunnel.ko / noise.c：握手协议与 random 模式完全相同   │
│    Ephemeral E_i 仍每次随机生成（768）                        │
└─────────────────────────────────────────────────────────────┘
```

### setup_configs.py PKI 分支（源码逻辑）

1. `PQST_CERT_KEY_STORE` 指向 `cert_key_store.txt`（默认自动查找 `certificate-management/certs/`）
2. `PQST_PKI_CLIENT_CERT` / `PQST_PKI_SERVER_CERT` 指定终端证书名（默认 `user_enc.crt`）
3. 读取 `cert_public_key_b64` / `cert_private_key_b64` 填入 Initiator/Responder 字段
4. 校验公钥 800B、私钥 1632B（与 ML-KEM-512 Static 一致）

**与 random 模式的唯一差别**：`.conf` 里 Static 钥来自证书库，而非 `./pqst genkey`  stdout。

---

## 7. random 模式（答辩主路径）简述

```bash
./pqst genkey
# 输出 InitiatorPublicKey / ResponderPublicKey / 对应 PrivateKey（Base64）
```

- 内核模块编译时 Static KEM = ML-KEM-512 → 公钥 800B
- `setup_configs.py` 校验长度后写入 `.conf`
- **不涉及** X.509、不涉及 ML-DSA-44

主套件 `run.sh` 默认走此路径；答辩实验演示用 random 即可。

---

## 8. 密钥在系统各层的分布

| 密钥材料 | random 模式 | PKI 模式 | 运行时位置 |
|---------|------------|---------|-----------|
| Static 公钥/私钥 | pqst genkey | cert_key_store | `.conf` → Netlink → 内核 `static_identity` |
| Ephemeral E_i | — | — | 仅内核 `noise_handshake`，每次握手刷新 |
| 会话 TK_i/TK_r | — | — | 握手完成后内核 `noise_keypair`，不写入磁盘 |

**安全提示**：实验用 `.conf` 和 `cert_key_store` 含明文私钥，仅用于 netns 测试，勿用于生产。

---

## 9. 答辩怎么说（30 秒）

> 我们支持两种 Static 身份来源：默认是 pqst 随机 genkey；可选 PKI 模式从 X.509 加密证书提取 ML-KEM-512 公钥写入配置。证书链用 ML-DSA-44 签名，与握手 KEM 分离。PKI 只影响配置下发，三次握手和内核 noise.c 路径不变；Ephemeral ML-KEM-768 仍每次握手临时生成。

---

## 10. 交叉引用

| 主题 | 文档 |
|------|------|
| Static/Ephemeral 角色 | [04](04-密码原语与套件.md)、[CONTEXT.md](../../CONTEXT.md) |
| run.sh / setup_configs | [07](07-工程架构与实验验证.md) |
| 握手如何用 Static 钥 | [03](03-握手协议逐步详解.md) |

## 11. 来源索引

| 内容 | 路径 |
|------|------|
| 工具链 README | `certificate-management/README.md` |
| 公钥库 | `certificate-management/certs/cert_key_store.txt` |
| PKI 配置生成 | `experiments/.../setup_configs.py` `generate_pki_keys()` |
| 示例证书 | `certificate-management/certs/user_enc.crt` |
