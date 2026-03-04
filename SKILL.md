---
name: Cross Monitor
description: AI-powered monitor for research tasks, tracking multiple research streams, model experiments, and data collection processes in real-time
version: 1.2.0
author: SMOUJBOT
tags: [research, ai, automation, monitoring, experiments]
dependencies:
  - python>=3.9
  - prometheus-client>=0.19.0
  - grafana-api>=1.0.0
  - openai>=1.0.0
  - pandas>=2.0.0
  - watchdog>=3.0.0
  - websockets>=12.0
required_tools: [docker, kubectl, curl, jq]
platforms: [linux, macos]
license: MIT
---

# Cross Monitor

AI-powered monitor that tracks research tasks, ML experiments, data pipelines, and model training processes across multiple environments. Provides real-time alerts, automated analysis, and intelligent rollback recommendations.

## Purpose

Cross Monitor solves critical research workflow problems:
- Track distributed research experiments across GPU clusters, cloud instances, and local workstations
- Detect anomalies in model training metrics (loss spikes, gradient explosions, memory leaks)
- Monitor data drift in research datasets and automatically trigger re-collection
- Alert on research provenance breaks (missing logs, corrupted checkpoints, dataset version mismatches)
- Automate experiment comparison and best model identification
- Maintain research audit trails for reproducibility compliance

Real use cases:
1. "I'm running 15 LLM fine-tuning experiments across 3 GPUs and need to know immediately when any fails or plateaus"
2. "Monitor my data scraping pipeline for rate limiting blocks and automatically rotate proxies"
3. "Track paper reading notes and cross-reference with my experiment results to find connections"
4. "Detect when my research notebook kernel dies during long simulations"
5. "Compare metrics across 50 hyperparameter tuning runs and identify top 3 candidates"

## Scope

Cross Monitor executes these specific commands:

### Primary Commands
```
cross-monitor start --config <config.yaml> --dashboard-port 9090
cross-monitor watch <target> --metrics <metrics.json> --threshold <thresholds.yaml>
cross-monitor alert --channel slack://<webhook> --severity high
cross-monitor compare --experiments <exp1,exp2,...> --metric <name> --window <hours>
cross-monitor rollback --experiment <id> --to-checkpoint <path_or_timestamp>
cross-monitor status --json --include-metrics
cross-monitor export --format csv/json --output <filepath>
```

### Secondary Commands
```
cross-monitor daemon --config <config.yaml> --log-level DEBUG
cross-monitor test-connection <target> --type ssh/docker/k8s
cross-monitor validate-config <config.yaml>
cross-monitor list-experiments --status running/failed/completed
cross-monitor sync --from <source_env> --to <target_env>
cross-monitor notify --template <template.md> --to researcher@team.com
cross-monitor cleanup --older-than <days>
```

## Work Process

### 1. Configuration Discovery & Validation
```bash
# Read config from priority locations:
# 1. ~/.config/cross-monitor/config.yaml (user overrides)
# 2. ./cross-monitor.yaml (project config)
# 3. Environment: CROSS_MONITOR_CONFIG
cross-monitor validate-config ./research-config.yaml
```

The configuration file structure:
```yaml
monitors:
  - name: "llama-finetune-gpu0"
    type: "training"
    target: "ssh://gpu0.research.cluster"
    metrics_endpoint: "http://localhost:6006"
    check_interval: 30s
    alert_on:
      - metric: "loss"
        condition: "spike > 2.0"
        cooldown: 5m
      - metric: "gpu_memory"
        condition: "> 0.95"
        severity: critical
    rollback_strategy:
      checkpoint_path: "/experiments/checkpoints"
      keep_last: 5
      auto_rollback: false
  
  - name: "data-pipeline-prod"
    type: "pipeline"
    target: "k8s://data-pipeline"
    watch_paths:
      - "/data/raw"
      - "/data/processed"
    anomaly_detection:
      rows_ingested:
        baseline: 10000
        threshold: 0.3  # 30% deviation
    actions:
      on_failure: ["slack:#data-alerts", "email:data-team@org.com"]
```

### 2. Target Connection & Health Check
```bash
# Automated at start, but manual check:
cross-monitor test-connection ssh://gpu0.research.cluster --type ssh
# Expected: {"status":"healthy","latency_ms":45,"disk_free_gb":234,"gpu_count":4}
```

Connection methods:
- **SSH**: Requires SSH key in ssh-agent or passwordless sudo for process inspection
- **Docker**: Connects to Docker socket, reads container logs and resource usage
- **Kubernetes**: Uses kubeconfig context, queries pod metrics via metrics-server
- **Local**: Direct process monitoring via /proc and psutil

### 3. Metric Collection Loop
```bash
# Start monitoring with custom port for dashboard UI
cross-monitor start --config config.yaml --dashboard-port 9090 --metrics-port 9100
```

