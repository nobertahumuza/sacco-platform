# Connect With The Empire

A hosted management system for Ugandan SACCOs. Connect With The Empire is the platform;
**SACCOs are the paying customers**, and each SACCO manages its own members on
top of it.

```
        Connect With The Empire  (the platform â€” you are here)
             â”‚
             â”‚  UGX 50,000 / month per SACCO
             â”‚  Standard Chartered Bank Uganda
             â”‚  Account 0100824048800
             â”‚
    â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”´â”€â”€â”€â”€â”€â”€â”€â”€â”
    â–¼                 â–¼
 SACCO "A"         SACCO "B"          â† tenants, self-signup
    â”‚                 â”‚
    â–¼                 â–¼
 Members           Members           â† each SACCO's own clients
```

---

## What changed in this revision

| Change | Detail |
|---|---|
| **Renamed** | `Connect With The Empire`. Mukama Nuwamanya is now a *customer*, not the product. |
| **Model inverted** | SACCOs are the clients. They sign themselves up and pay. The platform sees every SACCO. |
| **Bank account changed** | Standard Chartered Bank Uganda, `0100824048800`, account name `Connect With The Empire`. |
| **USSD removed** | Channel, state machine, gateway route, tests and tables all deleted. |
| **Signup added** | `POST /api/auth/register` + `/signup` provisions a SACCO with a trial in one transaction. |
| **Platform console added** | `/platform` shows every SACCO, revenue and what needs attention. |
| **Theme** | Continuous river: deep blue water with gold currents. CSS only, respects reduced-motion. |

### âš ï¸ Verify the bank account before going live

`0100824048800` is printed on the **public signup page**, in every reminder SMS,
on invoices and in the API docs. If it is wrong or is not an account you control,
every SACCO subscription payment goes somewhere you cannot see. Confirm the
number, the account name, and that you can receive on it, before accepting a
single real signup. It is one value in `.env`:

```
BILLING_BANK_NAME=Standard Chartered Bank Uganda
BILLING_ACCOUNT_NUMBER=0100824048800
BILLING_ACCOUNT_NAME=Connect With The Empire
```

The account is also **frozen onto each subscription row at signup time**, so
historical invoices keep showing what a SACCO was actually told to pay, even if
you change it later.

### âš ï¸ Removing USSD costs something

The end members of these SACCOs â€” small-scale traders, farmers, market vendors â€”
are exactly the people least likely to own a smartphone or have data. USSD was the
channel that reached them. The member portal works on any phone browser but
requires an internet connection, which is a real exclusion.

I removed it as instructed. If access for feature-phone members matters, the
cheapest path back is a WhatsApp bot (no telco contract, no short code, works on
basic handsets) rather than a full USSD gateway. Worth deciding deliberately
rather than by default.

---

## Status

| Built and tested (57 unit tests) | Written but unverified against a database |
|---|---|
| Subscription lifecycle state machine | SQL schema and migration runner |
| Capability matrix per status | Seed script (two SACCOs) |
| UGX money arithmetic and interest | All DB-backed request handlers |
| SACCO signup validation and slugs | `registerSacco` transaction |
| Platform revenue and usage queries | Vercel Cron billing job |
| API auth, rate limiting, audit | MTN / Airtel integrations |

**The migrations have never been run.** No Docker and no local Postgres on this
machine. The SQL is written and reviewed but unexecuted â€” see
`db/migrate-verify.txt`.

Absent rather than stubbed: loans, share capital, MoMo/Airtel callbacks, 2FA,
UMRA returns, PDF/Excel export, deposit-slip upload.

---

## Run it

```bash
cp .env.example .env        # edit JWT_SECRET and API_KEY_PEPPER at minimum
docker compose up -d
npm install
npm run db:migrate
npm run db:seed
npm run dev
```

Seeded logins â€” note the passwords differ per SACCO:

| SACCO | Email | Password |
|---|---|---|
| Mbarara City Mukama Nuwamanya | `admin@mukamanuwamanya.co.ug` | `ChangeMe!Admin2026ok` |
| Ntinda United SACCO | `admin@ntindaunited.co.ug` | `ChangeMe!Admin2027ok` |
| Platform owner | `SUPER_ADMIN_EMAILS` | `ChangeMe!Owner2026ok` |

Two SACCOs are seeded on purpose: if the platform console shows both but a SACCO
admin only sees their own members, tenant isolation works.

---

## The three decisions that shape everything

