# BayRentals Platform — Full Build Plan

> **How to use this document:** Paste it into Claude Code as your first prompt (or say
> "Read bayrentals/PLAN.md and start Phase 1"). It is a complete build spec: context,
> locked decisions, data model, API surface, iOS app structure, and phased tasks with
> acceptance criteria. Work through the phases in order. Each phase ends with something
> usable in production.

---

## 1. Context

You are building a full-stack rental car platform for **BayRentals** (Bayrentals.com), a
small rental car company whose fleet is also listed on **Turo**. The owner (Daniel,
daniel@dfnine.com) needs:

- An **iPhone app** used by BOTH customers (browse cars, book, pay) and staff
  (calendar, condition photos at pickup/return, contract signing at the counter).
  Role-based: same app, staff see extra tools.
- A **backend** holding the fleet, availability calendar, bookings, customer records,
  photo storage, and signed contracts.
- **Turo integration** (see the hard constraint below — this is email-ingestion based,
  not API based).
- **Bayrentals.com integration** — the website reads the same availability and booking
  API as the app.

## 2. Locked decisions (do not re-litigate)

| Decision | Choice |
|---|---|
| iOS app | **Native SwiftUI**, iPhone-first, iOS 17+ |
| Audience | **One app for both** customers and staff, role-based UI |
| Backend | **Supabase** (Postgres + Auth + Storage + Edge Functions) |
| E-signature | **Built-in**: finger-signing in app, self-generated PDF, own audit trail |
| Payments | **Stripe** — rental charge + refundable security-deposit hold |
| Transactional email | Postmark (or Resend) — also provides inbound parsing for Turo emails |
| Source of truth for availability | **Our database**, always. Turo bookings flow in; they never own the calendar. |

## 3. Hard constraint: Turo has no public API

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

## 4. Architecture

```
┌─────────────┐   ┌──────────────────┐   ┌─────────────────┐
│  iPhone app │   │  Bayrentals.com  │   │ Turo booking     │
│  (SwiftUI)  │   │  booking widget  │   │ emails (forward) │
└──────┬──────┘   └────────┬─────────┘   └────────┬────────┘
       │                   │                      │ Postmark inbound webhook
       ▼                   ▼                      ▼
┌─────────────────────────────────────────────────────────────┐
│                        SUPABASE                             │
│  Postgres (RLS)  ·  Auth  ·  Storage  ·  Edge Functions     │
│                                                             │
│  Functions: create-booking · turo-inbound · generate-       │
│  contract · finalize-contract · stripe-webhook              │
└──────────────┬──────────────────────────┬───────────────────┘
               ▼                          ▼
            Stripe                  Postmark (outbound
      (payments + deposits)         email: confirmations,
                                    signed contracts)
```

Admin/back-office: use **Supabase Studio** for raw data in early phases; a dedicated
admin web dashboard is Phase 5 (optional if the staff app covers daily needs).

## 5. Repository layout

```
bayrentals/
├── PLAN.md                  # this file
├── supabase/
│   ├── config.toml
│   ├── migrations/          # SQL migrations (numbered)
│   └── functions/
│       ├── create-booking/
│       ├── turo-inbound/
│       ├── generate-contract/
│       ├── finalize-contract/
│       └── stripe-webhook/
├── ios/
│   ├── project.yml          # XcodeGen spec (generates BayRentals.xcodeproj)
│   └── BayRentals/
│       ├── BayRentalsApp.swift
│       ├── Config.swift
│       ├── Models/
│       ├── Services/
│       └── Views/{Auth,Fleet,Booking,Staff,Account}/
├── web-widget/              # Phase 4: embeddable booking widget for Bayrentals.com
└── docs/
```

## 6. Data model (Postgres migrations)

Enable extensions: `btree_gist` (needed for the no-double-booking constraint), `pgcrypto`.

### Tables

**`profiles`** — extends `auth.users`
- `id uuid PK references auth.users`
- `role text check in ('admin','staff','customer') default 'customer'`
- `full_name text`, `phone text`
- Auto-created via trigger on auth signup.

