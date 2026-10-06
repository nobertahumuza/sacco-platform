# Design notes â€” Connect With The Empire

Why the system is built this way. The README covers usage; this covers reasoning,
assumptions, and the known gaps.

---

## 1. The hierarchy, and why it is drawn the way it is

```
Connect With The Empire (platform)
  â””â”€ tenants = SACCOs        â† the paying customers
       â””â”€ members            â† each SACCO's own clients
```

This is the opposite of the previous framing, where one SACCO was the product and
the "SaaS" was a plan attached to it. The consequence runs through everything:

- **Billing is platformâ†’SACCO.** The bank account is the platform's, not the
  SACCO's. `BILLING_ACCOUNT_NAME` is `Connect With The Empire`.
- **Templates are split by audience.** Member-facing SMS says "SACCO deposit of
  â€¦ received" and leaves the SACCO's branding to the SACCO. Billing SMS says
  "Connect With The Empire: your subscription is due" because Connect With The Empire issued the
  invoice.
- **Two separate consoles.** `/portal` for a SACCO admin (their subscription, their
  members) and `/platform` for the owner (every SACCO, revenue, what is lapsing).
  Conflating them is how a platform accidentally shows SACCO A's member balances
  to SACCO B's admin.
- **Member-facing URLs carry no SACCO brand**, because the platform chrome belongs
  to us. Each SACCO's own name appears inside their console.

### Multi-tenant from day one

Every table carries `tenant_id` and every query filters on it. Retrofitting that
onto twenty tables after the first twenty customers is far more expensive than
carrying it now.

The seed deliberately creates **two SACCOs**. Tenant isolation that has only ever
been tested against one tenant is an assumption, not a fact.

---

## 2. Assumptions made

Unspecified, so I chose. Each is worth confirming.

| Assumption | Reasoning |
|---|---|
| SACCO self-signup is open by default | `ALLOW_SACCO_SELF_SIGNUP`. A sales-assisted model does not scale past a handful of SACCOs and puts the owner in the loop of every signup. |
| 14-day trial, 7-day grace, 90-day retention | Spec said "e.g." for each. Lives in `subscription_plans`, changeable per plan. |
| `current_period_end` is the due date | Verifying a payment advances it one calendar month **from the current due date**, not from today â€” so a SACCO paying three months late does not silently lose the intervening months. |
| Payment is by bank transfer, not card | Matches a Ugandan SACCO's actual workflow. |
| SACCO name uniqueness is enforced case-insensitively | Two "Ntinda United SACCO" tenants would split one real SACCO's records in half, which is worse than refusing the second. |
| One super admin | Enforced by a partial unique index. Enough for a small platform; revisit before hiring. |
| Loan tables are out of scope | The ledger primitives they need are built; the spec's own ordering puts loans after the subscription core. |

**Not assumed:** anything that moves real money through MTN/Airtel. Those need
telco-id idempotency and a reconciliation model that do not exist yet.

---

## 3. Subscription status is a pure function

The single most important decision in the system.

The tempting design is a cron writing `status` and handlers reading the column:

```ts
// Do not do this.
if (subscription.status === "suspended") throw forbidden();
```

Its hidden property is that **access control depends on a scheduled job running
correctly.** Miss the cron, hit a Vercel plan limit, disable it during an incident â€”
and every non-paying SACCO keeps full access to financial functions. No error, no
alert, no log line. The exposure lasts until a human notices a revenue gap.

So enforcement computes instead:

```ts
const resolved = resolveStatus(row, new Date());   // pure
if (!can(resolved.status, "post_transaction")) throw forbidden();
```

The cron still exists and still writes the transition â€” but only so the console
and the revenue report tell a coherent story. It **cannot grant access**, because
nothing grants access.

A test pins this:

```ts
test("a missed cron cannot grant access: status is derived, not read", () => {
  const r = resolveStatus(activeSub({ status: "active" }), at("2026-12-01T00:00:00Z"));
  assert.equal(r.storedStatus, "active");
  assert.equal(r.status, "suspended");
});
```

The API surfaces `status`, `storedStatus` and `derivedFromClock` so an operator
can *see* that the row and the enforcement disagree, rather than having to know
to look.

