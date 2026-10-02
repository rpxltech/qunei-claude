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
   whose total expenses are $5 million or less. XRB A1 sets the exact
   test, including how earlier years count, and an entity may choose a
   higher tier, so the tier is the person's (or their accountant's)
   decision, never yours. Say so if this year's figures look near a
   limit.
2. A new charity's ledger starts from the `nz-charity` template
   (`init_entity`, template `nz-charity`). Its chart is already mapped to
   the standard's categories, tier 3 is set, and it declares a `fund`
   dimension for restricted funds. Change the tier with
   `configure_nfp_reporting` if the charity reports under Tier 4.
3. An existing ledger (an `nz-company` one, say) is set up with
   `configure_nfp_reporting`: the tier, and a `map` from each account to
   its category. The tool's description lists every category and cash
   line. Its answer lists what is still unmapped; an unmapped account
   reports in its type's "other" line until you map it.
4. The pack needs a ledger in NZD and at least one account mapped to
   `bank`. Map an overdraft account to `overdraft`, a term deposit of 90
   days or less to `term-deposits` (a longer one is an `investments`),
   and petty cash or undeposited takings to `cash-on-hand`: Tier 4's
   "Represented by" shows each kind.

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
   received or paid. When most of an account's cash is one kind, give it
   a cash line: `{"category": "debtors", "cash": "general-grants-received"}`.
   The report's findings name each account this applies to.
4. Property, plant and equipment is mapped by class (`ppe-land`,
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
5. Restricted funds are tagged as they are coded (`fund:<name>`), and
   reported with `profit_and_loss` and its `tags` filter, or a budget
   with a filter (qunei-reports). A fund's money still appears in the
   statements' categories.

## 3. The narrative is the charity's, never yours

1. `update_performance_report` sets entity information, the statement of
   service performance, who approves the report, the accounting policies
   and the words of the notes. Each argument replaces that part whole, so
   send every paragraph to keep.
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
   signature lines.
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
2. Read `findings` to the person and work through them: unmapped
   accounts, control accounts whose cash went to the other lines,
   missing words, a GST policy not set, the template's words for none,
   balances that need explaining (reserves, deferred revenue, market
   valuations, significant assets), a first year with no comparative
   figures, a tier threshold.
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
