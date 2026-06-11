# BWC Onboarding System — Architecture Specification

**Version:** 2.0.0
**Last updated:** 2026-06-11

---

## Overview

The BWC onboarding system spans four services. Two approval flow variants are documented below. The core service responsibilities and the bot's behaviour are identical in both — the difference is entirely in how and where staff approve applications after submission.

| Service | Responsibility |
|---|---|
| `bwc-onboarding-bot` | Discord event handling, role assignment, nickname setting |
| `bwc-onboarding-portal` | Discord OAuth2, forum account creation, application submission. In Variant B: staff approval UI. |
| `Forum` | Receives submitted applications. In Variant A: primary staff workspace. In Variant B: mirror/archive. |
| `OpServ` | Central authority on roles and nicknames. Reads from the shared forum database. Pushes provisioning payloads to the bot. |

---

## Shared pre-submission flow

Both variants share the same flow from member join through to application submission. They diverge only after the application reaches the forum.

```
1.  Discord user joins the BWC server
          │
          ▼
2.  Bot responds via two parallel channels:
      a. DMs the user with an onboarding link
      b. Persistent "Start BWC Onboarding" button in #welcome channel
          │
          ▼
3.  User clicks the onboarding link
          │
          ▼
4.  Portal initiates Discord OAuth2
        - User authenticates with Discord
        - Portal receives: Discord ID, display name, avatar
          │
          ▼
5.  Portal creates or links forum account
        - New user:       forum account created, linked to Discord ID
        - Returning user: existing forum account linked
          │
          ▼
6.  User completes game-specific application form on the portal
          │
          ▼
7.  Portal submits the completed application to the forum
        - Application posted to the correct subforum
        - Portal's involvement with the user ends here
          │
          ▼
8.  Portal triggers OpServ guest sync (sync point 1)
        - Payload: forum_user_id + discord_user_id
        - OpServ reads user state from shared DB
        - OpServ pushes guest provisioning payload to bot
          │
          ▼
9.  Bot executes guest provisioning:
        - Assigns Guest role
        - Sets nickname
        - Removes quarantine role
        - Posts summary to staff log channel
          │
          ▼
     [User has guest access. Portal connection ends.]
          │
          ▼
     ── VARIANT A or VARIANT B (see below) ──────────────────
```

---

## Variant A — Manual approval (forum + OpServ)

After guest provisioning, all further steps are handled by staff outside the portal. The portal is no longer involved.

```
10. Staff review the application on the forum
          │
          ▼
11. Staff approve or reject on the forum
        - Approval updates the user's forum groups in the shared database
          │
          ▼
12. Staff manually trigger a Discord sync in OpServ
        - OpServ reads the updated user state from the shared database
        - OpServ determines: member roles + updated nickname
        - OpServ pushes member provisioning payload to bot
          │
          ▼
13. Bot executes member provisioning:
        - Assigns game-specific member role(s)
        - Updates nickname
        - Removes Guest role
        - Posts updated summary to staff log channel
          │
          ▼
     [User has full member access]
```

### Variant A — Communication map

```
[Portal] ──► [Forum]         (application posted)
[Portal] ──► [OpServ]        (guest sync trigger, sync point 1)
[OpServ] ──► [Bot]           (guest provisioning payload)

--- portal connection ends ---

[Forum]                      (staff review and approve)
[OpServ]                     (staff manually trigger sync, sync point 2)
[OpServ] ──► [Bot]           (member provisioning payload)
```

### Variant A — Tradeoffs

| Consideration | Detail |
|---|---|
| Portal complexity | Low — portal has no approval UI, no staff authentication |
| Staff workflow | Split across two tools (forum for review, OpServ for sync trigger) |
| Automation | Minimal — sync is a deliberate manual step |
| Risk of desync | Low — staff explicitly confirm when to sync |
| Implementation effort | Low |

---

## Variant B — Portal-driven approval

After guest provisioning, staff log in to the portal to review and approve applications. The forum receives a mirror of the application for record-keeping but is not the primary staff workspace. The portal drives the full approval lifecycle and triggers OpServ automatically on approval.

```
10. Staff click "Staff login" on the portal landing page
          │
          ▼
11. Portal initiates Discord OAuth2 (same flow as member signup)
        - Staff member authenticates with Discord
        - Portal receives verified Discord ID
          │
          ▼
12. Portal queries MariaDB to resolve Discord ID → XenForo user
        - Looks up discord_user_id in opserv_discord_user_links
        - Joins to xf_user to confirm the account exists
        - Joins to xf_user_group_relation to fetch group memberships
          │
          ├── No linked forum account → deny access
          ├── Linked account found but no qualifying permission → deny access
          │
          ▼
13. Portal checks group memberships against the configured
    portal access permission or staff group ID list
    (specific permission/groups TBD — to be defined internally)
          │
          ▼
14. Portal issues a signed JWT scoped to the staff interface
        - Token contains XenForo user ID, Discord ID, and expiry
        - Staff session is portal-only — no XenForo forum session is created
          │
          ▼
15. Staff review the application in the portal
        - Portal displays application data, Discord identity, and forum account link
          │
          ▼
16. Staff approve or reject in the portal
        - Approval updates the user's forum groups via the shared MariaDB
        - Portal automatically triggers OpServ member sync (sync point 2)
          │
          ▼
17. OpServ receives sync trigger from portal
        - Reads updated user state from shared database
        - Determines: member roles + updated nickname
        - Pushes member provisioning payload to bot
          │
          ▼
18. Bot executes member provisioning:
        - Assigns game-specific member role(s)
        - Updates nickname
        - Removes Guest role
        - Posts updated summary to staff log channel
          │
          ▼
     [User has full member access]
```

