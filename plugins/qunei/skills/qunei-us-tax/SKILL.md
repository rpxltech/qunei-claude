---
name: qunei-us-tax
description: "Use when preparing a US business's year-end Schedule C workpaper in Qunei: mapping accounts to tax lines, choosing the cash or accrual method, declaring sales tax codes, adding adjustments, pinning the year for the preparer, or printing it."
---

# Qunei US tax

The year-end workpaper for a US business kept in USD: each line of its
Schedule C, with the accounts and adjustments behind it, drawn from the
books and handed to whoever prepares the return. Configure the pack once,
keep the books coded as qunei-bookkeeping §8b and qunei-code-statement
§4b say, then preview, clear, adjust, pin and print each year. This skill
assumes qunei-bookkeeping's session rules are already in effect: read
that first if you haven't.

The workpaper is an aid to the preparer's work, not advice, and not a
return: Qunei files nothing (§8). Say so whenever you hand one over.

## 1. Configure before anything else

1. Call `get_briefing` first and check whether its `us_tax` key is
   present. Its absence means `configure_us_tax` has never been run for
   this ledger, and every other tool in this pack (`us_tax_workpaper`,
   `pin_us_tax_workpaper`, `list_us_tax_workpapers`, and `render_report`
   for `us_tax_workpaper`) refuses with `us.not-configured`. The key
   carries `form`, `method`, `last_pinned`, `unpinned` (the ended years
   after the last pinned one, oldest first), `next_due` (§7) and, only
   when nothing can be pinned or dated, `problem`: the pack cannot serve
   the ledger (§1.2), a pinned workpaper cannot be read (§4.4), or
   `us-tax.txt` itself cannot be read. In that last case `form` and
   `method` are null too, and every tool in the pack, `configure_us_tax`
   included, refuses with the error that names the document, since no
   tool writes over a document it cannot read: tell the human. A year
   on `unpinned` can still be refused by `pin_us_tax_workpaper` when its
   own forms are not in the pack yet (§6.3).
2. A ledger made from `us-sole-prop` arrives configured: Schedule C, the
   cash method, every account mapped to its line. Any other USD ledger
   (`us-company`, say) needs `configure_us_tax` with `form` and `method`
   together the first time, then its `map`. Only `schedule-c` has a
   catalog so far: another form can be configured, but its workpaper is
   refused with `us.catalog-unavailable` until its catalog ships. The
   pack needs a ledger kept in USD (`us.currency-unsupported`) and, for
   Schedule C, a financial year ending 31 December
   (`us.year-end-unsupported`). Every tool in the pack refuses a ledger
   that is not, whatever year is asked, and the briefing's `problem` says
   why; `configure_us_tax` refuses every Schedule C call on it too, sales
   tax codes included. A `us-sole-prop` ledger created with another
   currency or year end is refused the same way, and a ledger's currency
   and year end cannot be changed afterwards (qunei-onboard §6). A
   posting in another currency refuses the whole workpaper
   (`query.multi-currency-unsupported`): refused, never pooled.
3. The method is a fact to ask for, never to guess or default: `cash`
   (income counts when it is paid) or `accrual`, whichever the business
   uses for its taxes. Confirm it with the human even where the template
   has set one, and never change it to make a finding go away (§5.5).
4. `ein` is optional and display only, written `NN-NNNNNNN`; an empty
   string removes it. A Social Security number is never accepted and
   never needed: never ask for one.
5. `map` places each account on a line key, a stable name for what the
   money is rather than a printed line number. It merges over what is
   set; name each account once, by its name or its code. `unmap` removes
   mappings. The whole call is checked before anything is saved, and
   every problem is named together in one refusal (`us.invalid`, or
   `account.unknown` for an account the chart lacks). The tool's
   description lists every key and the account types each takes, and its
   answer lists each mapping, each account still unmapped with the key it
   is reported on meanwhile (`reported_as`), and each sales tax code.
