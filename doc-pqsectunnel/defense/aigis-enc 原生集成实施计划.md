# AIGIS-ENC 原生集成实施计划

> 范围：把 AIGIS-ENC **mode 1**（K=2，pk=672，ct=736，sk=1568）作为 **static KEM** 接入 PQ-SecTunnel，
> ephemeral 继续走现有 ML-KEM-512，端到端握手通过 `PQST_KEY_SOURCE=random` 生成密钥。
> 暂不考虑证书支持。mode 2/3/4 留为后续扩展。
>
> 算法等级选择理由：mode 1 尺寸全部 ≤ ML-KEM-512（800/768/1632），握手包大小与现有
> `mlkem512-mlkem512-*` 量级一致甚至更小，分片策略无需变更。
>
> 实验目录命名：`aigis1-mlkem512-sm4gcm-sm3`（国密套件：SM4GCM + SM3 + AIGIS-ENC）。
> 内核 KEM 符号链接目标：`kem -> kems/aigis-enc`；编译宏：`STATIC_KEM_AIGIS1`。

---

## 0. 背景与现状回顾

### 0.1 现有 KEM 抽象点（决定接入方式）

- 内核侧 KEM 接口非常薄：每个算法在 `kems/<alg>/` 下提供 `api.h` / `kem.h` / `Kbuild.include`，
  通过三个 inline 函数 `kem_keygen / kem_encapsulate / kem_decapsulate` 和四个常量
  `KEM_PUBLIC_KEY_SIZE / KEM_PRIVATE_KEY_SIZE / KEM_CIPHERTEXT_SIZE / KEM_SS_SIZE` 接入。
- `build.sh` 在 `wireguard-linux-compat/src/` 下创建符号链接 `kem -> kems/<STATIC_KEM>`，
  由 `messages.h` 用 `#if defined(STATIC_KEM_MLKEM512/768)` 选择尺寸，
  `noise.c` 用同样宏在四处（`1053 / 1185 / 1353 / 1473` 行）选择静态 KEM 的封装函数。
- 临时 KEM 走 `EPHEMERAL_KEM_MLKEM768` 分支，与本次 AIGIS 集成不冲突。
- 用户态 `pqst` 沿用同一套 `kem/kem.h`；纯静态 ML-KEM 走独立 `nist-mlkem*-genkey` 二进制生成静态密钥对。
- 密钥来源在 `resources/setup_configs.py` 已分 `random`（调 `./pqst genkey`）和 `pki`两条；
  本次直接复用 `random` 路径，无需触碰证书链。
- 仓库已存在 PQMagic 源码：`/home/northadb/桌面/UsrApps/pqmagic/kem/aigis-enc/std/`，
  算法本体 10 个 C 文件、约 1700 行；自带的 SM3 后端位于 `pqmagic/hash/sm3/`。

### 0.2 关键约束

**内核模块里跑不了 OpenSSL provider / libcrypto。** `pqsectunnel.ko` 运行在内核态，
握手封装/解封装在 `noise.c` 内调用，必须用可链入内核的 C 源码（自带 `__KERNEL__` 分支、
`get_random_bytes_wait` 随机源、无 libc）。OpenSSL 的 `aigis.so` / `pqmagic.so` 是用户态
ELF `.so`，无法供 `noise.c` 调用。因此 **provider 路线只能解决用户态 genkey**，
内核封装/解封装必须有一份可链入内核的 AIGIS-ENC C 源码。

→ 选定 **方案 A：原生集成**（仿 `kems/mlkem-512/`），稍带一份 SM3 后端进 kernel。
多维护一份用户态 provider 桥对当前"无证书"场景没有收益。

---

## 1. 源码现状（PQMagic 原版摘要）

### 1.1 AIGIS-ENC 源树

`pqmagic/kem/aigis-enc/std/`：

| 文件 | 行数 | 作用 |
|---|---:|---|
| `kem.c` | 142 | KEM 三函数（keypair / enc / dec） |
| `owcpa.c` | 157 | OWCPA 加密本体 |
| `poly.c` | 469 | 多项式运算 |
| `polyvec.c` | 344 | 多项式向量运算 |
| `ntt.c` | 137 | NTT 与逆 NTT（含 zetas 常量表） |
| `cbd.c` | 165 | 中心二项分布噪声生成 |
| `reduce.c` | 40 | Montgomery / Barrett / caddq 归约 |
| `verify.c` | 49 | 常数时间比较与 cmov |
| `genmatrix.c` | 33 | 矩阵 A 生成 |
| `hashkdf.c` | 63 | Hash/KDF 桥（默认 SM3，可选 SHA-3） |
| `sm3kdf.c` | 37 | SM3-based KDF 状态机 |