**`cars`**
- `id uuid PK`, `slug text unique`
- `make, model, trim, color, license_plate, vin text`, `year int`
- `seats int`, `transmission text`, `fuel text`
- `daily_rate_cents int`, `deposit_cents int`
- `description text`, `status text check in ('active','maintenance','retired') default 'active'`
- `turo_listing_url text` (for the outbound deep-link checklist)

**`car_photos`**
- `id uuid PK`, `car_id FK`, `storage_path text`, `position int`

**`customers`** — walk-ins may have no auth account, so this is separate from profiles
- `id uuid PK`, `profile_id uuid FK nullable`
- `full_name, email, phone text`
- `license_number, license_state text`, `license_expiry date`
- `license_photo_path text` (private bucket)

**`bookings`** — one table for all three block types
- `id uuid PK`, `car_id FK`, `customer_id FK nullable` (null for turo/manual)
- `source text check in ('direct','turo','manual_block')`
- `status text check in ('pending','confirmed','active','completed','canceled')`
- `start_at, end_at timestamptz`, `daily_rate_cents, total_cents, deposit_cents int`
- `turo_trip_ref text nullable` (unique when not null — dedupes re-parsed emails)
- `turo_block_confirmed bool default false` (outbound checklist state for direct bookings)
- `notes text`, `created_by uuid`, `created_at timestamptz`
- **Critical constraint — no double booking, enforced by the database itself:**
  ```sql
  ALTER TABLE bookings ADD CONSTRAINT no_overlap
    EXCLUDE USING gist (car_id WITH =, tstzrange(start_at, end_at) WITH &&)
    WHERE (status NOT IN ('canceled'));
  ```

**`condition_reports`** — pickup/return walkarounds
- `id uuid PK`, `booking_id FK`, `kind text check in ('checkout','checkin')`
- `odometer int`, `fuel_level numeric`, `notes text`, `created_by uuid`

**`condition_photos`**
- `id uuid PK`, `report_id FK`, `storage_path text`
- `taken_at timestamptz`, `lat, lng numeric nullable` (EXIF-sourced — this is
  damage-dispute evidence, preserve capture metadata)

**`contract_templates`**
- `id uuid PK`, `name text`, `body_html text` with merge fields like
  `{{customer_name}}, {{car_display}}, {{start_at}}, {{end_at}}, {{daily_rate}},
  {{total}}, {{deposit}}`, `is_default bool`

**`contracts`**
- `id uuid PK`, `booking_id FK`, `template_id FK`
- `status text check in ('draft','signed','void')`
- `pdf_path text`, `signature_path text`
- `document_sha256 text` — hash of the final signed PDF
- `signed_at timestamptz`, `signer_name text`, `signer_ip inet`
- `audit jsonb` — append-only event list: generated, presented, signed, emailed

**`payments`**
- `id uuid PK`, `booking_id FK`
- `kind text check in ('rental','deposit_hold','deposit_capture','refund')`
- `stripe_payment_intent_id text`, `amount_cents int`, `status text`

**`turo_inbound_emails`** — raw ingestion log + review queue
- `id uuid PK`, `message_id text unique`, `from_email, subject text`, `raw_body text`
- `parse_status text check in ('parsed','needs_review','ignored')`
- `parsed jsonb`, `booking_id FK nullable`

### Row-level security

- `admin`/`staff` (checked via a `security definer` helper reading `profiles.role`):
  full access to everything.
- `customer`: read own `customers` row, own bookings, own contracts; no access to other
  customers, condition photos of others, or the Turo email log.
- `anon`: SELECT on active `cars`, `car_photos`, and an **`availability` view** exposing
  only `(car_id, start_at, end_at)` of non-canceled bookings — never customer data.
  This view powers the public website widget.

### Storage buckets

| Bucket | Access |
|---|---|
| `car-photos` | public read |
| `condition-photos` | staff only |
| `licenses` | staff only |
| `contracts` | staff + owning customer via signed URLs |

