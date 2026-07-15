# Online Food Delivery System — High-Level Project Planning Document

| | |
|---|---|
| **Document Type** | Software Design / Project Planning Document |
| **Project** | Online Food Delivery System (Spring Boot Microservices) |
| **Author Role** | Senior Software Architect (planning capacity) |
| **Target Timeline** | 1 week, single developer |
| **Status** | Draft v1.0 |

---

## 1. Project Overview

The Online Food Delivery System is a learning-oriented, microservices-based platform that replicates the core workflow of a real food delivery product: customers browse restaurants and menus, place orders, pay for them, and have those orders delivered by a delivery agent, with restaurant owners and platform admins managing the supply side.

The project's primary goal is educational — to give a single developer hands-on, end-to-end experience with the Spring Cloud microservices ecosystem (service discovery, centralized configuration, API gateway routing, inter-service REST communication, and independent per-service databases) without the scale or organizational overhead of a real production system.

The architecture intentionally favors **simplicity and completeness over sophistication**: every service should be runnable, testable, and demonstrably working within a one-week window, rather than gold-plated with enterprise patterns (sagas, event sourcing, CQRS, message brokers) that a solo learner doesn't yet need to internalize.

## 2. Objectives

| # | Objective |
|---|---|
| 1 | Build a working, end-to-end microservices system covering the full order lifecycle: browse → cart → order → pay → deliver → review. |
| 2 | Demonstrate correct **database-per-service** isolation with no shared schemas or cross-database foreign keys. |
| 3 | Demonstrate **service discovery** (Eureka) and **centralized configuration** (Spring Cloud Config) instead of hardcoded service URLs/settings. |
| 4 | Demonstrate a single **API Gateway** entry point that routes to all business services. |
| 5 | Demonstrate **synchronous inter-service communication** via OpenFeign, including basic resilience (timeouts, fallback handling). |
| 6 | Implement **stateless authentication** (JWT) validated at the gateway and/or each service. |
| 7 | Apply consistent cross-cutting concerns: validation, global exception handling, structured logging, and API documentation (Swagger/OpenAPI) across every service. |
| 8 | Achieve reasonable automated test coverage (JUnit 5 + Mockito) on business logic, without requiring full integration test infrastructure. |
| 9 | Keep the whole system realistically buildable, end-to-end, in **5–7 focused working days** by one developer. |

## 3. Scope

### 3.1 In Scope

- 9 business microservices + 3 infrastructure services (API Gateway, Eureka Server, Config Server), per the agreed baseline architecture.
- Customer-facing flows: registration/login, browsing restaurants & menus, cart management, placing orders, tracking order/delivery status, leaving reviews.
- Restaurant-owner-facing flows: managing restaurant profile, managing menu items, viewing/updating incoming orders.
- Delivery-agent-facing flows: accepting/updating delivery assignments.
- Admin-facing flows: approving restaurants, viewing/disabling users, basic audit trail.
- JWT-based authentication and role-based authorization (`CUSTOMER`, `RESTAURANT_OWNER`, `DELIVERY_AGENT`, `ADMIN`).
- Database-per-service PostgreSQL schemas (already designed in the companion schema documents).
- Swagger/OpenAPI docs per service.
- Unit and basic service-layer tests per service.

### 3.2 Out of Scope

- Real payment gateway integration (payment is **tracked**, not actually processed — `payment_status` is simulated/manually transitioned).
- Real-time location tracking / live map UI (delivery lat-long fields exist in the schema but no live socket-based tracking).
- Event-driven/async messaging (Kafka, RabbitMQ) — all inter-service calls are synchronous REST via OpenFeign for simplicity.
- Distributed transactions / sagas — cross-service consistency is handled with simple compensating calls or accepted eventual consistency, not a formal saga framework.
- Frontend UI (this plan covers backend microservices only; a thin Postman collection or Swagger UI serves as the interface for demonstration).
- Horizontal scaling, containerized orchestration (Kubernetes), or CI/CD pipelines — Docker Compose (optional) is the ceiling for this phase.
- Caching layers (Redis), rate limiting, and API monetization concerns.
- Multi-language/i18n support.

