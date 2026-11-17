# Logiscal Route Dispatch: Build Spec

Oct 14, 2026 · @Ruston

## Overview

We are building a standalone web app where Logiscal uploads its daily Onfleet order CSV, turns the orders into routes, and offers each route to a driver by text. Drivers accept or decline by replying to the text, and the app tracks every confirmation, release, rescue and completion through to pay. It replaces the Airtable "Logiscal Dispatcher" base, which proved the workflow but cannot be maintained or scaled as a no-code build.

Driver records stay in Airtable. The Gigee driver table remains the source of truth for who is onboarded and compliant, and the app only offers work to drivers that table says are active.

**Goals**

- Dispatchers offer a route in two clicks instead of texting drivers one at a time.
- Every route for the day is visible on one board, with its status and driver.
- Orders flow in from Onfleet by CSV, and re-uploads merge without losing work already done.
- Every offer, reply, confirmation and outcome is recorded, so driver reliability is measured, not remembered.
- Pay per route is calculated, adjusted and approved in the app, then exported.
- Compliance stays controlled in Airtable: the app reads driver eligibility from it and writes stats back.

## Users and roles

| Role | Who | Can do |
| --- | --- | --- |
| Dispatcher | Logiscal dispatch team | Upload CSVs, build and edit routes, offer routes, handle releases and rescues, close out routes |
| Client admin | Logiscal manager | Everything a dispatcher can, plus manage their users, areas and hubs, pricing per drop, and approve pay |
| Driver | Gigee driver in Airtable | Receives offers and reminders by SMS; replies YES, NO or STOP. No login |

Dispatchers only see drivers eligible for their client, and only see their own client's orders and routes.

## System context and integrations

The app owns orders, routes, offers, messages and pay. Airtable owns drivers and compliance. Twilio carries every text. Onfleet is the source of orders, delivered as a CSV.

&#91;embedded content: system context · app, Airtable, Twilio, Onfleet\]

Drivers flow in from Airtable and stats flow back out; the app never edits compliance fields.

**Airtable (Gigee driver table): source of truth for drivers**

- **Read (sync in):** every 10 minutes and on demand from the driver picker. Pull Gigee ID, first and last name, mobile phone, area and hub, onboarding status, client-specific status (today: "Logiscal Status"), Manual Tier, and the Airtable record ID.
- **Eligibility is decided in Airtable.** A driver can be offered work only if the synced record is active for this client. The app never edits compliance fields; if a driver is deactivated in Airtable, the next sync blocks new offers and flags any accepted future routes for that driver as Needs attention.
- **Write (sync out):** after each closeout and nightly, write the driver's reliability stats back to Airtable (see Driver reliability): offers sent, acceptance, completion, late bails, no-shows, confirm rate, score, status label, last route date. These go into dedicated read-only fields so compliance and ops can see them where they already work.
- Matching is by Airtable record ID, with Gigee ID as the human-readable key. Phone numbers are normalized to E.164 (+1, US only in v1).

**Twilio: all SMS in and out**

- Outbound offers, confirmations and reminders send from a registered 10DLC number or toll-free number owned by Gigee.
- Inbound replies arrive by webhook. The app matches the sender to a driver by normalized phone, then to that driver's open offer or confirmation.
- Delivery status callbacks are stored so dispatchers can see failed sends.
- STOP, START and HELP are honored and recorded per driver.

**Onfleet: orders by CSV upload**

- Logiscal exports the day's orders from Onfleet and uploads the file. No API in v1.

**Onfleet: where routes are built**

- Routes are built in Onfleet and reach the app through the CSV with their route ID. v1 keeps today's manual step: after a driver accepts, the dispatcher assigns them in Onfleet and ticks "Assigned in Onfleet" on the route. An API integration is a later phase (see Open questions).

## Onfleet CSV import and merge

A dispatcher can upload a new or updated Onfleet file at any time, and the app merges it into the orders it already has. An upload never deletes an order or undoes work on a route; it adds new orders, updates changed ones, and flags anything that needs a human decision.

