# ADR-03: Tenant-Scoped Authorization with RBAC

## 1) Title
Tenant-Scoped Authorization with RBAC and Input Validation

## 2) Status
**Status:** `Proposed`  
**Owner:** Backend Team  
**Date created:** 2024-01-20  
**Last updated:** 2024-01-20

## 3) Context

**Risk:** R-02 "Ошибки в валидации/авторизации заказа → создание/модификация заказов другими пользователями"  
**L×I Score:** L=4, I=4, Score=**16** (Top-2 priority)

**DFD Element:** API → OrdersSvc → DB (Order Create/Update/Read operations)

**NFR Links:** NFR-006 (Security-AuthZ/RBAC), NFR-007 (Data-Integrity), NFR-001 (InputValidation)

**Current State:**
- **IDOR уязвимость:** `GET /api/v1/orders/{order_id}` проверяет JWT, но не владельца ресурса
- Отсутствие tenant scoping в DB запросах
- Пример атаки: User A (tenant=1001) может читать заказы User B (tenant=1002)

**Business Impact:**
- Утечка данных между пользователями
- Возможность подмены заказов/платежей
- Нарушение GDPR (data isolation)

**Assumptions:**
- Multi-tenant SaaS (каждый пользователь в одном tenant)
- JWT содержит `tenant_id` и `user_id`
- PostgreSQL database, FastAPI + SQLAlchemy

---

## 4) Decision (что делаем)

Внедряем трёхуровневую защиту авторизации:

### Layer 1: Middleware - Tenant Context Injection

```python
@app.middleware("http")
async def inject_tenant_context(request: Request, call_next):
    token_claims = request.state.jwt_claims
    request.state.tenant_id = token_claims["tenant_id"]
    request.state.user_id = token_claims["user_id"]
    request.state.roles = token_claims["roles"]
    response = await call_next(request)
    return response
```

### Layer 2: Service/DAO - Tenant-Scoped Queries

**Before (уязвимый):**
```python
def get_order_by_id(order_id: UUID) -> Order:
    return db.query(Order).filter(Order.id == order_id).first()
    # ❌ No tenant filtering!
```

**After (безопасный):**
```python
def get_order_by_id(order_id: UUID, tenant_id: str) -> Order:
    return (
        db.query(Order)
        .filter(Order.id == order_id)
        .filter(Order.tenant_id == tenant_id)  # ✅ Tenant scoping
        .first()
    )
```

**Affected:** ~15 endpoints (Orders, Payments, Cart)

### Layer 3: DTO Validation - Prevent Client-Supplied tenant_id

**Before (уязвимый):**
```python
class OrderCreateRequest(BaseModel):
    tenant_id: str  # ❌ Client can supply!
```

**After (безопасный):**
```python
class OrderCreateRequest(BaseModel):
    items: List[OrderItem]
    # tenant_id removed, injected from JWT context
    class Config:
        extra = "forbid"  # Reject extra fields
```

### Layer 4: RBAC

| Role | Permissions | Scope |
|------|-------------|-------|
| `customer` | CRUD own orders | `user_id == current_user_id` |
| `manager` | Read all orders in tenant | `tenant_id == current_tenant_id` |
| `admin` | Full access (with audit) | All tenants (requires MFA) |

**Error Handling:** 404 (not 403) при cross-tenant access → предотвращение information disclosure

---

## 5) Alternatives (рассмотренные и отклонённые)

**Alt B: Database Row-Level Security (RLS)**
- Отклонено: Complex=4, Time=4, сложнее отладка (логика в DB)
- Net=-3 vs +2 для application-layer

**Alt C: API Gateway Policy Engine (OPA)**
- Отклонено: Complex=5, Deps=5 (новая инфраструктура, обучение Rego)
- Net=-6

---

## 6) Consequences (последствия)

**Положительные:**
+ IDOR vulnerability eliminated
+ Defence in depth (3 уровня)
+ RBAC flexibility
+ Audit trail для межтенантных попыток
+ Incremental rollout (endpoint-by-endpoint)

**Негативные:**
- Refactoring effort: ~15 endpoints, ~30 DAO methods (1-2 недели)
- Performance overhead: +2-5ms latency (tenant_id проверка)
- Breaking changes: если клиенты передавали tenant_id
- Testing complexity: комбинации tenant/role

---

## 7) DoD / Acceptance

