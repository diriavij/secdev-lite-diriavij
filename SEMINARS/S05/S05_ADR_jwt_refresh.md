# ADR-01: JWT Short-Lived Access Tokens with Refresh Flow

## 1) Title
JWT Short-Lived Access Tokens with Refresh Flow and Revocation

## 2) Status
**Status:** `Proposed`  
**Owner:** Security Team  
**Date created:** 2024-01-20  
**Last updated:** 2024-01-20

## 3) Context

**Risk:** R-01 "Подделка/кража JWT → несанкционированный доступ к заказам/платежам"  
**L×I Score:** L=5, I=5, Score=**25** (Top-1 priority)

**DFD Element:** Client → API (HTTPS, JWT verification, AuthN boundary)

**NFR Links:** NFR-006 (Security-AuthZ/RBAC), NFR-SEC-AUTHN, NFR-RATELIMIT, NFR-OBSERVABILITY

**Current State:**
- Access token TTL = 24 часа (слишком долгое окно атаки)
- Нет механизма отзыва токенов (logout не работает до истечения TTL)
- Bearer token без дополнительной защиты

**Assumptions:**
- Stateless микросервисная архитектура
- Публичный API для web/mobile приложений
- Невозможность mTLS для всех клиентов
- Существующая Redis инфраструктура

---

## 4) Decision (что делаем)

Внедряем двухтокеновую систему (access + refresh) с механизмом отзыва:

### Параметры:

1. **Access Token:**
   - TTL = **15 минут** (было: 24 часа)
   - Payload: `{user_id, tenant_id, roles, exp, nbf, iat, jti, iss, aud}`
   - Алгоритм: **RS256** (асимметричное подписание)
   - Key rotation: каждые **30 дней** (kid в JWKS)

2. **Refresh Token:**
   - TTL = **30 дней**
   - Хранение: **httpOnly, secure, SameSite=Strict** cookie
   - Алгоритм: **HS256**
   - One-time use: каждый refresh инвалидирует предыдущий

3. **Refresh Flow:**
   - Endpoint: `POST /api/v1/auth/refresh`
   - Rate limit: **5 запросов/минуту**

4. **Revocation (Blacklist):**
   - Redis: key `blacklist:{jti}`, TTL = refresh_ttl
   - Триггеры: logout, password change, admin revoke

5. **Clock Skew Tolerance:** ±60 секунд

**Boundaries:** Gateway Layer (проверка токена), Auth Service (выдача/отзыв), все защищённые endpoints

---

## 5) Alternatives (рассмотренные и отклонённые)

**Alt B: mTLS для всех клиентов**
- Отклонено: Complexity=5, Time=5, Dependencies=5 (PKI инфраструктура, мобильные клиенты)
- Net = -5 vs +2 для JWT

**Alt C: Session-based auth + Redis**
- Отклонено: Противоречит stateless архитектуре, Complex=4, Deps=4
- Net = -3

---

## 6) Consequences (последствия)

**Положительные:**
+ Окно атаки: 24 часа → 15 минут
+ Logout работает немедленно (через blacklist)
+ Соответствие GDPR, OWASP ASVS
+ Refresh token в httpOnly cookie (защита от XSS)
+ Key rotation без downtime

**Негативные:**
- Increased traffic: refresh каждые 15 мин (~4x больше запросов)
- Client complexity: обработка 401 + refresh retry logic
- Redis dependency для blacklist
- Latency overhead: ~5-10ms на проверку blacklist

---

## 7) DoD / Acceptance

### Scenario 1: Expired Access Token
```gherkin
Given: Access token истёк (exp + 1 минута)
When: GET /api/v1/orders
Then: 
  - 401 Unauthorized
  - RFC7807 response: type="token-expired", correlation_id
```

### Scenario 2: Successful Refresh
```gherkin
Given: Валидный refresh token в cookie
When: POST /api/v1/auth/refresh
Then:
  - 200 OK
  - Новые access + refresh tokens
  - Старый refresh в blacklist
  - Логируется `token.refreshed`
```

### Scenario 3: Logout Revocation
```gherkin
Given: Пользователь залогинен
When: POST /api/v1/auth/logout
Then:
  - Refresh token JTI в Redis blacklist (TTL=30 дней)
  - 204 No Content
  - Логируется `token.revoked`
```

### Проверки:
| Проверка | Критерий | Инструмент |
|----------|----------|------------|
| test: `test_expired_token_rejection()` | 401 + RFC7807 | pytest |
| test: `test_refresh_flow_success()` | New tokens, old blacklisted | pytest |
| test: `test_logout_revokes_token()` | Token in blacklist | pytest |
| log: `token.issued`, `token.refreshed` | Contains correlation_id | ELK |
| scan: Secret scanner | 0 findings | trufflehog |
| metric: `/auth/refresh` latency | p95 ≤ 200ms | Prometheus |

---

## 8) Rollback / Fallback

**Feature Flag:** `AUTH_SHORT_TTL_ENABLED` (env var)

```python
if os.getenv("AUTH_SHORT_TTL_ENABLED") == "true":
    ACCESS_TTL = 15 * 60  # 15 minutes
else:
    ACCESS_TTL = 24 * 3600  # Legacy: 24 hours
```

**Rollback Steps:**
1. Set `AUTH_SHORT_TTL_ENABLED=false`
2. Deploy config (no code deploy)
3. Monitor: 401 rate drops to baseline

**Graceful Degradation:**
- Redis unavailable → fail-open (no blacklist check), alert ops

---

## 9) Trace

- **DFD:** `SEMINARS/S04/S04_dfd.md` - Client → API edge
- **STRIDE:** `SEMINARS/S04/S04_stride_matrix.md` - Row "Edge: Client → API", Threat S
- **Risk:** `SEMINARS/S04/S04_risk_scoring.md` - R-01 (Score=25, Rank #1)
- **NFR:** `SEMINARS/S03/S03_register.md` - NFR-006, NFR-SEC-AUTHN

---

## 10) Ownership & Dates

**Owner:** Security Team (@security-lead)  
**Reviewers:** Tech Lead (approved), Security Architect (approved)  
**Date created:** 2024-01-20  
**Target implementation:** Sprint 24 (Week of 2024-02-01)

---

## 11) Open Questions

1. **Multi-device support:** Logout на одном устройстве → отзыв всех tokens?
   - Options: A) Глобальный logout, B) Device-specific
   - Decision needed: обсудить с Product Team

2. **Concurrent refresh requests:** Grace period 5 секунд или strict one-time use?
   - Proposed: Grace period (better UX for mobile)

3. **Audit log retention:** Как долго хранить `token.revoked` события?
   - Proposed: 90 дней

---

## 12) Appendix

### Config:
```yaml
jwt:
  access:
    ttl: 900s  # 15 minutes
    algorithm: RS256
  refresh:
    ttl: 2592000s  # 30 days
    algorithm: HS256
    cookie:
      httpOnly: true
      secure: true
      sameSite: "Strict"
  blacklist:
    redis:
      key_prefix: "blacklist:"
      ttl: 2592000
```

### Middleware Pseudocode:
```python
def verify_access_token(request):
    token = extract_bearer_token(request.headers["Authorization"])
    claims = jwt.decode(
        token,
        get_public_key(kid=token.header["kid"]),
        algorithms=["RS256"],
        leeway=60
    )
    request.state.user_id = claims["user_id"]
    request.state.tenant_id = claims["tenant_id"]
```
