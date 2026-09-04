# LLM API Gateway

迁移到正式仓库： https://github.com/boonya-hrgk/llm-api-gateway

> 大模型 API 代理网关 + `sk-*` 密钥授权管理，基于 FastAPI 构建，支持多上游路由、流式 SSE 透传、内置对话界面与 Docker 一键部署。

## 功能界面
系统登录：
![login](images/login.png)

系统概览：
 ![overview](images/overview.png)

用量统计：
![statistics](images/statistics.png)

密钥管理：
![token](images/token.png)

上游管理：

![provider](images/provider.png)

用户管理：
![user](images/user.png)

对话测试：

![chat](images/chat.png)

## 项目简介

`llm-api-gateway` 是一个轻量级的大模型 API 网关，部署在你的上游大模型服务（如 vLLM、Ollama、Xinference 等 OpenAI 兼容服务）之前，统一对外提供 `/v1/*` 接口，并为每个调用方发放独立的 `sk-` 密钥进行鉴权与用量追踪。

核心价值：
- **多上游路由**：支持配置多个上游大模型服务，每个密钥可绑定到不同上游，实现地址映射与多模型统一网关管理。
- **密钥隔离**：上游真实 API Key 不再分发给调用方，所有流量经网关统一鉴权后透传。
- **安全存储**：`/v1/*` 鉴权仅比对 `sha256` 哈希，列表展示统一脱敏为 `sk-ab12****`；明文 `sk-` 仅在内网后台落库，供创建/重置回显与对话测试（与上游 API Key 同等保护，管理员即密钥持有者）。
- **生命周期管理**：支持密钥发放、列表查询、过期时间设置、吊销与**一键重置**（丢失/泄露后立即换发新钥，旧值即刻失效）。
- **方言透传**：完整支持 SSE 流式响应，适配 `chat/completions` 等流式场景；上游标注 OpenAI / Anthropic 方言时网关自动做请求响应翻译。
- **内置对话界面**：管理后台自带聊天窗口，支持流式输出、多轮上下文、Markdown 渲染与原始响应调试。

## 功能特性

- 🔄 反向代理 `/v1/*` 全量透传至上游大模型服务
- 🌐 **多上游管理**：支持配置多个上游服务，每个密钥独立绑定
- 🔐 `sk-` 密钥鉴权（Bearer Token），支持创建后明文回显 / 丢失一键重置
- 🛡️ 管理接口 `MASTER_KEY` 保护，支持多用户（管理员/查看者）
- 🗄️ SQLite 持久化，零外部依赖
- 📊 密钥用量统计（最后使用时间、请求次数）
- ⏱️ 密钥过期（原生时间选择器设置）、吊销与一键重置
- 💬 内置 Web 对话界面（流式输出 + 多轮上下文 + Markdown 渲染 + 消息工具条 + 原始响应调试）
- 🌐 内置 Web 管理界面
- 🐳 Docker 容器化部署

## 架构图

![token](images/llm-api-gateway-architecture.png)

## 技术栈

| 层级 | 技术 |
|------|------|
| Web 框架 | FastAPI + Uvicorn |
| HTTP 客户端 | httpx（异步、流式） |
| 数据库 | SQLite + aiosqlite |
| 配置管理 | pydantic-settings |
| 前端渲染 | marked + highlight.js + DOMPurify（vendor 本地打包，离线/内网可用） |
| 容器化 | Docker (python:3.11-slim) |

## 项目结构

```
llm-api-gateway/
├── proxy/
│   ├── app/
│   │   ├── main.py        # FastAPI 入口，挂载路由与静态资源
│   │   ├── config.py      # 环境变量配置
│   │   ├── proxy.py       # /v1/* 反向代理（支持流式 SSE + 多上游路由）
│   │   ├── auth.py        # sk 鉴权 + MASTER_KEY 校验 + 用户管理
│   │   ├── admin.py       # /admin/* 密钥/上游/用户管理接口
│   │   ├── db.py          # SQLite 数据访问层
│   │   ├── schemas.py     # Pydantic 请求/响应模型
│   │   └── static/        # Web 管理界面（对话/上游/密钥/用户）
│   │       ├── index.html # 管理页面结构
│   │       ├── app.js     # 交互逻辑（对话流式、密钥/上游管理）
│   │       ├── styles.css # 界面样式
│   │       └── vendor/    # 前端依赖本地打包（marked/highlight.js/DOMPurify）
│   ├── main.py            # 一键启动入口：python main.py
│   ├── .env.example       # 配置文件模板
│   └── requirements.txt
├── Dockerfile
├── docker-run.example.sh  # 启动示例脚本
├── fix-docker-mirror.ps1  # Docker 镜像源修复脚本
└── .dockerignore
```

