# Skill: Dashboard Developer — Grafana Provisioning

## Роль
Dashboard Developer создаёт и поддерживает JSON-конфигурации дашбордов в репозитории. Ты работаешь с Grafana API, JSON-схемами панелей и системой провиженинга.

## Контекст репозитория
- `provisioning/dashboards/` — JSON файлы дашбордов
- `provisioning/dashboards/_provisioning.yaml` — конфиг провиженинга
- `scripts/export-dashboard.sh` — экспорт из Grafana
- `scripts/validate.sh` — валидация JSON

## Структура дашборда

### Скелет dashboard JSON
```json
{
  "uid": "service-overview-v1",
  "title": "Service Overview",
  "description": "RED метрики: rate, errors, latency для сервисов",
  "tags": ["team:platform", "tier:l1", "env:$env"],
  "schemaVersion": 39,
  "version": 1,
  "time": { "from": "now-1h", "to": "now" },
  "refresh": "30s",
  "editable": false,
  "fiscalYearStartMonth": 0,
  "graphTooltip": 1,
  "panels": [],
  "templating": { "list": [] },
  "annotations": { "list": [] },
  "links": []
}
```

## Типы панелей и их применение

### timeseries — временные ряды (основной тип)
```json
{
  "type": "timeseries",
  "title": "Request Rate",
  "gridPos": { "x": 0, "y": 0, "w": 12, "h": 8 },
  "datasource": { "type": "prometheus", "uid": "$datasource" },
  "targets": [
    {
      "refId": "A",
      "expr": "sum(rate(http_requests_total{job=\"$service\"}[$__rate_interval])) by (status_code)",
      "legendFormat": "{{status_code}}"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "reqps",
      "thresholds": {
        "mode": "absolute",
        "steps": [
          { "color": "green", "value": null },
          { "color": "yellow", "value": 1000 },
          { "color": "red",    "value": 5000 }
        ]
      }
    }
  },
  "options": {
    "tooltip": { "mode": "multi", "sort": "desc" },
    "legend":  { "displayMode": "table", "placement": "bottom", "calcs": ["last", "max"] }
  }
}
```

### stat — одиночное значение (KPI)
```json
{
  "type": "stat",
  "title": "Error Rate",
  "gridPos": { "x": 0, "y": 0, "w": 4, "h": 4 },
  "options": {
    "reduceOptions": { "calcs": ["lastNotNull"] },
    "colorMode": "background",
    "graphMode": "area",
    "orientation": "auto"
  },
  "fieldConfig": {
    "defaults": {
      "unit": "percentunit",
      "thresholds": {
        "mode": "absolute",
        "steps": [
          { "color": "green",  "value": null },
          { "color": "yellow", "value": 0.001 },
          { "color": "red",    "value": 0.01 }
        ]
      }
    }
  }
}
```

### gauge — прогресс-бар (SLO compliance)
```json
{
  "type": "gauge",
  "title": "SLO Availability",
  "fieldConfig": {
    "defaults": {
      "unit": "percentunit",
      "min": 0.99,
      "max": 1,
      "thresholds": {
        "mode": "absolute",
        "steps": [
          { "color": "red",    "value": null },
          { "color": "yellow", "value": 0.995 },
          { "color": "green",  "value": 0.999 }
        ]
      }
    }
  },
  "options": {
    "reduceOptions": { "calcs": ["lastNotNull"] },
    "showThresholdLabels": true,
    "showThresholdMarkers": true
  }
}
```

### table — таблица с трансформациями
```json
{
  "type": "table",
  "title": "Top Services by Error Rate",
  "transformations": [
    { "id": "sortBy", "options": { "fields": [{ "displayName": "Error Rate", "desc": true }] } },
    { "id": "limit",  "options": { "limitNumber": 10 } }
  ],
  "fieldConfig": {
    "overrides": [
      {
        "matcher": { "id": "byName", "options": "Error Rate" },
        "properties": [
          { "id": "unit", "value": "percentunit" },
          { "id": "custom.displayMode", "value": "color-background" },
          { "id": "thresholds", "value": {
            "mode": "absolute",
            "steps": [{ "color": "green" }, { "color": "red", "value": 0.01 }]
          }}
        ]
      }
    ]
  }
}
```

## Переменные шаблона

```json
{
  "templating": {
    "list": [
      {
        "name": "datasource",
        "type": "datasource",
        "query": "prometheus",
        "hide": 0
      },
      {
        "name": "env",
        "type": "custom",
        "query": "production,staging,dev",
        "current": { "text": "production", "value": "production" },
        "hide": 0
      },
      {
        "name": "service",
        "type": "query",
        "datasource": { "type": "prometheus", "uid": "$datasource" },
        "query": "label_values(http_requests_total{env=\"$env\"}, job)",
        "refresh": 2,
        "sort": 1,
        "multi": true,
        "includeAll": true
      }
    ]
  }
}
```

## Grid Layout система

Grafana использует 24-колоночную сетку (ширина 1536px → 1 col = 64px):
```
Полная строка:    w=24, h=8
Половина:         w=12, h=8
Треть:            w=8,  h=8
Четверть:         w=6,  h=4 (KPI stat панели)
Широкая таблица:  w=24, h=10
```

Типичная раскладка L1 дашборда:
```
Row: "Overview"
[stat w=6] [stat w=6] [stat w=6] [stat w=6]   y=0, h=4
[timeseries Rate w=12] [timeseries Errors w=12] y=4, h=8
[timeseries Latency p99 w=24]                   y=12, h=8

Row: "Details" (collapsed=true)
[table top-endpoints w=24]                      y=21, h=10
```

## Аннотации (деплои, инциденты)

```json
{
  "annotations": {
    "list": [
      {
        "name": "Deployments",
        "type": "dashboard",
        "datasource": { "type": "prometheus", "uid": "$datasource" },
        "expr": "changes(kube_deployment_status_observed_generation{namespace=\"$namespace\",deployment=\"$service\"}[2m]) > 0",
        "step": "60s",
        "titleFormat": "Deploy: {{deployment}}",
        "iconColor": "blue",
        "enable": true
      }
    ]
  }
}
```

## Скрипты разработки

```bash
# Экспортировать дашборд из Grafana в репозиторий
./scripts/export-dashboard.sh --uid service-overview-v1 --output provisioning/dashboards/

# Импортировать дашборд в Grafana
./scripts/import-dashboard.sh --file provisioning/dashboards/service-overview.json

# Валидировать все JSON файлы
./scripts/validate.sh --path provisioning/dashboards/

# Найти дашборды без uid
./scripts/lint-dashboards.sh --check uid
```

## Что НЕ делать
- Не коммить `"id": 123` — id меняется при импорте, только uid стабилен
- Не хардкодить datasource name — используй `{ "type": "prometheus", "uid": "$datasource" }`
- Не ставь `"editable": true` — правки только через Git PR
- Не используй `"refresh": "5s"` — минимум `"30s"` для production
