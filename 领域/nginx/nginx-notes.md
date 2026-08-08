# Nginx 反代部署笔记（联机版）

本文记录《政变》联机版在云服务器上使用 Nginx 反向代理的完整配置、原理与排查方法。对应服务器：`47.93.119.89`（Ubuntu 24.04 + 宝塔面板，nginx 1.20.2，配置目录 `/www/server/panel/vhost/nginx/`）。

## 一、为什么需要 Nginx

| 作用 | 说明 |
|---|---|
| **端口收敛** | 用户访问 80（无端口号）或 8787（旧链接兼容），不必记 `:8788` |
| **TLS 终结** | 将来证书签在 Nginx 层；后端 node 不配证书、不暴露公网 |
| **真实 IP 透传** | `X-Forwarded-For` 传给后端，配合 `COUP_TRUST_PROXY=1` 让房间号限速按真实客户端 IP 计数 |
| **静态资源缓存** | `/assets/`（vite 哈希文件名）在 Nginx 层永久缓存，不再打到后端 |
| **后端隐藏** | node 只监听 `127.0.0.1:8788`，公网（含安全组/防火墙）不放行 8788 |

## 二、当前架构

```
浏览器 → http://47.93.119.89        (80 端口，nginx，主入口)
浏览器 → http://47.93.119.89:8787   (nginx，旧 /join?code= 链接兼容)
                     │
                  nginx
                     │  proxy_pass → 127.0.0.1:8788
                     │  /assets/ 永久缓存
                     │  透传 Host / X-Real-IP / X-Forwarded-For / X-Forwarded-Proto
                     ▼
             127.0.0.1:8788 ──▶ node（联机版 apps/server，仅内网可达）
```

## 三、配置文件详解

### 3.1 主入口：`47.93.119.89.conf`（80 端口）

```nginx
server {
    listen 80;
    server_name 47.93.119.89;
    access_log /www/wwwlogs/47.93.119.89.log;

    # 静态资源（vite 带哈希文件名，可永久缓存）
    location /assets/ {
        proxy_pass http://127.0.0.1:8788;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_hide_header Cache-Control;                     # 去掉上游的 max-age=0
        add_header Cache-Control "public, max-age=2592000, immutable";
    }

    location / {
        proxy_pass http://127.0.0.1:8788;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 10s;
        proxy_read_timeout 60s;
    }
}
```

### 3.2 旧链接兼容：`coupgame-8787.conf`（8787 端口）

```nginx
server {
    listen 8787;
    server_name _;
    access_log /www/wwwlogs/coupgame-8787.log;

    location / {
        proxy_pass http://127.0.0.1:8788;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 3.3 域名预留：`coupgame.xyz.conf`（80 端口，暂被备案拦截）

结构与 3.1 相同，仅 `server_name coupgame.xyz`。备案通过后在此基础上加 443 + 证书即可。

### 3.4 关键指令逐条说明

| 指令 | 作用 |
|---|---|
| `listen 80;` / `listen 8787;` | 监听的端口。node 不再监听公网端口 |
| `server_name 47.93.119.89;` / `_` | 按 Host 头匹配虚拟主机；`_` 兜底任意 Host |
| `proxy_pass http://127.0.0.1:8788;` | 核心：把请求转发给内网后端（反向代理） |
| `proxy_http_version 1.1;` | 上游用 HTTP/1.1（keep-alive，避免 1.0 每请求重连） |
| `proxy_set_header Host $host;` | 保留用户原始 Host（后端 Origin 校验依赖它） |
| `proxy_set_header X-Real-IP $remote_addr;` | 直连 Nginx 的客户端 IP |
| `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;` | 真实客户端 IP 链（后端 `request.ip` 取它，须开 `COUP_TRUST_PROXY=1`） |
| `proxy_set_header X-Forwarded-Proto $scheme;` | 用户实际协议（http/https），供后端判断 |
| `proxy_hide_header Cache-Control;` | 丢弃上游（fastify-static）的 `public, max-age=0`，避免与下方缓存头叠加冲突 |
| `add_header Cache-Control "public, max-age=2592000, immutable";` | 静态资源 30 天永久缓存（文件名带哈希，内容不可变） |
| `proxy_connect_timeout 10s;` / `proxy_read_timeout 60s;` | 与后端的连接/读超时 |
| `access_log /www/wwwlogs/47.93.119.89.log;` | 访问日志（能看到真实客户端 IP） |

