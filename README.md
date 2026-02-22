# Multi-Tenancy Architecture

## Table of Contents
1. [System Overview](#1-system-overview)
2. [Personas & Roles](#2-personas--roles)
3. [Token Architecture (Existing)](#3-token-architecture-existing)
4. [Tenant & License Model](#4-tenant--license-model)
5. [Subscription Layer](#5-subscription-layer)
6. [Updated Gateway Flow](#6-updated-gateway-flow)
7. [Multi-Tenancy & Data Isolation](#7-multi-tenancy--data-isolation)
8. [User Cap Enforcement](#8-user-cap-enforcement)
9. [Frontend Subscription Access](#9-frontend-subscription-access)
10. [End-to-End Request Flow](#10-end-to-end-request-flow)
11. [What Changes, Where](#11-what-changes-where)

---

## 1. System Overview

### Microservices

| Service | Default Access | Requires Subscription |
|---|---|---|
| api-gateway | ✅ Always | — |
| auth-service | ✅ Always | — |
| admin-service | ✅ Always | — |
| banking-service | ❌ | ✅ |
| customer-service | ❌ | ✅ |
| loan-service | ❌ | ✅ |
| deposit-service | ❌ | ✅ |
| placement-service | ❌ | ✅ |
| **subscription-service** | Internal only | — |

### Subscription Rules

| State | Access |
|---|---|
| Trial (within date range) | All MSs — dates set manually per tenant by admin |
| Trial expired, no subscription | No access to paid MSs |
| Active subscription | Only subscribed MSs, for subscribed period |
| Subscription expired | No access until renewed |

**Plans:** 3 Month, 1 Year, Lifetime

---

## 2. Personas & Roles

```
┌─────────────────────────────────────────────────┐
│                  PRODUCT ADMIN SIDE             │
│                                                 │
│  SUPER_ADMIN      — founder/owner level         │
│  PRODUCT_ADMIN    — ops team, creates tenants   │
│                     and issues licenses         │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│                   BANK SIDE                     │
│                                                 │
│  BANK_ADMIN       — manages their own users,    │
│                     views their license         │
│  BANK_USER        — uses subscribed MSs only    │
└─────────────────────────────────────────────────┘
```

### Key Distinction in JWT Claims

| Role | tenantId in JWS | Scope |
|---|---|---|
| SUPER_ADMIN | ❌ None | All tenants |
| PRODUCT_ADMIN | ❌ None | All tenants |
| BANK_ADMIN | ✅ Yes | Own tenant only |
| BANK_USER | ✅ Yes | Own tenant only |

---

## 3. Token Architecture (Existing)

The two-layer token system remains **completely unchanged**.

```
Client → Gateway              :  JWE  (encrypted, RSA)
Gateway → Microservice        :  JWS  (signed, HMAC)
```

```
┌──────────┐        JWE         ┌─────────────┐       JWS        ┌──────────────┐
│  Client  │ ────────────────▶ │ API Gateway │ ───────────────▶  │ Microservice │
│          │                   │             │                   │              │
│          │                   │ Decrypt JWE │                   │ Validate JWS │
│          │                   │ → get JWS   │                   │ → get claims │
└──────────┘                   └─────────────┘                   └──────────────┘
```

### JWS Claims (Updated — add `tid`)

```json
{
  "e":   "user@hdfc.com",
  "r":   "BANK_USER",
  "p":   ["READ_ACCOUNTS"],
  "v":   3,
  "tid": "tenant-abc123"
}
```

`tid` is the `tenantId`. SUPER_ADMIN and _ADMIN tokens omit `tid`.

---

## 4. Tenant & License Model

admin issues a license to a bank through a single form. No API key needed — the **email domain is the license identifier**.

### License Issuance Form

```
Bank Name        : HDFC Bank
Admin Email      : admin@hdfc.com       ← first user, gets BANK_ADMIN role
Email Domain     : hdfc.com             ← all @hdfc.com emails belong to this tenant
License Type     : TRIAL / SUBSCRIPTION
Trial Start/End  : 2025-02-01 → 2025-03-01   (set manually, per tenant)
Subscribed MSs   : banking-service, loan-service, deposit-service
Plan             : 3_MONTH / 1_YEAR / LIFETIME
Start / End      : 2025-03-01 → 2026-03-01
Max Users        : 50                   ← non-deleted user cap
```

### Database Schema (subscription-service — isolated DB)

```sql
tenant (
  id               UUID PRIMARY KEY,
  name             VARCHAR,              -- "HDFC Bank"
  email_domain     VARCHAR UNIQUE,       -- "hdfc.com"
  admin_email      VARCHAR,
  status           ENUM('ACTIVE', 'INACTIVE', 'SUSPENDED'),
  created_at       TIMESTAMP,
  created_by       UUID                  --  admin who created this
)

license (
  id               UUID PRIMARY KEY,
  tenant_id        UUID REFERENCES tenant(id),
  type             ENUM('TRIAL', 'SUBSCRIPTION'),
  status           ENUM('ACTIVE', 'EXPIRED', 'CANCELLED'),
  max_users        INT,
  plan             ENUM('TRIAL', '3_MONTH', '1_YEAR', 'LIFETIME'),
  starts_at        TIMESTAMP,
  ends_at          TIMESTAMP,            -- NULL for LIFETIME
  created_at       TIMESTAMP,
  created_by       UUID
)

license_services (
  id               UUID PRIMARY KEY,
  license_id       UUID REFERENCES license(id),
  service_name     VARCHAR,              -- "banking-service", "loan-service"
  created_at       TIMESTAMP
)
```

### License Lifecycle

```
Admin creates tenant
         │
         ▼
License record created  (TRIAL or SUBSCRIPTION)
         │
         ▼
Bank Admin logs in → receives JWE with tenantId claim
         │
         ├── Trial expires, no subscription → DENY paid MSs
         │
         └── Subscription purchased → new license record created
                   (old trial record is EXPIRED, not mutated)
```

Each renewal or upgrade creates a **new license record**. The old one is marked `EXPIRED`. Full audit history preserved.

---

## 5. Subscription Layer

### subscription-service Endpoints

```
# Called by API Gateway (internal, not exposed to clients)
GET  /internal/access?tenantId=X&service=banking-service

# Called by frontend after login
GET  /subscriptions/me

# Called by admin-service /  admin operations
POST /internal/subscriptions/trial
POST /subscriptions
PUT  /subscriptions/{id}/renew
PUT  /subscriptions/{id}/cancel
```

### hasAccess Logic

```
hasAccess(tenantId, serviceName):

  1. Is trial active?
       NOW() is between trial_start and trial_end
       → ALLOW

  2. Trial expired AND no active subscription?
       → DENY  (reason: TRIAL_EXPIRED)

  3. Active subscription exists for this service?
       license.status = ACTIVE
       AND NOW() < license.ends_at (or ends_at IS NULL for LIFETIME)
       AND service in license_services
       → ALLOW

  4. Subscription exists but expired?
       → DENY  (reason: SUBSCRIPTION_EXPIRED)

  5. No subscription for this service?
       → DENY  (reason: NOT_SUBSCRIBED)
```

### Redis Cache

```
Key:   sub:{tenantId}:{serviceName}
Value: ALLOW | DENY
TTL:   5 minutes
```

On any subscription change (purchase, cancellation, expiry), subscription-service **immediately invalidates** the relevant Redis keys. Upgrades take effect instantly. Downgrades take effect within seconds.

---

## 6. Updated Gateway Flow

```
Request arrives at API Gateway
        │
        ▼
[Existing] Decrypt JWE → validate JWS
        │
        ▼
[Existing] AuthenticationFilterChain
        │
        ├── SystemLockFilter
        ├── UserLockFilter
        └── TokenVersionFilter
                │
                ▼
        SubscriptionFilter  ◄── NEW
                │
                ▼
        Is target service free?
        (api-gateway, auth-service, admin-service)
           │                │
          YES               NO
           │                │
        Pass            Extract tenantId from JWS claims
        through              │
                             ▼
                    Redis: sub:{tenantId}:{service}
                       │           │           │
                     HIT          HIT         MISS
                    ALLOW         DENY          │
                       │           │            ▼
                    Pass        Return    Call subscription-service
                   through      403 +         │
                               reason    Cache result (5 min)
                                              │
                                         ALLOW / DENY
        │
        ▼
Mutate: Authorization = Bearer JWS
        │
        ▼
Route → StripPrefix → RateLimit → Forward to MS
```

### 403 Response Payload

```json
{
  "status": 403,
  "error": "SUBSCRIPTION_REQUIRED",
  "reason": "NOT_SUBSCRIBED",
  "service": "banking-service",
  "upgradeUrl": "/subscriptions"
}
```

Possible `reason` values: `TRIAL_EXPIRED`, `SUBSCRIPTION_EXPIRED`, `NOT_SUBSCRIBED`

---

## 7. Multi-Tenancy & Data Isolation

All microservices share a single database. Every table must be scoped to a tenant. The isolation is enforced automatically via a **Hibernate Filter** — developers do not need to manually add `WHERE tenant_id = ?` to every query.

### Step 1 — BaseEntity gets tenantId

```java
@MappedSuperclass
@FilterDef(
  name = "tenantFilter",
  parameters = @ParamDef(name = "tenantId", type = String.class)
)
@Filter(name = "tenantFilter", condition = "tenant_id = :tenantId")
@Data
public abstract class BaseEntity {

    @Column(name = "tenant_id", nullable = false, updatable = false)
    private String tenantId;

    // existing base fields: createdAt, updatedBy, etc.
}
```

All entities that extend `BaseEntity` inherit `tenantId` automatically.

### Step 2 — JwtFilter sets tenantId as request attribute

In each microservice's existing `JwtFilter`:

```java
// After extracting claims (already done for email, role, privileges)
String tenantId = claims.get("tid", String.class);
request.setAttribute("tenantId", tenantId);
request.setAttribute("role", role);
```

### Step 3 — Interceptor enables Hibernate filter per request

```java
@Component
public class TenantFilterInterceptor implements HandlerInterceptor {

    @PersistenceContext
    private EntityManager entityManager;

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) {

        String role = (String) request.getAttribute("role");

        //staff see all tenants — no filter
        if ("_ADMIN".equals(role) || "SUPER_ADMIN".equals(role)) {
            return true;
        }

        String tenantId = (String) request.getAttribute("tenantId");
        Session session = entityManager.unwrap(Session.class);
        session.enableFilter("tenantFilter")
               .setParameter("tenantId", tenantId);

        return true;
    }
}
```

### What This Means for Developers

A developer writes a normal repository query:

```java
depositAccountRepository.findByCustomerId(customerId);
```

Hibernate automatically executes:

```sql
SELECT * FROM deposit_account
WHERE customer_id = ?
AND tenant_id = 'tenant-abc'   -- injected silently by the filter
```

No manual scoping. No risk of accidental data leak between tenants.

### Is This Stateless?

**Yes.** The Hibernate Session here is not an HTTP session. It is a per-request unit of work with the DB. It opens when the request arrives and closes when it ends. No state is shared between requests. The app remains fully stateless — JWE/JWS tokens are still the only auth mechanism.

### Database Migration

For existing tables, run a Flyway/Liquibase migration:

```sql
-- 1. Add column as nullable
ALTER TABLE deposit_account ADD COLUMN tenant_id VARCHAR(36);
ALTER TABLE user_master      ADD COLUMN tenant_id VARCHAR(36);
-- repeat for every table

-- 2. Backfill existing rows
UPDATE deposit_account SET tenant_id = 'tenant-default' WHERE tenant_id IS NULL;
UPDATE user_master      SET tenant_id = 'tenant-default' WHERE tenant_id IS NULL;

-- 3. Make NOT NULL
ALTER TABLE deposit_account ALTER COLUMN tenant_id SET NOT NULL;
ALTER TABLE user_master      ALTER COLUMN tenant_id SET NOT NULL;
```

---

## 8. User Cap Enforcement

`max_users` on the license means **non-deleted (non-inactive) users**, not concurrent sessions.

### Login Check

At login, auth-service checks:

```
1. Extract domain from email → look up tenant by domain
2. Tenant found? → proceed. Not found? → reject (unregistered domain)
3. Tenant active? → proceed. Suspended/inactive? → reject
4. Trial or subscription valid? → proceed. Expired? → reject
5. Credentials valid? → issue JWE
```

### Adding a New User (Race Condition Safe)

```
POST /users  (called by BANK_ADMIN)

1. Extract tenantId from JWS claims
2. GET /internal/license/limit?tenantId=X
   subscription-service returns { maxUsers: 50 }
   (just the cap — count comes from source of truth: main DB)

3. BEGIN TRANSACTION
   SELECT COUNT(*) FROM user_master
   WHERE tenant_id = ?
   AND status != 'INACTIVE'
   FOR UPDATE;                ← pessimistic lock prevents race condition

4. count >= maxUsers?
   → ROLLBACK → 403 LICENSE_USER_LIMIT_REACHED

5. count < maxUsers?
   → INSERT user → COMMIT → 201 Created
```

The `FOR UPDATE` lock ensures two simultaneous requests cannot both read 49, both pass the check, and both insert — pushing the count to 51.

### DB Separation

```
subscription-service DB  (isolated)
├── tenant
├── license
└── license_services

main shared DB  (existing, all other MSs)
└── user_master  ← source of truth for user count
```

Subscription-service provides `maxUsers`. Admin-service counts from its own DB. No sync, no eventual consistency drift.

---

## 9. Frontend Subscription Access

The JWE is encrypted and opaque to the frontend. Subscription data is **never embedded in the token**. After login, the frontend makes one additional call:

```
GET /subscriptions/me
Authorization: Bearer <JWE>

Response:
{
  "tenantId":        "tenant-abc",
  "plan":            "3_MONTH",
  "status":          "ACTIVE",
  "trialEndsAt":     null,
  "allowedServices": ["banking-service", "loan-service", "deposit-service"],
  "expiresAt":       "2026-03-01"
}
```

Frontend stores this in app state (Redux, Zustand, Pinia, etc.) and uses it for show/hide decisions only. The gateway enforces independently — the frontend's knowledge of allowed services is for UX, not security.

---

## 10. End-to-End Request Flow

```
┌──────────┐
│  Client  │  GET /banking-service/accounts
│          │  Authorization: Bearer JWE
└────┬─────┘
     │
     ▼
┌─────────────────────────────────────────────────────────┐
│                      API Gateway                        │
│                                                         │
│  1. AuthorizationHeaderFilter — forward header          │
│  2. JwtAuthenticationFilter                             │
│     a. Decrypt JWE → JWS  (JweService, RSA private key) │
│     b. Validate JWS  (JwtValidator, HMAC)               │
│  3. AuthenticationFilterChain                           │
│     a. SystemLockFilter   → Redis system_lock           │
│     b. UserLockFilter     → Redis user_lock:email       │
│     c. TokenVersionFilter → Redis token version         │
│     d. SubscriptionFilter → Redis sub:tenantId:service  │
│        └── MISS → subscription-service.hasAccess()      │
│  4. Mutate: Authorization = Bearer JWS                  │
│  5. Route /banking-service/** → BANKING-SERVICE         │
│  6. StripPrefix, RateLimit                              │
└────────────────────────┬────────────────────────────────┘
                         │  Bearer JWS (not JWE)
                         ▼
┌─────────────────────────────────────────────────────────┐
│                   Banking Service                       │
│                                                         │
│  1. JwtFilter — extract JWS, validate HMAC + expiry     │
│  2. Set SecurityContext (email, role, privileges)        │
│  3. TenantFilterInterceptor — enable Hibernate filter   │
│  4. anyRequest().authenticated() → OK                   │
│  5. Controller + @PreAuthorize                          │
│  6. Repository query (auto-scoped to tenantId)          │
└─────────────────────────────────────────────────────────┘
```

### Sequence Diagram

```
Client      Gateway     Redis       Sub-Service    Banking-Service
  │           │           │              │               │
  │──JWE──▶  │           │              │               │
  │           │─decrypt─▶ │              │               │
  │           │─system_lock,user_lock,token_version──▶  │
  │           │           │◀─ allow ─────│               │
  │           │─sub:tenant-abc:banking-service──▶        │
  │           │           │◀─ MISS ──────│               │
  │           │──────────────────────▶  │               │
  │           │                         │◀─ ALLOW ───── │
  │           │─cache ALLOW, TTL 5min─▶ │               │
  │           │──JWS──────────────────────────────────▶ │
  │           │                         │               │─validate JWS
  │           │                         │               │─enable tenant filter
  │           │                         │               │─query DB (scoped)
  │◀─ 200 ───────────────────────────────────────────── │
```

---

## 11. What Changes, Where

| Component | Change |
|---|---|
| **auth-service** | Add `tid` (tenantId) to JWS claims at token issuance. Add domain → tenant lookup at login. |
| **-common JwtValidator** | Add `getTenantId()` method to extract `tid` claim. |
| **API Gateway** | Add `SubscriptionFilter` at end of `AuthenticationFilterChain`. |
| **Each microservice JwtFilter** | Set `tenantId` and `role` as request attributes after claim extraction. |
| **-common BaseEntity** | Add `tenantId` field + `@FilterDef` + `@Filter` annotations. |
| **Each microservice** | Add `TenantFilterInterceptor` (can live in `-common`, auto-configured). |
| **admin-service** | Add user creation endpoint with pessimistic lock + license limit check. |
| **subscription-service** | New service, isolated DB, owns tenant/license/license_services tables. |
| **Redis** | New key pattern: `sub:{tenantId}:{serviceName}` |
| **Database** | Migration to add `tenant_id` column to all existing tables. |

### What Does NOT Change

- JWE/JWS two-layer token structure
- Existing filter chain order (SystemLock → UserLock → TokenVersion)
- Microservice SecurityConfig
- Route configuration in gateway
- Any business logic in any microservice
