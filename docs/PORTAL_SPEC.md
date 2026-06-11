# BWC Onboarding Portal — Specification

**Version:** 1.0.0
**Last updated:** 2026-06-11

---

## Overview

The onboarding portal is a TanStack Start application that serves as the user-facing onboarding interface for BWC. It handles Discord OAuth2, forum account creation, application submission, and applicant communication. In Variant B it also serves as the staff approval interface.

All communication with Discord happens via the bot. The portal never calls the Discord API directly.

---

## Tech stack

| Concern | Technology |
|---|---|
| Framework | TanStack Start |
| Routing | TanStack Router (file-based) |
| Forms | TanStack Form |
| Server functions | TanStack Start server functions |
| Database access | Drizzle ORM (MariaDB) |
| Authentication | Discord OAuth2 (applicants + Variant B staff) |
| Session tokens | JWT (signed, portal-scoped) |
| Email | SMTP via internal BWC mail server |
| Styling | Tailwind CSS |

---

## Database access

The portal connects directly to the shared MariaDB instance. It has **read access** to XenForo and OpServ tables, and **read/write access** to its own portal tables.

### Read-only tables (XenForo / OpServ)

| Table | Purpose |
|---|---|
| `opserv_discord_user_links` | Resolve Discord ID → XenForo user ID |
| `xf_user` | Confirm forum account existence, retrieve display name and email |
| `xf_user_group_relation` | Check staff group memberships for portal access (Variant B) |
| `opserv_games` | Retrieve active games list and OPSEC status |

### Portal-owned tables

| Table | Purpose |
|---|---|
| `portal_applications` | Application records |
| `portal_application_comments` | Comment thread per application |
| `portal_notification_log` | Record of sent notifications (DM + email) |

---

## Route structure

```
app/routes/
├── index.tsx                          # Landing page — Sign up with Discord (+ Staff login in Variant B)
├── auth/
│   ├── discord/callback.tsx           # OAuth2 callback handler
│   └── logout.tsx
├── onboarding/
│   ├── index.tsx                      # Post-OAuth2 entry — new vs returning account check
│   ├── apply.tsx                      # Game-specific application form
│   └── status.tsx                     # Applicant's application status + comment thread
└── staff/                             # Variant B only
    ├── login.tsx                      # Staff Discord OAuth2 entry
    ├── index.tsx                      # Application queue
    ├── applications/
    │   └── $applicationId.tsx         # Individual application review + comment thread + approval actions
    └── auth/
        └── discord/callback.tsx       # Staff OAuth2 callback
```

---

## Authentication flows

### Applicant — Discord OAuth2

```
1. User hits landing page → clicks "Sign up with Discord"
2. Portal redirects to Discord OAuth2
     Scopes: identify (Discord ID, username, avatar only — no email)
3. Discord redirects to /auth/discord/callback with code
4. Portal exchanges code for access token (server function)
5. Portal calls Discord /users/@me to get verified Discord ID
6. Portal queries opserv_discord_user_links for existing forum account
     │
     ├── No linked account → proceed to forum account creation
     │
     └── Linked account found → check for existing portal application
           │
           ├── Active application exists → redirect to /onboarding/status
           └── No active application → redirect to /onboarding/apply
7. Portal issues signed JWT (applicant-scoped)
     Payload: { discord_user_id, xenforo_user_id, scope: "applicant" }
     Expiry: 24 hours
```

### Staff — Discord OAuth2 (Variant B only)

```
1. Staff member hits landing page → clicks "Staff login"
2. Portal redirects to Discord OAuth2 (same scopes: identify)
3. Discord redirects to /staff/auth/discord/callback
4. Portal exchanges code for access token (server function)
5. Portal gets verified Discord ID from Discord /users/@me
6. Portal queries opserv_discord_user_links for linked XenForo user
     │
     └── No linked account → deny access, show error
7. Portal queries xf_user_group_relation for staff portal permission
     (specific group IDs / permission entry TBD — defined internally)
     │
     └── No qualifying permission → deny access, show error
8. Portal issues signed JWT (staff-scoped)
     Payload: { discord_user_id, xenforo_user_id, display_name, scope: "staff" }
     Expiry: 8 hours
```

---

## Forum account creation

If no `opserv_discord_user_links` record exists for the Discord ID, the portal creates a XenForo account via the XenForo REST API before proceeding to the application form.

```
POST /api/users/
Headers:
  XF-Api-Key: <super user key>
  XF-Api-User: <yeoman account user ID>   (Variant B)
                <applicant derived>        (Variant A — see Forum Posting section)
Body:
  username: <derived from Discord display name>
  email:    <collected on account creation form>
  password: <randomly generated, user sets their own later>
```

The created XenForo user ID is then inserted into `opserv_discord_user_links` to link the accounts.

> **Note:** Email is collected by the portal on a brief account setup step before the application form, since Discord's `identify` scope does not return an email address.

---

## Game selection

The application form populates its game dropdown by querying `opserv_games` directly:

```sql
SELECT game_id, game_name, taskforce_name, tag, icon
FROM opserv_games
WHERE retired = 0
ORDER BY game_name ASC
```

Retired games are excluded. The OPSEC flag is not surfaced to the applicant — it is read separately at submission time to determine the internal review flow.

---

## Application submission

### On submit, the portal executes the following in order:

1. Validate form data (server function)
2. Query OPSEC status for the selected game:
   ```sql
   SELECT opsec FROM opserv_games WHERE game_id = ? AND retired = 0
   ```
3. Create `portal_applications` record with initial internal state (see State machine)
4. Post mirror thread to XenForo (see Forum posting section)
5. Trigger OpServ guest sync (see `PORTAL_OPSERV_SPEC.md`, sync point 1)
6. Send applicant confirmation notification (see Notifications)
7. Redirect applicant to `/onboarding/status`

If any step from 4 onwards fails, the application record is still created — the portal logs the failure and retries asynchronously. The applicant is not shown an error for downstream failures.

---

## Application schema

```sql
CREATE TABLE portal_applications (
  id                  INT AUTO_INCREMENT PRIMARY KEY,
  discord_user_id     VARCHAR(64) NOT NULL,
  xenforo_user_id     INT NOT NULL,
  game_id             INT NOT NULL,
  join_reason         ENUM('to_play', 'guest') NOT NULL,
  form_data           JSON NOT NULL,
  internal_state      ENUM('pending', 's2_review', 'recruiter_review', 'on_hold', 'approved', 'rejected') NOT NULL DEFAULT 'pending',
  applicant_state     ENUM('pending', 'on_hold', 'approved', 'rejected') NOT NULL DEFAULT 'pending',
  is_opsec            TINYINT(1) NOT NULL DEFAULT 0,
  xenforo_thread_id   INT,
  previous_application_id INT,
  submitted_at        DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  resolved_at         DATETIME
);
```

`previous_application_id` links a re-application to the most recent prior record for that applicant, enabling the portal to surface history to staff during review.

`form_data` stores the raw application answers as JSON so the schema does not need to change if application questions are updated.

---

## State machine

### Internal states (staff-facing only)

| State | Meaning | Applicant sees |
|---|---|---|
| `pending` | Submitted, not yet actioned | `pending` |
| `s2_review` | Referred to game's S-2 division | `pending` |
| `recruiter_review` | S-2 cleared or skipped, with recruiter | `pending` |
| `on_hold` | Awaiting further information from applicant | `on_hold` |
| `approved` | Accepted | `approved` |
| `rejected` | Not accepted | `rejected` |

### State transitions

```
submitted
    │
    ▼
[is_opsec = 1?]
    ├── Yes → s2_review → recruiter_review
    └── No  → recruiter_review
                  │
                  ├── on_hold ──► (applicant responds via comment) ──► recruiter_review
                  ├── approved
                  └── rejected
```

### Re-application

A rejected applicant may submit a new application. This creates a fresh `portal_applications` record with `previous_application_id` set to the prior record's ID. The previous application is not reopened or modified. Staff reviewing the new application see a collapsible history panel showing all prior applications, their final states, submission dates, and resolved dates.

Re-application questions may differ from the initial application — `form_data` stores answers as JSON so question sets can evolve independently of the schema.

---

## Comment system

Each application has a comment thread stored in `portal_application_comments`. Comments are visible to both the applicant and staff, but staff-only internal notes are hidden from the applicant.

