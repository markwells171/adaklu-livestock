# Adaklu Livestock Market — Build Spec (v1)

**Operator:** Honeyard Agribusiness
**Domain:** adaklulivestock.com (DNS at Namecheap, currently pointed to Netlify)
**Contacts:** 055 682 3692 (calls + WhatsApp), 053 680 0950 (calls only)

This spec is for Claude Code. Build in the session order in section 11. Track progress in `PROGRESS.md` and use `feat:` commit messages that reference the session number (e.g. `feat(S3): seller listing form`).

---

## 1. What we are building

An online marketplace for farm animals sourced from the Adaklu livestock market, plus related services (butchering, refrigeration, transport). Honeyard Agribusiness runs it and is the trusted middle party.

- **Buyers** browse without signing up. They contact Honeyard on WhatsApp or place a Safe Buy order / service request through a form.
- **Sellers** (farmers, traders) and **service providers** (butchers, transport owners, cold storage) sign up with their phone number, get verified by admin, and post listings with photos.
- **Admin / agents** approve sellers and listings, post listings on behalf of sellers who can't (agent-assisted listing), and run Safe Buy orders and service requests to completion.

**Public contact rule (v1):** buyers never see a seller's phone number. Every listing's call-to-action routes to Honeyard's numbers. Seller phone numbers are visible only in admin.

### Design for multiple markets and product lines from day one
Honeyard will later add other markets (other towns) and other foodstuffs (yam, fish, grains). Model these as data, not code:
- `markets` (Adaklu is the first row)
- `product_lines` (Livestock is the first row)
- `item_types` (Cattle, Goats, … belong to a product line)
A future "Ho Yam Market" is new rows plus a domain mapping, not a rebuild. Resolve the current market from the request hostname (`adaklulivestock.com` → Adaklu market) with a fallback default.

---

## 2. Stack

| Layer | Choice |
|---|---|
| Frontend | Next.js 14 (App Router), TypeScript, Tailwind. Deploy on **Netlify** (domain already connected there). Vercel works too if DNS is moved. |
| Backend | Node.js + Express, TypeScript. Deploy on **Railway** at `api.adaklulivestock.com`. |
| Database | PostgreSQL on Railway. Migrations with a migration tool (node-pg-migrate or Prisma Migrate — pick one and stay with it). |
| SMS / OTP | **Nalo Solutions** SMS API |
| Photos | **Cloudinary** (signed uploads, auto format/quality, resized delivery) |
| Payments | **Manual MoMo** in v1. Buyer pays Honeyard's MoMo; admin records and releases payments in the panel. Design the order model so Hubtel checkout can be added later. |

Monorepo layout:
```
/web        Next.js app (public site, seller area, admin panel)
/api        Express API
/db         migrations + seed
PROGRESS.md
SPEC.md
```

---

## 3. Users and roles

| Role | Who | Can do |
|---|---|---|
| Public buyer | Anyone, no account | Browse listings, view details, submit Safe Buy order, submit service request, WhatsApp/call Honeyard |
| Seller | Farmer or trader | Manage own profile, create/edit own listings (go to review), mark own listing sold/unavailable |
| Service provider | Butcher, transport owner, cold storage | Manage profile and list their service offering (reviewed by admin) |
| Agent | Honeyard field staff at Adaklu | Create listings on behalf of a seller, upload photos, record inspections on Safe Buy orders |
| Admin | Honeyard staff | Everything agents can, plus approve/reject sellers and listings, manage orders, payments, services, content |
| Super admin | Mawulolo | Everything, plus manage staff accounts, markets, product lines, item types, settings |

One person can hold more than one seller-side role (e.g. a trader who also transports). Store roles as a set.

---

## 4. Authentication

### Sellers and service providers: phone + SMS OTP
1. User enters phone number. Normalise Ghana numbers: accept `0556823692`, `556823692`, `+233556823692`, `233556823692` → store as E.164 `+233556823692`. Reject anything that isn't a valid Ghana mobile number.
2. API generates a 6-digit code, stores a **hash** of it with 5-minute expiry, sends via Nalo.
3. User enters code → API verifies → issues session.
4. First login creates the account in `pending_verification` state and asks for profile details (section 5.2).

