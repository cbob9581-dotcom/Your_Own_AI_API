# 需求

大多数时候我们并不需要太强的通用模型，只需要它在某一方面能做事，利用其比传统算法更强的泛化性、鲁棒性以及对 NLP（自然语言处理）的强大能力。

众所周知调用 API 要花钱，而且往往会浪费——很多通用能力用不到；而运营商提供的"免费"模型通常也会限制 tokens 数。如果机器较强，固然可以自己本地部署，但在性能弱的机器上就无法运行大模型。

所以我们可以把大模型部署在一台性能不错的机器上，然后**以类似 API 的形式**提供给性能较弱的机器。只要弱机器与强机器之间能够进行数据传递，我们就具备了在弱设备上使用大模型的能力。

就我个人而言，我使用的是**公网通信**方案，这样只要设备能联网，在全球任何有 WiFi 的地方都能使用你部署的大模型。当然你也可以使用局域网——如果你只需要在局域网环境下使用，这会简单很多，不涉及内网穿透。本项目讲的是**公网通信版本**的实现与使用。

不过本项目并不只是做这个。这是核心，但我是围绕它搭建一个更大的系统，涉及 MCP 服务器、智能 IoT 设备，以及配套 Web 与桌面应用等的实现。

---

# 所需的东西（公网版）

- **云服务器**：有一个公网 IP。其实没有也可以，但把你的计算机直接内网穿透暴露在公网十分不安全。
- **域名**：其实没有也可以，我后续会在问题文档中说明。
- **一台性能还行的电脑**：决定你最后能运行什么模型，一般主要看显卡显存。

---

# 可行性

