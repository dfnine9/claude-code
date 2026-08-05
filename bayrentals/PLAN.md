# BayRentals Platform — Full Build Plan (v4, one-shot ultracode edition)

> **How to use this document:** This is a ONE-SHOT plan. Put this file in an empty
> directory (or repo), open Claude Code there, and send exactly:
>
> ```
> Read PLAN.md and build the entire BayRentals platform in one shot,
> following its one-shot execution protocol. ultracode
> ```
>
> The word **ultracode** opts Claude Code into multi-agent workflow orchestration —
> it will fan out parallel subagents per the protocol in §0 instead of working
> serially. Everything below §0 is the build spec those agents implement.

---

## 0. One-shot execution protocol (ultracode)

### What "one shot" delivers

The one-shot deliverable is the **complete repository**: every migration, the RLS test
suite, the pricing engine with its exhaustive tests, every edge function, the full
SwiftUI app, the web widget, CI, and docs — all buildable and verifiable **locally with
zero external accounts**. Every external service (Stripe, Postmark, APNs, Wallet) is
called through a thin client interface with secrets read from `.env.example` /
`Config.local.xcconfig` placeholders, so the code is complete and tests run against
stubs. Anything that genuinely requires Daniel's accounts or a physical iPhone is
written into **`docs/GO-LIVE.md`** as an ordered checklist — never silently skipped.

### Orchestration blueprint

Execute as waves. Parallelize within a wave; verify between waves. Use worktree
isolation for parallel agents that write files.

- **Wave 0 — Scaffold (single agent).** Repo layout (§6), `.gitignore`, CI skeleton,
  `docs/` stubs, and — critically — the **shared contracts** the parallel agents build
  against: SQL table definitions (§7) frozen into an initial migration draft, TypeScript
  types for API payloads and the quote's `line_items[]`, and Swift model definitions
  mirroring them. These contracts are the interface between agents; later waves may
  extend but not reshape them.
- **Wave 1 — Parallel build (one agent per module, worktrees).**
  1. Database: final migrations + RLS policies + `supabase/tests`
  2. Pricing module (`_shared/pricing.ts`) — test-first, full unit suite (§10)
  3. Edge functions, grouped: quote/create-booking/cancel-booking · contracts pair ·
     turo-inbound (+ fixture corpus) · stripe-webhook/send-push/wallet-pass ·
     scheduled-runner
  4. iOS foundation: design system, models, API/auth services, offline queue
  5. iOS customer surface: Browse, CarDetail, BookingFlow, MyTrips timeline, Account
  6. iOS staff surface: Today, OpsCalendar, wizards, DamageCompare, Claims, Fleet,
     Insights, TuroReviewQueue
  7. Web widget
  8. CI workflows + `docs/SETUP.md` + `docs/POLICIES.md` placeholders
  Rule: an agent touches only its module's directory; cross-module needs go through
  the Wave 0 contracts.
- **Wave 2 — Integration (single agent).** Merge worktrees, resolve seams, make the
  whole tree build: `supabase db reset` green with all database tests, `deno test`
  green across functions, widget typecheck/build green, `xcodebuild build` if the host
  has Xcode (record as a GO-LIVE item if not).
