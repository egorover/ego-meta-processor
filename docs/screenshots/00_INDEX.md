# Индекс доказательной базы (docs/screenshots/)

**Версия:** 1.0.0 | **Дата:** 31.10.2026
**Правило:** каждый файл в папке имеет строку в этом индексе и наоборот.
Новый скриншот добавляется только вместе со строкой таблицы.

**Легенда приоритетов:**
* **P0** — обязательны в репозитории: прямые доказательства требований ТЗ.
* **P1** — контекст и схема: используются в README и отчёте.
* **P2** — портфолио/кейс: в репозитории опциональны.

**Легенда статусов:**
* ✅ — файл сохранён и проверен
* ⬜ — файл запланирован, будет добавлен позже

---

## P0 — доказательства ТЗ (для сдачи ДЗ)

| № | Файл | Что доказывает | Ссылка в документах | Статус |
|:--:|:---|:---|:---|:---:|
| 01 | `01_orders_all_five_statuses.png` | Таблица `orders` со всеми 5 статусами: `done`, `needs_review`, `duplicate`, `failed`, `processing` | ТЗ п.11; Отчёт §3, §4 | ✅ |
| 02 | `02_log_triple_journaling_full.png` | `log`: цепочки validate→dedup→llm→parse + `rejected` (422/413/508/503) + `dedup/duplicate` + `llm/failed (502)` | ТЗ п.10; Отчёт §2, §4 | ✅ |
| 03 | `03_ew_settings_connected.png` | Workflow settings 3.06.13 → Error Workflow = 3.06.14 | ТЗ п.8; Отчёт §3 | ✅ |
| 04 | `04_ew_canvas_three_nodes.png` | Канвас 3.06.14: Error Trigger → Сборка ошибки → Telegram (3 ноды) | ТЗ п.8 | ✅ |
| 05 | `05_ew_telegram_alert_crash.png` | TG-алерт 🛑 при индуцированном крахе PATCH (EW работает, а не просто подключён) | ТЗ п.8; Отчёт §4 | ✅ |
| 06 | `06_sbt_t6_orders_failed_retry.png` | Строка `sbt_fedor`: `failed`, `review_reason`, заполненный `retry_payload` | ТЗ п.7; Отчёт §4 | ✅ |
| 07 | `07_sbt_t6_log_llm_failed_502.png` | `log`: validate/ok → dedup/ok → llm/failed (502) → parse/skipped | ТЗ п.7, п.10 | ✅ |
| 08 | `08_sbt_t7_log_rejected_503.png` | `log`: validate/rejected `processing_disabled` (503); строки в `orders` нет | ТЗ п.2; улучшение v.3 | ✅ |
| 09 | `09_tr_t5_orders_duplicate.png` | Строка Бориса `duplicate` + `retry_payload` со ссылкой на оригинал | ТЗ п.3, п.11 | ✅ |
| 10 | `10_tr_t5_log_dedup_duplicate.png` | `log`: dedup/duplicate — модель не вызывалась | ТЗ п.3, п.10 | ✅ |
| 11 | `11_canvas_v3_overview.png` | Общий вид канваса 3.06.13 | README §2 | ✅ |

## P1 — контекст и схема (планируется)

| № | Файл | Что показывает | Ссылка в документах | Статус |
|:--:|:---|:---|:---|:---:|
| 12 | `12_tg_alert_llm_unavailable_t6.png` | TG-алерт 🔴 по T6 (канал алертов) | ТЗ п.7 | ⬜ |
| 13 | `13_tr_summary_codes.png` | Output «Сводка результатов» TR: коды 200/422/413/508 | TESTING_PROTOCOL §2 | ⬜ |
| 14 | `14_nocodb_schema_orders.png` | Схема `orders`: `review_reason`, `http_code`, `retry_payload` | README §6 | ⬜ |
| 15 | `15_sticky_instructions_canvas.png` | Sticky-инструкции TR/SBT/EW на холстах | Культура сопровождения | ⬜ |

## P2 — портфолио / кейс (планируется)

| № | Файл | Что показывает | Использование | Статус |
|:--:|:---|:---|:---|:---:|
| 16 | `16_v2_vs_v3_log_coverage.png` | Сравнение: пустой `log` в SBT v.2 vs полный `log` в v.3 | Кейс «дефект → гипотеза → решение» | ⬜ |
| 17 | `17_orders_cost_columns.png` | Колонки `tokens_used`, `cost_rub` — экономика LLM | Кейс «стоимость под контролем» | ⬜ |

---

## Правила гигиены скриншотов

1. **Секреты:** на кадрах с NocoDB/API заблюрен `xc-token`; в TG — только служебные алерты.
2. **Имена стабильны:** не переименовывать после коммита — на них ссылаются документы.
3. **Колонки-доказательства видимы:** `status`, `review_reason`, `retry_payload`, `http_code` не обрезаются.

## Проверка целостности (PowerShell, из корня репозитория)

```powershell
$index = Get-Content docs/screenshots/00_INDEX.md -Raw
[regex]::Matches($index, '(\d{2}_[a-z0-9_]+\.png)') |
  ForEach-Object { $_.Groups[1].Value } | Sort-Object -Unique |
  ForEach-Object { if (Test-Path "docs/screenshots/$_") { "✅ $_" } else { "❌ ОТСУТСТВУЕТ: $_" } }