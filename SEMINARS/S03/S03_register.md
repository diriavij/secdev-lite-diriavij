# S03 - Шаблон реестра NFR (Given-When-Then)

Этот файл - рабочий **реестр NFR** на семинаре.
Заполняйте его у себя в репозитории в: `SEMINARS/S03/S03_register.md`.

---

## Поля реестра (data dictionary)

* **ID** - короткий идентификатор, например `NFR-001`.
* **User Story / Feature** - к какой истории/фиче относится требование (ссылка допустима).
* **Category** - выберите из банка (напр.: `Performance`, `Security-AuthZ/RBAC`, `RateLimiting`, `Privacy/PII`, `Observability/Logging`, …).
* **Requirement (NFR)** - *измеримое* требование (числа/пороги/границы действия).
* **Rationale / Risk** - зачем это нужно, какой риск/ценность покрываем.
* **Acceptance (G-W-T)** - проверяемая формулировка: *Given … When … Then …*.
* **Evidence (test/log/scan/policy)** - чем подтвердим выполнение (тест, лог-шаблон, сканер, политика).
* **Trace (issue/link)** - ссылка на задачу, обсуждение, артефакт.
* **Owner** - ответственный.
* **Status** - `Draft` | `Proposed` | `Approved` | `Implemented` | `Verified`.
* **Priority** - `P1 - High` | `P2 - Medium` | `P3 - Low`.
* **Severity** - `S1 - Critical` | `S2 - Major` | `S3 - Minor`.
* **Tags** - произвольные метки (через запятую).

---

## Таблица реестра

