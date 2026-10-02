---
name: qunei-bookkeeping
description: Use when doing any bookkeeping or accounting work in Qunei — recording, correcting, or querying journal entries, checking balances, or answering money questions for a Qunei entity. Not for unrelated coding tasks.
---

# Qunei Bookkeeping

Session discipline for working an entity's books through the Qunei MCP
tools. Read this before calling any tool that reads or writes journal
data. The server enforces the hard rules (posting policy, structured
validation); this skill explains how to work inside them like a careful
bookkeeper would.

## 1. Start every session with a briefing

1. Call `get_briefing` before any other tool, every session, no
   exceptions — except when no Qunei entity exists yet; then
   qunei-onboard governs, and briefing comes after `init_entity`. It
   returns the entity's currency, books, and posting policy; pending
   drafts and recent posting activity; and a capped view of
   `conventions.md` and `open-items.md` (8KB each, marked
   `truncated: true` when cut — see qunei-conventions / `read_conventions`
   for `conventions.md`'s full text).
2. Treat its `hints` array as live instructions, not decoration — it is
   the server itself restating "briefing first" and "respect the posting
   policy" for entities that skip this skill.
3. If `open_items` carries unresolved lines from a prior session, address
   them or explicitly re-flag them before doing new work.

## 2. Never guess an account name

1. Before posting to or reporting on an account, confirm it exists: call
   `list_accounts`, or check the conventions text from the briefing for a
   documented payee-to-account rule.
2. If nothing matches, do not invent a name or code. Ask the human which
   account to use, or propose adding one with `add_account` and get
   confirmation before you call it.
3. When this resolves into a stable, reusable rule, record it — see
   qunei-conventions.

## 3. Corrections replace; they never patch

1. Fix a wrong entry with `amend_entry` (replace it) or `reverse_entry`
   (negate it) — never by posting a second entry that manually offsets
   the mistake, and never by re-posting the transaction from scratch.
2. `amend_entry` posts the target's exact negation plus your replacement,
   atomically, in the entry's original layer — one call does the whole
   correction.
3. `amend_entry` and `reverse_entry` are always allowed regardless of
   posting policy. That is exactly why they are reserved for genuine
   corrections to entries that already exist — not used as a back door
   around draft review for entries that don't.

## 4. Structured errors: read, self-correct once, then ask

1. A failed call returns a structured error with a `code`, `message`,
   and `hint`. Example: a date sent as `13/01/2026` instead of ISO 8601
   fails with `grammar.malformed-line`; the message names the bad value
   — reformat to `YYYY-MM-DD` per §7 and retry.
2. Read the code and hint, fix the one thing they name, and retry —
   ONCE.
3. `account.unknown` is NOT self-correctable this way: its hint says to
   declare the account, but declaring one means calling `add_account`,
   which §2 already governs — ask the human or propose `add_account` and
   wait for confirmation, never call it unprompted just to satisfy this
   retry-once rule.
4. If a genuinely self-correctable retry still fails, stop guessing.
   Explain the error to the human in plain language and ask how to
   proceed.

## 5. Money questions come from query tools, never memory

1. Answer any balance, spend, or activity question by calling
   `trial_balance`, `profit_and_loss`, `balance_sheet`, `account_ledger`,
   or `find_entries` in the current session.
2. Never answer from a figure you recall from earlier in this
   conversation or a prior session — the books may have changed since.
   Re-query every time, even if you just asked a moment ago.
3. Use `get_entry` to inspect one entry, including its correction chain,
   before describing what happened to it.

## 6. Respect the posting policy

1. Check `entity.posting_policy` from the briefing before posting
   anything.
2. Under `draft-first` (the template default), `post_entries` accepts at
   most 3 entries per call; a larger batch is refused with
   `policy.draft-first`. Use `stage_drafts` for the batch, then
   `approve_drafts` once the human has reviewed it via `list_drafts`.
3. Under `direct-allowed`, `post_entries` has no size limit — but still
   prefer staging anything the human hasn't already approved in this
   conversation.
4. Never split one large batch into several small `post_entries` calls to
   dodge the limit. That defeats the review the policy exists for.

## 7. Formatting rules

1. Dates are ISO 8601 (`YYYY-MM-DD`) in every tool call.
2. Every entry sourced from a bank statement or other document carries
   its `doc` reference (`stage_drafts` also takes a batch-level
   `source_doc` when a whole batch comes from one file). An entry with no
   traceable source is a red flag, not a shortcut to take.
3. Qunei records who made each entry by itself: `posted_by` and
   `posted_via` (and, on an approved draft, `drafted_by` and
   `drafted_via`) come from the connection you are signed in through.
   There is no way to set them, and no need to mention who you are in a
   memo. `get_entry` shows them; `find_entries` can filter on
   `posted_by` when someone asks what a person posted.

## 8. GST- and VAT-registered entities

1. An entity is registered when the briefing carries a `gst` key (New
   Zealand, configured through qunei-nz-gst), or when its conventions
   record that it is registered for GST or VAT (Australian and UK
   ledgers, whose return packs don't exist yet; qunei-onboard §2.6
   records it). Then every taxable posting records the amount NET, with
   the tax split into its own line to the tax code's liability account,
   never gross-coded as one tax-inclusive figure. A convention that says
   the entity is not registered means amounts are recorded gross, with
   no tax tags.
2. Tag the net posting itself with `tax:<code>`; the tax-account line
   stays untagged. See qunei-nz-gst for the full worked example, which
   lines get tagged, and which never do; qunei-code-statement §4 gives
   the tax fraction at each rate.

## 9. End of session: leave open items

1. Before ending a session, call `append_open_item` for every unresolved
   question, ambiguous match, or follow-up the human still owes an
   answer to.
2. Write each as a complete sentence a future session — yours or
   someone else's — can act on without today's chat history.
3. Open items are for genuine unresolved questions, not a log of routine
   status.

## 10. Multi-ledger workspaces: name the ledger

1. In a workspace where more than one ledger is visible to you, every
   write call must name the ledger — pass `entity:` explicitly; the
   server refuses the default-ledger fallback with
   `entity.explicit-required`, by design. That refusal is the rule
   working, not an error to route around.
2. When the conversation has not made the target ledger obvious, ask
   the human which ledger before naming one — never guess. With
   exactly one ledger visible, omitting `entity:` is fine.

## 11. Foreign-currency amounts

1. The books are kept in the ledger's currency. When evidence arrives in
   another currency with no home-currency figure beside it — a supplier
   invoice in USD, a sale billed in AUD — ask `get_fx_rate` for the pair
   and the transaction date, and convert at that rate. Never convert from
   memory, from a rate someone quoted in passing, or from a rate fetched
   for a different day.
2. Write the conversion into the memo so the entry audits itself: the
   foreign amount, the rate, its date and its source, e.g.
   `USD 1,250.00 at NZD/USD 0.58975 (RBNZ, 2026-09-12)`. The tool's
   answer carries every part of that sentence — `rate`, `rate_date` and
   `attribution`.
3. `stale: true` means the print is more than five business days older
   than the date you asked for — the feed has stalled, or the date is
   well ahead of the prints. Say so in the memo and tell the human before
   posting; never use a stale rate quietly.
4. A bank line that already shows the amount in the ledger's currency is
   the truth for that transaction — the bank's own conversion, fees and
   all. Post that figure and leave the rate tool out of it.
5. `NZD/USD` is the RBNZ print (one NZD buys that many USD); `USD/NZD` is
   its inverse, and a pair between two served currencies such as
   `USD/AUD` is derived through NZD from the same day's two prints. Both
   come back marked `derived`, and the memo says so:
   `USD 1,250.00 at USD/AUD 1.54387 (RBNZ via NZD, 2026-09-12)`.
6. The currencies served today are the ones the RBNZ prints against NZD:
   USD, GBP, AUD, JPY, EUR, CAD, KRW, CNY, MYR, HKD, IDR, THB, SGD, TWD,
   INR, PHP and VND. For any other currency the tool has no rate and says
   so. Do not look one up elsewhere and do not guess: ask the human for
   the rate to apply, then write the rate, its date and who supplied it
   into the memo, e.g. `CHF 800.00 at NZD/CHF 0.5210 (rate supplied by
   the owner, 2026-09-12)`. Rule 4 still stands: a bank line already in
   the ledger's currency needs no rate at all.

## 12. Draft-only and read-only connections

1. The human chooses how much each connection may do when they make its
   token or connect it: full access, draft-only or read-only. The server
   lists only the tools your connection may call, and its connect-time
   instructions say so when yours is draft-only or read-only.
2. A draft-only connection reads everything, stages drafts with
   `stage_drafts`, and updates or discards only the drafts it staged
   itself. It never posts, approves, corrects, issues, voids, files,
   configures, invites, syncs or dismisses inbox items.
3. When your drafts are ready, call `request_approval` with the ledger.
   Qunei emails the human a link to that ledger's drafts page, where they
   sign in and approve or discard each draft. Tell the human you have
   asked. Never paste an approval link or ask them to approve in the
   chat: approval happens on Qunei's page, outside this conversation.
4. `request_approval` emails at most once an hour per connection and
   ledger. A request inside the hour succeeds with `emailed: false` and
   says when the last email went; the page already lists every waiting
   draft, so do not ask again to hurry the human.
5. `token.scope-denied` means your connection's level does not allow the
   call. It is the rule working, not an error to retry or route around:
   stage a draft instead, or tell the human the task needs a full-access
   connection, which they choose when they make a token or connect an
   assistant. A draft you did not stage is not yours to change; stage a
   corrected one and let the human discard the other.
6. A read-only connection answers questions from the query tools and
   changes nothing.

## 13. Expense claims

1. People spend their own money for a ledger and claim it back. A member
   sets the ledger up once with `configure_expense_claims`: the liability
   account claims are owed from (`payable`: usually the ledger's existing
   Accounts Payable, as Xero does; a separate liability only when the
   human wants staff reimbursements kept apart), whether the ledger splits
   tax (`split_tax: true` when it is registered for GST or VAT, §8), and
   the expense accounts people may claim against, each with the label they
   see. A claim's cost always goes to the expense account its category
   names (travel to the travel account); the payable only holds what the
   person is owed until they are paid back. Ask the human for every one of
   these; never guess them. Each call states the whole setup again.
2. A claim is always the claimant's own. `submit_expense_claim` stages a
   draft for the signed-in person, crediting the payable with their
   `claimant:` tag, and splits the tax out by the category's default code
   when the ledger splits tax. There is no way to name someone else, and
   you never make a claim for anyone through the claims tools. When a
   receipt for a colleague's spending reaches the inbox, code it as an
   ordinary draft with `stage_drafts`, crediting the payable with the
   colleague's `claimant:` tag (their email address lower-cased, with `@`
   written as `-at-`, as `list_expense_claims` shows it), and tell the
   human you did.
