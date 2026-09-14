# Family Dashboard — build brief

**How to use this doc:** paste this whole file into a fresh Claude Code (or
Claude Desktop/web) session as your starting prompt, then tell Claude about
*your* household — family members, what calendars/smart displays you
actually use, which optional modules you want. This describes a working
touchscreen family dashboard as a reference design, not source to fork:
build your own from scratch, with your own credentials, your own domain, and
your own family's names. Don't reuse anyone else's API keys, service account,
or tokens.

## The pitch

A wall/desk-mounted touchscreen (portrait orientation) that a family checks
throughout the day: everyone's calendars merged into one view, a chores
board, a shared running list of notes and to-dos that anyone in the family
can add to just by talking to Claude from their phone or laptop — not by
opening the dashboard — and a quick health-of-the-server widget since it runs
on a home server anyway. The standout idea is the **shared "brain"**: notes
and tasks aren't only editable through the touchscreen UI, they're also
exposed as an MCP (Model Context Protocol) server, so "remember the plumber's
coming Friday" or "add a task to renew the car registration," said to any
Claude surface, lands on the dashboard.

## Suggested tech stack

What worked well here — treat as a reasonable default, not a requirement:

- **Backend**: Node.js + Express, one process, no framework beyond that.
- **Frontend**: plain HTML/CSS/JS, no build step, no framework — kept the
  iteration loop fast for a project this size, and a kiosk browser doesn't
  need SPA routing.
- **Storage**: SQLite via `better-sqlite3` — a single file, zero setup,
  trivially backed up, and this app's write volume never remotely
  approaches a case for anything heavier.
- **Deployment**: one Docker container, `.env` for secrets/config, reverse
  proxied (e.g. Nginx Proxy Manager, Caddy, Traefik — whatever you already
  run) for HTTPS and remote/mobile access outside the LAN.

## Design / layout concepts

- **Portrait-first**: the primary display is a touchscreen turned sideways.
  Design the layout to read well tall-and-narrow first, then make sure it
  degrades sensibly to a normal landscape browser window (phone/laptop
  access) rather than the other way around.
- **Card-based sections**: each concern (calendar, chores, notes, tasks,
  status) is its own visually distinct card/panel on a page, not one long
  undifferentiated feed — makes it scannable from a few feet away.
- **Large tap targets**: this is a finger-operated kiosk, not a mouse-driven
  admin panel — buttons, checkboxes, and list rows should be sized for touch,
  with generous spacing so mis-taps are rare.
- **Per-profile accent color**: give each family member/profile a color
  (picked at setup, not hardcoded) and use it consistently as an avatar/badge
  accent wherever their name appears — calendar events, task ownership, chore
  assignment — so people can recognize "their" items at a glance without
  reading text.
- **A fixed nav for switching context**: a persistent small control (avatar
  row or tab bar) to switch between family-member profiles and between
  pages/modules, always visible rather than buried in a menu, since the
  primary interaction is a quick glance-and-tap, not a session.
- **Kiosk mode**: `chromium --kiosk --app=<url>` on a small fanless PC (an
  Intel N100-class mini PC is plenty) pointed at your dashboard's URL, with
  the display rotated to portrait in the OS settings.

## Core features (recommend every install has these)

1. **Merged calendar view** — pull events from Google Calendar (and
   optionally a smart-display service if you have one) into a single
   timeline, tagged by whose calendar each event came from so it can be
   filtered per profile or shown for everyone at once.
2. **Shared notes/tasks "brain," writable via MCP** — a small MCP server
   (see below) so *any* Claude conversation, not just the dashboard UI, can
   add a note or task. This is the feature worth prioritizing first: it's
   what makes the dashboard feel alive rather than like one more app to
   remember to open.
3. **Task lists with reordering, not just checkboxes** — beyond marking a
   task done, give it a numeric priority/sort order the user can nudge **up
   or down** (not only "push to the bottom") so the most-urgent-today item
   can float to the top with one tap. Implement as a `priority` integer
   column, `ORDER BY priority`, and a small delta update
   (`priority = priority + delta`) driven by up/down buttons on each row —
   simple, no drag-and-drop library needed, works great on touch.
4. **Chores board**, if you have a chore-tracking service or want to build
   your own simple version — who's assigned what, done/not-done.