### Variant B — Communication map

```
[Portal] ──► [Forum]         (application posted as mirror/archive)
[Portal] ──► [OpServ]        (guest sync trigger, sync point 1)
[OpServ] ──► [Bot]           (guest provisioning payload)

--- staff log in to portal ---

[Portal] ──► [Forum API/DB]  (update user groups on approval)
[Portal] ──► [OpServ]        (member sync trigger, sync point 2)
[OpServ] ──► [Bot]           (member provisioning payload)
```

### Variant B — Tradeoffs

| Consideration | Detail |
|---|---|
| Portal complexity | Higher — requires staff authentication, application review UI, approval actions |
| Staff workflow | Unified in one tool (portal handles everything post-submission) |
| Automation | High — approval immediately triggers sync with no manual OpServ step |
| Risk of desync | Low — sync is tied directly to the approval action |
| Implementation effort | Significantly higher |

---

## Variant comparison

| | Variant A | Variant B |
|---|---|---|
| Portal involvement post-submission | None | Staff approval UI |
| Forum role | Primary staff workspace | Mirror / archive |
| OpServ sync trigger (point 2) | Manual — staff trigger in OpServ | Automatic — portal triggers on approval |
| Staff tools required | Forum + OpServ | Portal only |
| Recommended for | Simpler builds, smaller teams, existing OpServ workflows | Teams wanting a unified portal experience |

---

## Service responsibilities

### bwc-onboarding-bot

Identical in both variants. The bot is a thin executor — it owns no business logic and makes no access decisions.

Responsibilities:
- On `on_member_join`: assign quarantine role, send DM with onboarding link
- Maintain persistent "Start BWC Onboarding" button in #welcome channel
- Expose internal webhook listener for OpServ provisioning payloads
- On valid OpServ payload: assign roles, remove roles, set nickname, post staff log embed
- Re-register persistent views on restart

Not responsible for:
- Discord OAuth2
- Application validation or approval
- Forum account management
- Determining roles or nicknames (OpServ decides this)

### bwc-onboarding-portal

**Variant A:** Responsible for OAuth2, forum account creation, application submission, and guest sync trigger only. Connection to the user ends after sync point 1.

**Variant B:** All of the above, plus a staff-facing approval interface. Staff authenticate separately (not via Discord OAuth2). On approval, the portal updates forum groups and triggers OpServ sync point 2.

Not responsible for (either variant):
- Communicating with the bot directly
- Determining roles or nicknames

### Forum

**Variant A:** Primary staff workspace. Staff review and approve applications here. Approval updates user groups in the shared database, which OpServ reads on the next manual sync.

**Variant B:** Mirror and archive. Receives a copy of every application for record-keeping. Staff do not action applications here — the portal is the approval interface.

Both the forum and OpServ run on the same **MariaDB** instance. OpServ reads forum data by querying the shared MariaDB database directly.

### OpServ

Identical in both variants. OpServ is the sole caller of the bot webhook.

Responsibilities:
- Receive sync triggers from the portal (both variants, both sync points in Variant B; sync point 1 only in Variant A)
- In Variant A: expose a manual sync action for staff to trigger sync point 2
- Read user groups, membership state, and profile from the shared forum database
- Resolve correct Discord roles and nickname
- Push fully resolved provisioning payload to the bot

---

## Entry points to the bot

| Surface | Type | Purpose |
|---|---|---|
| Discord gateway | WebSocket (discord.py) | Member join events and button interactions |
| `/webhook/member-provisioned` | HTTP POST (internal only) | Role and nickname provisioning from OpServ |

The webhook endpoint must not be publicly accessible. Restrict to OpServ's IP at the reverse proxy or firewall level.

---

## Quarantine role behaviour

All users receive a quarantine role on `on_member_join`, restricting access to #welcome only. The quarantine role is listed in `remove_roles` in the first OpServ payload (guest provisioning) and removed by the bot at that point.

Users who never complete the portal remain in quarantine indefinitely. No automatic kick occurs.

---

## Onboarding link format

```
https://portal.blackwidowcompany.com/
```

The portal landing page presents a **Sign up with Discord** entry point. No query parameters are needed. The Discord ID, display name, and avatar are obtained from the OAuth2 callback once the user authenticates — not before. Existing forum account detection happens after OAuth2 completes, at which point the identity is verified.

In **Variant B only**, the landing page also presents a **Staff login** option for the approval interface.

---

## Existing bot capabilities (out of scope for rebuild)

The following already exists in the bot codebase and is invoked by the new webhook handler without modification:

- Role assignment by role ID
- Nickname setting

---

## Related documents

- `WEBHOOK_SPEC.md` — OpServ → bot provisioning webhook
- `PORTAL_OPSERV_SPEC.md` — Portal → OpServ sync trigger (both variants)
