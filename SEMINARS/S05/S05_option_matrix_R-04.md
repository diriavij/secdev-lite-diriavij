# Матрица вариантов для R-04

## 0) Контекст риска

* **Risk-ID:** R-04 "Отсутствие проверки HMAC/replay для webhook → подделка уведомлений о платеже"
* **Threat:** S (Spoofing), T (Tampering)
* **DFD element/edge:** PayProvider → API (HTTPS webhook callback)
* **NFR link:** NFR-008 (Auditability), NFR-WEBHOOK-AUTH, NFR-REPLAY
* **L×I:** L=4, I=5, Score=20
* **Ограничения/предпосылки:**
  - Webhook endpoint публичный (PayProvider должен дотянуться)
  - Критичность: фейковые платежи = финансовые потери
  - PayProvider поддерживает HMAC-SHA256 подписи
  - Текущее состояние: webhook принимает любые запросы без проверки

---

## 1) Критерии и шкалы (1-5)

**Польза (↑):**
- **Security impact (↑):** предотвращение поддельных/модифицированных webhook
- **Blast radius reduction (↑):** ограничение финансовых потерь

**Стоимость (↓):**
- **Complexity (↓):** сложность внедрения
- **Time-to-mitigate (↓):** время до защиты
- **Dependencies (↓):** зависимости от провайдера/инфраструктуры

---

## 2) Таблица сравнения вариантов

| Alt | Summary | Sec.Impact (↑,1-5) | Blast (↑,1-5) | Complex (↓,1-5) | Time (↓,1-5) | Deps (↓,1-5) | Benefit | Cost | Net | Notes |
|-----|---------|-------------------:|-------------:|----------------:|-------------:|-------------:|--------:|-----:|----:|-------|
| A | HMAC-SHA256 + Timestamp + Nonce (Redis dedup) | 5 | 5 | 2 | 2 | 2 | 10 | 6 | **+4** | Индустриальный стандарт, PayProvider поддерживает |
| B | IP Whitelist только для PayProvider | 3 | 2 | 1 | 1 | 3 | 5 | 5 | **0** | IP могут меняться, легко обойти через compromised server |
| C | OAuth2 Client Credentials для webhook | 4 | 4 | 4 | 3 | 4 | 8 | 11 | **-3** | Нестандартно для webhook, не все провайдеры поддерживают |

**Расшифровка оценок:**

### Alt A: HMAC + Timestamp + Nonce
- Sec.Impact=5: Криптографическая подпись, невозможность подделки без секрета
- Blast=5: Replay protection через timestamp (окно ±5 мин) + nonce deduplication
- Complex=2: Стандартная библиотека hmac, добавить Redis для nonce
- Time=2: 2-3 дня разработки + тестирования
- Deps=2: Redis для nonce cache (уже есть), секрет от PayProvider

### Alt B: IP Whitelist
- Sec.Impact=3: Защита от случайных/массовых атак, но не от целевых
- Blast=2: IP можно подделать через compromised infrastructure
- Complex=1: Nginx/конфиг уровень
- Time=1: 1 день
- Deps=3: Зависимость от стабильности IP PayProvider

### Alt C: OAuth2 Client Credentials
- Sec.Impact=4: Сильная аутентификация
- Blast=4: Токен можно отозвать
- Complex=4: Нестандартный flow для webhook, требует поддержки от провайдера
- Time=3: 1-2 недели (координация с провайдером)
- Deps=4: Изменения на стороне PayProvider

---

## 3) Тай-брейкеры при равенстве Net

| Критерий | A (HMAC+Nonce) | B (IP Whitelist) | C (OAuth2) |
|----------|----------------|------------------|------------|
| **Compliance** | PCI DSS recommended | Недостаточно для финансовых транзакций | Сильная аутентификация |
| **Maintainability** | Ротация секрета 1 раз в год | Простой whitelist | Управление токенами |
| **Team fit** | Команда знает HMAC | Простой конфиг | Требует понимания OAuth2 для webhook |
| **Observability** | Логирование валидации, nonce hits | Только access logs | Token events |
| **Rollback safety** | Feature flag + fallback на IP whitelist | Простой rollback | Требует координации с провайдером |

---

## 4) Решение (для переноса в ADR)

* **Chosen alternative:** **A - HMAC-SHA256 + Timestamp Window + Nonce Deduplication**

* **Почему:**
  1. Максимальный Net (+4) - лучший баланс безопасности и сложности
  2. Индустриальный стандарт (Stripe, GitHub, PayPal используют HMAC)
  3. Криптографическая стойкость (невозможность подделки без секрета)
  4. Replay protection (timestamp ±5 мин + nonce cache)
  5. PayProvider уже поддерживает HMAC-SHA256
  6. Быстрая реализация (2-3 дня)
  7. Соответствует PCI DSS требованиям

* **ADR candidate:** "Webhook Authentication via HMAC-SHA256 with Replay Protection"

* **Связки:**
  - Risk: R-04 (L=4, I=5, Score=20)
  - NFR: NFR-008 (Auditability), NFR-WEBHOOK-AUTH
  - DFD: PayProvider → API (webhook boundary)
  - STRIDE: Row "Edge: PayProvider → API" / Threats S, T, R

* **Следующие шаги (минимум):**
  1. Получить webhook secret от PayProvider (хранить в secret manager)
  2. Реализовать middleware для проверки HMAC-SHA256 подписи в header `X-Webhook-Signature`
  3. Проверять timestamp в теле запроса (accept window: ±5 минут от server time)
  4. Внедрить nonce deduplication через Redis (TTL = 10 минут)
  5. Добавить логирование: `webhook.received`, `webhook.signature_valid`, `webhook.replay_detected`
  6. Реализовать endpoint для ручного replay webhook (admin only, для отладки)
  7. Feature flag: `WEBHOOK_HMAC_VALIDATION_ENABLED`
  8. Fallback: IP whitelist как второй уровень защиты (defence in depth)

---

## 5) Самопроверка

- [x] Риск и контекст из S04 указаны
- [x] Сравнили 3 альтернативы
- [x] Все критерии заполнены, Net рассчитан
- [x] Тай-брейкеры проверены
- [x] Решение принято и аргументировано
- [x] ADR candidate готов
