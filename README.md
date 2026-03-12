# ClawMonitor - OpenClaw Batch Task Scheduler

<div align="center">

[![Docker](https://img.shields.io/badge/Docker-required-blue.svg)](https://www.docker.com/)
[![Python](https://img.shields.io/badge/Python-3.12+-green.svg)](https://www.python.org/)
[![License](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

**High-performance asynchronous batch task execution framework for OpenClaw**

English | [简体中文](README_CN.md)

</div>

---

## 📖 Introduction

ClawMonitor is a Docker-based batch task scheduler designed for [OpenClaw](https://github.com/openclaw/openclaw). Key features:

- ✨ **Async Concurrency** - True parallel task execution with configurable concurrency
- 🔄 **Smart Retry** - Configurable exponential backoff retry strategy
- 🐳 **Docker Isolation** - Each task runs in an isolated container
- 📊 **Progress Tracking** - Real-time task execution monitoring
- 💾 **Result Persistence** - Automatic JSON export of execution results

---

## 🚀 Quick Start

### Prerequisites

Before you begin, ensure you have:

1. **Docker** (Required)
   - macOS: [Docker Desktop for Mac](https://docs.docker.com/desktop/install/mac-install/)
   - Windows: [Docker Desktop for Windows](https://docs.docker.com/desktop/install/windows-install/)
   - Linux: [Docker Engine](https://docs.docker.com/engine/install/)

2. **Python 3.12+** (For running the batch scheduler)
   ```bash
   python --version  # Verify Python version
   ```

3. **Conda/Miniconda** (Recommended for environment management)
   - Download: [Miniconda](https://docs.conda.io/en/latest/miniconda.html)

### Installation

#### 1. Clone Repository

```bash
git clone https://github.com/yourusername/ClawMonitor.git
cd ClawMonitor
```

#### 2. Setup Python Environment

**Option A: Using Conda (Recommended)**

```bash
# Create environment from environment.yml
conda env create -f environment.yml

# Activate environment
conda activate base
```

**Option B: Using pip**

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # Linux/macOS
# or venv\Scripts\activate  # Windows

# Install dependencies
pip install -r requirements.txt
```

#### 3. Configure API Keys

Edit `config.yaml` and fill in your API credentials:

```yaml
# ============ API Configuration (Required) ============
api:
  key: "your-api-key-here"           # Your API Key
  url: "https://your-api-url.com"    # API Endpoint URL

# ============ Gateway Configuration ============
gateway:
  auth_token: "your-secure-token"    # Gateway auth token

# ============ Concurrency Configuration ============
max_concurrent: 5                    # Number of concurrent tasks (3-10 recommended)
```

#### 4. Build Docker Image

```bash
# Build OpenClaw image
docker build -t clawmonitor-openclaw:latest -f Dockerfile.openclaw .
```

#### 5. Prepare Task Data

Place your tasks in `decomposed_epoch1.jsonl` with the following format:

```json
{"record_id": "task-001", "instruction": "Complete XX task", "category": "coding", "deomposed_query": "{\"turns\": [{\"thought\": \"Analyze requirements\", \"output\": \"Please help me...\"}, {\"thought\": \"Implement feature\", \"output\": \"Now start...\"}]}"}
```

#### 6. Run Batch Tasks

```bash
python batch_runner.py
```

---

## 📋 Configuration Guide

### Core Parameters

#### 1. API Configuration (Required)

```yaml
api:
  key: ""          # Your API key
  url: ""          # API endpoint URL
```

#### 2. Model Configuration

```yaml
model:
  id: "gpt-51-1113-global"           # Model ID
  name: "gpt-51-1113-global"         # Model name
  context_window: 400000             # Context window size
  max_tokens: 128000                 # Max generation tokens
```

#### 3. Concurrency Control

```yaml
max_concurrent: 5                    # Number of simultaneous containers
                                     # Recommended: 3-10 (depends on system resources)
                                     # Higher value = faster completion, more memory

agent:
  max_concurrent: 4                  # Agent internal concurrency
  subagents_max_concurrent: 8        # Sub-agent concurrency
```

**Performance Example:**
- `max_concurrent: 5` + 10 min/task = **10 minutes** for 5 tasks
- `max_concurrent: 1` + 10 min/task = **50 minutes** for 5 tasks

#### 4. Retry Strategy

```yaml
retry:
  # Prompt request retry configuration
  prompt:
    max_attempts: 3      # Retry up to 3 times on failure
    min_wait: 4          # Wait 4 seconds before first retry
    max_wait: 10         # Max wait 10 seconds (exponential backoff)

  # Health check retry configuration
  health_check:
    max_attempts: 30     # Container startup check retries
    min_wait: 2          # Wait 2 seconds each time
    max_wait: 5          # Max wait 5 seconds
```

**Retry Behavior Example:**
```
Attempt 1: Send immediately
Failure ❌ → Wait 4 seconds

Attempt 2: Retry
Failure ❌ → Wait 8 seconds

Attempt 3: Final attempt
Success ✅ or Failure ❌
```

#### 5. Gateway Configuration

```yaml
gateway:
  port: 18789                        # Gateway port
  mode: "local"                      # Operation mode
  bind: "lan"                        # Network binding (Docker requires lan)
  auth_token: ""                     # Auth token (required)
```

#### 6. Feature Flags

```yaml
features:
  browser_enabled: true              # Enable browser features
  browser_evaluate_enabled: true     # Enable browser evaluation
  browser_attach_only: true          # Attach to existing browser only
  cron_enabled: true                 # Enable cron jobs
  web_enabled: true                  # Enable web features
  canvas_host_enabled: true          # Enable canvas host
```

---

## 🔧 Usage Guide

### Basic Usage

#### 1. Single Batch Execution

```bash
# Use default config (config.yaml) and task file (decomposed_epoch1.jsonl)
python batch_runner.py
```

#### 2. Custom Config and Task Files

Modify `batch_runner.py` initialization:

```python
runner = BatchRunner(
    config_path="custom_config.yaml",    # Custom config
    jsonl_path="my_tasks.jsonl"          # Custom task file
)
runner.run()
```

#### 3. View Execution Logs

```bash
# Real-time container logs
docker logs -f openclaw-task-<record_id>

# List all running containers
docker ps
```

### Advanced Usage

#### 1. Adjust Concurrency

Tune concurrency based on your system resources:

```yaml
# config.yaml
max_concurrent: 10  # More aggressive concurrency (needs more memory)
```

**Resource Estimation:**
- Each container requires ~**2-4GB memory**
- `max_concurrent: 5` ≈ 10-20GB memory
- `max_concurrent: 10` ≈ 20-40GB memory

#### 2. Adjust Retry Strategy

For unstable APIs:

```yaml
retry:
  prompt:
    max_attempts: 5    # Increase retry count
    min_wait: 2        # Faster retry
    max_wait: 20       # Allow longer wait
```

#### 3. Persist Session Data

Edit `docker-compose.yml`:

```yaml
volumes:
  - ./sessions:/root/.openclaw/agents/main/sessions
  - ./workspace:/root/.openclaw/workspace
```

#### 4. Use Docker Compose (Optional)

```bash
# Single container test
docker-compose up -d

# View logs
docker-compose logs -f

# Stop service
docker-compose down
```

---

## 📂 Project Structure

```
ClawMonitor/
├── batch_runner.py              # Batch task scheduler (core)
├── api-server.py                # FastAPI server
├── config.yaml                  # Configuration file (API keys required)
├── requirements.txt             # Python dependencies
├── environment.yml              # Conda environment file
├── decomposed_epoch1.jsonl      # Task data file
├── Dockerfile.openclaw          # Docker image build file
├── docker-compose.yml           # Docker compose file
├── entrypoint.sh                # Container entry script
├── .gitignore                   # Git ignore rules
└── batch_output/                # Task output directory (auto-created)
    ├── task-001.json
    ├── task-002.json
    └── ...
```

---

## 📊 Output Format

Each completed task generates a JSON file in `batch_output/`:

```json
{
  "record_id": "task-001",
  "session_id": "sess-abc123",
  "instruction": "Original task description",
  "category": "coding",
  "total_turns": 3,
  "turns": [
    {
      "turn": 1,
      "thought": "Analyze requirements",
      "prompt": "Please help me analyze this requirement",
      "response": "Agent's response content..."
    },
    {
      "turn": 2,
      "thought": "Implement feature",
      "prompt": "Now start implementation",
      "response": "Agent's implementation content..."
    }
  ],
  "original_task": { ... }
}
```

---

## ⚠️ Important Notes

### Resource Requirements

- **Memory**: At least 16GB recommended, 32GB+ preferred
- **Disk**: ~1-5MB per task, prepare sufficient space
- **CPU**: Multi-core CPU significantly improves concurrent performance

### Troubleshooting

#### 1. Docker Container Fails to Start

```bash
# Check if Docker is running
docker info

# Check if image exists
docker images | grep clawmonitor

# Rebuild image
docker build -t clawmonitor-openclaw:latest -f Dockerfile.openclaw .
```

#### 2. API Request Failures

- Verify `api.key` and `api.url` in `config.yaml`
- Check API quota availability
- Review retry logs, increase `retry.prompt.max_attempts` if needed

#### 3. Out of Memory

```yaml
# Reduce concurrency
max_concurrent: 3  # From 5 to 3
```

Or adjust Docker resource limits:

```yaml
# docker-compose.yml
deploy:
  resources:
    limits:
      memory: 2G  # Limit per container
```

#### 4. Tasks Stuck

```bash
# View container logs
docker logs openclaw-task-<record_id>

# Manually stop stuck container
docker rm -f openclaw-task-<record_id>

# Re-run batch tasks
python batch_runner.py
```

---

## 🛠️ Development Guide

### Modify Retry Logic

Edit retry decorators in `batch_runner.py`:

```python
@retry(
    stop=stop_after_attempt(5),           # Change to 5
    wait=wait_exponential(multiplier=2),  # Adjust backoff multiplier
    ...
)
```

### Add Custom Logging

```python
import logging

logger = logging.getLogger(__name__)
logger.info("Custom log message")
```

### Debug Mode

```bash
# Set environment variable for verbose logging
export LOG_LEVEL=DEBUG
python batch_runner.py
```

---

## 📄 License

MIT License - See [LICENSE](LICENSE) file for details

---

## 🤝 Contributing

Issues and Pull Requests are welcome!

1. Fork this repository
2. Create feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to branch (`git push origin feature/AmazingFeature`)
5. Open Pull Request

---

## 📞 Support

If you have questions:

1. Check [Troubleshooting](#troubleshooting) section
2. Submit an [Issue](https://github.com/yourusername/ClawMonitor/issues)
3. Join our discussion group

---

<div align="center">

**If this project helps you, please give it a ⭐️ Star!**

</div>
