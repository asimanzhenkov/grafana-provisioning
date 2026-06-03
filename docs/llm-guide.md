# Руководство по работе с LLM в репозитории grafana-provisioning

## Обзор

Репозиторий `asimanzhenkov/grafana-provisioning` — AI-ready GitOps-репозиторий для провиженинга Grafana. Все конфигурации (дашборды, datasources, alerting) хранятся как код в `provisioning/`, а папка `skills/` содержит специализированные system prompts для восьми ролей LLM-агентов.

---

## Часть 1. Как работать с LLM вручную

### Базовый принцип: один вопрос — один skill

Перед каждым запросом к LLM определи, какая роль нужна, и загрузи соответствующий `SKILL.md` как system prompt или как первое сообщение в контексте.

```
┌─────────────────────────────────────────────────────┐
│  Что нужно сделать?   →   Какой SKILL.md загрузить? │
└─────────────────────────────────────────────────────┘
         │
         ▼
  [Загрузить SKILL.md] → [Сформулировать задачу] → [Получить результат]
                                                           │
                                                           ▼
                                              [Сохранить в provisioning/]
```

### Сценарий 1: Новый дашборд с нуля

**Шаг 1. Требования** — загрузи `skills/business-analyst/SKILL.md`

```
System prompt: <содержимое business-analyst/SKILL.md>

Запрос: Сервис checkout обрабатывает платежи.
        Аудитория: команда payments и дежурный SRE.
        Определи KPI и структуру дашборда.
```

Сохрани результат: список метрик, аудитория, уровни L0/L1/L2.

---

**Шаг 2. Схема метрик** — загрузи `skills/system-analyst/SKILL.md`

```
System prompt: <содержимое system-analyst/SKILL.md>

Запрос: Для checkout-сервиса на Go, метрики через Prometheus.
        Job label: "checkout-api", namespace: "payments".
        Спроектируй UID дашборда, folder, список метрик.
```

Получи: UID вида `payments-checkout-overview`, folder `Payments / Checkout`.

---

**Шаг 3. PromQL-запросы** — загрузи `skills/prometheus-querying/SKILL.md`

```
System prompt: <содержимое prometheus-querying/SKILL.md>

Запрос: Напиши PromQL для:
        - error rate по HTTP (RED-паттерн)
        - p50/p95/p99 latency (histogram_quantile)
        - throughput (rps)
        Datasource UID: prometheus-prod-main
        Job: checkout-api
```

---

**Шаг 4. Сборка JSON** — загрузи `skills/dashboard-developer/SKILL.md`

```
System prompt: <содержимое dashboard-developer/SKILL.md>

Запрос: Собери Grafana dashboard JSON.
        UID: payments-checkout-overview
        Title: "Checkout Overview"
        Tags: ["team:payments", "service:checkout"]
        Panels: [вставь PromQL из шага 3]
        Переменная $datasource типа datasource.
```

Сохрани файл: `provisioning/dashboards/payments/checkout-overview.json`

---

**Шаг 5. UX-проверка** — загрузи `skills/dashboard-designer/SKILL.md`

```
System prompt: <содержимое dashboard-designer/SKILL.md>

Запрос: Проверь этот Grafana dashboard JSON на:
        - визуальную иерархию (L0 → L1 → L2)
        - правильные типы визуализаций
        - thresholds и единицы измерения
        [вставь JSON]
```

---

**Шаг 6. SLO-панели** — загрузи `skills/sre-engineer/SKILL.md`

```
System prompt: <содержимое sre-engineer/SKILL.md>

Запрос: Добавь в dashboard JSON SLO-панели:
        - SLO gauge (availability target 99.9%)
        - Error budget burn rate (1h и 6h window)
        Job: checkout-api
```

---

**Шаг 7. Alert rules** — загрузи `skills/alerting-engineer/SKILL.md`

```
System prompt: <содержимое alerting-engineer/SKILL.md>

Запрос: Создай alerting YAML для checkout-api:
        - HighErrorRate (>1%, severity: critical)
        - HighLatency p99 (>500ms, severity: warning)
        - SLO burn rate fast (14.4x, 1h window)
        Datasource UID: prometheus-prod-main
        Runbook base URL: https://wiki.company.com/runbooks/
```

Сохрани: `provisioning/alerting/rules/checkout-alerts.yaml`

---