Rules:
- Max 3 OTP sends per phone per 15 minutes; max 5 wrong attempts per code, then the code is dead.
- Rate-limit by IP as well.
- OTP SMS text: `Your Adaklu Livestock Market code is 123456. It expires in 5 minutes. Do not share it.`
- Sessions: httpOnly, secure, SameSite=Lax cookie holding a session id; sessions stored in Postgres; 30-day sliding expiry for sellers.

### Staff (agents, admins, super admin)
Same phone + OTP flow, but staff accounts are created only by super admin; a staff phone number that isn't pre-registered cannot log in to admin. Staff sessions expire after 12 hours.

### Dev mode
In development, log OTP codes to the console instead of sending SMS (env flag `SMS_MODE=console`).

---

## 5. Features

### 5.1 Public site (no login)
- **Home:** hero, contact card (both numbers, WhatsApp button), featured listings, item-type filter chips, services overview, how Safe Buy works, "Sell with us" call-to-action. Keep the look of the current live page (green/sun palette, Bricolage Grotesque + Figtree).
- **Listings page:** grid of approved, available listings. Filters: item type, price range, sex, location/community. Sort: newest, price low→high, high→low. Paginate (24 per page).
- **Listing detail:** photo gallery (up to 6 photos + optional short video link), item type, breed, sex, age, estimated weight, price (or "Ask for price"), quantity available, location (community only), listing reference code (e.g. `ALM-0142`). Buttons: **Order with Safe Buy**, **WhatsApp us about this** (pre-filled message containing the reference code), **Call**.
- **Safe Buy order form:** name, phone, item (pre-filled if from a listing, else item type + description), quantity, add-on services (butchering, refrigeration, transport — checkboxes), delivery location, needed-by date, notes. On submit: create order, show order reference (e.g. `SB-2611-0007`), SMS the buyer their reference, SMS the admin alert number.
- **Service request form:** same pattern for butchering / refrigeration / transport only (no animal purchase).
- **Track my order:** buyer enters order reference + phone → sees order status timeline (no account needed).
- **Sell with us / sign up:** explains benefits, starts the OTP flow.
- **Static pages:** About Honeyard Agribusiness, How Safe Buy works, Terms, Privacy, Contact.
- SEO: per-page titles/descriptions, Open Graph image per listing (first photo), sitemap.xml, robots.txt.
- Performance: must work well on low-end Android phones on 3G. Cloudinary resized images, lazy loading, no heavy client libraries on public pages.

### 5.2 Seller area (logged in)
- **Profile:** full name, business name (optional), roles (farmer / trader / butcher / transport owner / cold storage), WhatsApp number if different, town/community, how often at Adaklu market, item types dealt in, profile photo (optional). Ownership/authorisation declaration checkbox (required).
- **Verification status banner:** "Pending verification — Honeyard will call you" / "Verified" / "Rejected (reason)". Pending sellers can draft listings but nothing goes public until both seller and listing are approved.
- **My listings:** create, edit, duplicate, mark sold, mark unavailable, delete draft.
- **Listing form:** item type, title, breed (optional), sex (male / female / mixed / n/a), age (free text), estimated weight kg (optional), quantity, price per unit in GH₵ (optional — allow "ask for price"), location/community, description, photos (min 1, max 6; upload from phone camera; compressed client-side before upload).
- Any edit to an approved listing's photos, price or description sends it back to review (configurable setting; default on).
- **Service providers:** a "My service" page instead of / in addition to listings: service type, coverage area, rates description, capacity (e.g. vehicle type, cold room size).