The monitoring loop executes:
1. Poll each target at configured interval (default 30s)
2. Scrape metrics from TensorBoard, MLflow, Weights & Biases, or custom HTTP endpoints
3. Read process metrics (CPU, memory, GPU utilization via nvidia-smi if available)
4. Check filesystem state (disk usage, file modification times, checkpoint counts)
5. Parse log files for error patterns (CUDA out of memory, convergence failure, data corruption)
6. Store time-series data in local SQLite database (~/.cache/cross-monitor/metrics.db)

### 4. Anomaly Detection & Alerting
```bash
# Configure anomaly detection thresholds in config.yaml:
# Statistical: moving average ± 3σ, IQR outliers
# ML-based: Isolation Forest (if scikit-learn installed)
# Pattern-based: regex in logs, file change detection

cross-monitor alert --channel slack://hooks.slack.com/services/XXX --severity high
# Sends formatted alert:
# 🚨 [CRITICAL] llama-finetune-gpu0: GPU memory 96% (threshold 95%)
# Experiment: exp_20240115_1423
# Action: Consider reducing batch size or clearing cache
```

Alert channels:
- Slack: `slack://<webhook_url>`
- Email: `smtp://user:pass@smtp.server.com:587?to=team@org.com`
- Discord: `discord://<webhook_url>`
- Teams: `teams://<webhook_url>`
- CLI output: `stdout` (default)
- File: `file:///path/to/alerts.log`

### 5. Experiment Comparison
```bash
# Compare hyperparameter tuning runs:
cross-monitor compare \
  --experiments exp_001,exp_002,exp_003 \
  --metric validation_loss \
  --window 24h \
  --output comparison.txt

# Output:
# exp_002: best final loss 0.234 (lowest)
# exp_003: fastest convergence (10 epochs to 0.3)
# exp_001: unstable (loss variance σ=0.05)
# Recommendation: deploy exp_002 checkpoint from epoch 45
```

### 6. Rollback Execution
```bash
# Automatic rollback when anomaly detected (if auto_rollback: true):
cross-monitor rollback --experiment llama-finetune-gpu0 --to-checkpoint /experiments/checkpoints/20240115_1423

# Or manual rollback:
cross-monitor rollback --experiment exp_001 --to-checkpoint epoch_40 --confirm

# Safety checks before rollback:
# ✓ Checkpoint exists and is complete
# ✓ Rollback will not lose >2 hours of training (configurable)
# ✓ No dependent experiments running
# ✓ Disk space sufficient for checkpoint copy
```

### 7. Dashboard & Export
```bash
# Built-in dashboard at http://localhost:9090 (when started with --dashboard-port)
# Metrics exposed at /metrics for Prometheus scraping

# Export all metrics for offline analysis:
cross-monitor export --format csv --output ./january-metrics.csv --start 2024-01-01 --end 2024-01-31
```

## Golden Rules

1. **Never auto-rollback without explicit checkpoint validation** - Always verify checkpoint integrity (checksum, .json metadata) before rollback
2. **Maintain audit trail** - Every alert, rollback, and config change must be logged to `~/.cache/cross-monitor/audit.log` with timestamp and user context (via SUDO_USER or $USER)
3. **Respect research training state** - Never interrupt training during gradient accumulation steps (detect via CUDA sync events or log patterns)
4. **Isolate environments** - Cross Monitor must not share database or config between unrelated research projects (use separate config files)
5. **Safe defaults** - Alert cooldown minimum 1 minute, auto_rollback default false, checkpoint retention minimum 3
6. **No data deletion** - Rollback only switches symbolic links; never delete checkpoints or logs
7. **Test rollback in staging** - Always validate rollback procedure on non-production environment first
8. **Encrypt sensitive data** - Any stored credentials (SSH keys, API tokens) must use OS keyring or encrypted config (age/gpg)
9. **Resource limits** - Cross Monitor itself must use <5% CPU and <500MB RAM; if exceeded, stop new monitors
10. **Fail open** - If Cross Monitor crashes, research processes continue unaffected (monitoring only, no control plane integration)

## Examples

### Example 1: Monitor Multiple GPU Training Runs
```bash
# Setup config: ~/.config/cross-monitor/config.yaml
cat > config.yaml << 'EOF'
monitors:
  - name: "research-llama-7b"
    type: "training"
    target: "ssh://gpu-cluster-01"
    metrics_endpoint: "http://localhost:6006"
    check_interval: 60s
    alert_on:
      - metric: "train_loss"
        condition: "nan OR spike > 5.0"
        severity: critical
      - metric: "gpu_util"
        condition: "< 10% for 5m"
        severity: warning
        cooldown: 10m
    rollback_strategy:
      checkpoint_dir: "/mnt/experiments/llama-7b/checkpoints"
      keep_last: 10
      auto_rollback: false
EOF

# Start monitoring
cross-monitor start --config config.yaml --dashboard-port 9091

# Simulate alert when loss spikes:
# Alert received in Slack:
# 🚨 [CRITICAL] research-llama-7b: train_loss=nan at step 1534
# Check logs: ssh gpu-cluster-01 "tail -n 50 / experiments/llama-7b/logs/train.log"
# Manual intervention required.

# Compare experiments after completion:
cross-monitor compare \
  --experiments llama-7b-lr1e4,llama-7b-lr2e4 \
  --metric eval_perplexity \
  --window 168h
```