3. `list_expense_claims` shows every claim, where it stands (waiting,
   approved, reversed, declined or withdrawn) and what each person is
   owed. A waiting claim is a draft: approve it with `approve_drafts` once
   the human says so, or decline it with `decline_expense_claim` and the
   reason the human gives, in one line, which the person sees. Prefer
   declining to `discard_draft`: a discarded claim leaves its person no
   record of why. The person may withdraw their own waiting claim with
   `withdraw_expense_claim`.
4. Reimbursing is ordinary bookkeeping. When the bank pays a person back,
   code the payment to the payable account with that person's `claimant:`
   tag, exactly as their claims carry it, so what they are owed drops by
   what was paid. One payment for several claims is one posting of the
   total.
5. `role.claims-only` means the person has expense claims only on that
   ledger: `list_expense_claims`, `submit_expense_claim` and
   `withdraw_expense_claim` work for them, and every other tool is refused.
   It is the rule working; do not try other tools for them. They see only
   their own claims, and each waits for a member.
6. Claims are in the ledger's own currency. A receipt in another currency
   is converted first (§11), or a member codes it.

See qunei-conventions for how learned rules get recorded and consulted,
qunei-onboard for setting up a new entity from scratch, and qunei-nz-gst
for the GST-registered net-split convention and the return it feeds.
