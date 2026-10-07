---
name: qunei-nz-charities
description: Use when preparing a New Zealand charity's or incorporated society's annual performance report in Qunei under the Tier 3 (NFP) or Tier 4 (NFP) Standard, choosing its tier, mapping its accounts to the standard's categories, writing its entity information and statement of service performance, or reviewing and rendering the finished report.
---

# Qunei NZ charities

The performance report a registered charity files with Charities Services
each year, drawn from the ledger under the XRB's Tier 3 (NFP) Standard
(accrual) or Tier 4 (NFP) Standard (cash), which apply to periods
beginning on or after 1 April 2024. The report is laid out as the XRB
staff's own Tier 3 and Tier 4 templates lay it out: their parts, lines,
notes and default sentences. Qunei computes every statement and note
figure from the books; the charity supplies the words. This skill
assumes qunei-bookkeeping's session rules are in effect.

## 1. Tier first, and ask rather than assume

1. Start with `get_briefing`, then ask the person which tier the entity
   reports under. Tier 4 is for an entity without public accountability
   whose annual operating payments are under $140,000; Tier 3 for one
   whose total expenses are less than $5 million, as the Tier 3
   template's own words have it. On Tier 3 the report's `tier-threshold`
   finding flags expenses of $5 million or more for a person to confirm.
   XRB A1 sets the exact test, including how earlier years count, and an
   entity may choose a higher tier, so the tier is the person's (or
   their accountant's) decision, never yours. Say so if this year's
   figures look near a limit.
2. A new charity's ledger starts from the `nz-charity` template
   (`init_entity`, template `nz-charity`). Its chart is already mapped to
   the standard's categories, tier 3 is set, and it declares a `fund`
   dimension for restricted funds. Change the tier with
   `configure_nfp_reporting` if the charity reports under Tier 4.
   A ledger made from an earlier `nz-charity` template may map
   `Assets:Accounts-Receivable` with `cash commercial-received`
   (`configure_nfp_reporting`'s answer lists each mapping with its cash
   line), which reports a receipt there that is not traced to an invoice
   as commercial sales, with no finding. Unless everything the charity
   invoices is commercial sales, drop that cash line by mapping the
   account again as `debtors` alone:
   `{"Assets:Accounts-Receivable": "debtors"}`. Keep the ledger's other
   cash lines, such as `Assets:Grants-Receivable`'s
   `general-grants-received`.
3. An existing ledger (an `nz-company` one, say) is set up with
   `configure_nfp_reporting`: the tier, and a `map` from each account to
   its category. The tool's description lists every category and cash
   line. Its answer lists what is still unmapped; an unmapped account
   reports in its type's "other" line until you map it.
4. The pack needs a ledger in NZD and at least one account mapped to
   `bank`. Map an overdraft account to `overdraft`, and petty cash or
   undeposited takings to `cash-on-hand`. Map a term deposit by the
   tier. On Tier 4 every term deposit is `term-deposits`, which is cash:
   the Tier 4 template counts any term deposits in the opening and
   closing balances. On Tier 3 a deposit with an original maturity of 90
   days or less is `term-deposits` and a longer one is `investments`, as
   the Tier 3 cash policy the report prints says. Map them again when
   the tier changes. Tier 4's "Represented by" shows each kind of cash.

## 2. Mapping well

1. Map from what the account is for, not its name. Grants received
   without conditions are `general-grants`; a grant for buildings or
   equipment is `capital-grants`; a contract to deliver services is
   `government-contracts` or `non-government-contracts`; koha and
   fundraising event income are `donations`.
2. Spending on the charity's own programmes is
   `service-delivery-expenses`; wages, KiwiSaver and ACC levies are
   `employee-expenses`; volunteer reimbursements and costs are
   `volunteer-expenses`; overheads that are none of these are
   `other-expenses`.
