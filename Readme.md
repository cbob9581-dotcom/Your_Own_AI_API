# 公网版：把本地大模型封装成可远程调用的类 API 服务

大多数时候，我们并不需要太强的通用模型，只需要它在某一方面能做事，利用其比传统算法更强的泛化性、鲁棒性以及对 NLP（自然语言处理）的强大能力。

众所周知，调用 API 要花钱，而且往往会浪费——很多通用能力其实根本用不到；而运营商或平台提供的“免费”模型通常也会限制 token 数。如果机器性能足够强，当然可以自己本地部署，但在性能较弱的机器上，大模型往往跑不起来。

所以，一个很自然的思路就是：

> 把大模型部署在一台性能不错的机器上，然后**以类似 API 的形式**提供给性能较弱的机器使用。

只要弱机器和强机器之间能够传递数据，我们就具备了在弱设备上使用大模型的能力。

就我个人而言，我使用的是**公网通信**方案。这样只要设备能联网，在全球任何有 Wi‑Fi 的地方都能使用你部署的大模型。当然你也可以只使用局域网——如果你的需求只发生在局域网环境下，那么那会简单很多，也不涉及内网穿透。本项目这里讲的是**公网通信版本**的实现与使用。

不过，本项目并不只是做“远程调用本地大模型”这件事。这是整个系统的核心能力之一，但我是围绕它搭建一个更大的系统，后续会涉及 MCP 服务器、智能 IoT 设备，以及配套 Web 与桌面应用等内容。

---

# 一、所需的东西（公网版）

- **一台云服务器**：最好有公网 IP，作为统一入口。
- **一个域名**：强烈建议有，方便做 HTTPS 证书与稳定访问。
- **一台性能还行的电脑**：决定你最终能跑什么模型，主要看 CPU / GPU / 显存。
- **可联网环境**：内网模型机需要能主动连接到云服务器。

> 其实没有域名也不是完全不行，但会导致 HTTPS、证书签发、客户端信任等环节麻烦很多。公网方案下，建议尽量准备一个域名。

---

# 二、整体可行性与各组件作用

## 1. 域名 + TLS 证书 + HTTPS

HTTPS 的核心作用有两个：

- **加密传输数据**
- **让客户端验证服务端身份**

它能让客户端与服务器之间的传输不以明文形式暴露，也能降低中间人攻击风险。

但这里要说准确一点：

> “你的电脑不用直接暴露在公网”并不是 HTTPS 本身带来的，而是因为我们使用了“**云服务器统一入口 + frp 反向连接 + Nginx 反代**”这一整套架构。

HTTPS 负责加密和可信访问；真正避免直接暴露内网设备的是“云服务器作跳板 + 内网机器主动连出”的通信方式。

---

## 2. frp

现在很多网络环境下，由于 NAT、运营商网络策略、IPv4 资源紧张等原因，公网无法直接访问你家里或办公室里的电脑。  
也就是说，即使你的电脑在线，外部也找不到它。

frp 就是用来解决这个问题的。

frp 主要由两个组件组成：

- **frps**：服务端，部署在云服务器上
- **frpc**：客户端，部署在你的内网模型机上

模型机上的 frpc 会**主动连接**云服务器上的 frps，建立一条隧道。之后公网流量先到达云服务器，再由 frps 通过这条隧道转发给内网机器。

本项目中采用的是 frp 的 **TCP 代理**方案。  
它实现简单、容易理解，但安全性上不是最优。后续如果长期公网使用，更推荐切换为 **STCP** 或结合更严格的访问控制。

这里必须强调一个容易混淆的点：

- `frp` 的 `auth.token`：是 **frpc 与 frps 之间的连接认证**
- 你的 API `Authorization: Bearer xxx`：是 **业务接口的访问认证**

它们**不是一回事**，不要混为一谈。

---

## 3. Ollama

Ollama 可以很方便地在本地运行大模型，并通过 HTTP 提供调用接口。

在本方案里，Ollama 的位置是：

- **只在模型机本地监听**
- **只由业务层调用**
- **不直接对公网暴露**

也就是说，Ollama 不是公网 API，它只是底层推理引擎。

---

## 4. Nginx