## 4. User Roles

| Role | Description |
|---|---|
| **Customer** | Browses restaurants/menus, manages cart, places orders, tracks delivery, leaves reviews. |
| **Restaurant Owner** | Manages their restaurant profile and menu, views and updates order status for their restaurant. |
| **Delivery Agent** | Views assigned deliveries, updates pickup/in-transit/delivered status. |
| **Admin** | Approves/rejects restaurant onboarding, manages users, views audit logs and platform-wide data. |

All four roles are represented by a single `users` table in Authentication Service (`role` column), with role-specific profile data owned by the corresponding business service (Customer, Restaurant, Delivery, Admin).

## 5. Core Features

| Feature Area | Description |
|---|---|
| Authentication & Authorization | Registration, login, JWT issuance/refresh, role-based access control |
| Restaurant Discovery | List/search restaurants by city/cuisine, view restaurant details |
| Menu Browsing | View categorized menu items per restaurant |
| Cart Management | Add/update/remove items, one active cart per customer per restaurant |
| Order Placement | Checkout a cart into an order, snapshot pricing and delivery address |
| Payment Tracking | Record a payment attempt/result against an order (simulated) |
| Order Status Tracking | Customer and restaurant owner both see live order status transitions |
| Delivery Assignment & Tracking | Assign an available agent, track pickup → in-transit → delivered |
| Notifications | Notify customer/restaurant/agent on key status changes (order placed, delivery assigned, order delivered) |
| Reviews & Ratings | Customer rates restaurant and/or delivery agent per completed order |
| Admin Oversight | Approve restaurants, manage/disable users, view audit trail |

## 6. Microservice Architecture Overview

### 6.1 Infrastructure Services

| Service | Responsibility |
|---|---|
| **Eureka Server** | Service registry — every business service and the gateway register here; enables discovery by logical name instead of hardcoded host:port. |
| **Config Server** | Centralized externalized configuration (Spring Cloud Config), backed by a local Git repo or classpath, serving each service's `application.yml` per profile. |
| **API Gateway** | Single entry point (Spring Cloud Gateway) for all client traffic; routes requests to the correct downstream service by path, and is the natural place to enforce JWT validation before requests reach business services. |

### 6.2 Business Services

| Service | Database | Core Responsibility |
|---|---|---|
| Authentication Service | `auth_db` | Credentials, roles, JWT issuance, refresh tokens |
| Customer Service | `customer_db` | Customer profiles, saved addresses |
| Restaurant Service | `restaurant_db` | Restaurant profiles, operating hours, approval status |
| Menu Service | `menu_db` | Menu categories and items per restaurant |
| Order Service | `order_db` | Cart, orders, order items, payment tracking |
| Delivery Service | `delivery_db` | Delivery agents, delivery assignment and status |
| Notification Service | `notification_db` | Notification log (email/SMS/push) |
| Review & Rating Service | `review_db` | Reviews/ratings for restaurants and delivery agents |
| Admin Service | `admin_db` | Admin profiles, audit logs |

### 6.3 High-Level Architecture Diagram (textual)

```
                         ┌───────────────────┐
                         │   API Gateway     │◄──── all client traffic
                         └─────────┬─────────┘
                                   │  (routes by path, validates JWT)
        ┌───────────────┬─────────┼─────────┬───────────────┬───────────────┐
        ▼               ▼         ▼         ▼               ▼               ▼
   Auth Service   Customer Svc  Restaurant  Menu Svc     Order Svc      Delivery Svc
                                  Svc                    (Cart+Pay)
        │               │         │         │               │               │
        ▼               ▼         ▼         ▼               ▼               ▼
    auth_db        customer_db restaurant_db menu_db     order_db       delivery_db

   Notification Svc     Review & Rating Svc      Admin Svc
        │                       │                     │
        ▼                       ▼                     ▼
  notification_db          review_db              admin_db

  All business services register with:  Eureka Server
  All business services fetch config from: Config Server
```

