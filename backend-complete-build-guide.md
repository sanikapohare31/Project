Infrastructure + Authentication Service

One file, full depth — build top to bottom, in this order, since later services Feign-call the ones built before them.


Companion doc: Database-Schema-Design.md — every entity prompt below references it directly. Keep it open alongside this file.



Table of Contents


Infrastructure (Config Server, Eureka, API Gateway) + Authentication Service

Customer Service → Restaurant Service → Menu Service

Order Service (Cart + Orders + Payment tracking)

Notification Service → Delivery Service

Review & Rating Service → Admin Service




Note on OpenFeign: No service built on Day 1 makes an outbound call to another service — Auth Service is a leaf service (only ever called, never calls out), and the Gateway/Eureka/Config Server are infrastructure, not Feign consumers. The OpenFeign conventions (package layout, fallback pattern, timeout config) belong in the Day 2 guide, introduced alongside the first real Feign client (CustomerServiceClient calling Auth Service's /internal/auth/users/{id}). Day 1's Auth Service still exposes that internal endpoint (Prompt 5.5) so it's ready for Day 2 to consume.



1. Project Skeleton

Prompt 1.1 — Parent project

Create a multi-module Maven project named "food-delivery-system" for a Spring Boot
microservices application using Java 21 and Spring Boot 3.2.x.

Generate a parent pom.xml (packaging: pom) that:
- Declares Spring Boot 3.2.x and Spring Cloud 2023.0.x BOMs via dependencyManagement
- Sets java.version to 21
- Declares these modules (empty folders for now): eureka-server, config-server,
  api-gateway, auth-service, customer-service, restaurant-service, menu-service,
  order-service, delivery-service, notification-service, review-rating-service,
  admin-service
- Includes common dependency versions for: lombok, spring-boot-starter-test,
  springdoc-openapi-starter-webmvc-ui

Do not generate application code yet — just the parent pom and empty module folders
with placeholder pom.xml files inheriting from the parent.


2. Config Server

Prompt 2.1 — Config Server module

Inside the config-server module of a Spring Cloud multi-module Maven project
(Java 21, Spring Boot 3.2.x, Spring Cloud 2023.0.x), generate:

1. pom.xml with spring-cloud-config-server dependency
2. ConfigServerApplication.java with @EnableConfigServer
3. application.yml configured to:
   - run on port 8888
   - serve configuration from a native/classpath location at
     classpath:/config-repo (not a Git repo, to keep local setup simple)
   - register itself with Eureka at http://localhost:8761/eureka

Also generate a config-repo folder under src/main/resources with placeholder
application.yml files for each of these services: auth-service, customer-service,
restaurant-service, menu-service, order-service, delivery-service,
notification-service, review-rating-service, admin-service.
Each placeholder file should just contain that service's spring.datasource.url,
username, and password for a local PostgreSQL database named
<service>_service_db (e.g. auth_service_db), plus server.port matching this table:

auth-service: 8081, customer-service: 8082, restaurant-service: 8083,
menu-service: 8084, order-service: 8085, delivery-service: 8086,
notification-service: 8087, review-rating-service: 8088, admin-service: 8089


3. Eureka Server

Prompt 3.1 — Eureka Server module

Inside the eureka-server module of the same Spring Cloud multi-module project,
generate:

1. pom.xml with spring-cloud-starter-netflix-eureka-server dependency
2. EurekaServerApplication.java with @EnableEurekaServer
3. application.yml configured to:
   - run on port 8761
   - disable self-registration and self-fetching of the registry
     (register-with-eureka: false, fetch-registry: false), since this is the
     registry itself
   - set a simple application name "eureka-server"

Keep this minimal — no custom dashboard changes, use Eureka's default UI.


4. API Gateway

Prompt 4.1 — Gateway module + routing

Inside the api-gateway module of the same project, generate:

1. pom.xml with spring-cloud-starter-gateway, spring-cloud-starter-netflix-eureka-client,
   and spring-boot-starter-security dependencies
2. ApiGatewayApplication.java
3. application.yml configured to:
   - run on port 8080
   - register with Eureka at http://localhost:8761/eureka
   - pull shared config from Config Server at http://localhost:8888
   - enable Spring Cloud Gateway's service-discovery-based routing
     (spring.cloud.gateway.discovery.locator.enabled: true)
   - define explicit routes (do NOT rely only on auto-discovery) for these
     public path prefixes, each routing by lb://<eureka-service-name>:
     /api/auth/**        -> lb://auth-service
     /api/customers/**   -> lb://customer-service
     /api/restaurants/** -> lb://restaurant-service
     /api/items/**       -> lb://menu-service
     /api/cart/**        -> lb://order-service
     /api/orders/**      -> lb://order-service
     /api/deliveries/**  -> lb://delivery-service
     /api/delivery-agents/** -> lb://delivery-service
     /api/notifications/** -> lb://notification-service
     /api/reviews/**     -> lb://review-rating-service
     /api/admin/**       -> lb://admin-service

   Do NOT add a route for any /internal/** path — those must never be reachable
   through the gateway, only via direct service-to-service Feign calls.

Do not implement JWT validation yet — that comes in the next prompt.

Prompt 4.2 — JWT validation filter at the gateway

In the api-gateway module, add a global Spring Cloud Gateway filter
(GlobalFilter + Ordered) named JwtAuthenticationFilter that:

- Skips validation entirely for requests to /api/auth/register and
  /api/auth/login (these must remain public)
- For every other request, reads the "Authorization: Bearer <token>" header
- Validates the JWT signature and expiry using a shared secret (read from
  application.yml as jwt.secret) — use the jjwt library (io.jsonwebtoken)
- If the token is missing or invalid, short-circuits the request and returns
  HTTP 401 with a JSON body {"error": "Unauthorized"}
- If valid, forwards the request unchanged (role-based authorization happens
  in each downstream service, not here)

Add io.jsonwebtoken:jjwt-api, jjwt-impl, jjwt-jackson as dependencies
(version 0.12.x) in the api-gateway pom.xml.
Add a placeholder jwt.secret value in application.yml with a comment that
it must match the same secret configured in auth-service.


5. Authentication Service

Reference the exact schema from Database-Schema-Design.md Section 3 (roles, users tables) in every prompt below — do not let Copilot invent extra columns.


Prompt 5.1 — Entities

In the auth-service module (Java 21, Spring Boot 3.2.x, Spring Data JPA, Lombok),
generate JPA entities for exactly these two tables. Do not add any fields beyond
what's listed.

Table: roles
- id: SERIAL PK
- name: VARCHAR(30) NOT NULL UNIQUE  (values: CUSTOMER, RESTAURANT_OWNER,
  DELIVERY_PARTNER, ADMIN)

Table: users
- id: BIGSERIAL PK
- email: VARCHAR(150) NOT NULL UNIQUE
- phone: VARCHAR(15) NOT NULL UNIQUE
- password_hash: VARCHAR(255) NOT NULL
- role_id: INT NOT NULL, FK to roles.id
- is_active: BOOLEAN NOT NULL DEFAULT TRUE
- is_verified: BOOLEAN NOT NULL DEFAULT FALSE
- created_at: TIMESTAMP NOT NULL, auto-set on insert
- updated_at: TIMESTAMP NOT NULL, auto-updated on every update

Use package com.fooddelivery.authservice.entity.
Use @ManyToOne for the users -> roles relationship.
Use Hibernate's @CreationTimestamp / @UpdateTimestamp for created_at/updated_at
instead of a manual trigger, since this is a JPA-managed entity.

Prompt 5.2 — Repositories, DTOs, mapper

In auth-service, generate:
1. RoleRepository and UserRepository (Spring Data JPA), with a
   UserRepository.findByEmail(String email) and findByPhone(String phone)
   method in addition to the standard CRUD methods.
2. DTOs in com.fooddelivery.authservice.dto:
   - RegisterRequest (email, phone, password, role — validate email format,
     phone pattern, password min length 8, role must be one of the 4 valid
     role names, using Bean Validation annotations)
   - LoginRequest (email, password)
   - AuthResponse (accessToken, tokenType default "Bearer", expiresInSeconds,
     userId, role)
3. A simple UserMapper (plain class, not MapStruct) converting User entity
   to a minimal UserSummary DTO (id, email, role) for internal use.

Do not add a refresh token DTO or entity — this service issues a single
stateless JWT access token only, no refresh-token flow.

Prompt 5.3 — JWT utility

In auth-service, create a JwtUtil component (com.fooddelivery.authservice.security)
that:
- Generates a signed JWT (io.jsonwebtoken / jjwt 0.12.x) containing claims:
  sub (user id), email, role, iat, exp
- Sets expiry to 24 hours from issuance (configurable via
  jwt.expiration-ms in application.yml, default 86400000)
- Uses a shared secret from jwt.secret in application.yml (must match the
  same value configured in api-gateway)
- Exposes: generateToken(User user), extractUserId(String token),
  extractRole(String token), isTokenValid(String token)

Add jjwt-api, jjwt-impl, jjwt-jackson (0.12.x) to auth-service's pom.xml.

Prompt 5.4 — Service layer

In auth-service, generate an AuthService (com.fooddelivery.authservice.service)
with:
- register(RegisterRequest request): validates email/phone aren't already
  taken (throw a custom DuplicateResourceException if so), hashes the
  password with BCrypt (BCryptPasswordEncoder), looks up the role_id by
  role name, saves the new user, returns a UserSummary
- login(LoginRequest request): finds the user by email, verifies the
  password with BCrypt, throws a custom InvalidCredentialsException if
  the user doesn't exist or password doesn't match, throws a custom
  AccountDisabledException if is_active is false, otherwise generates
  a JWT via JwtUtil and returns an AuthResponse

Do not implement any refresh/logout logic — there is no refresh token to
revoke in this design.

Prompt 5.5 — Controller + internal endpoint

In auth-service, generate an AuthController exposing:
- POST /api/auth/register -> calls AuthService.register, returns 201
- POST /api/auth/login -> calls AuthService.login, returns 200 with AuthResponse

Also generate a separate InternalUserController exposing:
- GET /internal/auth/users/{id} -> returns a UserSummary (id, email, role)
  for other services to validate a user_id via Feign. This endpoint is not
  routed through the API Gateway and is intended for service-to-service calls
  only.

Both controllers go in com.fooddelivery.authservice.controller.
Use @Valid on request bodies and return ResponseEntity<T>.

Prompt 5.6 — Global exception handling

In auth-service, generate a GlobalExceptionHandler (@RestControllerAdvice) in
com.fooddelivery.authservice.exception that handles:
- DuplicateResourceException -> 409 Conflict
- InvalidCredentialsException -> 401 Unauthorized
- AccountDisabledException -> 403 Forbidden
- MethodArgumentNotValidException (Bean Validation failures) -> 400 Bad Request
  with a map of field -> error message
- Exception (catch-all) -> 500 Internal Server Error

Return a consistent JSON error shape for every case:
{ "timestamp": "...", "status": 401, "error": "Invalid Credentials",
  "message": "..." }

This exact response shape and handler structure will be reused as the template
for every other service's exception handler — keep it generic enough to copy.

Prompt 5.7 — application.yml + Swagger

In auth-service, generate application.yml that:
- Sets spring.application.name to "auth-service"
- Pulls config from Config Server (http://localhost:8888) and registers with
  Eureka (http://localhost:8761/eureka)
- Runs on port 8081
- Configures spring.datasource for PostgreSQL database auth_service_db
- Sets spring.jpa.hibernate.ddl-auto to "validate" (schema is created via the
  SQL script, not Hibernate auto-generation)
- Adds jwt.secret and jwt.expiration-ms placeholders

Also add springdoc-openapi-starter-webmvc-ui to pom.xml and confirm
Swagger UI will be available at /swagger-ui.html with zero extra config
beyond the dependency.

Prompt 5.8 — Unit tests

In auth-service, generate JUnit 5 + Mockito unit tests for AuthService covering:
- register() succeeds with a new email/phone
- register() throws DuplicateResourceException when email already exists
- login() succeeds with correct credentials and returns a non-null token
- login() throws InvalidCredentialsException with wrong password
- login() throws AccountDisabledException when is_active is false

Mock UserRepository, RoleRepository, PasswordEncoder, and JwtUtil —
do not hit a real database or generate a real JWT in these tests.



Customer Service → Restaurant Service → Menu Service

Scope: Customer Service → Restaurant Service → Menu Service — each service taken through every layer before moving to the next.
Build order reason: Customer and Restaurant Service are independent of each other, but Menu Service Feign-calls Restaurant Service, so Restaurant must exist first.
Companion docs: Database-Schema-Design.md (Sections 4, 5, 6), Day1-Copilot-Prompt-Sequence.md



0. OpenFeign Convention — introduced here, reused for every remaining service

This is the first day any service makes an outbound call to another. Establish the convention now and paste it into every future Feign-client prompt for the rest of the week.


Package layout

com.fooddelivery.<service>.client/
├── AuthServiceClient.java        # @FeignClient interface
├── AuthServiceFallback.java      # fallback implementation
└── dto/
    └── UserSummaryResponse.java  # minimal DTO, only the fields this caller needs

Rules to paste into every Feign-client prompt


@FeignClient(name = "<eureka-app-name>", fallback = <X>Fallback.class) — reference the Eureka application name, never a hardcoded host/port.

The Feign interface only declares /internal/... endpoints — never a public customer-facing path.

Response DTOs are local to the calling service and contain only the fields it actually needs.

Every @FeignClient gets a fallback class that throws a clear ServiceUnavailableException rather than letting a raw Feign exception bubble up.

Timeouts come from the shared application.yml block below — don't set them per-client.


feign:
  client:
    config:
      default:
        connectTimeout: 3000
        readTimeout: 5000
        loggerLevel: basic


PART A — Customer Service

Reference Database-Schema-Design.md Section 4 (customers, addresses) — do not add fields beyond what's listed.


Prompt A.1 — Entities

In the customer-service module (Java 21, Spring Boot 3.2.x, Spring Data JPA,
Lombok), generate JPA entities for exactly these two tables. Do not add any
fields beyond what's listed.

Table: customers
- id: BIGSERIAL PK
- user_id: BIGINT NOT NULL UNIQUE (logical reference to Auth Service users.id,
  no JPA relationship annotation — this is a plain value column, not a
  cross-database FK)
- full_name: VARCHAR(100) NOT NULL
- email: VARCHAR(150) NOT NULL
- phone: VARCHAR(15) NOT NULL
- date_of_birth: DATE, nullable
- created_at: TIMESTAMP NOT NULL, auto-set on insert
- updated_at: TIMESTAMP NOT NULL, auto-updated on every update

Table: addresses
- id: BIGSERIAL PK
- customer_id: BIGINT NOT NULL, FK to customers.id
- label: VARCHAR(30) NOT NULL DEFAULT 'HOME'
- address_line1: VARCHAR(200) NOT NULL
- address_line2: VARCHAR(200), nullable
- city: VARCHAR(50) NOT NULL
- state: VARCHAR(50) NOT NULL
- pincode: VARCHAR(10) NOT NULL
- latitude: NUMERIC(9,6), nullable
- longitude: NUMERIC(9,6), nullable
- is_default: BOOLEAN NOT NULL DEFAULT FALSE
- created_at: TIMESTAMP NOT NULL, auto-set on insert

Use package com.fooddelivery.customerservice.entity.
Use @OneToMany (customers -> addresses) with cascade = ALL and
orphanRemoval = true, since addresses are owned entirely by their customer.
Use @CreationTimestamp / @UpdateTimestamp for the timestamp columns.

Prompt A.2 — Repositories and DTOs

In customer-service, generate:
1. CustomerRepository (Spring Data JPA) with findByUserId(Long userId)
2. AddressRepository (Spring Data JPA) with findByCustomerId(Long customerId)
3. DTOs in com.fooddelivery.customerservice.dto:
   - CreateCustomerRequest (userId, fullName, email, phone, dateOfBirth —
     Bean Validation: fullName not blank, email format, phone pattern)
   - CustomerResponse (id, userId, fullName, email, phone, dateOfBirth,
     createdAt, updatedAt)
   - CreateAddressRequest (label, addressLine1, addressLine2, city, state,
     pincode, latitude, longitude, isDefault — Bean Validation on the
     required fields)
   - AddressResponse (all address fields)
4. A plain CustomerMapper class converting between entity and DTOs
   (no MapStruct, just manual mapping methods).

Prompt A.3 — Feign client to Auth Service

In customer-service, generate an OpenFeign client following this exact
convention:

- Package com.fooddelivery.customerservice.client
- AuthServiceClient interface: @FeignClient(name = "auth-service",
  fallback = AuthServiceFallback.class), with one method:
  GET /internal/auth/users/{id} -> returns UserSummaryResponse
  (fields: id, email, role)
- UserSummaryResponse DTO in client/dto, containing only id, email, role
- AuthServiceFallback implements AuthServiceClient, and on any call throws
  a custom ServiceUnavailableException("Authentication service is
  unavailable") rather than returning a fake/default user

Add spring-cloud-starter-openfeign to customer-service's pom.xml and
@EnableFeignClients to the main application class.

Prompt A.4 — Service layer

In customer-service, generate a CustomerService
(com.fooddelivery.customerservice.service) with:

- createCustomerProfile(CreateCustomerRequest request): calls
  AuthServiceClient.getUser(request.getUserId()) first to confirm the
  user exists and has role CUSTOMER (throw a custom
  InvalidUserRoleException if the role doesn't match); throws a custom
  DuplicateResourceException if a customer profile already exists for
  that user_id; otherwise saves and returns a CustomerResponse
- getCustomerByUserId(Long userId): returns CustomerResponse or throws
  ResourceNotFoundException
- updateCustomerProfile(Long userId, ...): updates fullName/phone/dateOfBirth
- addAddress(Long customerId, CreateAddressRequest request): if
  request.isDefault() is true, first clears is_default on any existing
  address for that customer, then saves the new one
- listAddresses(Long customerId)
- updateAddress(Long addressId, ...) / deleteAddress(Long addressId)

Also generate a simple AlsoUsedInternally method:
getCustomerById(Long id): returns a CustomerResponse for internal Feign
consumption by other services.

Prompt A.5 — Controllers

In customer-service, generate two controllers:

1. CustomerController (public API):
   POST   /api/customers                       -> createCustomerProfile
   GET    /api/customers/me                     -> getCustomerByUserId
                                                    (userId from JWT claim,
                                                    read via a request
                                                    attribute/header set by
                                                    the gateway — assume a
                                                    header X-User-Id for now)
   PUT    /api/customers/me                     -> updateCustomerProfile
   POST   /api/customers/me/addresses           -> addAddress
   GET    /api/customers/me/addresses           -> listAddresses
   PUT    /api/customers/me/addresses/{id}      -> updateAddress
   DELETE /api/customers/me/addresses/{id}      -> deleteAddress

2. InternalCustomerController:
   GET /internal/customers/{id} -> getCustomerById, for other services'
   Feign clients (e.g. Order Service fetching the delivery address)

Use @Valid on all request bodies, ResponseEntity<T> return types, and
package com.fooddelivery.customerservice.controller.

Prompt A.6 — Global exception handling

In customer-service, generate a GlobalExceptionHandler
(@RestControllerAdvice) in com.fooddelivery.customerservice.exception
using the exact same response shape and structure as auth-service's
GlobalExceptionHandler (paste that file here as a reference), but handling
this service's exceptions instead:
- DuplicateResourceException -> 409
- ResourceNotFoundException -> 404
- InvalidUserRoleException -> 400
- ServiceUnavailableException -> 503
- MethodArgumentNotValidException -> 400 with field error map
- Exception (catch-all) -> 500

Prompt A.7 — application.yml + Swagger

In customer-service, generate application.yml that:
- Sets spring.application.name to "customer-service"
- Pulls config from Config Server, registers with Eureka
- Runs on port 8082
- Configures PostgreSQL datasource for customer_service_db
- Sets spring.jpa.hibernate.ddl-auto to "validate"
- Includes the shared Feign timeout block from Section 0 of this guide

Add springdoc-openapi-starter-webmvc-ui to pom.xml (same as auth-service).

Prompt A.8 — Unit tests

In customer-service, generate JUnit 5 + Mockito tests for CustomerService:
- createCustomerProfile() succeeds when AuthServiceClient returns a user
  with role CUSTOMER
- createCustomerProfile() throws InvalidUserRoleException when the role
  isn't CUSTOMER
- createCustomerProfile() throws DuplicateResourceException when a profile
  already exists for that user_id
- addAddress() clears the previous default address when the new one is
  marked default
- getCustomerByUserId() throws ResourceNotFoundException when not found

Mock CustomerRepository, AddressRepository, and AuthServiceClient.


PART B — Restaurant Service

Reference Database-Schema-Design.md Section 5 (restaurants — single table).


Prompt B.1 — Entity

In the restaurant-service module, generate a JPA entity for exactly this
table. Do not add any fields beyond what's listed.

Table: restaurants
- id: BIGSERIAL PK
- owner_user_id: BIGINT NOT NULL (logical reference to Auth Service
  users.id, plain value column, no JPA relationship)
- name: VARCHAR(150) NOT NULL
- description: TEXT, nullable
- email: VARCHAR(150), nullable
- phone: VARCHAR(15) NOT NULL
- address_line: VARCHAR(200) NOT NULL
- city: VARCHAR(50) NOT NULL
- state: VARCHAR(50) NOT NULL
- pincode: VARCHAR(10) NOT NULL
- latitude: NUMERIC(9,6), nullable
- longitude: NUMERIC(9,6), nullable
- approval_status: VARCHAR(20) NOT NULL DEFAULT 'PENDING'
  (values: PENDING, APPROVED, REJECTED)
- is_open: BOOLEAN NOT NULL DEFAULT FALSE
- opening_time: TIME, nullable
- closing_time: TIME, nullable
- avg_rating: NUMERIC(2,1) NOT NULL DEFAULT 0.0
- created_at: TIMESTAMP NOT NULL, auto-set on insert
- updated_at: TIMESTAMP NOT NULL, auto-updated on every update

Use package com.fooddelivery.restaurantservice.entity.
Use @CreationTimestamp / @UpdateTimestamp for the timestamp columns.

Prompt B.2 — Repository and DTOs

In restaurant-service, generate:
1. RestaurantRepository (Spring Data JPA) with:
   - findByCityContainingIgnoreCase(String city)
   - findByApprovalStatus(String status)
2. DTOs in com.fooddelivery.restaurantservice.dto:
   - CreateRestaurantRequest (ownerUserId, name, description, email, phone,
     addressLine, city, state, pincode, latitude, longitude — Bean
     Validation on name/phone/addressLine/city/state/pincode not blank)
   - UpdateRestaurantRequest (same editable fields, all optional/nullable
     for partial update)
   - RestaurantResponse (all columns)
   - UpdateApprovalStatusRequest (status — must be APPROVED or REJECTED)
3. A plain RestaurantMapper class (manual mapping, no MapStruct).

Prompt B.3 — Feign client to Auth Service

In restaurant-service, generate an OpenFeign client following the same
convention established in customer-service:

- Package com.fooddelivery.restaurantservice.client
- AuthServiceClient interface: @FeignClient(name = "auth-service",
  fallback = AuthServiceFallback.class), method:
  GET /internal/auth/users/{id} -> UserSummaryResponse (id, email, role)
- AuthServiceFallback throws ServiceUnavailableException on any call

Add spring-cloud-starter-openfeign to pom.xml and @EnableFeignClients to
the main application class.

Prompt B.4 — Service layer

In restaurant-service, generate a RestaurantService
(com.fooddelivery.restaurantservice.service) with:

- registerRestaurant(CreateRestaurantRequest request): calls
  AuthServiceClient.getUser(request.getOwnerUserId()) to confirm the user
  exists with role RESTAURANT_OWNER (throw InvalidUserRoleException if
  not); saves with approval_status defaulting to PENDING; returns
  RestaurantResponse
- getRestaurantById(Long id): throws ResourceNotFoundException if missing
- searchRestaurants(String city): returns only restaurants with
  approval_status = APPROVED
- updateRestaurant(Long id, UpdateRestaurantRequest request): owner-only
  update of editable profile fields
- updateApprovalStatus(Long id, UpdateApprovalStatusRequest request):
  admin-only, sets approval_status
- toggleOpenStatus(Long id, boolean isOpen): owner-only

Also generate getRestaurantForInternalUse(Long id): returns a
RestaurantResponse for internal Feign consumption (used by Menu Service
and Order Service to confirm a restaurant exists and its approval_status/
is_open).

Prompt B.5 — Controllers

In restaurant-service, generate two controllers:

1. RestaurantController (public API):
   POST /api/restaurants                  -> registerRestaurant
   GET  /api/restaurants?city=             -> searchRestaurants
   GET  /api/restaurants/{id}              -> getRestaurantById
   PUT  /api/restaurants/{id}              -> updateRestaurant (owner)
   PUT  /api/restaurants/{id}/status        -> updateApprovalStatus (admin)
   PUT  /api/restaurants/{id}/open-status   -> toggleOpenStatus (owner)

2. InternalRestaurantController:
   GET /internal/restaurants/{id} -> getRestaurantForInternalUse, for
   Menu Service and Order Service Feign calls

Use @Valid on request bodies, ResponseEntity<T>, package
com.fooddelivery.restaurantservice.controller.

Prompt B.6 — Global exception handling

In restaurant-service, generate a GlobalExceptionHandler
(@RestControllerAdvice) in com.fooddelivery.restaurantservice.exception,
using the same response shape as auth-service's handler (paste it as
reference), handling:
- ResourceNotFoundException -> 404
- InvalidUserRoleException -> 400
- ServiceUnavailableException -> 503
- MethodArgumentNotValidException -> 400 with field error map
- Exception (catch-all) -> 500

Prompt B.7 — application.yml + Swagger

In restaurant-service, generate application.yml that:
- Sets spring.application.name to "restaurant-service"
- Pulls config from Config Server, registers with Eureka
- Runs on port 8083
- Configures PostgreSQL datasource for restaurant_service_db
- Sets spring.jpa.hibernate.ddl-auto to "validate"
- Includes the shared Feign timeout block from Section 0

Add springdoc-openapi-starter-webmvc-ui to pom.xml.

Prompt B.8 — Unit tests

In restaurant-service, generate JUnit 5 + Mockito tests for
RestaurantService:
- registerRestaurant() succeeds when the owner has role RESTAURANT_OWNER
- registerRestaurant() throws InvalidUserRoleException for the wrong role
- searchRestaurants() only returns APPROVED restaurants
- updateApprovalStatus() correctly updates the status field
- getRestaurantById() throws ResourceNotFoundException when missing

Mock RestaurantRepository and AuthServiceClient.


PART C — Menu Service

Reference Database-Schema-Design.md Section 6 (categories, menu_items). Menu Service Feign-calls Restaurant Service — build this part last on Day 2.


Prompt C.1 — Entities

In the menu-service module, generate JPA entities for exactly these two
tables. Do not add any fields beyond what's listed.

Table: categories
- id: BIGSERIAL PK
- restaurant_id: BIGINT NOT NULL (logical reference to Restaurant Service
  restaurants.id, plain value column)
- name: VARCHAR(100) NOT NULL
- display_order: INT NOT NULL DEFAULT 0
- created_at: TIMESTAMP NOT NULL, auto-set on insert
- Unique constraint on (restaurant_id, name)

Table: menu_items
- id: BIGSERIAL PK
- restaurant_id: BIGINT NOT NULL (denormalized copy, same value as its
  category's restaurant_id, for fast filtering)
- category_id: BIGINT NOT NULL, FK to categories.id
- name: VARCHAR(150) NOT NULL
- description: TEXT, nullable
- price: NUMERIC(10,2) NOT NULL, must be >= 0
- is_veg: BOOLEAN NOT NULL DEFAULT TRUE
- is_available: BOOLEAN NOT NULL DEFAULT TRUE
- image_url: VARCHAR(500), nullable
- created_at: TIMESTAMP NOT NULL, auto-set on insert
- updated_at: TIMESTAMP NOT NULL, auto-updated on every update

Use package com.fooddelivery.menuservice.entity.
Use @ManyToOne (menu_items -> categories) with cascade on delete
(ON DELETE CASCADE at the DB level, orphanRemoval not required here).
Use @CreationTimestamp / @UpdateTimestamp for timestamps.

Prompt C.2 — Repositories and DTOs

In menu-service, generate:
1. CategoryRepository with findByRestaurantIdOrderByDisplayOrder(Long restaurantId)
2. MenuItemRepository with findByCategoryId(Long categoryId) and
   findByRestaurantIdAndIsAvailableTrue(Long restaurantId)
3. DTOs in com.fooddelivery.menuservice.dto:
   - CreateCategoryRequest (name, displayOrder)
   - CategoryResponse (id, restaurantId, name, displayOrder)
   - CreateMenuItemRequest (categoryId, name, description, price, isVeg,
     imageUrl — Bean Validation: name not blank, price >= 0)
   - UpdateMenuItemRequest (name, description, price, isVeg, isAvailable,
     imageUrl — all optional for partial update)
   - MenuItemResponse (all columns)
   - RestaurantMenuResponse (restaurantId, list of categories each
     containing their menu items — a nested structure for the "get full
     menu" endpoint)
4. A plain MenuMapper class (manual mapping, no MapStruct).

Prompt C.3 — Feign client to Restaurant Service

In menu-service, generate an OpenFeign client following the same
convention established in customer-service and restaurant-service:

- Package com.fooddelivery.menuservice.client
- RestaurantServiceClient interface: @FeignClient(name = "restaurant-service",
  fallback = RestaurantServiceFallback.class), method:
  GET /internal/restaurants/{id} -> RestaurantSummaryResponse
  (fields: id, approvalStatus, isOpen)
- RestaurantServiceFallback throws ServiceUnavailableException on any call

Add spring-cloud-starter-openfeign to pom.xml and @EnableFeignClients to
the main application class.

Prompt C.4 — Service layer

In menu-service, generate:

1. CategoryService (com.fooddelivery.menuservice.service):
   - createCategory(Long restaurantId, CreateCategoryRequest request):
     calls RestaurantServiceClient.getRestaurant(restaurantId) first and
     throws RestaurantNotApprovedException if approvalStatus isn't
     APPROVED; throws DuplicateResourceException if the category name
     already exists for that restaurant; otherwise saves
   - listCategories(Long restaurantId)

2. MenuItemService:
   - createMenuItem(Long restaurantId, CreateMenuItemRequest request):
     validates the category belongs to restaurantId, sets restaurant_id
     denormalized on the item, saves
   - updateMenuItem(Long itemId, UpdateMenuItemRequest request)
   - deleteMenuItem(Long itemId)
   - getFullMenu(Long restaurantId): returns RestaurantMenuResponse with
     categories nested with their items, ordered by display_order
   - getMenuItemById(Long itemId): for internal Feign consumption by
     Order Service (returns MenuItemResponse with price and is_available)

Prompt C.5 — Controllers

In menu-service, generate two controllers:

1. MenuController (public API):
   POST /api/restaurants/{restaurantId}/categories -> createCategory
   GET  /api/restaurants/{restaurantId}/menu         -> getFullMenu
   POST /api/restaurants/{restaurantId}/items        -> createMenuItem
   PUT  /api/items/{itemId}                          -> updateMenuItem
   DELETE /api/items/{itemId}                        -> deleteMenuItem

2. InternalMenuController:
   GET /internal/items/{itemId} -> getMenuItemById, for Order Service's
   Feign client to confirm current price/availability at cart-add time

Use @Valid on request bodies, ResponseEntity<T>, package
com.fooddelivery.menuservice.controller.

Prompt C.6 — Global exception handling

In menu-service, generate a GlobalExceptionHandler
(@RestControllerAdvice) in com.fooddelivery.menuservice.exception, using
the same response shape as auth-service's handler (paste it as
reference), handling:
- ResourceNotFoundException -> 404
- DuplicateResourceException -> 409
- RestaurantNotApprovedException -> 400
- ServiceUnavailableException -> 503
- MethodArgumentNotValidException -> 400 with field error map
- Exception (catch-all) -> 500

Prompt C.7 — application.yml + Swagger

In menu-service, generate application.yml that:
- Sets spring.application.name to "menu-service"
- Pulls config from Config Server, registers with Eureka
- Runs on port 8084
- Configures PostgreSQL datasource for menu_service_db
- Sets spring.jpa.hibernate.ddl-auto to "validate"
- Includes the shared Feign timeout block from Section 0

Add springdoc-openapi-starter-webmvc-ui to pom.xml.

Prompt C.8 — Unit tests

In menu-service, generate JUnit 5 + Mockito tests:
- CategoryService.createCategory() throws RestaurantNotApprovedException
  when the restaurant's approvalStatus isn't APPROVED
- CategoryService.createCategory() throws DuplicateResourceException for
  a duplicate name within the same restaurant
- MenuItemService.getFullMenu() returns categories ordered by display_order
  with their nested items
- MenuItemService.createMenuItem() correctly denormalizes restaurant_id
  from the category

Mock CategoryRepository, MenuItemRepository, and RestaurantServiceClient.



Order Service (Cart + Orders + Payment tracking)

Scope: Cart, Orders, and Payment tracking — the heaviest single service in this project (5 tables, the busiest set of Feign calls, the most business logic). Split across two days: Day 3 gets you to a working checkout; Day 4 adds payment recording, controllers, and tests.
Companion docs: Database-Schema-Design.md Section 7, Day1-Copilot-Prompt-Sequence.md, Day2-Copilot-Prompt-Sequence.md



0. Prerequisite — one small addition to Customer Service

Order Service needs to validate that a delivery_address_id actually belongs to the customer placing the order. Day 2's Customer Service only exposed GET /internal/customers/{id} — add one more internal endpoint before starting Order Service.


Prompt 0.1 — Extend Customer Service's internal controller

In the existing customer-service module, add one more endpoint to
InternalCustomerController:

GET /internal/customers/{customerId}/addresses/{addressId}
-> returns the AddressResponse if that address belongs to that customer,
   otherwise throws ResourceNotFoundException (handled by the existing
   GlobalExceptionHandler, resulting in a 404)

Add the corresponding method to CustomerService:
validateAndGetAddress(Long customerId, Long addressId)

Do not change any other existing files in customer-service.


PART 1 (Day 3) — Entities through Checkout

Prompt 1.1 — Entities

In the order-service module (Java 21, Spring Boot 3.2.x, Spring Data JPA,
Lombok), generate JPA entities for exactly these five tables. Do not add
any fields beyond what's listed.

Table: carts
- id: BIGSERIAL PK
- customer_id: BIGINT NOT NULL UNIQUE (logical reference to Customer
  Service customers.id — one cart per customer, reused across sessions)
- restaurant_id: BIGINT, nullable (logical reference to Restaurant
  Service restaurants.id; null when the cart is empty)
- created_at: TIMESTAMP NOT NULL, auto-set on insert
- updated_at: TIMESTAMP NOT NULL, auto-updated on every update

Table: cart_items
- id: BIGSERIAL PK
- cart_id: BIGINT NOT NULL, FK to carts.id
- menu_item_id: BIGINT NOT NULL (logical reference to Menu Service
  menu_items.id)
- item_name: VARCHAR(150) NOT NULL (snapshot at add-time)
- unit_price: NUMERIC(10,2) NOT NULL (snapshot at add-time)
- quantity: INT NOT NULL DEFAULT 1, must be > 0
- created_at: TIMESTAMP NOT NULL, auto-set on insert
- Unique constraint on (cart_id, menu_item_id)

Table: orders
- id: BIGSERIAL PK
- customer_id: BIGINT NOT NULL
- restaurant_id: BIGINT NOT NULL
- delivery_address_id: BIGINT NOT NULL
- order_status: VARCHAR(30) NOT NULL DEFAULT 'PLACED'
  (values: PLACED, ACCEPTED, PREPARING, READY_FOR_PICKUP, PICKED_UP,
  DELIVERED, CANCELLED)
- subtotal: NUMERIC(10,2) NOT NULL
- delivery_fee: NUMERIC(10,2) NOT NULL DEFAULT 0
- total_amount: NUMERIC(10,2) NOT NULL
- payment_method: VARCHAR(20) NOT NULL (values: ONLINE, COD)
- payment_status: VARCHAR(20) NOT NULL DEFAULT 'PENDING'
  (values: PENDING, PAID, FAILED, REFUNDED)
- placed_at: TIMESTAMP NOT NULL, auto-set on insert
- updated_at: TIMESTAMP NOT NULL, auto-updated on every update

Table: order_items
- id: BIGSERIAL PK
- order_id: BIGINT NOT NULL, FK to orders.id
- menu_item_id: BIGINT NOT NULL
- item_name: VARCHAR(150) NOT NULL (snapshot at order time)
- unit_price: NUMERIC(10,2) NOT NULL (snapshot at order time)
- quantity: INT NOT NULL, must be > 0
- subtotal: NUMERIC(10,2) NOT NULL (unit_price * quantity)

Table: payment_transactions
- id: BIGSERIAL PK
- order_id: BIGINT NOT NULL, FK to orders.id
- transaction_ref: VARCHAR(100), nullable, unique (null for COD)
- amount: NUMERIC(10,2) NOT NULL
- status: VARCHAR(20) NOT NULL DEFAULT 'PENDING'
  (values: PENDING, PAID, FAILED, REFUNDED)
- gateway_response: TEXT, nullable
- created_at: TIMESTAMP NOT NULL, auto-set on insert

Use package com.fooddelivery.orderservice.entity.
Use @OneToMany (carts -> cart_items, orders -> order_items,
orders -> payment_transactions) with cascade = ALL and
orphanRemoval = true where the child is fully owned by the parent
(cart_items and order_items; payment_transactions is append-only so
cascade = PERSIST is enough, no orphanRemoval).
Use @CreationTimestamp / @UpdateTimestamp for timestamp columns.

Prompt 1.2 — Repositories and DTOs

In order-service, generate:

1. CartRepository with findByCustomerId(Long customerId)
2. CartItemRepository with findByCartIdAndMenuItemId(Long cartId, Long menuItemId)
3. OrderRepository with findByCustomerId(Long customerId) and
   findByRestaurantId(Long restaurantId)
4. OrderItemRepository (standard CRUD only)
5. PaymentTransactionRepository with findByOrderId(Long orderId)

DTOs in com.fooddelivery.orderservice.dto:
- AddCartItemRequest (menuItemId, quantity — Bean Validation: quantity min 1)
- UpdateCartItemRequest (quantity)
- CartItemResponse (id, menuItemId, itemName, unitPrice, quantity, lineSubtotal)
- CartResponse (id, restaurantId, items: List<CartItemResponse>, subtotal)
- CheckoutRequest (deliveryAddressId, paymentMethod — Bean Validation:
  paymentMethod must be ONLINE or COD)
- OrderItemResponse (id, menuItemId, itemName, unitPrice, quantity, subtotal)
- OrderResponse (id, customerId, restaurantId, deliveryAddressId,
  orderStatus, subtotal, deliveryFee, totalAmount, paymentMethod,
  paymentStatus, placedAt, items: List<OrderItemResponse>)
- UpdateOrderStatusRequest (orderStatus)
- PaymentResultRequest (transactionRef, status, gatewayResponse)

A plain OrderMapper class (manual mapping, no MapStruct) covering
cart/cartItem/order/orderItem entity <-> DTO conversions.

Prompt 1.3 — Feign clients

In order-service, generate three OpenFeign clients following the
convention established in customer-service/restaurant-service/menu-service
(package com.fooddelivery.orderservice.client, one subfolder per client):

1. CustomerServiceClient (@FeignClient(name = "customer-service",
   fallback = CustomerServiceFallback.class)):
   GET /internal/customers/{customerId}/addresses/{addressId}
   -> AddressSummaryResponse (id, customerId, addressLine1, city)

2. RestaurantServiceClient (@FeignClient(name = "restaurant-service",
   fallback = RestaurantServiceFallback.class)):
   GET /internal/restaurants/{id}
   -> RestaurantSummaryResponse (id, approvalStatus, isOpen)

3. MenuServiceClient (@FeignClient(name = "menu-service",
   fallback = MenuServiceFallback.class)):
   GET /internal/items/{itemId}
   -> MenuItemSummaryResponse (id, restaurantId, name, price, isAvailable)

Each fallback class throws ServiceUnavailableException with a message
naming which service is unreachable. Response DTOs go in each client's
own dto subfolder, containing only the fields listed above.

Add spring-cloud-starter-openfeign to pom.xml and @EnableFeignClients to
the main application class. Also add the shared Feign timeout block to
application.yml (same block used in customer/restaurant/menu-service).

Prompt 1.4 — CartService

In order-service, generate a CartService (com.fooddelivery.orderservice.service)
with:

- getOrCreateCart(Long customerId): finds the customer's existing cart row
  or creates a new empty one (restaurant_id null)

- addItemToCart(Long customerId, AddCartItemRequest request):
  1. call MenuServiceClient.getMenuItem(request.getMenuItemId()); throw
     ResourceNotFoundException if not found, throw
     ItemUnavailableException if isAvailable is false
  2. get or create the cart via getOrCreateCart
  3. if cart.restaurantId is null, set it to the item's restaurantId
  4. if cart.restaurantId is already set to a DIFFERENT restaurant, throw
     a custom MultiRestaurantCartException ("Cart already contains items
     from a different restaurant — clear your cart first")
  5. if a cart_item for that menu_item_id already exists in this cart,
     increment its quantity by request.getQuantity(); otherwise create a
     new cart_item snapshotting item_name and unit_price from the Feign
     response
  6. return the updated CartResponse

- updateCartItemQuantity(Long cartItemId, UpdateCartItemRequest request):
  updates quantity, or deletes the row if the new quantity is 0

- removeCartItem(Long cartItemId): deletes the cart_item; if the cart
  becomes empty, reset cart.restaurantId to null

- getCart(Long customerId): returns CartResponse with computed subtotal
  (sum of unit_price * quantity across items)

- clearCart(Long customerId): deletes all cart_items and resets
  restaurant_id to null (used internally after a successful checkout)

Prompt 1.5 — OrderService: checkout logic

In order-service, generate an OrderService (com.fooddelivery.orderservice.service)
with a checkout(Long customerId, CheckoutRequest request) method that:

1. Loads the customer's cart via CartService; throws EmptyCartException
   if it has no items
2. Calls CustomerServiceClient.validateAddress(customerId,
   request.getDeliveryAddressId()) to confirm the address belongs to this
   customer; propagate ResourceNotFoundException if invalid
3. Calls RestaurantServiceClient.getRestaurant(cart.getRestaurantId());
   throws RestaurantClosedException if approvalStatus isn't APPROVED or
   isOpen is false
4. Computes subtotal as the sum of (unit_price * quantity) across all
   cart_items
5. Applies a flat delivery fee read from application.yml
   (order.delivery-fee-flat, default 40.00) — do not hardcode the value
   in the service class
6. Computes total_amount = subtotal + delivery_fee
7. Creates the Order row (order_status = PLACED, payment_status = PENDING),
   and creates one OrderItem per cart_item, copying item_name/unit_price/
   quantity and computing each line's subtotal
8. If payment_method is ONLINE, creates a PaymentTransaction row
   (status = PENDING, transaction_ref = null, amount = total_amount)
9. Calls CartService.clearCart(customerId) to empty the cart for next time
10. Returns the created OrderResponse

Wrap steps 4-9 in a single @Transactional method so a failure partway
through rolls back cleanly (no half-created order).


PART 2 (Day 4) — Payment, Controllers, Tests

Prompt 2.1 — PaymentService (simulated payment tracking)

In order-service, generate a PaymentService (com.fooddelivery.orderservice.service)
with:

- recordPaymentResult(Long orderId, PaymentResultRequest request):
  1. loads the order; throws ResourceNotFoundException if missing
  2. throws InvalidPaymentMethodException if the order's payment_method
     isn't ONLINE (COD orders don't go through this flow)
  3. finds the order's existing PENDING payment_transactions row (or
     creates one if somehow missing) and updates its status,
     transaction_ref, and gateway_response from the request
  4. updates the order's payment_status to match (PAID or FAILED)
  5. returns the updated OrderResponse

This method simulates a payment gateway callback — it does not call any
real external payment provider. Document that clearly in a Javadoc
comment on the method.

- getPaymentHistory(Long orderId): returns all payment_transactions rows
  for an order (List<PaymentTransactionResponse> — add this DTO with
  id, transactionRef, amount, status, createdAt)

Prompt 2.2 — Order status update logic

In order-service, add to OrderService:

- updateOrderStatus(Long orderId, UpdateOrderStatusRequest request):
  validates the requested order_status is one of the 7 valid values,
  rejects the update with InvalidOrderStatusTransitionException if the
  requested status would move backwards in this sequence: PLACED ->
  ACCEPTED -> PREPARING -> READY_FOR_PICKUP -> PICKED_UP -> DELIVERED
  (CANCELLED is allowed from any status except DELIVERED), otherwise
  updates order_status and returns the updated OrderResponse

- getOrderById(Long orderId): throws ResourceNotFoundException if missing

- getOrdersForCustomer(Long customerId)

- getOrdersForRestaurant(Long restaurantId)

- getOrderForInternalUse(Long orderId): returns OrderResponse for
  internal Feign consumption by Delivery Service and Review Service

Prompt 2.3 — Controllers

In order-service, generate three controllers:

1. CartController (public API), package
   com.fooddelivery.orderservice.controller:
   POST   /api/cart/items            -> addItemToCart
   GET    /api/cart                  -> getCart
   PUT    /api/cart/items/{id}       -> updateCartItemQuantity
   DELETE /api/cart/items/{id}       -> removeCartItem
   (customerId comes from an X-User-Id header, same convention as
   customer-service's controller)

2. OrderController (public API):
   POST /api/cart/checkout           -> checkout
   GET  /api/orders/{id}             -> getOrderById
   GET  /api/orders                  -> getOrdersForCustomer or
                                         getOrdersForRestaurant depending
                                         on an X-User-Role header
                                         (CUSTOMER vs RESTAURANT_OWNER)
   PUT  /api/orders/{id}/status      -> updateOrderStatus
   POST /api/orders/{id}/payment     -> recordPaymentResult
   GET  /api/orders/{id}/payments    -> getPaymentHistory

3. InternalOrderController:
   GET /internal/orders/{id} -> getOrderForInternalUse, for Delivery
   Service and Review Service Feign calls

Use @Valid on request bodies, ResponseEntity<T> return types.

Prompt 2.4 — Global exception handling

In order-service, generate a GlobalExceptionHandler
(@RestControllerAdvice) in com.fooddelivery.orderservice.exception, using
the same response shape as auth-service's handler (paste it as
reference), handling:
- ResourceNotFoundException -> 404
- EmptyCartException -> 400
- ItemUnavailableException -> 400
- MultiRestaurantCartException -> 409
- RestaurantClosedException -> 400
- InvalidPaymentMethodException -> 400
- InvalidOrderStatusTransitionException -> 409
- ServiceUnavailableException -> 503
- MethodArgumentNotValidException -> 400 with field error map
- Exception (catch-all) -> 500

Prompt 2.5 — application.yml + Swagger

In order-service, generate application.yml that:
- Sets spring.application.name to "order-service"
- Pulls config from Config Server, registers with Eureka
- Runs on port 8085
- Configures PostgreSQL datasource for order_service_db
- Sets spring.jpa.hibernate.ddl-auto to "validate"
- Includes the shared Feign timeout block
- Adds order.delivery-fee-flat: 40.00 as a configurable property

Add springdoc-openapi-starter-webmvc-ui to pom.xml.

Prompt 2.6 — Unit tests

In order-service, generate JUnit 5 + Mockito tests covering:

CartService:
- addItemToCart() throws ItemUnavailableException when the menu item
  isn't available
- addItemToCart() throws MultiRestaurantCartException when adding an
  item from a different restaurant than what's already in the cart
- addItemToCart() increments quantity instead of duplicating a row when
  the same item is added twice

OrderService:
- checkout() throws EmptyCartException for an empty cart
- checkout() throws RestaurantClosedException when the restaurant isn't
  APPROVED/open
- checkout() correctly computes subtotal, delivery_fee, and total_amount
- checkout() creates a PENDING payment_transactions row only when
  payment_method is ONLINE, not for COD
- checkout() clears the cart after a successful order
- updateOrderStatus() throws InvalidOrderStatusTransitionException when
  moving backwards (e.g. DELIVERED -> PREPARING)

PaymentService:
- recordPaymentResult() throws InvalidPaymentMethodException for a COD order
- recordPaymentResult() updates both the payment_transactions row and
  the order's payment_status together

Mock all repositories and all three Feign clients
(CustomerServiceClient, RestaurantServiceClient, MenuServiceClient).



Notification Service → Delivery Service

Scope: Notification Service (build first — no dependencies) → Delivery Service (depends on Order Service and Notification Service).
Design decision made before writing these prompts: Delivery Service owns the responsibility of keeping orders.order_status in sync with deliveries.delivery_status, via a Feign call back into Order Service. The client only ever talks to Delivery Service to update delivery progress — it never has to call both APIs itself.
Companion docs: Database-Schema-Design.md Sections 8–9, Day3-4-Copilot-Prompt-Sequence.md



0. Prerequisite — one small addition to Order Service

Prompt 0.1 — Internal status-update endpoint

In the existing order-service module, add one more endpoint to
InternalOrderController:

PUT /internal/orders/{id}/status
-> accepts the same UpdateOrderStatusRequest DTO already used by the
   public PUT /api/orders/{id}/status endpoint, reuses the existing
   OrderService.updateOrderStatus() method (same transition validation
   rules — no backwards moves, CANCELLED allowed except from DELIVERED),
   and returns the updated OrderResponse

This endpoint is for Delivery Service to call via Feign when a delivery's
status changes — it is not routed through the API Gateway. Do not change
any other existing files in order-service.


PART A — Notification Service (build first)

Reference Database-Schema-Design.md Section 9 (notifications — single table, append-only).


Prompt A.1 — Entity

In the notification-service module (Java 21, Spring Boot 3.2.x, Spring
Data JPA, Lombok), generate a JPA entity for exactly this table. Do not
add any fields beyond what's listed.

Table: notifications
- id: BIGSERIAL PK
- user_id: BIGINT NOT NULL (logical reference to Auth Service users.id)
- notification_type: VARCHAR(40) NOT NULL (values: ORDER_CONFIRMATION,
  PAYMENT_STATUS, DELIVERY_STATUS, RESTAURANT_APPROVAL,
  DELIVERY_PARTNER_APPROVAL)
- channel: VARCHAR(20) NOT NULL DEFAULT 'PUSH' (values: EMAIL, SMS, PUSH)
- title: VARCHAR(150) NOT NULL
- message: TEXT NOT NULL
- status: VARCHAR(20) NOT NULL DEFAULT 'PENDING' (values: PENDING, SENT, FAILED)
- sent_at: TIMESTAMP, nullable
- created_at: TIMESTAMP NOT NULL, auto-set on insert

Use package com.fooddelivery.notificationservice.entity.
Use @CreationTimestamp for created_at. This entity has no updated_at —
notifications are immutable after creation except for status/sent_at,
which are set once on dispatch.

Prompt A.2 — Repository and DTOs

In notification-service, generate:
1. NotificationRepository (Spring Data JPA) with
   findByUserIdOrderByCreatedAtDesc(Long userId)
2. DTOs in com.fooddelivery.notificationservice.dto:
   - SendNotificationRequest (userId, notificationType, channel, title,
     message — Bean Validation: notificationType must be one of the 5
     valid values, channel must be one of the 3 valid values, title and
     message not blank)
   - NotificationResponse (id, userId, notificationType, channel, title,
     message, status, sentAt, createdAt)

Prompt A.3 — Service layer

In notification-service, generate a NotificationService
(com.fooddelivery.notificationservice.service) with:

- send(SendNotificationRequest request): creates a notifications row
  with status = PENDING, then immediately simulates dispatch — log the
  notification via SLF4J (info level: "Sending {channel} notification to
  user {userId}: {title}"), set status = SENT and sent_at = now, save,
  and return the NotificationResponse.

  Add a clear Javadoc comment stating this simulates delivery — there is
  no real email/SMS/push provider integration in this project's scope.

- getNotificationsForUser(Long userId): returns
  List<NotificationResponse> ordered by most recent first

Prompt A.4 — Controllers

In notification-service, generate two controllers:

1. NotificationController (public API), package
   com.fooddelivery.notificationservice.controller:
   GET /api/notifications/me -> getNotificationsForUser (userId from an
   X-User-Id header, same convention as other services)

2. InternalNotificationController:
   POST /internal/notifications -> send, for other services' Feign
   clients to trigger a notification. Not routed through the API
   Gateway.

Use @Valid on request bodies, ResponseEntity<T>.

Prompt A.5 — Global exception handling

In notification-service, generate a GlobalExceptionHandler
(@RestControllerAdvice) in com.fooddelivery.notificationservice.exception,
using the same response shape as auth-service's handler (paste it as
reference), handling:
- MethodArgumentNotValidException -> 400 with field error map
- Exception (catch-all) -> 500

Prompt A.6 — application.yml + Swagger

In notification-service, generate application.yml that:
- Sets spring.application.name to "notification-service"
- Pulls config from Config Server, registers with Eureka
- Runs on port 8087
- Configures PostgreSQL datasource for notification_service_db
- Sets spring.jpa.hibernate.ddl-auto to "validate"

Add springdoc-openapi-starter-webmvc-ui to pom.xml.

Prompt A.7 — Unit tests

In notification-service, generate JUnit 5 + Mockito tests for
NotificationService:
- send() saves a notification with status transitioning from PENDING to
  SENT, and a non-null sentAt
- getNotificationsForUser() returns results ordered most-recent-first

Mock NotificationRepository.


PART B — Delivery Service

Reference Database-Schema-Design.md Section 8 (delivery_partners, deliveries). Depends on Order Service (Prompt 0.1) and Notification Service (Part A) — build this after both are working.


Prompt B.1 — Entities

In the delivery-service module, generate JPA entities for exactly these
two tables. Do not add any fields beyond what's listed.

Table: delivery_partners
- id: BIGSERIAL PK
- user_id: BIGINT NOT NULL UNIQUE (logical reference to Auth Service
  users.id)
- full_name: VARCHAR(100) NOT NULL
- phone: VARCHAR(15) NOT NULL
- vehicle_type: VARCHAR(30), nullable (e.g. BIKE, SCOOTER, BICYCLE)
- vehicle_number: VARCHAR(20), nullable
- approval_status: VARCHAR(20) NOT NULL DEFAULT 'PENDING'
  (values: PENDING, APPROVED, REJECTED)
- availability_status: VARCHAR(20) NOT NULL DEFAULT 'OFFLINE'
  (values: AVAILABLE, BUSY, OFFLINE)
- current_latitude: NUMERIC(9,6), nullable
- current_longitude: NUMERIC(9,6), nullable
- created_at: TIMESTAMP NOT NULL, auto-set on insert
- updated_at: TIMESTAMP NOT NULL, auto-updated on every update

Table: deliveries
- id: BIGSERIAL PK
- order_id: BIGINT NOT NULL UNIQUE (logical reference to Order Service
  orders.id — one delivery per order)
- delivery_partner_id: BIGINT NOT NULL, FK to delivery_partners.id
- delivery_status: VARCHAR(20) NOT NULL DEFAULT 'ASSIGNED'
  (values: ASSIGNED, PICKED_UP, DELIVERED, CANCELLED)
- assigned_at: TIMESTAMP NOT NULL, auto-set on insert
- picked_up_at: TIMESTAMP, nullable
- delivered_at: TIMESTAMP, nullable

Use package com.fooddelivery.deliveryservice.entity.
Use @ManyToOne (deliveries -> delivery_partners).
Use @CreationTimestamp / @UpdateTimestamp where applicable.

Prompt B.2 — Repositories and DTOs

In delivery-service, generate:
1. DeliveryPartnerRepository with findByUserId(Long userId) and
   findFirstByAvailabilityStatusAndApprovalStatus(String availabilityStatus,
   String approvalStatus)
2. DeliveryRepository with findByOrderId(Long orderId) and
   findByDeliveryPartnerId(Long partnerId)
3. DTOs in com.fooddelivery.deliveryservice.dto:
   - RegisterDeliveryPartnerRequest (userId, fullName, phone, vehicleType,
     vehicleNumber — Bean Validation on fullName/phone not blank)
   - DeliveryPartnerResponse (all columns)
   - UpdateAvailabilityRequest (availabilityStatus — must be AVAILABLE,
     BUSY, or OFFLINE)
   - UpdateLocationRequest (latitude, longitude)
   - UpdateApprovalStatusRequest (approvalStatus — must be APPROVED or
     REJECTED)
   - CreateDeliveryRequest (orderId)
   - DeliveryResponse (id, orderId, deliveryPartnerId, deliveryStatus,
     assignedAt, pickedUpAt, deliveredAt)
   - UpdateDeliveryStatusRequest (deliveryStatus)
4. A plain DeliveryMapper class (manual mapping, no MapStruct).

Prompt B.3 — Feign clients

In delivery-service, generate three OpenFeign clients following the
convention from previous services (package
com.fooddelivery.deliveryservice.client, one subfolder per client):

1. AuthServiceClient (@FeignClient(name = "auth-service",
   fallback = AuthServiceFallback.class)):
   GET /internal/auth/users/{id} -> UserSummaryResponse (id, email, role)

2. OrderServiceClient (@FeignClient(name = "order-service",
   fallback = OrderServiceFallback.class)):
   GET /internal/orders/{id} -> OrderSummaryResponse (id, customerId,
   restaurantId, orderStatus)
   PUT /internal/orders/{id}/status -> OrderSummaryResponse (body:
   { "orderStatus": "..." })

3. NotificationServiceClient (@FeignClient(name = "notification-service",
   fallback = NotificationServiceFallback.class)):
   POST /internal/notifications -> NotificationSummaryResponse (body:
   { "userId": ..., "notificationType": "DELIVERY_STATUS", "channel":
   "PUSH", "title": "...", "message": "..." })

Each fallback throws ServiceUnavailableException naming the unreachable
service, EXCEPT NotificationServiceFallback, which instead logs a warning
and returns null — a failed notification should never block a delivery
status update. Add spring-cloud-starter-openfeign to pom.xml and
@EnableFeignClients to the main application class, plus the shared Feign
timeout block in application.yml.

Prompt B.4 — DeliveryPartnerService

In delivery-service, generate a DeliveryPartnerService
(com.fooddelivery.deliveryservice.service) with:

- registerPartner(RegisterDeliveryPartnerRequest request): calls
  AuthServiceClient.getUser(request.getUserId()) to confirm the user
  exists with role DELIVERY_PARTNER (throw InvalidUserRoleException if
  not); throws DuplicateResourceException if a partner profile already
  exists for that user_id; saves with approval_status = PENDING,
  availability_status = OFFLINE

- updateApprovalStatus(Long partnerId, UpdateApprovalStatusRequest request):
  admin-only

- updateAvailability(Long partnerId, UpdateAvailabilityRequest request):
  throws InvalidUserRoleException-style validation isn't needed here,
  just updates the status; reject the change with a custom
  PartnerNotApprovedException if approval_status isn't APPROVED

- updateLocation(Long partnerId, UpdateLocationRequest request)

- findAvailablePartner(): returns the first partner with
  availability_status = AVAILABLE and approval_status = APPROVED, or
  throws NoPartnerAvailableException if none exist. Note in a comment
  that this is a simple "first available" strategy — no geo-distance
  matching in this project's scope.

- markPartnerBusy(Long partnerId) / markPartnerAvailable(Long partnerId):
  small helper methods used by DeliveryService during assignment and
  completion

Prompt B.5 — DeliveryService

In delivery-service, generate a DeliveryService
(com.fooddelivery.deliveryservice.service) with:

- createDelivery(CreateDeliveryRequest request):
  1. throws DuplicateResourceException if a delivery already exists for
     that order_id
  2. calls OrderServiceClient.getOrder(request.getOrderId()); throws
     ResourceNotFoundException if missing, throws
     InvalidOrderStateException if orderStatus is DELIVERED or CANCELLED
  3. calls DeliveryPartnerService.findAvailablePartner()
  4. creates the Delivery row (delivery_status = ASSIGNED, assigned_at =
     now), and calls DeliveryPartnerService.markPartnerBusy(partnerId)
  5. attempts NotificationServiceClient.send(...) with
     notificationType = DELIVERY_STATUS to the order's customer_id —
     wrap this call in a try/catch that only logs a warning on failure,
     never throws
  6. returns the DeliveryResponse

- updateDeliveryStatus(Long deliveryId, UpdateDeliveryStatusRequest request):
  1. validates the transition only moves forward:
     ASSIGNED -> PICKED_UP -> DELIVERED, with CANCELLED allowed from
     ASSIGNED or PICKED_UP only (not from DELIVERED); throw
     InvalidDeliveryStatusTransitionException otherwise
  2. sets picked_up_at when moving to PICKED_UP, delivered_at when
     moving to DELIVERED
  3. calls OrderServiceClient.updateOrderStatus(orderId, ...) mapping
     PICKED_UP -> PICKED_UP, DELIVERED -> DELIVERED, CANCELLED ->
     CANCELLED, so Order Service's order_status stays in sync
  4. if the new status is DELIVERED or CANCELLED, calls
     DeliveryPartnerService.markPartnerAvailable(partnerId)
  5. attempts a best-effort NotificationServiceClient call (same
     try/catch pattern as createDelivery)
  6. returns the updated DeliveryResponse

- getDeliveryById(Long id): throws ResourceNotFoundException if missing

- getDeliveryByOrderId(Long orderId): for internal Feign consumption by
  Review Service on Day 6 (to look up which partner delivered an order)

Prompt B.6 — Controllers

In delivery-service, generate three controllers:

1. DeliveryPartnerController (public API), package
   com.fooddelivery.deliveryservice.controller:
   POST /api/delivery-partners                -> registerPartner
   PUT  /api/delivery-partners/me/availability -> updateAvailability
   PUT  /api/delivery-partners/me/location     -> updateLocation
   PUT  /api/delivery-partners/{id}/approval   -> updateApprovalStatus (admin)

2. DeliveryController (public API):
   POST /api/deliveries         -> createDelivery
   GET  /api/deliveries/{id}    -> getDeliveryById
   PUT  /api/deliveries/{id}/status -> updateDeliveryStatus (partner-only)

3. InternalDeliveryController:
   GET /internal/deliveries/by-order/{orderId} -> getDeliveryByOrderId,
   for Review Service's Feign client on Day 6

Use @Valid on request bodies, ResponseEntity<T>.

Prompt B.7 — Global exception handling

In delivery-service, generate a GlobalExceptionHandler
(@RestControllerAdvice) in com.fooddelivery.deliveryservice.exception,
using the same response shape as auth-service's handler (paste it as
reference), handling:
- ResourceNotFoundException -> 404
- DuplicateResourceException -> 409
- InvalidUserRoleException -> 400
- InvalidOrderStateException -> 400
- NoPartnerAvailableException -> 409
- PartnerNotApprovedException -> 400
- InvalidDeliveryStatusTransitionException -> 409
- ServiceUnavailableException -> 503
- MethodArgumentNotValidException -> 400 with field error map
- Exception (catch-all) -> 500

Prompt B.8 — application.yml + Swagger

In delivery-service, generate application.yml that:
- Sets spring.application.name to "delivery-service"
- Pulls config from Config Server, registers with Eureka
- Runs on port 8086
- Configures PostgreSQL datasource for delivery_service_db
- Sets spring.jpa.hibernate.ddl-auto to "validate"
- Includes the shared Feign timeout block

Add springdoc-openapi-starter-webmvc-ui to pom.xml.

Prompt B.9 — Unit tests

In delivery-service, generate JUnit 5 + Mockito tests:

DeliveryPartnerService:
- registerPartner() throws InvalidUserRoleException for the wrong role
- findAvailablePartner() throws NoPartnerAvailableException when none
  are AVAILABLE and APPROVED

DeliveryService:
- createDelivery() throws DuplicateResourceException when a delivery
  already exists for the order
- createDelivery() throws InvalidOrderStateException when the order is
  already DELIVERED or CANCELLED
- createDelivery() marks the assigned partner as BUSY
- createDelivery() still succeeds and returns a result even when
  NotificationServiceClient throws (best-effort notification)
- updateDeliveryStatus() throws InvalidDeliveryStatusTransitionException
  for a backwards move
- updateDeliveryStatus() calls OrderServiceClient.updateOrderStatus()
  with the correctly mapped status
- updateDeliveryStatus() marks the partner AVAILABLE again once
  DELIVERED

Mock all repositories and all three Feign clients.



Review & Rating Service → Admin Service

Scope: Review & Rating Service (build first — simpler, depends only on Order Service) → Admin Service (last service — mostly orchestrates writes into Auth, Restaurant, and Delivery Services rather than owning much data itself).
Companion docs: Database-Schema-Design.md Sections 10–11, Day5-Copilot-Prompt-Sequence.md


This is the last service-build day. Day 7 is integration testing and polish, not new services.



0. Prerequisites — small additions to three existing services

Admin Service doesn't own restaurant approval, delivery-partner approval, or user status — those fields live in Restaurant, Delivery, and Auth Service respectively. None of the previous days exposed an internal (service-to-service) way to change them — only public, gateway-routed endpoints. Add these three small internal endpoints before starting Admin Service. Review & Rating Service's own prerequisite (updating restaurants.avg_rating) is listed separately in Part A.


Prompt 0.1 — Auth Service: internal user status update

In the existing auth-service module, add one more endpoint to
InternalUserController (or create it if the internal controller doesn't
exist yet — check auth-service's existing code first):

PUT /internal/auth/users/{id}/status
-> body: { "isActive": boolean }
-> updates the user's is_active column and returns UserSummary

Add the corresponding method to AuthService: updateUserStatus(Long userId,
boolean isActive). Do not change any other existing files in auth-service.

Prompt 0.2 — Restaurant Service: internal approval status update

In the existing restaurant-service module, add one more endpoint to
InternalRestaurantController:

PUT /internal/restaurants/{id}/status
-> body: { "approvalStatus": "APPROVED" | "REJECTED" }
-> reuses the existing RestaurantService.updateApprovalStatus() logic
   and returns the updated RestaurantResponse

This is for Admin Service to call via Feign — the existing public
PUT /api/restaurants/{id}/status endpoint stays as-is for direct gateway
access. Do not change any other existing files in restaurant-service.

Prompt 0.3 — Delivery Service: internal partner approval update

In the existing delivery-service module, add one more endpoint to
InternalDeliveryController:

PUT /internal/delivery-partners/{id}/approval
-> body: { "approvalStatus": "APPROVED" | "REJECTED" }
-> reuses the existing DeliveryPartnerService.updateApprovalStatus()
   logic and returns the updated DeliveryPartnerResponse

This is for Admin Service to call via Feign — the existing public
PUT /api/delivery-partners/{id}/approval endpoint stays as-is. Do not
change any other existing files in delivery-service.


PART A — Review & Rating Service

Reference Database-Schema-Design.md Section 10 (reviews — single table, restaurant reviews only, tied to a completed order).


Prompt A.1 — Entity

In the review-rating-service module (Java 21, Spring Boot 3.2.x, Spring
Data JPA, Lombok), generate a JPA entity for exactly this table. Do not
add any fields beyond what's listed.

Table: reviews
- id: BIGSERIAL PK
- customer_id: BIGINT NOT NULL (logical reference to Customer Service
  customers.id)
- restaurant_id: BIGINT NOT NULL (logical reference to Restaurant
  Service restaurants.id)
- order_id: BIGINT NOT NULL UNIQUE (logical reference to Order Service
  orders.id — one review per order)
- rating: SMALLINT NOT NULL, must be between 1 and 5
- comment: TEXT, nullable
- created_at: TIMESTAMP NOT NULL, auto-set on insert
- updated_at: TIMESTAMP NOT NULL, auto-updated on every update

Use package com.fooddelivery.reviewservice.entity.
Use @CreationTimestamp / @UpdateTimestamp for the timestamp columns.

Prompt A.2 — Repository and DTOs

In review-rating-service, generate:
1. ReviewRepository (Spring Data JPA) with:
   - findByRestaurantId(Long restaurantId)
   - findByCustomerId(Long customerId)
   - existsByOrderId(Long orderId)
   - a custom @Query method averageRatingForRestaurant(Long restaurantId)
     returning a Double (nullable), computing AVG(rating) WHERE
     restaurant_id = :restaurantId
2. DTOs in com.fooddelivery.reviewservice.dto:
   - CreateReviewRequest (orderId, rating, comment — Bean Validation:
     rating must be between 1 and 5)
   - ReviewResponse (id, customerId, restaurantId, orderId, rating,
     comment, createdAt)

Prompt A.3 — Feign clients

In review-rating-service, generate two OpenFeign clients following the
convention from previous services (package
com.fooddelivery.reviewservice.client, one subfolder per client):

1. OrderServiceClient (@FeignClient(name = "order-service",
   fallback = OrderServiceFallback.class)):
   GET /internal/orders/{id} -> OrderSummaryResponse (id, customerId,
   restaurantId, orderStatus)
   OrderServiceFallback throws ServiceUnavailableException — a review
   cannot be validated without confirming the order, so this one must
   fail loudly.

2. RestaurantServiceClient (@FeignClient(name = "restaurant-service",
   fallback = RestaurantServiceFallback.class)):
   PUT /internal/restaurants/{id}/rating -> body:
   { "avgRating": <BigDecimal> }
   RestaurantServiceFallback logs a warning and returns null instead of
   throwing — updating the restaurant's denormalized average rating is
   best-effort and should never block a review from being saved.

Add spring-cloud-starter-openfeign to pom.xml and @EnableFeignClients to
the main application class, plus the shared Feign timeout block in
application.yml.

Also add this prerequisite to restaurant-service (do this alongside the
clients, not skip it): add PUT /internal/restaurants/{id}/rating to
InternalRestaurantController, accepting { "avgRating": BigDecimal } and
calling a new RestaurantService.updateAvgRating(Long id, BigDecimal
avgRating) method that updates only the avg_rating column.

Prompt A.4 — Service layer

In review-rating-service, generate a ReviewService
(com.fooddelivery.reviewservice.service) with a submitReview(Long
customerId, CreateReviewRequest request) method that:

1. Calls OrderServiceClient.getOrder(request.getOrderId()); throws
   ResourceNotFoundException if the order doesn't exist
2. Throws UnauthorizedReviewException if order.getCustomerId() doesn't
   match the customerId parameter (a customer can only review their own
   orders)
3. Throws OrderNotDeliveredException if order.getOrderStatus() isn't
   DELIVERED
4. Throws DuplicateResourceException if
   reviewRepository.existsByOrderId(orderId) is true
5. Saves the review, using restaurant_id from the order response
6. Computes the new average rating via
   reviewRepository.averageRatingForRestaurant(restaurantId), then
   attempts RestaurantServiceClient.updateRating(restaurantId, avg) in a
   try/catch that only logs a warning on failure — never let a failed
   average-rating push fail the review submission
7. Returns the ReviewResponse

Also generate:
- getReviewsForRestaurant(Long restaurantId)
- getReviewsForCustomer(Long customerId)

Prompt A.5 — Controller

In review-rating-service, generate a ReviewController (public API),
package com.fooddelivery.reviewservice.controller:

POST /api/reviews                     -> submitReview (customerId from
                                          an X-User-Id header, same
                                          convention as other services)
GET  /api/reviews/restaurant/{id}     -> getReviewsForRestaurant
GET  /api/reviews/customer/me         -> getReviewsForCustomer

Use @Valid on request bodies, ResponseEntity<T>.

Prompt A.6 — Global exception handling

In review-rating-service, generate a GlobalExceptionHandler
(@RestControllerAdvice) in com.fooddelivery.reviewservice.exception,
using the same response shape as auth-service's handler (paste it as
reference), handling:
- ResourceNotFoundException -> 404
- UnauthorizedReviewException -> 403
- OrderNotDeliveredException -> 400
- DuplicateResourceException -> 409
- ServiceUnavailableException -> 503
- MethodArgumentNotValidException -> 400 with field error map
- Exception (catch-all) -> 500

Prompt A.7 — application.yml + Swagger

In review-rating-service, generate application.yml that:
- Sets spring.application.name to "review-rating-service"
- Pulls config from Config Server, registers with Eureka
- Runs on port 8088
- Configures PostgreSQL datasource for review_service_db
- Sets spring.jpa.hibernate.ddl-auto to "validate"
- Includes the shared Feign timeout block

Add springdoc-openapi-starter-webmvc-ui to pom.xml.

Prompt A.8 — Unit tests

In review-rating-service, generate JUnit 5 + Mockito tests for
ReviewService:
- submitReview() throws OrderNotDeliveredException when the order isn't
  DELIVERED
- submitReview() throws UnauthorizedReviewException when the order's
  customer_id doesn't match the requesting customer
- submitReview() throws DuplicateResourceException when a review
  already exists for that order
- submitReview() still saves and returns the review even when
  RestaurantServiceClient.updateRating() throws (best-effort average
  push)

Mock ReviewRepository, OrderServiceClient, and RestaurantServiceClient.


PART B — Admin Service

Reference Database-Schema-Design.md Section 11 (admin_users, audit_logs, platform_settings). This service orchestrates writes into Auth, Restaurant, and Delivery Service via the internal endpoints added in Section 0 — it never owns restaurant/user/partner state itself.


Prompt B.1 — Entities

In the admin-service module, generate JPA entities for exactly these
three tables. Do not add any fields beyond what's listed.

Table: admin_users
- id: BIGSERIAL PK
- user_id: BIGINT NOT NULL UNIQUE (logical reference to Auth Service
  users.id)
- full_name: VARCHAR(100) NOT NULL
- email: VARCHAR(150) NOT NULL
- created_at: TIMESTAMP NOT NULL, auto-set on insert

Table: audit_logs
- id: BIGSERIAL PK
- admin_id: BIGINT NOT NULL, FK to admin_users.id
- action: VARCHAR(100) NOT NULL (free text, e.g. APPROVE_RESTAURANT,
  REJECT_DELIVERY_PARTNER, DISABLE_USER, UPDATE_SETTING)
- entity_type: VARCHAR(50) NOT NULL (free text, e.g. RESTAURANT,
  DELIVERY_PARTNER, USER, SETTING)
- entity_id: BIGINT NOT NULL (logical reference to the affected entity,
  meaning depends on entity_type)
- description: TEXT, nullable
- created_at: TIMESTAMP NOT NULL, auto-set on insert

Table: platform_settings
- id: SERIAL PK
- setting_key: VARCHAR(100) NOT NULL UNIQUE
- setting_value: VARCHAR(500) NOT NULL
- updated_at: TIMESTAMP NOT NULL, auto-updated on every update

Use package com.fooddelivery.adminservice.entity.
Use @ManyToOne (audit_logs -> admin_users).
Use @CreationTimestamp / @UpdateTimestamp where applicable. audit_logs
has no updated_at — entries are immutable once created.

Prompt B.2 — Repositories and DTOs

In admin-service, generate:
1. AdminUserRepository with findByUserId(Long userId)
2. AuditLogRepository with findByAdminId(Long adminId) and
   findByEntityTypeAndEntityId(String entityType, Long entityId)
3. PlatformSettingRepository with findBySettingKey(String settingKey)
4. DTOs in com.fooddelivery.adminservice.dto:
   - RegisterAdminRequest (userId, fullName, email — Bean Validation on
     fullName/email)
   - AdminUserResponse (id, userId, fullName, email, createdAt)
   - UpdateApprovalRequest (approvalStatus — must be APPROVED or REJECTED)
   - AuditLogResponse (id, adminId, action, entityType, entityId,
     description, createdAt)
   - PlatformSettingRequest (settingValue)
   - PlatformSettingResponse (settingKey, settingValue, updatedAt)

Prompt B.3 — Feign clients

In admin-service, generate three OpenFeign clients following the
convention from previous services (package
com.fooddelivery.adminservice.client, one subfolder per client), each
with a fallback that throws ServiceUnavailableException (none of these
are best-effort — an admin action that silently fails to apply would be
worse than a visible error):

1. AuthServiceClient (@FeignClient(name = "auth-service")):
   GET /internal/auth/users/{id} -> UserSummaryResponse (id, email, role)
   PUT /internal/auth/users/{id}/status -> body: { "isActive": boolean }

2. RestaurantServiceClient (@FeignClient(name = "restaurant-service")):
   PUT /internal/restaurants/{id}/status -> body:
   { "approvalStatus": "..." }

3. DeliveryServiceClient (@FeignClient(name = "delivery-service")):
   PUT /internal/delivery-partners/{id}/approval -> body:
   { "approvalStatus": "..." }

Add spring-cloud-starter-openfeign to pom.xml and @EnableFeignClients to
the main application class, plus the shared Feign timeout block in
application.yml.

Prompt B.4 — Service layer

In admin-service, generate an AdminService
(com.fooddelivery.adminservice.service) with:

- registerAdmin(RegisterAdminRequest request): calls
  AuthServiceClient.getUser(request.getUserId()) to confirm the user
  exists with role ADMIN (throw InvalidUserRoleException if not); throws
  DuplicateResourceException if an admin_users row already exists for
  that user_id; saves and returns AdminUserResponse

- approveRestaurant(Long adminId, Long restaurantId, UpdateApprovalRequest
  request): calls RestaurantServiceClient.updateStatus(restaurantId,
  request.getApprovalStatus()), then writes an audit_logs row (action =
  "APPROVE_RESTAURANT" or "REJECT_RESTAURANT" depending on the value,
  entity_type = "RESTAURANT", entity_id = restaurantId)

- approveDeliveryPartner(Long adminId, Long partnerId,
  UpdateApprovalRequest request): same pattern, calling
  DeliveryServiceClient, entity_type = "DELIVERY_PARTNER"

- disableUser(Long adminId, Long userId): calls
  AuthServiceClient.updateUserStatus(userId, false), writes an
  audit_logs row (action = "DISABLE_USER", entity_type = "USER",
  entity_id = userId)

- enableUser(Long adminId, Long userId): same pattern with true /
  "ENABLE_USER"

- getAuditLogs(String entityType, Long entityId): returns all logs, or
  filtered by entity_type + entity_id if both are provided

- getSetting(String key) / updateSetting(Long adminId, String key,
  PlatformSettingRequest request): reads/creates-or-updates a
  platform_settings row; updateSetting also writes an audit_logs row
  (action = "UPDATE_SETTING", entity_type = "SETTING")

Extract the "write an audit_logs row" logic into one small private
helper method (e.g. writeAuditLog(adminId, action, entityType, entityId,
description)) so it isn't duplicated five times.

Prompt B.5 — Controller

In admin-service, generate an AdminController (public API), package
com.fooddelivery.adminservice.controller:

POST /api/admin/register                          -> registerAdmin
PUT  /api/admin/restaurants/{id}/approval          -> approveRestaurant
PUT  /api/admin/delivery-partners/{id}/approval    -> approveDeliveryPartner
PUT  /api/admin/users/{id}/disable                 -> disableUser
PUT  /api/admin/users/{id}/enable                  -> enableUser
GET  /api/admin/audit-logs                         -> getAuditLogs
                                                       (optional
                                                       entityType/
                                                       entityId query
                                                       params)
GET  /api/admin/settings/{key}                     -> getSetting
PUT  /api/admin/settings/{key}                      -> updateSetting

adminId comes from an X-User-Id header representing the admin_users.id,
same convention used for customerId/restaurantId in other services'
controllers. Use @Valid on request bodies, ResponseEntity<T>.

Prompt B.6 — Global exception handling

In admin-service, generate a GlobalExceptionHandler
(@RestControllerAdvice) in com.fooddelivery.adminservice.exception,
using the same response shape as auth-service's handler (paste it as
reference), handling:
- ResourceNotFoundException -> 404
- DuplicateResourceException -> 409
- InvalidUserRoleException -> 400
- ServiceUnavailableException -> 503
- MethodArgumentNotValidException -> 400 with field error map
- Exception (catch-all) -> 500

Prompt B.7 — application.yml + Swagger

In admin-service, generate application.yml that:
- Sets spring.application.name to "admin-service"
- Pulls config from Config Server, registers with Eureka
- Runs on port 8089
- Configures PostgreSQL datasource for admin_service_db
- Sets spring.jpa.hibernate.ddl-auto to "validate"
- Includes the shared Feign timeout block

Add springdoc-openapi-starter-webmvc-ui to pom.xml.

Prompt B.8 — Unit tests

In admin-service, generate JUnit 5 + Mockito tests for AdminService:
- registerAdmin() throws InvalidUserRoleException for a non-ADMIN
  user_id
- approveRestaurant() calls RestaurantServiceClient.updateStatus() with
  the right arguments and writes an audit_logs row with action
  "APPROVE_RESTAURANT"
- disableUser() calls AuthServiceClient.updateUserStatus(userId, false)
  and writes an audit_logs row with action "DISABLE_USER"
- getAuditLogs() filters correctly when entityType/entityId are provided

Mock all repositories and all three Feign clients.

