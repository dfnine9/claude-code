# BayRentals Platform — Full Build Plan (v2)

> **How to use this document:** Paste it into Claude Code as your first prompt (or say
> "Read PLAN.md and start Phase 0"). It is a complete build spec: context, locked
> decisions, seamlessness principles, data model, API surface, iOS app structure, and
> phased tasks with acceptance criteria. Work through the phases in order. Each phase
> ends with something usable in production.

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
- **Turo integration** (see the hard constraint below — email-ingestion based, not API
  based).
- **Bayrentals.com integration** — the website reads the same availability and booking
  API as the app.

The bar is **seamless**: renting a car from BayRentals should feel as smooth as an
Apple-native experience — for the customer AND for staff at the counter.

## 2. Locked decisions (do not re-litigate)

| Decision | Choice |
|---|---|
| iOS app | **Native SwiftUI**, iPhone-first, iOS 17+ |
| Audience | **One app for both** customers and staff, role-based UI |
| Backend | **Supabase** (Postgres + Auth + Storage + Edge Functions + Realtime) |
| Auth | **Passwordless only**: Sign in with Apple + email OTP. No passwords, ever. |
| E-signature | **Built-in**: finger-signing in app, self-generated PDF, own audit trail |
| Payments | **Stripe** — PaymentSheet with **Apple Pay**, rental charge + refundable deposit hold |
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
7. **The app talks first.** Push + email at every lifecycle moment (matrix in §10);
   the customer never wonders what's next.

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
│  Postgres (RLS) · Auth · Storage · Realtime · Functions     │
│                                                             │
│  Functions: create-booking · turo-inbound · generate-       │
│  contract · finalize-contract · stripe-webhook · send-push  │
│  · wallet-pass                                              │
└──────────────┬──────────────────────────┬───────────────────┘
               ▼                          ▼
            Stripe                  Postmark (outbound
   (Apple Pay, payments,            email: confirmations,
    deposit holds)                  signed contracts)