### One place this rule is duplicated, deliberately

`platformRevenue()` re-implements the lifecycle in SQL so the platform console can
aggregate across hundreds of tenants in one query. That is a second copy of the
transition table and it will drift if the TS changes. It is commented at the site
as the second place to update, and a follow-up should collapse it â€” either by
materialising a `resolved_status` view owned by one definition, or by computing
over rows fetched into TS.

---

## 4. Money and the ledger

**`BIGINT` only.** UGX has no minor unit in circulation. `parseAmount` rejects
`1000.5` rather than rounding it, because a silent 1-shilling drift makes a trial
balance unprovable.

**Balances are derived.** No `members.balance`, no `savings_accounts.balance`:

```
journal_entries      one immutable posting per business event
ledger_lines         debit/credit, must sum to zero per entry
savings_transactions read model with running_balance, for display
```

`postEntry` refuses to commit unless debits equal credits â€” an unbalanced trial
balance is *impossible*, not merely unlikely. `assertLedgerBalances` recomputes
every cached running balance and returns the differences; empty is the only
acceptable result.

**Corrections are reversals.** `reverseEntry` posts the mirror image and marks the
original. No `UPDATE`, no `DELETE`. Both sides stay visible to an auditor.

This is what makes the member portal and the API unable to disagree about a
balance, and what survives a UMRA inspection.

### One known rough edge

`savings_transactions.running_balance` is maintained inside the posting
transaction, which is correct, but the *deposit* and *withdrawal* paths both
compute `balanceBefore` from the ledger while writing the projection from that
same value. There is no independent reconciliation on the write path â€” only
`assertLedgerBalances` as an after-the-fact check. Adding a database trigger that
rejects a projection inconsistent with the ledger would close this properly.

---

## 5. SACCO signup

`POST /api/auth/register` is the front door and, being public, the highest-value
target on the system.

**Atomicity is the main design point.** Tenant + `sacco_admin` user + trial
subscription happen in one transaction. A half-provisioned tenant â€” a tenant with
no subscription, or an admin with no tenant â€” is *worse than a failed signup*,
because it looks successful and then denies every request for reasons the SACCO
cannot see.

Other decisions:

- **Password floor above consumer signup.** 10 characters minimum plus a
  guessability block. This password guards every member record in the SACCO; a
  SACCO admin will not have a password manager.
- **Duplicate detection before any write**, case-insensitively. Two tenants for one
  real SACCO splits its records, which is a data-integrity problem, not a nuisance.
- **Duplicate errors omit the tenant id.** Otherwise signup becomes an enumeration
  oracle for "which SACCOs are on this platform".
- **Honeypot field.** A filled one returns a plausible `202` rather than a
  validation error, so the bot learns nothing about what gave it away.
- **Rate limited per IP**, backed by Postgres because in-memory counters multiply
  the effective limit by the instance count on Vercel.

---

## 6. Payment verification

A deposit slip is a photograph of a piece of paper. Recording and verifying are
therefore separate operations, and the verifier must be a different person:

```
POST  /api/v1/subscription/payments      â†’ pending    cannot restore access
PATCH /api/v1/subscription/payments/:id  â†’ verified   advances the due date
```

Three independent controls, because one is not enough:

1. `recorded_by !== verified_by`, re-checked on every verification
2. partial unique index on `(subscription_id, period_covered)` where not refunded
3. amount `>=` plan price, checked arithmetically rather than judged from the image

**Honest limitation:** this makes the action attributable and auditable. It does
not make it true. The real fix is bank-statement reconciliation on amount, date
and reference, which turns this from an honour system into a matching problem.
That is not built, and until it is, the platform's revenue figure rests on a human
looking at a photograph correctly.

---

## 7. The river theme

Requested as a continuous water flow with gold currents. Implemented as **five
layered CSS gradients** behind the content:

- four `.river__band` layers at different speeds and opacities (`drift` translates
  them horizontally, the parallax is what makes it read as water)
- one `.river__glint` sweeping gold highlight on a long loop

No animation library, no canvas, no JavaScript on the critical path. A member's
low-end Android over 3G is the constraint that decided this.

