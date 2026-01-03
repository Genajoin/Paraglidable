# Маппинг полей: xcontest → Paraglidable Training Format

## Важно: IGC файлы недоступны

Для этих данных **нет IGC файлов**, поэтому мы используем данные из xcontest API:
- `score` = `league.route.points`
- `plaf` заменён на `league.route.avgSpeed` (средняя скорость по маршруту)

---

## Требуемый формат (из TRAINING_PROCESS.md)

### flights_by_cell_day_spot.pkl
```python
flights_by_cell_day_spot[cell_index][day_index] = {
    spot_id: [
        ('yyyy-mm-dd hh:mm:ss', (score, None, takeoff_alt, lat, lon)),
        ...
    ]
}
```

**Поля записи (SPOTS модель):**
| Позиция | Поле | Тип | Описание | Источник xcontest |
|---------|------|-----|----------|-------------------|
| 0 | datetime | str | `'yyyy-mm-dd hh:mm:ss'` | ✅ `pointStart.time` |
| 1 | score | float | Очки/балл полёта | ✅ `league.route.points` |
| 2 | alt | None | Всегда `None` для SPOTS | ✅ `None` |
| 3 | takeoff_alt | float | Высота точки взлёта (м) | ❌ **НЕТ** - нужен SRTM или spots.pkl |
| 4 | lat | float | Широта точки взлёта | ✅ `takeoff.link` |
| 5 | lon | float | Долгота точки взлёта | ✅ `takeoff.link` |

**Изменения для SPOTS (без IGC):**
- `plaf` заменён на `avg_speed` из `league.route.avgSpeed`
- `score` берётся из `league.route.points`

---

## Наши поля из xcontest API

| Поле xcontest | Маппинг на Paraglidable | Статус |
|---------------|------------------------|--------|
| `pointStart.time` | `datetime` | ✅ |
| `takeoff.link` | `lat`, `lon` | ✅ (парсинг URL) |
| `league.route.points` | `score` | ✅ |
| `league.route.avgSpeed` | `plaf` / `avg_speed` | ✅ (замена потолка) |
| `league.route.distance` | `route_distance` | ✅ (дополнительно) |
| `league.route.type` | `route_type` | ✅ (дополнительно) |
| `takeoff.id` | `spot_id` | ✅ |
| `takeoff.name` | `spot_name` | ✅ |
| `stats.duration` | `duration_sec` | ✅ |
| `pilot.id` | `pilot_id` | ✅ |
| `glider.name` | `glider` | ✅ |
| **takeoff_alt** | `takeoff_alt` | ❌ **НЕТ** (нужен SRTM или spots.pkl) |
| **mountainess** | `mountainess` | ❌ **НЕТ** (можно вычислить из SRTM) |
| **cell_index** | `cell_index` | ❌ **НЕТ** (нужен sorted_cells.pkl) |
| **day_index** | `day_index` | ❌ **НЕТ** (нужен meteo_days.pkl) |

---

## Что не хватает и как получить

### 1. **takeoff_alt** (высота точки взлёта над уровнем моря)
- **Статус:** Не приходит в xcontest API
- **Решение:**
  - Вариант A: Извлечь из существующего `neural_network/bin/data/spots.pkl` по spot_id
  - Вариант B: Извлечь из SRTM elevation tiles по координатам
- **Приоритет:** **СРЕДНИЙ** - для SPOTS модели не критичен (высота spots известна)

### 2. **mountainess** (гористость 0.0-1.0)
- **Статус:** Не приходит в xcontest API
- **Решение:**
  - Вариант A: Взять из существующего `neural_network/bin/data/mountainess_by_cell_alt.pkl`
  - Вариант B: Вычислить из SRTM данных (разница высот в ячейке)
- **Приоритет:** НИЗКИЙ для SPOTS модели

### 3. **cell_index** (индекс ячейки в sorted_cells.pkl)
- **Статус:** Не вычисляется
- **Решение:** Найти индекс ячейки (cell_lat, cell_lon) в `neural_network/bin/data/sorted_cells.pkl`
- **Приоритет:** **ВЫСОКИЙ** - необходим для структуры данных

### 4. **day_index** (индекс дня относительно meteo_days.pkl)
- **Статус:** Не вычисляется
- **Решение:** Сопоставить дату с индексом из `neural_network/bin/data/meteo_days.pkl`
- **Приоритет:** **ВЫСОКИЙ** - необходим для структуры данных

---

## Структура данных для обучения (SPOTS модель)