## 7. Service Responsibilities

| Service | Owns | Does NOT Own | Talks To (via Feign) |
|---|---|---|---|
| Authentication Service | Credentials, roles, tokens | Any profile data | — (leaf service) |
| Customer Service | Customer profile, addresses | Orders, payments | Authentication (validate user at registration, optional) |
| Restaurant Service | Restaurant profile, hours, approval status | Menu items | Authentication (owner validation, optional) |
| Menu Service | Categories, items, pricing, availability | Restaurant profile | Restaurant Service (validate restaurant exists/approved) |
| Order Service | Cart, orders, order items, payments | Menu content, delivery execution | Customer, Restaurant, Menu Service (validate/fetch snapshot data) |
| Delivery Service | Agents, delivery execution/status | Order content | Order Service (fetch order + addresses for a delivery) |
| Notification Service | Notification log | Business data itself | Consumes IDs only; no outbound Feign calls needed (receives triggers) |
| Review & Rating Service | Reviews/ratings | Restaurant/agent aggregate scores (computed on read) | Order Service (validate the order exists/is delivered) |
| Admin Service | Admin profiles, audit logs | Data being administered | Restaurant, Customer, Order Service (perform admin actions) |

## 8. Inter-Service Communication

- **Synchronous REST via OpenFeign** is the sole inter-service communication mechanism for this project — no message broker is introduced, keeping the mental model simple for a first microservices project.
- Every Feign client call goes through **Eureka-based service discovery** (logical service name, not hardcoded host/port), so services can be restarted or moved without reconfiguring callers.
- Each service exposes a small set of **internal/read-only endpoints** specifically for other services to call (e.g. Restaurant Service exposes `GET /internal/restaurants/{id}` for Menu Service to validate a restaurant exists and is approved before creating menu items).
- **Data snapshotting over live joins**: rather than calling another service on every read, services snapshot the data they need at write-time (e.g. Order Service stores `item_name`/`unit_price` at checkout instead of calling Menu Service on every order view). This deliberately limits the number of Feign calls needed and keeps each service resilient to the others being temporarily down.
- **Basic resilience**: each Feign client should define a request timeout and a simple fallback (e.g. `@FeignClient(fallback = ...)`) that returns a clear error rather than letting a slow downstream call hang the caller — full circuit-breaker libraries (Resilience4j) are optional/stretch, not required for week-one completion.

### 8.1 Key Feign Call Map

| Caller | Callee | Purpose |
|---|---|---|
| Menu Service | Restaurant Service | Validate restaurant exists & is `APPROVED` before allowing menu edits |
| Order Service | Customer Service | Fetch customer + default/selected address at checkout |
| Order Service | Restaurant Service | Validate restaurant is `APPROVED` and currently open |
| Order Service | Menu Service | Fetch current item price/availability when adding to cart |
| Delivery Service | Order Service | Fetch order details + addresses to create a delivery record |
| Review Service | Order Service | Validate the order exists and is `DELIVERED` before accepting a review |
| Admin Service | Restaurant Service | Approve/reject a restaurant |
| Admin Service | Customer / Delivery Service | Disable a user's related profile |

## 9. Database-per-Service Planning

Each business service owns exactly one PostgreSQL database, with no shared schemas and no cross-database foreign keys — cross-service references are plain value columns (e.g. `customer_id` in `order_db`), validated at the application layer via Feign calls where needed. Full table-level schema, column definitions, indexes, and SQL scripts for each service are provided in the companion **Database-per-Service Schema Documents** (already produced for all 9 services).