## 7. Edge functions

All in TypeScript (Deno). Secrets via `supabase secrets set`: `STRIPE_SECRET_KEY`,
`STRIPE_WEBHOOK_SECRET`, `POSTMARK_SERVER_TOKEN`, `POSTMARK_INBOUND_SECRET`.

1. **`create-booking`** — input: car_id, date range, customer info. Re-validates
   availability (the DB exclusion constraint is the final guard — handle its error as
   "just taken"), computes price server-side (never trust client totals), creates a
   `pending` booking + Stripe PaymentIntent (rental amount) and returns the client
   secret. Deposit is a separate manual-capture PaymentIntent created at pickup.
2. **`turo-inbound`** — Postmark inbound webhook (verify shared secret). Parses trip
   booked/modified/canceled emails: extract guest name, car (match against `cars` by
   make/model/year with fuzzy fallback), dates, trip ref. Confident parse → upsert
   booking by `turo_trip_ref`. Not confident → `needs_review` row; staff app surfaces it.
   Store every raw email in `turo_inbound_emails` regardless.
3. **`generate-contract`** — booking_id → merge template fields → render PDF (use
   `pdf-lib`; if HTML fidelity matters use a hosted HTML-to-PDF step) → store in
   `contracts` bucket → return signed URL for in-app display.
4. **`finalize-contract`** — input: contract_id, signature PNG (base64), signer name.
   Stamps signature + timestamp block into the PDF, computes SHA-256 of final bytes,
   writes audit events, emails the signed PDF to the customer via Postmark, marks
   `signed`.
5. **`stripe-webhook`** — verifies signature; on `payment_intent.succeeded` confirms the
   booking, records `payments`, emails confirmation.

## 8. iPhone app (SwiftUI)

- **Project:** define via XcodeGen `project.yml` (checked in; `.xcodeproj` is generated,
  gitignored). Bundle id `com.bayrentals.app`. iOS 17 minimum. Dependencies via SPM:
  `supabase-swift`, `stripe-ios` (PaymentSheet).
- **Auth:** Sign in with Apple (App Store requires it once any third-party login exists)
  + email/password. Session via supabase-swift. After auth, fetch `profiles.role` — it
  drives the tab layout.

### Navigation

- Customer tabs: **Browse** · **My Trips** · **Account**
- Staff/admin adds: **Calendar** · **Ops**

### Screens

**Customer:**
- `FleetListView` — active cars, photo, rate.
- `CarDetailView` — photo gallery, specs, availability calendar (booked ranges from the
  `availability` view), date-range picker, live price quote, Book.
- `BookingFlowView` — contact/license details → Stripe PaymentSheet → confirmation.
- `MyTripsView` — upcoming/past bookings, contract PDF viewer once signed.

**Staff:**
- `OpsCalendarView` — month/week grid, all cars × bookings, color-coded by source
  (direct/turo/manual). Tap to open; long-press to add a manual block.
- `BookingDetailView` — customer info, status transitions (confirm → active → completed),
  the **Turo block checklist** (deep link to Turo app, mark done), links to condition
  reports and contract.
- `ConditionReportView` — checkout/checkin flow: odometer, fuel, guided photo capture
  (front/rear/left/right/interior/extra). **Uploads must survive dead spots in a parking
  garage:** persist an upload queue to disk, retry with backoff, show per-photo status.
- `ContractSigningView` — render contract PDF → `SignatureCanvasView` (Canvas-based
  finger signing, clear/retry) → capture signer name → call `finalize-contract` →
  show "emailed to customer".
- `TuroReviewQueueView` — `needs_review` emails: show parsed guess, staff fixes
  car/dates, one tap to create the booking.

### Key implementation notes

- Camera: `AVFoundation` capture session (not the photo picker) for the guided
  walkaround; attach timestamp + GPS to each shot.
