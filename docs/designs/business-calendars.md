# Business calendars: per agent access, like mailboxes

Status: proposal, for discussion before any code (CONTRIBUTING, "A rewrite nobody asked for").

## Why

Three shipped job descriptions already promise calendar work:

- Restaurant: the Executive Admin "keeps the bookings calendar" (`docs/designs/agent-instructions/README.md:123`).
- Pro services: Pam "keeps appointments and filing dates on the calendar" (`:474`).
- Home services: Pam "keeps bookings current, catches double bookings and sends the night before schedule" (`:972`).

Today the only way a team member reaches a calendar is the Google Calendar connector on the
owner's Claude account (`docs/designs/claude-connectors.md`). It is switched on or off per
agent, and that is all:

| | Mailboxes today | Calendar today |
|---|---|---|
| How many | As many as the business runs on | The one Google account behind the owner's Claude login |
| Who uses it | One watcher, plus Send only members | Every agent the connector is given to |
| What it may do | Can send / Send on approval / Draft only | Everything the connector exposes, or nothing |
| The owner sees a write before it happens | Send on approval, edit and approve on Ask me | No |
| A retry never acts twice | Send journal (`mail-send-journal.json`) | No |
| Providers | Any IMAP mailbox with an app password | Google only |

For a restaurant, a cleaning company or an accounting office, a booking is as customer facing
as an email: a wrong or double booking sends a crew to the wrong house or seats two parties at
one table. The office already has the right shape for this in mail. This proposal gives
calendars the same shape.

## What the owner sees

1. **Settings, Connections, Calendars.** Add a calendar account (iCloud, Fastmail, Zoho, Nextcloud
   or any CalDAV server) with an app password. The login is tested before it is saved and the
   password is kept in the secret store, as mailboxes are. The account lists its calendars; the
   owner ticks the ones the office may use ("Bookings", "Crew schedule").
2. **Access tab, Calendar section**, on each team member. Add a calendar and choose one of:

   | Choice | Value | What the team member can do |
   |---|---|---|
   | Can book | `book` | Read, and create, move or cancel events at once |
   | Book on approval | `approval` | Read; every create, move or cancel goes on Ask me first |
   | Read only | `read` | Read events and free or busy time; no changes |

   New grants start on Read only, the most restrictive choice, as new mailboxes start on Draft only.
   Unlike a mailbox, a calendar may be given to several team members: the Admin books, Finance reads
   it to invoice finished jobs.
3. **Ask me card** for Book on approval: what, when, where, who is invited, and the clash check
   result. The owner can edit the time or title, Approve, Ask for changes or Don't book. The team
   member then makes exactly the approved change, as with Send on approval.
4. **Starter job** for the Executive Admin in packs that book work (restaurant, home services, pro
   services): "Send tomorrow's schedule", each office day at closing time minus one hour, focus:
   tomorrow's bookings, clashes, and anything missing an address or a phone number.

## Design

### Records

In `config.json`, beside `mailboxes`:

```ts
interface CalendarAccount {
  id: string;
  label: string;            // "Bookings (iCloud)"
  provider: 'icloud' | 'fastmail' | 'zoho' | 'caldav';
  serverUrl: string;        // filled in per provider, typed only for 'caldav'
  username: string;
  secretRef: string;        // the app password, in integration-secrets.json
  calendars: Array<{ id: string; url: string; name: string; enabled: boolean }>;
  status: 'connected' | 'needs-attention';
  statusReason?: string;
}
```

In `AgentCapabilities` (`src/shared/mailboxes.ts`), one new optional field, so every existing record
still reads:

```ts
calendars?: Array<{ calendar: string; access: CalendarAccess }>;   // CalendarAccess = 'book' | 'approval' | 'read'
```

### The one rule: `calendarAccess`

A new `src/shared/calendars.ts`, written like `mailAccess`: a pure function the broker and the
PreToolUse hook both call, returning `{ ok: true }` or `{ ok: false, reason }` with a reason
that says what happened and what to do.

```ts
type CalendarOp = 'list' | 'read' | 'propose' | 'write';
calendarAccess(cfg, agentId, calendarId, op, proposalId?): { ok: true } | { ok: false; reason: string }
```

- `list` needs any calendar grant.
- `read` needs a grant on that calendar, and the calendar still enabled in Settings.
- `propose` needs Book on approval. Can book refuses it ("make the change yourself"), as
  `mailAccess` refuses `propose` on Can send, so no card reaches Ask me for a member who needn't ask.
- `write` needs Can book, or Book on approval with the id of an approved proposal. The broker
  then writes the approved version it stored, never values from the call.
- Read only refuses `propose` and `write` and names the Access tab.

A `CALENDAR_TOOL_OPS` map, built on `Object.create(null)` like `MAIL_TOOL_OPS`, ties each tool to
its op.

### The tool: `md-calendar`

`resources/md-calendar-mcp.cjs`, modelled on `md-mail-mcp.cjs`: a stdio MCP server that holds no
credentials and forwards every call to the loopback broker as `POST /calendar/<tool>` with the
agent's broker token. It is passed through `--mcp-config` beside md-mail, so it survives the strip
of the owner's own servers.