| Service | Database | Table Count |
|---|---|---|
| Authentication Service | `auth_db` | 2 (`users`, `refresh_tokens`) |
| Customer Service | `customer_db` | 2 (`customers`, `addresses`) |
| Restaurant Service | `restaurant_db` | 2 (`restaurants`, `restaurant_operating_hours`) |
| Menu Service | `menu_db` | 2 (`menu_categories`, `menu_items`) |
| Order Service | `order_db` | 5 (`carts`, `cart_items`, `orders`, `order_items`, `payments`) |
| Delivery Service | `delivery_db` | 2 (`delivery_agents`, `deliveries`) |
| Notification Service | `notification_db` | 1 (`notifications`) |
| Review & Rating Service | `review_db` | 1 (`reviews`) |
| Admin Service | `admin_db` | 2 (`admin_users`, `audit_logs`) |

## 10. Main Application Flows

### 10.1 Customer Registration & Login

1. Customer registers → Authentication Service creates a `users` row (`role = CUSTOMER`) and returns success.
2. Client calls Customer Service to create the matching `customers` profile row, linked by `user_id`.
3. Customer logs in → Authentication Service validates credentials, issues JWT access + refresh token.

### 10.2 Browse Restaurants & Menu

1. Client calls Restaurant Service (via Gateway) to list restaurants by city/cuisine.
2. Client calls Menu Service to fetch categories/items for a chosen restaurant.

### 10.3 Cart → Checkout → Order

1. Customer adds items to cart → Order Service creates/updates an `ACTIVE` cart, calling Menu Service to confirm current price/availability.
2. Customer checks out → Order Service converts the cart into an `orders` row (+ `order_items` snapshot), calling Customer Service for the delivery address snapshot and Restaurant Service to confirm it's open.
3. Order Service creates a `payments` row (`PENDING`), then transitions it to `SUCCESS`/`FAILED` (simulated).
4. Order Service (or an event trigger) calls Notification Service to notify the customer and restaurant owner.

### 10.4 Order Fulfillment & Delivery

1. Restaurant owner updates order status: `CONFIRMED` → `PREPARING` → `READY_FOR_PICKUP`.
2. Delivery Service creates a `deliveries` row (`UNASSIGNED`), pulling addresses from Order Service.
3. An available delivery agent is assigned (`ASSIGNED`), then updates status through `PICKED_UP` → `IN_TRANSIT` → `DELIVERED`.
4. Order Service's `order_status` is updated to `OUT_FOR_DELIVERY` then `DELIVERED` in step with the delivery record (via Feign call or status callback).
5. Notification Service fires at each major transition.

### 10.5 Review & Rating

1. Once an order is `DELIVERED`, the customer may submit a review for the restaurant and/or delivery agent.
2. Review Service validates against Order Service that the order exists and is `DELIVERED` before accepting the review.

### 10.6 Admin Oversight

1. Admin reviews pending restaurant registrations and approves/rejects them (Restaurant Service status update via Feign).
2. Admin can disable a problematic user account (Authentication Service status update).
3. Every admin action is written to `audit_logs`.

## 11. REST API Planning (Grouped by Service)

### 11.1 Authentication Service

| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| POST | `/api/auth/register` | Register a new user | No |
| POST | `/api/auth/login` | Authenticate, issue JWT | No |
| POST | `/api/auth/refresh` | Refresh access token | No (refresh token) |
| POST | `/api/auth/logout` | Revoke refresh token | Yes |
| GET | `/internal/auth/users/{id}` | Internal: fetch user by id | Service-to-service |

### 11.2 Customer Service

| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| POST | `/api/customers` | Create customer profile | Yes |
| GET | `/api/customers/me` | Get own profile | Yes |
| PUT | `/api/customers/me` | Update own profile | Yes |
| POST | `/api/customers/me/addresses` | Add address | Yes |
| GET | `/api/customers/me/addresses` | List addresses | Yes |
| PUT | `/api/customers/me/addresses/{id}` | Update address | Yes |
| DELETE | `/api/customers/me/addresses/{id}` | Delete address | Yes |
| GET | `/internal/customers/{id}` | Internal: fetch customer + default address | Service-to-service |

