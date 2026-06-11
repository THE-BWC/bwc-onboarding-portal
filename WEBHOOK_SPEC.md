# BWC Member Provisioning Webhook Specification

**Version:** 1.1.0
**Last updated:** 2026-06-11
**Endpoint owner:** `bwc-onboarding-bot` (receiving end)
**Caller:** `OpServ` (dispatching end)

---

## Overview

When OpServ receives a sync trigger, it reads the user's current state from the shared forum database, resolves the correct Discord roles and nickname, and pushes a provisioning payload to the bot. The bot executes the payload without making any access decisions of its own.

This webhook is called at up to two points in the onboarding flow, depending on the approval variant in use:

| Sync point | Trigger | Variant | Expected outcome |
|---|---|---|---|
| 1 — Guest provisioning | Portal submits application to forum | Both A and B | User receives Guest role, nickname set, quarantine removed |
| 2 — Member promotion | Staff approve application | Variant A: staff trigger manually in OpServ. Variant B: portal triggers automatically on approval. | User receives game-specific member role(s), nickname updated, Guest role removed |

The payload shape is identical regardless of sync point or variant. The bot does not need to know which sync point it is responding to.

---

## Endpoint

```
POST /webhook/member-provisioned
```

Exposed on an internal port (default `8080`). Must not be publicly accessible. Restrict inbound traffic to OpServ's IP or network range at the reverse proxy or firewall level.

---

## Authentication

```
X-API-Key: <api_key>
```

The bot rejects requests with a missing or incorrect key with `401 Unauthorized`.

> Generate with `openssl rand -hex 32`. Rotate immediately if compromised.

---

## Request

### Headers

| Header | Required | Value |
|---|---|---|
| `Content-Type` | Yes | `application/json` |
| `X-API-Key` | Yes | Shared API key |
| `X-OpServ-Delivery-ID` | Yes | UUIDv4, unique per request, reused on retries |

### Body

```json
{
  "discord_user_id": "string",
  "forum_user_id": "string",
  "nickname": "string",
  "roles": ["string"],
  "remove_roles": ["string"],
  "member_type": "guest | member",
  "primary_game": "dune_awakening | mechwarrior_online | star_citizen | vanguard | null",
  "provisioned_at": "string (ISO 8601 UTC)"
}
```

### Field definitions

| Field | Type | Required | Description |
|---|---|---|---|
| `discord_user_id` | `string` | Yes | Discord snowflake ID of the member to provision. |
| `forum_user_id` | `string` | Yes | Forum database user ID. Stored by the bot for cross-reference and audit. |
| `nickname` | `string` | Yes | Nickname to set on the Discord member. Resolved by OpServ from the shared database. |
| `roles` | `string[]` | Yes | Discord role IDs to assign. Never empty. |
| `remove_roles` | `string[]` | No | Discord role IDs to remove. Sync point 1 typically includes the quarantine role. Sync point 2 typically includes the Guest role. Defaults to empty array if omitted. |
| `member_type` | `enum` | Yes | `guest` — temporary access pending approval. `member` — approved full member. |
| `primary_game` | `enum\|null` | Yes | Primary game division. `null` when `member_type` is `guest`. |
| `provisioned_at` | `string` | Yes | ISO 8601 UTC timestamp of when OpServ dispatched this payload. |

### Supported games

| `primary_game` value | Division |
|---|---|
| `dune_awakening` | Dune: Awakening |
| `mechwarrior_online` | MechWarrior Online |
| `star_citizen` | Star Citizen |
| `vanguard` | Vanguard |

### Example — sync point 1 (guest provisioning)

```json
{
  "discord_user_id": "123456789012345678",
  "forum_user_id": "4821",
  "nickname": "Guest_Hansen",
  "roles": ["111111111111111111"],
  "remove_roles": ["000000000000000001"],
  "member_type": "guest",
  "primary_game": null,
  "provisioned_at": "2026-06-11T14:32:00Z"
}
```

### Example — sync point 2 (member promotion)

```json
{
  "discord_user_id": "123456789012345678",
  "forum_user_id": "4821",
  "nickname": "Cmdr_Hansen",
  "roles": ["222222222222222222", "333333333333333333"],
  "remove_roles": ["111111111111111111"],
  "member_type": "member",
  "primary_game": "star_citizen",
  "provisioned_at": "2026-06-11T16:45:00Z"
}
```

---

## Bot processing steps

On receiving a valid, authenticated payload the bot executes the following in order. If any step fails, it returns `500` and OpServ should retry.

1. Validate API key — reject with `401` if invalid
2. Check `X-OpServ-Delivery-ID` for duplicates — return `409` if already processed
3. Validate payload structure — reject with `400` if malformed or missing required fields
4. Look up Discord member by `discord_user_id` in the configured guild — reject with `404` if not found
5. Assign all roles in `roles`
6. Remove all roles in `remove_roles` (if present)
7. Set member nickname to `nickname`
8. Post staff log embed to the staff log channel
9. Return `200 OK`

Steps 5, 6, and 7 use existing bot functionality. No new role or nickname logic is introduced.

---

## Response

No response body. Status code only.

