# Admin Routes Matrix

This document provides a comprehensive overview of all admin-protected routes in the Tycoon-Monorepo backend application.

## Overview

The backend uses two primary guards for admin access control:
- **AdminGuard**: Checks if `user.is_admin === true`
- **RolesGuard**: Checks if user has required role(s) specified via `@Roles()` decorator

## Admin-Protected Routes by Module

### 1. Admin Analytics Module

**Base Path**: `/admin/analytics`  
**Controller**: `AdminAnalyticsController`  
**Guards**: `JwtAuthGuard`, `AdminGuard`

| HTTP Method | Path | Purpose | Guard Used |
|-------------|------|---------|------------|
| GET | `/admin/analytics/dashboard` | Get dashboard analytics overview | AdminGuard |
| GET | `/admin/analytics/users/total` | Get total users count | AdminGuard |
| GET | `/admin/analytics/users/active` | Get active users count | AdminGuard |
| GET | `/admin/analytics/games/total` | Get total games count | AdminGuard |
| GET | `/admin/analytics/games/players/total` | Get total game players count | AdminGuard |

---

### 2. Admin Logs Module

**Base Path**: `/admin/logs`  
**Controller**: `AdminLogsController`  
**Guards**: `JwtAuthGuard`, `AdminGuard`

| HTTP Method | Path | Purpose | Guard Used |
|-------------|------|---------|------------|
| GET | `/admin/logs` | Retrieve admin audit logs with filters and pagination | AdminGuard |
| GET | `/admin/logs/export` | Export admin audit logs as CSV | AdminGuard |

---

### 3. Users Module

**Base Path**: `/users`  
**Controller**: `UsersController`  
**Guards**: `JwtAuthGuard`, `AdminGuard` (on specific endpoints)

| HTTP Method | Path | Purpose | Guard Used |
|-------------|------|---------|------------|
| GET | `/users` | List all users with pagination | AdminGuard |
| PATCH | `/users/:id` | Update a user by ID | AdminGuard |
| DELETE | `/users/:id` | Delete a user by ID | AdminGuard |
| POST | `/users/suspend` | Suspend a user account | AdminGuard |
| POST | `/users/unsuspend` | Unsuspend a user account | AdminGuard |
| GET | `/users/:id/suspensions` | Get suspension history for a user | AdminGuard |

---

### 4. Coupons Module

**Base Path**: `/coupons`  
**Controller**: `CouponsController`  
**Guards**: `JwtAuthGuard`, `AdminGuard` (on specific endpoints)

| HTTP Method | Path | Purpose | Guard Used |
|-------------|------|---------|------------|
| POST | `/coupons` | Create a new coupon | AdminGuard |
| PATCH | `/coupons/:id` | Update a coupon | AdminGuard |
| DELETE | `/coupons/:id` | Delete a coupon | AdminGuard |
| GET | `/coupons/:id/usage-logs` | Get coupon usage logs | AdminGuard |
| GET | `/coupons/:id/statistics` | Get coupon usage statistics | AdminGuard |

---

### 5. Perks Admin Module

**Base Path**: `/admin/perks`  
**Controller**: `PerksAdminController`  
**Guards**: `JwtAuthGuard`, `AdminGuard`

| HTTP Method | Path | Purpose | Guard Used |
|-------------|------|---------|------------|
| POST | `/admin/perks` | Create a new perk | AdminGuard |
| GET | `/admin/perks` | List all perks with pagination and filters | AdminGuard |
| GET | `/admin/perks/:id` | Get a perk by ID | AdminGuard |
| PATCH | `/admin/perks/:id` | Update a perk | AdminGuard |
| DELETE | `/admin/perks/:id` | Delete a perk (hard delete) | AdminGuard |
| PATCH | `/admin/perks/:id/activate` | Activate a perk | AdminGuard |
| PATCH | `/admin/perks/:id/deactivate` | Deactivate a perk | AdminGuard |
| GET | `/admin/perks/:perkId/boosts` | List boosts for a perk | AdminGuard |
| POST | `/admin/perks/:perkId/boosts` | Create a boost for a perk | AdminGuard |
| PATCH | `/admin/perks/:perkId/boosts/:boostId` | Update a boost | AdminGuard |
| DELETE | `/admin/perks/:perkId/boosts/:boostId` | Delete a boost | AdminGuard |

---

### 6. Waitlist Admin Module

**Base Path**: `/admin/waitlist`  
**Controller**: `WaitlistAdminController`  
**Guards**: `JwtAuthGuard`, `AdminGuard`

| HTTP Method | Path | Purpose | Guard Used |
|-------------|------|---------|------------|
| GET | `/admin/waitlist` | Retrieve all waitlist entries with pagination and filtering | AdminGuard |
| GET | `/admin/waitlist/export` | Export waitlist entries as CSV or Excel | AdminGuard |
| POST | `/admin/waitlist/bulk-import` | Bulk import waitlist entries from CSV | AdminGuard |
| PATCH | `/admin/waitlist/:id` | Update a waitlist entry | AdminGuard |
| DELETE | `/admin/waitlist/:id` | Soft delete a waitlist entry | AdminGuard |
| DELETE | `/admin/waitlist/:id/permanent` | Permanently delete a waitlist entry | AdminGuard |