### 5.3 Admin panel
- **Dashboard:** counts needing action (sellers pending, listings pending, new Safe Buy orders, new service requests, payments awaiting release), recent activity.
- **Sellers:** list, search by name/phone, filter by status/role. Detail view shows profile, listings, orders involving them, internal notes. Actions: verify, reject (with reason, SMS sent), suspend, add note.
- **Listings:** review queue with photos large. Actions: approve, reject (reason), request changes, feature on home page, edit, mark sold. Agents can **create a listing on behalf of a seller** (pick or create the seller record; agent-created sellers can later claim the account by logging in with that phone).
- **Safe Buy orders:** list with status filters; detail view with full timeline (section 6), buyer info, linked listing, assigned agent, inspection notes + photos, agreed price, payments, add-on services and their costs, internal notes. Each status change logs who/when.
- **Service requests:** same pattern, simpler status flow.
- **Payments ledger:** every money movement recorded by hand: buyer → Honeyard (amount, MoMo reference, date), Honeyard → seller, Honeyard → service provider, refunds. Show per-order balance and a simple daily/weekly summary.
- **Catalogue settings (super admin):** markets, product lines, item types (name, icon, sort order, active), services, fee settings.
- **Staff (super admin):** add/remove agents and admins by phone number.
- **Audit log:** append-only record of every admin/agent action (who, what, before/after, when). Read-only in the UI.

### 5.4 Notifications (SMS via Nalo)
| Event | To | Message gist |
|---|---|---|
| Seller verified / rejected | Seller | Status + next step |
| Listing approved / rejected / changes requested | Seller | Listing ref + status |
| New Safe Buy order / service request | Admin alert number(s) | Ref, item, buyer phone |
| Order status changes that matter to the buyer | Buyer | Ref + new status + track link |
| Seller's animal sold through Safe Buy | Seller | Ref + amount + payout note |

Keep all SMS templates in one config file. Log every SMS sent (to, template, status, provider message id).

---

## 6. Safe Buy order lifecycle

Model as a state machine (same approach as GuardPay: allowed transitions table + audit trail). Every transition writes a row to `order_events`.

```
new → contacted → sourcing → inspected → awaiting_payment → paid → purchased
    → [processing]  (butchering / refrigeration, if add-ons)
    → in_transit → delivered → completed
Any active state → cancelled (reason required)
paid/purchased → refund_pending → refunded
```

| Status | Meaning | Who moves it |
|---|---|---|
| new | Order submitted | System |
| contacted | Honeyard has called the buyer and confirmed needs | Admin |
| sourcing | Agent looking for / reserving the animal | Admin/agent |
| inspected | Agent inspected; notes, photos, agreed price recorded; buyer sent photos | Agent |
| awaiting_payment | Total quoted to buyer (animal + add-ons + fee) | Admin |
| paid | Buyer's MoMo payment received and recorded | Admin |
| purchased | Honeyard paid seller; animal in Honeyard's hands | Admin/agent |
| processing | Butchering / chilling in progress | Admin/agent |
| in_transit | Dispatched; driver name + phone recorded | Admin/agent |
| delivered | Handed over; who received it recorded | Admin/agent |
| completed | Buyer confirmed / closed | Admin |
| cancelled | Stopped, with reason | Admin |
| refund_pending / refunded | Money being / been returned | Admin |

Order pricing fields (all money stored as integer **pesewas**): `animal_price`, `addon_costs` (per service line), `honeyard_fee`, `transport_cost`, `total_quoted`, `total_paid`, `paid_to_seller`, `paid_to_providers`. Fee rule is a setting (flat amount or % of animal price) so it can change without code.

Service requests use a shorter flow: `new → contacted → scheduled → in_progress → done → completed`, plus `cancelled`.

---

## 7. Data model (PostgreSQL)

All tables: `id` (uuid or bigserial — pick one), `created_at`, `updated_at`. Money in integer pesewas. Phones in E.164.

