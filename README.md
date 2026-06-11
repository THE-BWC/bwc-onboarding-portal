# BWC Onboarding Portal

The onboarding portal for Black Widow Company. Handles Discord OAuth2, forum account creation, game-specific application submission, and applicant communication. In Variant B, also serves as the staff application review interface.

---

## Overview

New Discord members are directed to this portal to apply for BWC membership. The portal authenticates them via Discord OAuth2, creates or links their XenForo forum account, walks them through a game-specific application form, and notifies them of their application status throughout the review process.

Role assignment and Discord nickname changes are handled automatically by the BWC onboarding bot once a decision is made — the portal never communicates with Discord directly.

---

## Related repositories

| Repo | Description |
|---|---|
| [`bwc-onboarding-bot`](https://github.com/THE-BWC/bwc-onboarding-bot) | Discord bot — handles role assignment, nickname setting, and DM notifications |
| `opserv` | Central authority for role and permission resolution |

---

## Tech stack

| Concern | Technology |
|---|---|
| Framework | TanStack Start |
| Routing | TanStack Router (file-based) |
| Forms | TanStack Form |
| Database | Drizzle ORM + MariaDB |
| Auth | Discord OAuth2 |
| Sessions | JWT |
| Email | SMTP (internal BWC mail server) |
| Styling | Tailwind CSS |

---

## Approval flow variants

This portal supports two approval flow variants. The active variant is set via the `PORTAL_VARIANT` environment variable.

| | Variant A | Variant B |
|---|---|---|
| Post-submission workflow | Forum + OpServ (manual) | Portal staff interface |
| Forum role | Primary staff workspace | Mirror / archive |
| Staff sync trigger | Manual in OpServ | Automatic on portal approval |
| Staff login | Not present | Discord OAuth2 + XenForo group check |

See [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) for the full comparison.

---

## Repository structure

```
bwc-onboarding-portal/
├── docs/
│   ├── ARCHITECTURE.md          # Full system architecture — start here
│   ├── PORTAL_SPEC.md           # Portal-specific spec (auth, forms, state machine, notifications)
│   ├── WEBHOOK_SPEC.md          # Bot webhook contracts (provisioning + DM)
│   └── PORTAL_OPSERV_SPEC.md   # Portal → OpServ sync trigger spec
├── wireframes/
│   ├── portal_landing_page      # Landing page / login
│   ├── portal_application_form  # Application form (game selection + questions)
│   └── portal_staff_review      # Staff review queue and application detail (Variant B)
└── README.md
```

---

## Documentation

| Document | Description |
|---|---|
| [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | System-wide architecture, service responsibilities, full onboarding flows for both variants, communication map |
| [`docs/PORTAL_SPEC.md`](docs/PORTAL_SPEC.md) | Portal tech stack, auth flows, database schema, application lifecycle, state machine, comment system, notifications, forum posting |
| [`docs/WEBHOOK_SPEC.md`](docs/WEBHOOK_SPEC.md) | OpServ → bot member provisioning webhook; portal → bot DM webhook |
| [`docs/PORTAL_OPSERV_SPEC.md`](docs/PORTAL_OPSERV_SPEC.md) | Portal → OpServ sync trigger for guest provisioning and member promotion |

New to the project? Start with [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md).

---

## Application states

### Applicant-visible

| State | Meaning |
|---|---|
| `pending` | Application submitted and under review |
| `on_hold` | Further information required from the applicant |
| `approved` | Application accepted — Discord access granted |
| `rejected` | Application not accepted |

### Internal (staff only)

| State | Meaning |
|---|---|
| `pending` | Submitted, not yet actioned |
| `s2_review` | Referred to game S-2 division (OPSEC games only) |
| `recruiter_review` | S-2 cleared or skipped — with recruiter |
| `on_hold` | Awaiting applicant response |
| `approved` | Accepted |
| `rejected` | Not accepted |

S-2 review is inserted automatically for games with the `opsec` flag set in `opserv_games`. Games without an S-2 division go directly to `recruiter_review`.

---

## Supported games

Populated dynamically from the `opserv_games` table. Retired games (`retired = 1`) are excluded from the application form. Current active divisions:

- Dune: Awakening
- MechWarrior Online
- Star Citizen
- Vanguard

---

## Environment variables

Copy `.env.example` to `.env` and fill in the values before running.

| Variable | Description |
|---|---|
| `PORTAL_VARIANT` | `A` or `B` — determines approval flow and whether staff login is enabled |
| `DISCORD_CLIENT_ID` | Discord OAuth2 application client ID |
| `DISCORD_CLIENT_SECRET` | Discord OAuth2 application client secret |
| `DISCORD_REDIRECT_URI` | OAuth2 callback URL |
| `JWT_SECRET` | Secret for signing portal session tokens |
| `MARIADB_HOST` | Shared MariaDB host |
| `MARIADB_PORT` | Default `3306` |
| `MARIADB_USER` | Portal DB user |
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
| `STAFF_PORTAL_GROUP_IDS` | Comma-separated XenForo group IDs with portal access (Variant B — TBD internally) |

---

## Notification flow

The portal sends notifications to applicants on the following events:

- Application received
- Application placed on hold (further information required)
- Staff comment posted on the application
- Application approved
- Application rejected

Discord DM is attempted first via the bot. If the bot returns a non-200 response (DMs disabled, user not found, etc.), the portal falls back to email via SMTP.

---

## Contributing

This project is internal to Black Widow Company. For questions, reach out via the BWC staff channels or open an issue in this repository.
