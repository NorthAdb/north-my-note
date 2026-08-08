# 基于 openHiTLS 的 nginx 1.29.6 适配与单端口国密/国际自适应研究文档

> 课题：基于 openHiTLS 的 nginx 1.29.6 版本适配与 HTTPS 能力，并实现支持国密/国际协议单端口自适应
> 本文档为课题前期调研与方案设计的基础材料，持续更新。

---

## 1. 课题概述

### 1.1 拟解决问题

- Web 服务安全通信需同时支持国际算法体系（TLS 1.2/1.3）与国密算法体系（TLCP，SM2/SM3/SM4）
- 传统多端口部署（443 国际 + 8443 国密）增加运维复杂度与资源开销
- nginx 作为主流 Web 服务器，其 HTTPS 能力依赖底层 TLS 库实现
- 基于 openHiTLS 适配 nginx 1.29.6，实现单端口下国密与国际协议的自适应协商

### 1.2 研究内容

1. 分析 nginx 1.29.6 的 SSL/TLS 接入模型与握手流程
2. 基于 openHiTLS 设计适配层：TLS 上下文、连接握手、数据收发、证书加载接口对接
3. 研究 TLCP 与国际 TLS 在 ClientHello 阶段的识别与分流机制，设计单端口协议自适应策略
4. 基于握手特征（版本、扩展、CipherSuite）实现协议判别与动态切换
5. 完成 TLS 1.2/1.3 与国密协议的统一接入，支持 SNI、ALPN、双向认证、会话复用
6. 处理异常回退与安全策略控制，优化性能与资源占用

### 1.3 交付物与验收指标

| 类型 | 内容 |
|------|------|
| 方案设计 | nginx 1.29.6 与 openHiTLS 适配方案（SSL 核心源文件改造、TLCP 双证书集成、单端口自适应架构） |
| 代码实现 | nginx 1.29.6 完整适配代码，支持 TLCP 双证书体系与 TLS 1.2/1.3 单端口 443 工作；适配 nginx 1.29.6 新特性（Provider 接口、HTTP/2 对上游、新版 SNI 回调） |
| 测试报告 | 功能测试、性能测试（国密/国际单端口自适应互通性、与基于 OpenSSL 的国密 nginx 性能对比）、nginx 社区测试用例通过报告 |
| 验收指标 | ① 单端口 443 并发连接功能验证全部通过（TLCP 单/双向认证、TLS 1.3 初始化、Session 复用、HTTP/2 ALPN 协商等）② 性能评估与对比分析，确保正确性、兼容性、可部署性 |

---

## 2. 核心概念澄清

### 2.1 nginx 与 openHiTLS 的职责边界

```
客户端(国密/TLS) ── TCP ──> nginx（调度中心 + HTTP 管家）── 调用 ──> openHiTLS（加密专家）
                              │ 连接管理 / 事件循环 / 超时 / 路由 / HTTP处理 / 日志
                              │ 调 HITLS_Accept/Read/Write，等待-重试循环
                              └──────────────────────────────────────┘
                                 协议实现 / 证书校验 / 加解密 / 会话管理
```

- **nginx**：不实现任何 TLS 协议。负责 TCP 连接调度、进程模型、事件驱动、握手过程的"等待-重试"循环、超时控制、HTTP 请求处理、SNI 虚拟主机选择、ALPN 协商结果应用
- **openHiTLS**：实现 TLS 1.2/1.3 与 TLCP 协议的全部协议逻辑、证书加载与校验、密钥交换、数据加解密、会话管理

### 2.2 关键事实（已在本仓库核实）