### 11.3 Restaurant Service

| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| POST | `/api/restaurants` | Register a restaurant | Yes (Owner) |
| GET | `/api/restaurants` | List/search restaurants | No |
| GET | `/api/restaurants/{id}` | Get restaurant details | No |
| PUT | `/api/restaurants/{id}` | Update restaurant | Yes (Owner) |
| PUT | `/api/restaurants/{id}/hours` | Set operating hours | Yes (Owner) |
| PUT | `/api/restaurants/{id}/status` | Approve/reject/suspend | Yes (Admin) |
| GET | `/internal/restaurants/{id}` | Internal: fetch restaurant status/address | Service-to-service |

### 11.4 Menu Service

| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| POST | `/api/restaurants/{restaurantId}/categories` | Add category | Yes (Owner) |
| GET | `/api/restaurants/{restaurantId}/menu` | Get full menu | No |
| POST | `/api/restaurants/{restaurantId}/items` | Add menu item | Yes (Owner) |
| PUT | `/api/items/{itemId}` | Update item (price/availability) | Yes (Owner) |
| DELETE | `/api/items/{itemId}` | Remove item | Yes (Owner) |
| GET | `/internal/items/{itemId}` | Internal: fetch item price/availability | Service-to-service |

### 11.5 Order Service (Cart + Orders + Payments)

| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| POST | `/api/cart/items` | Add item to cart | Yes (Customer) |
| GET | `/api/cart` | View active cart | Yes (Customer) |
| PUT | `/api/cart/items/{id}` | Update quantity | Yes (Customer) |
| DELETE | `/api/cart/items/{id}` | Remove item | Yes (Customer) |
| POST | `/api/cart/checkout` | Convert cart to order | Yes (Customer) |
| GET | `/api/orders/{id}` | Get order details | Yes |
| GET | `/api/orders` | List own orders (customer) or incoming (owner) | Yes |
| PUT | `/api/orders/{id}/status` | Update order status | Yes (Owner) |
| POST | `/api/orders/{id}/payment` | Record payment attempt | Yes (Customer) |
| GET | `/internal/orders/{id}` | Internal: fetch order + addresses | Service-to-service |

### 11.6 Delivery Service

| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| POST | `/api/delivery-agents` | Register as delivery agent | Yes (Agent) |
| PUT | `/api/delivery-agents/me/status` | Update availability | Yes (Agent) |
| POST | `/api/deliveries` | Create delivery record for an order | Yes (System/Owner) |
| PUT | `/api/deliveries/{id}/assign` | Assign an agent | Yes (System/Admin) |
| PUT | `/api/deliveries/{id}/status` | Update delivery status | Yes (Agent) |
| GET | `/api/deliveries/{id}` | Get delivery status | Yes |

### 11.7 Notification Service

| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| POST | `/api/notifications` | Create/send a notification (internal trigger) | Service-to-service |
| GET | `/api/notifications/me` | List own notifications | Yes |

### 11.8 Review & Rating Service

| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| POST | `/api/reviews` | Submit a review | Yes (Customer) |
| GET | `/api/reviews/restaurant/{id}` | List reviews for a restaurant | No |
| GET | `/api/reviews/agent/{id}` | List reviews for a delivery agent | No |

### 11.9 Admin Service

| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| GET | `/api/admin/restaurants/pending` | List pending restaurant approvals | Yes (Admin) |
| PUT | `/api/admin/restaurants/{id}/approve` | Approve restaurant | Yes (Admin) |
| PUT | `/api/admin/users/{id}/disable` | Disable a user | Yes (Admin) |
| GET | `/api/admin/audit-logs` | View audit trail | Yes (Admin) |

## 12. Security