Two details worth keeping:

- **The masthead is translucent**, so the flow shows through the header. Opaque
  chrome would have looked like a banner pasted over a wallpaper.
- **`prefers-reduced-motion` freezes the layers** rather than removing them. The
  gradient still carries the theme; only the movement stops.

If the gold reads as too much on real hardware, the lever is opacity on
`.river__band--gold` â€” the layers are independent.

---

## 8. Removing USSD

Done as instructed: the state machine, its 20 tests, the gateway route, the
session and interaction tables, the channel enums and the rate limit are all gone.
The `ussd` savings channel was replaced by `portal`.

**What was lost:** the end members of these SACCOs â€” small-scale traders, farmers,
market vendors â€” are the people least likely to own a smartphone or have mobile
data. USSD was the channel that reached them. The member portal works on any phone
browser but needs a connection, which is a genuine exclusion.

If access for feature-phone members matters, the cheapest path back is a WhatsApp
bot: no telco contract, no short-code allocation, works on basic handsets. Cheaper
than a USSD aggregator contract and materially faster to ship.

---

## 9. Security posture

**Implemented:**

- bcrypt cost 12 for passwords
- JWT HS256, 8h, issuer and audience pinned
- API keys as `HMAC-SHA256(pepper, secret)`, shown exactly once
- lockout after 5 failures for 15 minutes
- timing-equalised credential comparison, including a dummy hash for unknown users
- append-only business audit log, retained indefinitely
- Postgres-backed rate limiting (correct across instances)
- redaction of credentials, national ID and consent text in audit diffs
- HSTS, nosniff, frame-deny, `no-store` on API paths
- signup: per-IP limit, honeypot, transactional provisioning, no id leakage

**Not implemented, and required before real SACCOs onboard:**

- **2FA.** Not built. The spec asked for TOTP or SMS.
- **Session cookies.** The login page stores the JWT in `localStorage`, an XSS
  exposure. Move to httpOnly + Secure + SameSite=Strict set by the server.
- **Penetration test.** None.
- **Upload scanning.** Deposit slips go to a private bucket with no malware scan
  or content-type validation. The upload endpoint is not built.
- **Data residency.** Undetermined.
- **Bank account ownership unconfirmed.** See README.

---

## 10. Compliance

I built the technical affordances. I cannot certify compliance and this needs a
human.

**UMRA.** Holding other SACCOs' member savings and issuing loans on their behalf is
regulated. Whether operating this platform requires registration or licensing as a
microfinance deposit-taking institution is a question for UMRA and counsel.

**Data Protection and Privacy Act 2019.** Processing member personal data at this
scale likely requires NDA registration as a data controller. Supported:
consent captured with timestamp and exact wording; member endpoints omit national
ID and consent text; a retention period on cancellation. Not supported: data
subject access and erasure workflows, and the post-retention anonymisation job.

---

## 11. Pricing

Implemented as specified: UGX 50,000/month to Standard Chartered 0100824048800.
It does not work. One 200-member SACCO's SMS cost (~UGX 24,000â€“80,000) can exceed
the entire subscription. Removing USSD removes a cost line, not the problem.

`DESIGN.md` Â§11 of the README has the viable shapes. Per-member pricing with a
floor is the honest fix. Confirm current aggregator rates directly; mine are
indicative.

---

## 12. Known gaps

Absent rather than stubbed to look complete:

- Loans (origination, approval, schedules, arrears)
- Share capital posting
- MTN / Airtel collection callbacks and webhook idempotency
- Withdrawal approval workflow (rejects `requireApproval` loudly instead of
  accepting and ignoring it)
- 2FA
- UMRA returns, PDF/Excel export
- Deposit-slip upload endpoint
- Data subject access / erasure
- Post-retention anonymisation job
- Notification templates themselves (the tables exist; the rows are seeded on demand)

Known rough edges in what is built:

- `platformRevenue()` duplicates the lifecycle transition table in SQL (Â§3)
- No DB-level trigger reconciling `savings_transactions.running_balance` (Â§4)
- `localStorage` sessions (Â§9)
- Migrations never executed â€” `db/migrate-verify.txt`