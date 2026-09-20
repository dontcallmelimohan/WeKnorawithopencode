# WeKnora 启动与关闭说明

## 1. 先准备环境

在项目根目录执行：

```bash
cp -n .env.example .env
```

如果要用 Lite 模式：

```bash
cp -n .env.lite.example .env.lite
```

## 2. 启动项目

### 方式 A：Lite 模式（推荐，适合本地快速启动，且不依赖 Docker 镜像拉取）

```bash
make run-lite
```

启动后访问：

```text
http://localhost:8080
```

### 方式 B：开发模式（启动基础设施 + 本地后端 + 前端）

先启动基础设施：

```bash
make dev-start
```

另开一个终端启动后端：

```bash
make dev-app
```

再另开一个终端启动前端：

```bash
make dev-frontend
```

如果你只需要最基础的本地开发依赖：

```bash
make dev-start
```

## 3. 关闭项目

### Lite 模式

在终端里按：

```bash
Ctrl + C
```

即可停止正在运行的服务进程。

### 开发模式

停止基础设施：

```bash
make dev-stop
```

如果需要重启：

```bash
make dev-restart
```

## 4. 额外说明

- 目前项目在本地运行时，Lite 模式最稳妥，且已经验证可启动。
- 完整 Docker 模式需要下载镜像，可能因网络或镜像源问题导致拉取失败。
- 如果项目在运行中遇到端口占用，请检查是否已有同名服务占用了 8080 端口。

## 5. 常用快速命令

```bash
# 启动 Lite
make run-lite

# 启动基础设施
make dev-start

# 启动后端
make dev-app

# 启动前端
make dev-frontend

# 停止基础设施
make dev-stop
```