## 快速开始

### 方式一：Docker 部署（推荐）

1. 构建镜像：

```powershell
docker build -t llm-api-gateway .
```

2. 启动容器（PowerShell）：

```powershell
docker run -d `
  --name llm-api-gateway `
  -p 9000:9000 `
  -e MASTER_KEY=change-me-master `
  -e VLLM_TARGET_URL=http://host.docker.internal:8000 `
  -e UPSTREAM_API_KEY=optional-upstream-key `
  -v "${PWD}/data:/app/data" `
  --restart unless-stopped `
  llm-api-gateway
```

或使用 CMD（命令提示符）：

```cmd
docker run -d ^
  --name llm-api-gateway ^
  -p 9000:9000 ^
  -e MASTER_KEY=change-me-master ^
  -e VLLM_TARGET_URL=http://host.docker.internal:8000 ^
  -e UPSTREAM_API_KEY=optional-upstream-key ^
  -v "%cd%/data:/app/data" ^
  --restart unless-stopped ^
  llm-api-gateway
```

3. 访问管理界面：<http://localhost:9000>

### 方式二：本地运行（推荐，最简单）

**第一步**：复制配置文件并修改

```powershell
cd proxy
Copy-Item .env.example .env
```

然后用记事本打开 `.env` 文件，把 `MASTER_KEY` 改成你自己的密码，把 `VLLM_TARGET_URL` 改成你的大模型服务地址。

**第二步**：安装依赖并启动

```powershell
pip install -r requirements.txt
python main.py
```

启动成功后访问 <http://localhost:9000> 即可。

## 环境变量

| 变量 | 必填 | 默认值 | 说明 |
|------|:----:|--------|------|
| `MASTER_KEY` | ✅ | - | 管理接口与 Web 登录的主密钥 |
| `VLLM_TARGET_URL` | ❌ | `http://127.0.0.1:8000` | 上游大模型服务地址 |
| `UPSTREAM_API_KEY` | ❌ | - | 上游自身开启鉴权时填入 |
| `PROXY_HOST` | ❌ | `0.0.0.0` | 监听地址 |
| `PROXY_PORT` | ❌ | `9000` | 监听端口 |
| `DB_PATH` | ❌ | `data/keys.db` | SQLite 数据库路径 |

## API 接口