| Tool | Op | Does |
|---|---|---|
| `list_calendars` | list | The calendars this member may use and its access on each |
| `search_events` | read | Events in a time range, optional text match |
| `free_busy` | read | Busy blocks in a range, across the member's calendars |
| `propose_event` | propose | Puts a create, move or cancel on Ask me; returns a proposal id |
| `create_event` | write | Creates an event |
| `update_event` | write | Moves or edits an event, by its id and ETag |
| `cancel_event` | write | Cancels an event; it is marked cancelled, never deleted |

Nothing deletes an event, for the same reason nothing in md-mail deletes mail: a cancelled event
can be restored in the calendar app.

### The broker and CalDAV

The broker (`src/main/integrationBroker.ts`) gets a `calendar` route handler, injected like
`mail` (`MailRouteHandler`). A new `src/main/calendar.ts` holds the CalDAV client:

- Discovery with `PROPFIND` (principal, calendar home, calendars).
- Reads with a `REPORT calendar-query` using `<C:expand>` over the asked range, so the server
  expands recurring events and the app never evaluates RRULEs itself.
- Writes with `PUT`, guarded by `If-Match` on the event's ETag, so a change made by the owner in
  their phone in the meantime is never overwritten. A 412 comes back to the agent as "the event
  changed since you read it; read it again".
- A small iCalendar reader and writer for the fields the tools use (UID, summary, start, end,
  time zone, location, description, attendees, status). Times are written with a `TZID` taken from
  the company profile's time zone.

Dependency choice is an open question below. Either way, network calls stay in the main process,
behind the same 30 second upstream timeout as the rest of the broker.

### Clash check

Before `propose_event` stores a proposal and before any `write`, the broker reads free or busy
time on the target calendar for the event's span. A clash does not refuse the call: it is written
on the Ask me card for Book on approval, and returned to the agent on Can book, which must call
again with `allowClash: true` and say why. A clash is never silent.

### Proposals

App owned, in a `calendar-proposals.json` in the app's data folder, next to `mail-proposals.json`,
and never in a folder an agent can write. The store reuses the proposal states of `src/shared/mailProposals.ts`
(`waiting`, `approved`, `changes`, `declined`, then `booking` and `booked` in place of `sending`
and `sent`, and `cancelled`), and the shared rules move into one helper so both stores keep one
behaviour. The Ask me card reuses `MailProposalCards` layout with
calendar fields.

### Never twice

Each write is recorded in `calendar-write-journal.json` before the upstream call and settled
after, keyed on agent, calendar, operation, event UID and a hash of the event, the pattern of
`mail-send-journal.json`. Creates also use a UID the broker makes once per proposal or call, so a
`PUT` repeated after a lost answer lands on the same event instead of a new one. A write left
pending after a stop that cannot be checked is refused with a plain reason telling the agent to
tell Michael.

### The hook

The PreToolUse hook (`src/main/hooks.ts`) gains an `^mcp__md-calendar__` branch next to the
md-mail one, reading `calendarAccess` with the same single config read. The broker checks again,
so a call that slips past the hook is still refused.

### What stays as it is

- The Google Calendar connector on the owner's Claude account keeps working, unchanged, under
  Claude connectors. Google's CalDAV needs OAuth, not an app password, so Google Calendar under
  this design is a follow up (below), not part of the first ship.
- Pete gets no calendar tools. Calendars hold customer names, addresses and phone numbers.

## Tests

| Case | Kind |
|---|---|
| `calendarAccess`: each op against each access level, missing grant, calendar disabled in Settings, unknown tool names such as `constructor` | unit |
| `propose_event` on Can book refused; `create_event` on Book on approval without a proposal refused; with an approved one, the stored version is written | unit |
| A record with no `calendars` field reads as no access | unit |
| CalDAV against a local test server: discovery, expanded recurring events, ETag clash returns the plain reason | integration, as `test/` does for mail with hoodiecrow and smtp-server |
| A repeated create after a lost answer leaves one event | integration |
| A pending journal entry after a restart is refused, not repeated | unit |
| Clash shows on the card and comes back to a Can book agent | unit |
| New owner facing strings: no dashes, `{{godName}}` for the manager, in en, ar and zh-CN | existing `no-dashes` and i18n tests |

## Phases

1. **Read only.** Accounts, the Access section, `list_calendars`, `search_events`, `free_busy`, the
   rule and the hook. Already enough for "Send tomorrow's schedule" and for Finance to invoice
   finished jobs.
2. **Book on approval.** Proposals, the Ask me card, the clash check, the journal.
3. **Can book**, and the starter job in the three packs.
4. **Follow ups:** Google Calendar through OAuth; Microsoft 365 with the same work as Outlook mail;
   standing approvals for one kind of booking, as Send on approval has.

## Open questions

1. **Dependency.** A small hand written CalDAV and iCalendar layer (only the verbs and fields above)
   or `tsdav` plus `ical.js`. The hand written one keeps the bundle small and its surface is narrow;
   the libraries handle server quirks that would otherwise show up as bug reports.
2. **Several members on one calendar.** Is a calendar shared by default, as proposed here, or one
   owner plus read only grants, as mailboxes do with Send only?
3. **Who may cancel.** Should cancelling always go on Ask me, even under Can book?
4. **Where it sits for Michael.** Does Michael get read access to every enabled calendar, as he
   reads every team folder, so the standup can see the day's bookings?
