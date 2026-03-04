name: Cross Monitor
description: Monitor impulsado por IA para tareas de investigación, rastreando múltiples flujos de investigación, experimentos con modelos y procesos de recopilación de datos en tiempo real
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

Monitor impulsado por IA que rastrea tareas de investigación, experimentos de ML, pipelines de datos y procesos de entrenamiento de modelos en múltiples entornos. Proporciona alertas en tiempo real, análisis automatizado y recomendaciones inteligentes de rollback.

## Propósito

Cross Monitor resuelve problemas críticos en flujos de trabajo de investigación:
- Rastrear experimentos de investigación distribuidos en clusters GPU, instancias en la nube y estaciones de trabajo locales
- Detectar anomalías en métricas de entrenamiento de modelos (picos de loss, explosiones de gradiente, fugas de memoria)
- Monitorear data drift en conjuntos de datos de investigación y activar automáticamente re-colección
- Alertar sobre rupturas de procedencia de investigación (logs faltantes, checkpoints corruptos, discrepancias de versión de datasets)
- Automatizar comparación de experimentos e identificación del mejor modelo
- Mantener trails de auditoría de investigación para cumplimiento de reproducibilidad

Casos de uso reales:
1. "Estoy ejecutando 15 experimentos de fine-tuning de LLM en 3 GPUs y necesito saber inmediatamente cuando alguno falle o se estanque"
2. "Monitorea mi pipeline de scraping para bloques de rate limiting y rota proxies automáticamente"
3. "Rastrea notas de lectura de papers y haz cross-reference con mis resultados de experimentos para encontrar conexiones"
4. "Detecta cuando muere el kernel de mi notebook de investigación durante simulaciones largas"
5. "Compara métricas en 50 corridas de hyperparameter tuning e identifica los 3 mejores candidatos"

## Alcance

Cross Monitor ejecuta estos comandos específicos:

### Comandos Principales
```
cross-monitor start --config <config.yaml> --dashboard-port 9090
cross-monitor watch <target> --metrics <metrics.json> --threshold <thresholds.yaml>
cross-monitor alert --channel slack://<webhook> --severity high
cross-monitor compare --experiments <exp1,exp2,...> --metric <name> --window <hours>
cross-monitor rollback --experiment <id> --to-checkpoint <path_or_timestamp>
cross-monitor status --json --include-metrics
cross-monitor export --format csv/json --output <filepath>
```

### Comandos Secundarios
```
cross-monitor daemon --config <config.yaml> --log-level DEBUG
cross-monitor test-connection <target> --type ssh/docker/k8s
cross-monitor validate-config <config.yaml>
cross-monitor list-experiments --status running/failed/completed
cross-monitor sync --from <source_env> --to <target_env>
cross-monitor notify --template <template.md> --to researcher@team.com
cross-monitor cleanup --older-than <days>
```

## Proceso de Trabajo

### 1. Descubrimiento y Validación de Configuración
```bash
# Lee config desde ubicaciones prioritarias:
# 1. ~/.config/cross-monitor/config.yaml (overrides de usuario)
# 2. ./cross-monitor.yaml (config de proyecto)
# 3. Environment: CROSS_MONITOR_CONFIG
cross-monitor validate-config ./research-config.yaml
```

Estructura del archivo de configuración:
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
        threshold: 0.3  # 30% desviación
    actions:
      on_failure: ["slack:#data-alerts", "email:data-team@org.com"]
```

### 2. Conexión con Target y Verificación de Salud
```bash
# Automatizado al inicio, pero verificación manual:
cross-monitor test-connection ssh://gpu0.research.cluster --type ssh
# Esperado: {"status":"healthy","latency_ms":45,"disk_free_gb":234,"gpu_count":4}
```

Métodos de conexión:
- **SSH**: Requiere SSH key en ssh-agent o sudo sin contraseña para inspección de procesos
- **Docker**: Se conecta al socket de Docker, lee logs de contenedores y uso de recursos
- **Kubernetes**: Usa contexto de kubeconfig, consulta métricas de pods via metrics-server
- **Local**: Monitoreo directo de procesos via /proc y psutil

### 3. Loop de Recolección de Métricas
```bash
# Inicia monitoreo con puerto personalizado para UI del dashboard
cross-monitor start --config config.yaml --dashboard-port 9090 --metrics-port 9100
```

El loop de monitoreo ejecuta:
1. Sondea cada target en intervalo configurado (default 30s)
2. Scrapea métricas de TensorBoard, MLflow, Weights & Biases, o endpoints HTTP personalizados
3. Lee métricas de proceso (CPU, memoria, utilización de GPU via nvidia-smi si disponible)
4. Verifica estado del filesystem (uso de disco, tiempos de modificación de archivos, counts de checkpoints)
5. Parsea archivos de log para patrones de error (CUDA out of memory, fallo de convergencia, corrupción de datos)
6. Almacena datos de series temporales en base de datos SQLite local (~/.cache/cross-monitor/metrics.db)

### 4. Detección de Anomalías y Alertas
```bash
# Configura umbrales de detección de anomalías en config.yaml:
# Estadística: media móvil ± 3σ, outliers IQR
# Basada en ML: Isolation Forest (si scikit-learn instalado)
# Basada en patrones: regex en logs, detección de cambios de archivo