---

### 7. Chance Module

**Base Path**: `/chances`  
**Controller**: `ChanceController`  
**Guards**: `JwtAuthGuard`, `RolesGuard` (on specific endpoints)

| HTTP Method | Path | Purpose | Guard Used |
|-------------|------|---------|------------|
| POST | `/chances` | Create a new chance card | RolesGuard + @Roles(Role.ADMIN) |

---

### 8. Shop Admin Module

**Base Path**: `/admin/shop`  
**Controller**: `ShopAdminController`  
**Guards**: `JwtAuthGuard`, `AdminGuard`

| HTTP Method | Path | Purpose | Guard Used |
|-------------|------|---------|------------|
| POST | `/admin/shop/items` | Create a shop item (SKU, price in minor units) | AdminGuard |
| PATCH | `/admin/shop/items/:id` | Update a shop item (catalog edit) | AdminGuard |
| DELETE | `/admin/shop/items/:id` | Delete a shop item | AdminGuard |
| PATCH | `/admin/shop/items/:id/inventory` | Adjust inventory for a SKU | AdminGuard |

**Notes**:
- All catalog mutations are audited via `AdminLogsService` (actor, SKU, before/after).
- Prices are stored and validated in minor units; clients never supply trusted prices.
- Inventory adjustments are atomic and must never drive stock negative.

---

### 9. Shop Purchases Module (shop-api)

**Base Path**: `/shop/purchases`  
**Controller**: `ShopPurchasesController` (shop-api)  
**Guards**: `JwtAuthGuard` (player auth); service-to-service calls use API-key auth only

| HTTP Method | Path | Purpose | Guard Used |
|-------------|------|---------|------------|
| POST | `/shop/purchases` | Authoritative purchase write path (requires `Idempotency-Key`) | JwtAuthGuard |
| GET | `/shop/purchases/:id` | Read a purchase by id (read model) | JwtAuthGuard |

**Notes**:
- shop-api is the single source of truth for purchases; the backend proxies reads/writes and never trusts client-supplied prices.
- `Idempotency-Key` is required on `POST /shop/purchases`. The request body hash is stored with the key; replays return the stored response, and a payload mismatch for the same key returns `409 Conflict`.
- Inventory is adjusted atomically (constraint / reservation TTL) so concurrent buys for the same SKU cannot oversell and stock never goes negative.
- DTO validation enforces SKU, quantity, and minor-unit bounds and rejects unknown fields.
- `requestId` is propagated end-to-end; errors map to `docs/API_ERROR_RESPONSE_STANDARDS.md`; RED metrics are emitted for the purchase path.
- Writes fail closed when shop-api (or its Postgres/Redis dependencies) is unavailable.

---

## Guard Implementations

### AdminGuard

**Location**: `src/modules/auth/guards/admin.guard.ts`

**Behavior**:
- Checks if `user.is_admin === true`
- Throws `ForbiddenException` with message "Access denied. Admin role required." if not admin
- Returns `true` if user is admin

**Usage**:
```typescript
@UseGuards(JwtAuthGuard, AdminGuard)
```

### RolesGuard

**Location**: `src/modules/auth/guards/roles.guard.ts`

**Behavior**:
- Checks if user has any of the required roles specified via `@Roles()` decorator
- Returns `true` if no roles are required (permissive by default)
- Returns `true` if user has at least one of the required roles
- Returns `false` if user doesn't have required roles

**Usage**:
```typescript
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles(Role.ADMIN)
```

---

## Security Notes

1. **Always use JwtAuthGuard first**: Admin guards should always be paired with `JwtAuthGuard` to ensure the user is authenticated before checking admin status.

2. **AdminGuard vs RolesGuard**: 
   - Use `AdminGuard` for simple admin-only checks (checks `is_admin` field)
   - Use `RolesGuard` with `@Roles()` decorator for role-based access (checks `role` field)

3. **Default Behavior**: 
   - `AdminGuard` denies by default (throws exception if not admin)
   - `RolesGuard` **now denies by default** (throws exception if no `@Roles()` decorator present or user doesn't have required role)

---

## Security Recommendations

1. **✅ RolesGuard Default Deny**: RolesGuard has been updated to deny access by default when no `@Roles()` decorator is present. This ensures routes must explicitly declare required roles.

2. **Consistent Guard Usage**: Ensure all admin endpoints use appropriate guards consistently.

3. **Integration Testing**: All admin-protected routes should have integration tests verifying 403 responses for non-admin users. See `test/admin-role-verification.e2e-spec.ts` for examples.

4. **Admin Action Logging**: Consider logging all admin actions for audit purposes using `AdminLogsService`.

5. **Rate Limiting**: Apply stricter rate limits to admin endpoints to prevent abuse.

---

## Last Updated

Document created: 2024
Last reviewed: 2024