| 事实 | 证据 |
|------|------|
| openHiTLS 原生支持 TLCP | `include/tls/hitls_config.h:39`：`HITLS_VERSION_TLCP_DTLCP11 0x0101` |
| TLCP 1.1 版本号为 0x0101，与国际 TLS（0x0301~0x0304）可区分 | 同上 |
| 单个 HITLS_Ctx 可同时启用 TLS 与 TLCP 版本族 | `include/tls/hitls_type.h`：`STREAM_VERSION_BITS = TLS13_VERSION_BIT \| TLCP11_VERSION_BIT` |
| openHiTLS API 非 OpenSSL 兼容 | 公开接口为 `HITLS_New/Accept/Read/Write` + `BSL_UIO`（非 BIO）+ 自有错误码 |
| 仓库内无 OpenSSL 兼容层 | 全局搜索 `SSL_CTX/ngx_*` 无结果 |
| nginx 官方（配标准 OpenSSL）不支持 TLCP | OpenSSL 主线无国密协议，需 Tongsuo/GmSSL 等分支；openHiTLS 的 TLCP 实现保证国密互通性 |

### 2.3 重要推论

- **协议层互通**：openHiTLS 是标准 TLS/TLCP 协议栈，与任何标准实现可正常握手通信（此能力现成）
- **集成层不兼容**：nginx 源码写死调用 OpenSSL API，openHiTLS 接口形状不同，无法直接链接
- **适配工作本质**：做"转接头"——改 nginx 侧插槽形状（改造 nginx SSL 层），openHiTLS 侧原则上零改动

---

## 3. nginx 1.29.6 SSL/TLS 接入模型分析

### 3.1 nginx 代码获取

nginx 不在 openhitls 仓库内，需单独获取：

```bash
# 官网源码包
wget https://nginx.org/download/nginx-1.29.6.tar.gz

# 或 GitHub 镜像
git clone -b release-1.29.6 https://github.com/nginx/nginx.git
```

### 3.2 改造文件清单

| nginx 文件 | 作用 | 改造内容 |
|-----------|------|---------|
| `src/event/ngx_event_openssl.c/.h` | **核心改造对象**：全部 SSL/TLS 调用集中于此（约 5000 行） | `ngx_ssl_create`（ctx 创建）、`ngx_ssl_handshake`（握手状态机）、`ngx_ssl_read/write`、`ngx_ssl_shutdown`、`ngx_ssl_certificate`（证书加载）、`ngx_ssl_client_certificate/trusted_certificate`、`ngx_ssl_get_session/set_session`、`ngx_ssl_alpn_advertise`、SNI 回调、`ngx_ssl_verify_callback` |
| `src/http/ngx_http_ssl_module.c/.h` | SSL 配置指令、变量、回调注册 | `ssl_certificate` 等指令、`$ssl_protocol/$ssl_cipher` 变量、SNI/ALPN 回调注册、按 server 切换证书 |
| `src/http/ngx_http_upstream_ssl_module.c` + `ngx_http_upstream.c` | HTTPS 对上游（客户端侧握手） | 上游连接握手流程适配（如需） |
| `src/event/ngx_event_openssl_stapling.c` | OCSP stapling | 可选，不需要可裁剪 |
| `auto/lib/openssl/*`、`auto/options` | 构建系统 | `--with-openssl` 改为链接 openHiTLS 库 |

### 3.3 OpenSSL ↔ openHiTLS 接口映射（适配层核心）

```
OpenSSL 语义                          →    openHiTLS
SSL_CTX_new / SSL_new                 →    HITLS_New(HITLS_Config) + HITLS_SetUio
SSL_do_handshake（服务器端）            →    HITLS_Accept
SSL_do_handshake（客户端端）            →    HITLS_Connect
SSL_read / SSL_write                  →    HITLS_Read / HITLS_Write
SSL_get_error 的 WANT_READ/WANT_WRITE →    HITLS_REC_NORMAL_RECV_BUF_EMPTY / HITLS_REC_NORMAL_IO_BUSY
SSL_CTX_use_certificate_chain_file    →    HITLS_CTX 证书加载（PKI + codecs 解析 PEM/DER）
SSL_CTX_set_session_cache             →    HITLS_Session 序列化/反序列化
SSL_CTX_set_sni / set_alpn_select_cb  →    hitls_sni.h / hitls_alpn.h 回调
SSL_shutdown                          →    HITLS_Close
```

