# Online Food Delivery System — Database Schema Design

**Architecture:** Database-per-Service (PostgreSQL) · **Primary Keys:** BIGSERIAL / SERIAL · **Scope:** 9 business services, one database per service

> This document uses the entities, tables, and fields from the *Online Food Delivery System — Software Design Document* as the source of truth, with one deliberate simplification: no separate `refresh_tokens` table (see Section 3, Authentication Service).

---

## 1. Design Principles

- **Database-per-Service.** Each microservice owns a private PostgreSQL database; no service accesses another service's tables directly.
- **Cross-service references are plain logical IDs, not foreign keys.** An order's `customer_id`, for example, is stored as a value only — there is no database-level FOREIGN KEY across `order_service_db` and `customer_service_db`, since PostgreSQL cannot enforce referential integrity across separate databases. These references are validated at the application layer: via a REST/OpenFeign call at write time, via a trusted JWT claim (for the authenticated user's own ID), or accepted as eventually consistent via a domain event.
- **BIGSERIAL / SERIAL primary keys**, not UUID — simpler debugging and smaller indexes at this project's scale. (This is a deliberate choice to match the source design document; UUID remains a reasonable upgrade later if IDs ever need to be globally unique before reaching a shared datastore.)
- **Status/type fields are CHECK-constrained VARCHAR, not native ENUM** — new values can be added with a constraint change instead of a type migration.
- **Snapshot over live lookup.** Where data from another service must be preserved permanently (e.g. an item's name and price at the moment it was ordered), it is copied into the owning table at write time rather than looked up live, so history stays accurate even if the source data later changes.
- **Infrastructure services** (API Gateway, Eureka Server, Config Server) own no business database and are excluded from the schemas below.

---

## 2. Service → Database Overview

| Service | Database | Tables |
|---|---|---|
| Authentication Service | `auth_service_db` | `roles`, `users` |
| Customer Service | `customer_service_db` | `customers`, `addresses` |
| Restaurant Service | `restaurant_service_db` | `restaurants` |
| Menu Service | `menu_service_db` | `categories`, `menu_items` |
| Order Service | `order_service_db` | `carts`, `cart_items`, `orders`, `order_items`, `payment_transactions` |
| Delivery Service | `delivery_service_db` | `delivery_partners`, `deliveries` |
| Notification Service | `notification_service_db` | `notifications` |
| Review & Rating Service | `review_service_db` | `reviews` |
| Admin Service | `admin_service_db` | `admin_users`, `audit_logs`, `platform_settings` |

---

## 3. Authentication Service — `auth_service_db`

**Purpose:** Owns identity and access control for every actor on the platform (customer, restaurant owner, delivery partner, admin). Issues a JWT on login; the API Gateway calls this service (or validates the token's signature directly) on incoming requests.

**Simplified JWT approach (no `refresh_tokens` table):** This project uses a single **stateless JWT access token** per login, with a moderate expiry (e.g. 24 hours). There is no server-side token store and no refresh/rotation flow — when the token expires, the user simply logs in again. This is a deliberate simplification for a one-week, beginner-friendly build: a refresh-token table only earns its complexity if the project needs server-side revocation (forced logout, "log out of all devices") or multi-device session tracking, neither of which is a stated requirement here. If that need arises later, a `refresh_tokens` table (`id`, `user_id`, `token`, `expires_at`, `revoked`, `created_at`) can be added without touching any other service.

### Tables

- **roles** — Static lookup of platform roles.
- **users** — One row per registered account, regardless of role.

### Table: `roles`

| Column Name | Data Type | Nullable | Default | Description |
|---|---|---|---|---|
| id | SERIAL | No | auto | Primary key |
| name | VARCHAR(30) | No | — | Role code: `CUSTOMER`, `RESTAURANT_OWNER`, `DELIVERY_PARTNER`, `ADMIN` |

### Table: `users`

| Column Name | Data Type | Nullable | Default | Description |
|---|---|---|---|---|
| id | BIGSERIAL | No | auto | Primary key; the platform-wide user identifier referenced logically by every other service |
| email | VARCHAR(150) | No | — | Login email, unique |
| phone | VARCHAR(15) | No | — | Login/contact phone, unique |
| password_hash | VARCHAR(255) | No | — | BCrypt hash, never plaintext |
| role_id | INT | No | — | FK to `roles.id` |
| is_active | BOOLEAN | No | `TRUE` | Soft-disable flag for account suspension |
| is_verified | BOOLEAN | No | `FALSE` | Email/phone verification flag |
| created_at | TIMESTAMP | No | `now()` | Row creation time |
| updated_at | TIMESTAMP | No | `now()` | Auto-updated via trigger |

### Relationships

- `roles` (1) → `users` (M): every user has exactly one role.
- Logical, cross-service (no DB-level FK): `users.id` is referenced by Customer Service (`customers.user_id`), Restaurant Service (`restaurants.owner_user_id`), Delivery Service (`delivery_partners.user_id`), Admin Service (`admin_users.user_id`), and Notification Service (`notifications.user_id`).

### Indexes & Constraints

- PK: `roles.id`, `users.id`
- UNIQUE: `roles.name`, `users.email`, `users.phone`
- FK: `users.role_id` → `roles.id`
- Index: `idx_users_email`, `idx_users_phone`, `idx_users_role_id`

### Design Decisions

- Roles kept as a normalized lookup table (not an ENUM) so new roles can be added without a schema migration.
- `password_hash` stores only a BCrypt digest; the service never persists plaintext passwords.
- No `refresh_tokens` table — see the simplified JWT approach above.
- This is the only service permitted to own authentication credentials — every other service treats `user_id` as an opaque foreign identifier and never stores passwords.

---

## 4. Customer Service — `customer_service_db`

**Purpose:** Owns customer profile data and saved delivery addresses. Does not store credentials — authentication lives in the Authentication Service.

### Tables

- **customers** — Extended profile for a user with the `CUSTOMER` role.
- **addresses** — Saved delivery addresses for a customer.

### Table: `customers`

| Column Name | Data Type | Nullable | Default | Description |
|---|---|---|---|---|
| id | BIGSERIAL | No | auto | Primary key; the customer identifier referenced logically by Order, Review, and Delivery services |
| user_id | BIGINT | No | — | Logical reference to Authentication Service `users.id`, unique (1:1) |
| full_name | VARCHAR(100) | No | — | Display name |
| email | VARCHAR(150) | No | — | Denormalized copy of login email for read convenience |
| phone | VARCHAR(15) | No | — | Contact phone |
| date_of_birth | DATE | Yes | NULL | Optional, for promotions/age checks |
| created_at | TIMESTAMP | No | `now()` | Row creation time |
| updated_at | TIMESTAMP | No | `now()` | Auto-updated via trigger |

### Table: `addresses`

| Column Name | Data Type | Nullable | Default | Description |
|---|---|---|---|---|
| id | BIGSERIAL | No | auto | Primary key; referenced logically by Order Service `orders.delivery_address_id` |
| customer_id | BIGINT | No | — | FK to `customers.id` |
| label | VARCHAR(30) | No | `'HOME'` | User-facing label, e.g. `HOME`, `WORK` |
| address_line1 | VARCHAR(200) | No | — | Street address |
| address_line2 | VARCHAR(200) | Yes | NULL | Apartment/suite/landmark |
| city | VARCHAR(50) | No | — | City |
| state | VARCHAR(50) | No | — | State/province |
| pincode | VARCHAR(10) | No | — | Postal code |
| latitude | NUMERIC(9,6) | Yes | NULL | Geo-coordinate for delivery-radius checks |
| longitude | NUMERIC(9,6) | Yes | NULL | Geo-coordinate for delivery-radius checks |
| is_default | BOOLEAN | No | `FALSE` | Marks the customer's default address |
| created_at | TIMESTAMP | No | `now()` | Row creation time |

### Relationships

- `customers` (1) → `addresses` (M): a customer can save multiple addresses.
- Logical: `customers.user_id` → Authentication Service `users.id`.
- Logical, inbound: Order Service (`orders.customer_id`), Review Service (`reviews.customer_id`) reference `customers.id` / `addresses.id` by value only.

### Indexes & Constraints

- PK: `customers.id`, `addresses.id`
- UNIQUE: `customers.user_id`
- FK: `addresses.customer_id` → `customers.id` ON DELETE CASCADE
- Partial UNIQUE index ensures only one default address per customer
- Index: `idx_customers_user_id`, `idx_addresses_customer_id`

### Design Decisions

- `email`/`phone` are denormalized copies from Authentication Service, kept in sync via an event (`UserRegistered`/`UserUpdated`) so Customer Service can serve profile reads without a synchronous call.
- A partial unique index (`WHERE is_default = TRUE`) enforces "at most one default address" at the database level instead of application logic alone.
- No FK to the `users` table across databases — cross-service references are logical IDs only, validated at write time via the Authentication Service API or trusted JWT claims.

---

## 5. Restaurant Service — `restaurant_service_db`

**Purpose:** Owns restaurant profiles, operating hours, availability, and the admin approval workflow for onboarding new restaurants.

### Tables

- **restaurants** — One row per restaurant listed on the platform.

### Table: `restaurants`

| Column Name | Data Type | Nullable | Default | Description |
|---|---|---|---|---|
| id | BIGSERIAL | No | auto | Primary key; referenced logically by Menu, Order, and Review services |
| owner_user_id | BIGINT | No | — | Logical reference to Authentication Service `users.id` |
| name | VARCHAR(150) | No | — | Restaurant display name |
| description | TEXT | Yes | NULL | Free-text description |
| email | VARCHAR(150) | Yes | NULL | Business contact email |
| phone | VARCHAR(15) | No | — | Business contact phone |
| address_line | VARCHAR(200) | No | — | Street address |
| city | VARCHAR(50) | No | — | City, used for restaurant discovery search |
| state | VARCHAR(50) | No | — | State/province |
| pincode | VARCHAR(10) | No | — | Postal code |
| latitude | NUMERIC(9,6) | Yes | NULL | Geo-coordinate |
| longitude | NUMERIC(9,6) | Yes | NULL | Geo-coordinate |
| approval_status | VARCHAR(20) | No | `'PENDING'` | `PENDING`, `APPROVED`, or `REJECTED` |
| is_open | BOOLEAN | No | `FALSE` | Live open/closed toggle set by the owner |
| opening_time | TIME | Yes | NULL | Daily opening time |
| closing_time | TIME | Yes | NULL | Daily closing time |
| avg_rating | NUMERIC(2,1) | No | `0.0` | Denormalized rolling average, updated asynchronously from Review & Rating events |
| created_at | TIMESTAMP | No | `now()` | Row creation time |
| updated_at | TIMESTAMP | No | `now()` | Auto-updated via trigger |

### Relationships

- Single-table service; no internal FKs.
- Logical: `restaurants.owner_user_id` → Authentication Service `users.id`.
- Logical, inbound: Menu Service (`menu_items.restaurant_id`), Order Service (`orders.restaurant_id`), Review Service (`reviews.restaurant_id`) all reference `restaurants.id` by value.

### Indexes & Constraints

- PK: `restaurants.id`
- CHECK: `approval_status IN ('PENDING','APPROVED','REJECTED')`
- CHECK: `avg_rating BETWEEN 0 AND 5`
- Index: `idx_restaurants_owner_user_id`, `idx_restaurants_city`, `idx_restaurants_approval_status`

### Design Decisions

- `approval_status` modeled as a checked VARCHAR rather than a Postgres ENUM, so new statuses can be added with a simple constraint change.
- `avg_rating` is intentionally denormalized here (Review & Rating Service remains the source of truth) to avoid a synchronous cross-service call every time a customer browses restaurants; it is updated by consuming a `RatingSubmitted` event.
- `city` is indexed directly to support restaurant-discovery search without introducing a dedicated search engine for this scope.

---

## 6. Menu Service — `menu_service_db`

**Purpose:** Owns food categories and menu items for every restaurant, including pricing and availability.

### Tables

- **categories** — Menu category/section within a restaurant (e.g. Starters, Main Course).
- **menu_items** — Individual food items available for order.

### Table: `categories`

| Column Name | Data Type | Nullable | Default | Description |
|---|---|---|---|---|
| id | BIGSERIAL | No | auto | Primary key |
| restaurant_id | BIGINT | No | — | Logical reference to Restaurant Service `restaurants.id` |
| name | VARCHAR(100) | No | — | Category name, unique per restaurant |
| display_order | INT | No | `0` | Sort order in the app UI |
| created_at | TIMESTAMP | No | `now()` | Row creation time |

### Table: `menu_items`

| Column Name | Data Type | Nullable | Default | Description |
|---|---|---|---|---|
| id | BIGSERIAL | No | auto | Primary key; referenced logically by Order Service `cart_items`/`order_items` |
| restaurant_id | BIGINT | No | — | Logical reference to Restaurant Service `restaurants.id` (denormalized for fast filtering) |
| category_id | BIGINT | No | — | FK to `categories.id` |
| name | VARCHAR(150) | No | — | Item name |
| description | TEXT | Yes | NULL | Item description |
| price | NUMERIC(10,2) | No | — | Current price |
| is_veg | BOOLEAN | No | `TRUE` | Vegetarian flag |
| is_available | BOOLEAN | No | `TRUE` | Toggled off when out of stock |
| image_url | VARCHAR(500) | Yes | NULL | CDN URL for item photo |
| created_at | TIMESTAMP | No | `now()` | Row creation time |
| updated_at | TIMESTAMP | No | `now()` | Auto-updated via trigger |

### Relationships

- `categories` (1) → `menu_items` (M): a category groups many items.
- Logical: `categories.restaurant_id` / `menu_items.restaurant_id` → Restaurant Service `restaurants.id`.
- Logical, inbound: Order Service references `menu_items.id` as a snapshot source when items are added to a cart/order.

### Indexes & Constraints

- PK: `categories.id`, `menu_items.id`
- UNIQUE: (`restaurant_id`, `name`) on `categories` — no duplicate category names within a restaurant
- FK: `menu_items.category_id` → `categories.id` ON DELETE CASCADE
- CHECK: `menu_items.price >= 0`
- Index: `idx_categories_restaurant_id`, `idx_menu_items_restaurant_id`, `idx_menu_items_category_id`, `idx_menu_items_is_available`

### Design Decisions

- `restaurant_id` is stored on both `categories` and `menu_items` (denormalized) rather than requiring a join through categories, since "list all items for restaurant X" is the dominant query path.
- Menu Service is kept separate from Restaurant Service because menu data changes far more frequently and independently — a restaurant owner editing prices shouldn't contend with restaurant-profile writes.
- Order Service snapshots item name/price into its own `order_items` table at order time, so later menu price changes never retroactively alter historical orders.

---

## 7. Order Service — `order_service_db`

**Purpose:** Merged service owning the shopping cart, order lifecycle, and payment tracking (Cart Service and Payment Service were folded into this service per the agreed architecture).

### Tables

- **carts** — One active cart per customer.
- **cart_items** — Line items currently in a customer's cart.
- **orders** — A placed order, from checkout through delivery.
- **order_items** — Immutable snapshot of items within a placed order.
- **payment_transactions** — Payment attempts/records against an order.

### Table: `carts`

| Column Name | Data Type | Nullable | Default | Description |
|---|---|---|---|---|
| id | BIGSERIAL | No | auto | Primary key |
| customer_id | BIGINT | No | — | Logical reference to Customer Service `customers.id`, unique (one cart per customer) |
| restaurant_id | BIGINT | Yes | NULL | Logical reference to Restaurant Service `restaurants.id`; cart is single-restaurant at a time |
| created_at | TIMESTAMP | No | `now()` | Row creation time |
| updated_at | TIMESTAMP | No | `now()` | Auto-updated via trigger |

### Table: `cart_items`

| Column Name | Data Type | Nullable | Default | Description |
|---|---|---|---|---|
| id | BIGSERIAL | No | auto | Primary key |
| cart_id | BIGINT | No | — | FK to `carts.id` |
| menu_item_id | BIGINT | No | — | Logical reference to Menu Service `menu_items.id` |
| item_name | VARCHAR(150) | No | — | Snapshot of item name at add-time |
| unit_price | NUMERIC(10,2) | No | — | Snapshot of price at add-time |
| quantity | INT | No | `1` | Number of units |
| created_at | TIMESTAMP | No | `now()` | Row creation time |

### Table: `orders`

| Column Name | Data Type | Nullable | Default | Description |
|---|---|---|---|---|
| id | BIGSERIAL | No | auto | Primary key; referenced logically by Delivery and Review services |
| customer_id | BIGINT | No | — | Logical reference to Customer Service `customers.id` |
| restaurant_id | BIGINT | No | — | Logical reference to Restaurant Service `restaurants.id` |
| delivery_address_id | BIGINT | No | — | Logical reference to Customer Service `addresses.id` |
| order_status | VARCHAR(30) | No | `'PLACED'` | `PLACED`, `ACCEPTED`, `PREPARING`, `READY_FOR_PICKUP`, `PICKED_UP`, `DELIVERED`, `CANCELLED` |
| subtotal | NUMERIC(10,2) | No | — | Sum of `order_items` subtotal |
| delivery_fee | NUMERIC(10,2) | No | `0` | Delivery charge |
| total_amount | NUMERIC(10,2) | No | — | subtotal + delivery_fee |
| payment_method | VARCHAR(20) | No | — | `ONLINE` or `COD` |
| payment_status | VARCHAR(20) | No | `'PENDING'` | `PENDING`, `PAID`, `FAILED`, `REFUNDED` |
| placed_at | TIMESTAMP | No | `now()` | Checkout time |
| updated_at | TIMESTAMP | No | `now()` | Auto-updated via trigger |

### Table: `order_items`

| Column Name | Data Type | Nullable | Default | Description |
|---|---|---|---|---|
| id | BIGSERIAL | No | auto | Primary key |
| order_id | BIGINT | No | — | FK to `orders.id` |
| menu_item_id | BIGINT | No | — | Logical reference to Menu Service `menu_items.id` |
| item_name | VARCHAR(150) | No | — | Snapshot of item name at order time |
| unit_price | NUMERIC(10,2) | No | — | Snapshot of price at order time |
| quantity | INT | No | — | Number of units ordered |
| subtotal | NUMERIC(10,2) | No | — | unit_price × quantity |

### Table: `payment_transactions`

| Column Name | Data Type | Nullable | Default | Description |
|---|---|---|---|---|
| id | BIGSERIAL | No | auto | Primary key |
| order_id | BIGINT | No | — | FK to `orders.id` |
| transaction_ref | VARCHAR(100) | Yes | NULL | Payment gateway reference, unique; NULL for COD |
| amount | NUMERIC(10,2) | No | — | Amount charged/attempted |
| status | VARCHAR(20) | No | `'PENDING'` | `PENDING`, `PAID`, `FAILED`, `REFUNDED` |
| gateway_response | TEXT | Yes | NULL | Raw response payload for auditing/debugging |
| created_at | TIMESTAMP | No | `now()` | Row creation time |

### Relationships

- `carts` (1) → `cart_items` (M); `orders` (1) → `order_items` (M); `orders` (1) → `payment_transactions` (M).
- Logical: `customer_id`/`restaurant_id`/`delivery_address_id`/`menu_item_id` reference Customer, Restaurant, and Menu services respectively.
- Logical, inbound: Delivery Service (`deliveries.order_id`) and Review Service (`reviews.order_id`) reference `orders.id`.

### Indexes & Constraints

- PK: all tables have a surrogate BIGSERIAL id
- UNIQUE: `carts.customer_id` (one cart per customer); (`cart_id`, `menu_item_id`) on `cart_items`; `payment_transactions.transaction_ref`
- FK: `cart_items.cart_id` → `carts.id` ON DELETE CASCADE; `order_items.order_id` → `orders.id` ON DELETE CASCADE; `payment_transactions.order_id` → `orders.id` ON DELETE CASCADE
- CHECK: `order_status`, `payment_method`, `payment_status`, `payment_transactions.status` restricted to their defined value sets; quantities `> 0`; `total_amount >= 0`
- Index: `idx_cart_items_cart_id`, `idx_orders_customer_id`, `idx_orders_restaurant_id`, `idx_orders_status`, `idx_order_items_order_id`, `idx_payment_transactions_order_id`

### Design Decisions

- Cart and Payment were merged into Order Service per the agreed architecture: cart is conceptually a draft order, and payment status is a tightly-coupled attribute of the order lifecycle — merging avoids a synchronous cross-service call on every checkout step.
- `order_items`/`cart_items` snapshot `item_name`/`unit_price` rather than joining live to Menu Service, so historical orders remain accurate even if menu prices change later.
- `payment_transactions` is kept as its own table (not columns on `orders`) to support multiple payment attempts per order (e.g. a failed online payment retried, or a partial refund), while `orders.payment_status` reflects the current summarized state.
- Trade-off accepted: folding payment into Order Service means PCI-adjacent payment concerns are no longer isolated — acceptable for this project's scope, but a production system integrating a real gateway would likely re-extract payment into its own service and database.

---

## 8. Delivery Service — `delivery_service_db`

**Purpose:** Owns delivery partner profiles, their approval workflow, live availability, and the assignment/tracking of deliveries against orders.

### Tables

- **delivery_partners** — One row per registered delivery partner.
- **deliveries** — One row per order assigned for delivery.

### Table: `delivery_partners`

| Column Name | Data Type | Nullable | Default | Description |
|---|---|---|---|---|
| id | BIGSERIAL | No | auto | Primary key |
| user_id | BIGINT | No | — | Logical reference to Authentication Service `users.id`, unique (1:1) |
| full_name | VARCHAR(100) | No | — | Display name |
| phone | VARCHAR(15) | No | — | Contact phone |
| vehicle_type | VARCHAR(30) | Yes | NULL | e.g. `BIKE`, `SCOOTER`, `BICYCLE` |
| vehicle_number | VARCHAR(20) | Yes | NULL | Registration number |
| approval_status | VARCHAR(20) | No | `'PENDING'` | `PENDING`, `APPROVED`, `REJECTED` |
| availability_status | VARCHAR(20) | No | `'OFFLINE'` | `AVAILABLE`, `BUSY`, `OFFLINE` |
| current_latitude | NUMERIC(9,6) | Yes | NULL | Last known location |
| current_longitude | NUMERIC(9,6) | Yes | NULL | Last known location |
| created_at | TIMESTAMP | No | `now()` | Row creation time |
| updated_at | TIMESTAMP | No | `now()` | Auto-updated via trigger |

### Table: `deliveries`

| Column Name | Data Type | Nullable | Default | Description |
|---|---|---|---|---|
| id | BIGSERIAL | No | auto | Primary key |
| order_id | BIGINT | No | — | Logical reference to Order Service `orders.id`, unique (one delivery per order) |
| delivery_partner_id | BIGINT | No | — | FK to `delivery_partners.id` |
| delivery_status | VARCHAR(20) | No | `'ASSIGNED'` | `ASSIGNED`, `PICKED_UP`, `DELIVERED`, `CANCELLED` |
| assigned_at | TIMESTAMP | No | `now()` | Assignment time |
| picked_up_at | TIMESTAMP | Yes | NULL | Set when partner collects the order |
| delivered_at | TIMESTAMP | Yes | NULL | Set when order marked delivered |

### Relationships

- `delivery_partners` (1) → `deliveries` (M): a partner completes many deliveries over time (only one active at a time by convention, enforced at the application level via `availability_status`).
- Logical: `delivery_partners.user_id` → Authentication Service `users.id`; `deliveries.order_id` → Order Service `orders.id`.

### Indexes & Constraints

- PK: `delivery_partners.id`, `deliveries.id`
- UNIQUE: `delivery_partners.user_id`, `deliveries.order_id`
- FK: `deliveries.delivery_partner_id` → `delivery_partners.id`
- CHECK: `approval_status IN ('PENDING','APPROVED','REJECTED')`; `availability_status IN ('AVAILABLE','BUSY','OFFLINE')`; `delivery_status IN ('ASSIGNED','PICKED_UP','DELIVERED','CANCELLED')`
- Index: `idx_delivery_partners_availability`, `idx_deliveries_partner_id`, `idx_deliveries_order_id`

### Design Decisions

- `availability_status` drives the partner-matching query ("find one `AVAILABLE`, `APPROVED` partner") — indexed for fast lookup during the assignment workflow.
- `current_latitude`/`current_longitude` are simple columns rather than a PostGIS geography type, keeping the schema dependency-light for this scope.
- `deliveries.order_id` is UNIQUE because the business rule is one active delivery per order; reassignment on cancellation is modeled by updating `delivery_status` rather than inserting a second row.

---

## 9. Notification Service — `notification_service_db`

**Purpose:** Owns delivery of order, payment, delivery-status, and approval notifications to users across email, SMS, and push channels. Consumes events from other services rather than being called synchronously in the critical path.

### Tables

- **notifications** — One row per notification sent (or attempted) to a user.

### Table: `notifications`

| Column Name | Data Type | Nullable | Default | Description |
|---|---|---|---|---|
| id | BIGSERIAL | No | auto | Primary key |
| user_id | BIGINT | No | — | Logical reference to Authentication Service `users.id` |
| notification_type | VARCHAR(40) | No | — | `ORDER_CONFIRMATION`, `PAYMENT_STATUS`, `DELIVERY_STATUS`, `RESTAURANT_APPROVAL`, `DELIVERY_PARTNER_APPROVAL` |
| channel | VARCHAR(20) | No | `'PUSH'` | `EMAIL`, `SMS`, or `PUSH` |
| title | VARCHAR(150) | No | — | Notification title |
| message | TEXT | No | — | Notification body |
| status | VARCHAR(20) | No | `'PENDING'` | `PENDING`, `SENT`, `FAILED` |
| sent_at | TIMESTAMP | Yes | NULL | Set when delivery to the channel provider succeeds |
| created_at | TIMESTAMP | No | `now()` | Row creation time |

### Relationships

- Single-table service; no internal FKs.
- Logical: `notifications.user_id` → Authentication Service `users.id`. The originating entity (order, restaurant, delivery partner) is referenced implicitly through the event payload, not stored relationally here, since this table's purpose is delivery tracking, not business-entity linkage.

### Indexes & Constraints

- PK: `notifications.id`
- CHECK: `notification_type`, `channel`, and `status` each restricted to their defined value sets
- Index: `idx_notifications_user_id`, `idx_notifications_status`

### Design Decisions

- No `updated_at`/trigger — notifications are append-only/immutable once created; only `status` and `sent_at` are set once on dispatch.
- No FK to a "source entity" table (order, restaurant, etc.) by design — Notification Service is intentionally decoupled from the schemas of the services that trigger it.
- This service is the clearest candidate for an event-driven (pub/sub) pattern rather than synchronous REST calls, since notification sends are fire-and-forget from the caller's perspective.

---

## 10. Review & Rating Service — `review_service_db`

**Purpose:** Owns customer-submitted ratings and reviews for restaurants, tied to a completed order.

### Tables

- **reviews** — One review per completed order.

### Table: `reviews`

| Column Name | Data Type | Nullable | Default | Description |
|---|---|---|---|---|
| id | BIGSERIAL | No | auto | Primary key |
| customer_id | BIGINT | No | — | Logical reference to Customer Service `customers.id` |
| restaurant_id | BIGINT | No | — | Logical reference to Restaurant Service `restaurants.id` |
| order_id | BIGINT | No | — | Logical reference to Order Service `orders.id`, unique (one review per order) |
| rating | SMALLINT | No | — | 1–5 star rating |
| comment | TEXT | Yes | NULL | Optional written review |
| created_at | TIMESTAMP | No | `now()` | Row creation time |
| updated_at | TIMESTAMP | No | `now()` | Auto-updated via trigger |

### Relationships

- Single-table service; no internal FKs.
- Logical: `customer_id` → Customer Service; `restaurant_id` → Restaurant Service; `order_id` → Order Service. A `RatingSubmitted` event is published so Restaurant Service can update its denormalized `avg_rating`.

### Indexes & Constraints

- PK: `reviews.id`
- UNIQUE: `order_id` — enforces exactly one review per order
- CHECK: `rating BETWEEN 1 AND 5`
- Index: `idx_reviews_restaurant_id`, `idx_reviews_customer_id`

### Design Decisions

- `order_id` carries the UNIQUE constraint rather than (`customer_id`, `restaurant_id`), since a customer legitimately orders from the same restaurant many times, and each order deserves its own review opportunity.
- Kept as its own service rather than folded into Restaurant Service because review volume, moderation needs, and read/write patterns differ from restaurant profile management.

---

## 11. Admin Service — `admin_service_db`

**Purpose:** Owns administrative identity, an audit trail of privileged actions (restaurant/partner approvals, user management), and platform-wide configuration settings.

### Tables

- **admin_users** — Extended profile for a user with the `ADMIN` role.
- **audit_logs** — Immutable record of privileged admin actions.
- **platform_settings** — Key/value platform configuration editable by admins.

### Table: `admin_users`

| Column Name | Data Type | Nullable | Default | Description |
|---|---|---|---|---|
| id | BIGSERIAL | No | auto | Primary key |
| user_id | BIGINT | No | — | Logical reference to Authentication Service `users.id`, unique (1:1) |
| full_name | VARCHAR(100) | No | — | Display name |
| email | VARCHAR(150) | No | — | Denormalized contact email |
| created_at | TIMESTAMP | No | `now()` | Row creation time |

### Table: `audit_logs`

| Column Name | Data Type | Nullable | Default | Description |
|---|---|---|---|---|
| id | BIGSERIAL | No | auto | Primary key |
| admin_id | BIGINT | No | — | FK to `admin_users.id` |
| action | VARCHAR(100) | No | — | e.g. `APPROVE_RESTAURANT`, `REJECT_DELIVERY_PARTNER`, `SUSPEND_USER` |
| entity_type | VARCHAR(50) | No | — | e.g. `RESTAURANT`, `DELIVERY_PARTNER`, `USER` |
| entity_id | BIGINT | No | — | Logical reference to the affected entity's id in its owning service |
| description | TEXT | Yes | NULL | Free-text detail |
| created_at | TIMESTAMP | No | `now()` | Row creation time |

### Table: `platform_settings`

| Column Name | Data Type | Nullable | Default | Description |
|---|---|---|---|---|
| id | SERIAL | No | auto | Primary key |
| setting_key | VARCHAR(100) | No | — | Unique setting name, e.g. `DEFAULT_DELIVERY_FEE` |
| setting_value | VARCHAR(500) | No | — | Setting value, stored as text and parsed by the consuming service |
| updated_at | TIMESTAMP | No | `now()` | Last modified time |

### Relationships

- `admin_users` (1) → `audit_logs` (M): every audit entry is attributed to one admin.
- Logical: `admin_users.user_id` → Authentication Service `users.id`; `audit_logs.entity_id` → the id of a row in Restaurant, Delivery, or Authentication service, disambiguated by `entity_type`.

### Indexes & Constraints

- PK: `admin_users.id`, `audit_logs.id`, `platform_settings.id`
- UNIQUE: `admin_users.user_id`, `platform_settings.setting_key`
- FK: `audit_logs.admin_id` → `admin_users.id`
- Index: `idx_audit_logs_admin_id`, `idx_audit_logs_entity` (composite on `entity_type`, `entity_id`)

### Design Decisions

- `audit_logs.entity_id` is a plain BIGINT with no FK — by nature it points across multiple other services' tables depending on `entity_type`, so referential integrity for it is enforced at the application layer.
- Approval actions themselves (e.g. setting `restaurants.approval_status = 'APPROVED'`) are still executed by the owning service via an authenticated admin-scoped API call; Admin Service records the audit trail rather than owning the approval state itself.
- `platform_settings` uses a generic key/value shape rather than dedicated columns so new settings can be added without a schema migration.

---

## 12. Summary of Deviations from the Source Design Document

| Item | Source Document | This Document | Reason |
|---|---|---|---|
| Auth session handling | `refresh_tokens` table + rotation | Single stateless JWT access token, no DB-backed refresh flow | Explicit simplification requested — revocation/multi-device session tracking isn't a project requirement, and removing it drops a table plus its rotation/cleanup logic for a one-week beginner build |

No other entities, tables, or fields were added beyond what the source design document specifies.
