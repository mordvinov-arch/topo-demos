# Climate Instability Index (CII)

Один индекс **0–100** климатической нестабильности региона на доменно-независимом движке topo-stress.
Этап 1 — глобальный пилот HadCRUT5 (аномалия температуры, 1850–2025).
Этап 2 (обновлённое ТЗ) — регионы + осадки/засуха + экономические потери.

- One-pager: `product/CII_One_Pager.md`
- One-pager страховые/ESG: `product/CII_One_Pager_Insurance_ESG.md`
- Глобальный отчёт: `results/REPORT_climate_cii.md` (`REPORT_climate_cii.txt`)
- Региональный отчёт: `results/regions/REPORT_climate_cii_regions.md` (`..._regions.txt`)
- Ряд индекса (глобал): `results/cii_series.csv`
- Ряды индекса (регионы): `results/regions/cii_<region>.csv`
- Сводные числа: `results/cii_report.json`, `results/regions/cii_regions_summary.json`
- Графики: `figures/`, `figures/regions/`

---

## 1. Установка и запуск

Зависимости: `numpy, pandas, scipy, scikit-learn, matplotlib, requests`. Python 3.10+.

```bash
# этап 1 — глобальный пилот (HadCRUT5, кэш Платформы)
python scripts/build_climate_cii.py

# этап 2 — регионы (Open-Meteo, сеть; данные кэшируются в data/regions/)
python scripts/build_climate_cii_regions.py   # индексы по 4 регионам
python scripts/economy_regions.py             # регрессия потерь + страховой индекс
python scripts/package_regions.py             # графики + отчёты регионов
```

Данные этапа 2 — Open-Meteo Historical Weather Archive (без ключа): суточные
средняя температура и сумма осадков по координатам каждого региона, 1940-01 … 2025-02.
Кэш: `data/regions/<region>_daily.csv`.

Параметры (в начале скрипта):

| Параметр | Значение | Смысл |
|---|---|---|
| `WINDOW` | 36 мес. | длина скользящего окна |
| `STEP` | 6 мес. | шаг окна |
| `EVT_Q` | 0.85 | порог хвоста температуры |
| `G_PREC_Q` | 0.20 | нижний квантиль осадков («сухость») |
| `WEIGHTS` | 0.20×5 | веса компонент (vol/trend/EVT/regime/drought) |

## 2. Выходные файлы (этап 2)

- `results/regions/cii_<region>.csv` — окна: `window_start, cii, vol, trend, tail_risk, drought_heat, drought, regime_change`;
- `results/regions/cii_regions_summary.json` — статистика + привязка событий к потерям;
- `figures/regions/cii_regions_overview.png` — 4 панели CII по регионам;
- `figures/regions/cii_regions_losses.png` — потери (млрд $) vs CII-процентиль.

## 3. Методология (этап 2)

1. Суточные T/осадки Open-Meteo → месячные (средняя T, сумма осадков);
2. Скользящее окно 36 мес → 5 компонент: `vol`, `trend`, `tail_risk` (EVT),
   `regime_change` (Махаланобис до доиндустриального базиса), **`drought_heat`**
   (термально-водный стресс = частота×глубина сухих месяцев с осадками < 20-го процентиля
   + вклад жары выше 90-го процентиля);
3. Каждая компонента → процентильный ранг [0,1] по истории;
4. `CII = 0.20*vol + 0.20*trend + 0.20*EVT + 0.20*regime_change + 0.20*drought_heat` → ×100;
5. Валидация: пики CII vs документированные события и экономические потери.

## 4. Формат данных окна (глобал)

Пример (`cii_series.csv`):

```text
window_start,cii,vol,trend,tail_risk,regime_change,regime_label,hsic_stat,hsic_p
2022-01-01,90.72,0.1235,0.0157,16.077,140.10,1,0.00476,0.000
```

## 5. Контракт API (черновик)

Базовый URL: `https://cii.example.com`

| Метод | Путь | Ответ |
|---|---|---|
| GET | `/cii/v1/status` | `{"service":"cii","status":"ok","api_version":"1.0"}` |
| GET | `/cii/v1/index?region=GLOBAL` | текущий CII по окну |
| GET | `/cii/v1/index?region=southern_europe` | то же для региона |
| GET | `/cii/v1/history?region=GLOBAL&from=2010-01-01&to=2025-01-01` | ряд CII |
| GET | `/cii/v1/components?region=GLOBAL` | значения компонент + веса |

Регионы: `GLOBAL | southern_europe | texas | india | australia`.

Пример ответа `GET /cii/v1/index?region=GLOBAL`:

```json
{
  "region": "GLOBAL",
  "window_start": "2022-01-01",
  "cii": 90.7,
  "components": {"vol": 0.25, "trend": 0.25, "evt": 0.25, "regime": 0.25},
  "regime_change": 140.1,
  "evt_shape_xi": -0.29,
  "tail_type": "Weibull",
  "hsic_p": 0.0,
  "level": "HIGH",
  "generated_at": "2026-08-29T00:00:00Z"
}
```

Для региона индекс использует 5 компонент (включая `drought_heat`), веса 0.20.

Правила сервиса:
- расчёт по месячным данным с задержкой; индекс отражает **текущий 36-мес режим**, не отдельный месяц;
- `level`: `LOW < 50 ≤ MEDIUM < 75 ≤ HIGH`.

## 6. Честные ограничения

- Индекс — мера **режима** (36 мес), а не прогноз даты экстремума;
- Композитный CII по муссонной **Индии**: засуха теперь меряется по сезону муссона (JJAS) —
  провал муссона ловится верно, но композитный 36-мес режим всё ещё ранжирует недавние
  муссонные экстремумы ниже волатильности середины XX в. — для точечного детекта нужна
  месячная SP(E)I-модель муссона;
- CII мерит жару/сухость/нестабильность: сухопутные потери положительно связаны с CII,
  а влажные (наводнения/ураганы) — **вне шкалы CII** (нужна отдельная «влажная» шкала);
- Общая экономическая R²≈0 (n=25) — следствие смеси противоположных типов и шумных оценок потерь;
- Веса компонент равновесные (0.20×5) и калибруются;
- Данные Open-Meteo (аппроксимированные по координатам точек), для отраслевых сценариев — ERA5/Copernicus.

## 7. Статус по ТЗ «Климат» (обновлённое)

| Пункт (этап) | Статус |
|---|---|
| 1. Региональные адаптеры (Юж. Европа / Техас / Индия / Австралия) | ✅ Open-Meteo, кэш |
| 2. Осадки / засуха | ✅ компонента `drought_heat` (сезонный SPI) |
| 3. Экономические потери | ✅ регрессия + страховой индекс потерь/ВВП (n=25) |
| 4. Региональная нормализация | ✅ z-score компонент по региону + сезонный SPI |
| 5. Упаковка в one-pager + перемещение в Эраст | ✅ `product/` + копия в `Эраст/Платформа/products/Климат_CII` |
| 6. Доп. события для регрессии | ✅ 25 документированных событий по 4 регионам |
| 7. Муссонная калибровка Индии | ✅ засуха по сезону муссона (JJAS) |
| 8. One-pager страховые/ESG | ✅ `product/CII_One_Pager_Insurance_ESG.md` |

Запуск этапа 2: `build_climate_cii_regions.py` → `economy_regions.py` → `package_regions.py`.
Дальнейшие шаги: месячная SP(E)I-модель муссона (точный детект для Индии), отдельная
«влажная» шкала (наводнения/ураганы), полная многолетняя эконометрика потерь.
