# BWC Portal → OpServ Sync Trigger Specification

**Version:** 2.0.0
**Last updated:** 2026-06-11
**Endpoint owner:** `OpServ` (receiving end)
**Caller:** `bwc-onboarding-portal` (dispatching end)

---

## Overview

The portal triggers an OpServ sync to initiate a Discord role and nickname update. The portal does not determine roles or nicknames — it only tells OpServ which forum user to sync. OpServ reads the user's current state from the shared forum database and resolves everything itself.

### When the portal triggers a sync

| Sync point | Variant | When triggered |
|---|---|---|
| 1 — Guest provisioning | Both A and B | Immediately after the portal submits the application to the forum |
| 2 — Member promotion | Variant B only | When a staff member approves the application in the portal |

In **Variant A**, sync point 2 is triggered manually by staff in OpServ — the portal is not involved. See `ARCHITECTURE.md` for the full comparison.

---

## Endpoint

```
POST /api/sync-discord
```

Internal OpServ endpoint. Restrict inbound traffic to the portal's IP or network range.

---

## Authentication

```
X-API-Key: <api_key>
```

OpServ rejects requests with a missing or incorrect key with `401 Unauthorized`.

> Generate with `openssl rand -hex 32`. Rotate immediately if compromised.

---

## Request

### Headers

| Header | Required | Value |
|---|---|---|
| `Content-Type` | Yes | `application/json` |
| `X-API-Key` | Yes | Shared API key |

### Body

```json
{
  "forum_user_id": "string",
  "discord_user_id": "string",
  "sync_reason": "guest_provisioning | member_promotion"
}
```

### Field definitions

| Field | Type | Required | Description |
|---|---|---|---|
| `forum_user_id` | `string` | Yes | Forum database user ID. OpServ uses this to look up the user's groups and profile in the shared database. |
| `discord_user_id` | `string` | Yes | Discord snowflake ID. Passed through to the bot payload. |
| `sync_reason` | `enum` | Yes | `guest_provisioning` — fired after application submission (both variants). `member_promotion` — fired after staff approval in portal (Variant B only). Used by OpServ for logging; does not change how OpServ resolves roles. |

### Example — sync point 1 (both variants)

```json
{
  "forum_user_id": "4821",
  "discord_user_id": "123456789012345678",
  "sync_reason": "guest_provisioning"
}
```

### Example — sync point 2 (Variant B only)

```json
{
  "forum_user_id": "4821",
  "discord_user_id": "123456789012345678",
  "sync_reason": "member_promotion"
}
```

---

## OpServ processing steps

1. Validate API key
2. Look up `forum_user_id` in the shared database — return `404` if not found
3. Read user groups, membership state, and profile
4. Resolve Discord roles, roles to remove, and nickname
5. Push provisioning payload to the bot (see `WEBHOOK_SPEC.md`)
6. Return `200 OK`

The portal does not wait for the bot to finish — OpServ's `200` confirms the sync has been dispatched to the bot, not that Discord has been updated yet.

---

## Response

No response body. Status code only.

| Status | Meaning | Portal action |
|---|---|---|
| `200 OK` | Sync dispatched to bot. | Mark sync complete. |
| `400 Bad Request` | Malformed payload or missing fields. | Do not retry. Log the error. |
| `401 Unauthorized` | API key missing or invalid. | Do not retry. Alert maintainer. |
| `404 Not Found` | `forum_user_id` not found in the database. | Do not retry. Log and surface to staff. |
| `500 Internal Server Error` | OpServ-side error. | Retry with exponential backoff (see below). |

---

## Retry policy

Retry only on `5xx` responses.

| Attempt | Delay |
|---|---|
| 1st retry | 5 seconds |
| 2nd retry | 30 seconds |
| 3rd retry | 5 minutes |
| Give up | Log failure, surface to staff for manual intervention |

---

## Variant A — Manual sync point 2

In Variant A the portal does not trigger sync point 2. After guest provisioning the portal's connection to the user ends. When staff approve the application on the forum, they manually trigger a Discord sync in OpServ. This is an OpServ-internal action and is outside the scope of this specification.

---

## Environment variables

### Portal

| Variable | Description |
|---|---|
| `OPSERV_SYNC_URL` | Full URL of OpServ's sync endpoint |
| `OPSERV_API_KEY` | Must match the API key configured in OpServ |

### OpServ

| Variable | Description |
|---|---|
| `PORTAL_API_KEY` | Shared API key for authenticating inbound portal requests |

---

## Versioning

### Changelog

| Version | Date | Notes |
|---|---|---|
| 1.0.0 | 2026-06-11 | Initial specification |
| 2.0.0 | 2026-06-11 | Added two-variant model. Variant A: portal triggers sync point 1 only. Variant B: portal triggers both sync points. Clarified OpServ processes both identically. |

---

## Related documents

- `ARCHITECTURE.md` — Full system architecture and variant comparison
- `WEBHOOK_SPEC.md` — OpServ → bot provisioning webhook specification
