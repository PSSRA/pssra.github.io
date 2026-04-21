+++
date = '2026-04-20T16:06:58-07:00'
draft = false
title = 'PSSBOT spec'
tags = ['temp']
+++

# pssbot

A Discord bot providing CRUD operations against a hosted Grist database via slash commands. Deployed to DigitalOcean; API keys for Grist and Discord are stored as DO secrets.

Grist serves as the database backend — Grist API instead of Django. The bot operates as a no-PII Grist editor account to limit data exposure. A higher-privilege account exists for operations that require it but is not used by the bot by default.

Bot-authored Discord messages (event posts, RSVP notifications, etc.) are editable via bot commands where needed — see per-command specs for which posts support in-place editing.

## Reference

- Grist API spec: https://raw.githubusercontent.com/gristlabs/grist-help/master/api/grist.yml
- Grist Python client: https://github.com/gristlabs/py_grist_api
- discord.py docs: https://discordpy.readthedocs.io/en/stable/

**Data model:** `db_code_view.py` in this repo contains the Grist table definitions ("code view"). Treat this as the schema source of truth. It is manually kept in sync with the live database.

**Architecture docs:** `README.notify.md` covers the `notify()` abstraction used across all member-facing notifications.

**Process documentation:** `/docs/` contains org SOPs and process context for builders:
- `gristUsage.md` — secretary workflows: screener intake, national roster ingest, status changes
- `treasurer-records-sop.md` — treasurer workflows: donation recording, LC renewal, arrears, waivers
- `emailTemplates.md` — standard email/DM templates used throughout the intake pipeline

These docs are reference material for understanding business rules behind bot commands. They may be partially out of date — the schema and individual command specs take precedence where there is a conflict.

## Development Environment

**Grist:** A snapshot mirror of the production Grist DB is used for dev. Keep `GRIST_DOC_ID` as an environment variable so switching between prod and dev doc is a config change only — no code changes.

**Discord:** A separate dev server mirrors prod. It has some of the same members but different channel IDs. All channel IDs are stored as environment variables. When dev/prod parity is needed on a specific channel, add or rename the channel in the dev server rather than hardcoding prod IDs anywhere.

**Environment switching:** The full prod/dev split is managed via environment variable sets. No code changes required to switch environments.

## CI/CD

GitHub Actions triggers a redeploy to a DigitalOcean Droplet on push to `main` via SSH. The bot runs as a systemd service. discord.py reconnects automatically on restart, and slash command registration is idempotent on startup — no Discord-side action needed when commands change.

Templates (`/templates/events/`) are committed directly to this private repo. They deploy automatically with every push to `main` — no separate deploy step needed.

Full provisioning and operations runbook: `DEPLOY.md`.

## Architecture Notes

**Member identity:** All `@member` references arrive as Discord snowflake IDs (long integer strings). The canonical person record is `$People` with `$People.person_id` as its UUID. The lookup chain for any incoming snowflake is:

```
Discord snowflake → Discord_users.discord_id → Discord_users.People → $People row
```

This is encapsulated in a shared `resolve_person(snowflake)` utility called at the top of every command handler. See `README.access_control.md` for the full lookup pattern and RBAC implementation.

**Autocomplete:** Discord's native `@mention` picker handles member selection — no custom autocomplete needed for people. Custom autocomplete is implemented for bot-owned enumerated fields: `region`, `structured_event` type, and action subcommands.

**Event context:** Most `/event` and `/rsvp` commands accept an optional `event_id` as their first argument. If omitted, the bot infers the event from the current organizing thread via `$Events.organizer_thread`. This is encapsulated in a shared `resolve_event_id(ctx, event_id)` utility.

**Bot-authored posts:** Event forum posts and RSVP notifications are stored by message ID so the bot can edit them in place. Message IDs are written back to Grist (`$Events.Discord_event`) after creation.

**HTTP server:** The bot runs an HTTP server alongside the Discord gateway to receive inbound webhooks (Grist new-row notifications, Open Collective donation events). Use `aiohttp` or FastAPI — both are compatible with discord.py's async loop.

**Template system:** Event post templates are Markdown files under `/templates/events/`, one per structured event type. Committed directly to this private repo — the repo being private is sufficient access control for confidential location and SOP content. Editable via the GitHub web editor; deploy automatically with every push to `main`. Placeholder syntax: `{{placeholder_name}}`.