3. A balance-sheet account that cash moves through on its way to income
   or spending (a receivable, a payable, prepayments, inventory) cannot
   be traced back automatically, so its cash is reported as other cash
   received or paid. The exception is a receipt of an invoice: one
   recorded with `record_invoice_payment` carries the invoice's number
   and is reported as what was invoiced. An invoice set off against a
   bill moved no cash, so it is not reported. When all of an account's
   cash is one kind, give it a cash line:
   `{"category": "debtors", "cash": "general-grants-received"}`. An
   account that carries more than one kind of payment, such as bills and
   expense claims, has no single right cash line: leave it without one,
   and its payments appear under other payments. Only a balance-sheet
   account takes a cash line; an income or expense account's cash is
   reported in its own category's line. The report's findings name each
   account whose cash went to the other lines, this year or last.
4. A fee or a discount taken out of a deposit or a payment is coded to
   its own income or expense account in the same entry, as a bank
   reconciliation books it. The cash statements then report it as its
   own payment or receipt, and what it was taken from in full: takings
   of 1,000 and 150 of GST banked as 1,100 after a 50 fee are 1,000 of
   sales and 150 of GST received, and 50 paid, so Net GST agrees with
   the GST returns. Code a fee's or a discount's own GST beside it in
   the same entry, and it goes with it: Net GST nets it off, as the
   return does. Map a discount kept on an income account to the
   category of the sales it discounts (`commercial-revenue` for a
   discount on sales), so that line nets; under another revenue category
   it prints that line negative. A fee taken out of an asset sale's
   proceeds is the exception: it stays in the sale, which is reported at
   what it brought in, with no fee line. A sale's or a trade-in's GST is
   reported at its full amounts, as the return has it, unless part of
   the price is left on account or financed.
5. Property, plant and equipment is mapped by class (`ppe-land`,
   `ppe-buildings`, `ppe-vehicles`, `ppe-furniture`,
   `ppe-office-equipment`, `ppe-computers`, `ppe-machinery`, or `ppe`
   for any other), and each accumulated depreciation account to its
   asset's class: Tier 3's Note 5 moves each class from opening to
   closing, and Tier 4's significant assets name land and buildings and
   vehicles. A revaluation reserve is `ppe-revaluation-reserve` or
   `investment-revaluation-reserve`; money set aside for a purpose is
   `reserves`. An investment held at market value says so:
   `{"category": "investments", "valuation": "market"}`; an income
   account for gains and losses on investments is `investment-gains`, so
   the investments note can tell a gain from income reinvested. Money
   lent to others is `loans-made`; money borrowed from people or bodies
   other than a lender is `loans-from-others`; money held for someone
   else is `held-for-others`.
6. Restricted funds are tagged as they are coded (`fund:<name>`), and
   reported with `profit_and_loss` and its `tags` filter, or a budget
   with a filter (qunei-reports). A fund's money still appears in the
   statements' categories.

## 3. The narrative is the charity's, never yours

1. `update_performance_report` sets entity information, the statement of
   service performance, who approves the report, the accounting policies
   and the words of the notes. Each argument replaces that part whole, so
   send every paragraph to keep. Send one paragraph per list item, with no
   line breaks and no markdown inside it: a value with a line break is
   refused and nothing is saved, and the report prints the words as plain
   text, so markdown prints as typed. In `policies`, the first `" | "` in a
   policy separates its title from its text, so an untitled policy cannot
   contain `" | "`.
2. Write only what the person tells you, or what last year's report
   says and they confirm. Never invent an achievement, a measure, a
   figure or a related party. If a part is unknown, leave it out and say
   so: the report's findings list what is missing.
3. Entity information follows the template's form. Tier 3 asks for the
   name, the entity identifier (charity registration number or NZBN),
   the type of entity, its purpose or mission, its structure, its
   governance arrangements, the other entities it controls (or that it
   controls none) and its reliance on volunteers and donated goods or
   services; Tier 4 for the name and type.
