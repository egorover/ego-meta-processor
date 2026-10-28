# 🤖 EgoMeta Processor: Обработка заказов v2
AI-powered pipeline для автоматической обработки заказов на базе n8n, NocoDB и LLM (YandexGPT/GigaChat). Система реализует принципы **Meta Automation**: минимум ручных операций, гарантированное логирование всех исходов и маршрутизация через данные, а не структуру схемы.

### 📄 `README.md` (Русскоязычная версия)

Документ адаптирован для русскоязычной аудитории с сохранением технических терминов на английском там, где это общепринятая практика в разработке.

## 🏗 Архитектура проекта

Проект состоит из трех изолированных воркфлоу, обеспечивающих безопасность разработки и надежность продакшена:

| Компонент | Файл | Назначение |
| :--- | :--- | :--- |
| **Order Processing (OP)** | `3.06.10 Qwe - OP - Order Processing v2 (improved final)` | Основной боевой пайплайн. Валидация, идемпотентность, разбор заказов ИИ-агентом, обновление статусов в NocoDB. |
| **Test Runner (TR)** | `3.06.11 Qwe - TR - Test Runner` | Автоматизированный регрессионный тест для проверки happy-path сценариев и граничных условий валидации. |
| **Smoke Broken Tests (SBT)** | `3.06.12 Qwe - SBT - Smoke Broken Tests` | Изолированный контур для проверки failure-handling: аварии LLM, circuit breaker, rate limiting. |

### 🔑 Ключевые фичи v2
- **Catalog-aware Parsing:** ИИ-агент сопоставляет позиции с внутренним каталогом (SKU matching). Неизвестные товары автоматически уходят в очередь `needs_review`.
- **Transparent Review Queue:** Поле `review_reason` в таблице заказов объясняет менеджеру причину ручной проверки (`no_address`, `unknown_items`, `llm_failed`).
- **Guaranteed Logging:** Все события (включая rejected/duplicate) фиксируются в таблице `log` с HTTP-кодами и кодами ошибок.
- **Circuit Breaker Pattern:** Возможность мгновенного отключения приема заказов через флаг `orders_enabled` в NocoDB без деплоя.
- **Isolated Failure Testing:** SBT-раннер использует префикс `sbt_` для контактов, исключая загрязнение боевых данных при проверке деградации.

## 📋 Протокол тестирования

Полная инструкция по запуску регрессии и сломанных тестов доступна в файле **[docs/TESTING_PROTOCOL.md](docs/TESTING_PROTOCOL.md)**.

> ⚠️ **Важно:** Перед первым запуском убедитесь, что в NocoDB созданы дополнительные колонки:
> - `orders.review_reason` (Text)
> - `log.http_code` (Number)

## 🚀 Развертывание (Deployment)

1. Импортируйте `3.06.10_Qwe_OP_Order_Processing_v2.json` в n8n.
2. Настройте credentials:
   - **NocoDB API Token** (для доступа к таблицам orders, log, settings)
   - **OpenAI account** (для YandexGPT)
   - **Header Auth** (для NocoDB REST API patch)
   - **Telegram Bot** (для алертов об ошибках LLM)
3. Активируйте webhook по пути `/webhook/order-v2`.
4. Для запуска тестов импортируйте TR и SBT воркфлоу. Убедитесь, что основной OP находится в режиме **Listen for Test Event**.

##  Технический стек

- **Orchestration:** n8n (v1.x/v2.x)
- **Database:** NocoDB (REST API v2)
- **LLM:** YandexGPT (via OpenAI-compatible node), GigaChat (via HTTP Request)
- **Notifications:** Telegram Bot API
- **Version Control:** Git (GitHub)

## 👤 Автор

**Александр Егоров (Alex Egorov)** | AI Developer & Prompt Engineer  
GitHub: [egorover](https://github.com/egorover) | Telegram: [@egorover_ai](https://t.me/egorover_ai)

---
*Проект разработан в рамках курса по AI-автоматизации. Версия 1.0.0 отражает стабильное состояние системы после полного цикла регрессионного и smoke-тестирования.*



# 🤖 EgoMeta Processor: Orders processing v2


AI-powered order processing pipeline built with n8n, NocoDB and LLM routing (YandexGPT/GigaChat).

### 📄 `README.md` (English version)

## 🏗 Architecture
- **3.06.10 Qwe - OP**: Main production workflow. Validates, deduplicates, parses orders via LLM, updates NocoDB.
- **3.06.11 Qwe - TR**: Automated regression test runner for happy-path scenarios.
- **3.06.12 Qwe - SBT**: Isolated smoke tests for failure handling (LLM errors, circuit breaker).

## 📋 Key Features
- Idempotent order processing with derived keys
- Catalog-based SKU matching with fallback to review queue
- Guaranteed logging of ALL outcomes (including rejected/duplicate) via pre-branch journal assembly
- Circuit breaker pattern via NocoDB settings toggle
- Telegram alerts on LLM failures with retry payload

##  Testing Protocol
See [docs/TESTING_PROTOCOL.md](docs/TESTING_PROTOCOL.md) for full test suite instructions including:
- Healthy path regression (E1-E6)
- Validation edge cases (422, 413, 508)
- Broken path tests (T6: LLM auth error, T7: circuit breaker)

## 🚀 Deployment
1. Import `3.06.10_Qwe_OP_Order_Processing_v2.json` into n8n
2. Configure credentials: NocoDB API Token, OpenAI account (YandexGPT), Header Auth, Telegram
3. Ensure NocoDB tables exist: `orders` (with `review_reason`), `log` (with `http_code`), `settings`
4. Activate webhook at `/webhook/order-v2`

##  Author
**Alexander Egorov (Alex Egorov)** | AI Developer & Prompt Engineer  
GitHub: [egorover](https://github.com/egorover) | Telegram: [@egorover_ai](https://t.me/egorover_ai)