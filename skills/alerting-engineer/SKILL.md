# Skill: Alerting Engineer — Grafana Alerting

## Роль
Alerting Engineer проектирует и реализует систему уведомлений: правила, каналы, политики маршрутизации и шаблоны сообщений.

## Структура файлов
```
provisioning/alerting/
├── rules/                      # Alert rules (YAML)
│   ├── {service}-alerts.yaml
│   └── recording-rules.yaml
├── contact-points/             # Каналы уведомлений
│   ├── slack.yaml
│   ├── pagerduty.yaml
│   └── email.yaml
└── notification-policies/     # Политики маршрутизации
    └── policies.yaml
```

## Contact Points

### Slack
```yaml
# provisioning/alerting/contact-points/slack.yaml
apiVersion: 1
contactPoints:
  - orgId: 1
    name: slack-platform
    receivers:
      - uid: slack-platform-receiver
        type: slack
        settings:
          url: "${SLACK_WEBHOOK_URL}"
          channel: "#alerts-platform"
          username: "Grafana Alerting"
          icon_emoji: ":grafana:"
          title: |
            {{ template "slack.title" . }}
          text: |
            {{ template "slack.body" . }}
        disableResolveMessage: false
```

### PagerDuty
```yaml
  - orgId: 1
    name: pagerduty-critical
    receivers:
      - uid: pagerduty-critical-receiver
        type: pagerduty
        settings:
          integrationKey: "${PAGERDUTY_INTEGRATION_KEY}"
          severity: "{{ .CommonLabels.severity }}"
          class:    "{{ .CommonLabels.alertname }}"
          group:    "{{ .CommonLabels.team }}"
          component: "{{ .CommonLabels.job }}"
          summary:  "{{ .CommonAnnotations.summary }}"
          details:
            runbook: "{{ .CommonAnnotations.runbook_url }}"
```

## Notification Policies

```yaml
# provisioning/alerting/notification-policies/policies.yaml
apiVersion: 1
policies:
  - orgId: 1
    receiver: slack-platform          # default fallback
    group_by: [alertname, cluster, namespace]
    group_wait:      30s
    group_interval:  5m
    repeat_interval: 4h
    routes:
      # Critical → PagerDuty
      - receiver: pagerduty-critical
        matchers:
          - severity = critical
        group_wait:      10s
        repeat_interval: 30m
        continue: false

      # SLO burn rate → отдельный канал
      - receiver: slack-slo-alerts
        matchers:
          - alertname =~ "SLOBurnRate.*"
        group_by: [service, env]
        continue: true

      # Team routing по лейблу
      - receiver: slack-payments
        matchers:
          - team = payments
        continue: false
```

## Alert Rule Templates

### Стандартный шаблон правила
```yaml
# provisioning/alerting/rules/template.yaml
apiVersion: 1
groups:
  - orgId: 1
    name: {service}-alerts
    folder: "{Team} Alerts"
    interval: 1m
    rules:
      - uid: {service}-{alertname}           # уникальный uid
        title: "{Service} {AlertName}"
        condition: threshold

        data:
          - refId: A
            relativeTimeRange: { from: 300, to: 0 }
            datasourceUid: prometheus-prod-main
            model:
              expr: '{promql_query}'
              instant: true

          - refId: threshold
            datasourceUid: __expr__
            model:
              type: threshold
              conditions:
                - evaluator: { type: gt, params: [{value}] }
                  operator:  { type: and }
                  query:     { params: [A] }
                  reducer:   { type: last }

        noDataState:  NoData    # NoData | Alerting | OK
        execErrState: Error     # Error | Alerting | OK
        for: 5m                 # pending duration

        annotations:
          summary:     "{краткое описание}"
          description: "Значение: {{ $value | printf \"%.2f\" }}"
          runbook_url:  "https://wiki/runbooks/{alert-name}"
          dashboard_url: "https://grafana/d/{dashboard-uid}?var-service={{ $labels.job }}"

        labels:
          severity:  critical     # critical | warning | info
          team:      {team}
          env:       "{{ $labels.env }}"
```

### Alert для Error Budget Burn Rate
```yaml
      - uid: slo-burn-rate-fast-{service}
        title: "SLO BurnRate Fast — {Service}"
        for: 2m
        annotations:
          summary: "Быстрое сжигание error budget: {{ $value | printf \"%.1f\" }}x"
          description: |
            Сервис {{ $labels.job }} сжигает error budget со скоростью {{ $value }}x.
            При текущей скорости бюджет закончится через {{ div 1 $value | printf "%.1f" }} периодов.
          runbook_url: "https://wiki/runbooks/slo-burn-rate"
        labels:
          severity: critical
          slo_type:  availability

        data:
          - refId: A
            model:
              expr: |
                (
                  1 - (
                    sum(rate(http_requests_total{job="$service", status!~"5.."}[1h]))
                    / sum(rate(http_requests_total{job="$service"}[1h]))
                  )
                ) / (1 - 0.999) > 14.4
```

## Шаблоны Slack сообщений

```yaml
# provisioning/alerting/message-templates.yaml
apiVersion: 1
templates:
  - orgId: 1
    name: slack.title
    template: |
      {{ if eq .Status "firing" }}🔥{{ else }}✅{{ end }}
      [{{ .Status | toUpper }}] {{ .CommonLabels.alertname }}
      ({{ .Alerts | len }} alert{{ if gt (len .Alerts) 1 }}s{{ end }})

  - orgId: 1
    name: slack.body
    template: |
      *Severity:* {{ .CommonLabels.severity }}
      *Team:* {{ .CommonLabels.team }}
      *Environment:* {{ .CommonLabels.env }}

      {{ range .Alerts }}
      *Alert:* {{ .Labels.alertname }}
      *Summary:* {{ .Annotations.summary }}
      *Value:* {{ .ValueString }}
      {{ if .Annotations.runbook_url }}*Runbook:* {{ .Annotations.runbook_url }}{{ end }}
      {{ if .Annotations.dashboard_url }}*Dashboard:* {{ .Annotations.dashboard_url }}{{ end }}
      *Started:* {{ .StartsAt | since }}
      ---
      {{ end }}
```

## Чеклист алерта

Перед созданием алерта проверь:
- [ ] Алерт отвечает на симптом (не причину)
- [ ] Есть `runbook_url` с реальным runbook
- [ ] `for:` ≥ 2m (без "flapping")
- [ ] `noDataState: NoData` (не Alerting)
- [ ] Лейблы: `severity`, `team`, `env`
- [ ] Аннотация `summary` понятна без контекста
- [ ] Значение `$value` отформатировано в human-readable вид
- [ ] Есть drill-down ссылка на дашборд
