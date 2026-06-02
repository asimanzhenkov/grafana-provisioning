# grafana-provisioning

AI-ready репозиторий для управления провиженингом Grafana через код (GitOps).

## Структура репозитория

```
├── provisioning/           # Конфигурации Grafana (YAML / JSON)
│   ├── dashboards/         # Dashboard JSON + провиженинг-конфиги
│   ├── datasources/        # Источники данных (Prometheus, Loki, etc.)
│   ├── alerting/           # Правила алертинга, каналы, политики
│   └── plugins/            # Плагины Grafana
├── skills/                 # LLM-навыки для работы с репозиторием
│   ├── business-analyst/
│   ├── system-analyst/
│   ├── sre-engineer/
│   ├── prometheus-querying/
│   ├── dashboard-developer/
│   ├── dashboard-designer/
│   ├── alerting-engineer/
│   └── datasource-engineer/
├── scripts/                # Утилиты: деплой, линтинг, экспорт
├── tests/                  # Тесты конфигураций и дашбордов
├── examples/               # Примеры готовых дашбордов и датасорсов
├── docs/                   # Документация
└── .github/workflows/      # CI/CD пайплайны
```

## Быстрый старт

```bash
git clone https://github.com/asimanzhenkov/grafana-provisioning.git
cd grafana-provisioning
./scripts/deploy.sh --env local
./scripts/validate.sh
```

## Работа с LLM

Каждый навык в `skills/` описывает контекст и инструкции для конкретной роли.
Подключай нужный `SKILL.md` как системный промпт или через `.cursor/rules`.

| Навык | Роль |
|---|---|
| `skills/business-analyst/SKILL.md` | Бизнес-аналитик: требования, KPI, SLO |
| `skills/system-analyst/SKILL.md` | Системный аналитик: архитектура, схемы |
| `skills/sre-engineer/SKILL.md` | SRE: error budget, алертинг, runbooks |
| `skills/prometheus-querying/SKILL.md` | PromQL: RED/USE, histograms, recording rules |
| `skills/dashboard-developer/SKILL.md` | Разработчик дашбордов: JSON, панели, переменные |
| `skills/dashboard-designer/SKILL.md` | Дизайнер дашбордов: UX, цвет, типографика |
| `skills/alerting-engineer/SKILL.md` | Алертинг: rules, contact points, routing |
| `skills/datasource-engineer/SKILL.md` | Датасорсы: Prometheus, Loki, Tempo, TLS |

## Соглашения

- Все дашборды хранятся в JSON через Grafana Dashboard API
- Датасорсы — YAML через Grafana provisioning API
- Алерты — YAML (Grafana Alerting v2 / Mimir Ruler)
- UID дашборда: стабильный, явный, никогда `null`
- `"editable": false` — правки только через Git PR
- Secrets: только `${ENV_VAR}` синтаксис, никаких plaintext

## Переменные окружения

| Переменная | Описание |
|---|---|
| `GRAFANA_URL` | URL инстанса Grafana |
| `GRAFANA_API_KEY` | Service account token |
| `GRAFANA_ORG_ID` | ID организации (default: 1) |
