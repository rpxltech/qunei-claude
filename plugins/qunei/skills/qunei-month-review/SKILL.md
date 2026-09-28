---
name: qunei-month-review
description: Use for a month-end or periodic review of a Qunei entity's books — scanning for posting oddities, sweeping stale drafts, and comparing this period's P&L to the prior one.
---

# Qunei Month Review

A periodic health check over an entity's books: scan for posting
oddities, sweep stale drafts, and compare this period's P&L to the prior
one. This skill only reads and appends open items — it never posts,
stages, or approves anything.

## 1. Start with a briefing

1. Call `get_briefing` first (qunei-bookkeeping's session rule). Note the
   current posting policy and any open items already on record — don't
   re-flag something already tracked.

## 2. Trial balance scan

1. Call `trial_balance` as of the review date (typically today or the
   period end). It is a working paper: every line carries its own `type`
   plus `opening`, `debits`, `credits` and `closing`, and the payload
   echoes the window it used as `from`, `as_of` and `financial_year`.
   No cross-check against another tool is needed to read it.
2. `opening` and `closing` are raw signed amounts — positive is a debit,
   negative is a credit — while `debits` and `credits` are both reported
   positive, since each column already names its side. (The balance
   sheet's own retained-earnings line uses the opposite, credit-positive
   convention; never carry a figure between the two statements.)
3. The scan is a sign test of `closing` against `type`. Asset and
   expense accounts are normally debit-positive, so a NEGATIVE closing
   is the flag; liability, equity and income accounts are normally
   credit-negative, so a POSITIVE closing is the flag. The classic two
   are a credit-balance asset (an overdrawn bank account, a customer
   sitting in credit) and a debit-balance liability (a GST refund due,
   an overpaid supplier): each is sometimes real, and each is always
   worth naming.
4. Read the movement columns too, not just the closing balance. An
   account whose `closing` is zero but whose `debits` and `credits` both
   moved went out and came back inside the window — which is exactly
   what a working paper exists to show. Look there for activity parked
   in a placeholder or suspense-style account instead of its real
   destination, and for a round trip that should have been one
   correction.
5. `balanced: false` is the loudest finding on the page: the closing
   column doesn't sum to zero, or the two movement totals disagree.
   Raise it ahead of everything else.
6. `retained_earnings_brought_forward` sits beside `lines`, not in them:
   one computed line, no `code`, no movement, carrying what earlier
   financial years earned. It is derived, not posted — don't flag it as
   a stray or unmapped account.
7. Note every oddity with the account, the column that looks wrong, and
   why — real ones become open items in §6.

## 3. Compare this period to the prior one

1. ONE call: `profit_and_loss` with the current period's `from`/`to` and
   `compare: "prior-period"`. The tool picks the comparative window
   itself, queries it, and returns the comparison already computed. Two
   calls and a subtraction is the old pattern and is wrong — it silently
   drops any account active in only one of the two periods, which the
   single call unions in with a zero instead. Use
   `compare: "prior-year"` when the real question is the same dates a
   year earlier.
2. Quote the window the payload names, in
   `comparative: { period: { from, to } }`, rather than restating it
   from your own date arithmetic.
3. Every line and every mapped section total then carries `comparative`,
   `variance` and `variance_pct` beside its own amount, and each
   subtotal becomes `{ amount, comparative, variance, variance_pct }`.
   Build the table straight from those four: current, prior, variance,
   percentage. Never compute a variance yourself.
4. `variance_pct` is a decimal string measured against the comparative's
   magnitude, and it is null when the comparative was zero. Render that
   cell as "n/a" — never 0%, never an infinity.
5. Call out the variances that are unusual for that account rather than
   the largest raw numbers, and give any line whose comparative is zero
   a sentence of its own: an account that started or stopped this period
   is a finding, not a rounding difference.

## 4. Draft sweep

1. Call `list_drafts` and flag anything stale — drafts that have sat
   unapproved since a prior session are worth surfacing even though this
   skill won't act on them (staging and approval belong to
   qunei-code-statement or qunei-bookkeeping, not here).

## 5. GST drift check

1. When the briefing carries a `gst` key, this entity runs GST — pull
   two periods: the CURRENT one (`briefing.gst.period.end`) and the LAST
   FILED one (`briefing.gst.last_filed.period_end`, when there is one).
   No `gst` key means the pack was never configured; skip this section
   rather than guessing a period.
2. Call `gst_return` for the current period. Report any warnings exactly
   as returned, one line per code — the same catalogue qunei-nz-gst
   explains in full; this skill only surfaces them, it never re-codes
   anything.
3. The canonical drift surface is that same current-period call's own
   `breakdown.late_claims` — one row per filed period still carrying an
   outstanding residual, already corrected for what that period's own
   filing absorbed and whatever a later one has since swept — together
   with its `warnings`, which carry the other integrity signals. Call
   `gst_return` again for the last-filed period, when one exists, only
   to reproduce what was actually filed, via that response's own
   `filed` block: a raw diff of its live `boxes` against `filed.boxes`
   is NOT a residual measure, because filed boxes already fold in
   absorbed late claims and any typed box9/box13 adjustments.
4. Fold both results into §6's write-up below — drift and unresolved
   warnings are exactly the kind of finding `append_open_item` exists
   for. Never file, amend, or reconfigure the GST pack from this skill
   to make one go away.

## 5b. Debtors

1. When the briefing carries an `ar` key, this entity issues invoices —
   the same gate §5 applies to `gst`. No `ar` key means invoicing was
   never configured; skip this section rather than calling a tool that
   will refuse.
2. Call `aged_receivables` with `at` set to the REVIEW DATE, the same
   date §2's trial balance was drawn at. Omitting `at` ages the ledger
   as at today, which quietly reviews a different period than every
   other section here. No `customer` filter: a review looks at the whole
   ledger.
3. Report every customer with anything in the `over-90` bucket as a
   finding, with the customer, the amount, and the oldest invoice's
   `number` and `due` date. Old debt is a collectability question and a
   possible bad-debt write-off, both of which are the human's to decide
   — this skill never drafts a chase letter (that is qunei-invoicing §7)
   and never writes anything off.
4. Read the `control` row before quoting any total. `reconciles: false`
   means the receivables account and the aging disagree by the
   `difference` shown: money booked straight to receivables without an
   invoice, a credit note tagged to the wrong number, an invoice posted
   twice. That is a finding in its own right — one of the more valuable
   ones a monthly review produces — and it is stated, never fixed here.
5. Fold both into §6's write-up. This section, like every other one
   above, is read-only.

## 6. Write findings, don't act on them

1. For every genuine oddity from §2–§5b that needs a human decision, call
   `append_open_item` — one call per question, per qunei-bookkeeping's
   end-of-session rule (a complete sentence a future session can act on
   without today's chat history). This skill's whole write surface is
   that one tool.
2. Finish with a summary to the human: the 3–5 things that actually
   matter this period, not a dump of every line you looked at. Lead with
   anything that looks like a real error over normal variance.

This skill is read-only except for `append_open_item` — never call
`post_entries`, `stage_drafts`, `approve_drafts`, `amend_entry`,
`reverse_entry`, `configure_nz_gst`, or `file_gst_return` from here. See
qunei-bookkeeping for how those get used once a finding needs action,
qunei-nz-gst for how a GST warning or drift finding actually gets
resolved, qunei-invoicing §7 for the chase a §5b finding often leads to,
and qunei-reports for turning review output into a presented statement
or dashboard — its §5 carries the full shape of a compared payload, its
§6 the working trial balance's columns and signs, and its §7 the aging's.
