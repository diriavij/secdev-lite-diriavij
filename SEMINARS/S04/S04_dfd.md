# S04 — Data Flow Diagram (DFD)

## Контекст

DFD описывает архитектуру сервиса на уровне контуров (Client ↔ API ↔ Service ↔ Storage ↔ External) с выделением **границ доверия** и **ключевых потоков данных**.
Эта диаграмма будет использоваться в STRIDE-анализе и приоритизации рисков L×I.

## Мини-DFD (Mermaid)

```mermaid
flowchart LR
  classDef boundary stroke-dasharray: 4 4,stroke:#888,color:#333

  subgraph Internet["Trust Boundary: Internet"]
    Client[Client / Front]
  end
  Internet:::boundary

  subgraph Service["Trust Boundary: Service"]
    API[API / Controller]
    OrdersSvc[Orders Service]
    PaySvc[Payment Service]
  end
  Service:::boundary

  subgraph Storage["Trust Boundary: Storage"]
    DB[(DB / RDBMS)]
  end
  Storage:::boundary

  subgraph External["Trust Boundary: External Integrations"]
    PayProvider[External Payment Provider]
  end
  External:::boundary

  %% ✅ Edge labels rewritten without commas and brackets
  Client -->|HTTPS JWT AuthN query correlation_id NFR-Security-AuthN RateLimit Observability| API
  API -->|HTTP response RFC7807 NFR-API-Contract| Client

  API -->|Order Create DTO PII Idempotency NFR-AuthZ Validation| OrdersSvc
  OrdersSvc -->|SQL Insert PII NFR-Privacy| DB
  PaySvc -->|Payment Init HTTPS NFR-Timeouts Retry CB| PayProvider
  PayProvider -->|Webhook HTTPS HMAC NFR-Webhook-Auth| API