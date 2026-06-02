# Skill: Datasource Engineer — Grafana Provisioning

## Роль
Datasource Engineer настраивает и поддерживает источники данных в Grafana. Ты управляешь YAML-конфигурациями датасорсов, безопасностью подключений и производительностью запросов.

## Файлы
```
provisioning/datasources/
├── prometheus-prod.yaml
├── prometheus-staging.yaml
├── victoriametrics-prod.yaml
├── loki-prod.yaml
├── tempo-prod.yaml
└── alertmanager-prod.yaml
```

## Конфигурации датасорсов

### Prometheus / VictoriaMetrics
```yaml
apiVersion: 1
datasources:
  - name: "Prometheus (prod)"
    type: prometheus
    uid: prometheus-prod-main       # Стабильный uid — обязателен!
    url: http://prometheus.monitoring:9090
    access: proxy
    isDefault: true
    jsonData:
      timeInterval: "15s"           # = scrape_interval
      httpMethod: POST              # POST эффективнее для больших запросов
      queryTimeout: "60s"
      incrementalQuerying: true     # кэш частичных запросов
      exemplarTraceIdDestinations:
        - name: traceID
          datasourceUid: tempo-prod-main
    secureJsonData:
      # Для Basic Auth:
      # basicAuthPassword: "${PROMETHEUS_PASSWORD}"
```

### VictoriaMetrics (расширенные настройки)
```yaml
  - name: "VictoriaMetrics (prod)"
    type: prometheus
    uid: victoriametrics-prod-main
    url: http://vmselect.monitoring:8481/select/0/prometheus
    access: proxy
    jsonData:
      timeInterval: "15s"
      httpMethod: POST
      customQueryParameters: "nocache=1"   # для realtime дашбордов
      prometheusType: Prometheus            # или Mimir/Cortex/Thanos
      prometheusVersion: "2.45.0"
```

### Loki
```yaml
  - name: "Loki (prod)"
    type: loki
    uid: loki-prod-main
    url: http://loki.logging:3100
    access: proxy
    jsonData:
      timeout: 60
      maxLines: 5000
      derivedFields:
        - datasourceUid: tempo-prod-main
          matcherRegex: "traceID=(\\w+)"
          name: traceID
          url: "$${__value.raw}"          # drill-down в Tempo
```

### Tempo (Distributed Tracing)
```yaml
  - name: "Tempo (prod)"
    type: tempo
    uid: tempo-prod-main
    url: http://tempo.tracing:3200
    access: proxy
    jsonData:
      serviceMap:
        datasourceUid: prometheus-prod-main   # для service map
      tracesToLogs:
        datasourceUid: loki-prod-main
        filterByTraceID: true
        filterBySpanID: false
        tags: ["job", "namespace"]
      tracesToMetrics:
        datasourceUid: prometheus-prod-main
      nodeGraph:
        enabled: true
      lokiSearch:
        datasourceUid: loki-prod-main
```

### Alertmanager
```yaml
  - name: "Alertmanager (prod)"
    type: alertmanager
    uid: alertmanager-prod-main
    url: http://alertmanager.monitoring:9093
    access: proxy
    jsonData:
      implementation: prometheus   # или cortex/mimir
      handleGrafanaManagedAlerts: false
```

## Безопасность

### Переменные окружения (обязательно для secrets)
```yaml
secureJsonData:
  basicAuthPassword: "${GRAFANA_DS_PASSWORD}"
  tlsClientCert:     "${GRAFANA_DS_TLS_CERT}"
  tlsClientKey:      "${GRAFANA_DS_TLS_KEY}"
```

Никогда не коммить реальные credentials. Используй:
- `${ENV_VAR}` в YAML → подставляется из env Grafana
- Grafana Vault plugin для динамических секретов
- Kubernetes Secrets → env для Grafana pod

### TLS настройка
```yaml
jsonData:
  tlsAuth:           true
  tlsAuthWithCACert: true
  serverName: "prometheus.example.com"
secureJsonData:
  tlsCACert:    "${PROMETHEUS_CA_CERT}"
  tlsClientCert: "${PROMETHEUS_CLIENT_CERT}"
  tlsClientKey:  "${PROMETHEUS_CLIENT_KEY}"
```

## Мультиенвайронмент стратегия

```yaml
# provisioning/datasources/prometheus-prod.yaml
apiVersion: 1
datasources:
  - name: "Prometheus (prod)"
    uid:  prometheus-prod-main
    url:  "${PROMETHEUS_PROD_URL}"

---
# provisioning/datasources/prometheus-staging.yaml
apiVersion: 1
datasources:
  - name: "Prometheus (staging)"
    uid:  prometheus-staging-main
    url:  "${PROMETHEUS_STAGING_URL}"
```

В дашбордах: переменная `$datasource` типа `datasource` позволяет переключать env.

## Удаление датасорсов
```yaml
apiVersion: 1
deleteDatasources:
  - name: "Old Prometheus"
    orgId: 1
```

## Чеклист датасорса

- [ ] `uid` задан явно и уникален
- [ ] `name` содержит окружение: "Prometheus (prod)"
- [ ] Нет secrets в YAML — только `${ENV_VAR}`
- [ ] `timeInterval` соответствует реальному scrape_interval
- [ ] Для Prometheus `httpMethod: POST`
- [ ] Настроены derived fields (трейсы → логи → метрики)
- [ ] `isDefault: true` только для одного датасорса
