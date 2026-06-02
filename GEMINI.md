# GEMINI.md — Grafana Provisioning Repository

> Этот файл читается Google Gemini CLI, Gemini Code Assist и агентами
> на базе Gemini API при работе в этом репозитории.

## Репозиторий: назначение

AI-ready GitOps-репозиторий для провиженинга Grafana.
Конфигурации дашбордов, datasources и alerting хранятся как код.
LLM-навыки для каждой роли — в папке `skills/`.

## Система навыков (Skills)

Каждый `SKILL.md` — это специализированный system prompt, описывающий роль,
контекст репозитория, шаблоны и правила для конкретного типа задач.

### Как использовать

Прочитай содержимое нужного файла и применяй как контекст для текущей задачи:

```bash
# Пример: прочитать навык перед работой с алертами
cat skills/alerting-engineer/SKILL.md
```

### Карта навыков

```
┌─────────────────────────────────────────────────────────────┐
│                  ТРЕБОВАНИЯ & АРХИТЕКТУРА                   │
│  business-analyst/SKILL.md   →  KPI, SLO, dashboard brief  │
│  system-analyst/SKILL.md     →  метрики, UIDs, topology     │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────┐
│                  ДАННЫЕ & ЗАПРОСЫ                           │
│  datasource-engineer/SKILL.md →  Prometheus, Loki, Tempo   │
│  prometheus-querying/SKILL.md →  PromQL, MetricsQL, RED/USE │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────┐
│                  ДАШБОРДЫ                                   │
│  dashboard-developer/SKILL.md →  JSON, переменные, layout  │
│  dashboard-designer/SKILL.md  →  UX, цвета, иерархия       │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────┐
│                  НАДЁЖНОСТЬ & АЛЕРТИНГ                      │
│  sre-engineer/SKILL.md        →  SLI/SLO, error budget     │
│  alerting-engineer/SKILL.md   →  rules, routing, runbooks  │
└─────────────────────────────────────────────────────────────┘
```

## Структура репозитория

| Путь | Содержимое |
|---|---|
| `provisioning/dashboards/` | JSON-файлы дашбордов + `_provisioning.yaml` |
| `provisioning/datasources/` | YAML-конфигурации datasources |
| `provisioning/alerting/rules/` | YAML-файлы alert rules |
| `provisioning/alerting/contact-points/` | Каналы: Slack, PagerDuty, email |
| `provisioning/alerting/notification-policies/` | Деревья маршрутизации |
| `docs/runbooks/` | Runbook Markdown по каждому алерту |
| `docs/slo/` | YAML-документы SLO сервисов |
| `skills/` | LLM-навыки по ролям |
| `scripts/` | validate.sh, deploy.sh, export-dashboard.sh |

## Обязательные конвенции

### Именование
```
Datasource UID:  {provider}-{env}-{alias}     # prometheus-prod-main
Dashboard UID:   {team}-{service}-{view}       # payments-api-overview
Alert UID:       {service}-{condition}         # payment-api-high-error-rate
Графана папка:   {Team} / {Service}           # Payments / PaymentAPI
```

### Обязательные поля

**Дашборд** (JSON):
```json
{
  "uid": "payments-api-overview",
  "title": "PaymentAPI Overview",
  "tags": ["team:payments", "service:payment-api"],
  "templating": { "list": [ {"name": "datasource", "type": "datasource"} ] }
}
```

**Alert rule** (YAML):
```yaml
annotations:
  runbook_url: "https://wiki/runbooks/{alert-uid}"   # ОБЯЗАТЕЛЬНО
  summary: "Краткое описание"
labels:
  severity: critical   # critical | warning | info
  team: payments
```

**Datasource** (YAML):
```yaml
apiVersion: 1
datasources:
  - name: Prometheus Prod
    uid: prometheus-prod-main        # ОБЯЗАТЕЛЬНО, стабильный UID
    type: prometheus
    url: ${PROMETHEUS_URL}           # env-переменная, не хардкод
    isDefault: false
```

## Запрещено

- ❌ Credentials/токены в plain text — только `${ENV_VAR}`
- ❌ Дашборд без поля `uid`
- ❌ Алерт без `runbook_url`
- ❌ `for: 0m` в alert rules
- ❌ Удалять `_provisioning.yaml` — он обязателен для Grafana

## Валидация

```bash
bash scripts/validate.sh   # Запускай перед каждым коммитом
```

CI/CD: `.github/workflows/validate.yaml` автоматически валидирует
JSON/YAML при каждом push и pull request.
