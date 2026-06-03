# Grafana Provisioning Skills Catalog

Отдельные skills для LLM-агентов, работающих с AI-ready репозиторием провиженинга Grafana.

## Список skills

| Skill | Файл | Назначение |
|---|---|---|
| Orchestrator | `skills/orchestrator/SKILL.md` | Декомпозиция сложных задач, генерация промптов для каждого шага |
| Business Analyst | `skills/business-analyst/SKILL.md` | Перевод бизнес-требований в KPI, SLO и dashboard briefs |
| System Analyst | `skills/system-analyst/SKILL.md` | Архитектура мониторинга, taxonomy метрик, UID и folder mapping |
| SRE Engineer | `skills/sre-engineer/SKILL.md` | SLI/SLO, error budget, runbooks, on-call и alerting practices |
| Prometheus Querying | `skills/prometheus-querying/SKILL.md` | PromQL/MetricsQL шаблоны, RED/USE, recording rules |
| Dashboard Developer | `skills/dashboard-developer/SKILL.md` | JSON-модель дашбордов Grafana, переменные, layout, export/import |
| Dashboard Designer | `skills/dashboard-designer/SKILL.md` | Визуальная иерархия, thresholds, подходы к визуализациям |
| Alerting Engineer | `skills/alerting-engineer/SKILL.md` | Alert rules, contact points, notification policies, templates |
| Datasource Engineer | `skills/datasource-engineer/SKILL.md` | Datasource provisioning, TLS, security, multi-env setup |

## Когда использовать Orchestrator

Используй `skills/orchestrator/SKILL.md` когда задача затрагивает **3 и более ролей**:
- Настройка полного мониторинга нового сервиса
- Аудит существующего мониторинга
- Миграция дашбордов между окружениями
- Настройка мониторинга для целого продукта/домена

Для точечных задач (написать PromQL, добавить алерт) — используй специализированный skill напрямую.

## Рекомендуемая схема использования

```
[Orchestrator]       → декомпозирует задачу, генерирует промпты
        ↓
[Business Analyst]   → формирует KPI, SLO, аудиторию дашбордов
        ↓
[System Analyst]     → проектирует схему метрик, источники данных, UIDs
        ↓
[Datasource Engineer]→ настраивает datasources в provisioning/datasources/
        ↓
[Prometheus Querying]→ пишет корректные PromQL-запросы
        ↓
[Dashboard Developer]→ собирает JSON дашборда
        ↓
[Dashboard Designer] → улучшает читаемость и UX
        ↓
[SRE Engineer]       → добавляет SLO-панели, error budget
        ↓
[Alerting Engineer]  → создаёт alert rules, contact points, policies
```

## Подключение в IDE

### Cursor
Файл `.cursor/rules/grafana-provisioning.mdc` уже настроен — skills загружаются автоматически для файлов в `provisioning/` и `skills/`.

### Claude Code
Файл `CLAUDE.md` в корне репо читается автоматически при открытии папки.

### Gemini CLI
```bash
cd /path/to/grafana-provisioning
gemini "<задача>"  # GEMINI.md подхватится автоматически
```

### ChatGPT / любой веб-интерфейс
```bash
cat skills/sre-engineer/SKILL.md | pbcopy  # скопировать в буфер
```

### OpenWebUI / LM Studio
Добавить содержимое `SKILL.md` в поле System Prompt перед началом диалога.

## Полная инструкция

См. [`docs/llm-guide.md`](../docs/llm-guide.md) — детальное руководство со сценариями, примерами промптов и советами.