Nginx 在这里承担的是公网入口和网关角色，主要负责：

- 监听公网 `80 / 443`
- 做 HTTP → HTTPS 跳转
- 终止 TLS
- 校验 API Key
- 限流
- 反向代理到 frp 映射出的本机端口
- 在后端不可用时返回统一错误

---

# 三、整体架构

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
 │   - 限流                 │
 │   - 反代到本机 9000      │
 │                          │
 │  FRP Server (frps)       │
 │   - 接收模型机连接       │
 │   - 将请求转发进隧道     │
 └──────────────────────────┘
      │
      │ FRP 隧道（模型机主动连出）
      ▼
    模型运行机器
 ┌──────────────────────────┐
 │  FRP Client (frpc)       │
 │                          │
 │  FastAPI 业务层          │
 │   - /v1/chat             │
 │   - /v1/parse-command    │
 │   - /v1/health           │
 │                          │
 │  Ollama                  │
 │   - 本地模型推理         │
 └──────────────────────────┘
```

---

# 四、所需端口梳理

| 位置     | 端口  | 用途 |
|----------|-------|------|
| 模型机   | 11434 | Ollama 本地服务 |
| 模型机   | 8080  | FastAPI 业务层 |
| 云服务器 | 7000  | frps 监听 |
| 云服务器 | 9000  | frp 映射出来的业务端口 |
| 云服务器 | 443   | Nginx HTTPS 入口 |
| 云服务器 | 80    | Nginx HTTP 跳转 HTTPS |

> **重要安全提醒**：  
> 云服务器上的 `9000` 虽然是 frp 映射出来的端口，但它**不应该对公网开放**。  
> 如果 9000 被公网访问，别人就可能绕过 Nginx，直接访问你的 FastAPI 服务，从而绕过 TLS、API Key 和部分安全策略。

公网防火墙建议只开放：

- `80`
- `443`
- `7000`

不要开放：

- `9000`
- `7500`（frps 仪表盘，如启用）
- 模型机上的任何端口

---

# 五、完整请求链路

```text
客户端
  POST https://api.example.com/v1/chat
  Header: Authorization: Bearer your-api-key
  Body:   {"message": "你好"}
    │
    │ DNS 解析到云服务器
    ▼
云服务器 Nginx (443)
  - 终止 TLS
  - 校验 API Key
  - 限流
  - 转发到 127.0.0.1:9000
    │
    ▼
云服务器 frps / 映射端口 (9000)
  - 通过 FRP 隧道转发请求
    │
    ▼
模型机 frpc
  - 接收来自 frps 的转发
  - 转发到本地 127.0.0.1:8080
    │
    ▼
模型机 FastAPI (8080)
  - 处理业务逻辑
  - 调用本地 Ollama
    │
    ▼
模型机 Ollama (11434)
  - 完成推理
  - 返回结果

  ↑ 原路返回
```

---

# 六、实现步骤

---

## 第一步：域名与证书准备

### 1. DNS 配置

在你的域名 DNS 管理面板新增一条 A 记录：

| 类型 | 主机记录 | 记录值 |
|------|----------|--------|
| A    | api      | 你的云服务器公网 IP |

即：

```text
api.example.com -> 你的云服务器公网 IP
```

等待 DNS 生效，通常几分钟到几小时。

---

### 2. 云服务器上申请 SSL 证书

如果你现有网站证书只覆盖：

- `example.com`
- `www.example.com`

那么它通常**不能直接用于** `api.example.com`，除非你使用的是通配符证书：

- `*.example.com`

可以先检查证书是否覆盖目标子域名：

```bash
openssl x509 -in /path/to/cert.pem -text -noout | grep -A1 "Subject Alternative"
```

如果没有覆盖 `api.example.com`，可使用 Certbot 重新申请：

```bash
sudo apt update
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d api.example.com
```

这会自动：

- 申请证书
- 配置 Nginx
- 启用定时续期

---

## 第二步：FRP 部分

---

## A. 云服务器部分

### 1. 安装并配置 frps

从官方 Release 下载对应版本：

- GitHub: https://github.com/fatedier/frp/releases

以下使用 TOML 配置格式示例。注意，旧教程里有很多 `.ini` 版配置，和新版本可能不同。

新建 `frps.toml`：

```toml
bindPort = 7000

