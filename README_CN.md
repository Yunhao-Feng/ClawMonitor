# ClawMonitor - OpenClaw 批量任务调度系统

<div align="center">

[![Docker](https://img.shields.io/badge/Docker-required-blue.svg)](https://www.docker.com/)
[![Python](https://img.shields.io/badge/Python-3.12+-green.svg)](https://www.python.org/)
[![License](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

**高性能异步并发的 OpenClaw 任务批量执行框架**

[English](README.md) | 简体中文

</div>

---

## 📖 项目简介

ClawMonitor 是一个基于 Docker 容器的批量任务调度系统，专为 [OpenClaw](https://github.com/openclaw/openclaw) 设计。它支持：

- ✨ **异步并发执行** - 真正的并发任务处理，支持自定义并发数
- 🔄 **智能重试机制** - 可配置的指数退避重试策略
- 🐳 **Docker 隔离** - 每个任务运行在独立的容器中
- 📊 **进度追踪** - 实时查看任务执行状态
- 💾 **结果持久化** - 自动保存任务执行结果到 JSON

---

## 🚀 快速开始

### 前置要求

在开始之前，请确保你的系统已安装：

1. **Docker** (必需)
   - macOS: [Docker Desktop for Mac](https://docs.docker.com/desktop/install/mac-install/)
   - Windows: [Docker Desktop for Windows](https://docs.docker.com/desktop/install/windows-install/)
   - Linux: [Docker Engine](https://docs.docker.com/engine/install/)

2. **Python 3.12+** (用于运行批量调度器)
   ```bash
   python --version  # 确认 Python 版本
   ```

3. **Conda/Miniconda** (推荐，用于环境管理)
   - 下载：[Miniconda](https://docs.conda.io/en/latest/miniconda.html)

### 安装步骤

#### 1. 克隆仓库

```bash
git clone https://github.com/yourusername/ClawMonitor.git
cd ClawMonitor
```

#### 2. 创建 Python 环境

**选项 A：使用 Conda (推荐)**

```bash
# 从 environment.yml 创建环境
conda env create -f environment.yml

# 激活环境
conda activate base
```

**选项 B：使用 pip**

```bash
# 创建虚拟环境
python -m venv venv
source venv/bin/activate  # Linux/macOS
# 或 venv\Scripts\activate  # Windows

# 安装依赖
pip install -r requirements.txt
```

#### 3. 配置 API 密钥

编辑 `config.yaml` 文件，填入你的 API 配置：

```yaml
# ============ API 配置 (必填) ============
api:
  key: "your-api-key-here"           # 填入你的 API Key
  url: "https://your-api-url.com"    # 填入 API 地址

# ============ Gateway 配置 ============
gateway:
  auth_token: "your-secure-token"    # 填入 Gateway 认证令牌

# ============ 并发配置 ============
max_concurrent: 5                    # 同时运行的任务数（建议 3-10）
```

#### 4. 构建 Docker 镜像

```bash
# 构建 OpenClaw 镜像
docker build -t clawmonitor-openclaw:latest -f Dockerfile.openclaw .
```

#### 5. 准备任务数据

将你的任务放在 `decomposed_epoch1.jsonl` 文件中，格式如下：

```json
{"record_id": "task-001", "instruction": "完成XX任务", "category": "coding", "deomposed_query": "{\"turns\": [{\"thought\": \"分析需求\", \"output\": \"请帮我...\"}, {\"thought\": \"实现功能\", \"output\": \"现在开始...\"}]}"}
```

#### 6. 运行批量任务

```bash
python batch_runner.py
```

---

## 📋 配置说明

### 核心配置参数

#### 1. API 配置 (必填)

```yaml
api:
  key: ""          # 你的 API 密钥
  url: ""          # API 端点 URL
```

#### 2. 模型配置

```yaml
model:
  id: "gpt-51-1113-global"           # 模型 ID
  name: "gpt-51-1113-global"         # 模型名称
  context_window: 400000             # 上下文窗口大小
  max_tokens: 128000                 # 最大生成 tokens
```

#### 3. 并发控制

```yaml
max_concurrent: 5                    # 同时运行的容器数量
                                     # 建议值：3-10（取决于系统资源）
                                     # 更高的值 = 更快完成，但占用更多内存

agent:
  max_concurrent: 4                  # Agent 内部并发数
  subagents_max_concurrent: 8        # 子 Agent 并发数
```

**性能示例：**
- `max_concurrent: 5` + 每任务 10 分钟 = **10 分钟**完成 5 个任务
- `max_concurrent: 1` + 每任务 10 分钟 = **50 分钟**完成 5 个任务

#### 4. 重试策略

```yaml
retry:
  # Prompt 请求重试配置
  prompt:
    max_attempts: 3      # 失败后最多重试 3 次
    min_wait: 4          # 首次重试等待 4 秒
    max_wait: 10         # 最长等待 10 秒（指数退避）

  # 健康检查重试配置
  health_check:
    max_attempts: 30     # 容器启动检测重试 30 次
    min_wait: 2          # 每次等待 2 秒
    max_wait: 5          # 最长等待 5 秒
```

**重试行为示例：**
```
尝试 1: 立即发送
失败 ❌ → 等待 4 秒

尝试 2: 重试
失败 ❌ → 等待 8 秒

尝试 3: 最后尝试
成功 ✅ 或 失败 ❌
```

#### 5. Gateway 配置

```yaml
gateway:
  port: 18789                        # Gateway 端口
  mode: "local"                      # 运行模式
  bind: "lan"                        # 网络绑定（Docker 必须用 lan）
  auth_token: ""                     # 认证令牌（必填）
```

#### 6. 功能开关

```yaml
features:
  browser_enabled: true              # 启用浏览器功能
  browser_evaluate_enabled: true     # 启用浏览器评估
  browser_attach_only: true          # 仅附加到现有浏览器
  cron_enabled: true                 # 启用定时任务
  web_enabled: true                  # 启用 Web 功能
  canvas_host_enabled: true          # 启用 Canvas 主机
```

---

## 🔧 使用指南

### 基础用法

#### 1. 单次批量执行

```bash
# 使用默认配置文件 (config.yaml) 和默认任务文件 (decomposed_epoch1.jsonl)
python batch_runner.py
```

#### 2. 指定配置和任务文件

修改 `batch_runner.py` 的初始化参数：

```python
runner = BatchRunner(
    config_path="custom_config.yaml",    # 自定义配置
    jsonl_path="my_tasks.jsonl"          # 自定义任务文件
)
runner.run()
```

#### 3. 查看执行日志

```bash
# 实时查看容器日志
docker logs -f openclaw-task-<record_id>

# 查看所有运行中的容器
docker ps
```

### 高级用法

#### 1. 调整并发数

根据你的系统资源调整并发数：

```yaml
# config.yaml
max_concurrent: 10  # 更激进的并发（需要更多内存）
```

**资源估算：**
- 每个容器约需 **2-4GB 内存**
- `max_concurrent: 5` ≈ 10-20GB 内存
- `max_concurrent: 10` ≈ 20-40GB 内存

#### 2. 调整重试策略

针对不稳定的 API：

```yaml
retry:
  prompt:
    max_attempts: 5    # 增加重试次数
    min_wait: 2        # 更快重试
    max_wait: 20       # 允许更长等待
```

#### 3. 持久化会话数据

编辑 `docker-compose.yml`：

```yaml
volumes:
  - ./sessions:/root/.openclaw/agents/main/sessions
  - ./workspace:/root/.openclaw/workspace
```

#### 4. 使用 Docker Compose（可选）

```bash
# 单容器测试
docker-compose up -d

# 查看日志
docker-compose logs -f

# 停止服务
docker-compose down
```

---

## 📂 项目结构

```
ClawMonitor/
├── batch_runner.py              # 批量任务调度器（核心）
├── api-server.py                # FastAPI 服务器
├── config.yaml                  # 配置文件（需填写 API 密钥）
├── requirements.txt             # Python 依赖
├── environment.yml              # Conda 环境文件
├── decomposed_epoch1.jsonl      # 任务数据文件
├── Dockerfile.openclaw          # Docker 镜像构建文件
├── docker-compose.yml           # Docker 编排文件
├── entrypoint.sh                # 容器启动脚本
├── .gitignore                   # Git 忽略规则
└── batch_output/                # 任务输出目录（自动创建）
    ├── task-001.json
    ├── task-002.json
    └── ...
```

---

## 📊 输出格式

每个任务完成后会在 `batch_output/` 目录生成对应的 JSON 文件：

```json
{
  "record_id": "task-001",
  "session_id": "sess-abc123",
  "instruction": "原始任务描述",
  "category": "coding",
  "total_turns": 3,
  "turns": [
    {
      "turn": 1,
      "thought": "分析需求",
      "prompt": "请帮我分析这个需求",
      "response": "Agent 的回复内容..."
    },
    {
      "turn": 2,
      "thought": "实现功能",
      "prompt": "现在开始实现",
      "response": "Agent 的实现内容..."
    }
  ],
  "original_task": { ... }
}
```

---

## ⚠️ 注意事项

### 资源要求

- **内存**: 建议至少 16GB，推荐 32GB+
- **磁盘**: 每个任务约占用 1-5MB，准备充足空间
- **CPU**: 多核 CPU 可显著提升并发性能

### 常见问题

#### 1. Docker 容器启动失败

```bash
# 检查 Docker 是否运行
docker info

# 检查镜像是否存在
docker images | grep clawmonitor

# 重新构建镜像
docker build -t clawmonitor-openclaw:latest -f Dockerfile.openclaw .
```

#### 2. API 请求失败

- 检查 `config.yaml` 中的 `api.key` 和 `api.url` 是否正确
- 确认 API 配额是否充足
- 查看重试日志，适当增加 `retry.prompt.max_attempts`

#### 3. 内存不足

```yaml
# 减少并发数
max_concurrent: 3  # 从 5 降到 3
```

或调整 Docker 资源限制：

```yaml
# docker-compose.yml
deploy:
  resources:
    limits:
      memory: 2G  # 限制单容器内存
```

#### 4. 任务卡住不动

```bash
# 查看容器日志
docker logs openclaw-task-<record_id>

# 手动停止卡住的容器
docker rm -f openclaw-task-<record_id>

# 重新运行批量任务
python batch_runner.py
```

---

## 🛠️ 开发指南

### 修改重试逻辑

编辑 `batch_runner.py` 中的重试装饰器：

```python
@retry(
    stop=stop_after_attempt(5),           # 改为 5 次
    wait=wait_exponential(multiplier=2),  # 调整退避倍数
    ...
)
```

### 添加自定义日志

```python
import logging

logger = logging.getLogger(__name__)
logger.info("自定义日志信息")
```

### 调试模式

```bash
# 设置环境变量启用详细日志
export LOG_LEVEL=DEBUG
python batch_runner.py
```

---

## 📄 License

MIT License - 详见 [LICENSE](LICENSE) 文件

---

## 🤝 贡献

欢迎提交 Issue 和 Pull Request！

1. Fork 本仓库
2. 创建特性分支 (`git checkout -b feature/AmazingFeature`)
3. 提交更改 (`git commit -m 'Add some AmazingFeature'`)
4. 推送到分支 (`git push origin feature/AmazingFeature`)
5. 开启 Pull Request

---

## 📞 支持

如有问题，请：

1. 查看 [常见问题](#常见问题) 章节
2. 提交 [Issue](https://github.com/yourusername/ClawMonitor/issues)
3. 加入讨论组

---

<div align="center">

**如果这个项目对你有帮助，请给一个 ⭐️ Star！**

</div>