| Status | Meaning | OpServ action |
|---|---|---|
| `200 OK` | All steps completed successfully. | Mark delivery complete. |
| `400 Bad Request` | Malformed payload or missing required fields. | Do not retry. Log the error. |
| `401 Unauthorized` | API key missing or invalid. | Do not retry. Alert maintainer. |
| `404 Not Found` | Discord member not found — may have left the server. | Do not retry. Log and flag for manual staff review. |
| `409 Conflict` | Duplicate delivery ID — already processed. | Treat as success. |
| `500 Internal Server Error` | Bot-side processing error. | Retry with exponential backoff (see below). |

---

## Retry policy

Retry only on `5xx` responses. Reuse the same `X-OpServ-Delivery-ID` on every retry.

| Attempt | Delay |
|---|---|
| 1st retry | 5 seconds |
| 2nd retry | 30 seconds |
| 3rd retry | 5 minutes |
| Give up | Log failure, flag member for manual staff review in OpServ |

Do not retry `4xx` responses. Treat `409` as success.

---

## Staff log embed

On successful provisioning the bot posts an embed to the configured staff log channel.

| Field | Value |
|---|---|
| Title | `Guest provisioned` or `Member provisioned` based on `member_type` |
| Discord user | Mention + snowflake ID |
| Nickname set | Value of `nickname` |
| Member type | `guest` or `member` |
| Primary game | Game name or `N/A` |
| Forum user ID | `forum_user_id` |
| Roles assigned | Comma-separated role mentions |
| Roles removed | Comma-separated role mentions, or `none` |
| Timestamp | `provisioned_at` |

---

## Environment variables

### Bot

| Variable | Description |
|---|---|
| `OPSERV_API_KEY` | Shared API key for authenticating inbound OpServ webhooks |
| `WEBHOOK_PORT` | Port the internal webhook listener binds to. Default `8080` |
| `DISCORD_GUILD_ID` | The BWC Discord server ID |
| `DISCORD_STAFF_LOG_CHANNEL_ID` | Channel ID for staff log embeds |
| `DISCORD_WELCOME_CHANNEL_ID` | Channel ID for the persistent onboarding button |
| `DISCORD_QUARANTINE_ROLE_ID` | Role ID assigned to all new members on join |

### OpServ

| Variable | Description |
|---|---|
| `BOT_WEBHOOK_URL` | Full internal URL of the bot's provisioning endpoint |
| `BOT_API_KEY` | Must match `OPSERV_API_KEY` on the bot |

---

## Versioning

### Changelog

| Version | Date | Notes |
|---|---|---|
| 1.0.0 | 2026-06-11 | Initial specification |
| 1.1.0 | 2026-06-11 | Added `forum_user_id`. Added two sync point model. Added variant context to sync point table. Clarified `remove_roles` per sync point. |

---

## Related documents

- `ARCHITECTURE.md` — Full system architecture and variant comparison
- `PORTAL_OPSERV_SPEC.md` — Portal → OpServ sync trigger specification

---

# Bot DM Webhook

**Endpoint owner:** `bwc-onboarding-bot`
**Caller:** `bwc-onboarding-portal`

---

## Overview

The portal instructs the bot to send a Discord DM to an applicant when a notification event occurs. The bot attempts delivery and reports success or failure. On failure, the portal falls back to email.

---

## Endpoint

```
POST /webhook/send-dm
```

Same internal port as `/webhook/member-provisioned`. Same IP restriction applies.

---

## Authentication

```
X-API-Key: <api_key>
```

Same shared key as the provisioning webhook (`OPSERV_API_KEY` on the bot, `BOT_API_KEY` on the portal).

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
  "discord_user_id": "string",
  "event": "string",
  "message": "string"
}
```

### Field definitions

| Field | Type | Required | Description |
|---|---|---|---|
| `discord_user_id` | `string` | Yes | Discord snowflake ID of the DM recipient. |
| `event` | `string` | Yes | Notification event type for logging purposes. One of: `application_received`, `application_on_hold`, `staff_comment`, `application_approved`, `application_rejected`. |
| `message` | `string` | Yes | Fully formatted message body to send. The portal is responsible for composing the message content — the bot sends it as-is. |

### Example

```json
{
  "discord_user_id": "123456789012345678",
  "event": "staff_comment",
  "message": "Recruiter Hansen has left a comment on your BWC application. Log in to the portal to view and respond: https://portal.blackwidowcompany.com/onboarding/status"
}
```

---

## Bot processing steps

1. Validate API key — reject with `401` if invalid
2. Validate payload — reject with `400` if malformed
3. Attempt to fetch Discord user by `discord_user_id`
4. Attempt to send DM
5. Return response

```python
try:
    user = await bot.fetch_user(discord_user_id)
    await user.send(message)
    return 200
except discord.Forbidden:
    return 500  # DMs disabled or bot blocked — portal falls back to email
except discord.NotFound:
    return 404  # User no longer exists — portal falls back to email
```

---

## Response

No response body. Status code only.

| Status | Meaning | Portal action |
|---|---|---|
| `200 OK` | DM delivered successfully. | Log as sent via Discord. No email sent. |
| `400 Bad Request` | Malformed payload. | Log error. Do not fall back to email. |
| `401 Unauthorized` | API key invalid. | Alert maintainer. Do not fall back to email. |
| `404 Not Found` | Discord user not found. | Fall back to email. |
| `500 Internal Server Error` | DM could not be delivered (DMs disabled, bot blocked, or other Discord error). | Fall back to email. |

The portal does **not** retry DM delivery. A single attempt is made — on any non-200 response the portal proceeds directly to email fallback.