### Сценарий 2: Добавить datasource

```
System prompt: <содержимое datasource-engineer/SKILL.md>

Запрос: Добавь datasource для Prometheus в staging окружении.
        URL через env-переменную PROMETHEUS_STAGING_URL.
        TLS: CA cert из переменной PROMETHEUS_CA_CERT.
        UID: prometheus-staging-main
```

Сохрани: `provisioning/datasources/staging-prometheus.yaml`

---

### Сценарий 3: Разбор существующего дашборда

```
System prompt: <содержимое dashboard-developer/SKILL.md>

Запрос: Объясни этот Grafana dashboard JSON.
        Что делает каждая панель? Какие переменные используются?
        Как оптимизировать запросы?
        [вставь JSON файла]
```

---

### Быстрая шпаргалка: задача → skill

| Задача | Skill |
|--------|-------|
| Какие метрики мониторить для сервиса X? | `business-analyst` |
| Спроектировать схему метрик, UIDs, папки | `system-analyst` |
| Написать PromQL / MetricsQL запрос | `prometheus-querying` |
| Собрать или изменить Grafana dashboard JSON | `dashboard-developer` |
| Улучшить читаемость, цвета, thresholds | `dashboard-designer` |
| Настроить SLO, error budget, on-call панели | `sre-engineer` |
| Создать alert rules, настроить роутинг | `alerting-engineer` |
| Подключить новый datasource (Prometheus/Loki/Tempo) | `datasource-engineer` |
| Сложная задача на 3+ ролей | `orchestrator` |

---

## Часть 2. Интеграция с AI-инструментами

### Claude Code (рекомендуется)

`CLAUDE.md` в корне репо автоматически читается при открытии папки. Никаких дополнительных настроек не нужно — просто открой репозиторий через Claude Code.

```bash
# Открыть репозиторий через Claude Code CLI
claude /path/to/grafana-provisioning
```

Затем работай напрямую:

```
Создай dashboard для сервиса payment-api с RED-метриками
```

Claude прочитает CLAUDE.md, поймёт контекст репо и сам определит нужные skills.

---

### Cursor IDE

Файл `.cursor/rules/grafana-provisioning.mdc` с `alwaysApply: true` уже настроен. При работе с файлами в `provisioning/` или `skills/` Cursor автоматически применяет правила репозитория.

---

### Gemini CLI

```bash
# Gemini CLI читает GEMINI.md автоматически из текущей директории
cd /path/to/grafana-provisioning
gemini "Добавь SLO-панели к dashboard payments-checkout-overview.json"
```

---

### ChatGPT / любой веб-интерфейс

```bash
# Скопировать нужный skill в буфер
cat skills/dashboard-developer/SKILL.md | pbcopy  # macOS
cat skills/dashboard-developer/SKILL.md | xclip   # Linux

# Вставить как первое сообщение в чат, затем задать вопрос
```

---

### OpenWebUI / LM Studio / локальные модели

В поле **System Prompt** вставь содержимое нужного `SKILL.md` перед началом диалога. Для лучшего результата используй модели с контекстом ≥32k токенов (llama3, mistral-nemo, qwen2.5).

---

## Часть 3. Оркестратор

### Зачем нужен оркестратор

При сложных задачах (создать полный мониторинг для нового сервиса, провести аудит существующих дашбордов) нужно последовательно применять несколько skills. Оркестратор — это мета-агент, который:

1. Принимает высокоуровневую задачу
2. Декомпозирует её на подзадачи
3. Определяет порядок выполнения и нужные skills
4. Генерирует готовые промпты для каждого шага
5. Отслеживает результаты и зависимости между шагами

### Пример использования оркестратора

```
System prompt: <содержимое skills/orchestrator/SKILL.md>

Задача: Настрой полный мониторинг для нового сервиса order-service.
        - Go-сервис, метрики через Prometheus
        - Job label: "order-service", namespace: "orders"
        - SLO: availability 99.5%, latency p99 < 300ms
        - Уведомления: Slack #alerts-orders + PagerDuty для critical
        - Команда: orders, wiki: https://wiki.company.com/runbooks/
```

Оркестратор вернёт декомпозицию задачи по шагам с указанием skill и готовым промптом для каждого.

---

### Схема работы оркестратора

