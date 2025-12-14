# S04 — STRIDE Pattern Bank

## Edge: Internet → API (публичная поверхность)

| Element | Data / Boundary | Threat | Description | NFR link (ID) | Mitigation idea |
|---------|------------------|--------|-------------|---------------|-----------------|
| Edge: Internet → API | JWT / public | S | JWT replay/forgery на публичных эндпоинтах | NFR-AuthN, NFR-RateLimit | Короткий JWT TTL + refresh; проверка iss/aud/exp/nbf; rate limit /auth/* |
| Edge: Internet → API | mTLS/Client binding | S | Отсутствует mTLS/привязка клиента для интеграционных ключей | NFR-AuthN | mTLS для машинных клиентов, IP binding |
| Edge: Internet → API | JSON / Query | T | Подмена параметров/JSON-инъекция в невалидированном теле запроса | NFR-InputValidation | Pydantic-схемы, строгий content-type, лимиты размера тела |
| Edge: Internet → API | File uploads | T | Directory traversal при загрузке файлов | NFR-InputValidation | Запрет относительных путей, генерация имён, MIME типизация |
| Edge: Internet → API | HTTP | R | Нет traceability (нет correlation_id, логов логинов/ошибок) | NFR-Observability/Logging | correlation_id, структурные логи |
| Edge: Internet → API | HTTP | I | Подробные ошибки/stacktrace наружу | NFR-API-Contract/Errors | RFC7807 без чувствительных деталей |
| Edge: Internet → API | CORS/Headers | I | Слабый CORS/утечка через Referrer | NFR-Privacy/PII, NFR-API-Contract | строгий CORS; Referrer-Policy: no-referrer |
| Edge: Internet → API | Request | D | Нет rate-limit/body-limit/timeout | NFR-RateLimit | лимиты RPS/burst, body-size limit, server timeouts |
| Edge: Internet → API | Tenant / Resource ID | E | IDOR/недостаточные проверки авторизации | NFR-AuthZ/RBAC, NFR-TenantIsolation | Проверка владения ресурсом; tenant scoping |

---

## Node: API / Controller

| Element | Data / Boundary | Threat | Description | NFR link (ID) | Mitigation idea |
|---------|------------------|--------|-------------|---------------|-----------------|
| Node: API | Internal | S | Проброс доверия внутренним сервисам без проверки токена сервиса | NFR-AuthN | Сервисные токены/mTLS |
| Node: API | Headers | T | Header/host injection при проксировании | NFR-InputValidation | Нормализация заголовков, фиксированный upstream host |
| Node: API | Logs | R | Логи без user/tenant/route | NFR-Observability/Logging | шаблон логов (user_id, tenant_id, route, latency) |
| Node: API | Logs / traces | I | PII в логах/метриках | NFR-Privacy/PII | Маскирование, запрет PII в trace |
| Node: API | Worker pool | D | Безграничный пул/воркеры | NFR-DoS/Resilience | Лимиты concurrency/queue, back-pressure |
| Node: API | Admin routes | E | Debug/админ ручки без RBAC | NFR-AuthZ/RBAC | Feature-flag + RBAC, защита от сканирования маршрутов |

---

## Node: Service / Бизнес-логика

| Element | Data / Boundary | Threat | Description | NFR link (ID) | Mitigation idea |
|---------|------------------|--------|-------------|---------------|-----------------|
| Node: Service | External auth | S | Отсутствует взаимная аутентификация к DB/внешним | NFR-AuthN | выделенные учётки, mTLS |
| Node: Service | Shell / Command | T | Command/OS-инъекция при вызове shell-утилит | NFR-InputValidation | subprocess без shell, whitelist |
| Node: Service | SQL | T | SQL-конкатенация (динамические фильтры) | NFR-Data-Integrity | параметризация, whitelist диапазонов |
| Node: Service | Audit | R | Нет бизнес-аудита ключевых действий | NFR-Audit | Audit trail, неизменяемые логи |
| Node: Service | Config | I | Секреты в коде/конфиге | NFR-Config/Secrets | Secret-store, env, ревью на утечки |
| Node: Service | External calls | D | Нет timeout/retry/circuit breaker | NFR-Timeouts/Retry/CB | timeout ≤2s, retry ≤3, CB |
| Node: Service | Flags | E | Обход бизнес-контролей (feature-flags/параметры) | NFR-AuthZ/RBAC | серверные гварды, immutable флаги |

---

## Node: DB / Хранилище

| Element | Data / Boundary | Threat | Description | NFR link (ID) | Mitigation idea |
|---------|------------------|--------|-------------|---------------|-----------------|
| Node: DB | Credentials | S | Учётка приложения с избыточными правами | NFR-AuthZ/RBAC | least-privilege user, запрет DDL/DROP |
| Node: DB | Data | T | Tampering данных/схемы без целостности | NFR-Data-Integrity | транзакции, FK, checksum |
| Node: DB | Audit | R | Нет аудит-логов доступа/админ-операций | NFR-Audit | audit logs, отдельное хранение |
| Node: DB | PII | I | PII без маскирования/шифрования/бэкапы открыты | NFR-Privacy/PII | колоночное шифрование, маскирование |
| Node: DB | Queries | D | Долгие запросы/нет индексов | NFR-DoS/Resilience | индексы, пагинация |
| Node: DB | Row-level | E | Обход row-level security/тенант-изоляции | NFR-TenantIsolation | RLS политики, tenant filters |

---

## Edge: Service → External API

| Element | Data / Boundary | Threat | Description | NFR link (ID) | Mitigation idea |
|---------|------------------|--------|-------------|---------------|-----------------|
| Edge: Service → External | TLS | S | Доверие внешнему без проверки (pinning/issuer) | NFR-AuthN | TLS pinning, CA check |
| Edge: Service → External | HTTP | T | Подмена/искажение ответа при слабом TLS/валидации | NFR-API-Contract | строгая проверка схем/подписей |
| Edge: Service → External | Idempotency | R | Нет отзыв-идентификаторов/idempotency | NFR-Audit | idempotency-key, логирование |
| Edge: Service → External | Query/Logs | I | Передача PII/секретов в query/логи | NFR-Privacy/PII | только в headers/body, redaction |
| Edge: Service → External | Timeouts | D | Нет retry/CB/лимитов | NFR-Timeouts/Retry/CB | разумные retry, CB, таймауты |
| Edge: Service → External | OAuth | E | Ошибки в OAuth (over-scoped токен) | NFR-AuthZ/RBAC | минимальные scope, ротация |

---

## Edge: Events / Queue (опционально)

| Element | Data / Boundary | Threat | Description | NFR link (ID) | Mitigation idea |
|---------|------------------|--------|-------------|---------------|-----------------|
| Edge: Queue | Producers/Consumers | S | Спуфинг продюсера/консюмеров | NFR-AuthN | аутентификация клиентов, ACL |
| Edge: Queue | Messages | T | Тамперинг сообщений без подписи | NFR-Data-Integrity | схема-регистры, подписи |
| Edge: Queue | Events | R | Нет аудита публикаций/доставки | NFR-Audit | лог поставки/неудач, DLQ мониторинг |
| Edge: Queue | Payload | I | PII в событиях без маскирования | NFR-Privacy/PII | редактирование, политика ретенции |
| Edge: Queue | Flow | D | Back-pressure/переполнение очереди | NFR-DoS/Resilience | квоты, лимиты, алерты |
| Edge: Queue | Admin | E | Доступ к админ-топикам | NFR-AuthZ/RBAC | разделение ролей, запрет админ-доступа |

---

## Общие паттерны на границах доверия

| Element | Data / Boundary | Threat | Description | NFR link (ID) | Mitigation idea |
|---------|------------------|--------|-------------|---------------|-----------------|
| Boundary | Trust Zones | S | Отсутствует чёткая граница доверия | — | Разграничить Internet/Service/External, разделить учётки |
| Boundary | Correlation | R | Нет сквозного correlation_id через границы | NFR-Observability/Logging | прокидывать header, логировать |
| Boundary | Logs | I | «Шумные» логи на границах содержат PII | NFR-Privacy/PII | маскирование ingress/egress |
| Boundary | Rate limiting | D | Нет лимитов между зонами | NFR-RateLimit | rate-limit, очереди, CB |