```sql
CREATE TABLE portal_application_comments (
  id                  INT AUTO_INCREMENT PRIMARY KEY,
  application_id      INT NOT NULL,
  author_discord_id   VARCHAR(64) NOT NULL,
  author_display_name VARCHAR(128) NOT NULL,
  body                TEXT NOT NULL,
  is_internal         TINYINT(1) NOT NULL DEFAULT 0,
  created_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

`is_internal = 1` marks a comment as staff-only. The portal filters these out of the applicant's view entirely. Internal notes are shown only in the staff review interface.

`author_display_name` is stored at write time from the JWT payload so comment history remains accurate even if a staff member's display name changes later.

### Comment visibility rules

| Author | `is_internal` | Applicant sees | Staff sees |
|---|---|---|---|
| Applicant | 0 | Yes | Yes |
| Staff | 0 | Yes | Yes |
| Staff | 1 | No | Yes |

When a staff member posts a non-internal comment, the applicant receives a notification (see Notifications).
When an applicant posts a comment, all staff with access to that application receive a notification.

---

## Notifications

All notifications are dispatched server-side. The portal instructs the bot to send a Discord DM. If the bot returns a non-200 response, the portal falls back to email via SMTP.

### Bot DM endpoint

```
POST /webhook/send-dm
```

See `WEBHOOK_SPEC.md` for the full DM webhook specification.

### Email fallback

The portal retrieves the applicant's email from `xf_user.email` and sends via the internal BWC SMTP server. No XenForo email system is used.

### Notification events

| Event | Recipient | Trigger |
|---|---|---|
| Application received | Applicant | On submission |
| Application on hold | Applicant | Internal state → `on_hold` |
| Staff comment posted | Applicant | Staff posts non-internal comment |
| Applicant comment posted | All staff with queue access | Applicant posts comment |
| Application approved | Applicant | Internal state → `approved` |
| Application rejected | Applicant | Internal state → `rejected` |

### Notification content

Notifications reference the staff member's `author_display_name` where relevant (e.g. "Recruiter Hansen has left a comment on your application"). Staff Discord handles and forum accounts are not exposed to applicants.

### Notification log schema

```sql
CREATE TABLE portal_notification_log (
  id                  INT AUTO_INCREMENT PRIMARY KEY,
  application_id      INT NOT NULL,
  event               VARCHAR(64) NOT NULL,
  recipient_discord_id VARCHAR(64) NOT NULL,
  channel             ENUM('discord_dm', 'email') NOT NULL,
  status              ENUM('sent', 'failed') NOT NULL,
  sent_at             DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

---

## Forum posting

The portal posts a mirror thread to XenForo on application submission. The mechanism differs by variant.

### Variant A — Post as applicant (super user key)

The application needs to be authored by the applicant for staff to interact with them directly on the forum. A super user key is required to post as another user.

```
POST /api/threads/
Headers:
  XF-Api-Key: <super user key>
  XF-Api-User: <applicant's xenforo_user_id>
  Content-Type: application/x-www-form-urlencoded
Scopes required: thread:write, thread:read
Body:
  node_id:  <application subforum node ID>
  title:    <applicant username> — <game name> Application
  message:  <see thread body template below>
```

### Variant B — Post as Yeoman account (user key)

The forum is a passive archive. Authorship is irrelevant — the Yeoman account posts on behalf of the portal.

```
POST /api/threads/
Headers:
  XF-Api-Key: <yeoman user key>
  Content-Type: application/x-www-form-urlencoded
Scopes required: thread:write, thread:read
Body:
  node_id:  <application subforum node ID>
  title:    <applicant username> — <game name> Application
  message:  <see thread body template below>
```

### Thread body template

```
Applicant:      <discord_username> (<discord_user_id>)
Forum account:  <xenforo_username> (ID: <xenforo_user_id>)
Game applied:   <game_name> (<taskforce_name>)
Join reason:    <join_reason>
Submitted:      <submitted_at>

--- Application ---
<form_data rendered as key: value pairs>
```

### Configuration

The active variant, API key type, and subforum node ID are all environment config. The portal branches its forum posting logic based on the `PORTAL_VARIANT` environment variable.

---

## Environment variables

| Variable | Description |
|---|---|
| `PORTAL_VARIANT` | `A` or `B` — determines forum posting method and whether staff login is enabled |
| `DISCORD_CLIENT_ID` | Discord OAuth2 application client ID |
| `DISCORD_CLIENT_SECRET` | Discord OAuth2 application client secret |
| `DISCORD_REDIRECT_URI` | OAuth2 callback URL |
| `JWT_SECRET` | Secret for signing portal session tokens |
| `MARIADB_HOST` | Shared MariaDB host |
| `MARIADB_PORT` | Default `3306` |
| `MARIADB_USER` | Portal DB user (read access to XenForo/OpServ tables, read/write to portal tables) |
| `MARIADB_PASSWORD` | Portal DB password |
| `MARIADB_DATABASE` | Database name |
| `XENFORO_API_URL` | XenForo REST API base URL |
| `XENFORO_API_KEY` | Super user key (Variant A) or Yeoman user key (Variant B) |
| `XENFORO_APPLICATION_NODE_ID` | Subforum node ID for application threads |
| `XENFORO_YEOMAN_USER_ID` | Yeoman account XenForo user ID (Variant B only) |
| `OPSERV_SYNC_URL` | OpServ sync endpoint URL |
| `OPSERV_API_KEY` | API key for OpServ sync trigger |
| `BOT_WEBHOOK_URL` | Bot webhook base URL |
| `BOT_API_KEY` | API key for bot webhook calls |
| `SMTP_HOST` | Internal BWC mail server host |
| `SMTP_PORT` | SMTP port |
| `SMTP_USER` | SMTP username |
| `SMTP_PASSWORD` | SMTP password |
| `SMTP_FROM` | From address for notification emails |
| `STAFF_PORTAL_GROUP_IDS` | Comma-separated XenForo group IDs permitted portal access (Variant B, TBD internally) |

---

## Versioning

### Changelog

| Version | Date | Notes |
|---|---|---|
| 1.0.0 | 2026-06-11 | Initial specification |

---

## Related documents

- `ARCHITECTURE.md` — Full system architecture and variant comparison
- `WEBHOOK_SPEC.md` — OpServ → bot provisioning webhook and bot DM webhook
- `PORTAL_OPSERV_SPEC.md` — Portal → OpServ sync trigger specification