**Upload flow**

1. Dispatcher drags in a CSV on the Upload screen and picks the client (defaults to their own).
2. The app parses it with that client's saved column mapping. On the first upload, or when columns change, the dispatcher maps columns once and the mapping is saved.
3. The app validates every row and shows a preview before anything is saved: counts of new, changed, unchanged, missing and rejected orders, with the changed fields shown old to new.
4. Dispatcher confirms. The import is saved as an import batch with the file, uploader, time and counts.
5. New orders flow into routes (see Turning orders into routes). Changes to orders already on routes raise alerts on those routes.

**Merge rules** (the merge key is the Onfleet order ID, unique per client)

| Situation | What the app does |
| --- | --- |
| Order ID not seen before | Insert it as a new order |
| Order ID exists, nothing changed | Skip it; counts as unchanged |
| Order ID exists, fields changed, route not yet offered | Update the order and recalculate its route |
| Order ID exists, fields changed, route offered or accepted | Update the order, log the change, and flag the route Needs attention so the dispatcher decides whether to re-text the driver |
| Order in the system but missing from the new file | Keep it, mark it "Not in latest file", and show it in the preview for the dispatcher to cancel or keep. Never auto-delete |
| Order marked cancelled in the file (if Onfleet provides a status) | Cancel the order, remove it from its route, flag the route if it was offered or accepted |
| Same order ID twice in one file | Reject both rows and show them as errors |
| Row missing a required field or with an unreadable date | Reject the row, show the reason, import the rest |

Uploading the same file twice changes nothing. Every change to an order is kept in an order history (field, old value, new value, import batch), so a dispatcher can see what changed and when.

**Required fields** (final list depends on the Onfleet export; see Open questions)

- Order ID
- Service date
- Delivery address (street, city, state, ZIP)
- Area or hub (or a value the app can map to one)
- Onfleet route ID (required; every order arrives on a route)
- Delivery window or route start time, if provided
- Optional: customer name, phone, notes, package count, vehicle type

## Turning orders into routes

A route is a set of orders for one service date, built in Onfleet and run by one driver. Routes arrive already defined in the CSV, and the app takes them exactly as they come: it never builds, splits or rearranges routes.

**How routes are formed**

- Every order in the CSV carries its Onfleet route ID. The app groups orders by service date + route ID, one app route per Onfleet route.
- An order with no route ID is rejected at upload with that reason, so nothing lands unassigned to a route.
- Route changes are made in Onfleet, not in the app. A later upload that adds orders to a route, drops them, or moves an order to another route updates stop counts and payout, with the same alerts as any other change.
- Dispatchers can override a route's start time and vehicle type, but not its orders.

**What each route carries**

| Field | Source |
| --- | --- |
| Route name | The Onfleet route ID in the CSV |
| Service date and start time | Start time = the hub's pickup time from area settings, unless the CSV provides one; dispatcher can override |
| Area and hub | From the orders; all orders on a route share one area |
| Stop count | Number of active orders on the route |
| Vehicle type | Car, SUV, Van or Box; set by dispatcher or CSV |
| Proposed payout | Stop count x the area's price per drop |
| Route fee | The greater of $10 or 5% of proposed payout |
| Adjustments | Added later at closeout (see Closeout and pay) |
| Net payout | Proposed payout minus route fee plus adjustments |

Price per drop, pickup time, pickup address and driver instructions are kept per area in Settings, so they are entered once and reused. The fee rule ($10 minimum, 5%) is a setting, not code. Payout recalculates whenever stops change, until the route is approved for pay; after that it is locked.

## Offering routes by SMS

A dispatcher offers a route to one driver at a time; the driver has 15 minutes to reply YES or NO, and the route locks to them the moment they say yes. No blasting a route to many drivers at once.

**Offering**

1. Dispatcher opens an unassigned route and clicks Offer.
2. The driver picker lists only eligible drivers, best first: active in Airtable for this client, matching the route's area or hub, not already booked that service date, and without another open offer. Each row shows reliability status, score, completion rate, days since last route and phone.
3. Dispatcher picks a driver. The app creates an offer, sends the offer text, and sets the route to Offered.
4. While the offer is open, the route shows who it is with and a countdown.