- Signature: SwiftUI `Canvas` accumulating stroke paths; export as PNG at 3× scale.
- Push (Phase 5): APNs via a `send-push` edge function; notify staff on new bookings
  and on Turo review-queue items.

## 9. Bayrentals.com widget (Phase 4)

A single embeddable `<script>` widget (vanilla TS, no framework — it must not fight the
host site): renders per-car availability calendar + "Book" flow, talking to the anon
`availability` view and `create-booking`. Works on any platform (WordPress, Squarespace,
custom). **Ask Daniel what Bayrentals.com currently runs on before building** — if it's
custom, a proper page using the same API beats an embed.

## 10. Build phases

Work in order. Commit per meaningful step. Each phase ends with the acceptance checks
passing.

### Phase 1 — Backend foundation
Supabase project setup (local dev via `supabase start`), all migrations (schema, RLS,
availability view, signup trigger, seed data: 3 sample cars, default contract template),
storage buckets + policies.
**Accept:** migrations apply cleanly on a fresh local stack; RLS verified (anon sees
availability but no customer data; customer sees only own rows); overlapping-booking
insert is rejected by the DB.

### Phase 2 — iOS MVP (browse + book, no payments yet)
XcodeGen project, auth (Apple + email), role-based tabs, FleetList/CarDetail with live
availability, BookingFlow creating `pending` bookings, MyTrips, staff OpsCalendar with
manual blocks.
**Accept:** on-device demo: sign in, browse seeded cars, book a range, see the range
blocked for everyone else; staff sees it on the calendar; double-book attempt is
rejected with a friendly error.

### Phase 3 — Photos + contracts
Condition-report flow with persistent upload queue; `generate-contract` +
`finalize-contract` functions; in-app signing; signed PDF emailed and archived.
**Accept:** full counter flow works offline-ish (photos queue and drain when signal
returns); signed PDF has signature, timestamp, and matching SHA-256 in `contracts`;
customer receives the email.

### Phase 4 — Turo ingestion + website
`turo-inbound` parser + review queue UI; Turo block checklist on direct bookings;
web widget embedded on Bayrentals.com.
**Accept:** forwarded real Turo emails (booked/modified/canceled) create/update/cancel
the right blocks; malformed email lands in review queue, staff resolves it in-app in
under 30 seconds; a booking made on the website blocks the app calendar instantly.

### Phase 5 — Payments + launch
Stripe PaymentSheet in booking flow, deposit hold at pickup / release or capture at
return, `stripe-webhook`, push notifications, App Store assets + review submission,
TestFlight beta first.
**Accept:** end-to-end money flow in Stripe test mode including deposit
hold-and-release; app passes App Store review guidelines checklist (Sign in with Apple,
privacy nutrition labels, camera/location purpose strings).

## 11. Working agreements for Claude

- Supabase CLI for everything backend: `supabase start`, `supabase db reset`,
  `supabase functions serve`. Write at least smoke tests for edge functions (Deno test).
- Never trust client-side prices, availability checks, or roles — recompute server-side;
  RLS and the exclusion constraint are the last line of defense, not the first.
- Photos and contracts are legal evidence: never delete, only void/supersede.
- Keep secrets out of the repo — `Config.swift` reads from a gitignored
  `Config.local.xcconfig`; document every required key in `docs/SETUP.md` as you go.
- When something needs Daniel (accounts, credentials, a decision), add it to a running
  `docs/OPEN-ITEMS.md` and continue with what's unblocked.

## 12. Open items for Daniel (not Claude)

1. Create accounts: Supabase project, Stripe, Postmark, Apple Developer Program
   ($99/yr — start now, review can take days).
2. Set up email auto-forward of Turo notifications to the Postmark inbound address
   (Claude will give the exact address in Phase 4).
3. Send the current paper rental agreement to use as the contract template — and have
   an attorney bless the electronic version once (ESIGN-valid, but worth the check).
4. What platform does Bayrentals.com run on today? (Decides widget vs native page in
   Phase 4.)
5. Long-term: move this project into its own dedicated repository.
