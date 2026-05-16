# Raycast AI OpenRouter Proxy 使用文档

> [为 Raycast AI 接入中转服务](https://github.com/tofrankie/blog/issues/393)

这份文档基于项目 `README.md` 和实际代码整理。

## 1. 这个项目做什么

这个服务会把 Raycast 发来的 Ollama 风格请求，转发成 OpenAI 兼容接口请求（默认是 OpenRouter）。

- 你在 Raycast 里把 `Ollama Host` 指向本地代理
- 代理再拿你的 `API_KEY` 调用真实模型服务

## 2. 先决条件

- 已安装 Docker Desktop（Mac/Windows）或 Docker Engine（Linux）
- 有可用的 API Key（例如 OpenRouter key）

快速确认 Docker 可用：

```bash
docker --version
docker compose version
```

## 3. 核心配置文件说明

项目里和 Docker 相关的关键文件：

- `docker-compose.yml`
- `Dockerfile`
- `models.json`

### 3.1 `docker-compose.yml`

当前默认配置等价于：

- 容器服务名：`raycast-ai-proxy`
- 宿主机端口映射：`${HOST_PORT:-11435}:${PORT:-3000}`
- 挂载模型配置：`./models.json:/app/models.json:ro`
- 环境变量：
  - `PORT=${PORT:-3000}`
  - `API_KEY=${API_KEY}`
  - `BASE_URL=${BASE_URL:-https://openrouter.ai/api/v1}`

说明：

- 代码里服务监听端口是 `PORT`，默认 `3000`（`src/config.ts`）
- 通过 `.env` 可同时控制容器内端口（`PORT`）和宿主机端口（`HOST_PORT`）
- 你实际在 Raycast 里填写的是 `localhost:HOST_PORT`

### 3.2 `models.json`（决定 Raycast 看到哪些模型）

启动时会从容器内 `/app/models.json` 读取并校验，校验失败会直接启动失败。

每个模型至少要有：

- `name`（Raycast 显示名）
- `id`（提供商模型 ID）
- `contextLength`（正整数）
- `capabilities`（`vision` / `tools` 组合）

可选项：

- `temperature`（0-2）
- `topP`（0-1）
- `max_tokens`（正整数）
- `extra`（透传给提供商）

## 4. 第一次启动（推荐流程）

### 步骤 1：进入项目目录

```bash
cd /Users/frankie/Web/Git/raycast-ai-openrouter-proxy
```

### 步骤 2：从模板创建 `.env`

先复制模板文件：

```bash
cp .env.example .env
```

再按需修改 `.env`。

推荐最小配置：

```env
PORT=3000
HOST_PORT=11435
API_KEY=你的真实key
BASE_URL=https://openrouter.ai/api/v1
```

如果你不用 OpenRouter，而是别的 OpenAI 兼容服务，把 `BASE_URL` 改成对应地址。

### 步骤 3：检查 `models.json`

先用项目自带示例，后续再按需增删模型。

注意：

- `name` 不能重复（代码里会检查，重复会报错）
- `id` 必须是你目标平台支持的模型 ID

### 步骤 4：构建并启动

```bash
docker compose up -d --build
```

### 步骤 5：看服务是否起来

```bash
docker compose ps
docker compose logs --tail=200
```

日志里出现类似 `Server is up on port 3000` 说明容器内服务正常启动。

## 5. Raycast 侧配置

在 Raycast 设置里：

- 打开 `AI > Ollama Host`
- 填 `localhost:HOST_PORT`（例如 `localhost:11435`）

如果你在 `.env` 里把 `HOST_PORT` 改成了 `22435`，这里就填 `localhost:22435`。

## 6. 日常操作命令

启动（后台）：

```bash
docker compose up -d
```

停止：

```bash
docker compose down
```

重启：

```bash
docker compose restart
```

只看最近日志：

```bash
docker compose logs --tail=200
```

持续看日志：

```bash
docker compose logs -f
```

## 7. 修改配置后是否需要重启

- 改 `models.json`：需要 `docker compose restart`
- 改 `docker-compose.yml`（环境变量、端口、挂载）：建议 `docker compose up -d --build`
- 改 TypeScript 源码：需要重新构建镜像，执行 `docker compose up -d --build`

## 8. 常见问题排查

### 8.1 Raycast 连不上

先确认：

- 容器是否在跑：`docker compose ps`
- Raycast 的 Host 是否是 `localhost:HOST_PORT`
- 端口是否被占用（可改 `.env` 中 `HOST_PORT`，例如 `22435`）

### 8.2 启动失败并提示模型配置错误

`models.json` 字段类型或值范围不符合要求（如 `topP` 超过 1、`contextLength` 非正整数）。

看日志定位：

```bash
docker compose logs --tail=200
```

### 8.3 能启动但对话报模型不存在

Raycast 传入的是模型 `name`，代码按 `name` 查找配置。

排查：

- Raycast 中选择的模型名是否和 `models.json` 里的 `name` 完全一致
- 是否在改完 `models.json` 后执行过 `docker compose restart`

### 8.4 API Key 或上游地址问题

检查：

- `API_KEY` 是否有效
- `BASE_URL` 是否是正确的 OpenAI 兼容接口根地址（例如 OpenRouter 用 `https://openrouter.ai/api/v1`）

可用以下命令检查 compose 最终读取到的值：

```bash
docker compose config
```
