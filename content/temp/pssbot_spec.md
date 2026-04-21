+++
date = '2026-04-21T16:06:58-07:00'
draft = false
title = 'pssbot Spec'
tags = ['temp pages']
+++

# pssbot

Discord bot for PSSRA — slash commands that read/write the Grist roster database. Deployed on a DigitalOcean droplet. Grist handles data storage and formula logic; the bot is purely an interface layer on top of the Grist API.

The bot runs as a no-PII Grist editor account. This limits data exposure but means some fields (emails, national member numbers) are either unavailable or require masking before display. A higher-privilege account exists for operations that need it.

Bot-authored Discord messages (event posts, RSVP notifications) are editable via slash commands after posting.

## Specs

Each command family has its own spec file:

| Spec | Commands |
|---|---|
| `README.events.md` | `/event new`, `/event role`, `/event attendance` |
| `README.rsvp.md` | `/rsvp`, `/rsvp withdraw`, `/rsvp list/accept/waitlist/info` |
| `README.member.md` | `/member events`, `/member tog`, `/member me` |
| `README.roster.md` | `/roster screener`, `/roster status`, `/roster discord link` |
| `README.onboard.md` | `/onboard` |
| `README.vetting.md` | `/vetting session` |
| `README.donation.md` | `/donation` |
| `README.pssbot.md` | `/pssbot help`, `/pssbot feedback` |
| `README.access_control.md` | RBAC, permission tiers, channel routing |
| `README.notify.md` | `notify()` for member-facing notifications |

## Reference

- Grist API: https://raw.githubusercontent.com/gristlabs/grist-help/master/api/grist.yml
- Grist Python client: https://github.com/gristlabs/py_grist_api
- discord.py: https://discordpy.readthedocs.io/en/stable/

`db_code_view.py` is the Grist schema in code-view format — treat it as the data model source of truth. Manually kept in sync with the live database.

`/docs/` has org SOPs for context:
- `gristUsage.md` — secretary workflows
- `treasurer-records-sop.md` — treasurer workflows  
- `emailTemplates.md` — standard email/DM templates

These may be partially out of date. The schema and command specs take precedence.

## Dev Setup

**Grist:** dev uses a snapshot of the prod doc. `GRIST_DOC_ID` is an env var — switching environments is a config change only.

**Discord:** separate dev server with different channel IDs. All channel IDs are env vars. When you need a channel in dev that doesn't exist yet, add it there rather than hardcoding prod IDs anywhere.

See `DEPLOY.md` for provisioning, CI/CD, and the full env var list.

## Architecture

### Member identity

`@member` references arrive as Discord snowflake IDs. The lookup chain to the canonical person record:

```
Discord snowflake -> Discord_users.discord_id -> Discord_users.People -> $People row
```

Wrapped in `resolve_person(snowflake)` — called at the top of every command handler that takes a member argument.

### Event context

Most `/event` and `/rsvp` commands take an optional `event_id` first argument. If omitted, the event is inferred from `$Events.organizer_thread` — so running commands from inside an organizing thread in `#firearms-instruction-workstream` is the normal workflow. Wrapped in `resolve_event_id(ctx, event_id)`.

### notify()

All member-facing notifications go through `notify(person, message, subject)`. Do not call `discord.User.send()` directly in command handlers. Currently dispatches to Discord DM; email path (P3) is a stub. See `README.notify.md`.

`GUILD_MEMBERS` privileged intent is required for DMs — enable in the Discord Developer Portal before first deploy.

### Webhooks

The bot runs an `aiohttp` HTTP server alongside the Discord gateway to receive Grist new-row webhooks (screener submissions) and Open Collective `transaction.created` webhooks.

### Templates

Event post templates live in `/templates/events/`, one per structured event type. Intake email templates in `/templates/intake/`. Placeholder syntax: `{{placeholder_name}}`. All committed to this private repo — editable via GitHub web editor, deployed on every push to main.

### PII masking

Commands that return identifying info mask it for Discord output: `p**********l@p***n.me` — first and last character of each segment, middle asterisked. Applies to `preferred_email_address`, `initial_email_address`, `national_member_number`. `#records-workstream` is NDA/PII-ok and can receive unmasked output.

### Email

Outgoing mail via Proton SMTP relay (`smtp.protonmail.com:587`, STARTTLS). `aiosmtplib` for async compatibility. Domain SPF/DKIM/DMARC already configured — bot-sent mail from the org domain works without additional DNS changes. Confirm SMTP relay is enabled on the Proton plan before building (Settings -> All Settings -> Email -> SMTP submission).