```

Admin/back-office: use **Supabase Studio** for raw data in early phases; a dedicated
admin web dashboard is optional later if the staff app covers daily needs.

## 6. Repository layout (dedicated repo — see Phase 0)

```
bayrentals/                  # repo ROOT (its own private GitHub repo)
├── PLAN.md                  # this file
├── supabase/
│   ├── config.toml
│   ├── migrations/          # SQL migrations (numbered)
│   └── functions/
│       ├── create-booking/
│       ├── turo-inbound/
│       ├── generate-contract/
│       ├── finalize-contract/
│       ├── stripe-webhook/
│       ├── send-push/
│       └── wallet-pass/
├── ios/
│   ├── project.yml          # XcodeGen spec (generates BayRentals.xcodeproj)
│   └── BayRentals/
│       ├── BayRentalsApp.swift
│       ├── Config.swift
│       ├── DesignSystem/    # colors, type scale, components, haptics
│       ├── Models/
│       ├── Services/
│       └── Views/{Auth,Fleet,Booking,Trips,Staff,Account}/
├── web-widget/              # Phase 4: embeddable booking widget for Bayrentals.com
└── docs/
```

## 7. Data model (Postgres migrations)

Enable extensions: `btree_gist` (needed for the no-double-booking constraint), `pgcrypto`.

### Tables

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
- `description text`, `status text check in ('active','maintenance','retired') default 'active'`
- `pickup_instructions text` (shown in the customer trip timeline)
- `turo_listing_url text` (for the outbound deep-link checklist)

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
- Realtime enabled on this table (powers live calendars everywhere).

**`condition_reports`** — pickup/return walkarounds
- `id uuid PK`, `booking_id FK`, `kind text check in ('checkout','checkin')`
- `odometer int`, `fuel_level numeric`, `notes text`, `created_by uuid`

**`condition_photos`**
- `id uuid PK`, `report_id FK`, `storage_path text`
- `slot text check in ('front','rear','left','right','interior','extra')` — enables
  side-by-side checkout-vs-checkin comparison per angle
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

**`notifications_log`** — every push/email sent, for debugging "did they get it?"
- `id uuid PK`, `booking_id FK nullable`, `recipient text`, `channel text`,
  `event text`, `sent_at timestamptz`

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

## 8. Edge functions

All in TypeScript (Deno). Secrets via `supabase secrets set`: `STRIPE_SECRET_KEY`,
`STRIPE_WEBHOOK_SECRET`, `POSTMARK_SERVER_TOKEN`, `POSTMARK_INBOUND_SECRET`,
`APNS_KEY` (+ team/key ids), `WALLET_PASS_CERT`.

1. **`create-booking`** — input: car_id, date range, customer info. Re-validates
   availability (the DB exclusion constraint is the final guard — handle its error as
   "just taken"), computes price server-side (never trust client totals), creates a
   `pending` booking + Stripe PaymentIntent (rental amount) with the customer's
   `stripe_customer_id` so cards save for one-tap rebooking, and returns the client
   secret for PaymentSheet (Apple Pay enabled). Deposit is a separate manual-capture
   PaymentIntent created at pickup.
2. **`turo-inbound`** — Postmark inbound webhook (verify shared secret). Parses trip
   booked/modified/canceled emails: extract guest name, car (match against `cars` by
   make/model/year with fuzzy fallback), dates, trip ref. Confident parse → upsert
   booking by `turo_trip_ref`. Not confident → `needs_review` row + staff push
   notification. Store every raw email in `turo_inbound_emails` regardless.
3. **`generate-contract`** — booking_id → merge template fields → render PDF (use
   `pdf-lib`) → store in `contracts` bucket → return signed URL for in-app display.
   Triggered automatically when a booking is confirmed, so the customer can pre-sign.
4. **`finalize-contract`** — input: contract_id, signature PNG (base64), signer name.
   Stamps signature + timestamp block into the PDF, computes SHA-256 of final bytes,
   writes audit events, emails the signed PDF to the customer via Postmark, marks
   `signed`.
5. **`stripe-webhook`** — verifies signature; on `payment_intent.succeeded` confirms the
   booking, records `payments`, triggers confirmation email + push + Wallet pass offer.
6. **`send-push`** — APNs sender (token-based auth). Called by DB webhooks on the
   events in the notification matrix (§10). Logs to `notifications_log`.
7. **`wallet-pass`** — generates an Apple Wallet PKPass for a confirmed booking (car,
   dates, pickup address, booking ref). Updates the pass if dates change.

## 9. iPhone app (SwiftUI)

- **Project:** XcodeGen `project.yml` (checked in; `.xcodeproj` generated, gitignored).
  Bundle id `com.bayrentals.app`. iOS 17 minimum. SPM deps: `supabase-swift`,
  `stripe-ios` (PaymentSheet), `sentry-cocoa`.
- **Auth:** Sign in with Apple + email OTP only (passwordless). Guest checkout: booking
  flow collects email + payment; the OTP that confirms the email simultaneously creates
  the account. After auth, fetch `profiles.role` — it drives the tab layout.
- **Design system first:** `DesignSystem/` holds color tokens (light + dark mode),
  type scale, spacing, reusable components (buttons, cards, sheets, empty states,
  skeleton loaders, toast-with-retry), and a haptics helper. Every screen is built from
  these — this is what makes the app feel like one product instead of forty screens.

### Navigation

- Customer tabs: **Browse** · **My Trips** · **Account**
- Staff/admin adds: **Calendar** · **Ops**

### Customer experience

- `FleetListView` — active cars, hero photo, rate; skeleton loading, pull-to-refresh.
- `CarDetailView` — swipeable gallery, specs, live availability calendar (Realtime —
  ranges gray out the moment anyone books), date-range picker, live price quote, Book.
- `BookingFlowView` — dates → contact (or Apple sign-in autofill) → **PaymentSheet with
  Apple Pay** → confirmation with confetti. Guest-friendly: email OTP creates the
  account inline. Saved cards make rebooking two taps.
- `MyTripsView` — upcoming/past. Each trip is a **timeline**: booked → license on file →
  contract signed → pickup (with instructions + map) → active → returned → receipt.
  Every incomplete step is tappable and completable in-app *before arrival*:
  - **License capture:** camera sheet scans the PDF417 barcode on the license back
    (Vision framework), auto-fills name/DOB/number/expiry, flags expired licenses,
    stores front photo. No typing.
  - **Pre-sign contract:** view PDF → `SignatureCanvasView` → done. Counter time
    collapses to photos + keys.
  - **Add to Apple Wallet** button once confirmed.
- `AccountView` — profile, saved license status, payment methods, past receipts.

### Staff experience

- `OpsCalendarView` — month/week grid, all cars × bookings, color-coded by source
  (direct/turo/manual), **live via Realtime**. Tap to open; long-press to add a manual
  block. Badge shows Turo review-queue count.
- `BookingDetailView` — customer info, status, the **Turo block checklist** (deep link
  to Turo app, mark done), links to condition reports and contract, and the two wizards:
- **`PickupWizardView`** — one linear guided flow, each step auto-advances:
  1. Verify license (scan barcode → match against record; skip if pre-verified)
  2. Deposit hold (Stripe manual-capture intent, one tap)
  3. Guided walkaround: front → rear → left → right → interior, camera stays open
     between slots, thumbnails confirm each shot
  4. Contract: skip if pre-signed, else sign on device
  5. Hand over keys → booking flips to `active`, customer gets "You're on the road" push
- **`ReturnWizardView`** — mirror image: odometer + fuel → guided walkaround →
  **`DamageCompareView`** (checkout vs checkin photos side-by-side per angle, flag
  differences) → release or capture deposit → booking `completed`, receipt emailed.
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

## 10. Notification matrix

| Event | Customer push | Customer email | Staff push |
|---|---|---|---|
| Booking confirmed (payment ok) | ✓ | ✓ (+ Wallet pass link) | ✓ |
| License or signature still missing, T-48h | ✓ | ✓ | — |
| Pickup reminder, T-24h (instructions) | ✓ | ✓ | — |
| Trip started (keys handed over) | ✓ | — | — |
| Return reminder, T-3h | ✓ | — | — |
| Trip completed + receipt + deposit released | ✓ | ✓ | — |
| New Turo booking parsed | — | — | ✓ |
| Turo email needs review | — | — | ✓ |
| New direct booking (block Turo reminder) | — | — | ✓ |

All sends logged to `notifications_log`.

## 11. Bayrentals.com widget (Phase 4)

A single embeddable `<script>` widget (vanilla TS, no framework — it must not fight the
host site): renders per-car availability calendar + "Book" flow, talking to the anon
`availability` view and `create-booking`. Works on any platform (WordPress, Squarespace,
custom). **Ask Daniel what Bayrentals.com currently runs on before building** — if it's
custom, a proper page using the same API beats an embed. The site also hosts
`apple-app-site-association` for universal links.

## 12. Build phases

Work in order. Commit per meaningful step. Each phase ends with the acceptance checks
passing.

### Phase 0 — Dedicated repository + bootstrap
This project must live in its own clean private repo (never alongside unrelated code —
App Store secrets and CI live here).
1. `gh repo create bayrentals --private` (or create it in the GitHub UI).
2. Copy the contents of the `bayrentals/` directory (this file and everything beside
   it) to the new repo's **root**; initial commit; push.
3. Scaffold the repo layout from §6, `.gitignore` (xcodeproj, xcuserdata,
   `Config.local.xcconfig`, `.env`, `supabase/.temp`), and `docs/SETUP.md` listing every
   account/secret needed as they come up.
4. All further work happens in the new repo.
**Accept:** new repo builds the empty skeleton; nothing references the old repo.

### Phase 1 — Backend foundation
Supabase project setup (local dev via `supabase start`), all migrations (schema, RLS,
availability view, signup trigger, Realtime enabled on `bookings`, seed data: 3 sample
cars, default contract template), storage buckets + policies.
**Accept:** migrations apply cleanly on a fresh local stack; RLS verified (anon sees
availability but no customer data; customer sees only own rows); overlapping-booking
insert is rejected by the DB; a booking INSERT is visible over a Realtime subscription.

### Phase 2 — iOS MVP (browse + book, no payments yet)
XcodeGen project, **design system first**, passwordless auth (Apple + email OTP),
role-based tabs, FleetList/CarDetail with live availability, BookingFlow creating
`pending` bookings (guest checkout path included), MyTrips timeline skeleton, staff
OpsCalendar with manual blocks and Realtime updates.
**Accept:** on-device demo: sign in with Apple, browse seeded cars, book a range —
the range grays out on a second device *without refreshing*; double-book attempt shows
a friendly "just taken" error; a first-time guest books with only email + OTP.

### Phase 3 — Counter experience (photos, contracts, license scan)
Condition-report flow with the persistent offline upload queue; license PDF417 scan +
auto-fill; `generate-contract` + `finalize-contract`; in-app pre-signing from the trip
timeline; PickupWizard and ReturnWizard end to end; DamageCompareView.
**Accept:** full counter flow in airplane-mode-then-reconnect (photos queue and drain);
license scan fills every field with zero typing and flags an expired license; a
pre-signed customer's pickup takes under 2 minutes; signed PDF has signature, timestamp,
and matching SHA-256; customer receives the email.

### Phase 4 — Turo ingestion + website
`turo-inbound` parser + review queue UI + staff push on new items; Turo block checklist
on direct bookings; web widget embedded on Bayrentals.com; universal links live.
**Accept:** forwarded real Turo emails (booked/modified/canceled) create/update/cancel
the right blocks; malformed email lands in review queue and is resolved in-app in under
30 seconds; a website booking blocks the app calendar instantly; an emailed booking
link opens the app.

### Phase 5 — Payments, notifications, launch
Stripe PaymentSheet with **Apple Pay** + saved cards; deposit hold at pickup, release or
capture at return; `stripe-webhook`; full notification matrix via `send-push` + Postmark
templates; Apple Wallet pass; Sentry release tracking; App Store assets, privacy
nutrition labels, purpose strings (camera, location); TestFlight beta with real staff,
then submission.
**Accept:** end-to-end money flow in Stripe test mode including deposit hold-and-release
and Apple Pay; every matrix event observed firing once (check `notifications_log`);
Wallet pass installs and updates on a date change; TestFlight build used for one real
rental day without a blocker.

## 13. Parking lot (post-launch, do not build now)

- Live Activity / Dynamic Island during active rental (return countdown)
- SMS reminders via Twilio (email + push first; add SMS only if no-shows happen)
- Web-based remote signing page for customers without the app
- Loyalty / repeat-renter discounts, promo codes
- Admin web dashboard (if Supabase Studio + staff app ever feel insufficient)
- Multi-location support, additional staff roles/permissions
- Android (React Native or Kotlin — revisit demand after iOS launch)

## 14. Working agreements for Claude

- Supabase CLI for everything backend: `supabase start`, `supabase db reset`,
  `supabase functions serve`. Write at least smoke tests for edge functions (Deno test).
- Never trust client-side prices, availability checks, or roles — recompute server-side;
  RLS and the exclusion constraint are the last line of defense, not the first.
- Photos and contracts are legal evidence: never delete, only void/supersede.
- Keep secrets out of the repo — `Config.swift` reads from a gitignored
  `Config.local.xcconfig`; document every required key in `docs/SETUP.md` as you go.
- Polish is part of done: every screen uses the design system, handles loading/empty/
  error states, and works in dark mode before its phase is accepted.
- When something needs Daniel (accounts, credentials, a decision), add it to a running
  `docs/OPEN-ITEMS.md` and continue with what's unblocked.

## 15. Open items for Daniel (not Claude)

1. Create accounts: Supabase project, Stripe (enable Apple Pay), Postmark, Apple
   Developer Program ($99/yr — start now, review can take days).
2. Set up email auto-forward of Turo notifications to the Postmark inbound address
   (Claude will give the exact address in Phase 4).
3. Send the current paper rental agreement to use as the contract template — and have
   an attorney bless the electronic version once (ESIGN-valid, but worth the check).
4. What platform does Bayrentals.com run on today? (Decides widget vs native page in
   Phase 4, and where `apple-app-site-association` gets hosted.)
