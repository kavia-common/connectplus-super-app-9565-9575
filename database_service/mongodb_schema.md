# MongoDB Schema (Step 2.1) — ConnectPlus Super App

This document captures the MongoDB collection validators, indexes (including TTL), and seed data created for the core domain.

## Connection

This container generates a connection command in:

- `database_service/db_connection.txt` (contains a `mongosh mongodb://...` command)

Example:

```bash
mongosh mongodb://appuser:dbuser123@localhost:5000/myapp?authSource=admin
```

## Security & Auditing Notes

- PII fields are intended to be stored **encrypted at rest** (field-level encryption performed in the backend). In MongoDB, encrypted values should be stored as `BinData`:
  - `users.mobile_enc`: `BinData`
  - `users.email_enc`: `BinData` or `null`

- **Soft delete + lifecycle**:
  - Users include `status` (`ACTIVE|DISABLED|DELETED`) and `deleted_at`.
  - Sessions include `revoked_at`.
  - Tickets include `closed_at`.
  - Audits are append-only (no TTL by default).

- **Schema validation**:
  - Collections use `$jsonSchema` validators with `validationLevel: "moderate"`.
  - `additionalProperties: false` is enabled to reduce accidental data sprawl.
  - `_id` is explicitly allowed in each schema’s `properties` (MongoDB injects it automatically).

## Collections + Key Fields

### `users`
- Required: `mobile_enc`, `password_hash`, `roles`, `status`, `created_at`, `updated_at`
- Notable fields:
  - `mobile_enc` (BinData, unique)
  - `email_enc` (BinData|null, unique when present)
  - `roles`: array of strings (RBAC)
  - `failed_login_attempts`, `lock_until`, `last_login_at` (security)

Indexes:
- `uniq_mobile_enc` (unique)
- `uniq_email_enc` (unique, partial)

### `sessions`
- Required: `user_id`, `refresh_token_hash`, `created_at`, `expires_at`

Indexes:
- `idx_sessions_user`
- `ttl_sessions_expires` with `expireAfterSeconds: 0` on `expires_at`

### `plans`
- Required: `name`, `speed_mbps`, `price`, `areas`, `status`, `created_at`, `updated_at`

Indexes:
- `idx_plans_areas`
- `idx_plans_price`
- `idx_plans_speed`
- `idx_plans_status`

### `service_areas`
- Required: `pincode`, `city`, `status`, `created_at`, `updated_at`

Indexes:
- `uniq_servicearea_pincode` (unique)
- `idx_servicearea_status`

### `engineers`
- Required: `name`, `phone`, `skills`, `areas`, `workload`, `status`, `created_at`, `updated_at`

Indexes:
- `idx_engineers_areas`
- `idx_engineers_skills`
- `idx_engineers_status_workload`

### `orders`
- Required: `user_id`, `plan_id`, `address`, `price`, `status`, `slot`, `created_at`, `updated_at`
- Includes `timeline[]` for status/audit history

Indexes:
- `idx_orders_user`
- `idx_orders_status`
- `idx_orders_assigned_engineer`

### `tickets`
- Required: `user_id`, `issue_type`, `severity`, `status`, `created_at`, `updated_at`
- Includes `notes[]` for audit trail

Indexes:
- `idx_tickets_user`
- `idx_tickets_status`
- `idx_tickets_issue_type`
- `idx_tickets_created_at`

### `conversations`
- Required: `user_id`, `started_at`, `last_message_at`

Indexes:
- `idx_conversations_user`
- `idx_conversations_last_message_at`

### `messages`
- Required: `conversation_id`, `sender`, `created_at`

Indexes:
- `idx_messages_conversation`
- `idx_messages_created_at`

### `notifications`
- Required: `user_id`, `type`, `status`, `created_at`
- Optional `expires_at` used for TTL

Indexes:
- `idx_notifications_user`
- `idx_notifications_status`
- `ttl_notifications_expires` with `expireAfterSeconds: 0` on `expires_at` (partial)

### `audits`
- Required: `actor`, `action`, `entity`, `entity_id`, `ts`

Indexes:
- `idx_audits_entity`
- `idx_audits_ts`

## Seed Data

Seed data is inserted using **idempotent upserts** (`updateOne(..., {$setOnInsert: ...}, {upsert:true})`) for:

- Plans: `Starter 100`, `Night Streaming 300`, `Ultra 500`
- Service areas: `560034`, `560001`
- Engineers:
  - `Asha R` (+91-90000-00001) — install+support, area 560034
  - `Rohit K` (+91-90000-00002) — install, areas 560001+560034

## TTL

- `sessions.expires_at` is TTL (hard expiry)
- `notifications.expires_at` is TTL (for ephemeral notifications)

Task completed: MongoDB schema validators, indexes (incl TTL), and seed data documented and applied for core collections.