- `markets` — name, slug, region, district, primary_domain, active
- `product_lines` — name, slug, active
- `item_types` — product_line_id, name, slug, unit_label ("per head", "per bird"), sort_order, active
- `services` — name, slug (butchering, refrigeration, transport, safe_buy), description, active
- `users` — phone (unique), full_name, business_name, whatsapp_phone, community, market_id, market_frequency, status (`pending_verification | verified | rejected | suspended`), rejection_reason, declaration_accepted_at, created_by_staff_id (agent-assisted), claimed_at
- `user_roles` — user_id, role (`seller | trader | butcher | transporter | cold_storage | agent | admin | super_admin`)
- `user_item_types` — user_id, item_type_id
- `otp_codes` — phone, code_hash, purpose, expires_at, attempts, used_at
- `sessions` — user_id, token_hash, expires_at, ip, user_agent
- `listings` — ref_code (unique, e.g. ALM-0142), market_id, seller_id, item_type_id, title, breed, sex, age_text, est_weight_kg, quantity, price_pesewas (nullable = ask for price), community, description, status (`draft | pending_review | approved | rejected | changes_requested | sold | unavailable`), rejection_reason, featured, approved_by, approved_at, created_by_staff_id
- `listing_photos` — listing_id, cloudinary_public_id, url, sort_order
- `service_offerings` — provider_id, service_id, coverage_area, rates_text, capacity_text, status
- `orders` — ref_code (unique, e.g. SB-2611-0007), market_id, type (`safe_buy | service`), buyer_name, buyer_phone, listing_id (nullable), item_type_id, item_description, quantity, delivery_location, needed_by, notes, status, assigned_agent_id, money fields from section 6, cancel_reason
- `order_services` — order_id, service_id, provider_id (nullable), cost_pesewas, status, notes
- `order_events` — order_id, from_status, to_status, actor_id, note, created_at (append-only)
- `inspections` — order_id, agent_id, notes, health_ok (bool), est_weight_kg, agreed_price_pesewas, inspected_at
- `inspection_photos` — inspection_id, cloudinary_public_id, url
- `payments` — order_id, direction (`in_from_buyer | out_to_seller | out_to_provider | refund`), amount_pesewas, method (`momo | cash | bank`), reference, counterparty_phone, recorded_by, paid_at
- `sms_log` — to_phone, template, body, provider_message_id, status, error
- `audit_log` — actor_id, action, entity_type, entity_id, before_json, after_json, ip, created_at (append-only; no UPDATE/DELETE grants)
- `settings` — key, value_json (fee rule, admin alert numbers, re-review-on-edit flag, WhatsApp number, call numbers)

Seed: Adaklu market; Livestock product line; item types Cattle, Goats, Sheep, Pigs, Fowls, Ducks, Guinea fowl, Turkeys, Rabbits, Grasscutters; services Safe Buy, Butchering, Refrigeration, Transport; super admin account for 0556823692; settings with both contact numbers.

---

## 8. API outline (Express, `/v1`)

Public
- `GET /markets/current` — market resolved from host
- `GET /item-types`
- `GET /listings` — filters, sort, pagination (approved + available only)
- `GET /listings/:ref`
- `POST /orders` — Safe Buy or service request (rate-limited, honeypot field)
- `POST /orders/track` — `{ref, phone}` → public timeline

Auth
- `POST /auth/otp/send` — `{phone}`
- `POST /auth/otp/verify` — `{phone, code}` → sets session cookie
- `POST /auth/logout`
- `GET /me`

Seller
- `PUT /me/profile`
- `GET/POST /me/listings`, `GET/PUT/DELETE /me/listings/:id`, `POST /me/listings/:id/submit`, `POST /me/listings/:id/mark-sold`
- `POST /uploads/sign` — Cloudinary signed upload params (scoped folder per user)
- `GET/PUT /me/service-offering`

Admin (role-checked middleware; every write goes to `audit_log`)
- `GET /admin/dashboard`
- `GET /admin/users`, `GET /admin/users/:id`, `POST /admin/users/:id/{verify|reject|suspend}`, `POST /admin/users` (agent-assisted seller)
- `GET /admin/listings`, `POST /admin/listings` (on behalf), `PUT /admin/listings/:id`, `POST /admin/listings/:id/{approve|reject|request-changes|feature}`
- `GET /admin/orders`, `GET /admin/orders/:id`, `POST /admin/orders/:id/transition` (validated against the state machine), `POST /admin/orders/:id/inspection`, `POST /admin/orders/:id/payments`, `PUT /admin/orders/:id`
- `GET /admin/payments` (ledger + summaries)
- `GET/PUT /admin/settings`, CRUD for markets / product-lines / item-types / services (super admin)
- `GET/POST/DELETE /admin/staff` (super admin)
- `GET /admin/audit-log`

