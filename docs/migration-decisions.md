# Migration Decision Log

This document records the architectural decisions made when decomposing the food delivery monolith into microservices, and the reasoning behind each choice.

---

## Service boundary decisions

### Why four domain services?

The boundaries were drawn along **bounded contexts** — areas of the business where concepts have distinct meanings and change at different rates.

| Bounded context | Driver |
|---|---|
| Customer | Identity and auth are a universal cross-cutting concern; isolating them lets every other service remain stateless with respect to users |
| Restaurant | Menu management changes on the restaurant owner's schedule, independent of order activity |
| Order | Orders are the core transaction; they orchestrate across customers, restaurants, and deliveries |
| Delivery | Logistics have their own lifecycle (ASSIGNED → PICKED_UP → IN_TRANSIT → DELIVERED) and could be swapped for a third-party provider without touching order logic |

### Why is Discovery Server a separate service?

Eureka needs to be up before any other service registers. Keeping it isolated means it can be restarted without disturbing the rest of the cluster. It holds no business logic and its data is ephemeral (registrations re-populate on restart).

### Why is API Gateway a separate service?

The gateway is the only component the public internet touches. Separating it means:
- JWT validation is centralised — individual services trust the gateway's pre-validated requests
- Routing rules can be updated without redeploying any domain service
- Rate limiting, CORS, and request logging can be applied at one layer

---

## Communication pattern decisions

### Synchronous (Feign) for order placement validation

When a customer places an order, the order service **must** verify that:
1. The customer account exists and is active
2. The restaurant exists and is accepting orders
3. Every requested menu item is available and belongs to that restaurant

These checks are synchronous because the result must be known before the order is persisted. A failure in any check should immediately return an error to the customer — not eventually.

**Trade-off accepted:** If customer-service or restaurant-service is down, order placement fails. Circuit breakers limit the blast radius, and fallback factories return a 503 rather than hanging the request.

### Asynchronous (RabbitMQ) for delivery assignment

After an order is placed, creating the delivery record does **not** need to block the customer's response. The customer already has their order ID and a `PLACED` status. The delivery record appearing seconds later (via `OrderPlacedEvent`) is acceptable.

Using async here means:
- Delivery Service can be down without blocking order placement
- Delivery Service can be scaled or replaced independently
- Retry logic (DLQ + exponential backoff) handles transient failures without losing events

**Trade-off accepted:** The order status update to `CONFIRMED` arrives asynchronously. The customer sees `PLACED` first, then `CONFIRMED` a moment later. This eventual consistency is acceptable for a delivery flow.

### Asynchronous (RabbitMQ) for order cancellation

Cancellation follows the same pattern: the order is immediately marked `CANCELLED`, and an `OrderCancelledEvent` is published. Delivery Service consumes it and marks the delivery `FAILED`. This decouples the cancellation acknowledgement from the logistics side-effects.

---

## Database-per-service

Each service has its own database (separate PostgreSQL schema and connection). This was chosen because:

1. **Independent deployability** — a migration in `order_db` cannot break `restaurant_db`
2. **Technology freedom** — a future Delivery Service could swap to a different store (e.g. a graph DB for route optimisation) without touching other services
3. **Failure isolation** — a corrupt or locked table in one DB does not cascade

**Trade-off accepted:** Cross-service queries require Feign calls. For example, displaying `ownerName` on a restaurant response requires a call to customer-service at read time. This adds latency and a failure surface. It is mitigated by the best-effort fallback (returns `"unknown"` on Feign failure) so the read always completes.

---

## Security decisions

### Role-based access with JWT

Roles (`CUSTOMER`, `RESTAURANT_OWNER`, `SERVICE`) are embedded in the JWT at login time. The gateway validates the token on every request and rejects it if expired or invalid. Individual services trust the claims in the forwarded token — they do not re-verify with the customer service on every call.

**Why not sessions?** Stateless JWT allows horizontal scaling of every service without shared session storage.

### Internal service token

Service-to-service Feign calls carry an `X-Internal-Service-Token` header (shared secret from environment config). The gateway strips this header from all inbound public requests, so no external caller can forge a SERVICE-role request.

### ROLE_SERVICE endpoints are never exposed externally

Delivery endpoints (`POST /api/deliveries`, `PATCH /api/deliveries/{id}/status`) require `ROLE_SERVICE`. Since `ROLE_SERVICE` tokens are never issued to registered users and the gateway blocks the header, these endpoints are unreachable from outside the Docker network.

---

## What was not migrated (intentional omissions)

| Feature | Reason left out |
|---|---|
| Admin panel / role promotion UI | Role promotion done via direct SQL — acceptable for a lab; a real system would add an admin service |
| Payment service | Out of scope for this migration phase |
| Notification service | `DeliveryStatusEvent` is produced but only consumed by order-service; a notification service could subscribe to the same exchange |
| Search / discovery beyond basic filters | Elasticsearch or similar would be a next-phase addition |