**n8n deprecation:** n8n currently handles OC donation and screener notifications posted to `#records-workstream`. The bot replaces all of these. Do not add new n8n automations. Migrate existing ones to the bot as each command is built and confirmed stable. Turn off n8n once `/donation` and `/roster` webhook ingest are both working.

**Email sending:** The bot sends transactional emails (screener confirmation, dues renewal notices, etc.) via Proton SMTP relay (`smtp.protonmail.com:587`, STARTTLS) using an app-specific password. Domain SPF/DKIM/DMARC settings already configured for the domain cover bot-sent mail. Templates live in `/templates/intake/` alongside event templates. Use `aiosmtplib` for async compatibility with discord.py. Confirm SMTP relay is enabled on the Proton plan before building (Settings → All Settings → Email → SMTP submission).

**Documentation site:** Public-facing bot and process documentation is published at `pssbot.pugetsoundsra.org` (or similar subdomain) via MkDocs on GitHub Pages. The existing `README.*.md` spec files are the source — MkDocs renders them into a searchable static site, auto-deployed on push to `main` via GitHub Actions. No auth required for reading. Confidential content (templates, `.env`, internal SOPs) is excluded via `mkdocs.yml` config. `/pssbot help` links to this site. See `mkdocs.yml` in the repo root for configuration.

**PII masking:** Any command that returns potentially sensitive identifying information (emails, etc.) should mask it for Discord output. Use the pattern `p**********l@p***n.me` — show first and last character of each segment, mask the middle with asterisks. This allows identification confirmation without full PII exposure. Apply this to `preferred_email_address`, `initial_email_address`, and `national_member_number` in any bot response.

**`notify()` abstraction:** All member-facing notifications (DMs, future emails) route through a shared `notify(person, message, subject)` utility. Never call `discord.User.send()` directly in command handlers. The abstraction dispatches to Discord DM now, email later (P3), and logs undeliverable attempts. See `README.notify.md` for the full interface and dispatch logic. This must be implemented before any command that sends a DM.

**Discord role sync:** An open question is whether the bot should be responsible for keeping Discord roles (Organizer, Representative, Central Committee, region roles) in sync with `$Officer_roles` and `$Region_roles` in Grist. This would make the bot the authoritative sync mechanism rather than manual updates. Not yet specced — flag as a design decision before building `/member` or `/onboard` commands that depend on role accuracy.

## Access Control

RBAC is enforced via Discord roles, not Grist officer records. Three tiers:

| Tier | Discord Role |
|---|---|
| Central Committee | Full access |
| Representative | Standard admin |
| Organizer | Operational |

Full spec: `README.access_control.md`

## Command Index

Commands are listed in implementation priority order — highest frequency of use first.

---

### 1. `/event` — Event management
`README.events.md`

```
/event new [region] [date] [structured_event]
/event [optional:event_id] role [add|remove] [instructor|TA|greeter|medic] [@member ...]
/event [optional:event_id] attendance [@member ...] [optional:guests]
```

Creates and manages events in `$Events` and their corresponding forum posts in `#range-days-and-events`. `/event new` checks for duplicate events (±2 days), selects a template by event type, creates the Grist row and the Discord forum post atomically, and writes both URLs back to Grist. Role and attendance updates edit the live forum post in place.

Access: Organizer+. Prefer running from inside the event organizing thread in `#firearms-instruction` to auto-link `$Events.organizer_thread`.

---

### 2. `/rsvp` — RSVP submission and management
`README.rsvp.md`

**Member-facing:**
```
/rsvp [optional:event_id]              — submit RSVP for yourself
/rsvp withdraw [optional:event_id]     — cancel your RSVP
```

**Organizer-facing:**
```
/rsvp [optional:event_id] list
/rsvp [optional:event_id] accept [n ...]
/rsvp [optional:event_id] waitlist [n ...]
/rsvp [optional:event_id] info @member
```

Members RSVP via slash command (not a form), so `validated_identity` and `validated_event` are populated at submission time with no manual linking required. RSVP submission opens a Discord modal for gear needs and notes. New RSVPs post a notification to the organizing thread with pre-computed context (last taken, prereqs, months since vetted).

Organizer actions (accept/waitlist) update `$Event_RSVPs.status_from_organizer` and edit the live event forum post. The `list` action returns a numbered embed with footnote legend matching the existing Grist display (`^` grass toucher, `º` repeat attendee, `~` joined <2mo, `+` gear needed).

Access: any vetted member for self-service; Organizer+ for admin actions.