### n8n

n8n currently posts OC donation and screener notifications to `#records-workstream`. The bot replaces all of these. Don't add new n8n automations — retire existing ones as each bot command goes stable. Can be shut off once `/donation` and `/roster` webhook ingest are confirmed working.

## Access Control

RBAC uses Discord roles, not Grist officer records. Three tiers — membership is inclusive upward:

| Tier | Discord Role |
|---|---|
| CCC | Central Committee |
| Representative | Representative Committee |
| Organizer | Organizing Committee |

Vetted-member-only commands (self-service `/rsvp`, `/member me`) use a separate Grist lookup against `$People.most_recent_status`.

Full spec and channel routing: `README.access_control.md`

## Command Index

Ordered by implementation priority.

---

### 1. `/event`
`README.events.md`

```
/event new [region] [date] [structured_event]
/event [event_id] role [add|remove] [instructor|TA|greeter|medic] [@member ...]
/event [event_id] attendance [@member ...] [guests: "..."]
```

Creates and manages events in `$Events` and posts in `#range-days-and-events`. `/event new` checks for duplicate events (±2 days), picks a template by event type, creates the Grist row and forum post together, and writes both URLs back to Grist. Role and attendance updates edit the live post in place. Run from an organizing thread in `#firearms-instruction-workstream` to auto-link the thread to the event record.

---

### 2. `/rsvp`
`README.rsvp.md`

Members RSVP via slash command rather than a form, so `validated_identity` and `validated_event` are set at submission time — no manual linking. Submission opens a modal for gear needs and notes. New RSVPs post to the organizing thread with context from pre-computed Grist fields (last taken, prereqs, months since vetted).

Organizer actions update `$Event_RSVPs.status_from_organizer` and edit the live event post. Status values: `interest noted` (default on submission), `accepted`, `waitlisted`, `dropped`, `ignored`.

```
/rsvp [event_id]                         member submits
/rsvp withdraw [event_id]                member cancels
/rsvp [event_id] list
/rsvp [event_id] accept [n ...]
/rsvp [event_id] waitlist [n ...]
/rsvp [event_id] info @member
```

Self-service: any vetted member. Organizer actions: Organizing Committee+.

---

### 3. `/member`
`README.member.md`

```
/member events [role] @member
/member tog [add|remove|info] @member
/member me [info|edit]
```

`/member events` — read-only event history by type and role, from pre-computed `$People` fields. `/member tog` — view and manage Toucher of Grass status. `/member me` — self-service profile view/edit (preferred email, handle); PII masked in output.

Note: `Touchers_of_grass` is a separate sync table from the `is_ToG` formula on `$People` — builder should confirm whether `/member tog add` needs to write there in addition to event attendance.

---

### 4. `/roster`
`README.roster.md`

```
/roster screener list
/roster screener process [id]
/roster screener link [id] @member
/roster status add @member [status]
/roster discord link @member [people_id]
/roster note @member [text]
```

Secretary commands replacing the manual Grist button workflow. New screener submissions trigger a Grist webhook -> bot posts to `#records-workstream` with masked email and any flags (prohibitions, veteran status). `/roster screener process` replicates the two Grist action buttons in one command and sends the confirmation email via Proton SMTP automatically.

When members join the unvetted Discord, the bot DMs them to self-identify and auto-links their `Discord_users` record to `$People`. Ambiguous matches go to `#records-workstream` for human resolution via `/roster discord link`.

Status history is append-only. Run from `#records-workstream`.

---

### 5. `/onboard`
`README.onboard.md`

```
/onboard @onboardee @onboarder
/onboard availability list [region]
```

Assigns onboarding buddies with a capacity check (`$People.remaining_capacity`). Availability list uses `Region_onboard_capacity.region_rem_capacity` for regional totals. Returns `$People.onboarding_pair_c_p` for copy-paste.

---

### 6. `/vetting`
`README.vetting.md`

```
/vetting session new [date] [time] vettees: @member ... panel: @member ...
/vetting session timeful
```

Creates `Vetting_sessions` rows. `vettees` and `vetting_panel` are both `ReferenceList("People")` — write integer row ids directly. `vettees_person_id` is a read-only formula field, not a write target. Vetting notes capture in Discord is deferred — `Vetting_session_notes` is heavily PII-laden and incompatible with the no-PII bot user.

---

### 7. `/pssbot`
`README.pssbot.md`

