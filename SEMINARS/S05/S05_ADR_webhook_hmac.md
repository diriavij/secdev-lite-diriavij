# ADR-02: Webhook Authentication via HMAC-SHA256

## 1) Title
Webhook Authentication via HMAC-SHA256 with Replay Protection

## 2) Status
**Status:** `Proposed`  
**Owner:** Payments Team  
**Date created:** 2024-01-20  
**Last updated:** 2024-01-20

## 3) Context

**Risk:** R-04 "Отсутствие проверки HMAC/replay для webhook → подделка уведомлений о платеже"  
**L×I Score:** L=4, I=5, Score=**20** (Top-4 priority)

**DFD Element:** PayProvider → API (HTTPS webhook callback)

**NFR Links:** NFR-008 (Auditability), NFR-WEBHOOK-AUTH, NFR-REPLAY-PROTECTION

**Current State:**
- Webhook endpoint `/api/v1/payments/webhook` публично доступен без аутентификации
- Любой может отправить POST → подделка статусов платежей
- Отсутствует защита от replay attacks

**Business Impact:**
- Прямые финансовые потери (отгрузка без оплаты)
- Compliance: PCI DSS требует защиты платёжных webhook

**Assumptions:**
- PayProvider поддерживает HMAC-SHA256
- Webhook endpoint должен оставаться публичным

---

## 4) Decision (что делаем)

Внедряем трёхуровневую защиту webhook:

### Layer 1: HMAC-SHA256 Signature Verification

**Параметры:**
- Algorithm: HMAC-SHA256
- Secret: от PayProvider (хранится в secret manager)
- Signature location: HTTP header `X-Webhook-Signature`
- Format: `sha256=<hex_digest>`
- Signing payload: `{timestamp}.{raw_body}`

```python
payload = f"{timestamp}.{request.body}"
expected_signature = hmac.new(
    webhook_secret.encode(),
    payload.encode(),
    hashlib.sha256
).hexdigest()

if not hmac.compare_digest(f"sha256={expected_signature}", 
                           request.headers["X-Webhook-Signature"]):
    raise Unauthorized("Invalid signature")
```

### Layer 2: Timestamp Window (Replay Protection)

**Параметры:**
- Timestamp location: `event.created` в теле webhook
- Acceptance window: ±5 минут от server time

```python
event_time = datetime.fromisoformat(webhook_body["event"]["created"])
delta = abs((datetime.utcnow() - event_time).total_seconds())
if delta > 300:  # 5 minutes
    raise Forbidden("Webhook timestamp expired")
```

### Layer 3: Nonce Deduplication

**Параметры:**
- Nonce: `event.id` в теле webhook
- Storage: Redis (key: `webhook:nonce:{event_id}`, TTL: 10 минут)

```python
if redis.exists(f"webhook:nonce:{event_id}"):
    raise Conflict("Duplicate webhook (replay detected)")
redis.setex(f"webhook:nonce:{event_id}", 600, "processed")
```

**Defence in Depth:** IP whitelist как дополнительный слой (Nginx/CloudFlare)

---

## 5) Alternatives (рассмотренные и отклонённые)

**Alt B: IP Whitelist только**
- Отклонено: Sec.Impact=3, легко обойти через compromised server
- Net=0 vs +4 для HMAC

**Alt C: OAuth2 Client Credentials**
- Отклонено: Нестандартно для webhook, Complex=4, Deps=4 (изменения на стороне провайдера)
- Net=-3

---

## 6) Consequences (последствия)

**Положительные:**
+ Криптографическая защита (невозможность подделки без секрета)
+ Replay protection (timestamp + nonce)
+ Industry standard (Stripe, GitHub, PayPal)
+ Соответствие PCI DSS

**Негативные:**
- Latency overhead: +10-15ms (HMAC + Redis lookup)
- Redis dependency для nonce cache
- Secret management (rotation при компрометации)
- Clock synchronization требование (NTP)

---

## 7) DoD / Acceptance

### Scenario 1: Valid Webhook
```gherkin
Given: PayProvider отправляет webhook с валидной HMAC
When: POST /api/v1/payments/webhook
Then:
  - 200 OK
  - Логируется `webhook.received` (event_id, signature_valid:true)
  - Nonce сохранён в Redis (TTL=10 мин)
```

### Scenario 2: Invalid Signature
```gherkin
Given: Атакующий отправляет webhook с неправильной подписью
When: POST /api/v1/payments/webhook
Then:
  - 401 Unauthorized
  - Response: {"error": "invalid_signature"}
  - Логируется `webhook.signature_invalid` + alert
```