四档参数（`params.h`）：

| Mode | K | pk | ct | sk | SS | 对应关系 |
|---|---:|---:|---:|---:|---:|---|
| 1 | 2 | **672** | **736** | 1568 | 32 | AIGIS-ENC-128（本次接入） |
| 2 | 3 | 896 | 992 | 2208 | 32 | AIGIS-ENC-192 备选 |
| 3 | 3 | 992 | 1056 | 2304 | 32 | AIGIS-ENC-192 备选 |
| 4 | 4 | 1088 | 1184 | 2944 | 32 | AIGIS-ENC-256 |

mode 1 全部尺寸 < ML-KEM-512，握手包：
- init = 116 (fixed) + 800 (eph pk) + 736 (static ct) = **1652 字节**
- resp = 60 (fixed) + 768 (eph ct) + 736 (static ct) = **1564 字节**
- 两者 > 1400，自动启用 APP 层分片（与 `mlkem512-mlkem512-*` 同行为）。

### 1.2 哈希后端

- 默认 **SM3**（`hashkdf.c` 不定义 `USE_SHAKE`），位于 `pqmagic/hash/sm3/`：
  - `x86-64/sm3.c` (276 行)：纯标量 C 实现，仅 `ROTATELEFT` 位旋转 + `uint32_t Ti[64]` 表，无 SIMD/atomics。
  - `sm3_extended.c` (67 行)：SM3 任意长度输出扩展。
  - `x86-64/include/sm3.h`、`include/sm3_extended.h`：头文件。
- 与 PQ-SecTunnel 协议层 `NOISE_HASH_SM3`（走内核 `crypto_alloc_shash("sm3")`）天然契合，
  双方都用国密 SM3，整套套件 `SM4GCM + SM3 + AIGIS-ENC` 是端到端国产后量子方案。

### 1.3 已识别的内核适配点

| 风险点 | 位置 | 处理 |
|---|---|---|
| `#include <stdio.h>` 引而未用 | `poly.c:1` `polyvec.c:1` `owcpa.c:1` `cbd.c:1` | 删除 |
| `#include <stdlib.h>` 引而未用 | `sm3kdf.c:3` | 删除 |
| `<stdio.h>/<stdlib.h>/<string.h>` 内核不可用 | `sm3.h:5-8` | 改 `__KERNEL__` 分支用 `<linux/string.h>` |
| `<immintrin.h>` 引而未用 | `cbd.c:3-5` `polyvec.c:7-9` | 加 `&& !defined(__KERNEL__)` 守卫 |
| **`malloc/free` 实际使用** | `sm3_extended.c:45-46` | **必改**：重写为 VLA 栈缓冲（详见阶段 2.5） |
| `*(uint64_t*)&buf[...]` type-punning | `cbd.c:31,54,78,108` | 默认 OK；若 lint 触发改 `get_unaligned` |
| SM3 平台守卫 `#error` | `sm3_extended.c:1-9` | 仅 i386/x86_64/arm/aarch64 受限；内核 x86_64 OK，删硬路径 |
| namespace 宏 `AIGIS_ENC_NAMESPACE(s)` | `config.h` | 为 mode 1 提供 `pqmagic_aigis_enc_1_std_*` 符号隔离 |

---

## 2. 详细实施阶段

### 阶段 0｜源码可行性收尾（0.5 d）

**目的**：在拷贝前把所有会爆的依赖全部识别、出落地方案。

**任务**：

- 0.1 列出所有 libc 依赖（见 1.3 表），形成"源码改动清单"。
- 0.2 **`hash/sm3/sm3_extended.c` 用 `malloc/free`**（第 45–46 行）：内核最大障碍。调用面分析：
  - `Hash = hash256`（out=32，in≤672）→ `in_ext=36B, running_res=64B`，共 ~100B 栈
  - `Hash2 = hash512`（out=64，in≤32）→ `in_ext=36B, running_res=96B`
  - `kdf256` 在 `polyvec_ee_getnoise` mode 1 走 ETA_E=8 → `outlen=512` → `running_res=544B`
  - 所有范围内单次调用 ≤ ~600B 栈，合计上 owcpa 现有帧仍 < 4 KB
  → **决策：改写为 `__KERNEL__` 分支用 VLA 栈缓冲**，用户态仍保留 malloc 兼容现有 PQMagic build。落地见阶段 2.5。
- 0.3 评估 `cbd.c` type-punning：内核 GCC `-O3` 默认不严格 alias 校验；决定保持现状，若 lint 触发在阶段 5 收口。
- 0.4 确认 `<immintrin.h>` 未实际使用 intrinsics：改为 `#if defined(__x86_64__) && !defined(__KERNEL__)`。