| ID      | User Story / Feature      | Category                 | Requirement (NFR)                                                   | Rationale / Risk                     | Acceptance (G-W-T)                                                                                                    | Evidence (test/log/scan/policy)               | Trace (issue/link) | Owner  | Status   | Priority    | Severity   | Tags              |
| ------- | ------------------------- | ------------------------ | ------------------------------------------------------------------- | ------------------------------------ | --------------------------------------------------------------------------------------------------------------------- | --------------------------------------------- | ------------------ | ------ | -------- | ----------- | ---------- | ----------------- |
| NFR-001 | As a user I upload avatar | Security-InputValidation | Reject payloads >1 MiB; MIME из allowlist; **extra** поля запрещены | Защита от DoS/грязных входных данных | **Given** тело 2 MiB и неизвестные поля<br>**When** POST `/api/files/avatar`<br>**Then** 413 с RFC7807 + запрет extra | test: `e2e-upload-limit`; policy: schema/size | #123               | team-a | Proposed | P2 - Medium | S2 - Major | limits,validation |
| NFR-002 | As a client I call /api/* | Performance              | P95 ≤ 300 ms при 50 RPS в течение 5 мин                             | UX/SLO                               | **Given** сервис здоров<br>**When** 50 RPS на `/api/*` 5 минут<br>**Then** P95 ≤ 300 ms; error rate ≤ 1%              | test: `load-50rps`; log: latency quantiles    | #124               | team-a | Draft    | P1 - High   | S2 - Major | perf,slo          |

> Продолжайте ниже добавлять строки до достижения **8-10 NFR** (или больше, если нужно):

| ID | User Story / Feature | Category | Requirement (NFR) | Rationale / Risk | Acceptance (G-W-T) | Evidence (test/log/scan/policy) | Trace (issue/link) | Owner | Status | Priority    | Severity   | Tags |
| -- | -------------------- | -------- | ----------------- | ---------------- | ------------------ | ------------------------------- | ------------------ | ----- | ------ | ----------- | ---------- | ---- |
|  NFR-003  |    Поиск по каталогу                  |   Performance       |       Среднее время ответа API ≤ 300 мс при 95-м перцентиле для запросов до 50 символов            |        Медленный поиск снижает UX и конверсию.          |         **Given** пользователь делает GET /api/search?q=текст (≤50 симв.) при нормальной нагрузке **When** сервис обрабатывает 1000 RPS **Then** 95 % ответов ≤ 300 мс           |               test: `search_p95.yaml`, APM-метрики                  |                   |    team-search   | Proposed  | P2 - Medium | S2 - Major |   performance, search   |
| NFR-004 |	Поиск по каталогу | RateLimiting |	Ограничение 100 запросов в минуту на IP/токен с ответом 429 при превышении	| Защита от DoS и чрезмерного потребления API. |	**Given** клиент превышает 100 запросов/мин **When** отправляется 101-й запрос **Then** API отвечает 429 с Retry-After	| test: `ratelimit_search`; NGINX/Envoy конфиг |	|	team-search	| Proposed |	P2 - Medium	| S2 - Major |	ratelimiting, api-gateway |
| NFR-005 |	Поиск по каталогу | Timeouts/Retry	| API должен иметь timeout ≤ 2 с и не более 2 автоматических retry с экспоненциальной задержкой	| Бесконечные запросы перегружают backend и клиентов.	| **Given** upstream не отвечает **When** клиент делает GET /api/search **Then** запрос завершается за ≤2 с с понятной ошибкой и не более 2 повторов	| unit: `retry_config_test`; policy: http-client.yaml	| 	| team-search |	Proposed |	P2 - Medium |	S3 - Minor |	timeout, retry, stability |
| NFR-006 |	Оформление заказа |Security-AuthZ/RBAC |	Только аутентифицированные пользователи с ролью “customer” могут создавать заказы | Защита от несанкционированных покупок.	| **Given** неавторизованный пользователь **When** POST /api/orders **Then** 401 Unauthorized |	test: `order_authz`; scan: RBAC policy |	 |	team-payments	| Proposed	| P1 - High	| S1 - Critical	| security, authz, orders |
| NFR-007 |	Оформление заказа | Data-Integrity	| Никакие поля заказа не могут быть изменены после успешной оплаты	| Исключение мошенничества и расхождений данных.	| **Given** заказ оплачен **When** отправляется PUT/PATCH для его изменения **Then** API возвращает 409 Conflict | test: `order_integrity`; DB immutability policy	| |	team-payments |	Proposed |	P1 - High	| S1 - Critical	 | integrity, data,orders |
| NFR-008 |	Оформление заказа | Auditability	| Все изменения статуса заказа и платежа должны логироваться с userID и временем	| Для расследований и бухгалтерии.	| **Given** изменение статуса заказа **When** POST /api/payments/webhook приходит **Then** создаётся аудиторская запись с userID, временем, старым и новым статусом	| audit-log: `orders_audit.log`; policy: audit.yaml |	 | team-payments | Proposed |	P2 - Medium |	S2 - Major	|audit, compliance |
| NFR-009 |	Оформление заказа | Timeouts/Retry	| Вебхуки платежей повторяются не более 3 раз с экспоненциальной задержкой, timeout ≤ 5 с	| Избежать дублирования и зависаний при сетевых сбоях.	| **Given** сбой доставки вебхука **When** платёжный провайдер шлёт уведомление **Then** сервер повторяет до 3 раз, максимум 5 с ожидания за попытку | test: `webhook_retry`; config: webhook.yaml |	 |	team-payments |	Proposed |	P2 - Medium	| S3 - Minor |	timeout, retry, payments |
| NFR-010 |	Оформление заказа | Idempotency	| POST /api/orders и /api/payments/session должны быть идемпотентными по client_token	| Предотвращение двойного создания заказов и платежей при повторной отправке.	| **Given** два одинаковых POST с одним client_token **When** запросы приходят почти одновременно **Then** создаётся один заказ/сессия и возвращается одинаковый ответ	| test: `idempotency_orders`; policy: http-idempotency.md	|  |	team-payments |	Proposed |	P1 - High |	S2 - Major |	idempotency, orders, payments |

---

## Памятка по заполнению

* **Измеримость.** В `Requirement` фиксируйте числа и границы (мс, RPS, минуты, MiB, коды 4xx/5xx, CVSS).
* **Проверяемость.** В `Acceptance (G-W-T)` используйте объективные условия и наблюдаемые факты (код ответа, квантиль, наличие заголовка, запись в лог).
* **Связность.** Сверяйте, чтобы NFR не конфликтовали (timeouts vs retry, rate limits vs SLO, privacy vs logging).
* **План проверки.** В `Evidence` укажите, чем это будет подтверждаться позже (test/log/scan/policy). В рамках семинара **реальные артефакты не требуются**.
* **Трассировка.** В `Trace` добавляйте ссылки на Issues/документы, чтобы потом не искать контекст.

---

## После семинара

* Перенесите/доработайте **8-10 утверждённых NFR** (по сути, те же строки) в раздел **NFR** вашего `GRADING/TM.md`.
* На S04-S05 свяжете эти NFR с угрозами (STRIDE) и ADR - по ID.