### Example 2: Monitor Data Pipeline with Anomaly Detection
```bash
# Monitor daily data scrapes for drift:
cross-monitor start --config pipeline-monitor.yaml &

# Config snippet:
#   - name: "reddit-scraper"
#     type: "pipeline"
#     target: "docker://scraper-prod"
#     watch_paths: ["/data/reddit/jsonl"]
#     anomaly_detection:
#       file_count_daily:
#         baseline: 50000
#         threshold: 0.25
#     actions:
#       on_anomaly: ["slack:#data-alerts", "exec:/opt/scripts/rotate-proxy.sh"]

# Query status:
cross-monitor status --json | jq '.monitors[] | {name: .name, last_check: .last_check, anomalies: .anomaly_count}'
# Output:
# {
#   "name": "reddit-scraper",
#   "last_check": "2024-01-15T14:30:00Z",
#   "anomaly_count": 0
# }

# When anomaly detected (file count dropped 40%):
# Alert: ⚠️ reddit-scraper: file_count deviation -40% (threshold -25%)
# Automatic action: /opt/scripts/rotate-proxy.sh executed
```

### Example 3: Rollback Failed Experiment
```bash
# List experiments to find stable checkpoint:
cross-monitor list-experiments --status completed | grep llama-7b
# exp_20240114_0930: completed, final_loss=0.421, checkpoint=/exp/20240114_0930

# Rollback to known-good checkpoint:
cross-monitor rollback \
  --experiment llama-7b-gpu3 \
  --to-checkpoint /exp/20240114_0930 \
  --confirm

# Rollback sequence:
# 1. Verify checkpoint exists: ls -la /exp/20240114_0930
# 2. Check dependencies: ps aux | grep train.py (confirm not at epoch boundary)
# 3. Update symlink: ln -sfn /exp/20240114_0930 /exp/current
# 4. Signal training process: kill -USR1 <pid> (optional reload)
# 5. Log rollback to audit.log
# Verification:
cross-monitor status | grep llama-7b-gpu3
# {"experiment":"llama-7b-gpu3","active_checkpoint":"20240114_0930","status":"running"}
```

## Rollback Commands

Specific rollback procedures:

### 1. Training Experiment Rollback
```bash
# Check available checkpoints before rollback:
cross-monitor checkpoints --experiment <id> --format json

# Perform rollback with dry-run first:
cross-monitor rollback --experiment exp_001 --to-checkpoint /ckpts/epoch_30 --dry-run
# Output: Would switch symlink /exp/current → /ckpts/epoch_30

# Execute with confirmation:
cross-monitor rollback --experiment exp_001 --to-checkpoint /ckpts/epoch_30 --confirm

# Verify:
cross-monitor status --experiment exp_001 | grep checkpoint
# checkpoint: /ckpts/epoch_30
```

Rollback fails if:
- Checkpoint missing required files (pytorch_model.bin, trainer_state.json, config.json)
- Experiment currently saving checkpoint (lock file present)
- Rollback would lose >24h training (check config `max_rollback_hours`)

### 2. Pipeline Configuration Rollback
```bash
# For pipeline monitors, rollback config to previous version:
cross-monitor config-rollback --monitor data-pipeline --to-version 3 --yes

# Stored in: ~/.cache/cross-monitor/config-versions/<monitor-name>/
# Each config.yaml.N with timestamp and git commit if available
```

### 3. Dashboard Metrics Reset
```bash
# If metrics database corrupted, reset with checkpoint backup:
cross-monitor metrics-reset --backup-only --to /backups/metrics-20240115.db

# Full reset (irreversible):
cross-monitor metrics-reset --confirm --vacuum
```

### 4. Complete Monitor Rollback
```bash
# Revert entire monitor configuration to last known-good:
cross-monitor full-rollback \
  --from-backup ~/.cache/cross-monitor/backups/config-20240114.yaml \
  --restart \
  --dry-run

# Verify with:
cross-monitor validate-config ~/.config/cross-monitor/config.yaml
```

Manual rollback steps (if command fails):
```bash
# 1. Stop Cross Monitor daemon:
systemctl --user stop cross-monitor  # if running as service
# or pkill -f cross-monitor

# 2. Restore config from backup:
cp ~/.cache/cross-monitor/backups/config-20240114.yaml ~/.config/cross-monitor/config.yaml

# 3. Restore symlink for current experiment:
ln -sfn /experiments/backup/checkpoint_v3 /experiments/current

# 4. Restart:
cross-monitor start --config ~/.config/cross-monitor/config.yaml --dashboard-port 9090

# 5. Verify alerts cleared:
cross-monitor status --include-metrics | grep anomalies
# anomalies: 0
```

```