**交付**：源码改动清单（临时记在 `docs/其他/aigis-enc 集成笔记.md` 或 commit message）。

**验证**：在 PQMagic 原路径 `cd pqmagic/kem/aigis-enc/std && make test/test_aigis_enc_1` 跑通 KAT，作为后续 bug 排查基线。

---

### 阶段 1｜新建 `kems/aigis-enc/` 容器（0.5 d）

**目的**：建立目录结构、拷源、不接 kernel build。

**目录布局**：

```
kems/aigis-enc/
  api.h                    # 阶段 2 写
  kem.h                    # 阶段 2 写
  Kbuild.include           # 阶段 2 写
  randombytes.c            # 阶段 2 写（拷自 kems/mlkem-512/randombytes.c）
  config.h                 # 拷 + 改
  params.h                 # 拷（4 个 PARAMS 分支保留，仅 mode 1 被编）
  align.h                  # 拷
  cbd.{c,h}                # 拷 + 改
  genmatrix.{c,h}          # 拷 + 改
  hashkdf.{c,h}            # 拷 + 改
  kem.c                    # 拷 + 改（加 aigis1_* 包装）
  ntt.{c,h}                # 拷 + 改
  owcpa.{c,h}              # 拷 + 改
  poly.{c,h}               # 拷 + 改
  polyvec.{c,h}            # 拷 + 改
  reduce.{c,h}             # 拷 + 改
  verify.{c,h}             # 拷 + 改
  sm3kdf.{c,h}             # 拷 + 改
  sm3/
    sm3.c                  # 拷自 pqmagic/hash/sm3/x86-64/sm3.c
    sm3.h                  # 拷自 pqmagic/hash/sm3/x86-64/include/sm3.h
    sm3_extended.c         # 拷 + 改写（去 malloc）
    sm3_extended.h         # 拷自 pqmagic/include/sm3_extended.h
```

**统一处理**（拷贝时）：
- 删 `#include <stdio.h>` / `#include <stdlib.h>`
- 顶部加 `#ifdef __KERNEL__` `#include <linux/string.h>` `<linux/types.h>` `#else` 原用户态 include `#endif`

**交付**：`kems/aigis-enc/` 完整源树（不接 build.sh）。

**验证**：在 `kems/aigis-enc/` 下写临时 `Makefile.test`，纯用户态编译 mode 1 三函数 + KAT，确认拷贝无破坏。

---

### 阶段 2｜写 api.h / kem.h / Kbuild.include / randombytes.c（0.5 d）

**目的**：与 `noise.c` KEM 抽象点对齐，使 `kem_keygen/encapsulate/decapsulate` 三个符号可用。

**任务**：

- 2.1 **`kems/aigis-enc/api.h`** 暴露固定 mode 1 别名（避免依赖 namespace 宏）：
  ```c
  #define AIGIS1_CRYPTO_PUBLICKEYBYTES  672
  #define AIGIS1_CRYPTO_SECRETKEYBYTES  1568
  #define AIGIS1_CRYPTO_CIPHERTEXTBYTES 736
  #define AIGIS1_CRYPTO_BYTES           32
  #define AIGIS1_CRYPTO_ALGNAME         "AIGIS-ENC-1"

  int aigis1_keypair(uint8_t *pk, uint8_t *sk);
  int aigis1_enc(uint8_t *ct, uint8_t *ss, const uint8_t *pk);
  int aigis1_dec(uint8_t *ss, const uint8_t *ct, const uint8_t *sk);
  ```
  实现：在 `kem.c` 加 `#if AIGIS_ENC_MODE==1` 包装层，调用 `crypto_kem_*` 宏展开后的符号。

- 2.2 **`kems/aigis-enc/kem.h`** 完全仿 `kems/mlkem-512/kem.h`：
  ```c
  enum kem_lengths {
      KEM_PUBLIC_KEY_SIZE  = AIGIS1_CRYPTO_PUBLICKEYBYTES,
      KEM_PRIVATE_KEY_SIZE = AIGIS1_CRYPTO_SECRETKEYBYTES,
      KEM_SS_SIZE          = AIGIS1_CRYPTO_BYTES,
      KEM_CIPHERTEXT_SIZE  = AIGIS1_CRYPTO_CIPHERTEXTBYTES,
  };

  static inline bool kem_keygen(u8 pk[KEM_PUBLIC_KEY_SIZE], u8 sk[KEM_PRIVATE_KEY_SIZE]) {
      return aigis1_keypair(pk, sk) == 0;
  }
  static inline bool kem_encapsulate(u8 ss[KEM_SS_SIZE], u8 ct[KEM_CIPHERTEXT_SIZE],
                                      const u8 pk[KEM_PUBLIC_KEY_SIZE], struct kem_buffer *buf) {
      (void)buf;
      return aigis1_enc(ct, ss, pk) == 0;
  }
  static inline bool kem_decapsulate(u8 ss[KEM_SS_SIZE], const u8 ct[KEM_CIPHERTEXT_SIZE],
                                      const u8 sk[KEM_PRIVATE_KEY_SIZE], struct kem_buffer *buf) {
      (void)buf;
      return aigis1_dec(ss, ct, sk) == 0;
  }

  struct kem_buffer { };
  ```
  双分支 `__KERNEL__` / 非 `__KERNEL__`（仿 mlkem-512 第 6–17 行）。