auth.method = "token"
auth.token = "your-strong-secret-token"

webServer.addr = "127.0.0.1"
webServer.port = 7500
webServer.user = "admin"
webServer.password = "change-this-password"
```

说明：

- `bindPort = 7000`：供 frpc 连入
- `auth.token`：frpc 与 frps 的连接认证令牌
- `webServer.addr = "127.0.0.1"`：仪表盘仅本机访问，更安全

启动：

```bash
./frps -c ./frps.toml
```

生产环境建议写成 systemd 服务，后文会给示例。

---

### 2. 配置云服务器防火墙 / 安全组

如果你使用的是云厂商安全组，请仅放行：

- TCP 80
- TCP 443
- TCP 7000

如果你使用 UFW，可以参考：

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 7000/tcp
sudo ufw deny 9000/tcp
sudo ufw deny 7500/tcp
sudo ufw enable
```

> 如果 9000 已被云安全组放行，那么即使你本机不想开放，它仍可能被公网打到。一定要同时检查“云安全组”和“服务器本机防火墙”。

---

### 3. 配置 Nginx

目录结构一般如下：

```text
/etc/nginx/
├── nginx.conf
├── sites-available/
│   └── api.example.com
└── sites-enabled/
```

---

#### 3.1 修改 `nginx.conf`

编辑 `/etc/nginx/nginx.conf`，在 `http {}` 块内加入限流 zone 和 API Key 映射：

```nginx
http {
    # ... 其他已有配置

    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=5r/s;

    map $http_authorization $api_key_ok {
        default 0;
        "Bearer your-api-key" 1;
    }

    include /etc/nginx/sites-enabled/*;
}
```

这里需要特别注意：

- `limit_req_zone` **只能写在 `http` 块里**
- 不能写在 `server` 块或 `location` 块里
- 你真正使用时，把 `your-api-key` 换成自己的值

---

#### 3.2 新建站点配置

新建 `/etc/nginx/sites-available/api.example.com`：

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

    location /v1/ {
        if ($api_key_ok = 0) {
            default_type application/json;
            return 401 '{"error":{"code":"UNAUTHORIZED","message":"Invalid API key"}}';
        }

        limit_req zone=api_limit burst=10 nodelay;

        proxy_pass http://127.0.0.1:9000;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_http_version 1.1;
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

    location / {
        default_type application/json;
        return 404 '{"error":{"code":"NOT_FOUND","message":"Not found"}}';
    }

    access_log /var/log/nginx/api.example.com.access.log;
    error_log  /var/log/nginx/api.example.com.error.log;
}
```

这份配置的含义是：

- 所有 `http://api.example.com` 的请求都会跳转到 HTTPS
- 所有 `/v1/` 请求都必须带正确的 `Authorization: Bearer your-api-key`
- 请求会被转发到云服务器本机的 `127.0.0.1:9000`
- 若后端不可达，返回统一 `503`
- 其他路径直接返回 `404`

启用配置：

```bash
sudo ln -s /etc/nginx/sites-available/api.example.com /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## B. 模型运行机器部分

### 1. 安装 Ollama

官网：

- https://ollama.com/

拉取模型：

```bash
ollama pull qwen2.5:7b
```

启动服务：

```bash
ollama serve
```

默认 Ollama 监听：

```text
http://127.0.0.1:11434
```

这是我们需要的效果。

> 不建议把 Ollama 监听到 `0.0.0.0`，更不建议直接用 frp 映射 Ollama 端口到公网。  
> Ollama 应只作为底层推理服务，被 FastAPI 在本机调用。

---

### 2. 安装并配置 frpc

从官方 Release 下载对应系统版本，然后创建 `frpc.toml`：

```toml
serverAddr = "你的云服务器公网IP"
serverPort = 7000

auth.method = "token"
auth.token = "your-strong-secret-token"