**P3 note:** Non-Discord member path (email/form-based RSVP) is architecturally planned but not implemented. See `README.rsvp.md` for the `notify()` abstraction stub required from day one.

---

### 3. `/member` — Member event and ToG history
`README.member.md`

```
/member events [optional:role] @member
/member tog [add|remove|info] @member
```

`/member events` returns a read-only summary of a member's event history by structured event type and role (attendee / instructor / TA), pulling from pre-computed `$People` fields — no bot-side calculation. `/member tog` reads and manages Toucher of Grass status via `$People.is_ToG` and `$People.tog_dates_attended_instructed_etc`.

Note: there is a separate `Touchers_of_grass` sync table (distinct from the formula fields on `$People`) with a `to_deToG` flag for periodic cleanup. The `/member tog add` command should write to event attendance as specced in `README.member.md`, but the builder should confirm whether `Touchers_of_grass` also needs a corresponding write or whether it is managed separately.

Access: Organizer+ for `/member events`; Representative+ for `/member tog`.

---

### 3b. `/member me` — Self-service member info

```
/member me info
/member me edit [field] [value]
```

Self-service command for any vetted member to view and edit their own non-sensitive profile fields. Caller is always resolved from their own Discord snowflake — no `@member` param.

`info` returns their own record: preferred email (masked), common handle, region roles, dues status, ToG status.

`edit` allows updating a limited whitelist of self-editable fields — at minimum `preferred_email_address` and `common_handle`. All edits are PATCHed to `$People` directly. PII returned in `info` is masked per the masking pattern in Architecture Notes.

Access: any vetted member.

Not yet fully specced — add to `README.member.md` when implementing.

---

### 4. `/roster` — Screener intake and roster management
`README.roster.md`

```
/roster screener list
/roster screener process [id]
/roster screener link [id] @member
/roster status add @member [status]
/roster note @member [text]
```

Secretary-facing commands replacing the manual Grist button workflow. New screener submissions fire a Grist webhook → bot posts to `#records-workstream` with masked email, flags for prohibitions/veteran status, and referral info. `/roster screener process` replicates the two-step Grist button sequence (create `$People` row + link screener + create `"join backlog"` status, backdated to submission time) in a single command, and **sends the screener-received confirmation email automatically** via Proton SMTP. Referral confirmation is sent as a bot DM to the referrer.

When members join the unvetted Discord, the bot DMs them to self-identify and auto-links their `Discord_users` record to `$People`, creating `"unvetted (ready for vetting)"` status — replacing the manual Grist Discord linking workflow. Ambiguous matches escalate to `#records-committee` with `/roster discord link` for human resolution.

`/roster status add` covers all status transitions throughout the intake pipeline via Discord. Status history is append-only.

Access: Representative+. Run from `#records-workstream` and `#records-committee`.

---

### 5. `/onboard` — Onboarding buddy assignment
`README.onboard.md`

```
/onboard @onboardee @onboarder
/onboard availability list [optional:region]
```

Assigns an onboarding buddy, checking capacity before writing (`$People.remaining_capacity`). Returns pre-formatted copy-paste output from `$People.onboarding_pair_c_p`. The availability list surfaces members with open slots, optionally filtered by region. When filtering by region, use `Region_onboard_capacity.region_rem_capacity` for the regional total rather than summing across people.

Access: Representative+.

---

### 6. `/vetting` — Vetting session management
`README.vetting.md`

```
/vetting session new [date] [time] vettees: @member ... panel: @member ...
/vetting session timeful
```

Creates `Vetting_sessions` rows and surfaces scheduling assistance for finding panel availability. Vetting notes capture in Discord is explicitly deferred — `Vetting_session_notes` contains heavily PII-laden fields (ideology, activism history, support system, etc.) that are incompatible with the no-PII bot user.

`Vetting_sessions` schema is confirmed: `vettees` and `vetting_panel` are both `ReferenceList("People")` — the bot writes to these directly using integer row ids, same pattern as all other ReferenceLists. `vettees_person_id` is a read-only formula field, not a write target.

Access: Representative+.

---

### 7. `/pssbot` — Bot meta commands
`README.pssbot.md`

```
/pssbot help
/pssbot [command] help
/pssbot feedback
```

`help` returns a permission-filtered command index. `[command] help` returns detailed usage for a command family. `feedback` opens a modal writing to a new `Bot_feedback` Grist table, with a notification posted to an internal channel.

All commands open to any server member.

**Builder note:** Implement a `COMMANDS` registry dict `{name: {description, required_tier, subcommands}}` that drives both the `@requires_tier` decorator and the help output. Adding a new command should update help automatically.