- 2.3 **`kems/aigis-enc/randombytes.c`**：从 `kems/mlkem-512/randombytes.c` 直接复制，
  已写好 `__KERNEL__ → get_random_bytes_wait` / 用户态 `getrandom` 双分支。**0 改动**。

- 2.4 **`kems/aigis-enc/Kbuild.include`** 仿 `kems/mlkem-512/Kbuild.include`：
  ```make
  ccflags-y += -I$(src)/kems/aigis-enc -I$(src)/kems/aigis-enc/sm3
  pqsectunnel-y += kems/aigis-enc/cbd.o
  pqsectunnel-y += kems/aigis-enc/genmatrix.o
  pqsectunnel-y += kems/aigis-enc/hashkdf.o
  pqsectunnel-y += kems/aigis-enc/kem.o
  pqsectunnel-y += kems/aigis-enc/ntt.o
  pqsectunnel-y += kems/aigis-enc/owcpa.o
  pqsectunnel-y += kems/aigis-enc/poly.o
  pqsectunnel-y += kems/aigis-enc/polyvec.o
  pqsectunnel-y += kems/aigis-enc/reduce.o
  pqsectunnel-y += kems/aigis-enc/sm3kdf.o
  pqsectunnel-y += kems/aigis-enc/verify.o
  pqsectunnel-y += kems/aigis-enc/randombytes.o
  pqsectunnel-y += kems/aigis-enc/sm3/sm3.o
  pqsectunnel-y += kems/aigis-enc/sm3/sm3_extended.o
  ```
  关键：**不复用** `kems/mlkem-512/common/fips202.o`——AIGIS-ENC 走 SM3 后端，
  ML-KEM 走 SHA-3 后端，两套独立 object 不冲突。

- 2.5 **`sm3_extended.c` 重写**：原版 `malloc/free` 必须去除。改写为 VLA 栈缓冲：
  ```c
  void sm3_extended(uint8_t *out, size_t outlen, const uint8_t *in, size_t inlen) {
      uint8_t in_ext[inlen + 4];                  /* VLA 栈 */
      uint8_t running[outlen + SM3_DIGEST_LENGTH]; /* 最坏 ~600B */
      uint32_t nonce = 0;
      uint8_t *res = running;
      int64_t running_outlen = (int64_t)outlen;

      memcpy(in_ext + 4, in, inlen);
      while (running_outlen > 0) {
          in_ext[0] = (uint8_t)(nonce >> 24) & 0xff;
          in_ext[1] = (uint8_t)(nonce >> 16) & 0xff;
          in_ext[2] = (uint8_t)(nonce >> 8)  & 0xff;
          in_ext[3] = (uint8_t)(nonce)       & 0xff;

          sm3(in_ext, inlen + 4, running);
          running   += SM3_DIGEST_LENGTH;
          running_outlen -= SM3_DIGEST_LENGTH;
          nonce++;
      }
      memcpy(out, res, outlen);
  }
  ```
  在内核栈大小 OK（前述最大 ~600B）；用户态同样可编。
  验证：写一个简短 unit test 校 `hash256(672B) → 32B`、`hash512(32B) → 64B` 与 PQMagic 原版字节一致。

**交付**：`kems/aigis-enc/{api.h, kem.h, Kbuild.include, randombytes.c}` + `sm3/sm3_extended.c` 栈版本。

**验证**：把 `kems/aigis-enc/` 临时挂成独立用户态 build（`-DAIGIS_ENC_MODE=1`），跑 KAT 向量对齐 PQMagic 原版输出。

---

### 阶段 3｜接入 `build.sh`（0.5 d）

**目的**：让 `sudo ./build.sh -e aigis1-mlkem512-sm4gcm-sm3` 能产出 `pqsectunnel.ko`。

**任务**：

