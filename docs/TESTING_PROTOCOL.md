# Протокол тестирования EgoMeta Processor v3

**Версия:** 1.2.1  
**Дата актуализации:** 31.10.2026  
**Автор:** Александр Егоров (Alex Egorov) | AI Developer & Prompt Engineer  
**GitHub:** [egorover](https://github.com/egorover) | **Telegram:** [@egorover_ai](https://t.me/egorover_ai)  

---

> **Изменения v1.2.1:** Зафиксировано ограничение ручного автотестирования (race condition в тестовом вебхуке n8n) с рекомендацией использовать PowerShell-команды. План внедрения полноценного автотестирования перенесен на v.4.  
> **Изменения v1.2.0:** Переход на архитектуру v.3. Внедрено тройное логирование (гарантия записи всех исходов), материализация статуса `duplicate`, интеграция Error Workflow (3.06.14). Таблицы тестов дополнены готовыми PowerShell-командами для мгновенного ручного запуска.

---

> ⚠️ **Важное замечание о режиме автотестирования (v1.2.1)**
>
> В текущей версии проекта (v.3) автоматизированные прогоны через воркфлоу **3.06.11 TR** и **3.06.12 SBT** демонстрируют **нестабильную воспроизводимость** из-за **асинхронной природы выполнения запросов**:
>
> *   HTTP-запросы к тестовому вебхуку n8n (`/webhook-test/`) возвращают `202 Accepted` **до завершения** обработки заказа.
> *   Следующий запрос в цепочке TR может поступить на воркфлоу раньше, чем предыдущий заказ успеет записаться в NocoDB и проиндексироваться.
> *   Это вызывает **race condition** (состояние гонки) в логике дедупликации и rate limiting: например, T5 может не распознать E2 как дубль, а T8c — не сработать как rate_limited.
>
> **Рекомендация для v.3:** Для получения **детерминированных, воспроизводимых результатов** используйте **ручное тестирование** по готовым PowerShell-командам из таблиц разделов 2 и 3. Интервал между запросами (~1.5 сек) гарантирует корректную индексацию записей в NocoDB.
>
> **План на v.4:** Полноценное автотестирование будет реализовано через переход на боевой webhook (`/webhook/`), транзакционные ожидания записи и асинхронный polling результатов через NocoDB API.

---

## 1. Архитектура тестовой среды

| Сценарий | Файл / ID | Назначение | Контакты-маркеры |
| :--- | :--- | :--- | :--- |
| **Основной процесс** | `3.06.13 Qwe - OP - Order Processing v.3` | Боевой воркфлоу обработки заказов (v.3) | Реальные клиенты |
| **Здоровый прогон (TR)** | `3.06.11 Qwe - TR - Test Runner` | Регрессия бизнес-логики, happy path, валидация | Обычные контакты (`alexey@...`, `boris@...`) |
| **SBT (Smoke Broken)** | `3.06.12 Qwe - SBT - Smoke Broken Tests` | Проверка обработки ошибок, деградации, circuit breaker | Префикс `sbt_` (`sbt_fedor@...`, `sbt_switch@...`) |
| **Error Workflow** | `3.06.14 Qwe - EW - Error Workflow` | Перехват необработанных крахов инфраструктуры | Н/Д (триггерится системно) |

⚠️ **Критическое правило:** Никогда не запускайте SBT вместе со здоровым прогоном. Никогда не оставляйте среду сломанной после теста.

---

## 2. Здоровый прогон (Регрессия через TR)

**Цель:** Убедиться, что штатные сценарии работают, дедупликация срабатывает, а данные корректно пишутся в NocoDB.

### Карта тестовых сценариев (3.06.11 TR)

<table>
  <thead>
    <tr>
      <th align="left">№</th>
      <th align="left">Метка</th>
      <th align="left">Контакт</th>
      <th align="left">Текст заказа / Особенности</th>
      <th align="left">Ожидаемый статус в <code>orders</code></th>
    </tr>
  </thead>
  <tbody>
    <!-- Строка 1 -->
    <tr>
      <td>1</td>
      <td><code>ОРИГИНАЛ #1</code></td>
      <td><code>alexey@test.ru</code></td>
      <td>«Здравствуйте! Нужно масло OIL-5W40 2 шт» <em>(без адреса)</em></td>
      <td><code>needs_review</code> (<code>no_address</code>)</td>
    </tr>
    <tr>
      <td colspan="5" style="background-color: #1e1e24; color: #ffffff; font-family: monospace; padding: 8px; font-size: 0.9em;">
        <code>$body = @{ customer="Алексей"; contact="alexey@test.ru"; text="Здравствуйте! Нужно масло OIL-5W40 2 шт" } | ConvertTo-Json -Compress; Invoke-RestMethod -Uri "http://localhost:5678/webhook-test/order-v2" -Method Post -ContentType "application/json; charset=utf-8" -Body $body</code>
      </td>
    </tr>
    <!-- Строка 2 -->
    <tr>
      <td>2</td>
      <td><code>E1 — без адреса</code></td>
      <td><code>boris@test.ru</code></td>
      <td>«Нужно масло OIL-5W40 2 шт» <em>(без адреса)</em></td>
      <td><code>needs_review</code> (<code>no_address</code>)</td>
    </tr>
    <tr>
      <td colspan="5" style="background-color: #1e1e24; color: #ffffff; font-family: monospace; padding: 8px; font-size: 0.9em;">
        <code>$body = @{ customer="Борис"; contact="boris@test.ru"; text="Нужно масло OIL-5W40 2 шт" } | ConvertTo-Json -Compress; Invoke-RestMethod -Uri "http://localhost:5678/webhook-test/order-v2" -Method Post -ContentType "application/json; charset=utf-8" -Body $body</code>
      </td>
    </tr>
    <!-- Строка 3 -->
    <tr>
      <td>3</td>
      <td><code>E2 — + адрес → DONE</code></td>
      <td><code>boris@test.ru</code></td>
      <td>«Нужно масло OIL-5W40 2 шт. Доставка: г. Москва, ул. Тверская, д. 1, кв. 5»</td>
      <td><code>done</code></td>
    </tr>
    <tr>
      <td colspan="5" style="background-color: #1e1e24; color: #ffffff; font-family: monospace; padding: 8px; font-size: 0.9em;">
        <code>$body = @{ customer="Борис"; contact="boris@test.ru"; text="Нужно масло OIL-5W40 2 шт. Доставка: г. Москва, ул. Тверская, д. 1, кв. 5" } | ConvertTo-Json -Compress; Invoke-RestMethod -Uri "http://localhost:5678/webhook-test/order-v2" -Method Post -ContentType "application/json; charset=utf-8" -Body $body</code>
      </td>
    </tr>
    <!-- Строка 4 -->
    <tr>
      <td>4</td>
      <td><code>E3 — срочно</code></td>
      <td><code>viktor@test.ru</code></td>
      <td>«Срочно нужно масло OIL-5W40 2 шт! Доставка: г. Москва, ул. Тверская, д. 1, кв. 5. Примечание: позвоните»</td>
      <td><code>done</code> <em>(с наценкой 15%)</em></td>
    </tr>
    <tr>
      <td colspan="5" style="background-color: #1e1e24; color: #ffffff; font-family: monospace; padding: 8px; font-size: 0.9em;">
        <code>$body = @{ customer="Виктор"; contact="viktor@test.ru"; text="Срочно нужно масло OIL-5W40 2 шт! Доставка: г. Москва, ул. Тверская, д. 1, кв. 5. Примечание: позвоните" } | ConvertTo-Json -Compress; Invoke-RestMethod -Uri "http://localhost:5678/webhook-test/order-v2" -Method Post -ContentType "application/json; charset=utf-8" -Body $body</code>
      </td>
    </tr>
    <!-- Строка 5 -->
    <tr>
      <td>5</td>
      <td><code>E4 — мульти</code></td>
      <td><code>galina@test.ru</code></td>
      <td>«Нужны колодки BRK-310 1 шт и фильтр воздушный FLT-200 3 шт. Доставка: г. Санкт-Петербург, Невский пр., д. 100»</td>
      <td><code>done</code></td>
    </tr>
    <tr>
      <td colspan="5" style="background-color: #1e1e24; color: #ffffff; font-family: monospace; padding: 8px; font-size: 0.9em;">
        <code>$body = @{ customer="Галина"; contact="galina@test.ru"; text="Нужны колодки BRK-310 1 шт и фильтр воздушный FLT-200 3 шт. Доставка: г. Санкт-Петербург, Невский пр., д. 100" } | ConvertTo-Json -Compress; Invoke-RestMethod -Uri "http://localhost:5678/webhook-test/order-v2" -Method Post -ContentType "application/json; charset=utf-8" -Body $body</code>
      </td>
    </tr>
    <!-- Строка 6 -->
    <tr>
      <td>6</td>
      <td><code>E5 — вне каталога</code></td>
      <td><code>dmitry@test.ru</code></td>
      <td>«Нужен антифриз G12 5 литров. Доставка: г. Казань, ул. Баумана, д. 5»</td>
      <td><code>needs_review</code> (<code>unknown_items:1</code>)</td>
    </tr>
    <tr>
      <td colspan="5" style="background-color: #1e1e24; color: #ffffff; font-family: monospace; padding: 8px; font-size: 0.9em;">
        <code>$body = @{ customer="Дмитрий"; contact="dmitry@test.ru"; text="Нужен антифриз G12 5 литров. Доставка: г. Казань, ул. Баумана, д. 5" } | ConvertTo-Json -Compress; Invoke-RestMethod -Uri "http://localhost:5678/webhook-test/order-v2" -Method Post -ContentType "application/json; charset=utf-8" -Body $body</code>
      </td>
    </tr>
    <!-- Строка 7 -->
    <tr>
      <td>7</td>
      <td><code>E6 — смешанный</code></td>
      <td><code>elena@test.ru</code></td>
      <td>«Нужно масло OIL-5W40 2 шт и антифриз G12 1 л. Доставка: г. Казань, ул. Баумана, д. 5»</td>
      <td><code>needs_review</code> (<code>unknown_items:1</code>)</td>
    </tr>
    <tr>
      <td colspan="5" style="background-color: #1e1e24; color: #ffffff; font-family: monospace; padding: 8px; font-size: 0.9em;">
        <code>$body = @{ customer="Елена"; contact="elena@test.ru"; text="Нужно масло OIL-5W40 2 шт и антифриз G12 1 л. Доставка: г. Казань, ул. Баумана, д. 5" } | ConvertTo-Json -Compress; Invoke-RestMethod -Uri "http://localhost:5678/webhook-test/order-v2" -Method Post -ContentType "application/json; charset=utf-8" -Body $body</code>
      </td>
    </tr>
    <!-- Строка 8 -->
    <tr>
      <td>8</td>
      <td><code>T2 — нет контакта</code></td>
      <td><em>(пусто)</em></td>
      <td>«Нужно масло»</td>
      <td><strong>Не создается</strong> (HTTP <code>422</code>)</td>
    </tr>
    <tr>
      <td colspan="5" style="background-color: #1e1e24; color: #ffffff; font-family: monospace; padding: 8px; font-size: 0.9em;">
        <code>$body = @{ customer="Тест"; contact=""; text="Нужно масло" } | ConvertTo-Json -Compress; Invoke-RestMethod -Uri "http://localhost:5678/webhook-test/order-v2" -Method Post -ContentType "application/json; charset=utf-8" -Body $body</code>
      </td>
    </tr>
    <!-- Строка 9 -->
    <tr>
      <td>9</td>
      <td><code>T3 — длинный текст</code></td>
      <td><code>test_long@test.ru</code></td>
      <td>Строка из 2500 символов «А»</td>
      <td><strong>Не создается</strong> (HTTP <code>413</code>)</td>
    </tr>
    <tr>
      <td colspan="5" style="background-color: #1e1e24; color: #ffffff; font-family: monospace; padding: 8px; font-size: 0.9em;">
        <code>$text = "A" * 2500; $body = @{ customer="Тест"; contact="test_long@test.ru"; text=$text } | ConvertTo-Json -Compress; Invoke-RestMethod -Uri "http://localhost:5678/webhook-test/order-v2" -Method Post -ContentType "application/json; charset=utf-8" -Body $body</code>
      </td>
    </tr>
    <!-- Строка 10 -->
    <tr>
      <td>10</td>
      <td><code>T4 — петля</code></td>
      <td><code>test_loop@test.ru</code></td>
      <td>«Тест петель» + заголовок <code>x-loop-depth: 1</code></td>
      <td><strong>Не создается</strong> (HTTP <code>508</code>)</td>
    </tr>
    <tr>
      <td colspan="5" style="background-color: #1e1e24; color: #ffffff; font-family: monospace; padding: 8px; font-size: 0.9em;">
        <code>$headers = @{ "x-loop-depth" = "1" }; $body = @{ customer="Тест"; contact="test_loop@test.ru"; text="Тест петель" } | ConvertTo-Json -Compress; Invoke-RestMethod -Uri "http://localhost:5678/webhook-test/order-v2" -Method Post -ContentType "application/json; charset=utf-8" -Headers $headers -Body $body</code>
      </td>
    </tr>
    <!-- Строка 11 -->
    <tr>
      <td>11</td>
      <td><code>T5 — дубль E2</code></td>
      <td><code>boris@test.ru</code></td>
      <td><strong>Точная копия текста E2</strong> <em>(г. Москва, ул. Тверская, д. 1, кв. 5)</em></td>
      <td><code>duplicate</code> <em>(модель не вызывается)</em></td>
    </tr>
    <tr>
      <td colspan="5" style="background-color: #1e1e24; color: #ffffff; font-family: monospace; padding: 8px; font-size: 0.9em;">
        <code>$body = @{ customer="Борис"; contact="boris@test.ru"; text="Нужно масло OIL-5W40 2 шт. Доставка: г. Москва, ул. Тверская, д. 1, кв. 5" } | ConvertTo-Json -Compress; Invoke-RestMethod -Uri "http://localhost:5678/webhook-test/order-v2" -Method Post -ContentType "application/json; charset=utf-8" -Body $body</code>
      </td>
    </tr>
    <!-- Строка 12 -->
    <tr>
      <td>12</td>
      <td><code>ОРИГИНАЛ #2</code></td>
      <td><code>maria@test.ru</code></td>
      <td>«Здравствуйте! Нужны колодки тормозные BRK-310 2 шт и масло моторное OIL-5W40 1 шт. Доставка: г. Новосибирск, ул. Ленина, д. 25, кв. 8»</td>
      <td><code>done</code></td>
    </tr>
    <tr>
      <td colspan="5" style="background-color: #1e1e24; color: #ffffff; font-family: monospace; padding: 8px; font-size: 0.9em;">
        <code>$body = @{ customer="Мария"; contact="maria@test.ru"; text="Здравствуйте! Нужны колодки тормозные BRK-310 2 шт и масло моторное OIL-5W40 1 шт. Доставка: г. Новосибирск, ул. Ленина, д. 25, кв. 8" } | ConvertTo-Json -Compress; Invoke-RestMethod -Uri "http://localhost:5678/webhook-test/order-v2" -Method Post -ContentType "application/json; charset=utf-8" -Body $body</code>
      </td>
    </tr>
    <!-- Строка 13 -->
    <tr>
      <td>13</td>
      <td><code>ОРИГИНАЛ #3</code></td>
      <td><code>petr@test.ru</code></td>
      <td>«Срочно: нужен фильтр масляный FLT-100 1 шт. Самовывоз: г. Екатеринбург, ул. Ленина, д. 15, офис 3. Примечание: пропуск закажите заранее»</td>
      <td><code>done</code> <em>(с наценкой 15%)</em></td>
    </tr>
    <tr>
</tbody>
</table>

> **Примечание к T5:** Для гарантированного срабатывания дедупа при ручном запуске между E2 и T5 должна пройти пауза ~1.5 сек, чтобы запись E2 успела проиндексироваться в NocoDB.

---

## 3. SBT: Smoke Broken Tests (3.06.12)

**Цель:** Подтвердить корректную обработку сбоев и работу Error Workflow.  
**Критерии приёмки в v.3:** Код HTTP-ответа, состояние таблицы `orders`, наличие записей в `log` (гарантировано тройным логированием), Telegram-алерт.

### Карта тестовых сценариев (3.06.12 SBT)

|<table>
  <thead>
    <tr>
      <th align="left">№</th>
      <th align="left">Метка</th>
      <th align="left">Контакт</th>
      <th align="left">Текст заказа / Особенности</th>
      <th align="left">Ожидаемый статус в <code>orders</code></th>
    </tr>
  </thead>
  <tbody>
    <!-- Строка 1 -->
    <tr>
      <td>1</td>
      <td><code>T6 — авария модели</code></td>
      <td><code>sbt_fedor@test.ru</code></td>
      <td>«Нужен фильтр масляный FLT-100 1 шт.» <em>(Требует предварительной поломки модели)</em></td>
      <td><code>failed</code> (<code>llm_failed:llm_unavailable</code>)</td>
    </tr>
    <tr>
      <td colspan="5" style="background-color: #1e1e24; color: #ffffff; font-family: monospace; padding: 8px; font-size: 0.9em;">
        <code>$body = @{ customer="SST_Fedor"; contact="sbt_fedor@test.ru"; text="Нужен фильтр масляный FLT-100 1 шт." } | ConvertTo-Json -Compress; Invoke-RestMethod -Uri "http://localhost:5678/webhook-test/order-v2" -Method Post -ContentType "application/json; charset=utf-8" -Body $body</code>
      </td>
    </tr>
    <!-- Строка 2 -->
    <tr>
      <td>2</td>
      <td><code>T7 — выключатель</code></td>
      <td><code>sbt_switch@test.ru</code></td>
      <td>«Тест выключателя SST» <em>(Требует установки <code>orders_enabled = false</code>)</em></td>
      <td><strong>Не создается</strong> (HTTP <code>503</code>)</td>
    </tr>
    <tr>
      <td colspan="5" style="background-color: #1e1e24; color: #ffffff; font-family: monospace; padding: 8px; font-size: 0.9em;">
        <code>$body = @{ customer="SST_Test"; contact="sbt_switch@test.ru"; text="Тест выключателя SST" } | ConvertTo-Json -Compress; Invoke-RestMethod -Uri "http://localhost:5678/webhook-test/order-v2" -Method Post -ContentType "application/json; charset=utf-8" -Body $body</code>
      </td>
    </tr>
    <!-- Строка 3 -->
    <tr>
      <td>3</td>
      <td><code>EW — крах инфраструктуры</code></td>
      <td><code>test@test.ru</code></td>
      <td>«Нужно масло» <em>(Требует смены порта в PATCH-ноде на 8081 и включения Active)</em></td>
      <td><code>processing</code> <em>(зависает до обновления)</em></td>
    </tr>
    <tr>
      <td colspan="5" style="background-color: #1e1e24; color: #ffffff; font-family: monospace; padding: 8px; font-size: 0.9em;">
        <code>$body = @{ customer="Тест"; contact="test@test.ru"; text="Нужно масло" } | ConvertTo-Json -Compress; Invoke-RestMethod -Uri "http://localhost:5678/webhook/order-v2" -Method Post -ContentType "application/json; charset=utf-8" -Body $body</code>
      </td>
    </tr>
  </tbody>
</table>

---

## 4. Архитектурные улучшения v.3 (Решенные проблемы v.2)

✅ **Тройное логирование (Triple Journaling):** Ноды записи в `log` перенесены **до** точек ветвления (validate → dedup → llm/parse). Теперь **любой** исход (включая `rejected`, `duplicate`, `rate_limited`) гарантированно оставляет след в таблице `log`. Ограничение v.2 полностью устранено.  
✅ **Материализация дублей:** Статус `duplicate` теперь создает аудит-строку в таблице `orders` с указанием исходного заказа в `retry_payload`, что выполняет требование ТЗ п.11.  
✅ **Error Workflow:** Внедрен отдельный сценарий 3.06.14 для перехвата критических инфраструктурных сбоев (ТЗ п.8).

---

## 5. Чек-лист чистоты данных

Перед каждым прогоном убедитесь:
- [ ] Удалены строки `orders` и `log` с контактами `sbt_*`, `spam@test.ru` и тестовыми пользователями.
- [ ] В NocoDB `settings.orders_enabled = true`.
- [ ] Credential YandexGPT корректен, модель установлена в `.../yandexgpt/latest`.
- [ ] В `3.06.12 Qwe - SBT` все тестовые блоки закомментированы.
- [ ] В `3.06.13 Qwe - OP` порт в ноде PATCH равен `8080`, переключатель **Active** выключен (если не проводится тест EW).
- [ ] Скрипт автоматизации `run_full_test_v3.ps1` не содержит захардкоженных токенов (токен вынесен в env или файл игнорируется в `.gitignore`).

---

## 6. Типичные ошибки и решения

| Симптом | Причина | Решение |
| :--- | :--- | :--- |
| `ERR_DATABASE_OP_FAILED` при PATCH | Нет колонки `review_reason` или `http_code` в NocoDB | Создать колонки (типы Text и Number) до запуска |
| T5 не создает `duplicate` | Race condition или микроразличия в тексте | Используйте паузу 1.5с между E2 и T5; проверьте текст байт-в-байт |
| T6 не падает | Кэш кредов/модели n8n | Пересохранить credential, перезапустить n8n или сменить имя модели |
| T7 создает заказ | `orders_enabled` не применен в БД | Проверить значение в NocoDB, обновить страницу браузера |
| Нет записей в `log` при SBT | **Устаревшая логика v.2** | В v.3 это исправлено. Проверьте, что запущен именно `3.06.13`, а не `3.06.10` |
| Webhook возвращает `404` | Тестовый режим выключен или воркфлоу не Active | Нажать "Listen for Test Event" или включить тумблер Active для боевого URL |

---

## 7. Глоссарий

*   **SBT (Smoke Broken Test)** — изолированный набор тестов путей обработки ошибок.
*   **Circuit Breaker** — аварийное отключение приёма заказов флагом в БД (`orders_enabled = false`).
*   **Review Reason** — машиночитаемая причина статуса `needs_review`.
*   **Idempotency Key** — ключ запроса (клиентский или derived-хэш), предотвращающий дубли.
*   **Triple Journaling** — архитектура v.3 с тремя точками записи лога до ветвлений бизнес-логики, гарантирующая 100% наблюдаемость.
*   **Error Workflow (EW)** — внешний сценарий-предохранитель, перехватывающий необработанные крахи основного воркфлоу.

---
*Этот документ является живым. При изменении архитектуры или добавлении новых тестов — обновляйте соответствующие разделы и повышайте версию.*