```
/pssbot help
/pssbot [command] help
/pssbot feedback
```

`help` returns a permission-filtered command list with a link to the public docs site. `feedback` writes to a new `Bot_feedback` Grist table. The `COMMANDS` registry dict drives both the permission decorator and help output — adding a command updates help automatically.

---

### 8. `/donation`
`README.donation.md`

```
/donation list [unmatched]
/donation link [id] @member
/donation info [id]
```

OC fires a `transaction.created` webhook on every transaction (including recurring charges). Bot creates a `Local_donations` row and posts to `#records-workstream`. Effective date logic is non-trivial — see `README.donation.md`. Slash commands handle manual member linking.

---

## Schema Notes

All tables required for specced commands are confirmed in `db_code_view.py`. Key facts for the builder:

- `Vetting_sessions.vettees` is `ReferenceList("People")` — write target is `vettees`, not `vettees_person_id`
- Vetted statuses for the self-service gate: `"vetted"`, `"vetted (left discord)"`
- Status ladder: `join backlog` -> `waiting for donation` -> `sent vetting invite` -> `unvetted (ready for vetting)` -> `vetted`
- Region strings: `North`, `South`, `East`, `Central`, `Peninsula`
- `Vetting_session_notes` is heavily PII-laden — no bot access, full stop
- `Touchers_of_grass` is a sync table separate from `$People.is_ToG` — confirm write behavior for `/member tog add`
- `Local_donations.effective_date` is computed on ingest, not just copied from `trans_date` — see `README.donation.md`

## Logging

Python `logging` module to stdout; systemd/journalctl captures it. Log levels:
- `INFO` — command invocations, Grist reads/writes, webhook events
- `WARNING` — expected failures (member not found, permission denied)
- `ERROR` — Grist/Discord API errors
- `CRITICAL` — startup failures

Every log line should include the caller's Discord snowflake and command name. No PII in logs.

Unhandled exceptions post to `#bot-stuff` via a top-level discord.py error handler. See `README.access_control.md` for channel details.

## Deferred / Parking Lot

- **`/treasurer renewals`** — surfaces members due for LC renewal, replacing the manual Grist dashboard. Spec after `/donation` is stable.
- **`/treasurer arrears [@member]`** — removes vetted role, creates arrears status, logs to `#moderator-action-log`. Spec after `/donation` is stable.
- **Discord role sync** — whether the bot should own syncing Discord roles to `$Officer_roles`/`$Region_roles`. Decision needed before `/member` or `/onboard` build.
- **Proton Drive for templates** — lets non-technical members edit templates without GitHub. Not needed initially.
- **Email parsing via n8n** — Proton doesn't support outbound webhooks; IMAP polling is possible but brittle. Better to replace email workflows with bot commands than to parse replies. Revisit only if a specific high-value case comes up.
- **Non-Django architecture** — Grist as backend is the starting point. Revisit only if API limitations become a recurring blocker.

## Environment Variables

All secrets in `.env` on the droplet — never committed. See `.env.example` and `DEPLOY.md`.

| Variable | Notes |
|---|---|
| `DISCORD_TOKEN` | Separate bot application for prod and dev |
| `DISCORD_GUILD_ID` | Main server |
| `UNVETTED_GUILD_ID` | Unvetted Discord — bot listens for member joins |
| `GRIST_API_KEY` | No-PII editor account |
| `GRIST_DOC_ID` | Prod doc vs dev snapshot |
| `GRIST_BASE_URL` | `https://pssra.getgrist.com` |
| `OC_WEBHOOK_SECRET` | HMAC verification for OC webhooks |
| `PROTON_SMTP_USER` | SMTP relay username |
| `PROTON_SMTP_PASSWORD` | App-specific password |
| `EVENTS_FORUM_CHANNEL_ID` | `#range-days-and-events` |
| `FIREARMS_INSTRUCTION_WORKSTREAM_CHANNEL_ID` | `#firearms-instruction-workstream` |
| `RECORDS_WORKSTREAM_CHANNEL_ID` | `#records-workstream` (NDA/PII-ok) |
| `ONBOARDING_WORKSTREAM_CHANNEL_ID` | `#onboarding-workstream` |
| `MODERATOR_ACTION_LOG_CHANNEL_ID` | `#moderator-action-log` |
| `BOT_ALERTS_CHANNEL_ID` | `#bot-stuff` |
| `DOCS_URL` | `https://pssbot.pugetsoundsra.org` |
