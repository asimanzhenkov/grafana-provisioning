# Skill: System Analyst — Grafana Provisioning

## Роль
Системный аналитик проектирует архитектуру мониторинга: какие метрики экспортируются, как они стыкуются с Grafana, какова схема данных. Ты работаешь между инфраструктурой и бизнес-требованиями.

## Контекст репозитория
- `provisioning/datasources/` — YAML-конфигурации источников данных
- `provisioning/dashboards/` — JSON-дашборды + `_provisioning.yaml`
- `provisioning/alerting/` — правила алертинга
- `skills/` — роли для LLM-агентов

## Архитектура источников данных

### Типы датасорсов в репозитории
```
Prometheus / VictoriaMetrics  → time-series метрики
Loki                          → логи (LogQL)
Tempo / Jaeger                → трейсы
Alertmanager                  → статус алертов
PostgreSQL / MySQL            → бизнес-данные
Elasticsearch / OpenSearch    → события и логи
```

### Схема провиженинга датасорсов
```yaml
# provisioning/datasources/{name}.yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    uid: prometheus-main          # uid должен быть стабильным!
    url: http://prometheus:9090
    access: proxy
    isDefault: true
    jsonData:
      timeInterval: "15s"         # scrape interval
      httpMethod: POST
      exemplarTraceIdDestinations:
        - name: traceID
          datasourceUid: tempo-main
```

### UID соглашения
UID датасорса должен быть детерминированным (не auto-generated):
```
{type}-{env}-{region}
prometheus-prod-eu-west
loki-staging-eu-west
victoriametrics-prod-main
```

## Проектирование схемы метрик

### Label taxonomy (обязательные лейблы)
```
job          — имя scrape job (node-exporter, app-name)
instance     — host:port
env          — production | staging | dev
cluster      — имя кластера k8s
namespace    — k8s namespace
service      — имя сервиса
```

### Naming conventions (Prometheus)
```
# Counter
{namespace}_{subsystem}_{name}_total
http_server_requests_total

# Gauge
{namespace}_{subsystem}_{name}
node_memory_available_bytes

# Histogram
{namespace}_{subsystem}_{name}_bucket / _count / _sum
http_request_duration_seconds_bucket
```

## Схема репозитория

### Структура дашборда (мета-поля)
```json
{
  "uid": "service-overview-v1",     // стабильный uid
  "title": "Service Overview",
  "tags": ["team:platform", "tier:l1", "domain:api"],
  "schemaVersion": 39,
  "version": 1,
  "__inputs": [],                    // переменные для provisioning
  "templating": {
    "list": [
      { "name": "datasource", "type": "datasource", "query": "prometheus" },
      { "name": "env",        "type": "custom",     "options": ["prod","staging"] },
      { "name": "service",    "type": "query",      "datasource": "$datasource" }
    ]
  }
}
```

## Системные решения

### Folder mapping
```yaml
# provisioning/dashboards/_provisioning.yaml
apiVersion: 1
providers:
  - name: platform-dashboards
    type: file
    disableDeletion: true
    updateIntervalSeconds: 30
    allowUiUpdates: false           # запрет UI-правок (IaC only)
    options:
      path: /var/lib/grafana/dashboards
      foldersFromFilesStructure: true  # автоматические папки из директорий
```

### Импорт/экспорт дашбордов
- Экспорт: `scripts/export-dashboard.sh {uid}` → сохраняет в `provisioning/dashboards/`
- Импорт: автоматически при деплое через CI
- Всегда удаляй `id` из JSON перед коммитом (только `uid` стабилен)

## Что НЕ делать
- Не используй `uid: null` — всегда задавай явный стабильный uid
- Не храни credentials в YAML — используй Grafana Vault plugin или env vars
- Не создавай датасорс "Default" без явного имени среды