### 1. Subscription status is computed, never trusted from the column

`resolveStatus(subscription, now)` is a pure function. The obvious alternative â€”
a daily cron writes `status = 'suspended'`, handlers read the column â€” means
**access control depends on a scheduled job running correctly.** Miss the cron, hit
a Vercel plan limit, pause it during an incident, and every non-paying SACCO keeps
full access to financial functions. Silently. Until someone notices a revenue gap.

So the guard calls the pure function. The cron still runs and still writes the
transition, but only so the console and revenue report agree. **It cannot grant
access.** The API exposes `status` (enforced), `storedStatus` (on the row) and
`derivedFromClock` so this is observable rather than folklore.

### 2. Money is `BIGINT`, never `FLOAT`

UGX has no minor unit in circulation. `parseAmount` rejects `1000.5` rather than
rounding it â€” a silent 1-shilling drift makes a trial balance unprovable. Amounts
travel as strings and are `bigint` in process.

### 3. The ledger is append-only double-entry; balances are derived

No `members.balance` column. `postEntry` refuses to commit unless debits equal
credits, so an unbalanced trial balance is *impossible* rather than unlikely.
Corrections are reversals, not `UPDATE`/`DELETE`. This is what makes the member
portal and the API unable to disagree, and what survives an audit.

---

## Subscription lifecycle

```
      self-signup
           â”‚
           â–¼
        trial â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â–º active â—„â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
       14 days               (paid)              â”‚ verify payment
           â”‚                    â”‚               â”‚
           â”‚ clock elapses      â”‚ clock elapsesâ”‚
           â–¼                    â–¼               â”‚
      grace_period â”€â”€â”€â”€â”€â”€â”€â–º suspended â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
      7 days unpaid        (payment restores
                             full access)
           â”‚
           â”‚ admin cancels
           â–¼
       cancelled â”€â”€ 90-day retention â”€â”€â–º anonymise / delete
```

### Capability matrix

| Status | Read | Post txn | Loans | Export | Products |
|---|---|---|---|---|---|
| `trial` | yes | yes | yes | yes | yes |
| `active` | yes | yes | yes | yes | yes |
| `grace_period` | yes | yes | yes | yes | yes |
| `suspended` | yes | **no** | **no** | yes | **no** |
| `cancelled` | yes | **no** | **no** | until retention lapses | **no** |

Two deliberate choices:

- **Suspended keeps `export_data`.** A SACCO being wound down needs its records to
  hand over. Locking data behind payment turns a customer relationship into a
  hostage situation, and it is the fastest way to earn a regulator's attention.
- **Recording a subscription payment is allowed in every state.** Otherwise a
  suspended SACCO can never pay its way out and becomes a permanent write-off.

`grace_period` withholds no capability. The warning banner is the enforcement.

---

## Payment verification

Recording and verifying are separate, and the verifier must be a different person.

```
POST  /api/v1/subscription/payments      â†’ pending    cannot restore access
PATCH /api/v1/subscription/payments/:id  â†’ verified   advances the due date
```

A bank deposit slip is a photograph of a piece of paper. If the person who uploads
it can also mark the account paid, revenue control reduces to "whoever holds the
platform owner password". Three independent controls:

1. `recorded_by !== verified_by`, re-checked on every verification
2. partial unique index â€” one non-refunded payment per `(subscription, month)`
3. amount `>=` plan price, checked arithmetically

**This makes the action attributable, not true.** The honest fix is bank-statement
reconciliation on amount + date + reference, which converts it from an honour
system into a matching problem. Not built.

---

## SACCO signup

`POST /api/auth/register` is public and unauthenticated by necessity, which makes
it the highest-value target on the system. Defences:

- rate limited per IP
- duplicate SACCO name and duplicate admin email rejected before any write
- tenant + admin user + trial subscription created in **one transaction** â€” a
  half-provisioned tenant looks successful and then denies everything
- password floor of 10 characters plus a guessability block, higher than consumer
  signup, because this password guards every member record
- honeypot field: a filled one gets a plausible `202` so the bot learns nothing
- duplicate-name errors deliberately omit the tenant id, so signup cannot be used
  to enumerate which SACCOs are on the platform

The seed creates two staff accounts per SACCO so the four-eyes rule can actually
be exercised rather than assumed.

---

## Platform console

`GET /api/v1/platform/saccos` and `/platform` show every SACCO with its enforced
status, usage and revenue. Two things worth knowing about how it is built:

- **Revenue counts verified payments only.** A figure that included unverified slip
  uploads is not revenue, and a board that reads it will make decisions on it.
- **Customer deposits are reported separately and labelled.** Member savings held
  on the platform are not platform revenue, and mixing them up is how a platform
  ends up unable to explain a balance sheet.

The cross-tenant routes reject any tenant-scoped credential before running a
query. They bypass `withApiHandler`, which would refuse a super admin for having
no tenant.

---

## Hosting

| Concern | Choice |
|---|---|
| App, portal, signup, API | Vercel (Next.js) |
| Database | Managed Postgres â€” Neon or Supabase, PITR enabled |
| Deposit slips | Vercel Blob, private bucket |
| Billing reminders | Vercel Cron (Pro only) |

Short, stateless endpoints fit serverless well. **The ledger is the part to think
harder about** â€” no SLA, no private networking, no support commitment you could
show a SACCO board. Budget for managed Postgres with point-in-time recovery and
be able to explain the failure plan to a board.

---

## âš ï¸ The pricing still does not work

UGX 50,000 â‰ˆ USD 13â€“14 per SACCO per month. Ballpark costs:

| Item | Monthly |
|---|---|
| Revenue per SACCO | UGX 50,000 |
| SMS, ~200 members Ã— ~4 messages | ~UGX 24,000â€“80,000 |
| Managed Postgres, backups, support | the floor |

One 200-member SACCO can spend its entire subscription on SMS. The USSD cost line
is gone but the SMS problem is untouched. Mitigation in code: SMS statements are
opt-in, suppressed sends are logged with a reason. That is not a business model.

Viable shapes:

1. **Per-member pricing with a floor** â€” a 50-member SACCO should not pay the same
   as a 500-member one.
2. **Usage metering with overage** on SMS above an allowance.
3. **Pre-funded SMS balance** the SACCO tops up.
4. **Treat 50,000 as a marketing headline.**

Confirm current aggregator rates directly; mine are indicative.

---

## Compliance â€” a human gate

- **UMRA.** Operating a platform that holds other SACCOs' member savings may
  require registration or licensing as a microfinance deposit-taking institution.
  Counsel, not an AI.
- **Data Protection and Privacy Act 2019.** Processing member data at this scale
  likely requires NDA registration as a data controller. The schema captures
  consent with its exact wording; the member API omits national ID and consent
  text. None of that is legal advice.
- **Security review before real members.** No penetration test, no 2FA, the login
  page still stores the JWT in `localStorage` (move to httpOnly cookies), no upload
  scanning, undetermined data residency.

---

## Commands

```bash
npm run dev          npm run build      npm run typecheck
npm test             # 57 unit tests, no database needed
npm run db:migrate   # forward-only, checksummed, advisory-locked
npm run db:seed
```

---

## Layout

```
db/
  migrations/         identity+tenancy Â· subscriptions+ledger Â· notifications+audit
  migrate.ts          forward-only runner with advisory lock
  seed.ts             plan, chart of accounts, two SACCOs, staff pairs
src/lib/
  subscription.ts     pure lifecycle + capability matrix
  onboarding.ts       SACCO self-signup
  platform.ts         cross-tenant SACCO list, revenue, usage
  money.ts            BIGINT arithmetic, interest
  ledger.ts           double-entry posting, derived balances, reversal
  savings.ts          transactional deposit/withdrawal
  api.ts              withApiHandler guard chain
  auth.ts ratelimit.ts audit.ts notifier.ts
src/app/
  page.tsx            marketing site
  signup/             SACCO self-registration
  login/ portal/      SACCO console
  platform/           platform owner console
  api/auth/{login,register}/
  api/v1/             public API incl. /platform/* (super admin)
  api/openapi/  api/docs/  api/health/  api/internal/cron/
```

Design rationale and the full security gap list: **[DESIGN.md](./DESIGN.md)**.

---

## Next, in order

1. **Run the migrations.** Nothing else matters until a trial balance can be proved.
2. **Loans** â€” origination, approval, disbursement, amortisation, arrears.
3. **Statement reconciliation** to replace slip trust with matching.
4. **Member portal UI** and deposit-slip upload to a private bucket.
5. **MTN / Airtel** with telco-id idempotency.
6. **2FA and httpOnly sessions** before real SACCOs onboard.
7. **Decide the mobile-money-member question** the USSD removal opened.