[[proxies]]
name = "ai-api"
type = "tcp"
localIP = "127.0.0.1"
localPort = 8080
remotePort = 9000
```

说明：

- `serverAddr`：云服务器地址
- `auth.token`：必须与 frps 保持一致
- `localPort = 8080`：本地 FastAPI 服务端口
- `remotePort = 9000`：云服务器映射出的端口，仅供本机 Nginx 访问

启动：

```bash
frpc -c frpc.toml
```

Windows 下一般是：

```bash
frpc.exe -c frpc.toml
```

---

### 3. 实现业务层（FastAPI）

安装依赖：

```bash
pip install fastapi uvicorn httpx pydantic
```

创建 `main.py`：

```python
import json
import logging
import time

import httpx
from fastapi import FastAPI, HTTPException, Request
from fastapi.responses import JSONResponse
from pydantic import BaseModel

app = FastAPI(title="My AI API")

OLLAMA_URL = "http://127.0.0.1:11434"
MODEL_NAME = "qwen2.5:7b"

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("my-ai-api")


# ── 数据模型 ────────────────────────────────────────────────

class ChatRequest(BaseModel):
    message: str
    system: str = "你是一个有帮助的助手。"


class ParseCommandRequest(BaseModel):
    text: str


# ── 工具函数 ────────────────────────────────────────────────

def extract_json_text(text: str) -> str:
    """
    尝试从模型输出中提取 JSON 文本。
    兼容 ```json ... ``` 代码块场景。
    """
    text = text.strip()
    if text.startswith("```"):
        lines = text.splitlines()
        if len(lines) >= 3:
            text = "\n".join(lines[1:-1]).strip()
    return text


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
                "stream": False
            },
        )
        response.raise_for_status()
        data = response.json()
        return data["message"]["content"]


# ── 接口 ────────────────────────────────────────────────────

@app.get("/v1/health")
async def health():
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
    except Exception as e:
        logger.exception("health check failed: %s", e)
        return JSONResponse(
            status_code=503,
            content={"status": "degraded", "backend": "offline"},
        )


@app.post("/v1/chat")
async def chat(req: ChatRequest):
    try:
        result = await call_ollama(req.message, req.system)
        return {
            "reply": result,
            "model": MODEL_NAME
        }
    except httpx.HTTPError as e:
        logger.exception("ollama request failed: %s", e)
        raise HTTPException(status_code=502, detail="backend request failed")
    except Exception as e:
        logger.exception("chat endpoint failed: %s", e)
        raise HTTPException(status_code=500, detail="internal server error")


@app.post("/v1/parse-command")
async def parse_command(req: ParseCommandRequest):
    system = """你是一个智能家居指令解析器。
请将用户输入解析为一个 JSON 对象，格式严格为：
{"device":"设备名","action":"动作","value":参数或null}

要求：
1. 只输出 JSON
2. 不要输出 Markdown 代码块
3. 不要输出解释文字
4. 无法确定时使用 null
"""

    try:
        result = await call_ollama(req.text, system)
        cleaned = extract_json_text(result)
        parsed = json.loads(cleaned)
        return {"command": parsed, "raw": result}
    except json.JSONDecodeError:
        logger.warning("model returned invalid json")
        return {"command": None, "raw": result, "warning": "模型未返回合法 JSON"}
    except httpx.HTTPError as e:
        logger.exception("ollama request failed: %s", e)
        raise HTTPException(status_code=502, detail="backend request failed")
    except Exception as e:
        logger.exception("parse-command failed: %s", e)
        raise HTTPException(status_code=500, detail="internal server error")


# ── 全局错误处理 ────────────────────────────────────────────

@app.exception_handler(HTTPException)
async def http_exception_handler(request: Request, exc: HTTPException):
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": {"code": "HTTP_ERROR", "message": exc.detail}},
    )


@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    logger.exception("unhandled exception: %s", exc)
    return JSONResponse(
        status_code=500,
        content={"error": {"code": "INTERNAL_ERROR", "message": "internal server error"}},
    )
```

启动：

```bash
uvicorn main:app --host 127.0.0.1 --port 8080
```

这里同样要强调：

> `--host 127.0.0.1` 是有意为之，不要改成 `0.0.0.0`。  
> 因为 FastAPI 业务层不应该直接暴露出去，它只需要本机可访问，然后由 frpc 抓本地 8080 转发即可。

---

# 七、systemd 服务化（推荐）

如果你不做服务化，那么：

- 重启后服务会消失
- 断线后不会自动恢复
- 管理和日志都会比较麻烦

下面给几个基础示例。

---

## 1. 云服务器：frps systemd

创建：

```bash
sudo nano /etc/systemd/system/frps.service
```

内容：

```ini
[Unit]
Description=FRP Server
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/frp
ExecStart=/opt/frp/frps -c /opt/frp/frps.toml
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