### Scenario 3: Replay Attack
```gherkin
Given: Легитимный webhook уже обработан (nonce в Redis)
When: Повторная отправка с тем же event_id
Then:
  - 409 Conflict
  - Логируется `webhook.replay_detected` + alert
```

### Scenario 4: Expired Timestamp
```gherkin
Given: Webhook с валидной подписью, но timestamp > 5 минут
When: POST /api/v1/payments/webhook
Then:
  - 403 Forbidden
  - Логируется `webhook.timestamp_invalid`
```

### Проверки:
| Проверка | Критерий | Инструмент |
|----------|----------|------------|
| test: `test_valid_webhook_signature()` | 200 OK + event processed | pytest |
| test: `test_invalid_signature_rejected()` | 401 | pytest |
| test: `test_replay_attack_blocked()` | 409 Conflict | pytest |
| test: `test_expired_timestamp_rejected()` | 403 | pytest |
| log: `webhook.received`, `webhook.replay_detected` | Contains event_id | ELK |
| scan: Webhook secret in code | 0 findings | trufflehog |
| metric: `/payments/webhook` latency | p95 < 100ms | Prometheus |

---

## 8) Rollback / Fallback

**Feature Flag:** `WEBHOOK_HMAC_VALIDATION_ENABLED` (env var)

```python
if not os.getenv("WEBHOOK_HMAC_VALIDATION_ENABLED") == "true":
    log_warning("Webhook HMAC validation DISABLED")
    return True  # Legacy mode
```

**Rollback Steps:**
1. Set `WEBHOOK_HMAC_VALIDATION_ENABLED=false`
2. Deploy config
3. Monitor: webhook rejection rate → 0%

**Graceful Degradation:**
- Redis unavailable → **Fail-closed** (reject webhook, 409), alert ops

---

## 9) Trace

- **DFD:** `SEMINARS/S04/S04_dfd.md` - PayProvider → API edge
- **STRIDE:** `SEMINARS/S04/S04_stride_matrix.md` - Row "Edge: PayProvider → API", Threats S, T, R
- **Risk:** `SEMINARS/S04/S04_risk_scoring.md` - R-04 (Score=20, Rank #4)
- **NFR:** `SEMINARS/S03/S03_register.md` - NFR-008 (Auditability)

---

## 10) Ownership & Dates

**Owner:** Payments Team (@payments-lead)  
**Reviewers:** Security Team (approved), Infrastructure Team (approved)  
**Date created:** 2024-01-20  
**Target implementation:** Sprint 24

---

## 11) Open Questions

1. **Secret Rotation:** Как часто? Процесс координации с PayProvider?
   - Proposed: 1 раз в год, немедленно при leak

2. **Rate Limiting:** Нужен ли rate limit на webhook endpoint?
   - Proposed: 100 req/min per source IP (CloudFlare)

3. **Multi-Provider Support:** Разные форматы подписей у провайдеров?
   - Proposed: Factory pattern для выбора verifier по `X-Provider` header

---

## 12) Appendix

### Config:
```yaml
webhook:
  hmac:
    enabled: true
    algorithm: sha256
    header_name: "X-Webhook-Signature"
    secret_key_name: "WEBHOOK_SECRET_PAYPROVIDER"
  timestamp:
    tolerance_seconds: 300  # ±5 minutes
  nonce:
    redis_prefix: "webhook:nonce:"
    ttl_seconds: 600  # 10 minutes
```

### Middleware Pseudocode:
```python
def verify_webhook(request):
    # Layer 1: HMAC
    signature = request.headers["X-Webhook-Signature"]
    timestamp = request.json["event"]["created"]
    payload = f"{timestamp}.{request.body.decode()}"
    expected = hmac.new(secret, payload.encode(), hashlib.sha256).hexdigest()
    if not hmac.compare_digest(signature[7:], expected):
        raise Unauthorized("Invalid HMAC")
    
    # Layer 2: Timestamp
    delta = abs((datetime.utcnow() - parse(timestamp)).total_seconds())
    if delta > 300:
        raise Forbidden("Timestamp expired")
    
    # Layer 3: Nonce
    event_id = request.json["event"]["id"]
    if redis.exists(f"webhook:nonce:{event_id}"):
        raise Conflict("Replay detected")
    redis.setex(f"webhook:nonce:{event_id}", 600, "processed")
```