### 公开接口

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/info` | 网关基本信息 |
| GET | `/health` | 健康检查（探测上游可达性） |

### 代理接口（需 `sk-` 密钥）

| 方法 | 路径 | 说明 |
|------|------|------|
| * | `/v1/{path}` | 透传至上游 `/v1/*`，支持 GET/POST/PUT/DELETE/PATCH |

调用示例：

```bash
curl http://localhost:9000/v1/chat/completions \
  -H "Authorization: Bearer sk-你的密钥" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "你的模型名",
    "messages": [{"role": "user", "content": "你好"}],
    "stream": true
  }'
```

### 管理接口（需 `MASTER_KEY`）

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/admin/keys` | 发放新密钥（响应含明文，弹窗回显） |
| GET | `/admin/keys` | 密钥列表 |
| GET | `/admin/keys/{id}` | 查询单个密钥 |
| GET | `/admin/keys/{id}/reveal` | 回显密钥明文（管理员；旧版无明文密钥返回 410） |
| POST | `/admin/keys/{id}/reset` | 重置密钥：换发新明文，旧值立即失效 |
| DELETE | `/admin/keys/{id}` | 吊销密钥 |
| PATCH | `/admin/keys/{id}/upstream` | 修改密钥绑定的上游 |
| GET | `/admin/upstreams` | 上游列表 |
| POST | `/admin/upstreams` | 添加上游 |
| PUT | `/admin/upstreams/{id}` | 更新上游 |
| DELETE | `/admin/upstreams/{id}` | 删除上游 |
| GET | `/admin/users` | 用户列表 |
| POST | `/admin/users` | 创建用户 |
| PUT | `/admin/users/{id}` | 更新用户 |
| DELETE | `/admin/users/{id}` | 删除用户 |

发放密钥示例：

```bash
curl -X POST http://localhost:9000/admin/keys \
  -H "Authorization: Bearer change-me-master" \
  -H "Content-Type: application/json" \
  -d '{"name": "测试客户端", "expires_at": "2026-12-31T23:59:59+00:00"}'
```

响应：

```json
{
  "id": 1,
  "key": "sk-aBcDeFgHiJkLmNOpQrStUvWxYz123456",
  "name": "测试客户端",
  "status": "active",
  "created_at": "2026-08-13T10:00:00+00:00",
  "expires_at": "2026-12-31T23:59:59+00:00"
}
```

> ⚠️ `key` 字段为明文。创建/重置时弹窗回显一次；密钥丢失可在后台「重置」换发新钥，旧值立即失效。

## Web 管理界面

访问 <http://localhost:9000>，使用 `MASTER_KEY` 登录（管理员账号）。

### 功能页面

| 页面 | 权限 | 说明 |
|------|:----:|------|
| 系统概览 | 所有用户 | 网关状态、密钥数量、上游数量等概览信息 |
| 用量统计 | 所有用户 | 密钥使用次数统计 |
| 密钥管理 | 所有用户 | 密钥列表、创建（时间选择器设过期）、重置、吊销、绑定上游 |
| 上游管理 | 管理员 | 多上游服务配置（名称、地址、API Key、协议类型、可用模型、默认状态） |
| 用户管理 | 管理员 | 后台用户管理（管理员/查看者角色） |
| 对话 | 所有用户 | 内置聊天窗口：流式输出、Markdown 渲染、消息工具条、原始响应调试 |

### 密钥管理

列表对所有登录用户可见，创建 / 重置 / 吊销 / 回显明文等写操作需**管理员**权限：

- **创建密钥**：填写名称 → 绑定上游（可留空走默认上游）→ 用原生时间选择器设置过期时间（留空永不过期）→ 创建后弹窗回显明文 `sk-`，请立即复制保存。
- **查看明文**：密钥明文不会常驻列表（列表仅显示 `sk-ab12****` 前缀）。需要时可回到对话测试页「从列表选择」该密钥自动获取完整 Key；旧版本创建的密钥可能无明文，重置即可换发一把新钥。
- **重置**：密钥丢失或泄露时点击「重置」——立即生成新密钥替换旧值，旧 key 即刻失效，名称/绑定上游/调用统计保持不变，新明文弹窗回显一次。
- **吊销**：彻底作废该密钥，吊销后不可恢复。

### 对话界面使用

对话页面支持两种使用模式：

- **聊天模式**（默认）：流式输出 + 多轮上下文，体验类似 ChatGPT
- **调试模式**：勾选「显示原始响应」，可查看完整的 JSON / SSE 原始数据

回复渲染与消息操作：

- **Markdown 渲染**：AI 回复按 Markdown 实时渲染——标题、表格、列表、引用、代码块高亮（自动识别语言并带语言标签与「复制代码」按钮）均开箱即用；整条回复也可一键复制 Markdown 原文。
- **消息工具条**：每条消息下方提供复制 Markdown、点赞/点踩、朗读（TTS）、重新生成、分享六项快捷操作。
- **生成状态**：等待响应时显示「AI 正在思考…」动态指示器；流式输出期间平滑渐进渲染，结束后再补全代码高亮。

API Key 支持两种方式：

- **从列表选择**：管理员可直接选择已有密钥，自动获取完整 Key
- **手动输入**：粘贴任何 `sk-` 密钥进行测试

模型选择：

- 模型输入框支持**下拉选择或直接键入**——下拉候选自动聚合全部上游配置的「可用模型」（去重保序）。
- 切换密钥后自动跟随该密钥绑定上游的首个模型；手动指定的模型不会被自动覆盖。

## 多上游路由（地址映射）

本项目支持配置多个上游大模型服务，通过密钥与上游的绑定关系实现地址映射。

### 使用场景

- 本地有多个 vLLM 实例（8000、8001、8002...），分别承载不同模型
- 同时接入本地模型和云端 API（如 DeepSeek、智谱等）
- 不同调用方使用不同的大模型服务

### 工作原理

```
密钥A ──绑定──▶ 上游1 (vLLM-8000) ──▶ deepseek-coder
密钥B ──绑定──▶ 上游2 (vLLM-8001) ──▶ qwen2.5-72b
密钥C ──绑定──▶ 上游3 (DeepSeek云) ─▶ 外部API
```

调用方只需携带网关分配的 `sk-` 密钥，网关自动根据绑定关系转发到对应的上游服务。

### 配置步骤

1. 登录管理后台 → **上游管理** → 添加上游（填写名称、基础地址、可选 API Key）
2. 按需配置上游的**协议类型**与**可用模型**（见下）
3. 在**密钥管理**中创建或编辑密钥，选择绑定的上游
4. 调用方使用该密钥请求 `/v1/chat/completions`，网关自动转发到对应上游

> 💡 密钥未绑定上游时，会使用默认上游。可在上游管理中设置默认上游。

### 上游字段：协议类型与可用模型

- **协议类型**：标注该上游的方言——`OpenAI` 或 `Anthropic`。网关会自动完成两种方言间请求/响应的翻译（流式与非流式均支持），例如把 OpenAI 客户端请求转成 Anthropic Messages 格式转发给 Claude 类上游。
- **可用模型**：一行一个，填写该上游可用的模型名（如 `deepseek-chat`）。对话测试页的模型下拉会**聚合所有上游的可用模型**自动生成候选；留空则该上游不贡献候选，只能手动输入。

## 安全说明

- **哈希鉴权 + 明文回显**：`/v1/*` 网关鉴权只比对 `sha256` 哈希；明文 `sk-` 同时落库，仅在内网管理后台经管理员权限回显（创建/重置弹窗、对话测试「从列表选择」），列表与查询接口一律只返回 `sk-ab12****` 前缀。
- **一键重置**：密钥丢失/泄露可在密钥管理中「重置」，立即换发新明文并覆盖哈希，旧值即刻失效。
- **管理接口隔离**：`/admin/*` 与 `/v1/*` 使用独立密钥体系，互不影响。
- **过期与吊销**：过期或被吊销的密钥立即失效，无法继续调用。

## 常见问题

### Q1: `pip install` 报错 `check_hostname requires server_hostname`

**原因**：系统开了代理软件，代理格式异常导致 pip 崩溃。

**解决**：

```powershell
# 临时清除当前窗口的代理变量
$env:HTTP_PROXY = $null
$env:HTTPS_PROXY = $null
$env:http_proxy = $null
$env:https_proxy = $null
$env:NO_PROXY = "*"

# 用清华源安装
pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple
```

如果还是不行，打开 `Internet 选项` → `连接` → `局域网设置`，取消代理勾选后重启 PowerShell。

### Q2: 启动报错 `端口被占用` (error 10048)

**原因**：9000 端口已经被其他程序占了。

**解决一：杀掉占用进程**

```powershell
netstat -ano | findstr :9000
taskkill /F /PID 进程号
```

**解决二：换个端口**

在 `.env` 文件里加一行：

```
PROXY_PORT=9001
```

然后重新 `python main.py`，访问 <http://localhost:9001>。

### Q3: 创建密钥后弹窗里明文是空的

**原因**：旧版本的服务还在运行，代码修复未生效。

**解决**：杀掉旧进程，重新 `python main.py` 启动新版本。

### Q4: Docker 构建失败 `403 Forbidden`

**原因**：Docker 镜像加速器（如阿里云）已失效。

**解决**：运行项目根目录下的修复脚本：

```powershell
.\fix-docker-mirror.ps1
```

按提示选择中科大镜像源，然后重启 Docker Desktop 再构建。

## 许可证

本项目仅供学习与内部使用。