- **Wave 3 — Adversarial verification (parallel skeptic agents, loop until dry).**
  Independent auditors, each prompted to find real failures, with findings fixed and
  re-audited until two consecutive sweeps find nothing new:
  - RLS/security audit (customer reaching another's data? anon reaching PII?)
  - Pricing audit: recompute §10 scenarios by hand, compare to module output
  - Failure-mode drill: every row of the §12 catalog — does the designed behavior
    exist in code, and where?
  - Booking race-condition review (exclusion constraint actually the last guard?)
  - Completeness critic: every section of this plan is either implemented or has an
    explicit GO-LIVE.md line — nothing dropped silently
- **Wave 4 — Handoff (single agent).** Final `docs/GO-LIVE.md` (ordered: accounts,
  secrets, `supabase link` + deploy, Turo email forward, on-device test day, TestFlight,
  App Store submission — with §13's on-device acceptance criteria mapped in), README
  quickstart, and a build report: what was built, test counts, what awaits Daniel.

### What stays human, honestly

Camera flows, the offline counter drill, the 2-minute pickup target, and the
zero-training test need a physical iPhone and real staff — they are preserved as
GO-LIVE.md's "first TestFlight day" checklist, not claimed as done. Same for account
creation, secret provisioning, attorney review of the contract template, and App Store
review itself. A one-shot that pretends otherwise is lying; this one doesn't.

---

## 1. Context

You are building a full-stack rental car platform for **BayRentals** (Bayrentals.com), a
small rental car company whose fleet is also listed on **Turo**. The owner (Daniel,
daniel@dfnine.com) needs:

- An **iPhone app** used by BOTH customers (browse cars, book, pay) and staff
  (calendar, condition photos at pickup/return, contract signing at the counter).
  Role-based: same app, staff see extra tools.
- A **backend** holding the fleet, availability calendar, bookings, customer records,
  pricing rules, photo storage, and signed contracts.
- **Turo integration** (see the hard constraint below — email-ingestion based, not API
  based).
- **Bayrentals.com integration** — the website reads the same availability, quote, and
  booking API as the app.
- **Fleet operations** — maintenance tracking, damage claims, and owner insights, so
  the platform runs the business, not just the bookings.

The bar is **seamless**: renting a car from BayRentals should feel as smooth as an
Apple-native experience — for the customer AND for staff at the counter.

## 2. Locked decisions (do not re-litigate)

| Decision | Choice |
|---|---|
| iOS app | **Native SwiftUI**, iPhone-first, iOS 17+ |
| Audience | **One app for both** customers and staff, role-based UI |
| Backend | **Supabase** (Postgres + Auth + Storage + Edge Functions + Realtime + pg_cron) |
| Auth | **Passwordless only**: Sign in with Apple + email OTP. No passwords, ever. |
| E-signature | **Built-in**: finger-signing in app, self-generated PDF, own audit trail |
| Payments | **Stripe** — PaymentSheet with **Apple Pay**, rental charge + refundable deposit hold |
| Pricing | **Server-side pricing engine**, single shared module; clients only display quotes |
| Transactional email | Postmark (or Resend) — also provides inbound parsing for Turo emails |
| Source of truth for availability | **Our database**, always. Turo bookings flow in; they never own the calendar. |

## 3. Seamlessness principles

These govern every feature. When making a UX decision, check it against this list:

1. **No passwords.** Sign in with Apple or a 6-digit email code. Guests can book with
   just email + payment; the account materializes around the booking.
2. **Never type what can be scanned.** The US driver's license has a PDF417 barcode on
   the back encoding name, DOB, license number, and expiry (AAMVA format). Scan it with
   the Vision framework and auto-fill the entire customer record. Typing is the fallback,
   not the flow.
3. **Nothing waits for the counter.** Contract review + signing, license capture, and
   payment all happen in-app **before pickup**. Counter time target: under 2 minutes —
   walkaround photos and keys.
4. **Calendars are never stale.** Supabase Realtime pushes booking changes to every
   open screen — a booking on the website blocks the app calendar the same second.
5. **No dead ends offline.** Photo uploads and status changes queue on-device and drain
   when signal returns. Staff never lose work in a parking-garage dead spot.
6. **One tap where possible, one screen where not.** Apple Pay for payment. Rebook a
   past car in one tap. Every multi-step staff task is a single guided wizard, not a
   scavenger hunt across screens.
7. **The app talks first.** Push + email at every lifecycle moment (matrix in §11);
   the customer never wonders what's next.
8. **Money is itemized and never surprising.** Every quote and every return charge is a
   line-item breakdown (rate × days, discounts, extras, fees, taxes) computed in one
   server-side module. Customers see the same numbers before booking, at pickup, and on
   the receipt.
9. **The fleet takes care of itself.** Maintenance comes due by miles or months and the
   app says so before a customer is stranded; registrations and inspections never expire
   silently.
10. **Zero training required.** A brand-new employee completes a pickup on day one with
    no instruction — the wizard is the training. A first-time customer books without
    reading anything. Every empty state teaches; every error says what happened and what
    to do next, in plain words, with a button that does it.

## 4. Hard constraint: Turo has no public API

Turo offers no developer API, and scraping or unofficial APIs violate their Terms of
Service and risk the host account. **Never build against unofficial Turo endpoints.**

The integration that works and is ToS-safe:

- **Inbound (automated):** Turo emails the host on every trip booked / modified /
  canceled. Daniel sets up an auto-forward rule from his email to an inbound-parse
  address (Postmark inbound webhook). An edge function parses each email, matches the
  car, and creates/updates a `bookings` row with `source = 'turo'`, blocking those dates.
  Anything the parser isn't confident about lands in a **review queue** in the staff app.
- **Outbound (guided manual):** when a direct booking is created, Turo must be blocked
  by hand in the Turo host app. The staff app shows a checklist item ("Block these dates
  on Turo") with a deep link to the Turo app, and tracks whether it was marked done.

## 5. Architecture

```
┌─────────────┐   ┌──────────────────┐   ┌─────────────────┐
│  iPhone app │   │  Bayrentals.com  │   │ Turo booking     │
│  (SwiftUI)  │   │  booking widget  │   │ emails (forward) │
└──────┬──────┘   └────────┬─────────┘   └────────┬────────┘
       │ ▲ Realtime        │                      │ Postmark inbound webhook
       ▼ │                 ▼                      ▼
┌─────────────────────────────────────────────────────────────┐
│                        SUPABASE                             │
│  Postgres (RLS) · Auth · Storage · Realtime · pg_cron       │
│                                                             │
│  Functions: quote · create-booking · cancel-booking ·       │
│  turo-inbound · generate-contract · finalize-contract ·     │
│  stripe-webhook · send-push · wallet-pass · scheduled-runner│
└──────────────┬──────────────────────────┬───────────────────┘
               ▼                          ▼
            Stripe                  Postmark (outbound
   (Apple Pay, payments,            email: confirmations,
    deposit holds, refunds,         signed contracts,
    Radar fraud rules)              receipts)
```

Admin/back-office: use **Supabase Studio** for raw data in early phases; the staff app's
Ops tools + Insights screen are the real back office. A dedicated admin web dashboard is
parked (§14) unless a real need appears.

## 6. Repository layout (dedicated repo — see Phase 0)

```
bayrentals/                  # repo ROOT (its own private GitHub repo)
├── PLAN.md                  # this file
├── .github/workflows/       # CI (see §12)
├── supabase/
│   ├── config.toml
│   ├── migrations/          # SQL migrations (numbered, never edited after merge)
│   ├── tests/               # RLS + constraint tests (database-level)
│   └── functions/
│       ├── _shared/         # pricing module, notification templates, clients
│       ├── quote/
│       ├── create-booking/
│       ├── cancel-booking/
│       ├── turo-inbound/
│       ├── generate-contract/
│       ├── finalize-contract/
│       ├── stripe-webhook/
│       ├── send-push/
│       ├── wallet-pass/
│       └── scheduled-runner/
├── ios/
│   ├── project.yml          # XcodeGen spec (generates BayRentals.xcodeproj)
│   └── BayRentals/
│       ├── BayRentalsApp.swift
│       ├── Config.swift
│       ├── DesignSystem/    # colors, type scale, components, haptics
│       ├── Models/
│       ├── Services/
│       └── Views/{Auth,Fleet,Booking,Trips,Staff,Insights,Account}/
├── web-widget/              # Phase 4: embeddable booking widget for Bayrentals.com
└── docs/                    # SETUP.md, OPEN-ITEMS.md, POLICIES.md, GO-LIVE.md
```

## 7. Data model (Postgres migrations)

Enable extensions: `btree_gist` (needed for the no-double-booking constraint),
`pgcrypto`, `pg_cron`.

### Core tables

**`profiles`** — extends `auth.users`
- `id uuid PK references auth.users`
- `role text check in ('admin','staff','customer') default 'customer'`
- `full_name text`, `phone text`
- `push_token text nullable` (APNs device token)
- Auto-created via trigger on auth signup.

**`cars`**
- `id uuid PK`, `slug text unique`
- `make, model, trim, color, license_plate, vin text`, `year int`
- `seats int`, `transmission text`, `fuel text`
- `daily_rate_cents int`, `deposit_cents int`
- `mileage_limit_per_day int nullable` (null = unlimited)
- `min_renter_age int nullable` (falls back to `policies.min_renter_age`)
- `description text`, `status text check in ('active','maintenance','retired') default 'active'`
- `pickup_instructions text` (shown in the customer trip timeline)
- `turo_listing_url text` (for the outbound deep-link checklist)
- `registration_expiry date nullable`, `inspection_expiry date nullable`

**`car_photos`**
- `id uuid PK`, `car_id FK`, `storage_path text`, `position int`

**`customers`** — walk-ins may have no auth account, so this is separate from profiles
- `id uuid PK`, `profile_id uuid FK nullable`
- `full_name, email, phone text`
- `license_number, license_state text`, `license_expiry date`, `date_of_birth date`
- `license_photo_path text` (private bucket)
- `license_verified_at timestamptz nullable` (set when barcode scan matched the photo)
- `stripe_customer_id text nullable` (saved payment methods, one-tap rebooking)

**`bookings`** — one table for all three block types
- `id uuid PK`, `car_id FK`, `customer_id FK nullable` (null for turo/manual)
- `source text check in ('direct','turo','manual_block')`
- `status text check in ('pending','confirmed','active','completed','canceled')`
- `start_at, end_at timestamptz`
- `quote jsonb` — the full itemized quote frozen at booking time (line items, discounts,
  extras, taxes, total, deposit)
- `total_cents, deposit_cents int` (denormalized from quote for fast queries)
- `turo_trip_ref text nullable` (unique when not null — dedupes re-parsed emails)
- `turo_block_confirmed bool default false` (outbound checklist state for direct bookings)
- `notes text`, `created_by uuid`, `created_at timestamptz`
- **Critical constraint — no double booking, enforced by the database itself:**
  ```sql
  ALTER TABLE bookings ADD CONSTRAINT no_overlap
    EXCLUDE USING gist (car_id WITH =, tstzrange(start_at, end_at) WITH &&)
    WHERE (status NOT IN ('canceled'));
  ```
- Realtime enabled on this table (powers live calendars everywhere).

### Pricing & policy tables

**`policies`** — single row of business rules, editable without deploys
- `min_renter_age int`, `grace_period_minutes int`
- `late_fee_cents_per_hour int`, `mileage_overage_cents_per_mile int`
- `fuel_charge_cents_per_eighth int` (charge per missing 1/8 tank at return)
- `cancellation_tiers jsonb` — e.g. `[{"hours_before":72,"refund_pct":100},
  {"hours_before":24,"refund_pct":50},{"hours_before":0,"refund_pct":0}]`
- `tax_rate_pct numeric`, `booking_fee_cents int`
- Values live in `docs/POLICIES.md` until Daniel confirms them (§15) — seed with
  placeholders clearly marked `TODO`.

**`rate_rules`** — pricing engine inputs, evaluated by priority
- `id uuid PK`, `car_id FK nullable` (null = fleet-wide)
- `kind text check in ('weekend','season','duration_discount')`
- `params jsonb` — e.g. weekend: `{"days":["fri","sat"],"multiplier":1.25}`;
  season: `{"from":"06-01","to":"08-31","multiplier":1.2}`;
  duration: `{"min_days":7,"discount_pct":15}` / `{"min_days":28,"discount_pct":30}`
- `priority int`, `active bool`

**`extras`** — bookable add-ons
- `id uuid PK`, `name text` (child seat, toll pass, prepaid fuel, additional driver,
  delivery), `kind text check in ('one_time','per_day')`, `price_cents int`,
  `active bool`

**`booking_extras`**
- `booking_id FK`, `extra_id FK`, `qty int`, `price_cents int` (frozen at booking)

**`booking_charges`** — post-booking itemized charges (return + incidents)
- `id uuid PK`, `booking_id FK`
- `kind text check in ('mileage','fuel','late','damage','other')`
- `description text`, `amount_cents int`
- `stripe_payment_intent_id text nullable`, `created_at timestamptz`

### Operations tables

**`condition_reports`** — pickup/return walkarounds
- `id uuid PK`, `booking_id FK`, `kind text check in ('checkout','checkin')`
- `odometer int`, `fuel_level_eighths int` (0–8), `notes text`, `created_by uuid`

**`condition_photos`**
- `id uuid PK`, `report_id FK`, `storage_path text`
- `slot text check in ('front','rear','left','right','interior','extra')` — enables
  side-by-side checkout-vs-checkin comparison per angle
- `taken_at timestamptz`, `lat, lng numeric nullable` (EXIF-sourced — this is
  damage-dispute evidence, preserve capture metadata)

**`car_services`** — maintenance tracking
- `id uuid PK`, `car_id FK`, `kind text` (oil, tires, brakes, registration, inspection,
  other), `due_at date nullable`, `due_odometer int nullable`
- `completed_at timestamptz nullable`, `completed_odometer int nullable`, `notes text`
- Scheduled runner flags anything due; completing a service can log the next one.
  Scheduling a service creates a `manual_block` booking so the calendar stays honest.

**`claims`** — damage found at return
- `id uuid PK`, `booking_id FK`, `report_id FK nullable`
- `status text check in ('open','estimated','charged','waived','closed')`
- `description text`, `amount_cents int nullable`
- `stripe_payment_intent_id text nullable` (deposit capture or separate charge)
- `created_at timestamptz`

### Integration tables

**`contract_templates`**
- `id uuid PK`, `name text`, `body_html text` with merge fields like
  `{{customer_name}}, {{car_display}}, {{start_at}}, {{end_at}}, {{quote_table}},
  {{deposit}}`, `is_default bool`

**`contracts`**
- `id uuid PK`, `booking_id FK`, `template_id FK`
- `status text check in ('draft','signed','void')`
- `pdf_path text`, `signature_path text`
- `document_sha256 text` — hash of the final signed PDF
- `signed_at timestamptz`, `signer_name text`, `signer_ip inet`
- `audit jsonb` — append-only event list: generated, presented, signed, emailed

**`payments`**
- `id uuid PK`, `booking_id FK`
- `kind text check in ('rental','deposit_hold','deposit_capture','extra_charge','refund')`
- `stripe_payment_intent_id text`, `amount_cents int`, `status text`

**`turo_inbound_emails`** — raw ingestion log + review queue
- `id uuid PK`, `message_id text unique`, `from_email, subject text`, `raw_body text`
- `parse_status text check in ('parsed','needs_review','ignored')`
- `parsed jsonb`, `booking_id FK nullable`

**`notifications_log`** — every push/email sent; also the idempotency guard for the
scheduled runner (event + booking + recipient unique)
- `id uuid PK`, `booking_id FK nullable`, `recipient text`, `channel text`,
  `event text`, `sent_at timestamptz`

**`app_config`** — single row, fetched at app launch
- `min_supported_build int` — builds below this show a friendly "update required"
  screen instead of breaking against a newer API
- `maintenance_message text nullable` — emergency banner without an app release

### Row-level security

- `admin`/`staff` (checked via a `security definer` helper reading `profiles.role`):
  full access to everything.
- `customer`: read own `customers` row, own bookings/quotes/charges, own contracts; no
  access to other customers, condition photos of others, claims, services, or the Turo
  email log.
- `anon`: SELECT on active `cars`, `car_photos`, `extras`, and an **`availability`
  view** exposing only `(car_id, start_at, end_at)` of non-canceled bookings — never
  customer data. This view + the `quote` function power the public website widget.
- **RLS is tested, not assumed** — `supabase/tests/` contains per-role tests that run
  in CI (§12).

### Storage buckets

| Bucket | Access |
|---|---|
| `car-photos` | public read |
| `condition-photos` | staff only |
| `licenses` | staff only |
| `contracts` | staff + owning customer via signed URLs |

## 8. Edge functions

All in TypeScript (Deno). Shared code in `functions/_shared/` — **the pricing module
lives here and nowhere else**. Secrets via `supabase secrets set`: `STRIPE_SECRET_KEY`,
`STRIPE_WEBHOOK_SECRET`, `POSTMARK_SERVER_TOKEN`, `POSTMARK_INBOUND_SECRET`,
`APNS_KEY` (+ team/key ids), `WALLET_PASS_CERT`.

1. **`quote`** — the pricing engine's public face. Input: car_id, date range, extra ids.
   Output: itemized quote (base rate × nights, weekend/season adjustments, duration
   discount, extras, booking fee, tax, total, deposit) + eligibility flags (car
   available? renter age ok? license on file?). Used by the app's booking flow, the
   MyTrips timeline, and the website widget — everyone shows the same numbers. Pure
   function over `rate_rules` + `policies` + `extras`; heavily unit-tested.
2. **`create-booking`** — input: car_id, date range, extras, customer info.
   Re-runs the quote server-side (never trust client totals), re-validates availability
   (the DB exclusion constraint is the final guard — handle its error as "just taken"),
   freezes the quote into `bookings.quote`, creates the Stripe PaymentIntent (rental
   amount) with the customer's `stripe_customer_id` so cards save for one-tap rebooking,
   and returns the client secret for PaymentSheet (Apple Pay enabled). Deposit is a
   separate manual-capture PaymentIntent created at pickup.
3. **`cancel-booking`** — applies `policies.cancellation_tiers` to compute the refund,
   issues the Stripe refund, cancels the booking (freeing the dates), notifies the
   customer with the itemized refund math, and — for direct bookings — reminds staff to
   unblock the dates on Turo.
4. **`turo-inbound`** — Postmark inbound webhook (verify shared secret). Parses trip
   booked/modified/canceled emails: extract guest name, car (match against `cars` by
   make/model/year with fuzzy fallback), dates, trip ref. Confident parse → upsert
   booking by `turo_trip_ref`. Not confident → `needs_review` row + staff push.
   Store every raw email in `turo_inbound_emails` regardless.
5. **`generate-contract`** — booking_id → merge template fields (including the itemized
   `{{quote_table}}`) → render PDF (`pdf-lib`) → store in `contracts` bucket → return
   signed URL. Triggered automatically when a booking is confirmed, so the customer can
   pre-sign.
6. **`finalize-contract`** — input: contract_id, signature PNG (base64), signer name.
   Stamps signature + timestamp block into the PDF, computes SHA-256 of final bytes,
   writes audit events, emails the signed PDF via Postmark, marks `signed`.
7. **`stripe-webhook`** — verifies signature; on `payment_intent.succeeded` confirms the
   booking, records `payments`, triggers confirmation email + push + Wallet pass offer.
   Enable **Stripe Radar** default rules; decline handling surfaces a clean retry in
   PaymentSheet.
8. **`send-push`** — APNs sender (token-based auth). Called by DB webhooks and the
   scheduled runner. Logs to `notifications_log`.
9. **`wallet-pass`** — generates an Apple Wallet PKPass for a confirmed booking (car,
   dates, pickup address, booking ref). Updates the pass if dates change.
10. **`scheduled-runner`** — invoked by pg_cron (hourly). One idempotent sweep
    (guarded by `notifications_log` uniqueness) that:
    - sends T-48h "license/signature missing", T-24h pickup reminder, T-3h return
      reminder;
    - flags **overdue returns** (past `end_at` + grace period, still `active`) to staff
      and the customer;
    - surfaces **maintenance due** (`car_services` by date or odometer) and
      registration/inspection expiry to staff;
    - expires stale `pending` bookings (created > 30 min ago, never paid) to free the
      dates;
    - **reconciles Stripe daily**: compares PaymentIntents against `payments` rows and
      flags any mismatch (missed webhook, orphaned hold, un-released deposit) to staff —
      money state can never silently drift.

## 9. iPhone app (SwiftUI)

- **Project:** XcodeGen `project.yml` (checked in; `.xcodeproj` generated, gitignored).
  Bundle id `com.bayrentals.app`. iOS 17 minimum. SPM deps: `supabase-swift`,
  `stripe-ios` (PaymentSheet), `sentry-cocoa`.
- **Auth:** Sign in with Apple + email OTP only (passwordless). Guest checkout: booking
  flow collects email + payment; the OTP that confirms the email simultaneously creates
  the account. OTP screens always offer resend-with-countdown and a "check spam / use a
  different email" fallback — never a dead end. After auth, fetch `profiles.role` — it
  drives the tab layout.
- **Design system first:** `DesignSystem/` holds color tokens (light + dark mode),
  type scale, spacing, reusable components (buttons, cards, sheets, empty states,
  skeleton loaders, toast-with-retry), and a haptics helper. Every screen is built from
  these — this is what makes the app feel like one product instead of forty screens.

### Navigation

- Customer tabs: **Browse** · **My Trips** · **Account**
- Staff/admin adds: **Today** · **Calendar** · **Ops**

### Customer experience

- `FleetListView` — active cars, hero photo, "from $X/day"; skeleton loading,
  pull-to-refresh.
- `CarDetailView` — swipeable gallery, specs, live availability calendar (Realtime —
  ranges gray out the moment anyone books), date-range picker, **live itemized quote**
  (from the `quote` function: nightly breakdown, discounts, extras, taxes — no
  surprises at checkout), extras picker, Book.
- `BookingFlowView` — dates + extras → contact (or Apple sign-in autofill) →
  **PaymentSheet with Apple Pay** → confirmation with confetti. Guest-friendly: email
  OTP creates the account inline. Saved cards make rebooking two taps.
- `MyTripsView` — upcoming/past. Each trip is a **timeline**: booked → license on file →
  contract signed → pickup (with instructions + map) → active → returned → receipt.
  Every incomplete step is tappable and completable in-app *before arrival*:
  - **License capture:** camera sheet scans the PDF417 barcode on the license back
    (Vision framework), auto-fills name/DOB/number/expiry, flags expired licenses and
    under-age renters, stores front photo. No typing.
  - **Pre-sign contract:** view PDF → `SignatureCanvasView` → done. Counter time
    collapses to photos + keys.
  - **Add to Apple Wallet** button once confirmed.
  - **Self-serve cancellation** with the refund math shown *before* confirming.
- Post-trip: receipt with any return charges itemized; happy path prompts an App Store
  rating (`SKStoreReviewController`) and links to review BayRentals on Google.
- `AccountView` — profile, saved license status, payment methods, past receipts.

### Staff experience

- **`TodayView`** (staff home) — the day at a glance: pickups and returns due today
  with one-tap entry into the wizards, overdue returns in red, Turo review-queue badge,
  maintenance flags. A staff member should open the app and know the whole day in five
  seconds.
- `OpsCalendarView` — month/week grid, all cars × bookings, color-coded by source
  (direct/turo/manual), **live via Realtime**. Tap to open; long-press to add a manual
  block.
- `BookingDetailView` — customer info, frozen quote, status, the **Turo block
  checklist** (deep link to Turo app, mark done), links to condition reports, contract,
  charges, and the two wizards:
- **`PickupWizardView`** — one linear guided flow, each step auto-advances:
  1. Verify license (scan barcode → match against record; skip if pre-verified)
  2. Deposit hold (Stripe manual-capture intent, one tap)
  3. Guided walkaround: front → rear → left → right → interior, camera stays open
     between slots, thumbnails confirm each shot; odometer + fuel recorded
  4. Contract: skip if pre-signed, else sign on device
  5. Hand over keys → booking flips to `active`, customer gets "You're on the road" push

  Both wizards **persist every completed step to disk immediately** — kill the app,
  drop the phone, switch to answer a call, reopen: the wizard resumes exactly where it
  left off. No step is ever redone, no photo ever re-taken.
- **`ReturnWizardView`** — mirror image: odometer + fuel → guided walkaround →
  **`DamageCompareView`** (checkout vs checkin photos side-by-side per angle, flag
  differences) → **auto-computed return charges** (mileage overage, missing fuel, late
  fee — from `policies`, itemized, staff can waive any line) → damage found? open a
  claim with the flagged photos → release or capture deposit → booking `completed`,
  receipt emailed with the same line items.
- **`ClaimsView`** — open claims list: attach estimate, charge (deposit capture or
  saved card via Stripe), waive, close. Every claim links its evidence photos.
- **`FleetView`** (in Ops) — per-car: status, upcoming services (due by date or miles,
  auto-flagged), registration/inspection expiry, service history; scheduling a service
  blocks the calendar.
- **`InsightsView`** (admin) — the owner dashboard, in the same app: utilization % per
  car, revenue per car per month, direct vs Turo mix, upcoming 7 days. Computed from a
  few SQL views — no analytics vendor.
- `TuroReviewQueueView` — `needs_review` emails: parsed guess shown, staff fixes
  car/dates, one tap creates the booking. Target: under 30 seconds per item.

### Cross-cutting implementation notes

- **Offline queue:** a disk-persisted operation queue (photos, status changes) with
  backoff retry and per-item status UI. Staff flows never block on connectivity.
- Camera: `AVFoundation` capture session (not the photo picker) for the guided
  walkaround; attach timestamp + GPS to each shot.
- Signature: SwiftUI `Canvas` accumulating stroke paths; export as PNG at 3× scale.
- **Universal links:** `apple-app-site-association` on Bayrentals.com so
  `bayrentals.com/book/<car>` and links in emails open the app when installed, web
  otherwise.
- Accessibility: Dynamic Type, VoiceOver labels on every interactive element — also an
  App Store review favorite.
- Crash/error reporting: Sentry from day one; a seamless app is one that doesn't
  silently break.

## 10. Pricing engine (single source of truth)

Lives in `supabase/functions/_shared/pricing.ts`. Pure and deterministic:
`(car, range, extras, rate_rules, policies) → itemized quote`. Rules:

- Base: nightly rate × nights (a "day" is a 24h period from pickup time).
- `rate_rules` applied by priority: weekend multipliers per-night, seasonal multipliers
  per-night, then the best matching duration discount (7+/28+ days) on the subtotal.
- Extras: `one_time` flat, `per_day` × days.
- Booking fee + tax from `policies` on top; deposit passed through from the car.
- Output is a `line_items[]` array with labels — the SAME array renders in the app
  quote screen, the contract's `{{quote_table}}`, the receipt email, and the widget.
- **Test-first:** this module gets exhaustive Deno unit tests (boundary days, DST
  transitions, overlapping season+weekend, discount thresholds) before any UI uses it.

Return-time charges (mileage overage, fuel, late fee) are computed by a sibling
function from the two condition reports + `policies`, with every line waivable by staff.

## 11. Notification matrix

| Event | Customer push | Customer email | Staff push |
|---|---|---|---|
| Booking confirmed (payment ok) | ✓ | ✓ (+ Wallet pass link) | ✓ |
| License or signature still missing, T-48h | ✓ | ✓ | — |
| Pickup reminder, T-24h (instructions) | ✓ | ✓ | — |
| Trip started (keys handed over) | ✓ | — | — |
| Return reminder, T-3h | ✓ | — | — |
| **Return overdue (past grace period)** | ✓ | ✓ | ✓ |
| Trip completed + receipt + deposit released | ✓ | ✓ | — |
| **Cancellation confirmed (with refund math)** | ✓ | ✓ | ✓ |
| **Claim charged (with evidence + receipt)** | ✓ | ✓ | — |
| New Turo booking parsed | — | — | ✓ |
| Turo email needs review | — | — | ✓ |
| New direct booking (block Turo reminder) | — | — | ✓ |
| **Maintenance / registration / inspection due** | — | — | ✓ |

All sends logged to `notifications_log`, which doubles as the scheduled runner's
idempotency guard — no event fires twice.

## 12. Engineering practices

### Environments

| Env | Backend | App |
|---|---|---|
| Local | `supabase start` (Docker) | Simulator, `Config.local.xcconfig` → local |
| Staging | Supabase project `bayrentals-staging`, Stripe test mode | TestFlight internal, bundle id suffix `.staging` |
| Prod | Supabase project `bayrentals`, Stripe live | App Store |

Migrations flow local → staging → prod via CI; never run by hand against prod.

### CI/CD (GitHub Actions)

- **On PR:** `supabase db reset` against a fresh Postgres (all migrations + `supabase/
  tests` RLS/constraint tests), Deno lint + unit tests for `_shared` and every function
  (pricing module has the deepest suite), `web-widget` typecheck/build.
- **On merge to main:** deploy functions + push migrations to **staging**
  automatically; prod deploy is a manually-triggered workflow (one green button).
- **iOS:** Xcode Cloud (free tier) builds on PR and ships merge-to-main to TestFlight
  internal. If Xcode Cloud fights back, fall back to fastlane on a macOS runner.

### Testing strategy

- **Database:** RLS per-role tests, exclusion-constraint tests, pricing SQL views.
- **Functions:** unit tests (pricing exhaustively; Turo parser against a corpus of
  fixture emails — collect real ones in `supabase/tests/fixtures/turo/` as they arrive,
  redacting personal data), plus one happy-path integration test per function.
- **iOS:** unit tests for view models and the offline queue; one XCUITest smoke per
  critical flow (book, pickup wizard, return wizard) run on PR via Xcode Cloud.
- **Manual:** each phase's acceptance list is a literal checklist run on a real device.

### Failure-mode catalog (the bulletproof standard)

Every one of these has a designed behavior — not an accident. Each gets a test or a
manual drill in its phase's acceptance list. When you discover a new failure mode
during the build, add it here with its designed behavior before fixing it.

| Failure | Designed behavior |
|---|---|
| Two customers book the same dates in the same second | DB exclusion constraint rejects the loser; app shows "just taken" with the next 3 available ranges, one tap to rebook |
| Payment declines mid-booking | `pending` booking holds the dates 30 min; PaymentSheet retry with another card/Apple Pay; auto-expires and frees dates if abandoned |
| Phone dies / app killed mid-walkaround | Wizard resumes at the exact step; captured photos already on disk in the upload queue |
| No signal at the counter (garage) | Entire pickup/return flow works offline except the deposit hold; queued work drains on reconnect with visible per-item status; deposit step clearly marked "needs signal" and can be done seconds later |
| Customer's OTP email never arrives | Resend with countdown; switch-email fallback; Sign in with Apple always available |
| Turo changes their email format | Parser confidence drops → items land in review queue (30-second manual fix), staff push fires; **never silently dropped** — raw email is always stored |
| Stripe webhook lost | Daily reconciliation sweep flags the mismatch to staff; booking confirmation also polls PaymentIntent status as a fallback on the confirmation screen |
| Deposit never released (human forgot) | Reconciliation flags holds older than booking end + 48h |
| Customer disputes damage weeks later | Timestamped, geotagged, slot-labeled photos from both walkarounds + hash-verified signed contract — the evidence pack is one tap in ClaimsView |
| Staff phone stolen | Passwordless sessions revocable from Supabase dashboard; licenses/contracts only ever displayed via short-lived signed URLs, nothing cached unencrypted |
| Old app version against new API | `app_config.min_supported_build` gates launch with a friendly update screen |
| Overdue return, customer unreachable | Escalating notifications (customer + staff) from the scheduled runner; late fees accrue per `policies`; booking flagged red on Today |
| A migration breaks staging | Prod deploy is a separate manual gate; staging soak is the drill; PITR backup restore is rehearsed quarterly |

### Security, privacy & data retention

- PII inventory: licenses, DOB, signatures, photos — all in private buckets, served
  only via short-lived signed URLs; never in logs or analytics.
- Contracts and condition photos are legal evidence: **never deleted, only
  void/superseded**. License photos: retention period is a `policies` value (attorney
  input, §15), enforced by the scheduled runner.
- Account deletion (App Store requirement): in-app request → anonymize profile, keep
  contractual/financial records as legally required.
- Supabase PITR backups on from day one (staging + prod); quarterly restore drill noted
  in `docs/SETUP.md`.
- Stripe Radar default rules on; deposits sized per car (`deposit_cents`) as the main
  fraud lever. (Optional stronger step is parked: §14.)
- Privacy nutrition labels + purpose strings (camera, location, notifications) written
  in Phase 6 from the PII inventory, not improvised in App Store Connect.

## 13. Build phases

**In one-shot mode (§0) these phases are the milestone structure, not the schedule:**
the waves build everything at once, and each phase's acceptance list becomes either an
automated verification gate (Wave 2/3) or a GO-LIVE.md checklist item (anything
needing real accounts or a physical device). If executing incrementally instead, work
the phases in order — each ends with something usable in production.

### Phase 0 — Dedicated repository + bootstrap
This project must live in its own clean private repo (never alongside unrelated code —
App Store secrets and CI live here).
1. `gh repo create bayrentals --private` (or create it in the GitHub UI).
2. Copy the contents of the `bayrentals/` directory (this file and everything beside
   it) to the new repo's **root**; initial commit; push.
3. Scaffold the repo layout from §6, `.gitignore` (xcodeproj, xcuserdata,
   `Config.local.xcconfig`, `.env`, `supabase/.temp`), CI skeleton from §12, and
   `docs/SETUP.md` + `docs/OPEN-ITEMS.md` + `docs/POLICIES.md` (placeholder business
   numbers marked TODO).
4. All further work happens in the new repo.
**Accept:** new repo's CI runs green on the empty skeleton; nothing references the old
repo.

### Phase 1 — Backend foundation + pricing engine
All migrations (schema incl. pricing/policy/ops tables, RLS, availability view, signup
trigger, Realtime on `bookings`, pg_cron schedule), storage buckets + policies, RLS
test suite, seed data (3 sample cars, rate rules, extras, default contract template,
placeholder policies). **Pricing module test-first** with its full unit suite; `quote`
function serving it.
**Accept:** CI green: migrations on fresh stack, all RLS tests, all pricing tests
(including DST and boundary cases); overlapping-booking insert rejected by the DB; a
booking INSERT visible over Realtime; `quote` returns correct itemized math for a
weekend + weekly-discount + extras scenario computed by hand.

### Phase 2 — iOS MVP (browse + book, no payments yet)
XcodeGen project, **design system first**, passwordless auth (Apple + email OTP),
role-based tabs, FleetList/CarDetail with live availability + live itemized quotes +
extras picker, BookingFlow creating `pending` bookings (guest checkout path included),
MyTrips timeline skeleton, staff Today screen (basic: today's pickups/returns) +
OpsCalendar with manual blocks and Realtime updates.
**Accept:** on-device demo: sign in with Apple, browse seeded cars, pick dates + an
extra and see the same line items the `quote` tests verify, book — the range grays out
on a second device *without refreshing*; double-book shows a friendly "just taken"
error; a first-time guest books with only email + OTP; stale pending bookings expire
and free the dates.

### Phase 3 — Counter experience (photos, contracts, license scan)
Condition-report flow with the persistent offline upload queue; license PDF417 scan +
auto-fill + age/expiry checks; `generate-contract` + `finalize-contract`; in-app
pre-signing from the trip timeline; PickupWizard and ReturnWizard end to end including
**auto-computed, waivable return charges**; DamageCompareView; claims opened from
return flow (charging comes in Phase 5).
**Accept:** full counter flow in airplane-mode-then-reconnect (photos queue and drain);
**kill the app mid-walkaround, reopen — wizard resumes at the same step with photos
intact**; license scan fills every field with zero typing, flags expired license and
under-age; a pre-signed customer's pickup takes under 2 minutes; return with 40 extra
miles and a quarter tank missing produces the exact charges `policies` math says, staff
waives one line; signed PDF has signature, timestamp, itemized quote table, and
matching SHA-256; customer receives the email.

### Phase 4 — Turo ingestion + website
`turo-inbound` parser (tested against the fixture corpus) + review queue UI + staff
push on new items; Turo block checklist on direct bookings; web widget (quote +
availability + book) embedded on Bayrentals.com; universal links live.
**Accept:** forwarded real Turo emails (booked/modified/canceled) create/update/cancel
the right blocks; malformed email lands in review queue and is resolved in-app in under
30 seconds; a website booking shows identical pricing to the app and blocks the app
calendar instantly; an emailed booking link opens the app.

### Phase 5 — Money end to end
Stripe PaymentSheet with **Apple Pay** + saved cards; deposit hold at pickup, release
or capture at return; return charges + claim charges actually collected (deposit
capture or saved card); `cancel-booking` with tiered refunds self-serve;
`stripe-webhook` with Radar; full notification matrix via `send-push` + scheduled
runner + Postmark templates; Apple Wallet pass.
**Accept:** in Stripe test mode: book with Apple Pay → deposit hold at pickup → return
with charges → partial deposit capture + remainder released — every movement visible
as `payments` rows and on the customer receipt; self-serve cancellation at T-48h
refunds the right tier percentage; every matrix event observed firing exactly once
(check `notifications_log`); Wallet pass installs and updates on a date change.

### Phase 6 — Fleet ops, insights & launch
FleetView (services, registration/inspection tracking, service-blocks calendar),
ClaimsView complete, InsightsView (utilization, revenue per car, direct/Turo mix),
post-trip review prompts; Sentry release tracking; account deletion flow; App Store
assets, privacy nutrition labels, purpose strings; TestFlight beta with real staff,
then submission.
**Accept:** a service due by odometer surfaces on Today and blocks the calendar when
scheduled; a claim goes open → charged with evidence attached; Insights numbers match
hand-computed SQL for the seed data; **the zero-training test: someone who has never
seen the app completes a full pickup and a full return unassisted**; every row of the
failure-mode catalog (§12) has been drilled at least once; TestFlight build used for
one real rental day without a blocker; App Store submission passes review.

## 14. Parking lot (post-launch, do not build now)

- Live Activity / Dynamic Island during active rental (return countdown)
- SMS reminders via Twilio (email + push first; add SMS only if no-shows happen)
- Web-based remote signing page for customers without the app
- Stripe Identity selfie-match verification, behind a per-car flag (luxury cars)
- Loyalty / repeat-renter discounts, promo codes
- Admin web dashboard (if Supabase Studio + staff app ever feel insufficient)
- Multi-location support, additional staff roles/permissions
- Home-screen widget (today's pickups/returns) and Siri App Intents
- Android (React Native or Kotlin — revisit demand after iOS launch)

## 15. Working agreements for Claude

- Supabase CLI for everything backend: `supabase start`, `supabase db reset`,
  `supabase functions serve`. Migrations are append-only — never edit one after merge.
- Never trust client-side prices, availability checks, or roles — recompute
  server-side; RLS and the exclusion constraint are the last line of defense, not the
  first. **All pricing math lives in `_shared/pricing.ts`; if a number appears anywhere
  else, that's a bug.**
- Photos and contracts are legal evidence: never delete, only void/supersede.
- Keep secrets out of the repo — `Config.swift` reads from a gitignored
  `Config.local.xcconfig`; document every required key in `docs/SETUP.md` as you go.
- Polish is part of done: every screen uses the design system, handles loading/empty/
  error states, and works in dark mode before its phase is accepted.
- Scheduled/notification work must be idempotent — `notifications_log` is the guard;
  design every sweep to be safe to run twice.
- When something needs Daniel (accounts, credentials, a decision), add it to
  `docs/OPEN-ITEMS.md` and continue with what's unblocked.

## 16. Open items for Daniel (not Claude)

1. Create accounts: Supabase project (staging + prod), Stripe (enable Apple Pay),
   Postmark, Apple Developer Program ($99/yr — start now, review can take days).
2. Set up email auto-forward of Turo notifications to the Postmark inbound address
   (Claude will give the exact address in Phase 4).
3. Send the current paper rental agreement to use as the contract template — and have
   an attorney bless the electronic version once (ESIGN-valid, but worth the check).
   Ask about license-photo retention period at the same time.
4. What platform does Bayrentals.com run on today? (Decides widget vs native page in
   Phase 4, and where `apple-app-site-association` gets hosted.)
5. **Business policy numbers** for `docs/POLICIES.md` (placeholders until then):
   minimum renter age; daily mileage caps per car (or unlimited); late fee + grace
   period; fuel charge; cancellation tiers; weekend/seasonal pricing; weekly and
   monthly discount percentages; deposit amounts per car; tax rate + booking fee.
6. Google Business review link (for the post-trip review prompt).
