---
name: qunei-nz-gst
description: Use when computing, previewing, or filing a NZ GST return for a Qunei entity — GST101 boxes on either basis, filed-return history, late claims, and the net-plus-GST-split coding convention behind them.
---

# Qunei NZ GST

The GST101 return lifecycle for a GST-registered NZ entity: configure the
pack once against the entity's actual IRD registration, keep taxable
postings split correctly as they're coded, then preview, clear, and file
each period's return. This skill assumes qunei-bookkeeping's session rules
are already in effect — read that first if you haven't.

## 1. Configure before anything else

1. Call `get_briefing` first (qunei-bookkeeping's session rule) and check
   whether its `gst` key is present. Its absence means `configure_nz_gst`
   has never been run for this entity — every other tool in this pack
   (`gst_return`, `file_gst_return`, `list_gst_returns`) refuses with
   `gst.not-configured` until it has.
2. Configuring is a short conversation, not a silent setup call: ask the
   human their entity's actual IRD registration facts — accounting basis
   (`payments` or `invoice`), filing frequency (`monthly`, `two-monthly`,
   or `six-monthly`), and the cycle-anchor month (one real period-end
   month under that frequency, e.g. `2026-08` for a two-monthly entity
   whose periods end Feb/Apr/Jun/Aug/Oct/Dec). These are facts to ask
   for, never to guess or default — a wrong basis makes every return
   this pack computes wrong in a way that won't be obvious until it
   disagrees with what was actually filed with Inland Revenue.
3. `configure_nz_gst` also seeds the three canonical tax codes
   (`gst-standard` at 15%, `gst-zero`, `gst-exempt`) into tax-codes.txt
   if they aren't already declared — a no-op on `nz-company` (which
   ships them), additive on `global` (which doesn't) — each defaulting
   to the matching return role (`standard`/`zero-rated`/`exempt`). An
   entity-specific extra code (an import-GST code, say) needs its own
   `roles` entry passed explicitly; the three canonical ones only need
   one to override a default.
4. The first `configure_nz_gst` call for an entity must supply `basis`,
   `frequency`, and `cycle_end` together. It's safe to call again later
   to change one field or declare a role for a new code — every omitted
   field keeps its current value, and `roles` merges over what's already
   declared rather than replacing it.

## 2. The net + GST split: how a taxable line gets coded

1. A GST-registered entity never posts a taxable amount gross. Every
   taxable P&L line splits into its net amount (tagged with the tax
   code) plus GST to the code's liability account, as its own untagged
   line. Worked example, standard-rated, 15%: a 45.00 expense splits
   into 39.13 to the expense account (tagged `tax:gst-standard`) + 5.87
   to `Liabilities:GST` (no tag) + the 45.00 bank leg. The tax
   fraction is gross × 3/23 (45.00 × 3 ÷ 23 = 5.8695, rounds to 5.87);
   net is gross minus that. Every GST figure this pack computes is
   rounded the same way, per posting, never off a period total — see
   §6.
2. Tag both standard-rated AND zero-rated/exempt lines. A zero-rated or
   exempt line (role `zero-rated`/`exempt`, e.g. `gst-zero`/`gst-exempt`)
   has no GST to carve out — no third posting — but still needs its own
   `tax` tag: an exempt line reaches no box on the return at all
   (outside the GST system, not a zero-rated supply inside it), but a
   bare, untagged P&L line reads as `untagged-pl` on the next preview
   whether or not it was actually meant to be in scope.
3. Never tag: the GST liability account posting itself (a `tax` tag on
   a liability or equity posting is flagged `unexpected-side` rather
   than guessed at — leave that line bare); an IRD GST payment or
   refund (code it to the same liability account, untagged — see §3's
   last step); or anything genuinely outside GST's scope.
4. This is the convention's short form; qunei-bookkeeping states it in
   one line and qunei-code-statement applies it coding a statement line
   by line. This skill is where the return it feeds gets read, cleared,
   and filed.

## 3. Preview, clear, adjust, confirm, file

1. File periods oldest-first: check `briefing.gst.unfiled_periods` and
   never file a later period while an earlier ended period is still
   unfiled. A period skipped this way becomes permanently unfilable —
   `gst.out-of-order-filing` refuses it the moment a later period is
   filed — and its GST never reaches any return.
2. `gst_return` with a `period` (the period-END month, `YYYY-MM`)
   previews that period's full return: all eleven boxes, the per-code
   breakdown, and every warning — on an open period (a live view of
   filing it today) or an already-filed one (a live recompute shown
   beside what was actually filed).
3. Clear EVERY warning before filing — each one names something worth
   understanding, not always something broken:
   - `untagged-pl` — a P&L posting with no `tax` tag at all: genuinely
     out of scope, or a line nobody coded yet.
   - `gross-coded-suspect` — a standard-rated tagged posting whose
     entry never touched the GST account: the classic mistake of
     coding the GST-inclusive total as if it were the net, with no
     split line.
   - `unmapped-code` — the tagged code has no role declared in this
     pack's config (or vanished from tax-codes.txt after posting):
     give it a role with `configure_nz_gst`, or `amend_entry` to
     re-tag the posting with the right code.
   - `unexpected-side` — a `tax` tag landed on a liability or equity
     posting, most often the GST account line itself: `amend_entry`
     to strip it.
   - `no-money-leg` — a tagged entry with no asset-type posting:
     normal for an invoice-basis accrual (revenue + AR, no bank yet),
     but on a payments basis it usually means GST is being claimed on
     money that hasn't moved yet.
   - `unknown-invoice-payment` — a receipt tagged `invoice:<number>`
     names a number no issued invoice matches: check the number, or
     `amend_entry` a mis-tagged receipt.
   - `gst-account-residual` — the GST account's actual movement this
     period doesn't match what the return computed; carries a signed
     `amount` to reconcile against. Three legitimate, non-bug causes:
     it usually equals an IRD payment or refund recorded in the
     period (settling the LAST return, coded untagged per §2.3); on a
     payments basis, a period containing an invoice ISSUE entry
     legitimately shows one too, because that entry's own GST-account
     leg posts at issue while the payments-basis boxes wait for the
     money to move — accrual timing, not an error; and on the invoice
     basis, a residual of a few cents can also arise from per-line vs
     grouped-posting rounding (an invoice's charged GST is rounded per
     line, while the return recomputes per grouped revenue posting) —
     legitimate, not an error. Anything else, `amend_entry`.
   Re-code every genuine mistake via `amend_entry` (qunei-bookkeeping
   §3's replace-never-patch rule) and preview again until the list is
   either empty or fully explained to the human.
4. Manual adjustments — box 9 (other adjustments, e.g. entertainment)
   and box 13 (other deductions, e.g. bad debts) — are typed, not
   computed: pass `box9`/`box13` with a `box9_memo`/`box13_memo` to
   `gst_return` to preview their effect first. A memo isn't optional
   decoration; it's what explains the adjustment to a future session,
   or to Inland Revenue.
5. Get the human's explicit go-ahead before filing anything. Filing is
   a deliberate act, exactly like `issue_invoice` — always allowed
   regardless of posting policy, which describes what the tool will
   let you do, not what you should do unprompted.
6. Only once confirmed: `file_gst_return`, with the SAME period and any
   box9/box13 values just previewed. It heals the books, absorbs
   outstanding late claims (§4), and pins a new version of that
   period's return document — always at the entity's configured basis;
   there is no basis override on this call, unlike the preview (§6).
7. File the identical eleven boxes in myIR — this tool pins the
   ledger's own record of the return, it doesn't transmit anything to
   Inland Revenue itself.
8. When IRD later settles (pays out a refund, or takes payment), code
   that movement to the GST liability account, untagged — never a
   fresh `tax` tag, per §2.3. That's the amount §3.3's
   `gst-account-residual` will most likely explain in whichever period
   it lands.

## 4. Late claims

1. A late claim is journal activity recorded, inside an already-filed
   period's dates, AFTER that period's own filing ceiling — a backdated
   bill, a correction discovered after the fact. `gst_return` on the
   current open period shows what's currently outstanding under
   `breakdown.late_claims`, one row per period still carrying
   something.
2. Nothing is swept until that NEXT return is actually filed — filing
   is what carries an outstanding claim forward into the return being
   filed, automatically, never sideways into some other preview.
3. A REFILE (amending the latest filed period again, before the next
   one files) re-absorbs whatever its own previous version already
   swept up, plus anything new since — nothing already claimed is lost
   or double-counted across versions.
4. `list_gst_returns` shows every filed return, oldest first, so it's
   always clear which period is the latest one still open to a
   same-period refile.
5. A SUPERSEDED period — one with a later period already filed on top
   of it — cannot be refiled directly: `gst.out-of-order-filing` is
   the guard that stops it, not an obstacle to route around. A later
   return's own absorbed rows are measured against the older return
   exactly as it was filed; rewriting that older return would silently
   move the baseline out from under it. A material correction to a
   superseded period goes to Inland Revenue as a formal amendment
   instead — never chase it back into the ledger by refiling.
6. A period still in progress previews cleanly with `gst_return` any
   time — useful for a mid-cycle check. Filing it is what
   `gst.period-open` refuses, until the period's own last day has
   passed.

## 5. Due dates

1. Default: the 28th of the month after the period ends.
2. Two fixed exceptions, regardless of filing frequency: a period
   ending in March is due 7 May; a period ending in November is due
   15 January the following year.
3. Not modeled: a due date landing on a weekend or public holiday is
   never shifted forward here — every date this pack reports is the
   plain calendar date, which is always at least as early as Inland
   Revenue's own real deadline, never later. Treat it as a safe floor,
   not a substitute for confirming the actual date when one looks
   borderline.

## 6. Comparing bases

1. `gst_return`'s `basis:` parameter overrides the entity's configured
   basis for that one preview only — a side-by-side "what would this
   look like on the other basis" lens. It never changes what's on
   file.
2. `file_gst_return` takes no basis parameter at all: filing always
   uses the entity's registered basis from `configure_nz_gst`. Compare
   on preview; file on record.
3. Box figures are per-posting sums, computed and rounded line by line
   (§2.1), never a period total times a fraction — so box 8 here can
   land a cent or two off the paper form's own `box7 × 3/23` shortcut.
   That's not a bug to chase: the ledger's figure is the actual GST
   charged, posting by posting.

See qunei-bookkeeping and qunei-code-statement for where the net + GST
split in §2 actually gets applied day to day, qunei-month-review for the
periodic drift check that reads the current period's `gst_return`
preview for its `breakdown.late_claims` rows and warnings, and
qunei-reports for presenting a filed or previewed return.
