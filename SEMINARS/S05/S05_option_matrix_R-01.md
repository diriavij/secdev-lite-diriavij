# Матрица вариантов для R-01

## 0) Контекст риска

* **Risk-ID:** R-01 "Подделка/кража JWT → несанкционированный доступ"
* **Threat:** S (Spoofing)
* **DFD element/edge:** Client → API (HTTPS, JWT, AuthN)
* **NFR link:** NFR-006 (Security-AuthZ/RBAC), NFR-SEC-AUTHN, NFR-RATELIMIT, NFR-OBSERVABILITY
* **L×I:** L=5, I=5, Score=25
* **Ограничения/предпосылки:**
  - Stateless микросервисная архитектура
  - Публичный API без возможности mTLS для всех клиентов (мобильные приложения)
  - Требование быстрого внедрения (≤ 2 недели)
  - Текущий TTL токенов: 24 часа (слишком долго)

---

## 1) Критерии и шкалы (1-5)

**Польза (↑, чем больше - тем лучше):**
- **Security impact (↑):** насколько снижает риск компрометации токена
- **Blast radius reduction (↑):** насколько ограничивает окно атаки/ущерб

**Стоимость (↓, чем меньше - тем лучше):**
- **Complexity (↓):** сложность внедрения (код/конфиг/политики)
- **Time-to-mitigate (↓):** время до эффекта (дни/спринты)
- **Dependencies (↓):** внешние/внутренние зависимости

**Итоговые метрики:**
- Benefit = Security impact + Blast radius reduction
- Cost = Complexity + Time-to-mitigate + Dependencies
- Net = Benefit − Cost

---

## 2) Таблица сравнения вариантов

| Alt | Summary | Sec.Impact (↑,1-5) | Blast (↑,1-5) | Complex (↓,1-5) | Time (↓,1-5) | Deps (↓,1-5) | Benefit | Cost | Net | Notes |
|-----|---------|-------------------:|-------------:|----------------:|-------------:|-------------:|--------:|-----:|----:|-------|
| A | JWT Short TTL (15 min) + Refresh Token Flow | 4 | 4 | 2 | 2 | 2 | 8 | 6 | **+2** | Стандартный паттерн OAuth2, быстрая реализация |
| B | mTLS для всех клиентов | 5 | 5 | 5 | 5 | 5 | 10 | 15 | **-5** | Требует PKI, сертификаты на мобильных клиентах |
| C | Session-based auth + Redis (Stateful) | 4 | 4 | 4 | 3 | 4 | 8 | 11 | **-3** | Противоречит stateless архитектуре |

**Расшифровка оценок:**

### Alt A: JWT Short TTL + Refresh
- Sec.Impact=4: Окно атаки 15 мин вместо 24 часа
- Blast=4: Refresh token можно отозвать через blacklist
- Complex=2: Стандартные библиотеки (PyJWT), добавить 1 endpoint
- Time=2: 3-5 дней разработки + 2 дня тестирования
- Deps=2: Только Redis для blacklist (уже есть)

### Alt B: mTLS
- Sec.Impact=5: Взаимная аутентификация, максимальная защита
- Blast=5: Компрометация требует доступа к сертификату клиента
- Complex=5: PKI инфраструктура, управление сертификатами
- Time=5: 4-6 недель (создание CA, распространение сертификатов)
- Deps=5: Внешняя PKI, поддержка клиентами (мобильные приложения)

### Alt C: Session + Redis
- Sec.Impact=4: Сессии отзываются немедленно
- Blast=4: Контроль на сервере
- Complex=4: Изменение архитектуры (stateful), миграция
- Time=3: 2 недели (рефакторинг всех сервисов)
- Deps=4: Redis cluster для сессий, балансировка sticky sessions

---

## 3) Тай-брейкеры при равенстве Net

| Критерий | A (JWT+Refresh) | B (mTLS) | C (Session) |
|----------|-----------------|----------|-------------|
| **Compliance/Privacy** | Соответствует GDPR (right to be forgotten через blacklist) | Максимальная безопасность | Хранение сессий = PII в Redis |
| **Maintainability** | Стандартный паттерн, широкая поддержка | Сложное управление сертификатами | Дополнительная инфраструктура |
| **Team fit** | Команда знает JWT/OAuth2 | Нет опыта с PKI | Потребуется обучение stateful подходу |
| **Observability** | События refresh логируются | mTLS события в nginx/envoy | Сессии сложнее трассировать |
| **Rollback safety** | Feature flag на уровне TTL | Сложный откат (сертификаты у клиентов) | Требует миграции данных |

---

## 4) Решение (для переноса в ADR)

* **Chosen alternative:** **A - JWT Short TTL + Refresh Token Flow**

* **Почему:**
  1. Наилучший баланс Net (+2 vs -5 и -3)
  2. Быстрая реализация (≤ 1 неделя) vs требования курса
  3. Команда владеет технологией JWT
  4. Простой откат через feature flag
  5. Соответствует stateless архитектуре
  6. Стандартное решение (OAuth2 RFC 6749)

* **ADR candidate:** "JWT Short-Lived Access Tokens with Refresh Flow and Revocation"

* **Связки:**
  - Risk: R-01 (L=5, I=5, Score=25)
  - NFR: NFR-006 (Security-AuthZ/RBAC), NFR-SEC-AUTHN
  - DFD: Client → API (JWT authentication boundary)
  - STRIDE: Row "Edge: Client → API" / Threat S

* **Следующие шаги (минимум):**
  1. Установить access token TTL = **15 минут** (было 24 часа)
  2. Установить refresh token TTL = **30 дней** (httpOnly, secure cookie)
  3. Реализовать endpoint `POST /api/v1/auth/refresh` (обмен refresh → новый access)
  4. Настроить key rotation (kid rotation каждые **30 дней**)
  5. Внедрить blacklist для refresh tokens в Redis (TTL = 30 дней)
  6. Добавить логирование событий: `token.issued`, `token.refreshed`, `token.revoked` с correlation_id
  7. Реализовать endpoint `POST /api/v1/auth/logout` (blacklist refresh token)
  8. Feature flag: `AUTH_SHORT_TTL_ENABLED` (env var)

---

## 5) Самопроверка

- [x] Риск и контекст из S04 указаны (Risk-ID, DFD, NFR-ID, L×I)
- [x] Сравнили минимум 2 альтернативы (3 в данном случае)
- [x] Заполнены все критерии 1-5; посчитаны Benefit/Cost/Net
- [x] Описаны тай-брейкеры
- [x] Принято решение и зафиксирован ADR candidate
- [x] Готовы перенести решение в `S05_ADR_jwt_refresh.md`
