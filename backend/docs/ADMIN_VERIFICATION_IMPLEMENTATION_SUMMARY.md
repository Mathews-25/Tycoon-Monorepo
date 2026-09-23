# Admin Verification Implementation Summary

This document summarizes the admin verification surface for the Tycoon backend and
shop-api, including the server-authoritative purchase write path, idempotency
handling, inventory guarantees, and the migration/index verification steps that
operators must confirm before promoting a release.

## Scope

- Backend (NestJS 11) admin routes for catalog and shop management.
- shop-api (NestJS) as the source of truth for purchases, inventory, and money.
- Migrations and indexes under `shop-api/src/migrations/`.

## Server-authoritative purchase write path

All purchase mutations are handled by shop-api. The backend never trusts
client-supplied prices, totals, or inventory counts; it proxies authenticated
requests to shop-api using API-key service auth only.

Write path:

1. Client sends `POST /purchases` with `Idempotency-Key` header and a validated
   body (SKU, quantity, minor units).
2. shop-api computes a body hash and looks up the idempotency record.
   - Replay with the same key and same body hash returns the stored response.
   - Replay with the same key and a different body hash returns `409 Conflict`.
3. Inventory is adjusted atomically (constraint or reservation TTL) so concurrent
   buys for the same SKU cannot oversell and inventory never goes negative.
4. `requestId` is propagated through the call chain and included in error
   responses.
5. Errors are mapped per `docs/API_ERROR_RESPONSE_STANDARDS.md`.

## DTO validation

- SKU: required, bounded length, allow-listed characters.
- Quantity: required, positive integer, bounded maximum.
- Minor units: required, non-negative integer, bounded maximum.
- Unknown fields are rejected (deny-by-default) per policy.

## Idempotency

- Keyed by `Idempotency-Key` plus body hash.
- Stored response is returned on replay within the TTL window.
- Payload conflict returns `409`.
- TTL expiry reuse is treated as a new request; operators should monitor replay
  rates to detect client retry storms.

## Inventory guarantees

- Atomic adjustment via DB constraint or reservation TTL.
- Concurrent checkout for the same SKU is serialized; oversell is rejected.
- Inventory is never allowed to go negative.

## Observability

- RED metrics for purchase: rate, errors, duration.
- `requestId` propagated to logs and error responses.
- No secrets or PII in telemetry labels.

## Migration and index verification

Before promoting a release, verify the shop-api purchases cleanup indexes
migration:

1. Confirm the migration under `shop-api/src/migrations/` is applied in the
   target environment and recorded in the migrations table.
2. Confirm the expected indexes exist on the purchases and inventory tables
   (idempotency key, SKU, created-at) and that no stale/duplicate indexes remain
   from the cleanup.
3. Confirm partial migration / canary states are resolved; no dual-write paths
   remain active.
4. Confirm fail-closed behavior: when shop-api is unavailable, writes are
   rejected rather than silently accepted.

## Failure modes

- Concurrent duplicate requests / reconnect retries: handled by idempotency.
- Dependency outage (Postgres/Redis/shop-api/RPC): writes fail closed.
- Auth expiry mid-flow and forbidden role access: rejected with mapped errors.
- Invalid or adversarial input: rejected by DTO validation and deny-by-default.
- Catalog edit during purchase: purchase uses the server-side catalog snapshot.

## Acceptance criteria

- [ ] No double purchase for one `Idempotency-Key`.
- [ ] Inventory never negative.
- [ ] Docs/runbooks updated.
- [ ] CI e2e green for purchase path.
- [ ] Fail-closed when shop-api unavailable on writes.

## References

- `shop-api/src/migrations/`
- `docs/API_ERROR_RESPONSE_STANDARDS.md`
- `SHOP_PURCHASES_RUNBOOK.md`
- ADR-001, ADR-003