启用：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now frps
sudo systemctl status frps
```

---

## 2. 模型机：FastAPI systemd

创建：

```bash
sudo nano /etc/systemd/system/my-ai-api.service
```

内容：

```ini
[Unit]
Description=My AI FastAPI Service
After=network.target

[Service]
Type=simple
User=your-user
WorkingDirectory=/path/to/your/project
ExecStart=/usr/bin/python3 -m uvicorn main:app --host 127.0.0.1 --port 8080
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

启用：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now my-ai-api
sudo systemctl status my-ai-api
```

---

## 3. 模型机：frpc systemd

创建：

```bash
sudo nano /etc/systemd/system/frpc.service
```

内容：

```ini
[Unit]
Description=FRP Client
After=network.target

[Service]
Type=simple
User=your-user
WorkingDirectory=/opt/frp
ExecStart=/opt/frp/frpc -c /opt/frp/frpc.toml
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

启用：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now frpc
sudo systemctl status frpc
```

---

## 4. 模型机：Ollama 自启动

如果你的平台支持 systemd，也可以托管为服务。  
如果 Ollama 安装包已经注册过服务，可直接启用；否则你也可以自己创建 unit 文件。

例如：

```bash
sudo systemctl enable --now ollama
sudo systemctl status ollama
```

不同系统安装方式略有不同，请按实际环境调整。

---

# 八、验证部署是否成功

---

## 1. 本机验证 Ollama

在模型机上执行：

```bash
curl http://127.0.0.1:11434/api/tags
```

如果返回模型列表或 JSON，说明 Ollama 正常。

---

## 2. 本机验证 FastAPI

在模型机上执行：

```bash
curl http://127.0.0.1:8080/v1/health
```

如果返回：

```json
{
  "status": "ok",
  "backend": "online",
  "model": "qwen2.5:7b",
  "timestamp": 1710000000
}
```

说明业务层正常。

---

## 3. 云服务器验证 9000 是否可本机访问

在云服务器上执行：

```bash
curl http://127.0.0.1:9000/v1/health
```

如果 frp 已打通，应能拿到业务层响应。

同时你还应从**另一台外部机器**测试：

```bash
curl http://你的云服务器IP:9000/v1/health
```

这一步**理应失败**或超时。  
如果它成功了，说明 9000 被公网开放了，需要立刻检查：

- 云安全组
- UFW / iptables
- frp 暴露策略

---

## 4. 验证 HTTPS 入口

测试健康检查：

```bash
curl https://api.example.com/v1/health \
  -H "Authorization: Bearer your-api-key"
```

测试对话接口：

```bash
curl https://api.example.com/v1/chat \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your-api-key" \
  -d '{"message":"你好"}'
```

测试指令解析：

```bash
curl https://api.example.com/v1/parse-command \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your-api-key" \
  -d '{"text":"把客厅灯打开"}'