- **Authentication**: JWT-based, issued by Authentication Service. Access tokens are short-lived; refresh tokens are longer-lived and stored (hashed) server-side for revocation support.
- **Authorization**: Role claim embedded in the JWT (`CUSTOMER`, `RESTAURANT_OWNER`, `DELIVERY_AGENT`, `ADMIN`); enforced with Spring Security method-level (`@PreAuthorize`) or endpoint-level rules in each business service.
- **Gateway-level enforcement**: API Gateway validates JWT signature/expiry on every request before routing, rejecting unauthenticated calls early; fine-grained role checks still happen in the target service.
- **Password storage**: BCrypt hashing only, never plaintext or reversible encryption.
- **Service-to-service calls**: internal endpoints (`/internal/**`) are not exposed through the Gateway's public routes and are restricted to intra-network calls, avoided from direct client access.
- **Input validation**: Bean Validation (`@Valid`, `@NotNull`, `@Size`, etc.) on every request DTO across all services.
- **Global exception handling**: a `@ControllerAdvice`-based handler per service standardizes error responses (consistent JSON shape, correct HTTP status codes) and avoids leaking stack traces to clients.

## 13. Testing Strategy

| Layer | Approach | Tooling |
|---|---|---|
| Unit tests | Service-layer business logic (pricing calculations, status transitions, validation rules) tested in isolation with mocked repositories/Feign clients | JUnit 5 + Mockito |
| Repository tests | Basic CRUD/query behavior against an in-memory or test PostgreSQL instance for custom queries only (not every generated method) | JUnit 5 + Spring Data JPA test slice |
| Controller tests | Request/response shape, validation error handling, security rules (authenticated/unauthenticated, wrong role) | JUnit 5 + MockMvc |
| Manual/API-level testing | End-to-end flow verification through Swagger UI or a Postman collection, run against the full stack (Gateway → Eureka → services) | Swagger UI / Postman |
| Out of scope (week one) | Full contract testing (Pact), performance/load testing, mutation testing | — |

Given the one-week timeline, testing effort should prioritize **Order Service and Authentication Service** first (highest business-logic complexity and security sensitivity), with lighter coverage on simpler CRUD-heavy services (Notification, Admin).

## 14. Suggested Project Structure

A multi-module Maven layout (or a set of independent repositories, if preferred) with a consistent internal package structure per service:

```
food-delivery-system/
├── eureka-server/
├── config-server/
│   └── config-repo/                 # externalized application.yml per service/profile
├── api-gateway/
├── auth-service/
├── customer-service/
├── restaurant-service/
├── menu-service/
├── order-service/
├── delivery-service/
├── notification-service/
├── review-rating-service/
├── admin-service/
└── pom.xml                          # parent/aggregator pom (optional)
```

Each business service follows a consistent internal structure:

```
<service-name>/
├── src/main/java/com/fooddelivery/<service>/
│   ├── <Service>Application.java
│   ├── config/                      # security config, OpenAPI config, Feign config
│   ├── controller/                  # REST controllers
│   ├── service/                     # business logic
│   ├── repository/                  # Spring Data JPA repositories
│   ├── entity/                      # JPA entities
│   ├── dto/                         # request/response DTOs
│   ├── mapper/                      # entity <-> DTO mapping
│   ├── client/                      # OpenFeign clients + fallbacks
│   ├── exception/                   # custom exceptions + @ControllerAdvice handler
│   └── security/                    # JWT filter/utils (where applicable)
├── src/main/resources/
│   ├── application.yml               # bootstrap-only; full config from Config Server
│   └── logback-spring.xml
├── src/test/java/com/fooddelivery/<service>/
│   ├── service/                      # unit tests
│   ├── controller/                   # controller tests
│   └── repository/                   # repository tests
└── pom.xml
```

---

**Document Status:** This plan, together with the companion per-service database schema documents, forms the complete design baseline before implementation begins. Any deviation from this plan during the build week (e.g. simplifying a flow further) should be a deliberate, noted trade-off rather than untracked scope creep.