4. The statement of service performance: on Tier 3, the medium to long
   term `objectives`; on both, what the entity did this year
   (`activities`) and, where it can be counted, `measures` with this
   year's and last year's quantities as the entity counted them.
5. `approvers` names those who approve the report for those charged with
   governance (name and position); their names print under the
   signature lines. Qunei records no approval date: the Date and
   Signature lines print blank, to be written by hand when the report is
   approved. The finding `approver-missing` says when no one is named.
6. Set `gst` (`registered` or `not-registered`): the GST policy follows
   it, in the template's words. Leave `basis` out unless the person wants
   their own wording; the default is the template's. `income_tax:
   ["exempt"]` prints the template's income tax exemption, if the charity
   is exempt. When the entity began during the year, set `commenced` and
   the basis says so.
7. Some notes print the template's words for none when nothing is
   written: commitments, contingent liabilities, related party
   transactions and events after the balance date (Tier 3), related
   party transactions (Tier 4), and on Tier 3 "no changes in accounting
   policies". Ask about each; the finding `default-statement` lists the
   ones printed. Write what happened when anything did.
8. Other notes take the charity's words when they apply: the source and
   date of valuations, heritage assets not recorded, each reserve's
   nature and purpose, deferred revenue's conditions, goods or services
   in kind, assets used as security, assets held on behalf of others,
   doubts about continuing to operate, and corrections of errors.
   Anything else goes in `notes` with a title.

## 4. Review before anyone files

1. Call `performance_report` for the year end (it defaults to the most
   recent). Check `checks`: `position_balances` (Tier 3) and
   `cash_reconciles` are always true on balanced books, so a false one
   means something is wrong in the ledger itself. Stop and say so.
2. Read `findings` to the person and work through each one, by its
   `kind`:
   - `unmapped-account`: an account nobody mapped, reported in its
     type's "other" line. Map it with `configure_nfp_reporting`.
   - `control-account-cash`: a balance-sheet account whose cash went to
     the other lines, this year or last. Give it a cash line when all of
     its cash is one kind; a receivable or payable that carries more
     than one kind keeps none.
   - `cash-line-ignored`: a cash line saved on an income or expense
     account, which the report sets aside. Map the account again by its
     category alone with `configure_nfp_reporting`.
   - `balances-brought-forward`: each entry the report takes as
     balances brought forward from earlier records, by its date and
     memo; its cash is opening cash, not cash received or paid. Confirm
     each with the person. One that is really a receipt, a payment or a
     transfer between funds is amended with `amend_entry` to post to the
     income or expense it belongs to, or its transfer is posted as an
     entry of its own.
   - `narrative-missing`, `gst-unset` and `approver-missing`: words the
     template requires, the GST policy or the approvers, not yet
     written with `update_performance_report`.
   - `default-statement`: the notes that print the template's words for
     none. Confirm each with the charity.
   - `reserves-undescribed`, `deferred-revenue-undescribed`,
     `valuation-undescribed` and `asset-values-undescribed`: balances the
     templates ask the charity to explain.
   - `first-year`: no entries before the year began, so last year's
     column is all zero. If the entity began this year, set `commenced`.
   - `tier-threshold`: this year's figures at or over the tier's limit.
     The tier is the person's decision.
3. Present the statements in the payload's own order, row by row
   (`heading`, `subheading`, `line`, `total`, `grand_total`), with the
   current year beside last year and each line's `note`. Never recompute
   a figure or net two lines.
4. The layout follows the XRB staff templates, which are guidance, not
   the standards themselves. Ask the person (or their reviewer or
   auditor) to confirm the presentation before the report is approved
   and filed.

## 5. The finished document

1. `render_report` with `report: performance_report` and `at` set to the
   year end renders the whole report in the ledger's template and brand
   kit. Show its `html` as returned; it is the document, never one to
   edit by hand.
2. To keep the report exactly as issued, save it (qunei-reports explains
   saved reports). Filing it with Charities Services is the person's own
   act: Qunei does not file the annual return.