```

---

# 九、返回结果示例

## 1. `/v1/chat`

请求：

```json
{
  "message": "你好"
}
```

可能返回：

```json
{
  "reply": "你好！有什么我可以帮助你的吗？",
  "model": "qwen2.5:7b"
}
```

---

## 2. `/v1/parse-command`

请求：

```json
{
  "text": "把客厅灯调到 50%"
}
```

可能返回：

```json
{
  "command": {
    "device": "客厅灯",
    "action": "设置亮度",
    "value": 50
  },
  "raw": "{\"device\":\"客厅灯\",\"action\":\"设置亮度\",\"value\":50}"
}
```

如果模型没严格返回 JSON，则可能返回：

```json
{
  "command": null,
  "raw": "好的，以下是解析结果：{\"device\":\"客厅灯\",\"action\":\"设置亮度\",\"value\":50}",
  "warning": "模型未返回合法 JSON"
}
```

---

# 十、安全说明

这一部分非常重要，建议你在正式发布文章时单独列出来。

## 1. 不要开放 9000 给公网

这是整个方案里最容易被忽视、也是最危险的一点。

如果 9000 对公网开放，别人就可能直接访问：

- FastAPI 接口
- 绕过 Nginx 的 HTTPS
- 绕过 API Key 校验
- 绕过限流和统一错误处理

所以必须保证：

- 云安全组不放行 9000
- 本机防火墙不放行 9000
- Nginx 只通过 `127.0.0.1:9000` 访问它

---

## 2. 不要让 FastAPI 监听 `0.0.0.0`

正确示例：

```bash
uvicorn main:app --host 127.0.0.1 --port 8080
```

如果你改成：

```bash
uvicorn main:app --host 0.0.0.0 --port 8080
```

那么局域网内其他机器就可能直接访问你的业务层。

---

## 3. 不要把 Ollama 暴露给公网

Ollama 应只供本机 FastAPI 使用，不要：

- 改成 `0.0.0.0`
- 用 frp 直接映射 11434
- 对公网暴露原生 Ollama API

否则你会失去业务层带来的：

- 鉴权
- 请求格式控制
- 结果封装
- 日志与审计
- 功能扩展能力

---

## 4. `frp token` 不是业务 API Key

再次强调：

- `auth.token` 只保证 frpc 能否连上 frps
- 它不负责业务请求的用户访问控制

所以你的 API 仍然需要：

- API Key
- 或 JWT / OAuth / 网关鉴权
- 或更细粒度访问控制

---

## 5. 不要把异常原文返回给客户端

开发阶段很多人会习惯这样写：

```python
raise HTTPException(status_code=500, detail=str(e))
```

这会泄露很多内部信息。  
生产环境应统一返回：

- `internal server error`
- `backend request failed`

真正的异常细节写进日志，而不是直接回给调用方。

---

## 6. 密钥不要硬编码进仓库

以下内容都不建议直接写死在提交到 Git 的代码中：

- `your-api-key`
- `your-strong-secret-token`
- dashboard 密码
- 数据库密码
- 证书私钥路径和明文密码

更合理的方式是：

- 环境变量
- `.env`
- systemd Environment
- Docker secrets / secret manager

---

## 7. 公网长期使用建议升级安全方案

当前示例为了便于理解，使用了 frp 的 TCP 代理。  
如果要长期稳定运行，建议你进一步升级到：

- **STCP**
- 更严格的网关
- mTLS
- 零信任访问
- WAF / fail2ban / 审计日志

这会比直接长期裸跑 TCP 映射更稳妥。

---

# 十一、当前方案的优点

这套架构比较适合个人或小型自建系统，有几个明显优点：

1. **弱设备可以像调用 API 一样调用大模型**
2. **模型仍运行在你自己的机器上**
3. **云服务器只承担入口与转发，不承担推理成本**
4. **业务层可以自由扩展**
   - 自定义接口
   - 指令解析
   - 工具调用
   - MCP 适配
   - IoT 控制
5. **统一网关便于后续做鉴权、限流、日志与权限控制**

---

# 十二、当前方案的局限性

同时也要清楚它的边界：

1. **延迟会高于纯本地调用**
   - 请求要经过公网
   - 还要走云服务器转发
2. **稳定性依赖两台机器**
   - 云服务器
   - 模型机
3. **模型机如果断网 / 睡眠 / 宕机，服务就不可用**
4. **家庭网络上行带宽会影响体验**
5. **安全性很大程度取决于你是否正确处理暴露面**

---

# 十三、后续可扩展方向

在这个基础上，你后续其实可以很自然地扩展出很多东西，比如：

- `/v1/chat`：通用对话
- `/v1/parse-command`：自然语言转结构化指令
- `/v1/embedding`：文本向量化
- `/v1/rerank`：重排序
- `/v1/tools/...`：工具调用
- `/v1/iot/...`：智能家居控制
- `/v1/mcp/...`：接 MCP Server
- 配套 Web 面板
- 桌面客户端
- 手机端快捷入口

也就是说，这套架构不仅是“远程调用一个模型”，它可以成为你整个自建 AI 系统的底座。

---