**Replies**

| Driver texts | What happens |
| --- | --- |
| YES, Y, ACCEPT (any case) | Offer accepted. Route locks to the driver. Driver gets the confirmation text with pickup details |
| NO, N, DECLINE | Offer declined. Route returns to Unassigned. Driver gets a short acknowledgment |
| Nothing within 15 minutes | Offer expires. Route returns to Unassigned. Optional text to the driver that it expired |
| YES after the offer expired or was withdrawn | Not accepted. Driver is told the route is no longer available |
| Anything else | Logged, no state change. Driver gets a help reply; message appears in the dispatcher's Messages inbox |
| STOP | Driver is opted out of texts; blocked from offers until they text START. Flag shows in the picker |
| Text from a number no driver has | Logged in the inbox as unmatched |

A driver has at most one open offer at a time, so every reply maps to exactly one offer. Dispatchers can withdraw an open offer, which cancels it without counting against the driver. The expiry time is a setting (default 15 minutes).

**Message templates** (editable in Settings, with merge fields)

- **Offer:** "Gigee route offer: {date}, start {start\_time} at {hub}. {stops} stops, pays ${payout}. Reply YES to accept or NO to pass. Expires in {expiry} min."
- **Accepted:** "You're confirmed for {route\_name} on {date}. Pickup {start\_time} at {pickup\_address}. {instructions}"
- **Declined:** "Got it, thanks for letting us know."
- **Expired / no longer available:** "That route offer has closed. We'll text you with the next one."
- **Help:** "Reply YES or NO to your latest route offer. Questions? Contact {support\_contact}."

Every outbound and inbound message is stored with its Twilio ID, delivery status, and the offer or route it belongs to.

## Confirmation, release and rescue

Once a driver accepts, the app checks in with them twice before the route and pulls the dispatcher in only when a driver goes quiet, bails, or a route needs rescuing. Routes use six statuses, down from ten in the Airtable build; rescue is a flag on the route, not three extra statuses.

&#91;embedded content: route lifecycle · main path and returns\]

A release can also happen straight from Accepted, and a client cancel moves any open route to Cancelled.

**Route statuses**

| Status | Meaning |
| --- | --- |
| Unassigned | Route imported, no driver |
| Offered | Open offer with one driver, countdown running |
| Accepted | Driver locked in |
| Needs attention | A driver went quiet on a confirmation, an accepted route changed, or the driver was deactivated in Airtable. Driver stays attached until the dispatcher acts |
| Completed | Route run; awaiting closeout and pay approval |
| Cancelled | Client cancelled the route |