### Scenario 1: IDOR Prevention
```gherkin
Given: User A (tenant=1001), Order exists (owner=userB, tenant=1002)
When: User A делает GET /api/v1/orders/{order_B_id}
Then:
  - 404 Not Found (not 403)
  - Логируется `order.access_denied`:
    {user_id: userA, tenant_id: 1001, 
     requested_tenant_id: 1002}
```

### Scenario 2: Same-Tenant Access
```gherkin
Given: User A (tenant=1001, role=customer), Order (owner=userA)
When: GET /api/v1/orders/{order_id}
Then:
  - 200 OK with order data
  - DB query: WHERE tenant_id='1001' AND user_id='userA'
```

### Scenario 3: Manager Role
```gherkin
Given: Manager (tenant=1001, role=manager)
When: GET /api/v1/orders
Then:
  - Returns all orders in tenant 1001
  - DB query: WHERE tenant_id='1001' (no user_id filter)
```

### Scenario 4: DTO Validation
```gherkin
Given: User A (tenant=1001)
When: POST /api/v1/orders with body {"tenant_id": "1002", ...}
Then:
  - 422 Unprocessable Entity
  - Pydantic error: "extra fields not permitted"
```

### Проверки:
| Проверка | Критерий | Инструмент |
|----------|----------|------------|
| test: `test_idor_cross_tenant_blocked()` | 404, no data leak | pytest |
| test: `test_same_tenant_access_allowed()` | 200 OK | pytest |
| test: `test_manager_sees_all_tenant_orders()` | List contains same tenant only | pytest |
| test: `test_dto_rejects_tenant_id_injection()` | 422 validation error | pytest |
| log: `order.access_denied` | Contains requested_tenant_id | ELK |
| scan: SAST missing tenant_id | 0 findings | Semgrep |
| metric: IDOR attempt rate | < 0.1% baseline | Grafana |

---

## 8) Rollback / Fallback

**Feature Flag:** `TENANT_SCOPED_AUTH_ENABLED` (per service)

```python
def get_order_by_id(order_id: UUID, tenant_id: str = None):
    query = db.query(Order).filter(Order.id == order_id)
    if TENANT_SCOPED_AUTH_ENABLED and tenant_id:
        query = query.filter(Order.tenant_id == tenant_id)
    return query.first()
```

**Incremental Rollout:**
- Week 1: Orders service (10% traffic)
- Week 2: 50% + Payments service
- Week 3: Full rollout (100%)

---

## 9) Trace

- **DFD:** `SEMINARS/S04/S04_dfd.md` - API → OrdersSvc → DB
- **STRIDE:** `SEMINARS/S04/S04_stride_matrix.md` - Rows "Node: API" (E), "Edge: API → OrdersSvc" (T)
- **Risk:** `SEMINARS/S04/S04_risk_scoring.md` - R-02 (Score=16, Rank #2)
- **NFR:** `SEMINARS/S03/S03_register.md` - NFR-006, NFR-007, NFR-001

---

## 10) Ownership & Dates

**Owner:** Backend Team (@backend-lead)  
**Reviewers:** Security Team (approved), Product Team (approved)  
**Date created:** 2024-01-20  
**Target implementation:** Sprint 24-25 (2 weeks)

---

## 11) Open Questions

1. **Historical Data:** Старые записи без tenant_id?
   - Action: Data quality check, backfill перед rollout

2. **Customer Support Access:** Как support team помогает через tenant границы?
   - Proposed: Support role с audit trail + MFA

3. **Performance Impact:** tenant_id filtering на сложных запросах?
   - Action: Load testing, acceptance: p95 latency рост < 10%

---

## 12) Appendix

### Database Indexes:
```sql
CREATE INDEX idx_orders_tenant_user ON orders(tenant_id, user_id);
CREATE INDEX idx_orders_tenant_id ON orders(tenant_id, id);
```

### Semgrep Rule:
```yaml
rules:
  - id: missing-tenant-filter
    pattern: db.query($MODEL).filter($MODEL.id == $ID)
    message: "Missing tenant_id filter - IDOR risk"
    severity: ERROR
```

### Audit Log Schema:
```json
{
  "event_type": "order.access_denied",
  "user_id": "userA",
  "tenant_id": "1001",
  "requested_resource": {
    "id": "order-uuid",
    "owner_tenant_id": "1002"
  },
  "result": "denied"
}
```