- 3.1 `build.sh:601–610` 的 `STATIC_KEM_VAL` 分发 case 加分支：
  ```bash
  elif [[ "$STATIC_KEM_VAL" == "aigis1" ]]; then
      KEM="aigis-enc"
      STATIC_KEM_MACRO="-DSTATIC_KEM_AIGIS1"
  ```

- 3.2 `build.sh:612–619` 的 ephemeral 范围检查不动；`aigis1` static 时 `EPHEMERAL_KEM_VAL=mlkem512` 符合现有规则。

- 3.3 `build.sh:723–730` 用户态 `pqst` 的 `kem` symlink：当前 mceliece 走 `wireguard-tools/src/kems/mceliece`，
  其它走 `kems/$KEM`。`aigis-enc` 走后者，无需新增分支。

- 3.4 用户态 genkey helper 分发：`build.sh:736–745` 现有两段是 `mlkem512` / `mlkem768`，加一段：
  ```bash
  if [[ "$STATIC_KEM_VAL" == "aigis1" ]]; then
      make -B -j3 nist-aigis1-genkey
      cp "nist-aigis1-genkey" "${experiment_dir}"
  fi
  ```
  （阶段 4 会建出 `nist-aigis1-genkey` target）

- 3.5 `build.sh:505–551` 的 `pqst_resolve_app_layer_fragmentation` 的 `auto` 分支：
  - 第 524 行 `mceliece_path` 仍只对 `STATIC_KEM_VAL == "mceliece"` 置 1
  - 第 526–531 行加 `aigis1` 处理：`static_macro="STATIC_KEM_AIGIS1"`

- 3.6 `build.sh:469–503` 的 `pqst_handshake_message_sizes`：
  - 当前 `static_macro` 仅识别 `STATIC_KEM_MLKEM768`，否则 `static_ct=$mlk512_ct`
  - 加分支：`[[ "$static_kem_macro" == "STATIC_KEM_AIGIS1" ]] && static_ct=$aigis1_ct`
  - 在变量区加 `local aigis1_pk=672 aigis1_ct=736`
  - mode 1 eph 仍 mlkem512：`eph_pk=$mlk512_pk eph_ct=$mlk512_ct`
  - 结果 `init_sz=116+800+736=1652`，`resp_sz=60+768+736=1564`，两者 > 1400，
    APP 层分片自动开启（与 `mlkem512-mlkem512-*` 行为一致）

- 3.7 `build.sh:666–673` AEAD/HASH case：不动（SM4GCM/SM3 已存在）。

**交付**：`build.sh` 4 个分支点

**验证**：
- `sudo ./build.sh -e aigis1-mlkem512-sm4gcm-sm3 --keep-experiments`
- 期望 `experiments/aigis1-mlkem512-sm4gcm-sm3/pqsectunnel.ko` 产出
- `strings pqsectunnel.ko | grep -i aigis` 应看到 `AIGIS-ENC-1` / `aigis1_*` 符号

---

### 阶段 4｜接入 `messages.h` / `noise.c`（0.5 d）

**目的**：让握手消息尺寸与 static KEM 封装调用在编译期走 AIGIS 分支。

**任务**：

- 4.1 `wireguard-linux-compat/src/messages.h:57` 扩展 `#if` 链：
  ```c
  #if defined(STATIC_KEM_MLKEM512) || defined(STATIC_KEM_MLKEM768) || defined(STATIC_KEM_AIGIS1)
  ```
  所有尺寸来自 `kem/kem.h`（即 `kems/aigis-enc/kem.h` via symlink），mode 1 时
  `KEM_PUBLIC_KEY_SIZE=672` 等自动正确。

- 4.2 ephemeral 分支：`messages.h:66` `#if defined(EPHEMERAL_KEM_MLKEM768)` 仍只对 mlkem768；
  AIGIS1 path 走第 76 行 `#else` 用 `KEM_*`（= mlkem-512 尺寸）。**无须改动**。

- 4.3 `noise.c:25–44` 的 `#include "kems/.../kem_*.h"` 与 `pqst_ephemeral_*` 宏：
  aigis1 不需要 768 头，第 41–43 行 `#else` 分支 `pqst_ephemeral_keygen=kem512_*` 自动生效。**无须改动**。

- 4.4 `noise.c:24` `#include "kems/mceliece/api.h"`：历史遗留，对 AIGIS 路径无意义。
  - **风险点**：`kems/mceliece/api.h` 引入 `CRYPTO_PUBLICKEYBYTES=...`，与 AIGIS `api.h` 的
    `#define CRYPTO_PUBLICKEYBYTES KEM_PUBLICKEYBYTES` 撞名。
  - 修法：用 `#if defined(STATIC_KEM_MCELIECE) || ...` 守起来。

