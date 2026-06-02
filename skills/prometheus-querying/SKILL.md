# Skill: Prometheus Querying — PromQL Expert

## Роль
Специалист по написанию PromQL-запросов для Grafana дашбордов. Пишешь корректные, производительные и читаемые запросы для Prometheus и VictoriaMetrics.

## Контекст репозитория
Запросы используются в JSON-панелях дашбордов в `provisioning/dashboards/`. Datatsource uid берётся из `provisioning/datasources/`.

## PromQL фундаментальные концепции

### Типы метрик
```
Counter   — монотонно возрастает. rate() / increase() для скорости изменения
Gauge     — текущее значение. Прямые сравнения, min/max/avg
Histogram — распределение. histogram_quantile() для перцентилей
Summary   — квантили на стороне клиента. Нельзя агрегировать между инстансами
```

### Обязательные правила
1. **Всегда добавляй `job` лейбл** в selectors для изоляции метрики
2. **Для counter всегда используй `rate()` или `increase()`** — никогда голый counter
3. **Для `histogram_quantile` — `by (le)`** обязателен
4. **Range vector** (`[5m]`) должен быть ≥ 2× scrape_interval

## Шаблоны запросов

### RED Metrics (Rate / Errors / Duration)

```promql
# Rate — запросы в секунду
sum(rate(http_requests_total{job="$service", env="$env"}[$__rate_interval])) by (status_code)

# Error Rate — доля ошибок
sum(rate(http_requests_total{job="$service", status=~"5.."}[$__rate_interval]))
/
sum(rate(http_requests_total{job="$service"}[$__rate_interval]))

# Duration — перцентили latency
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket{
    job="$service", env="$env"
  }[$__rate_interval])) by (le)
)

# Несколько перцентилей в одном запросе (Grafana transform)
histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket{job="$service"}[$__rate_interval])) by (le))
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{job="$service"}[$__rate_interval])) by (le))
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{job="$service"}[$__rate_interval])) by (le))
```

### USE Metrics (Utilization / Saturation / Errors)

```promql
# CPU Utilization (node-exporter)
1 - avg(rate(node_cpu_seconds_total{mode="idle", instance="$instance"}[$__rate_interval]))

# Memory Utilization
1 - (
  node_memory_MemAvailable_bytes{instance="$instance"}
  / node_memory_MemTotal_bytes{instance="$instance"}
)

# Disk I/O Saturation
rate(node_disk_io_time_seconds_total{device!~"dm-.*", instance="$instance"}[$__rate_interval])

# Network saturation
rate(node_network_receive_bytes_total{device!~"lo", instance="$instance"}[$__rate_interval])
```

### Kubernetes метрики

```promql
# Pod CPU usage
sum(rate(container_cpu_usage_seconds_total{
  namespace="$namespace",
  pod=~"$service.*",
  container!=""
}[$__rate_interval])) by (pod)

# Pod Memory
sum(container_memory_working_set_bytes{
  namespace="$namespace",
  pod=~"$service.*",
  container!=""
}) by (pod)

# Pod restart count
increase(kube_pod_container_status_restarts_total{
  namespace="$namespace"
}[1h])

# Deployment availability
kube_deployment_status_replicas_available{namespace="$namespace", deployment="$service"}
/
kube_deployment_spec_replicas{namespace="$namespace", deployment="$service"}
```

### Агрегации и оконные функции

```promql
# Скользящее среднее (сглаживание)
avg_over_time(rate(http_requests_total[5m])[30m:5m])

# Сравнение с прошлой неделей
rate(http_requests_total[5m]) offset 1w

# Top 5 потребителей памяти
topk(5, container_memory_working_set_bytes{container!=""})

# Прогноз (linear regression)
predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[1h], 4*3600) < 0
```

## Grafana-специфичные переменные

```promql
# $__rate_interval — автоматический interval (рекомендуется вместо [5m])
rate(metric[$__rate_interval])

# $__interval — dashboard refresh interval
rate(metric[$__interval])

# $__range — полный временной диапазон дашборда
increase(metric[$__range])

# Переменные шаблона
rate(http_requests_total{job="$service", env="$env", namespace="$namespace"}[$__rate_interval])
```

## Recording Rules (оптимизация)

Для тяжёлых запросов создавай recording rules:

```yaml
# В Grafana Alerting или Prometheus rules
groups:
  - name: slo_recording
    interval: 30s
    rules:
      - record: job:http_requests:rate5m
        expr: sum(rate(http_requests_total[5m])) by (job, env, status_code)

      - record: job:http_errors:rate5m
        expr: sum(rate(http_requests_total{status=~"5.."}[5m])) by (job, env)

      - record: job:http_request_duration_seconds:p99
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (job, env, le)
          )
```

## VictoriaMetrics расширения

```promql
# MetricsQL: default() для заполнения пустых значений
rate(http_requests_total[5m]) default 0

# keep_last_value() для стабильных gauge
keep_last_value(up{job="$service"}, 10m)

# aggr_over_time() — агрегация по времени
aggr_over_time("last", http_requests_total[5m])
```

## Антипаттерны PromQL

```promql
# НЕПРАВИЛЬНО — не используй rate на gauge
rate(node_memory_MemAvailable_bytes[5m])  -- gauge, не counter!

# ПРАВИЛЬНО
node_memory_MemAvailable_bytes

# НЕПРАВИЛЬНО — отсутствует by (le)
histogram_quantile(0.99, rate(duration_bucket[5m]))

# ПРАВИЛЬНО
histogram_quantile(0.99, sum(rate(duration_bucket[5m])) by (le))

# НЕПРАВИЛЬНО — слишком короткий range vector
rate(requests_total[30s])  -- если scrape_interval = 15s, минимум [30s], лучше [1m]

# ПРАВИЛЬНО
rate(requests_total[$__rate_interval])
```