- **域名 + TLS 证书 → HTTPS**：加密传输数据，客户端信任且更安全（两方面：数据传输加密；你的电脑不用直接暴露在公网）。
- **frp**：为节约 IPv4 资源而采用的 NAT 路由等技术破坏了端到端通信——别人在公网上找不到你的计算机，自然无法调用你的服务。frp 主要由两个组件组成：**客户端 (frpc)** 和 **服务端 (frps)**。服务端部署在具有公网 IP 的机器上，客户端部署在需要穿透的内网服务所在机器上。用户访问服务端 frps，frp 根据请求的端口或其他信息将请求路由到相应的内网机器，从而实现通信。参考：[frp 概念说明](https://gofrp.org/zh-cn/docs/concepts/)。我采取的是 frp 的 **TCP 型代理**（后续会改为 **STCP**，会更安全）。
- **Ollama**：可以轻松地部署本地大模型。仓库：[Ollama](https://github.com/ollama/ollama)。
- **Nginx**（"Engine X"）：高性能 HTTP Web 服务器，同时可作为反向代理、负载均衡器和缓存服务器；还支持 TCP/UDP 代理和邮件代理。这里用它来**监听端口、终止 TLS、校验 API Key、反向代理到 frp 暴露的端口**。

---

# 整体架构

```text
客户端 / 弱设备（跑不了大模型的设备）
      │
      │ HTTPS
      ▼
api.example.com
      │
      ▼
   云服务器
 ┌──────────────────────────┐
 │  Nginx                   │
 │   - TLS 终止             │
 │   - API Key 校验         │
 │   - 反代到 frp 暴露端口  │
 │                          │
 │  FRP Server (frps)       │
 │   - 接收笔记本连接       │
 │   - 暴露内网端口         │
 └──────────────────────────┘
      │
      │ FRP 隧道（笔记本主动连入）
      ▼
    笔记本
 ┌──────────────────────────┐
 │  FRP Client (frpc)       │
 │                          │
 │  业务层 (FastAPI 自实现) │
 │   - /v1/chat             │
 │   - /v1/parse-command    │
 │   - /v1/health           │
 │                          │
 │  Ollama                  │
 │   - 本地模型推理         │
 └──────────────────────────┘
```

## 实现步骤

### 第一步：域名与证书准备

#### 1. DNS 配置

在你的域名 DNS 面板新增一条 A 记录：

| 类型 | 主机记录 | 记录值          |
|------|----------|-----------------|
| A    | api      | 你的云服务器 IP |

也就是：`api.example.com → 云服务器 IP`。等待 DNS 生效，通常几分钟到几小时。

#### 2. 云服务器申请 SSL 证书

你现有的网站证书大概率只覆盖 `example.com` 或 `www.example.com`。子域名 `api.example.com` 需要单独申请，除非你原来用的是通配符证书 `*.example.com`，就可以直接复用。

检查现有证书是否覆盖子域名：

```bash
openssl x509 -in /path/to/cert.pem -text -noout | grep -A1 "Subject Alternative"
```

- 如果已包含 `*.example.com`，直接用。
- 否则申请一个新证书。

用 Certbot 申请（推荐）：

```bash
sudo apt update
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d api.example.com
```

Certbot 会自动写入 Nginx 配置并设置定时续期。

### 第二步：FRP 部分

#### 云服务器部分

##### 一、安装并配置 frps

从 Github 仓库 https://github.com/fatedier/frp/releases 下载与云服务器匹配的版本（以 frp 0.58.0、Ubuntu 20.04 为例，按实际替换）。  
解压进入目录，编辑 `frps.toml`：

```toml
# frps.toml

bindPort = 7000

# 鉴权 token（笔记本那边要填一样的，用于确保只有特定笔记本能连入）
auth.method = "token"
auth.token = "your-strong-secret-token"

# 仪表盘（可选，方便查看连接状态）
webServer.addr = "127.0.0.1"
webServer.port = 7500
webServer.user = "admin"
webServer.password = "admin-password"
```

启动：

```bash
./frps -c ./frps.toml
```

> 生产环境建议写成 systemd 服务。

##### 二、配置 Nginx

```
/etc/nginx/
├── nginx.conf              # 主配置
├── sites-available/        # 所有站点配置（我们在这里写）
│   └── api.example.com     # 要新增的文件
└── sites-enabled/          # 实际启用的配置（软链接）
```

这个站点配置的本质：当有人访问 `api.example.com` 时，服务器应该怎么处理请求。

① 在 `nginx.conf` 的 `http { ... }` 块里加入限流 zone（否则 `limit_req` 会报错）：

```nginx
http {
    # ... 其他已有配置
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=5r/s;
}
```

② 新建 `/etc/nginx/sites-available/api.example.com`：

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name api.example.com;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name api.example.com;

    ssl_certificate     /etc/letsencrypt/live/api.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;

    add_header X-Content-Type-Options nosniff;
    add_header X-Frame-Options DENY;

    # API Key 校验（仅允许白名单客户端）
    if ($http_authorization != "Bearer your-api-key") {
        default_type application/json;
        return 401 '{"error":{"code":"UNAUTHORIZED","message":"Invalid API key"}}';
    }

    location /v1/ {
        limit_req zone=api_limit burst=10 nodelay;

        proxy_pass http://127.0.0.1:9000;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # 流式输出相关
        proxy_http_version 1.1;
        proxy_buffering          off;
        proxy_request_buffering  off;
        chunked_transfer_encoding on;
        add_header X-Accel-Buffering no;

        proxy_connect_timeout 10s;
        proxy_read_timeout    3600s;
        proxy_send_timeout    3600s;

        proxy_intercept_errors on;
        error_page 502 503 504 = @backend_offline;
    }

    location @backend_offline {
        default_type application/json;
        return 503 '{"error":{"code":"BACKEND_OFFLINE","message":"Inference backend is currently offline"}}';
    }

    access_log /var/log/nginx/api.example.com.access.log;
    error_log  /var/log/nginx/api.example.com.error.log;
}
```

③ 启用配置：

```bash
sudo ln -s /etc/nginx/sites-available/api.example.com /etc/nginx/sites-enabled/
sudo nginx -t            # 检查格式
sudo systemctl reload nginx
```

> 只有 `sites-enabled/` 里的配置才会被加载，所以必须建立软链。

#### 模型运行机器部分

##### 一、下载安装 Ollama

从官网下载：[Ollama](https://ollama.com/)。

```bash
ollama pull qwen2.5:7b   # 拉取你要使用的模型（按需替换）
ollama serve
```

`ollama serve` 会启动 Ollama 服务，监听本地端口（默认 `11434`）：`http://127.0.0.1:11434`。此时它将本地大模型暴露为一个可通过 HTTP 调用的接口服务（仅本机可访问）。

##### 二、安装并配置 frpc

从 https://github.com/fatedier/frp/releases 下载对应平台版本（以 frp 0.58.0、Windows 10 为例）。  
解压后编辑 `frpc.toml`（注意是 frpc，c 代表 client）：

```toml
# frpc.toml

serverAddr = "你的云服务器IP"
serverPort = 7000

auth.method = "token"
auth.token = "your-strong-secret-token"   # 必须与 frps 保持一致

[[proxies]]
name       = "ai-api"
type       = "tcp"
localIP    = "127.0.0.1"
localPort  = 8080      # 本地"真正提供服务"的端口（FastAPI 监听此端口）
remotePort = 9000      # 云服务器上要暴露的端口
```

启动：

```bash
frpc.exe -c frpc.toml
```

##### 三、业务层实现

安装依赖：

```bash
pip install fastapi uvicorn httpx
```

创建 `main.py`：

```python
import json
import time

import httpx
from fastapi import FastAPI, HTTPException, Request
from fastapi.responses import JSONResponse
from pydantic import BaseModel

app = FastAPI(title="My AI API")

OLLAMA_URL = "http://127.0.0.1:11434"
MODEL_NAME = "qwen2.5:7b"

# ── 数据模型 ────────────────────────────────────────────────

class ChatRequest(BaseModel):
    message: str
    system: str = "你是一个有帮助的助手。"

class ParseCommandRequest(BaseModel):
    text: str

# ── 工具函数 ────────────────────────────────────────────────

async def call_ollama(prompt: str, system: str = "") -> str:
    messages = []
    if system:
        messages.append({"role": "system", "content": system})
    messages.append({"role": "user", "content": prompt})

    async with httpx.AsyncClient(timeout=90.0) as client:
        response = await client.post(
            f"{OLLAMA_URL}/api/chat",
            json={
                "model": MODEL_NAME,
                "messages": messages,
                "stream": False,
            },
        )
        response.raise_for_status()
        return response.json()["message"]["content"]

# ── 接口 ────────────────────────────────────────────────────

@app.get("/v1/health")
async def health():
    """健康检查"""
    try:
        async with httpx.AsyncClient(timeout=5.0) as client:
            r = await client.get(f"{OLLAMA_URL}/api/tags")
            r.raise_for_status()
        return {
            "status": "ok",
            "backend": "online",
            "model": MODEL_NAME,
            "timestamp": int(time.time()),
        }
    except Exception:
        return JSONResponse(
            status_code=503,
            content={"status": "degraded", "backend": "offline"},
        )

@app.post("/v1/chat")
async def chat(req: ChatRequest):
    """通用对话"""
    try:
        result = await call_ollama(req.message, req.system)
        return {"reply": result, "model": MODEL_NAME}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/v1/parse-command")
async def parse_command(req: ParseCommandRequest):
    """把自然语言解析成结构化指令"""
    system = """你是一个智能家居指令解析器。
把用户输入解析为 JSON 格式，结构如下：
{"device": "设备名", "action": "动作", "value": 参数或null}
只返回 JSON，不要有多余文字。"""

    result = ""
    try:
        result = await call_ollama(req.text, system)
        parsed = json.loads(result)
        return {"command": parsed, "raw": result}
    except json.JSONDecodeError:
        return {"command": None, "raw": result, "warning": "模型未返回合法 JSON"}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# ── 全局错误处理 ────────────────────────────────────────────

@app.exception_handler(HTTPException)
async def http_exception_handler(request: Request, exc: HTTPException):
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": {"code": "HTTP_ERROR", "message": exc.detail}},
    )

@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    return JSONResponse(
        status_code=500,
        content={"error": {"code": "INTERNAL_ERROR", "message": str(exc)}},
    )
```

启动：

```bash
uvicorn main:app --host 127.0.0.1 --port 8080
```

### 端口梳理

| 位置     | 端口  | 用途                     |
|----------|-------|--------------------------|
| 笔记本   | 11434 | Ollama 模型服务          |
| 笔记本   | 8080  | FastAPI 业务层           |
| 云服务器 | 7000  | FRP 通信端口             |
| 云服务器 | 9000  | FRP 映射出来的笔记本接口 |
| 云服务器 | 443   | Nginx HTTPS 入口         |
| 云服务器 | 80    | Nginx HTTP（跳转 HTTPS） |

### 完整请求链路

```text
客户端
  POST https://api.example.com/v1/chat
  Header: Authorization: Bearer your-api-key
  Body:   {"message": "你好"}
    │
    │ DNS 解析到云服务器 IP
    ▼
云服务器 Nginx (443)
  - 终止 TLS
  - 校验 API Key
  - 转发到 127.0.0.1:9000
    │
    ▼
云服务器 FRP Server (9000)
  - 通过隧道转发给笔记本
    │
    ▼
笔记本 FRP Client
  - 接收请求，转发到本地 8080
    │
    ▼
笔记本 FastAPI (8080)
  - 处理请求
  - 调用 Ollama (11434)
    │
    ▼
笔记本 Ollama (11434)
  - 模型推理
  - 返回结果

  ↑ 原路返回
```