cross-monitor alert --channel slack://hooks.slack.com/services/XXX --severity high
# Envía alerta formateada:
# 🚨 [CRÍTICO] llama-finetune-gpu0: Memoria GPU 96% (umbral 95%)
# Experiment: exp_20240115_1423
# Acción: Considera reducir batch size o limpiar cache
```

Canales de alerta:
- Slack: `slack://<webhook_url>`
- Email: `smtp://user:pass@smtp.server.com:587?to=team@org.com`
- Discord: `discord://<webhook_url>`
- Teams: `teams://<webhook_url>`
- Salida CLI: `stdout` (default)
- Archivo: `file:///path/to/alerts.log`

### 5. Comparación de Experimentos
```bash
# Compara corridas de hyperparameter tuning:
cross-monitor compare \
  --experiments exp_001,exp_002,exp_003 \
  --metric validation_loss \
  --window 24h \
  --output comparison.txt

# Salida:
# exp_002: mejor loss final 0.234 (más bajo)
# exp_003: convergencia más rápida (10 epochs para 0.3)
# exp_001: inestable (varianza de loss σ=0.05)
# Recomendación: despliega checkpoint de exp_002 de epoch 45
```

### 6. Ejecución de Rollback
```bash
# Rollback automático cuando se detecta anomalía (si auto_rollback: true):
cross-monitor rollback --experiment llama-finetune-gpu0 --to-checkpoint /experiments/checkpoints/20240115_1423

# O rollback manual:
cross-monitor rollback --experiment exp_001 --to-checkpoint epoch_40 --confirm

# Verificaciones de seguridad antes de rollback:
# ✓ Checkpoint existe y está completo
# ✓ Rollback no perderá >2 horas de entrenamiento (configurable)
# ✓ No hay experimentos dependientes ejecutándose
# ✓ Espacio en disco suficiente para copia de checkpoint
```

### 7. Dashboard y Exportación
```bash
# Dashboard integrado en http://localhost:9090 (cuando se inicia con --dashboard-port)
# Métricas expuestas en /metrics para scraping de Prometheus

# Exporta todas las métricas para análisis offline:
cross-monitor export --format csv --output ./january-metrics.csv --start 2024-01-01 --end 2024-01-31
```

## Reglas de Oro

1. **Nunca auto-rollback sin validación explícita de checkpoint** - Siempre verifica integridad de checkpoint (checksum, metadata .json) antes de rollback
2. **Mantén trail de auditoría** - Cada alerta, rollback y cambio de config debe ser logueado a `~/.cache/cross-monitor/audit.log` con timestamp y contexto de usuario (via SUDO_USER o $USER)
3. **Respeta estado de entrenamiento de investigación** - Nunca interrumpas entrenamiento durante pasos de acumulación de gradiente (detectar via eventos CUDA sync o patrones de log)
4. **Aísla entornos** - Cross Monitor no debe compartir base de datos o config entre proyectos de investigación no relacionados (usa archivos de config separados)
5. **Valores seguros por defecto** - Cooldown de alerta mínimo 1 minuto, auto_rollback default false, retención de checkpoint mínimo 3
6. **Sin borrado de datos** - Rollback solo cambia enlaces simbólicos; nunca borrar checkpoints o logs
7. **Prueba rollback en staging** - Siempre valida procedimiento de rollback en entorno no-productivo primero
8. **Cifra datos sensibles** - Cualquier credencial almacenada (SSH keys, API tokens) debe usar keyring del SO o config cifrada (age/gpg)
9. **Límites de recursos** - Cross Monitor mismo debe usar <5% CPU y <500MB RAM; si se excede, detén nuevos monitores
10. **Fail open** - Si Cross Monitor falla, los procesos de investigación continúan sin afectación (solo monitoreo, sin integración de plano de control)

## Ejemplos

### Ejemplo 1: Monitorear Múltiples Corridas de Entrenamiento en GPU
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

# Inicia monitoreo
cross-monitor start --config config.yaml --dashboard-port 9091

# Simula alerta cuando loss tiene pico:
# Alerta recibida en Slack:
# 🚨 [CRÍTICO] research-llama-7b: train_loss=nan en step 1534
# Revisa logs: ssh gpu-cluster-01 "tail -n 50 /experiments/llama-7b/logs/train.log"
# Intervención manual requerida.