- 4.5 `noise.c:1053` 的 `#if defined(STATIC_KEM_MLKEM512) || defined(STATIC_KEM_MLKEM768)`：
  扩展为 `|| defined(STATIC_KEM_AIGIS1)`

- 4.6 `noise.c:1185, 1353, 1473` 同样三处扩展

- 4.7 检查 `wireguard-linux-compat/src/uapi/pqwireguard.h:139–145` 是否需加 AIGIS 分支
  （同 mlkem768 写法）

**交付**：`messages.h`、`noise.c`、可能 `pqwireguard.h` 共 ~6 处 `#if` 扩展

**验证**：
- `make -B` 编译 `pqsectunnel.ko` 不报错
- `modinfo pqsectunnel.ko` 看到新模块
- `readelf -s pqsectunnel.ko | grep aigis1` 应有 `aigis1_keypair/enc/dec` 符号

---

### 阶段 5｜用户态 `pqst` 与 `nist-aigis1-genkey`（0.5 d）

**目的**：让 `pqst genkey` 产出 AIGIS-1 静态密钥对，复用 `PQST_KEY_SOURCE=random` 路径。

**任务**：

- 5.1 `wireguard-tools/src/Makefile:118`：
  ```make
  PQST_C_SOURCES := $(filter-out nist_mlkem512_genkey_main.c nist_mlkem768_genkey_main.c nist_aigis1_genkey_main.c,$(wildcard *.c))
  ```
  并加 `nist-aigis1-genkey` target，仿 `nist-mlkem512-genkey`（第 121–141 行）：
  ```make
  AIGIS1_SRC := $(PQ_ROOT)/kems/aigis-enc
  NIST_AIGIS1_GENKEY_CFLAGS := -std=gnu99 -D_GNU_SOURCE -O3 -Wall \
      -iquote $(AIGIS1_SRC) -iquote $(AIGIS1_SRC)/sm3 -DAIGIS_ENC_MODE=1

  nist_aigis1_srcs := cbd.c genmatrix.c hashkdf.c kem.c ntt.c owcpa.c \
      poly.c polyvec.c reduce.c sm3kdf.c verify.c randombytes.c
  nist_aigis1_objs := $(addprefix obj/nist_aigis1/,$(nist_aigis1_srcs:.c=.o)) \
      obj/nist_aigis1/sm3.o obj/nist_aigis1/sm3_extended.o

  obj/nist_aigis1/%.o: $(AIGIS1_SRC)/%.c
	  @mkdir -p $(dir $@)
	  $(CC) $(NIST_AIGIS1_GENKEY_CFLAGS) -c -o $@ $<

  obj/nist_aigis1/sm3.o: $(AIGIS1_SRC)/sm3/sm3.c
	  @mkdir -p $(dir $@)
	  $(CC) $(NIST_AIGIS1_GENKEY_CFLAGS) -c -o $@ $<

  obj/nist_aigis1/sm3_extended.o: $(AIGIS1_SRC)/sm3/sm3_extended.c
	  @mkdir -p $(dir $@)
	  $(CC) $(NIST_AIGIS1_GENKEY_CFLAGS) -c -o $@ $<

  nist_aigis1_genkey_main.o: nist_aigis1_genkey_main.c
	  $(CC) $(NIST_AIGIS1_GENKEY_CFLAGS) -c -o $@ $<

  nist-aigis1-genkey: $(nist_aigis1_objs) nist_aigis1_genkey_main.o
	  $(CC) $(LDFLAGS) -o $@ $^ $(LDLIBS)
  ```
  clean 部分加 `nist-aigis1-genkey` 和 `obj/nist_aigis1`。

- 5.2 新建 `wireguard-tools/src/nist_aigis1_genkey_main.c`，仿 `nist_mlkem512_genkey_main.c`：
  调 `aigis1_keypair()` 两次（initiator + responder），按 base64 输出
  `InitiatorPublicKey / InitiatorPrivateKey / ResponderPublicKey / ResponderPrivateKey` 4 行。

- 5.3 `wireguard-tools/src/nist_aigis1_helper.{c,h}` 仿 `nist_mlkem512_helper.{c,h}`：
  `pqst genkey` 内部 spawn `./nist-aigis1-genkey` 取密钥（先走 spawn 方式，与 mlkem512 一致）。

- 5.4 `wireguard-tools/src/genkey.c:118–119` 当前是
  ```c
  kem_keygen(ipkey, isk);
  kem_keygen(rpkey, rsk);
  ```
  判断 `genkey.c:118` 上下文：当前 mlkem512 在 `#if defined(STATIC_KEM_MLKEM512)` 分支下走 helper，
  否则 inline。补 `#elif defined(STATIC_KEM_AIGIS1)` 走 `nist_aigis1_helper` 路径。