**适配层建议独立成文件**（如 `src/event/ngx_event_hitls.c`），不做散落式替换，便于维护与对比评审。

---

## 4. 改造范围边界

### 4.1 工作分配

| 侧 | 工作量 | 内容 |
|----|--------|------|
| nginx 侧 | 约 85% | `ngx_event_openssl.c/.h`、`ngx_http_ssl_module.c/.h` 改写；TLCP 双证书配置指令；单端口自适应；构建系统 |
| openHiTLS 侧 | 约 15% | 以只读调研 + 验证为主，原则上不改代码 |

### 4.2 openHiTLS 潜在缺口（需实测确认）

| 潜在缺口 | nginx 需要什么 | 应对策略 |
|---------|--------------|---------|
| Session 序列化 | `SSL_SESSION` 二进制跨 worker 共享（`ssl_session_cache shared:`） | 退化为进程内缓存，或向 openHiTLS 提增强 |
| SNI 回调中切换配置 | 回调里换证书/配置后继续握手 | 适配层获取 ClientHello 数据重建配置，或 openHiTLS 增强 |
| 非阻塞错误码粒度 | 精确区分"想读/想写/致命错误" | openHiTLS 已有 `RECV_BUF_EMPTY/IO_BUSY`，大概率够用 |

### 4.3 重要原则

若必须修改 openHiTLS，**不得只改私有 fork**：应以 PR 方式提交到官方仓 `openhitls/openhitls` 合入，本地 fork 保持同步跟踪。
- nginx 侧改造：私有、随课题交付
- openHiTLS 侧改动：社区化、走贡献流程（注册 GitCode、签署 CLA、提 Issue → PR → 流水线检查 → 评审）

---

## 5. 单端口自适应架构设计

### 5.1 判别依据（国标 GB/T 38636 / GM/T 0024 特征）

| 特征 | TLCP 1.1 | 国际 TLS |
|------|----------|---------|
| ClientHello 版本号 | `0x0101` | `0x0301` ~ `0x0304` |
| CipherSuite | `0xE011/0xE013/0xE017/0xE019`（SM4+SM3 系） | `0x13xx/0xC0xx/0x00xx` |

### 5.2 推荐架构（两层兜底）

**主路径：单 ctx 融合（库内自适应）**
- 一个 HITLS_CTX 同时开启 `TLS12 | TLS13 | TLCP11` 版本位
- 同时挂载 SM2 签名证书 + SM2 加密证书 + RSA/ECDSA 证书
- openHiTLS 服务端按 ClientHello 的版本号 + CipherSuite 自动选择协议族与证书
- nginx 对协议判别完全无感知，适配层最薄

**兜底路径：nginx 层嗅探分流**
- 在握手入口读取 5 字节 TLS record 头，`version == 0x0101` 走 TLCP ctx，否则走 TLS ctx
- 两个独立 HITLS_CTX 配置不同证书/套件，支持"动态切换"
- 用于处理异常场景（如库内融合协商异常）与课题要求的"基于握手特征的协议判别"

### 5.3 异常回退与安全策略

- TLCP 协商失败不得静默降级到弱配置（防降级攻击）
- 建立协议判别白名单：仅当 ClientHello 特征明确匹配时才按 TLCP 处理
- 记录并监控异常握手（协议混用、版本回退探测）

---

## 6. TLCP 双证书集成方案

- TLCP 要求**双证书**：SM2 签名证书（sign）+ SM2 加密证书（encrypt）
- nginx 现有 `ssl_certificate` 为单证书指令，需新增指令（如 `ssl_sign_certificate` / `ssl_enc_certificate`），或设计 TLCP 模式下合并加载
- 同一 ctx 内 SM2 双证书与国际 RSA/ECDSA 证书共存，按协商结果选择
- 证书格式：PEM/DER 解析依赖 openHiTLS PKI + codecs 组件

---

## 7. nginx 1.29.6 新特性适配