# Compara experimentos después de completarse:
cross-monitor compare \
  --experiments llama-7b-lr1e4,llama-7b-lr2e4 \
  --metric eval_perplexity \
  --window 168h
```

### Ejemplo 2: Monitorear Pipeline de Datos con Detección de Anomalías
```bash
# Monitorea scrapes diarias de datos para drift:
cross-monitor start --config pipeline-monitor.yaml &

# Snippet de config:
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

# Consulta estado:
cross-monitor status --json | jq '.monitors[] | {name: .name, last_check: .last_check, anomalies: .anomaly_count}'
# Salida:
# {
#   "name": "reddit-scraper",
#   "last_check": "2024-01-15T14:30:00Z",
#   "anomaly_count": 0
# }

# Cuando se detecta anomalía (file count cayó 40%):
# Alerta: ⚠️ reddit-scraper: desviación file_count -40% (umbral -25%)
# Acción automática: /opt/scripts/rotate-proxy.sh ejecutado
```

### Ejemplo 3: Rollback de Experimentos Fallidos
```bash
# Lista experimentos para encontrar checkpoint estable:
cross-monitor list-experiments --status completed | grep llama-7b
# exp_20240114_0930: completed, final_loss=0.421, checkpoint=/exp/20240114_0930

# Rollback a checkpoint conocido-bueno:
cross-monitor rollback \
  --experiment llama-7b-gpu3 \
  --to-checkpoint /exp/20240114_0930 \
  --confirm

# Secuencia de rollback:
# 1. Verifica que checkpoint existe: ls -la /exp/20240114_0930
# 2. Revisa dependencias: ps aux | grep train.py (confirma no está en epoch boundary)
# 3. Actualiza symlink: ln -sfn /exp/20240114_0930 /exp/current
# 4. Señaliza proceso de entrenamiento: kill -USR1 <pid> (reload opcional)
# 5. Loguea rollback a audit.log
# Verificación:
cross-monitor status | grep llama-7b-gpu3
# {"experiment":"llama-7b-gpu3","active_checkpoint":"20240114_0930","status":"running"}
```

## Comandos de Rollback

Procedimientos específicos de rollback:

### 1. Rollback de Entrenamiento de Experimentos
```bash
# Revisa checkpoints disponibles antes de rollback:
cross-monitor checkpoints --experiment <id> --format json

# Realiza rollback con dry-run primero:
cross-monitor rollback --experiment exp_001 --to-checkpoint /ckpts/epoch_30 --dry-run
# Salida: Would switch symlink /exp/current → /ckpts/epoch_30

# Ejecuta con confirmación:
cross-monitor rollback --experiment exp_001 --to-checkpoint /ckpts/epoch_30 --confirm

# Verifica:
cross-monitor status --experiment exp_001 | grep checkpoint
# checkpoint: /ckpts/epoch_30
```

Rollback falla si:
- Checkpoint faltan archivos requeridos (pytorch_model.bin, trainer_state.json, config.json)
- Experiment está guardando checkpoint actualmente (archivo lock presente)
- Rollback perdería >24h de entrenamiento (check config `max_rollback_hours`)

### 2. Rollback de Configuración de Pipeline
```bash
# Para monitores de pipeline, revierte config a versión anterior:
cross-monitor config-rollback --monitor data-pipeline --to-version 3 --yes

# Almacenado en: ~/.cache/cross-monitor/config-versions/<monitor-name>/
# Cada config.yaml.N con timestamp y git commit si disponible
```

### 3. Reset de Métricas de Dashboard
```bash
# Si base de datos de métricas corrupta, resetea con backup de checkpoint:
cross-monitor metrics-reset --backup-only --to /backups/metrics-20240115.db

# Reset completo (irreversible):
cross-monitor metrics-reset --confirm --vacuum
```

### 4. Rollback Completo de Monitor
```bash
# Revierta toda la configuración de monitor a último conocido-bueno:
cross-monitor full-rollback \
  --from-backup ~/.cache/cross-monitor/backups/config-20240114.yaml \
  --restart \
  --dry-run

# Verifica con:
cross-monitor validate-config ~/.config/cross-monitor/config.yaml
```

Pasos manuales de rollback (si comando falla):
```bash
# 1. Detén daemon de Cross Monitor:
systemctl --user stop cross-monitor  # si corre como servicio
# o pkill -f cross-monitor

# 2. Restaura config desde backup:
cp ~/.cache/cross-monitor/backups/config-20240114.yaml ~/.config/cross-monitor/config.yaml

# 3. Restaura symlink para experimento actual:
ln -sfn /experiments/backup/checkpoint_v3 /experiments/current

# 4. Reinicia:
cross-monitor start --config ~/.config/cross-monitor/config.yaml --dashboard-port 9090

# 5. Verifica alertas limpias:
cross-monitor status --include-metrics | grep anomalies
# anomalies: 0
```