- 5.5 `resources/setup_configs.py:68–86` 的 `check_genkey_helper` 加分支：
  ```python
  elif static_kem == "aigis1":
      helper = Path("nist-aigis1-genkey")
      if not helper.exists():
          raise FileNotFoundError(
              f"STATIC_KEM=aigis1 需要 {helper.name}（请 rebuild 实验目录）"
          )
      if not os.access(helper, os.X_OK):
          raise PermissionError(f"{helper.name} 无执行权限")
  ```

- 5.6 `resources/setup_configs.py:18–21` 加常量：
  ```python
  EXPECTED_AIGIS1_PUBLIC_KEY_LEN = 672
  EXPECTED_AIGIS1_PRIVATE_KEY_LEN = 1568
  ```
  `validate_static_key_lens` 第 49 行 `elif static_kem == "aigis1"` 分支对应。

**交付**：
- `Makefile`、`nist_aigis1_genkey_main.c`、`nist_aigis1_helper.{c,h}`、`genkey.c` 小改、`setup_configs.py` 加分支
- `experiments/aigis1-mlkem512-sm4gcm-sm3/nist-aigis1-genkey` 产出

**验证**：
- `./nist-aigis1-genkey` 单独跑，base64 解码校验 pk=672B / sk=1568B
- `cd experiments/aigis1-mlkem512-sm4gcm-sm3 && PQST_KEY_SOURCE=random ./setup_configs.py` 生成 conf，
  校验 4 个 key 字节正确

---

### 阶段 6｜新建 wgtest vars + 端到端（0.5 d）

**目的**：定义实验配置、跑 netns 握手。

**任务**：

- 6.1 新建 `wgtest/wgtest-aigis1-mlkem512-sm4gcm-sm3/vars`：
  ```bash
  # AIGIS-ENC-1 静态 + ML-KEM512 临时
  export STATIC_KEM=aigis1
  export EPHEMERAL_KEM=mlkem512
  export AEAD=SM4GCM
  export HASH=SM3
  export NOISE_HANDSHAKE_NAME="Noise_IKpsk1kem_SM4GCM_SM3_StaticAIGIS1"
  export NOISE_IDENTIFIER_NAME="PQWireGuard v2 zx2c4 Jason@zx2c4.com"
  export NOISE_MAC1_KEY_LABEL="mac1----"
  ```

- 6.2 `sudo ./build.sh -e aigis1-mlkem512-sm4gcm-sm3 --keep-experiments`

- 6.3 `sudo ./test.sh --network` 走 netns 握手 + iperf3

- 6.4 失败排查清单（按概率降序）：
  - **a)** `CRYPTO_*BYTES` 宏名冲突（阶段 4.4）：症状 `enum kem_lengths` 重复定义或 `KEM_PUBLIC_KEY_SIZE` 值错。
    修法：条件 include `mceliece/api.h`。
  - **b)** SM3 KAT 不对齐：症状握手 hash mismatch。修法：阶段 2.5 的 `sm3_extended` 与 PQMagic 原版字节对齐验证。
  - **c)** 栈过大告警（`-Wframe-larger-than=`）：阶段 0.2 已预算 < 4 KB；若仍报，按 `build.sh:5–7` 注释容忍。
  - **d)** `setup_configs.py` 校验 672/1568 字节不通过：阶段 5.6 helper 常量没加。

**交付**：
- `wgtest/wgtest-aigis1-mlkem512-sm4gcm-sm3/vars`
- `experiments/aigis1-mlkem512-sm4gcm-sm3/` 完整产物
- 握手成功 `dmesg` 日志 + iperf3 throughput 报告

**验收门槛**：
- `sudo ./build.sh -e aigis1-mlkem512-sm4gcm-sm3 --keep-experiments` 退出码 0
- `sudo ./test.sh --network` 通过握手测试
- `dmesg | grep -i aigis` 看到模块加载日志含 `AIGIS-ENC-1`
- iperf3 throughput 与 `mlkem512-mlkem512-sm4gcm-sm3` 同量级（差异 < 30%），
  AIGIS-1 帧更小理论上略快

---

### 阶段 7｜收尾（0.5 d）

**任务**：

- 7.1 `build.sh:22–27` 的 `EXPERIMENTS=()` 默认列表**先不加** `aigis1-mlkem512-sm4gcm-sm3`
  （保留默认四人组）；仅在文档中说明按需 `-e` 触发。

