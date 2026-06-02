# Skill: SRE Engineer — Grafana Provisioning

## Роль
SRE отвечает за надёжность сервисов через мониторинг, алертинг и SLO/SLI. Ты используешь дашборды для on-call работы, постмортемов и capacity planning.

## Контекст репозитория
- `provisioning/alerting/rules/` — Grafana alerting rules (YAML)
- `provisioning/alerting/contact-points/` — каналы уведомлений
- `provisioning/alerting/notification-policies/` — политики маршрутизации
- `docs/runbooks/` — runbook по каждому алерту
- `docs/slo/` — SLO-документы

## SLO / SLI / Error Budget

### Шаблон SLO
```yaml
# docs/slo/{service}.yaml
service: payment-api
team: payments
sli:
  availability:
    good_events: 'sum(rate(http_requests_total{status!~"5.."}[5m]))'
    total_events: 'sum(rate(http_requests_total[5m]))'
  latency:
    threshold_ms: 500
    percentile: 99
    query: 'histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))'
slo:
  availability_target: 99.9
  latency_compliance: 95.0
error_budget:
  window: 30d
  burn_rate_alert_fast: 14.4   # 1h window, 2% budget
  burn_rate_alert_slow: 6.0    # 6h window, 5% budget
```

### Формулы Error Budget
```promql
# Availability SLI
(
  sum(rate(http_requests_total{status!~"5.."}[5m]))
  /
  sum(rate(http_requests_total[5m]))
)

# Error Budget Burn Rate (6h / 30d)
(
  1 - (
    sum(rate(http_requests_total{status!~"5.."}[6h]))
    /
    sum(rate(http_requests_total[6h]))
  )
) / (1 - 0.999)
```

## Алертинг

### Структура alerting rule
```yaml
# provisioning/alerting/rules/{service}-alerts.yaml
apiVersion: 1
groups:
  - orgId: 1
    name: payment-api
    folder: SRE Alerts
    interval: 1m
    rules:
      - uid: payment-api-high-error-rate
        title: "PaymentAPI HighErrorRate"
        condition: C
        data:
          - refId: A
            relativeTimeRange: { from: 300, to: 0 }
            datasourceUid: prometheus-prod-main
            model:
              expr: |
                sum(rate(http_requests_total{job="payment-api",status=~"5.."}[5m]))
                /
                sum(rate(http_requests_total{job="payment-api"}[5m])) > 0.01
              instant: true
          - refId: C
            datasourceUid: __expr__
            model:
              type: classic_conditions
              conditions:
                - evaluator: { type: gt, params: [0] }
                  operator:  { type: and }
                  query:     { params: [A] }
                  reducer:   { type: last }
        noDataState: NoData
        execErrState: Error
        for: 5m
        annotations:
          summary:     "Высокий error rate на {{ $labels.job }}"
          description: "Error rate: {{ $value | humanizePercentage }}"
          runbook_url:  "https://wiki/runbooks/payment-api-high-error-rate"
        labels:
          severity: critical
          team:     payments
```

### Severity уровни
| Severity | Реакция | Примеры |
|---|---|---|
| `critical` | PagerDuty немедленно | error rate > 1%, SLO burn rate 14x |
| `warning`  | Slack, рабочее время | latency p99 > 1s, error rate > 0.1% |
| `info`     | Только Grafana | необычный трафик, высокое CPU |

### Multi-window Multi-burn-rate Alerting
```yaml
# Fast burn: 1h window, burn > 14.4x (2% error budget consumed in 1h)
- alert: SLOBurnRateFast
  expr: |
    (
      error_budget_burn_rate:1h > 14.4
      and
      error_budget_burn_rate:5m > 14.4
    )
  for: 2m
  labels: { severity: critical }

# Slow burn: 6h window, burn > 6x (5% budget in 6h)
- alert: SLOBurnRateSlow
  expr: |
    (
      error_budget_burn_rate:6h > 6
      and
      error_budget_burn_rate:30m > 6
    )
  for: 15m
  labels: { severity: warning }
```

## On-call дашборды

### Обязательные панели для L1 дашборда
1. **SLO Gauge** — текущий error budget (%)
2. **Error Rate** — rate(errors[5m]) + burn rate
3. **Latency p50/p95/p99** — histogram_quantile
4. **Throughput** — rps per service
5. **Active Alerts** — Alertmanager панель
6. **Recent Deployments** — аннотации из CI/CD

## Runbook шаблон
```markdown
# Runbook: {AlertName}

## Описание
[Что происходит и почему это важно]

## Диагностика
1. [Первый шаг — ссылка на дашборд]
2. [Команды для проверки]
   ```bash
   kubectl top pods -n {namespace}
   kubectl logs -n {namespace} -l app={service} --tail=100
   ```

## Возможные причины
- [ ] [Причина 1]
- [ ] [Причина 2]

## Исправление
[Пошаговые действия]

## Эскалация
[Когда и кому эскалировать]
```

## Что НЕ делать
- Не ставь alert без runbook — `runbook_url` обязателен
- Не используй `for: 0m` — минимум 2-5 минут для стабилизации
- Не создавай алерты на каждую метрику — только на симптомы, а не причины