| 新特性 | nginx 侧 | openHiTLS 侧 |
|--------|---------|-------------|
| Provider 接口（`ssl_provider` 指令，1.27+） | 将指令映射为 openHiTLS 的 provider 加载（`crypto/provider` 可加载算法模块），或做等价能力映射 | 确认 provider 加载 API 语义 |
| HTTP/2 对上游 | 保证 ALPN 协商正确（server 侧 `h2`，upstream 侧 `h2c`） | 原生支持 ALPN |
| 新版 SNI 回调（1.25+） | 适配"ClientHello 解析完成后回调、回调中可切换配置"的语义 | 确认 SNI 回调切换 ctx/配置的支持度（`hitls_sni.h`） |
| 会话复用 | `ssl_session_cache shared:SSL:10m` 跨 worker 共享 | 确认 session 导出/导入（`hitls_session.h`） |
| 双向认证 | `ssl_verify_client` → CA 列表 + 校验回调 | 确认 TLS 1.3 CertificateRequest 语义 |

---

## 8. 关键难点

1. **非阻塞事件模型映射**：nginx 事件驱动，`ngx_ssl_handshake()` 依赖 WANT_READ/WANT_WRITE 决定挂哪个事件。错误码映射不精确会导致忙等或死锁，这是适配成败的关键
2. **证书双轨**：单证书指令 vs TLCP 双证书体系的配置与加载设计
3. **SNI 回调配置切换**：多虚拟主机场景下的证书/配置动态切换语义
4. **Session 跨进程共享**：共享内存缓存要求 session 可序列化

---

## 9. 分阶段实施路径

| 阶段 | 内容 | 验证方式 |
|------|------|---------|
| 0 | openHiTLS 构建与自测 | 本仓库 `testcode/script` 三件套（build_hitls.sh / build_sdv.sh / execute_sdv.sh） |
| 1 | 最小闭环：单证书 TLS 1.2/1.3 跑通 nginx HTTPS | 标准 OpenSSL s_client / 浏览器访问 |
| 2 | TLCP 双证书握手 | Tongsuo / GmSSL 的 s_client（标准 s_client 不支持 TLCP） |
| 3 | 单端口自适应 | 国密客户端 + 国际客户端并发访问 443 |
| 4 | 补齐特性：SNI / ALPN / 会话复用 / 双向认证 / Provider / HTTP2 上游 | 分项用例 |
| 5 | 性能对比 | 与 OpenSSL + Tongsuo 的国密 nginx 对比（openssl s_time / wrk / nginx 日志） |
| 6 | 社区用例 | nginx 官方测试套件 nginx-tests（Test::Nginx Perl 框架），跑 HTTPS/SNI/ALPN/session 相关组 |

---

## 10. 测试与验证方案要点

- **国密互通**：测试客户端必须支持 TLCP（Tongsuo/GmSSL s_client、国密浏览器）；标准 OpenSSL s_client 只能测国际 TLS 链路
- **验收指标 1**：单端口 443 并发连接功能验证——TLCP 单/双向认证、TLS 1.3 初始化、Session 复用、HTTP/2 ALPN 协商
- **验收指标 2**：性能评估与对比（与基于 OpenSSL 的国密 nginx 实现对比）
- **回归**：nginx 社区测试用例通过报告

---

## 11. 参考资料

- openHiTLS 仓库：`docs/zh/5_开发指南/2_安全通信应用开发指南.md`（HITLS API 用法）
- openHiTLS 仓库：`docs/zh/5_开发指南/5_TLS证书接口参考.md`（证书加载）
- openHiTLS 仓库：`include/tls/hitls.h`（TLS 公开 API，约 1900 行）
- openHiTLS 仓库：`include/tls/hitls_config.h` / `hitls_type.h`（版本与特性位定义）
- nginx 1.29.6 源码：`src/event/ngx_event_openssl.c`（OpenSSL 调用面全清单）
- 国标：GB/T 38636-2020（TLCP 1.1）、GM/T 0024-2023
- nginx-tests：http://hg.nginx.org/nginx-tests