---

### 8. `/donation` — Donation ingest and linking
`README.donation.md`

```
/donation list [optional:unmatched]
/donation link [id] @member
/donation info [id]
```

Primarily webhook-driven: Open Collective fires a `transaction.created` webhook → bot creates a `Local_donations` row in Grist → bot posts a notification to the donations channel for organizer review. Slash commands handle manual linking of donations to `$People` records.

On ingest, bot writes: `donation_name`, `trans_date`, `amount`, `transaction_ID`, `donation_type`, `effective_date` (same as trans_date initially), `duration_days_` (default 365), and `person` if auto-matched. The `contribution_ID` formula field is a stub — write to `transaction_ID` instead.

One open question remains: whether the no-PII bot user can read `preferred_email_address` and `initial_email_address` for auto-matching donations to members. If not, name-only fuzzy matching is the fallback. See `README.donation.md`.

Access: Representative+ for all commands.

---

## Schema Status

All core tables reviewed via `db_code_view.py`. Resolved items from earlier open questions:

- `Vetting_sessions` — confirmed: `vettees` and `vetting_panel` are `ReferenceList("People")`. Write target is `vettees`, not `vettees_person_id` (that's a read-only formula).
- `Status_actions.status` — vetted statuses for RSVP self-service gate confirmed as: `"vetted"`, `"vetted (left discord)"`. Full status ladder: `"join backlog"` → `"waiting for donation"` → `"sent vetting invite"` → `"unvetted (ready for vetting)"` → `"vetted"` (or rejection/separation variants).
- `Officer_roles` — role field is free text; RBAC does not use this table directly. Bot reads Discord roles from the guild member object instead. See `README.access_control.md`.
- Region strings confirmed from schema: `North`, `South`, `East`, `Central`, `Peninsula`.
- `Vetting_session_notes` — confirmed PII-laden (ideology, activism, support system fields). Vetting notes wizard remains deferred indefinitely.
- `Touchers_of_grass` — a separate sync table with a `to_deToG` cleanup flag, distinct from the `is_ToG` formula on `$People`. Builder should confirm write behavior for `/member tog add` against this table.

All tables required for specced commands are now confirmed in `db_code_view.py`.

**`Local_donations` confirmed schema:**

| Field | Type | Notes |
|---|---|---|
| `donation_id` | Text (UUID) | Auto-generated |
| `donation_type` | Choice | "open collective one time", "open collective recurring", "cashapp", "waiver" |
| `donation_name` | Text | Name as submitted with donation |
| `trans_date` | DateTime | Defaults to NOW() |
| `transaction_ID` | Text | OC transaction ID |
| `amount` | Numeric | Defaults to 25 |
| `effective_date` | Date | May differ from trans_date — organizer corrects if needed |
| `person` | Reference("People") | Null until linked |
| `notes` | Text | |
| `refunded` | Bool | |
| `duration_days_` | Numeric | Defaults to 365 — drives `expires` formula |
| `expires` | Formula (Date) | `DATEADD(effective_date, days=duration_days_)` |
| `contribution_ID` | Formula (Text) | Stub — empty string, likely for OC integration |

The `amount` field exists and defaults to 25 — resolves the open question in `README.donation.md`. The `contribution_ID` formula stub suggests OC transaction ID linkage was planned but not yet implemented; `transaction_ID` (Text) is the right write target for the webhook ingest.

## Logging and Alerting

**Application logging:** The bot uses Python's standard `logging` module, configured at startup to write structured log lines to stdout. systemd captures stdout and makes it available via journalctl — no separate log file needed.

Log levels used consistently across the codebase:
- `INFO` — command invocations, Grist reads/writes, webhook ingest events
- `WARNING` — expected failure states (member not found, duplicate event, permission denied)
- `ERROR` — unexpected failures, Grist API errors, Discord API errors
- `CRITICAL` — startup failures, unrecoverable states

Every command log line should include the caller's Discord snowflake and the command name at minimum, so failures are traceable to a user and action without PII.

**Viewing logs on the Droplet:**
```bash
journalctl -u pssbot -f              # live tail
journalctl -u pssbot -n 100          # last 100 lines
journalctl -u pssbot --since "1 hour ago"
journalctl -u pssbot --since today
```

Configure journald retention in `/etc/systemd/journald.conf` — set `MaxRetentionSec=30d` and `SystemMaxUse=500M` to keep 30 days without unbounded disk growth.

**Discord alerting:** The bot posts to a `#bot-alerts` channel on any unhandled exception. Implement as a top-level error handler in discord.py that catches anything not handled by a command's own error handling, formats the exception and truncated stack trace into an embed, and posts it. Add `BOT_ALERTS_CHANNEL_ID` to the env var list.

This means errors surface in Discord without requiring SSH — the primary alerting mechanism for on-call purposes.

```
🚨 Unhandled error
  Command: /event new
  User: 123456789
  Error: GristAPIError: 403 Forbidden on PATCH /tables/Events/records
  [truncated traceback]
```

**Future:** If log search or retention beyond journalctl becomes necessary, Grafana Cloud's free tier (50GB/month, 14-day retention) with a `promtail` agent on the Droplet is the lowest-cost upgrade path. Not needed to start.

Add `BOT_ALERTS_CHANNEL_ID` to `.env.example` and `DEPLOY.md` env var table.

## Parking Lot

Decisions and questions explicitly deferred — do not investigate until flagged for prioritization.

**Proton Drive for templates:** Templates could be read from Proton Drive at startup instead of the repo filesystem, allowing non-technical members to update them without GitHub access. Not needed for initial build — filesystem templates work fine. Revisit if template editing becomes a pain point for non-technical contributors.

**Is Grist overkill? Should this be a Django app?** This is a significant architectural question that would require rebuilding the existing database, form infrastructure, reporting, and formula logic. The current Grist-as-backend approach is the explicit starting point. Revisit only if Grist API limitations become a recurring blocker in practice.

**Future command: `/treasurer renewals`** — surfaces members due or overdue for LC renewal, replacing the manual Grist dashboard (orange/red dues dates) and records ticket creation. Would query `$People` for `dues_expiration` within 30 days or past due, filtered to vetted members. High value for the treasurer role but not needed for initial launch.

**Future command: `/treasurer arrears [@member]`** — when dues lapse after follow-up: removes vetted Discord role, creates `Status_actions` row with `"arrears (local)"`, posts to `#moderator-action-log`. Currently a multi-step manual process. Spec when `/donation` is stable.

**Email automation via n8n:** Proton Mail does not support outbound webhooks natively. The viable path is n8n IMAP trigger polling join@ on a schedule (every 5-10 minutes) → parse → Grist write or bot action. Most secretary/treasurer email workflows are better replaced by bot commands (which notify in Discord) than by email parsing (which is brittle). The OC donation email notification is already redundant once the OC → bot webhook is reliable. Do not design new workflows around email parsing — design around the bot commands that replace the email steps instead. Revisit if a specific high-value parsing case emerges.

## Environment Variables

All runtime secrets live in a `.env` file on the Droplet — never committed. CI only holds the Droplet IP and SSH deploy key. See `DEPLOY.md` for the full variable list, `.env.example` template, and prod/dev split.

| Variable | Description |
|---|---|
| `DISCORD_TOKEN` | Bot token — separate application for prod and dev |
| `DISCORD_GUILD_ID` | Guild ID for slash command registration |
| `GRIST_API_KEY` | No-PII editor account API key |
| `GRIST_DOC_ID` | Doc ID — prod doc vs dev snapshot |
| `GRIST_BASE_URL` | e.g. `https://pssra.getgrist.com` |
| `OC_WEBHOOK_SECRET` | HMAC secret for OC webhook verification |
| `EVENTS_FORUM_CHANNEL_ID` | Prod: `1080200301501497354` |
| `DONATIONS_CHANNEL_ID` | Donation notifications |
| `FEEDBACK_CHANNEL_ID` | Bot feedback notifications |
| `FALLBACK_CHANNEL_ID` | Unmatched RSVP / unlinked event notifications |
| `FIREARMS_INSTRUCTION_CHANNEL_ID` | Organizing thread parent channel |
| `BOT_ALERTS_CHANNEL_ID` | Unhandled exception alerts |
| `RECORDS_WORKSTREAM_CHANNEL_ID` | Secretary/treasurer command channel; current OC webhook capture |
| `RECORDS_COMMITTEE_CHANNEL_ID` | Secretary/treasurer/vetting ops — primary command channel for records committee |
| `UNVETTED_GUILD_ID` | Unvetted Discord server ID — bot listens for member join events |
| `PROTON_SMTP_USER` | Proton account username for SMTP relay |
| `PROTON_SMTP_PASSWORD` | App-specific password for Proton SMTP relay |