5. **Server/container health widget** — since this runs on a home server
   anyway, a small "is everything up, how's disk space" panel is cheap to
   add (Docker socket + a disk-space check) and catches problems before they
   become a "why is the dashboard down" moment.

## Optional modules (take what's useful, skip the rest)

These were useful for one household — treat each as a menu item, not a
checklist:

- **A lightweight CRM** (customers → projects → updates/decisions/todos) —
  useful if someone in the house freelances/consults and wants a quick
  pipeline view without a heavyweight tool.
- **A sports-bet or hobby-spend tracker** — accounts, individual bets/entries,
  running balance and win-rate — a pattern that generalizes to "track a
  recurring personal ledger of *something*," not necessarily gambling
  specifically.
- **A health/sleep log** — daily entries, a rolling summary — another
  "personal time-series log" pattern.

If you build any of these, consider from the start whether they should be
visible to the whole family or just one profile (see below) — it's much
easier to design that in from day one than retrofit later.

## Multi-profile pattern

Model family members as data, not code: each profile is `{id, name,
initials, color}`, stored wherever your app's config lives (a settings
table, a config file — just not hardcoded string literals scattered across
files). Everything else — which calendar belongs to whom, who owns a task,
which profile's view shows which optional module — refers to a profile by
its `id`, looked up from that one source of truth, so adding, renaming, or
removing a family member never means hunting through code. Decide up front:
do optional modules belong to one specific profile (e.g. one adult's private
CRM/tracker), everyone, or are they independently visible per-profile? Any
of those is fine — just make it a config decision, not an assumption baked
into the code.

## MCP connector pattern

Two entry points sharing one set of tool definitions:

1. **A local stdio MCP server** for dev/debugging directly on the machine
   running the dashboard (or for the Claude Code CLI, pointed at it via
   `claude mcp add`).
2. **A remote HTTP MCP endpoint** (e.g. `/mcp/<random-token>`) that Claude
   Desktop, mobile, and web can add as a custom connector — no per-device
   install, works from anywhere on the account. Protect it with a random
   token baked into the URL path (rotatable independently of any other
   secret), and have the dashboard serve a small `/mcp-setup` page that
   builds the exact URL/command to paste in, using the request's own
   host/protocol rather than a hardcoded domain.

Tools worth exposing: `add_note`, `search_notes`, `add_task`, `list_tasks` at
minimum; extend per whichever optional modules you build (e.g.
`add_customer`/`add_project`/`add_bet`/`add_health_log` and their
list/update/delete counterparts). Keep write endpoints behind a bearer
token separate from the MCP-URL token, since same-origin dashboard reads
don't need auth but writes originating from outside the container do.

## Integration notes

- **Calendar**: Google Calendar via your own Google Cloud service account
  (Calendar API enabled, calendars shared with the service account's email) —
  don't reuse anyone else's credentials file. Design the integration to
  degrade gracefully (empty calendar list, not a crash) if credentials are
  missing or a calendar fetch fails, so a partially-configured install still
  boots.
- **Smart display / chores**: if you own a smart-frame-style device with its
  own API (Skylight was this build's choice), wire it in as an optional
  source that also degrades to "no data" gracefully when unconfigured, rather
  than a hard dependency.
- **General principle**: every external integration should be something the
  app can simply not have, not something that blocks startup — check for
  config at call time, return empty results if absent, and let the UI hide
  or gray out the corresponding panel.

## Deployment shape

- One Docker container, built from a small `Dockerfile` (this build used
  `node:20-alpine`, with `python3 make g++` installed for native deps like
  `better-sqlite3`).
- All secrets/config via `.env`, never committed — ship a `.env.example`
  documenting every variable and what generates it (e.g. `openssl rand -hex
  32` for tokens).
- SQLite data file on a mounted volume for persistence across rebuilds.
- Bearer-token-protected write endpoints; unauthenticated same-origin reads
  for the dashboard's own frontend.
- Reverse proxy in front for HTTPS + a real domain — this is deployment
  infrastructure you already have, not something to build.

---

Now tell Claude: how many people are in your family and what should they be
called, what calendar/smart-display services you actually use, and which of
the optional modules (if any) you want — then have it design and build your
own version from there.