```python
# Формат flights_by_cell_day_spot:
flights_by_cell_day_spot[cell_index][day_index] = {
    spot_id: [
        ('2023-10-01 10:12:58', (105.2, None, None, 46.272567, 13.473567)),
        #                          score   alt  takeoff_alt  lat         lon
    ]
}

# Для наших данных (дополнительно с avg_speed):
{
    'datetime': '2023-10-01 10:12:58',
    'score': 105.2,              # Из league.route.points
    'alt': None,                 # Для SPOTS всегда None
    'avg_speed': 22.34,          # Из league.route.avgSpeed (замена plaf)
    'takeoff_alt': None,         # TODO: из SRTM или spots.pkl
    'lat': 46.272567,
    'lon': 13.473567,
    'cell_lat': 46,
    'cell_lon': 13,
    'cell_index': None,          # TODO: из sorted_cells.pkl
    'day_index': None,           # TODO: из meteo_days.pkl
    'spot_id': 23,
    'spot_name': 'Stol',
    'mountainess': None,         # TODO: из SRTM
}
```

---

## План действий

### Этап 1: Добавить cell_index и day_index (КРИТИЧНО)
```python
# Загрузить существующие данные
with open('neural_network/bin/data/sorted_cells.pkl', 'rb') as f:
    sorted_cells = pickle.load(f)

with open('neural_network/bin/data/meteo_days.pkl', 'rb') as f:
    meteo_days = pickle.load(f)

# Вычислить индексы
cell_to_index = {cell: i for i, cell in enumerate(sorted_cells)}
day_to_index = {day: i for i, day in enumerate(meteo_days)}

cell_index = cell_to_index.get((cell_lat, cell_lon))
day_date = datetime.strptime(dt_formatted, '%Y-%m-%d %H:%M:%S').date()
day_index = day_to_index.get(day_date)
```

### Этап 2: Добавить takeoff_alt
```python
# Вариант A: Из существующего spots.pkl
with open('neural_network/bin/data/spots.pkl', 'rb') as f:
    spots_data = pickle.load(f)
    takeoff_alt = spots_data.get(spot_id, {}).get('alt')

# Вариант B: Из SRTM
# См. scripts/download_elevation_tiles.py
```

### Этап 3: Создать PKL файлы
```python
# Конвертация JSON в PKL формат для обучения
import pickle

flights_by_cell_day_spot = ...  # структура из наших данных

with open('neural_network/bin/data/flights_by_cell_day_spot.pkl', 'wb') as f:
    pickle.dump(flights_by_cell_day_spot, f)
```

---

## Статистика по данным

Текущий набор данных (xcontest):
- Всего полётов: ~11,759
- Уникальных spots: ~252
- Основной регион: Словения (SI), Северная Италия
- Диапазон дат: 2023-2025 (нужно проверить)

Топ-3 ячейки:
- 46_13 (46°N, 13°E) — 6,393 полёта
- 45_13 (45°N, 13°E) — 3,468 полётов
- 46_14 (46°N, 14°E) — 1,049 полётов

---

## Обновление PKL структуры (Issue #16)

**Дата:** 2026-01-03

### Удалены неиспользуемые поля из CELLS model

Поскольку xContest JSON не предоставляет данные о высоте (`alt`, `plaf`), а эти поля не использовались в обучении, они были удалены из PKL структуры.

**Было (7 полей):**
```python
(datetime, (score, alt, plaf, lat, lon, takeoff_alt, mountainess))
#         [0]    [1][0] [1][1] [1][2] [1][3] [1][4] [1][5]      [1][6]
```

**Стало (5 полей):**
```python
(datetime, (score, lat, lon, takeoff_alt, mountainess))
#         [0]    [1][0] [1][1] [1][2] [1][3]      [1][4]
```

### Убраны:
- **`alt`** (f[1][1]) - был 0.0, не использовался
- **`plaf`** (f[1][2]) - был avgSpeed, семантически неверно, не использовался

### Остались:
- **`score`** (f[1][0]) - используется для crossability threshold
- **`lat`** (f[1][1]) - используется для super-resolution
- **`lon`** (f[1][2]) - используется для super-resolution
- **`takeoff_alt`** (f[1][3]) - из DEM/SRTM, используется для altitude binning
- **`mountainess`** (f[1][4]) - из DEM/SRTM, используется в модели

### TODO для будущей оптимизации:
- `takeoff_alt` можно вычислять на лету из DEM по lat/lon (~16 байт/полёт)
- `mountainess` хранится дублированно (per-flight и per-cell в отдельном файле)