**Confirmation schedule** (times are settings; all in the hub's local time zone)

| Check | When | Driver must | If silent |
| --- | --- | --- | --- |
| Heads-up | 48 hours before start | Nothing; informational | n/a |
| Reconfirm | 26 hours before start | Reply YES within 60 minutes | Route goes to Needs attention |
| Final confirm | 2 hours before start; if start is before 8:00am, 6:00pm the evening before | Reply YES within 30 minutes | Route goes to Needs attention |

A NO reply to either check releases the driver as a driver-initiated release. Texts never go out between 9:00pm and 7:00am local except the final confirm rule above. From Needs attention, the dispatcher can mark **Confirmed by phone** (records the confirmation with channel Phone and returns the route to Accepted) or release the driver.

**Releasing a driver**

The dispatcher clicks Release and must pick who initiated it. The app cancels the offer, stamps the release time, sets the offer's outcome, and frees the route.

| Release initiated by | Offer outcome | Route goes to |
| --- | --- | --- |
| Driver, 24 or more hours before start | Early release | Unassigned |
| Driver, under 24 hours before start | Late bail | Unassigned, flagged Rescue |
| Dispatcher (reassigning for ops reasons) | Dispatcher reassign (neutral) | Unassigned |
| Client cancel | Client cancel (excluded from stats) | Cancelled |

After a release, all of the route's reminder timers reset so the next driver gets their own heads-up, reconfirm and final confirm.

**Rescue**

- Any route that loses its driver inside 24 hours, or sits in Needs attention, is flagged Rescue and pinned to the top of the board with time to pickup.
- The dispatcher offers it like any route, or, when a driver agreed by phone, uses **Assign by phone**: this records an accepted offer with channel Phone, without an offer text, and sends the confirmation text.
- A completed route that was rescued credits the covering driver with a rescue in their stats.

## Driver reliability

Reliability is computed from a permanent record of what happened on each offer, never from a status that later changes. Each offer is a ledger entry: every milestone gets its own timestamp, written once and never overwritten, plus one final outcome set at closeout.

**Recorded on every offer**

- Sent at, responded at, reply (Yes / No / None)
- Accepted at
- Reconfirm sent at, reconfirmed at; final confirm sent at, final confirmed at; confirm channel (SMS / Phone)
- Released at, release initiated by (Driver / Dispatcher / Client cancel), hours before start at release
- Rescue cover (yes / no)
- Outcome: Completed, Incomplete, No-show, Late bail, Early release, Dispatcher reassign, Client cancel

The app also writes an append-only offer event log (offer sent, reply received, accepted, released, outcome set, and by whom), which is the audit trail behind every number.

**Rules**

- Window: rolling 90 days by offer sent date.
- Excluded entirely: client cancels, test routes, and offers with no driver.
- Late bail versus early release: 24 hours before route start.
- Dispatcher reassigns count as offers sent and accepted, but not against completion.
- No score until a driver has 5 countable offers in the window; they show as New.

**Metrics per driver**

| Metric | Formula | Tells you |
| --- | --- | --- |
| Response rate | Answered (Yes or No) / offers sent | Do they engage |
| Acceptance rate | Accepted / offers sent | Do they want the work |
| Avg response time | Mean minutes from sent to reply | How fast routes fill with them |
| Completion rate | Completed / committed\* | Do they keep their word (weighted highest) |
| Late bail rate | Late bails / committed | Costliest failure |
| Early release rate | Early releases / committed | Mild penalty |
| No-show rate | No-shows / committed | Worst failure |
| On-time confirm rate | Confirms answered in window / confirms sent | Early warning for bails |
| Rescues | Completed rescue covers | Positive credit |
| Days since last route | Today minus last completed route date | Rotation and churn |

\*Committed = outcomes the driver is accountable for: Completed, Incomplete, No-show, Late bail, Early release.

**Score and status label**

Reliability score = completion rate x 50 + acceptance rate x 20 + on-time confirm rate x 10 + 1 per completed route (max 10) + 3 per rescue (max 9) - 5 per early release - 15 per late bail - 25 per no-show + Manual Tier bump from Airtable (A +15, B +5). Weights live in Settings, not code.

Status label shown to dispatchers: **New** (under 5 offers), **No routes yet** (5+ offers, never committed), **Watch** (2+ no-shows, 3+ late bails, or completion under 85%), **Solid** (85 to 94% completion), **Reliable** (95% and up).

Stats recalculate on every event and are written back to Airtable after each closeout and nightly.

## Closeout and pay

No route can be paid until it has an outcome and an approval, so pay and reliability stats come from the same closeout step.

1. **Mark the result.** After the service date, the Closeout screen lists every accepted route. The dispatcher marks each one Completed (bulk select allowed), Incomplete, or No-show. Completed is the default; No-show and Incomplete are deliberate choices.
2. **Adjust.** Add pay adjustments per route with an amount (plus or minus), a reason from a list (extra stops, wait time, damage, other) and a note. Adjustments are their own records, never edits to the base payout.
3. **Approve.** A client admin approves routes for pay. Approval locks payout, fee and adjustments; changing them after needs Gigee ops and is logged.
4. **Export.** Gigee ops runs a pay export for a date range: one row per driver per route with Gigee ID, driver name, route, date, stops, proposed payout, route fee, adjustments total and net payout. Exported routes are stamped with the export batch.
5. **Mark paid.** When payment is confirmed, the export batch is marked paid, which marks its routes paid.

The export format must match whatever the existing Gigee payout process ingests today (see Open questions).

## Data model

Thirteen core entities. Drivers are a synced mirror of Airtable, never edited directly in the app. All timestamps are stored in UTC and shown in the hub's time zone.

| Entity | Key fields | Notes |
| --- | --- | --- |
| Client | name, time zone, settings | Logiscal first; supports more clients later |
| User | name, email, role, client | Roles from Users and roles |
| Area / Hub | client, name, hub, time zone, price per drop, pickup time, pickup address, driver instructions | Pricing and pickup details live here |
| Driver (mirror) | Airtable record ID, Gigee ID, name, phone (E.164), areas, eligibility per client, Manual Tier, SMS opt-out, last synced | Read-only except opt-out and computed stats |
| Import batch | client, file, uploader, time, counts (new, changed, unchanged, missing, rejected) | Original file kept |
| Order | client, Onfleet order ID (unique per client), service date, address, area, route identifier, window, status, in latest file (yes/no), route | Merge key = Onfleet order ID |
| Order change | order, field, old value, new value, import batch, time | Full history per order |
| Route | client, name, service date, start time, area, vehicle type, stop count, proposed payout, route fee, status, rescue flag, assigned in Onfleet, approved at/by, paid, export batch | Payout locks at approval |
| Offer | route, driver, sent at, responded at, reply, accepted at, reconfirm sent/confirmed at, final confirm sent/confirmed at, confirm channel, released at, release initiated by, rescue cover, outcome, outcome set by | The reliability ledger; milestone fields are write-once |
| Offer event | offer, type, actor (driver, user, system), time, details | Append-only audit trail |
| Message | direction, from, to, body, Twilio SID, delivery status, driver, offer, route, time | Unmatched inbound kept with no driver |
| Adjustment | route, amount, reason, note, created by | Never edits base payout |
| Pay export | client, date range, created by, file, status (exported / paid) | Stamps its routes |

Driver reliability stats are computed from Offer rows (and cached per driver for fast sorting in the picker).

## Screens

Nine screens. The Dispatch board is home, and everything a dispatcher does daily is reachable from it in one click. All screens work on a laptop and are usable on a phone.

| Screen | Who | What it does |
| --- | --- | --- |
| Dispatch board | Dispatcher | Tabs by day (Today, Tomorrow, 2 days, 3 days, Later). Route cards grouped by status, Rescue routes pinned on top with time to pickup. Counts per status. Filter by area |
| Upload routes | Dispatcher | Drop a CSV, map columns, preview new / changed / missing / rejected rows, confirm |
| Route detail | Dispatcher | Orders on the route, payout, offer history, message thread with the driver, Offer / Withdraw / Release / Confirmed by phone / Assign by phone actions, Assigned in Onfleet tick |
| Driver picker | Dispatcher | Eligible drivers ranked by reliability, with status label, score, completion rate, days since last route, opt-out flag |
| Needs attention | Dispatcher | Every route needing a decision and why: silent on confirm, order changed after acceptance, driver deactivated, order not in latest file |
| Messages inbox | Dispatcher | Inbound texts that did not match an action, unmatched numbers, failed sends |
| Closeout and pay | Dispatcher, client admin, Gigee ops | Mark outcomes, add adjustments, approve, export, mark paid |
| Drivers | Dispatcher, Gigee ops | Driver list with 90-day stats and status labels; driver page with offer history. Read-only profile data from Airtable |
| Settings | Client admin, Gigee ops | Areas and pricing, pickup details, column mapping, message templates, timing rules, score weights, users |

## Non-functional requirements

- **Access:** email and password login with password reset; SSO optional later. Every query scoped to the user's client. Gigee ops sees all clients.
- **Audit:** every state change on an order, route, offer, adjustment or setting records who, what and when. Nothing a dispatcher does is unrecoverable.
- **Scheduler:** expiries, reminders and confirm windows run from a job runner checked every minute. Jobs are idempotent: a retry never sends a second text or double-stamps a field.
- **SMS reliability:** outbound sends retry on Twilio errors with backoff; failures surface in the Messages inbox within a minute. Inbound webhook verifies Twilio's signature and responds in under 2 seconds.
- **SMS compliance:** drivers consent to texts during Gigee onboarding (record kept in Airtable). STOP / START / HELP handled automatically. Quiet hours enforced per hub time zone.
- **Time zones:** stored in UTC; every date shown and every reminder scheduled in the hub's local zone.
- **Performance:** a 5,000-row CSV previews in under 30 seconds. The Dispatch board loads a day of 300 routes in under 2 seconds.
- **Airtable limits:** sync respects Airtable API rate limits (5 requests per second per base), batches writes, and retries. If Airtable is unreachable, the app keeps working on last-synced data and shows the last sync time.
- **Data protection:** driver phone numbers and addresses encrypted at rest; HTTPS everywhere; nightly backups kept 30 days.
- **Environments:** staging with a Twilio test number and a copy of the Airtable driver table, separate from production.

## Phasing and acceptance

Three phases. Phase 1 alone replaces the Airtable base for daily dispatching; Logiscal can run live on it before Phase 2 starts.

**Phase 1: Dispatch core**

Scope: login and roles, Airtable driver sync (read), area settings, CSV upload with merge, routes, manual offers by SMS with YES / NO / expiry, Dispatch board, Route detail, Driver picker, Messages inbox, Offer ledger and event log recording from day one.

- [ ] Uploading the same file twice changes nothing; uploading an updated file adds new orders, updates changed ones, and lists missing ones without deleting them
- [ ] A dispatcher offers a route in two clicks and the driver receives the text within 10 seconds
- [ ] A YES reply accepts and locks the route; a NO or 15 minutes of silence returns it to Unassigned, with no dispatcher action
- [ ] A driver deactivated in Airtable cannot be offered work within 10 minutes of the change
- [ ] A driver cannot hold two open offers or two routes on the same service date
- [ ] Every text in and out is visible on the route and in the inbox

**Phase 2: Reliability, confirmations and pay**

Scope: heads-up, reconfirm and final confirm texts; Needs attention; release with reason; rescue and Assign by phone; closeout outcomes; adjustments; approval; pay export; reliability stats and status labels; Airtable write-back.

- [ ] Reminders go out on schedule in the hub's time zone and never during quiet hours
- [ ] A silent driver moves the route to Needs attention at 60 minutes (reconfirm) and 30 minutes (final)
- [ ] A driver release under 24 hours records a Late bail; client cancels never affect stats
- [ ] No route can be exported for pay without an outcome and approval
- [ ] Driver stats on screen match a hand count of that driver's offers for the last 90 days
- [ ] Stats appear in Airtable after closeout

**Phase 3: Later**

Candidates, each to be scoped separately: Onfleet API assignment, automatic offer queue (offer the next best driver when one declines or expires), client-facing reports, Onfleet API instead of CSV.

## Round 2 additions (Nov 16, 2026)

### Onfleet task export
The real export is one row per task. Each `Driver Number` is one route. Rows whose `taskType` is not `dropoff` are the hub start and are skipped. The market and hub come from `facilityId`, falling back to the `workerName` suffix (for example `INDY_LOGISCAL`). Order ID is `orderId`, else `shortId`. Service date is `shippingDate`, else the date in `completeAfterTime`. `destinationLonLat` and `Task Number` give stop order, road miles and the first leg. App route names are hub code, month and day, and Driver Number (for example `IN-1018-02`). Rows are rejected for a missing address, a blank Driver Number, no service date, an unknown hub or a duplicate order ID; the rest of the file still imports.

### Pricing by market
Each market (a client in a city) holds its own parameters: client rate per drop and per B2B drop; driver base per drop per hub; extra per B2B drop and per apartment or unit stop; included route miles and a per-mile rate over that; included first-leg miles and a rate over that; an hourly floor; a target margin; a glide limit; rounding; and the speed, minutes per stop and load time used to estimate hours. Pay = base + extras + mileage + first leg, topped up to the floor for the estimated hours, with per-stop pay held within the glide limit of the lane's last run, then rounded. Margin = (client revenue − driver pay) ÷ client revenue; routes under target are flagged. Pay locks when the offer or confirm text goes out; parameter changes re-price only routes nobody has been texted about. Every change is logged with who and when. Pricing access is per user: edit, view (sees rates, revenue, margin) or none (sees only driver pay).

### Driver already set in Onfleet
When a route arrives with a `workerName`, that driver is the preferred driver. If they are in Airtable and eligible, the app texts them a confirm (setting: on import, or one click) with exact pay, a stop range, cities, miles and hours. If they are not in Airtable or not eligible, the route falls back to lane history and shows the reason. Recognition of old routes matches delivery addresses without the unit number, optionally also customer names (setting).

### Onfleet check
Each upload records the driver Onfleet has on each route. The Onfleet check screen lists routes for today through 3 days out as Matches, Waiting on driver, Set in Onfleet (accepted in the app, no driver in Onfleet) or Out of sync (Onfleet and the app disagree). A later file that moves an accepted route to a different driver flags it Needs attention, with "Keep current driver, I fixed Onfleet" or "Switch to Onfleet's driver". Writing the assignment back through the Onfleet API is a later phase.

## Round 4 changes (Nov 16, 2026)

These supersede the sections above where they differ.

- **Uploads only add routes.** A route already in the app is never re-imported or changed by a later file. Uploading the same file twice imports nothing and says so. The preview lists routes already in the app with their stop count in the app and in the file.
- **Adjust route.** When stops or pay change in Onfleet after a route is in the app, the dispatcher records an adjustment on the route: stops (plus or minus), pay change (suggested at the hub's base per drop), reason, note, and an optional text to the driver. Adjustments are logged on the route and the offer ledger, add to the stop count and pay, and stop once the route is signed off. This replaces re-importing changed orders, missing-order decisions and the Onfleet check screen.
- **Onfleet check screen removed.** The route page keeps the Onfleet driver from the file and an "Assigned in Onfleet" tick.
- **Markets and pricing is admin only** (client admin and Gigee ops). Dispatchers do not see the page, client rates or margin. Per-user pricing access and the Regional Lead role are removed from the demo.
- **Pay is daily.** Daily closeout replaces the weekly Closeout and pay: per service day, the dispatcher marks each route's result and signs it off as complete and ready to pay. Sign-off locks the pay. Gigee ops exports that day's pay and marks it paid. No separate client admin approval.
- **Holds removed** for now: no manual or suggested holds.
- **Messages inbox removed** for now. Each route keeps its own text history.

## Open questions

- [ ] **Onfleet export:** get a real sample file. Confirm the exact columns for order ID, Onfleet route ID, order status (for cancellations), and area or hub.
- [ ] **Airtable driver table:** confirm the base and table, the exact fields that mean "eligible for Logiscal," and where stats should be written back (new fields we own vs. an existing table).
- [ ] **Onfleet:** is an API integration wanted in Phase 3, to pull routes directly and push the assigned driver back instead of the CSV and the manual tick?
- [ ] **Pay export:** what format does the Gigee payout process take today, and who approves pay: Logiscal, Gigee ops, or both?
- [ ] **Twilio number:** which number sends, and is its 10DLC or toll-free registration complete?
- [ ] **Offer expiry:** keep 15 minutes, or use the 30 minutes from the original June design?
- [ ] **Heads-up text:** is the 48-hour heads-up a plain reminder, or should it ask for a reply too?
- [ ] **Late changes:** when an order on an accepted route changes, should the app re-text the driver automatically, or always leave it to the dispatcher?
- [ ] **Other clients:** should v1 support a second client, or is client separation built in but only Logiscal turned on?
- [ ] **Migration:** do we import the Airtable base's history (90 routes, 55 offers, all before July 21) or start clean?
