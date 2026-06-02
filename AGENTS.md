# AGENTS.md — Grafana Provisioning Repository

> Этот файл автоматически читается агентами OpenAI (Codex), Devin, Aider
> и другими инструментами, поддерживающими `AGENTS.md` convention.

## Назначение репозитория

AI-ready репозиторий для GitOps-провиженинга Grafana: дашборды, datasources, alerting.
Все конфигурации хранятся как код в `provisioning/`, LLM-навыки — в `skills/`.

## Структура

```
provisioning/
  dashboards/          ← JSON дашборды + _provisioning.yaml
  datasources/         ← YAML-конфигурации datasources
  alerting/
    rules/             ← Alert rules (YAML)
    contact-points/    ← Каналы уведомлений
    notification-policies/ ← Политики маршрутизации
skills/                ← LLM-навыки (system prompts по ролям)
docs/
  runbooks/            ← Runbook по каждому алерту
  slo/                 ← SLO-документы сервисов
scripts/               ← validate.sh, deploy.sh, export-dashboard.sh
```

## Skills — загружай перед началом работы

Перед выполнением задачи прочитай соответствующий `SKILL.md` как контекст:

| Задача | Skill-файл |
|---|---|
| Анализ бизнес-требований, KPI | `skills/business-analyst/SKILL.md` |
| Архитектура мониторинга, taxonomy | `skills/system-analyst/SKILL.md` |
| SLO, error budget, runbooks | `skills/sre-engineer/SKILL.md` |
| Написание PromQL-запросов | `skills/prometheus-querying/SKILL.md` |
| Сборка JSON-дашбордов Grafana | `skills/dashboard-developer/SKILL.md` |
| Визуальный дизайн дашбордов | `skills/dashboard-designer/SKILL.md` |
| Alert rules, contact points | `skills/alerting-engineer/SKILL.md` |
| Datasource provisioning | `skills/datasource-engineer/SKILL.md` |

Полный индекс: [`skills/SKILLS-INDEX.md`](skills/SKILLS-INDEX.md)

## Правила для агента

### Обязательно
- Читай нужный `SKILL.md` **перед** изменением файлов в `provisioning/`
- Дашборды сохраняй в `provisioning/dashboards/{team}/{service}.json`
- Datasources — в `provisioning/datasources/{env}-{name}.yaml`
- Alert rules — в `provisioning/alerting/rules/{service}-alerts.yaml`
- Каждый алерт **обязан** иметь `runbook_url` в annotations
- UID дашбордов формируй по шаблону: `{team}-{service}-{type}` (например `payments-api-overview`)
- Datasource UID: `{provider}-{env}-{alias}` (например `prometheus-prod-main`)

### Никогда
- Не хардкоди credentials или токены — используй `${ENV_VAR}` или Grafana Vault
- Не создавай дашборды без поля `uid` — это сломает GitOps-синхронизацию
- Не ставь `for: 0m` в алертах — минимум `2m` для стабилизации
- Не удаляй `_provisioning.yaml` — он связывает дашборды с папками Grafana

### Порядок работы для нового фича-цикла
1. Прочитай `skills/system-analyst/SKILL.md` → спроектируй схему метрик
2. Прочитай `skills/prometheus-querying/SKILL.md` → напиши PromQL
3. Прочитай `skills/dashboard-developer/SKILL.md` → собери JSON дашборда
4. Прочитай `skills/alerting-engineer/SKILL.md` → добавь alerting rules
5. Добавь runbook в `docs/runbooks/`

## Конвенции именования

```yaml
# Datasource UID
{provider}-{env}-{alias}        # prometheus-prod-main, loki-staging-apps

# Dashboard UID
{team}-{service}-{view}         # payments-api-overview, infra-nodes-detail

# Alert rule UID
{service}-{condition}           # payment-api-high-error-rate

# Folder в Grafana
{Team} / {Service}              # Payments / PaymentAPI
```

## Валидация перед коммитом

```bash
bash scripts/validate.sh        # JSON/YAML lint + проверка обязательных полей
```
