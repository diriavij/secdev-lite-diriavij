# Матрица вариантов для R-02

## 0) Контекст риска

* **Risk-ID:** R-02 "Ошибки в валидации/авторизации заказа → создание/модификация заказов другими пользователями"
* **Threat:** T (Tampering), E (Elevation of Privilege)
* **DFD element/edge:** API → OrdersSvc → DB (Order Create/Update flow)
* **NFR link:** NFR-006 (Security-AuthZ/RBAC), NFR-007 (Data-Integrity), NFR-001 (InputValidation)
* **L×I:** L=4, I=4, Score=16
* **Ограничения/предпосылки:**
  - Multi-tenant система (каждый пользователь в своём tenant)
  - Текущая авторизация: проверка JWT, но нет проверки tenant_id на уровне ресурсов
  - Бизнес-требование: пользователь видит только свои заказы
  - Обнаружена IDOR уязвимость: `GET /api/orders/{order_id}` не проверяет владельца

---

## 1) Критерии и шкалы (1-5)

**Польза (↑):**
- **Security impact (↑):** предотвращение межтенантных утечек/подмен
- **Blast radius reduction (↑):** ограничение IDOR/BOLA атак

**Стоимость (↓):**
- **Complexity (↓):** сложность внедрения
- **Time-to-mitigate (↓):** время до закрытия уязвимости
- **Dependencies (↓):** зависимости от других команд/систем

---

## 2) Таблица сравнения вариантов

| Alt | Summary | Sec.Impact (↑,1-5) | Blast (↑,1-5) | Complex (↓,1-5) | Time (↓,1-5) | Deps (↓,1-5) | Benefit | Cost | Net | Notes |
|-----|---------|-------------------:|-------------:|----------------:|-------------:|-------------:|--------:|-----:|----:|-------|
| A | Tenant-scoped queries + RBAC на уровне API + Input validation | 5 | 5 | 3 | 3 | 2 | 10 | 8 | **+2** | Комплексное решение, требует рефакторинга |
| B | Database Row-Level Security (RLS) policies | 4 | 4 | 4 | 4 | 3 | 8 | 11 | **-3** | Защита на уровне DB, но сложнее отладка |
| C | API Gateway Policy Engine (OPA/Casbin) | 4 | 4 | 5 | 4 | 5 | 8 | 14 | **-6** | Требует внедрения новой инфраструктуры |

**Расшифровка оценок:**

### Alt A: Tenant-scoped queries + RBAC + Validation
- Sec.Impact=5: Комплексная защита на всех уровнях (API, Service, DTO)
- Blast=5: Невозможность IDOR даже при обходе одного контроля
- Complex=3: Требует изменения всех DAO методов + middleware
- Time=3: 1-2 недели (рефакторинг ~15 endpoints)
- Deps=2: Только внутренняя команда

### Alt B: Database RLS
- Sec.Impact=4: Сильная защита, но требует правильной настройки сессий
- Blast=4: Защита даже при ошибках в коде приложения
- Complex=4: Миграция схемы, политики для каждой таблицы
- Time=4: 2-3 недели (тестирование всех запросов)
- Deps=3: Требует PostgreSQL 9.5+, изменения в connection pooling

### Alt C: Policy Engine (OPA)
- Sec.Impact=4: Централизованная авторизация
- Blast=4: Единая точка управления политиками
- Complex=5: Внедрение OPA sidecar, миграция логики авторизации
- Time=4: 3-4 недели
- Deps=5: Новая инфраструктура, обучение команды Rego

---

## 3) Тай-брейкеры при равенстве Net

| Критерий | A (Tenant-scoped+RBAC) | B (DB RLS) | C (OPA Gateway) |
|----------|------------------------|------------|-----------------|
| **Compliance** | Явный контроль в коде | Defence in depth | Audit trail |
| **Maintainability** | Понятная логика в коде | Сложнее отладка (скрыто в DB) | Требует знания Rego |
| **Team fit** | Команда знает Python/FastAPI | Средний опыт с RLS | Нет опыта с OPA |
| **Observability** | Логирование в application layer | DB logs менее структурированы | Policy decision logs |
| **Rollback safety** | Feature flag per endpoint | Миграция схемы сложно откатить | Можно отключить OPA, но рискованно |

---

## 4) Решение (для переноса в ADR)

* **Chosen alternative:** **A - Tenant-scoped Queries + RBAC + Input Validation**

* **Почему:**
  1. Положительный Net (+2) vs отрицательные у B и C
  2. Команда владеет технологией (Python/FastAPI)
  3. Явный контроль в application layer (легче отладка)
  4. Можно внедрять инкрементально (endpoint by endpoint)
  5. Feature flags для безопасного rollout
  6. Не требует новой инфраструктуры

* **ADR candidate:** "Tenant-Scoped Authorization with RBAC and Input Validation"

* **Связки:**
  - Risk: R-02 (L=4, I=4, Score=16)
  - NFR: NFR-006 (AuthZ/RBAC), NFR-007 (Data-Integrity), NFR-001 (InputValidation)
  - DFD: API → OrdersSvc → DB
  - STRIDE: Rows "Node: API" (Threat E), "Edge: API → OrdersSvc" (Threat T)

* **Следующие шаги (минимум):**
  1. **Middleware:** Извлекать `tenant_id` и `user_id` из JWT, добавлять в request context
  2. **DAO Layer:** Рефакторинг всех запросов к таблице `orders`:
     ```sql
     SELECT * FROM orders WHERE tenant_id = :tenant_id AND user_id = :user_id
     ```
  3. **DTO Validation:** Pydantic schemas с `Field(exclude=True)` для `tenant_id`
  4. **RBAC Implementation:**
     - `customer`: read/write только свои заказы
     - `manager`: read все заказы своего tenant
     - `admin`: full access (с аудитом)
  5. **Audit Logging:** События `order.access_denied` с `requested_tenant_id` и `actual_tenant_id`
  6. **Integration Tests:** Тесты на IDOR для всех endpoints
  7. **Feature Flag:** `TENANT_SCOPED_AUTH_ENABLED` (per service)

---

## 5) Самопроверка

- [x] Риск и контекст из S04 указаны
- [x] Сравнили 3 альтернативы
- [x] Все критерии заполнены, Net рассчитан
- [x] Тай-брейкеры учтены
- [x] Решение принято с обоснованием
- [x] ADR candidate сформулирован