6. Map from what the account is for, not its name, and ask when that is
   not clear: a mapping decides which line of the return an amount lands
   on, and the pack never infers a treatment. An account left unmapped
   still counts, on the default key for its report section, and is named
   in an `unmapped-account` finding once it has postings in the year.
7. Meals take a deductible fraction: `{"key": "meals", "deductible":
   "0.50"}` deducts half of the account and leaves the rest
   nondeductible. The fraction is a decimal string strictly between 0
   and 1, and only a deduction key takes one. A meals account mapped
   without one is deducted in full and raises `meals-fraction`. Keep
   entertainment in its own account, mapped `nondeductible`: a fraction
   applies to a whole account, so the two must never share one.

## 2. Sales tax codes

1. No template ships a sales tax code, because US rates differ by place.
   Ask whether the business collects sales tax, in which places, and the
   combined rate for each (state, county and city together, as the
   customer pays it). Never guess a rate.
2. Declare each with `configure_us_tax`'s `sales_tax`, a list of `{code,
   rate, account}`: `code` is `sales-tax` or `sales-tax-<place>`, as
   `sales-tax-ny`; `rate` is a decimal string from 0 up to but not
   including 1, as `"0.08875"` for 8.875%, never a JSON number;
   `account` is a liability account, `Liabilities:Sales-Tax-Payable`
   (which both US templates ship) when left out. It can ride in the same
   call as the form, the method and the map. A code that is already
   declared is never changed here: the same code at another rate or
   account is refused with `us.invalid`, so tell the human.
3. An invoice line takes the code like any tax code (qunei-invoicing).
   The tax posts to the code's liability account and is never income,
   and the invoice prints it as "Sales tax" with its rate: `sales-tax-ny`
   at 8.875% prints `Sales tax 8.875%` (qunei-invoicing §3.6).
4. Qunei does not prepare sales tax returns (§8). Paying the tax over is
   coded straight to the liability account (qunei-bookkeeping §8b).

## 3. Preview, clear, adjust, pin, print

1. **Preview.** `us_tax_workpaper` with `year_end`, one of the ledger's
   year ends up to the year in progress (left out, the most recent year
   that has ended; `us.year-end-invalid` names the nearest). It draws the
   year live from the books: each schedule's lines with their figures,
   the line keys and the accounts and adjustments behind each, the other
   deductions statement, what is listed for the owner's own return, the
   reconciliation from net income per the books to the return's income,
   the `adjustments`, `checks` and `findings`. `checks.reconciles` is
   true by construction, so a false one is a defect: stop and say so.
2. **Clear the findings.** Read every one to the human. A finding never
   refuses a workpaper: it says what to review, not always something
   broken. Each has a `code`:
   - To fix, with the human's yes: `unmapped-account` (map it, step 3),
     `meals-fraction` (§1.7), `not-applicable` (a key that cannot occur
     on Schedule C, such as officer compensation: a sole proprietor's own
     pay and income tax are draws, so record them in an equity account
     mapped `distributions` and re-code what was posted with
     `amend_entry`), `inventory-unchanged` (cost of goods sold with no
     inventory moving: the year-end count is probably missing, so ask for
     it) and the cash method's three (§5.4).
   - To state, with nothing to fix: `owners-return` (amounts that belong
     on the owner's own return, such as interest, dividends, rents and
     charitable gifts, listed for their preparer), `not-prepared`
     (amounts needing a form the pack does not prepare, such as a gain on
     a sale, left off and named for the preparer) and `catalog-provisional`
     (§6).
   Re-code every genuine mistake with `amend_entry` (qunei-bookkeeping
   §3) and preview again until the list is empty or fully explained.
3. **Finish the map.** `configure_us_tax` with `map` (and `unmap`) for
   what the findings and the answer's `unmapped` list show, each account
   confirmed with the human. Preview again: the figures move as the map
   does.
4. **Adjust, with memos.** An adjustment carries what the books do not
   hold, each `{key, amount, memo}`: a home office (`home-office`, line
   30), the standard mileage rate in place of actual car costs
   (`car-truck`, the difference), tax depreciation in place of book
   depreciation (`depreciation`, the difference), or a cash method
   correction (§5.4).
   - `amount` is a decimal string with at most two places, such as
     `"1500.00"` or `"-200.00"`, never a JSON number. It runs in the
     key's own direction: a positive amount to a deduction key deducts
     more, a negative one deducts less.
   - `memo` is one line of at most 200 characters saying why. It is not
     optional decoration: it explains the adjustment to the preparer, and
     it prints on the workpaper.
   - The figure is the human's or their preparer's. Never work out a
     depreciation or a home office deduction yourself and never infer an
     adjustment from the books: the pack takes it as written or refuses
     it.
   - A key must land on a line of the form. The pack refuses one that
     belongs on the owner's return, needs a form it does not prepare,
     cannot occur on Schedule C or is on no line, and `inventories`,
     whose lines come from the books (`us.invalid`).
   - A `meals` adjustment lands as the deductible figure on line 24b, in
     full: the 50% is never applied to an adjustment, so work out the
     deductible amount first. A fraction such as `deductible 0.50`
     belongs to an account's map line (§1.7), never to an adjustment.
   - Pass them to `us_tax_workpaper` to see their effect. A preview
     writes nothing, so pass the same ones to `pin_us_tax_workpaper` to
     record them.
5. **Pin.** `pin_us_tax_workpaper` with `year_end`, once the human says
   the year is ready. `year_end` is required: a year is pinned
   deliberately, never by default. It works the year out as the books
   stand at that moment and saves it as a new version of that year's
   pinned document, with the adjustments used, and answers in the
   preview's shape with `pinned` filled in. Pinning is for the preparer,
   not filing, and locks no period: what is recorded afterwards is drift
   (§4.1). Its two usual refusals:
   - `us.year-open`: the year has not ended (its last day still counts
     as in progress). Preview it with `us_tax_workpaper` meanwhile.
   - `us.catalog-unavailable`: the year has ended but its own forms are
     not in the pack yet (§6). The briefing lists such a year under
     `unpinned` all the same.
6. **Print.** `render_report` with `report` set to `us_tax_workpaper` and
   `at` the year end (left out, the most recent year that has ended)
   renders one print-ready document in the ledger's template and brand
   kit. A pinned year prints from its pinned version, with its own
   figures and the method it was pinned under, its cover saying when it
   was pinned; a year with no pin prints from the books as they stand,
   its cover saying Draft. Pin first, then print: the pinned document is
   the one the preparer is given. A printed pinned year with a
   `pin-drift` finding has lines the books now give another figure for,
   read through the map and the chart as they are now: the pin is stale
   against the current map or chart. The finding names each such line
   with both figures; tell the human and offer a repin (§4.2). Every
   page ends "Prepared from the books for the preparer. Not a filed
   return." Show the `html` as returned and never hand-edit it
   (qunei-reports §8). The call takes only `at` and `prepared_on`, and
   refuses `codes` and every other report's parameters.

## 4. After the pin

1. `us_tax_workpaper` for a pinned year carries `pinned` beside the live
   figures: `pinned_at`, `version`, every pinned line by its id (`p1.31`
   is Schedule C line 31, the net profit) and the adjustments. What was
   recorded after the pin shows as the difference. Report it to the
   human and repin only on their yes, because the new version is what
   the preparer is given next. The printed copy of a pinned year never
   moves for a later posting: only the preview shows it.
2. Pinning a year again writes a new version of the same document, and
   its history (who, when, every prior figure) is the amendment trail. A
   repin with `adjustments` omitted reuses the latest pinned version's,
   so a repin that fixes one figure cannot drop a home office deduction;
   `[]` clears them; a list replaces them. Tell the human which of the
   three a repin does before you make it.
3. To compare a pinned year with the live preview like with like, pass
   the pinned adjustments back to the preview as strings. The preview
   uses only the adjustments it is passed, and `pinned.adjustments` comes
   back as money hashes, so send each `amount` as the decimal string
   inside it (`"1500.00"`, not the `{amount, currency}` object), with the
   same `key` and `memo`. What is left of the difference is what changed
   since: postings recorded after the pin, or an edit to the map, the
   method or the chart.
4. A pinned workpaper that cannot be read was edited by hand or damaged,
   since only `pin_us_tax_workpaper` writes one. The briefing then
   carries `problem` naming it, and the readers stop with a hint. Never
   edit it: ask the human what adjustments that year had (never guess
   them) and pin the year again with them passed explicitly, `[]` only if
   there were none.
5. `list_us_tax_workpapers` lists every pinned year, oldest first: its
   year end, form, when last pinned, how many versions and its headline
   (net profit on Schedule C). It reads the pinned documents only and
   recomputes nothing, so call it freely to say what the preparer already
   has.

## 5. The cash and accrual methods

1. The method is the return's accounting method (§1.3). On `accrual`,
   income and expenses count as the books record them, so an invoice's
   income counts when the invoice is issued.
2. On `cash`, income counts when it is paid. Invoicing posts an invoice's
   income when it is issued, and the pack sets that income aside (credit
   notes with it). Each payment then brings in its share: it is
   apportioned over the invoice's lines, each income account taking its
   net on the invoice over the invoice's total. The sales tax share of a
   payment is left out: it is the liability it always was, never income.
   An unpaid invoice never becomes income.
3. A payment counts only through its receivable posting tagged
   `invoice:<number>`: what `record_invoice_payment` writes, and what
   qunei-code-statement §3 says to write for a customer receipt on a
   statement. A receipt coded to receivables without the tag settles no
   invoice, so its income is never counted.
4. The cash method adds three findings, each naming its entries:
   - `unknown-invoice-payment`: a payment tagged to an invoice that does
     not exist or was never issued, left out of income. Check the number,
     or `amend_entry` a mis-tagged receipt.
   - `cash-bad-debt`: a bad debt written off (postings to an account
     mapped `bad-debts`, or to one mapped `bad-debt-allowance` when the
     receivable is written off against an allowance) is not deductible,
     because the income it would reverse was never counted. A write-off
     settles nothing by itself,
     and one in the same entry as a short payment counts only the cash
     that came in. Adjust the deduction out with a negative `bad-debts`
     adjustment and a memo, or leave it for the preparer: the human
     decides.
   - `cash-accrual`: an expense posted to accounts payable is deductible
     when it is paid, and the books cannot say when that was without
     bills. Check the year each was paid and adjust it into that year, or
     leave it for the preparer. An expense charged to a credit card needs
     no correction: it is deductible when charged. The finding knows
     accounts payable only by its mapping: `us-sole-prop` maps its
     payables account to `accounts-payable` already, but on any other
     ledger map that account to `accounts-payable` (§1.5), or
     `cash-accrual` never fires.
5. Never switch the method to clear a finding: it is for the human and
   their preparer. A pinned year keeps the method it was pinned under.

## 6. A year whose own forms are not final

1. A catalog is the form as the IRS printed it for one tax year, the year
   the financial year begins in, and the pack carries it only once those
   forms are final. A year without a catalog of its own previews on the
   latest earlier year's, so the year in progress can be previewed before
   its forms exist. The workpaper's `catalog` block names the catalog
   used and says `provisional: true`, and a `catalog-provisional`
   finding says so. Tell the human plainly, and that line numbers and
   amounts may move when the year's own forms ship.
2. Such a year cannot be pinned, so nothing reaches a preparer on last
   year's lines: `us.year-open` while it is in progress, then
   `us.catalog-unavailable` once it has ended, until its own catalog
   ships. The message names the latest catalog there is. A year earlier
   than the pack's first catalog has none at all, and is refused the same
   way on the preview too.
3. The briefing lists such a year anyway. From January 2027, for
   example, the year ending 31 December 2026 has ended and is on the
   briefing's `unpinned`, yet `pin_us_tax_workpaper` refuses it as
   provisional until its own catalog ships; so does every later ended
   year, each until its own. Say so: the year is waiting on the forms,
   not on the human. Meanwhile it previews provisionally and prints as a
   Draft whose cover names the catalog it borrowed. That is good for
   planning, but it is on the earlier year's lines: the preparer's copy
   waits for the pin.

## 7. Due dates

1. Schedule C is due 15 April after the calendar year, with the owner's
   Form 1040: 15 April 2027 for the year ending 31 December 2026. A
   workpaper carries it as `year.due`, and the briefing's `next_due` is
   the earliest unpinned year's, even when it has passed: an overdue year
   is the most important fact the briefing carries.
2. Not modelled: a due date that lands on a weekend or a legal holiday
   is never moved to the next business day, and an extension (Form 4868)
   is never applied. Every date this pack reports is the plain 15 April,
   which is never later than the real deadline. Confirm the actual date
   with the preparer when one looks borderline.

## 8. What the pack does not do

Qunei files nothing and locks no period, and you give no tax advice: say
what the books show, which line a figure lands on and what the pack did
with it, and send the tax question to the preparer. Outside the pack,
with the path for each:

1. **Filing.** There is no IRS e-file and no official form layout. The
   preparer keys the workpaper into tax software.
2. **State and local returns.** State and local income and franchise tax
   returns are not prepared: the preparer works from the federal
   figures.
3. **The owner's own return.** Schedule SE, the Section 199A deduction
   itself and estimated tax (Form 1040-ES) are not prepared. The
   workpaper lists what flows to the owner's return (`owners-return`).
4. **Depreciation, amortization, home office and vehicles.** Form 4562,
   Form 8829 and Schedule C Part IV are not prepared: carry the result in
   as adjustments with memos (§3.4).
5. **Disposals and rental real estate.** Form 4797, Schedule D, Form 8949
   and Form 8825 are not prepared. A gain or loss on a sale is a
   `not-prepared` finding that names the accounts, and rents mapped
   `rental-income` are listed for the owner's return (`owners-return`).
6. **Beyond Schedule C.** International reporting (Schedules K-2 and
   K-3), shareholder basis (Form 7203), the large-entity reconciliation
   (Schedule M-3) and the corporate estimated tax penalty (Form 2220) are
   the preparer's. The S corporation, partnership and C corporation
   forms (1120-S, 1065 and 1120), with their owners and K-1s, are not
   prepared: the pack's workpaper is Schedule C.
7. **Carryforwards and questionnaires.** Net operating loss and
   charitable contribution carryforwards are not tracked: the preparer
   tracks them. Schedule B and the other questionnaires are the
   preparer's to answer.
8. **Payroll, 1099s and sales tax returns.** Qunei has no payroll
   engine: a payroll provider's journals are the interface. Contractor
   1099s and sales tax returns are not prepared.
9. **US bank feeds.** There are none: code statements through
   qunei-code-statement and the inbox (qunei-intake).
10. **Whole dollars, and locking.** Figures stay to the cent, as the
    books are; the preparer rounds. Pinned workpapers pin numbers without
    blocking postings, so what is posted afterwards is drift (§4.1).

See qunei-bookkeeping §8b and qunei-code-statement §4b for how a US
ledger's lines are coded so the map reads them right, qunei-onboard for
choosing the template and asking the method and sales tax questions,
qunei-invoicing for the invoices whose payments the cash method reads,
qunei-month-review §5c for the periodic read of the year in progress, and
qunei-reports for presenting the printed workpaper.