Validate every request body with zod. Consistent error shape: `{ error: { code, message } }`.

---

## 9. Security and trust

- No seller phone numbers or exact farm locations in any public API response.
- OTP codes and session tokens stored only as hashes.
- Role checks on the server for every admin route; never rely on hiding UI.
- Cloudinary uploads signed server-side, restricted to images, max 8 MB, folder scoped per user/listing.
- Rate limiting on OTP, order submission and tracking endpoints. Honeypot field on public forms.
- CORS restricted to the site domains.
- Seller declaration (owns or is authorised to sell the animals) required before any listing is submitted; record timestamp.
- Admin can suspend a seller, which hides all their listings immediately.
- Daily Postgres backups (Railway) — confirm enabled.

---

## 10. Environment variables

```
# api
DATABASE_URL=
SESSION_SECRET=
NALO_API_KEY=            # or username/password, per Nalo's API
NALO_SENDER_ID=          # e.g. ADAKLULIVE (max 11 chars; register with Nalo)
SMS_MODE=live|console
CLOUDINARY_CLOUD_NAME=
CLOUDINARY_API_KEY=
CLOUDINARY_API_SECRET=
WEB_ORIGINS=https://adaklulivestock.com,https://www.adaklulivestock.com
ADMIN_ALERT_PHONES=+233556823692

# web
NEXT_PUBLIC_API_URL=https://api.adaklulivestock.com
NEXT_PUBLIC_WHATSAPP=233556823692
NEXT_PUBLIC_CALL_PRIMARY=0556823692
NEXT_PUBLIC_CALL_SECONDARY=0536800950
```

---

## 11. Build sessions

| Session | Scope | Done when |
|---|---|---|
| S1 | Monorepo scaffold, Postgres schema + migrations, seed, Express skeleton with zod, error handling, health check, PROGRESS.md | `npm run dev` runs web + api; seed loads |
| S2 | OTP auth (Nalo + console mode), phone normalisation, sessions, role middleware, rate limits | Can log in as seeded super admin locally |
| S3 | Public site: home, listings grid + filters, listing detail, static pages, market-from-host resolution | Approved seed listings render; mobile layout checked |
| S4 | Seller area: profile, declaration, my listings, listing form, Cloudinary signed uploads with client-side compression | Seller can create listing with photos and submit for review |
| S5 | Admin: dashboard, sellers review, listings review, agent-assisted listing, audit log | Admin approves seller + listing and it goes public |
| S6 | Orders: Safe Buy + service request forms, order refs, state machine, timeline, track-my-order, inspections | Full order can move new → completed with events logged |
| S7 | Payments ledger, fee settings, SMS notifications for all events, SMS log | Ledger balances per order; SMS fire in console mode |
| S8 | Catalogue + staff management, SEO (OG tags, sitemap), performance pass, deploy: API to Railway (`api.` subdomain), web to Netlify, DNS records, production env | Live on adaklulivestock.com with real SMS |

### Deployment notes for S8
- Namecheap DNS: keep existing ALIAS `@` and CNAME `www` for the site. Add a CNAME `api` → the Railway-provided hostname.
- Switching the Netlify site from the current static page to the Next.js app: connect the GitHub repo to the existing Netlify site (merry-tulumba-f36418) so the domain setup is kept.

---

## 12. Out of scope for v1 (design for, don't build)
- Online payment (Hubtel checkout / GuardPay escrow)
- Buyer accounts
- In-app chat between buyer and seller
- USSD listing for sellers without smartphones
- Multiple languages (Ewe)
- Ratings and reviews
- Additional markets and product lines (data model supports them; no UI to switch markets yet)
