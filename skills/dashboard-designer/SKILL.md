# Skill: Dashboard Designer — Grafana Provisioning

## Роль
Dashboard Designer создаёт визуально понятные, информационно эффективные дашборды. Ты применяешь принципы data visualization к мониторингу: иерархию, цвет, типографику и UX.

## Принципы информационного дизайна

### Иерархия внимания (F-pattern)
Пользователь смотрит сверху-слева направо. Размещай критичные метрики:
```
[🔴 Error Rate]  [🟡 Latency p99]  [🟢 Availability]  [📈 RPS]   ← строка 1: KPI
[─── Request Rate timeseries ─────────] [─ Error Rate timeseries ─] ← строка 2: тренды
[─────────────── Latency histogram / heatmap ─────────────────────] ← строка 3: детали
```

### Цветовая семантика (обязательна в Grafana)
```
🟢 Green  → OK, нормальное состояние
🟡 Yellow → Warning, требует внимания
🔴 Red    → Critical, требует немедленного действия
🔵 Blue   → Информация, нейтральные данные
⚫ Gray   → No Data, отключено, неприменимо
```

Настройка thresholds в fieldConfig:
```json
"thresholds": {
  "mode": "absolute",
  "steps": [
    { "color": "green",  "value": null },
    { "color": "yellow", "value": 0.001 },
    { "color": "red",    "value": 0.01  }
  ]
}
```

## Типы визуализаций — когда что использовать

| Визуализация | Используй когда | Не используй когда |
|---|---|---|
| **timeseries** | Тренды, изменения во времени | Одно текущее значение |
| **stat** | Одна KPI-метрика с текущим значением | Нужна история |
| **gauge** | Прогресс к цели (SLO %) | Метрика без предела |
| **bar gauge** | Сравнение N значений | Много временных точек |
| **table** | Несколько атрибутов по сущностям | Временные ряды |
| **heatmap** | Распределения, гистограммы | Простые тренды |
| **pie/donut** | Доли целого (≤6 сегментов) | Временные ряды |
| **logs panel** | Текстовые логи (Loki) | Числовые метрики |
| **node graph** | Топологии, зависимости | Временные ряды |
| **geomap** | Гео-распределение | Нет координат |

## Дашборд-паттерны

### USE Dashboard (Infrastructure)
```
Row: Server Overview
[CPU %] [Memory %] [Disk I/O %] [Network %]         ← 4 stat, w=6

Row: CPU Detail
[CPU Usage per core — timeseries w=12]
[CPU Load Average — timeseries w=12]

Row: Memory Detail
[Memory usage breakdown — timeseries w=24]

Row: Disk
[Disk read/write IOPS — timeseries w=12]
[Disk latency — timeseries w=12]
```

### RED Dashboard (Application)
```
Row: Service Health
[Error Rate — stat, red threshold] [Latency p99 — stat] [RPS — stat] [Availability — gauge]

Row: Traffic
[Request Rate by status — timeseries, stacked] [Error Rate % — timeseries]

Row: Latency
[p50 / p95 / p99 — timeseries, 3 lines] [Latency heatmap — heatmap]

Row: Saturation
[Thread pool — bar gauge] [Connection pool — bar gauge]
```

### SLO Dashboard
```
[SLO Gauge — большой, w=8] [Error Budget remaining — stat w=8] [Burn Rate — stat w=8]

[Error Budget consumption — timeseries w=24, цветные зоны]

[Availability rolling 30d — timeseries] [Incidents table — table]
```

## Настройка осей и единиц

```json
"fieldConfig": {
  "defaults": {
    "unit": "percentunit",   // 0.01 → "1%"
    "min": 0,
    "max": 1,
    "decimals": 2,
    "mappings": [
      { "type": "value", "options": {
        "0": { "text": "DOWN", "color": "red" },
        "1": { "text": "UP",   "color": "green" }
      }}
    ]
  }
}
```

### Единицы измерения Grafana
```
reqps        — запросов в секунду
ms           — миллисекунды
s            — секунды
bytes        — байты (авто-конверт KB/MB/GB)
percentunit  — 0.0–1.0 → "0%"–"100%"
short        — автоматические суффиксы K/M/B
none         — без единиц
```

## Цветовые схемы для серий

```json
"fieldConfig": {
  "defaults": {
    "color": { "mode": "palette-classic" }   // автоматические цвета
  },
  "overrides": [
    {
      "matcher": { "id": "byName", "options": "p99" },
      "properties": [{ "id": "color", "value": { "mode": "fixed", "fixedColor": "red" } }]
    },
    {
      "matcher": { "id": "byName", "options": "p50" },
      "properties": [{ "id": "color", "value": { "mode": "fixed", "fixedColor": "green" } }]
    }
  ]
}
```

## Визуальные anti-patterns

### Избегай
- **Pie charts для временных рядов** — используй timeseries
- **Более 8 линий на одном графике** — разбивай на несколько панелей или используй переменную
- **Stat панель без единиц** — всегда указывай unit
- **Gauge без min/max** — без ограничений шкала бессмысленна
- **Цвет ради цвета** — цвет несёт смысл (OK/warn/crit), не декоратор
- **Маленькие панели** — минимальный h=4 для stat, h=8 для timeseries
- **Слишком много панелей на экране** — L1 дашборд: ≤12 панелей выше фолда

### Делай
- **Collapsed rows** для детализации — keep L1 clean
- **Tooltips в режиме `shared crosshair`** — `"graphTooltip": 1`
- **Legend с calcs** — `["last", "max", "min"]` дают контекст без запроса
- **Аннотации деплоев** — деплои должны быть видны на графиках
- **Ссылки между дашбордами** — drill-down от L1 к L2/L3

## Именование

```
Панель:     "HTTP Request Rate" (не "rate" или "requests")
Легенда:    "{{method}} {{status}}" (используй метки)
Dashboard:  "Service Overview — {service}" (не "Monitoring")
Row:        "RED Metrics", "USE Metrics", "Kubernetes"
```

## Responsive layout

Grafana адаптирует под ширину:
- Desktop (1536px): 24 колонки в полную ширину
- Mobile (768px): всё в одну колонку
- Ставь stat панели по 4 в ряд (w=6), они хорошо схлопываются