## 四、后端配套（缺一不可）

| 环境变量 | 值 | 原因 |
|---|---|---|
| `COUP_TRUST_PROXY=1` | 必须开 | 否则 fastify 的 `request.ip` 恒为 `127.0.0.1`，房间号限速变成全体用户共享一个桶 |
| `COUP_ALLOWED_ORIGINS` | 含 `http://47.93.119.89`、`http://47.93.119.89:8787`、域名项 | 浏览器 Origin 白名单，缺了全部 API 403 `origin_not_allowed` |
| `COUP_PUBLIC_HOST` | `coupgame.xyz` | 展示用主机名 |
| `COUP_PORT=8788` | 内网口 | 与 nginx 转发目标一致 |

## 五、验证方法

```bash
# 服务端本地
curl -s http://127.0.0.1:8788/api/hosting        # node 直连正常
curl -s http://127.0.0.1:8788/ -o /dev/null -w "%{http_code}\n"

# 外网（从自己电脑）
curl -o /dev/null -w "%{http_code}\n" http://47.93.119.89/        # 期望 200（走 nginx）
curl -o /dev/null -w "%{http_code}\n" http://47.93.119.89:8787/   # 期望 200（旧链接）
curl -o /dev/null -w "%{http_code}\n" http://47.93.119.89:8788/   # 期望超时/000（后端已隐藏）

# 静态缓存头
curl -sI http://47.93.119.89/assets/<哈希文件名>.js | findstr Cache-Control
# 期望: cache-control: public, max-age=2592000, immutable（且只有一份）

# 限速（真实 IP）
# 连续 11 次 GET /api/rooms/0001 → 第 11 次 429
```

## 六、踩坑记录

1. **nginx `400 Bad Request`**：请求没带 Host 头（常见于从 Windows PowerShell 经 ssh 传 `curl -H "Host: ..."` 时引号被吞）。真实浏览器访问不会发生；排查时用外部真实 URL 测，或确认 Host 头确实发出。
2. **`add_header` 报错**：`invalid parameter "immutable"` / `invalid number of arguments` —— 值是含空格/逗号的字符串，引号在 ssh 传参链中被破坏。**对策：配置文件本地写好再 scp 上传**，不要在 ssh 命令行里内联带引号的配置。
3. **缓存头叠加**：`expires` 或上游默认 `Cache-Control: max-age=0` 会与 `add_header` 叠加成两份（浏览器取最严，缓存失效）。用 `proxy_hide_header Cache-Control` 去掉上游的，只保留自己的一份。
4. **`nginx -t` 失败会拒绝重载**：改配置后先 `nginx -t`，通过再 `nginx -s reload`。
5. **域名被 ICP 拦截**：大陆服务器上未备案域名访问 80/443 直接被阿里云拦截（返回备案提示页），与 nginx 无关；备案前域名不可用。

## 七、运维命令

```bash
nginx -t                          # 校验配置
nginx -s reload                   # 平滑重载（不断连接）
tail -f /www/wwwlogs/47.93.119.89.log        # 访问日志（真实 IP）
tail -f /www/server/nginx/logs/error.log     # 错误日志
systemctl status coup             # 后端服务状态
systemctl restart coup            # 重启后端
```

## 八、将来切换域名（备案通过后）

1. 安全组放行 80/443（443 已放行）。
2. 申请 Let's Encrypt 证书（`certbot --nginx` 或宝塔一键 SSL）。
3. `47.93.119.89.conf` 与 `coupgame.xyz.conf` 合并为一个 server 块：`server_name coupgame.xyz;` + `listen 443 ssl;` + 证书路径 + 80 跳 443。
4. 后端环境变量：`COUP_ALLOWED_ORIGINS` 保持含 `https://coupgame.xyz`（已配置），`COUP_PUBLIC_HOST=coupgame.xyz`（已配置）。
5. 程序代码零改动。