```
Пользователь → [ORCHESTRATOR skill]
                      │
          ┌───────────┼───────────────┐
          ▼           ▼               ▼
  [business-analyst] [system-analyst] [datasource-engineer]
          │                           │
          └─────────┬─────────────────┘
                    ▼
          [prometheus-querying]
                    │
          ┌─────────┴──────────┐
          ▼                    ▼
  [dashboard-developer] [sre-engineer]
          │                    │
          ▼                    ▼
  [dashboard-designer] [alerting-engineer]
          │                    │
          └─────────┬──────────┘
                    ▼
             [Готовые файлы]
         provisioning/ + docs/runbooks/
```

---

## Часть 4. Валидация и деплой

### Перед коммитом

```bash
# Валидация всех JSON/YAML файлов
bash scripts/validate.sh

# Проверяет:
# - Валидность JSON дашбордов
# - Наличие обязательных полей (uid, title, tags)
# - Валидность YAML datasources и alerting rules
# - Наличие runbook_url в каждом алерте
```

### CI/CD pipeline

```
push / pull_request
      │
      ▼
.github/workflows/validate.yaml
      │
      ├── JSON lint (jq)
      ├── YAML lint (yamllint)
      ├── Проверка обязательных полей
      └── (на main) → deploy.yaml → Grafana API
```

### Экспорт дашборда из Grafana в репо

```bash
# Экспортировать существующий дашборд по UID
bash scripts/export-dashboard.sh payments-checkout-overview

# Файл сохранится в provisioning/dashboards/
```

---

## Часть 5. Советы по работе с LLM

### Давай максимум контекста

```
❌ "Создай дашборд для сервиса"

✅ "Создай Grafana dashboard JSON для сервиса checkout-api.
    Job label: checkout-api, namespace: payments.
    Datasource UID: prometheus-prod-main.
    Нужны панели: error rate, latency p50/p95/p99, rps, active connections.
    Dashboard UID: payments-checkout-overview"
```

### Итерируй по одному файлу

Не пытайся получить полный мониторинг одним запросом. Лучший подход:
1. Получи черновик → сохрани
2. Проверь → уточни
3. Улучши через следующий skill

### Включай результаты предыдущего шага

```
# Шаг 4 (dashboard-developer):
"Вот PromQL запросы из шага 3: [...]
 Собери из них Grafana JSON с этими панелями."

# Шаг 5 (dashboard-designer):
"Вот готовый JSON: [...]
 Улучши визуальную иерархию и добавь thresholds."
```

### Проси объяснять PromQL

```
"Напиши PromQL для burn rate и объясни каждую часть запроса —
 что считает каждая функция и почему именно эти значения."
```

Это помогает находить ошибки и понимать, что именно мониторится.

---

## Приложение: структура репозитория

```
grafana-provisioning/
├── AGENTS.md                    ← Auto-context для OpenAI Codex, Devin, Aider
├── CLAUDE.md                    ← Auto-context для Claude Code
├── GEMINI.md                    ← Auto-context для Gemini CLI / Code Assist
├── README.md
├── .cursor/rules/               ← Cursor IDE rules (alwaysApply)
├── .github/workflows/
│   ├── validate.yaml            ← Lint при каждом push/PR
│   └── deploy.yaml              ← Deploy на main
├── provisioning/
│   ├── dashboards/
│   │   ├── _provisioning.yaml   ← Grafana provisioning config (не удалять!)
│   │   └── {team}/{service}.json
│   ├── datasources/
│   │   └── {env}-{name}.yaml
│   └── alerting/
│       ├── rules/{service}-alerts.yaml
│       ├── contact-points/
│       └── notification-policies/
├── skills/
│   ├── SKILLS-INDEX.md
│   ├── orchestrator/SKILL.md    ← Мета-агент для сложных задач
│   ├── business-analyst/SKILL.md
│   ├── system-analyst/SKILL.md
│   ├── sre-engineer/SKILL.md
│   ├── prometheus-querying/SKILL.md
│   ├── dashboard-developer/SKILL.md
│   ├── dashboard-designer/SKILL.md
│   ├── alerting-engineer/SKILL.md
│   └── datasource-engineer/SKILL.md
├── docs/
│   ├── llm-guide.md             ← Этот файл
│   ├── runbooks/{alert-uid}.md
│   └── slo/{service}.yaml
└── scripts/
    ├── validate.sh
    ├── deploy.sh
    └── export-dashboard.sh
```