- 7.2 更新 `CLAUDE.md`：
  - 第 56 行 `kems/` 列表加 `aigis-enc`
  - 第 46 行 `STATIC_KEM` 值枚举加 `aigis1`
  - 第 52 行 "Pure ML-KEM512 static ..." 段后补：
    > AIGIS-ENC-1 static + 512 ephemeral: init=1652, resp=1564（启用 APP 层分片）

- 7.3 在 `docs/其他/` 留一份 `aigis-enc 集成笔记.md`，记录：
  - 阶段 0.1 改动清单
  - `sm3_extended` 改写理由与 VLA 方案
  - mode 2/3/4 后续迁移要点

- 7.4 OpenSpec（可选）：若团队约定按 OpenSpec 流程，
  `/opsx:propose` 一条 `add-aigis-enc-static-kem` change 并把本计划作为 design 输入。

**交付**：CLAUDE.md 更新 + 集成笔记 + (可选) OpenSpec change 归档

---

## 3. 工作量汇总

| 阶段 | 估时 | 关键产物 |
|---|---|---|
| 0 源码可行性收尾 | 0.5 d | libc 清单 + `sm3_extended` 改写方案 |
| 1 拷源建容器 | 0.5 d | `kems/aigis-enc/` 源树 |
| 2 三件套 + randombytes + sm3_extended | 0.5 d | api.h / kem.h / Kbuild.include / sm3_extended |
| 3 接入 `build.sh` | 0.5 d | 4 个分支 |
| 4 接入 `messages.h` / `noise.c` | 0.5 d | 6 处 `#if` 扩展 |
| 5 用户态 genkey + setup_configs | 0.5 d | `nist-aigis1-genkey` + helper |
| 6 wgtest vars + 端到端 | 0.5 d | 握手通过 |
| 7 文档收尾 | 0.5 d | CLAUDE.md 更新 |
| **合计** | **3.5 d** | mode 1 fully working |

不确定性 buffer（首次跑通链路排查 macro / `#if` 漏分支）：+1~2 d。
**总计 4–5 person-day**。

---

## 4. 后续扩展（不在本次范围）

- **mode 2/3（AIGIS-ENC-192）**：阶段 2 的 `Kbuild.include` 改为 per-mode object 子目录
  （仿 mceliece 之于 mlkem 的"再加一组 .o"），`messages.h` / `noise.c` 的 `#if` 链补
  `STATIC_KEM_AIGIS2/3`。再 2–3 d。
- **mode 4（K=4）**：同理。
- **AIGIS-ENC 作为 ephemeral KEM**：当前 ephemeral 分支死锁 mlkem；扩 `EPHEMERAL_KEM_AIGIS` 类似工程量 2 d。
- **PKI/证书**：AIGIS public key 是 672B 定长字节串，与 `certificate-management/` 嵌 ML-KEM public key
  流程同构；provider 路线此时才值得讨论。

---

## 5. 风险登记表

| # | 风险 | 阶段 | 缓解 |
|---|---|---|---|
| R1 | `mceliece/api.h` 的 `CRYPTO_*BYTES` 与 AIGIS `api.h` 宏撞名 | 4 | 编译第一次报错即可定位；条件 include |
| R2 | `sm3_extended` 改写后字节输出与 PQMagic 原版不一致 | 2 | KAT 向量先独立验证 |
| R3 | `*(uint64_t*)&buf` type-punning 触发 `-Wstrict-aliasing` | 5 | 改 `get_unaligned` 一次性收口 |
| R4 | 栈帧超出 `-Wframe-larger-than=` 阈值 | 6 | 沿用 `build.sh:5–7` 的容忍策略 |
| R5 | mode 1 eph=mlkem512 时响应消息 1564B 仍 > MTU | 6 | 阶段 3.5 自动开 APP 层分片 |
| R6 | `genkey.c` 现有 `kem_keygen` inline 路径与 helper 路径分支错乱 | 5 | 仿 mlkem512 风格补 `#elif STATIC_KEM_AIGIS1` |

---

## 6. 关键路径

```
阶段 0 (可行性)  →  阶段 1 (拷源)  →  阶段 2 (三件套)
                                            │
                                            ├─ KAT 通过 → 阶段 3 (build.sh)
                                            │                  ↓
                                            │           阶段 4 (messages.h/noise.c)
                                            │                  ↓
                                            │           阶段 5 (pqst + genkey)
                                            │                  ↓
                                            │           阶段 6 (端到端握手)  ★
                                            │                  ↓
                                            └─ KAT 失败 → 回阶段 2.5 修 sm3_extended
                                                              ↓
                                                       阶段 7 (文档收尾)
```

**阶段 0–2 是关键路径**，做完就能判断后续是否需要返工；阶段 3–6 都是机械适配。