---
name: qunei-reports
description: Use when asked to present, report, or visualize a Qunei entity's financial statements — in-chat tables, a CSV, an HTML dashboard, or a rendered, brand-styled report document — including restyling the report template or setting the ledger's brand kit (logo, colours, fonts).
---

# Qunei Reports

Presentation layer over the query tools: render statements for whoever
asked, in whatever format they need. This skill never invents a number —
it renders what the query tools return, this session, every time.

## 1. Numbers always come from query tools, this session

1. Every figure in any report comes from `trial_balance`,
   `profit_and_loss`, `balance_sheet`, `aged_receivables`,
   `account_ledger`, or `find_entries` called in the current session —
   never from memory, never from a number you or the human mentioned
   earlier in this conversation or a prior one. One scoped exception:
   the HTML dashboard's `{{PENDING_DRAFTS}}` block (§11.3) is built from
   `list_drafts` (or `get_briefing`'s `work_state.pending_drafts`) —
   none of the query tools above report on unposted drafts, so this
   block is the one place in a report that draws from somewhere else,
   still queried live, this session.
2. Re-query even if you produced a very similar report minutes ago in
   the same session. The books may have changed; a cached figure is a
   wrong figure waiting to happen.
3. A comparative comes from the statement's own `compare` parameter
   (§5), never from two calls and a subtraction. The tool queries both
   windows, unions their accounts, re-runs every subtotal formula over
   the comparative window, and hands back `comparative`, `variance` and
   `variance_pct` already computed. Arithmetic of your own will disagree
   with it the first time an account is active in only one of the two
   periods.
4. The working trial balance's four columns (`opening`, `debits`,
   `credits`, `closing`) and its `retained_earnings_brought_forward`
   line are `trial_balance`'s own. Never recompute one column from
   another, never derive an opening balance by calling the tool again at
   an earlier date, and never net the two movement columns yourself —
   `closing` already is that net.
5. If a query fails with `query.multi-currency-unsupported`, that report
   genuinely spans more than one currency — don't work around it by
   guessing a total; query a single currency's accounts, or use
   `find_entries` instead.

## 2. Pick an idiom, and say which one you picked

1. Accountant or tax context: use Dr/Cr idiom (debits and credits, as
   the underlying data already carries).
2. Owner or casual context: use money in/out phrasing (plain language,
   no accounting jargon).
3. Either way, state out loud which idiom you're using before the
   figures — the human should never have to guess whether a number meant
   a debit or a credit.

## 3. In-chat: clean markdown tables

1. Default output is a markdown table built directly from the query
   tool's JSON — account, amount, and whatever else the audience rule
   calls for. No prose paragraph pretending to be a table.
2. For `profit_and_loss` and `balance_sheet`, walk `sections` in payload
   order and render each one as it comes: the section's `title`, then
   its `lines` (`code`, `account`, `amount`) in the order given, then
   that section's `total`. The order is the statement's order — Revenue,
   Cost of sales, Other income, Operating expenses, Income tax; Current
   assets, Non-current assets, Current liabilities, Non-current
   liabilities, Equity — with any Unmapped section standing inside the
   family it belongs to (§3.6). Never re-sort it, and render a section
   that came back with no lines as an empty section rather than dropping
   it.
3. Every line amount is already signed for the section it sits in, so a
   positive figure reads normally on that section's own side. A negative
   one is a contra — an expense-type "Discounts allowed" mapped under
   Revenue displays negative and reduces the revenue total, which is
   exactly how an accountant presents it. Print the sign as given; never
   flip it.
4. Then print the `subtotals` block, by name, in the order the payload
   carries them: `gross_profit`, `total_income`, `total_expenses`,
   `income_tax`, `net_profit_before_tax`, `net_profit` on a P&L;
   `total_assets`, `total_liabilities`, `net_assets`, `total_equity` on
   a balance sheet. Each one is the sum of the SECTION TOTALS on its own
   side of the page — `total_income` of the credit-side sections
   (Revenue, Other income, Unmapped income), `total_expenses` of the
   debit-side ones (Cost of sales, Operating expenses, Unmapped
   expenses, Income tax), and `total_assets`, `total_liabilities`,
   `total_equity` of their own families, equity including the two
   computed lines below. So a subtotal always foots the column printed
   above it, whichever section an accountant chose for a contra: the
   "Discounts allowed" of item 3 counts against `total_income` where it
   is printed and never into `total_expenses`, and net profit is the
   same figure either way. Print them as returned — a reader who adds up
   the column lands on the same number, and if they ever do not, that is
   a finding worth raising.
5. A balance sheet's Equity section ends with two computed lines,
   `Retained earnings brought forward` and `Current year earnings`, each
   carrying `computed: true` and NO `code` — they are not accounts. The
   ledger has no closing entry, so the sheet derives them; both fold
   into the Equity section's total and into `total_equity`. Render them
   inside Equity, after the declared accounts, with the code column
   blank. `financial_year` names the year that split was taken across,
   and `balanced: false` means assets did not equal liabilities plus
   equity: report it as a finding, never as a rounding note.
6. There is one Unmapped section per account TYPE, and each one stands
   where it foots: `pl.unmapped-income` ("Unmapped income") straight
   after Other income, `pl.unmapped-expenses` ("Unmapped expenses")
   straight after Operating expenses, and `bs.unmapped-assets`,
   `bs.unmapped-liabilities` and `bs.unmapped-equity` at the end of
   their own families. A section appears only when some account of that
   type has no usable mapping, so a chart with one unmapped expense gets
   one such section and no others. Render each where the payload puts
   it, never as a footnote at the bottom: the subtotal below it counts
   its lines, so a reader has to have seen them first. Each carries a
   real `total` (one type, one side, so the sum means something) and
   each line keeps its `type`, which is what says why it is there.
7. The top-level `unmapped` list is present either way, and is empty on
   a fully mapped chart. When it has entries — each `{ code, account,
   type, report }`, with `report` carrying whatever unusable value is on
   record — close the report by offering the mapping chore of §4: one
   `update_account` call per entry, from the ten keys named there. Say
   which key you would propose for each and why, and wait. Never guess a
   mapping silently.

## 4. The mapping chore: a report key for every account

1. An unmapped account is not a missing number: it sits in its own
   type's Unmapped section rather than in the section an accountant
   would have chosen for it, and because that section stands inside its
   own family it counts in the subtotals exactly as a mapped one does. The fix is one `update_account` call per
   account, `ref` naming the account (name or code) and `report`
   carrying one of ten keys. The five `*.unmapped-*` keys are not among
   them: they are computed, and `update_account` refuses them.
2. The ten keys, and what belongs under each:
   - `pl.revenue` — trading income: what the business actually sells.
   - `pl.cost-of-sales` — the costs that move with that trading.
   - `pl.other-income` — income from outside the trade: interest,
     rebates, one-offs.
   - `pl.operating-expenses` — the overheads of trading: rent, wages,
     software, insurance, everything run to keep the doors open.
   - `pl.income-tax` — tax on the profit itself. Not GST, not PAYE.
   - `bs.current-assets` — assets expected to turn over inside twelve
     months: bank, receivables, stock, prepayments.
   - `bs.non-current-assets` — assets held longer than that: equipment,
     vehicles, intangibles.
   - `bs.current-liabilities` — owed inside twelve months: payables, GST,
     tax payable, credit cards, the current portion of a loan.
   - `bs.non-current-liabilities` — owed beyond twelve months: term
     loans, shareholder loans not on demand.
   - `bs.equity` — owner capital, drawings, and reserves.
3. `pl.*` keys belong to income and expense accounts, `bs.*` keys to
   asset, liability and equity accounts. Crossing statements is refused
   as `report.mapping-type-mismatch`, and a key outside the ten as
   `report.mapping-unknown`. Inside the right statement the section is
   free, which is what a contra needs: an expense-type account mapped to
   `pl.revenue` is legal, and displays negative there. A cross-family
   mapping on the balance sheet is legal for the same reason — a
   liability-type provision for doubtful debts presented under
   `bs.current-assets` — and the subtotals follow the page, so the sheet
   still foots and still balances (§3.4).
4. Contra accounts sit where the accountant says, not where the type
   suggests. Ask; don't guess. Same for anything genuinely ambiguous — a
   "Sundry" or suspense account, a loan that could be current or not, a
   deposit that could be either side of twelve months. One question
   beats a quietly wrong statement.
5. One `update_account` per account, one at a time, so a refusal names
   the account it belongs to. `clear_report: true` on the same tool
   removes a mapping that was set wrong.
6. Re-run the report afterwards to confirm the fix: the account has
   moved into its section, the `unmapped` list has shrunk, and once no
   account of a type is left unmapped that type's Unmapped section is
   gone from `sections` entirely. Never hand-edit the earlier output to
   show the change.
7. `list_accounts` reports every account's current `report` value, which
   is the quickest way to size up the whole chore before starting it.

## 5. Comparatives: one call, not two

1. `profit_and_loss` and `balance_sheet` both take `compare`, which is
   `"prior-period"` or `"prior-year"` and nothing else (anything else is
   refused as `grammar.malformed-line`). `trial_balance` has no
   comparative.
2. `prior-period` is the preceding span of the same shape: a P&L over
   whole calendar months steps back by that many months (August against
   July, a quarter against the quarter before it, a financial year
   against the year before it), and any other window steps back by its
   own length in days. A balance sheet's prior period is the month end
   BEFORE its as-of date. `prior-year` is the same dates one year
   earlier.
3. Never state the comparative window from your own date arithmetic: the
   payload names it. A compared P&L carries `comparative: { period: {
   from, to } }`, a compared balance sheet `comparative: { as_of }`.
   That is the column heading.
4. With `compare` set, every line gains `comparative`, `variance` and
   `variance_pct` beside its `amount`; every section carries the same
   three beside its `total`; and every subtotal becomes
   `{ amount, comparative, variance, variance_pct }`, where `amount`
   now holds the money value an uncompared subtotal carried directly.
   Render current, comparative and variance as three columns, with the
   percentage beside the variance.
5. `variance` is current minus comparative, signed the way the column
   beside it is signed, so it reads in the direction the reader expects.
   `variance_pct` is a decimal string (for example `"33.3"`), measured
   against the comparative's magnitude — which is why a swing up out of
   a loss reads positive rather than inverted.
6. `variance_pct` is null when the comparative is zero: no magnitude,
   no percentage. Show that cell as "n/a" or leave it blank. Never 0%,
   never an infinity, and never a percentage worked out some other way.
7. An account active in only one of the two periods still appears, with
   a zero on the side where it had no activity. Those rows are usually
   the interesting ones — a line that started or stopped — so don't drop
   them, and say which of the two windows it was live in.
8. On a balance sheet the two computed equity lines are compared like
   any other line, from the comparative window's own earnings split. An
   Unmapped section is compared like any other section, lines and total
   alike — and it appears whenever EITHER window has an unmapped account
   of that type, so a section can come back with a zero on one side.

## 6. The working trial balance

1. `trial_balance` is a working paper, not a list of balances. Each line
   is `{ code, account, type, opening, debits, credits, closing }`: what
   the account carried into the window, what was posted to each side
   inside it, and what it carried out. `closing` is already computed as
   `opening + debits - credits`; don't recompute it.
2. The window is a financial year by default. `at` picks the year, which
   comes back as `financial_year`; `from` defaults to that year's first
   day and is echoed in the payload. An explicit `from` may narrow the
   window but must stay inside that year — outside it the call is
   refused as `query.window-outside-financial-year`, whose message names
   the year to pick from. Always state the window you rendered: `from`
   and `as_of` are both there.
3. Opening means two different things by `type`, and the report only
   reads correctly if you say so. Asset, liability and equity accounts
   open with everything ever posted to them up to the day before `from`,
   because their balances are cumulative and no year rolls them over.
   Income and expense accounts open with the financial year to date
   only, so on the year's first day they open at zero: last year's
   trading is not part of this year's performance.
4. What those accounts earned in EARLIER years therefore belongs to no
   account's opening balance, so it comes back as one computed line,
   `retained_earnings_brought_forward`, sitting beside `lines` rather
   than in them, with no `code` and `computed: true`. Every posting in
   it predates the window, so it has no movement and its `opening` and
   `closing` are the same number. Render it under the accounts, as the
   last line before the totals.
5. SIGNS, and the one trap worth naming out loud. `opening` and
   `closing` are raw signed amounts on every line, the brought-forward
   line included: positive is a debit, negative is a credit. So a
   retained credit balance shows NEGATIVE here (`-100.00`, for example),
   while the balance sheet shows that same figure credit-positive on its
   `Retained earnings brought forward` equity line (`100.00`). Both are
   right; neither is the other's sign error. Never carry a figure across
   from one statement to the other, and never restate one in the other's
   convention without saying that is what you did. The `debits` and
   `credits` columns are the exception: both are reported positive,
   because a column that names its own side has no use for a sign.
6. `totals` is `{ opening: { debits, credits }, debits, credits,
   closing: { debits, credits } }` — the opening and closing columns
   split into their two sides, the movement columns already one-sided.
   Print all six figures; the two sides of a column are what a reader
   checks the page with.
7. A line appears when ANY of its four figures is nonzero, not when its
   closing balance is. An account that moved and came back is precisely
   what a working paper is for, so leave those rows in and let the zero
   closing sit beside the movement that produced it.
8. `balanced: false` is a finding, not a formatting note: the closing
   column does not sum to zero, or the two movement totals disagree.
   Lead the output with it in plain words and hand it to
   qunei-month-review or to the human. Never add a balancing figure of
   your own to make the page tie.

## 7. Aged receivables

1. `aged_receivables` is the debtors report: every open invoice as at a
   date, bucketed by how far past its `due` date it was on that date —
   an overpaid one excepted, which is always `current` (item 5).
   `at` defaults to today — pass it whenever the report is being drawn
   at a period end, so it matches the statements beside it. The entity
   must have run `configure_invoicing`; a ledger that never has is
   refused with `invoicing.not-configured`, not answered with an empty
   report.
2. `basis` is always `"due-date"` and `buckets` is always `["current",
   "1-30", "31-60", "61-90", "over-90"]` — print them in that order,
   left to right. Due today counts as `current`; day 31 opens `1-30`'s
   successor and so on, and `over-90` starts at day 91.
3. `customers` is one group per (customer, currency) pair, sorted by
   both. A customer holding balances in two currencies gets two groups
   and never one pooled sum — currencies are never added together
   anywhere in this payload. Each group carries `customer` (the slug),
   `name` (nil if the customer block has since been deleted from the
   registry), `currency`, its `invoices`, and a `totals` map with all
   five bucket names plus `total`.
4. Each invoice row is `{ number, date, due, days_past_due, bucket,
   credit, total, balance }`. `total` is what the invoice was raised
   for; `balance` is what is still outstanding on it as at `at`. Render
   `balance` in the bucket columns — `total` is context, not the debt.
   `days_past_due` is floored at zero, so a not-yet-due invoice reads 0
   rather than a negative. Put each row in the bucket the payload gives
   it (`bucket`), never one you compute from `days_past_due` yourself:
   for an overpayment the two deliberately disagree (item 5).
5. `credit: true` marks an OVERPAYMENT: a negative balance, an invoice
   the customer has paid more against than it was raised for. It is
   always in the `current` bucket, whatever its due date, and its
   `days_past_due` still states the true count beside it. That is the
   rule, not an accident: a credit left in a due-date bucket cancels
   real overdue debt inside it, and a customer with 150.00 sixty days
   late and 150.00 overpaid would print a totals row of zeros while the
   money is genuinely late. Show the credit as the negative it is (in
   parentheses if that is the house style for this report) and call it
   out in words — it is money owed back, sitting in a report about money
   owed to us, and it still reduces the customer's own total, which is
   the number that should net.
6. `totals` (top level) is one row per currency over the same groups,
   with the same five bucket names plus `total`: the grand total line at
   the foot of the report, one line per currency.
7. `control` is the reconciliation, one row per currency:
   `{ currency, account, balance, aging_total, difference, reconciles }`.
   `account` is the configured receivables account, `balance` is what
   that account holds as at `at` on the primary book, `aging_total` is
   the aging's own grand total, and `difference` is the first minus the
   second. Print this row. A report of what customers owe, next to the
   ledger account that is supposed to say the same thing, is the one
   check that makes the page trustworthy.
8. `reconciles: false` is a finding, not a formatting note — the same
   rule §6.8 states for `balanced: false`. It means receivables were
   raised or cleared without an invoice, or an invoice was tagged wrong.
   Lead with it in plain words, quote the `difference`, and hand it to
   qunei-month-review or the human. Never adjust a bucket to make the
   page tie.
9. The `customer` parameter narrows `customers` and `totals` and NOTHING
   else. The control row always spans the whole ledger — that is its
   job, and re-scoping it to agree with a filtered slice would destroy
   the only check on the page. So a single-customer report normally
   shows a `control.aging_total` larger than its own visible groups,
   still reading `reconciles: true`. Say which customer the report was
   filtered to, and never present the control row as that customer's
   balance.
10. `corrupt` lists `{ ulid, corrupt: true }` for any invoice file that
    could not be parsed. It is almost always empty. When it isn't, say
    so on the page: those invoices are in none of the figures above.

## 8. Rendered documents: one `render_report` call

1. `render_report` returns the finished report as a complete HTML
   document. `report` names which one — `profit_and_loss`,
   `balance_sheet`, `trial_balance` or `aged_receivables`, and nothing
   else (anything else is refused as `grammar.malformed-line`, naming
   the four). One call is the whole job: the server re-runs the query,
   builds the statement table, applies the ledger's own template and
   brand kit, and hands back the document.
2. The payload is `entity`, `report`, `title`, `period`, `source`,
   `template_version`, `brand`, `brand_version`, `html`. `html` is the
   document. `title` is what it prints ("Profit and loss", "Balance
   sheet", "Trial balance", "Aged receivables"). `period` is what was
   actually drawn — `{ from, to }` for a P&L, `{ from, as_of }` for the
   trial balance (which is drawn AT a date over a movement window, and
   prints both), and the as-of date alone for the balance sheet and the
   aging — so quote the window from here rather than from the parameters
   you sent, which is how you catch a default you did not set (a trial
   balance's `from` defaults to the start of the financial year). `source` and `template_version` say which chrome
   produced these bytes, `brand` and `brand_version` which kit; each
   pair reads `"default"` with a null version until the ledger has
   written one of its own.
3. Deliver it as written. Save the `html` to a file next to the books,
   or publish it as an Artifact when the client supports one (§11.4).
   PDF is the browser's own print dialogue — Print, then Save as PDF:
   the shipped chrome carries `@page` and page-break rules for exactly
   that, and there is no PDF tool on this server.
4. Never hand-edit the returned markup, and never author report markup
   of your own — the rule qunei-invoicing §3.1 holds for invoices, for
   the same reason. A wrong figure is fixed in the books, a wrong layout
   in the template (§9), a wrong colour in the brand kit (§10); then
   render again. The next render replaces the document outright, so an
   edited copy is a page that no longer matches the ledger it claims to
   come from.
5. Each report takes its own parameters, and a parameter the chosen
   report cannot consume is REFUSED rather than ignored:
   - `profit_and_loss` — `from` and `to` (both required), `book`,
     `view`, `as_at`, `compare`.
   - `balance_sheet` — `at` (required), `book`, `view`, `as_at`,
     `compare`.
   - `trial_balance` — `at` (required), `from`, `book`, `view`, `as_at`.
   - `aged_receivables` — `at` (defaults to today), `customer`.

   `codes` and `prepared_on` are accepted by all four. So `customer` on
   a P&L, `compare` on a trial balance, or `view` on an aging comes back
   as `grammar.malformed-line` naming the parameter, rather than being
   quietly dropped while you believe it took effect. Missing required
   parameters and refused ones arrive together in ONE error, so read the
   whole list before retrying. `as_at` still only takes effect under
   `view: "as-at"` and is refused under any other view, exactly as on
   the query tools.
6. `codes` prints the account code beside each account name. It defaults
   to TRUE for `trial_balance` — a working paper is read against the
   chart — and to false for the other three; pass it either way to
   override. The aging has no code column at all, so `codes` changes
   nothing there.
7. `compare` is §5's parameter unchanged: `"prior-period"` or
   `"prior-year"`, on `profit_and_loss` and `balance_sheet` only. The
   document then grows three columns — the comparative figure, the
   variance, the percentage — and each amount column is headed by its
   own dates (a P&L's window, a balance sheet's as-of date) rather than
   by the word "Comparative".
8. `prepared_on` is the date printed on the document's prepared line,
   defaulting to today. Pass the original date to reproduce an earlier
   document: rendering reads no clock of its own, so the same ledger,
   the same parameters, the same template and the same kit render byte
   for byte. That is how a lost copy is replaced without it reading as a
   freshly prepared one.
9. `aged_receivables` renders §7's report with §7's rules intact: `at`
   defaults to today, `customer` narrows the customer and totals blocks
   ONLY, and the control row still covers every open invoice in the
   ledger regardless — so never present a filtered document's control
   row as that customer's balance. It needs `configure_invoicing`;
   without it the call is refused `invoicing.not-configured`.
10. In-chat tables (§3) and rendered documents serve different
    audiences, and neither is built from the other: the table is the
    conversation, the document is what the human sends to a bank or an
    accountant. Each call queries the books itself, so a document is
    never assembled out of a table you already printed.

## 9. Restyling: a template edit, never a per-render option

1. One template styles all four reports, and editing it changes every
   future `render_report` call — every report, every period, starting
   with the very next render. There is no per-render style override, the
   same owner ruling qunei-invoicing §4.1 states for invoices. A request
   for "just this one report, differently" has no tool behind it; say so
   rather than improvising one.
2. The loop is three calls and a look. `get_report_template` returns the
   ledger's current document — its own customization if it has one, else
   the built-in default, `source` saying which and `version` being null
   for the default. Edit the returned `html` directly. Pass the result
   to `update_report_template`, which reports the new version number.
   Then call `render_report` and look at what came out. Never report a
   restyle done on the strength of the write alone.
3. `update_report_template` takes the WHOLE document and validates it
   before writing anything. A failure writes nothing — the previous
   template, default or custom, is left exactly as it was — and it names
   every violation at once as `template.invalid`, so fix all of them and
   resubmit the complete document. There is no partial apply to build
   on.
4. What a valid template holds to:
   - The six required tokens, each present at least once and free to
     repeat: `{{BRAND_NAME}}`, `{{REPORT_TITLE}}`, `{{PERIOD_LINE}}`,
     `{{BASIS_LINE}}`, `{{REPORT_BODY}}`, `{{PREPARED_LINE}}`.
   - `{{BRAND_LOGO}}` and `{{FOOTER_LINE}}` are optional — use either
     any number of times, or leave it out. A ledger with no brand kit
     has neither, which is why they are not required.
   - Every token sits in ordinary text content: never in an attribute,
     a tag name, a `<style>` block, or an HTML comment.
   - The document is fully self-contained: no `<script>`, no `@import`,
     and no `http:`, `https:` or protocol-relative `//` reference
     anywhere in it. A logo or a font is a quoted `data:` URI. Elements,
     attributes and CSS functions are allowlisted, and a refusal names
     the one it caught.
   - No BACKSLASH anywhere in the CSS, in any position — not only in a
     `url()`. The rule is the character, not the construct, because a
     CSS escape can rebuild a scheme after any scanner has looked
     (`url("\68 ttp:...")`), so `content: "\201C"` is refused as
     squarely as that is. Write the character itself instead.
   - 256KB, checked first and alone: an oversized document is refused
     for its size before anything else in this list is looked at.

   qunei-invoicing §4.4 carries the long form of these rules; the gate
   is the same one, reading the report vocabulary above in place of the
   invoice one.
5. Keep a `<head>` whose first tag is `<meta charset="utf-8">`. Every
   render splices a server-written `<style id="brand">` block in
   immediately after that meta tag — or immediately after `<head>` when
   there is none — and an embedded font runs to hundreds of kilobytes of
   base64, enough to push a charset declaration past the window a
   browser reads to sniff a saved file's encoding.
6. The template owns the PAGE, never the STATEMENT. Masthead,
   typography, page furniture and print rules are yours;
   `{{REPORT_BODY}}` arrives as one complete
   `<table class="statement statement-…">` the server already built, so
   a template declares no rows, columns or section structure of its own.
   The body is restyled by writing CSS against the classes it carries.
7. That class vocabulary is the whole styling surface, and the only
   classes a body will ever carry — the server refuses any other, so a
   rule written for an invented name like `.total-row` matches nothing,
   silently, forever:
   - Table: `statement`, plus one of `statement-profit_and_loss`,
     `statement-balance_sheet`, `statement-trial_balance`,
     `statement-aged_receivables`.
   - Row groups: `section` (one per statement section) and
     `customer-group` (one per customer and currency on the aging).
   - Rows: `section-title`, `line`, `line-computed` (a derived line —
     the balance sheet's two equity lines, the trial balance's
     brought-forward line), `line-unmapped`, `section-total`,
     `subtotal`, `total` (on the statement's closing subtotal),
     `customer-total`, `grand-total`, and `control` carrying either
     `control-ok` or `control-off`.
   - One class per subtotal, so each can be weighted on its own:
     `subtotal-gross_profit`, `subtotal-total_income`,
     `subtotal-total_expenses`, `subtotal-income_tax`,
     `subtotal-net_profit_before_tax`, `subtotal-net_profit`,
     `subtotal-total_assets`, `subtotal-total_liabilities`,
     `subtotal-net_assets`, `subtotal-total_equity`.
   - Cells: `account`, `code`, `type`, `amount`, `neg` (added to any
     figure printed in parentheses), `comparative`, `variance`,
     `variance-pct`; the trial balance's `opening`, `debits`, `credits`,
     `closing`; and the aging's `bucket` with `bucket-current`,
     `bucket-1-30`, `bucket-31-60`, `bucket-61-90`, `bucket-over-90`.
   - `control-off` and `neg` are the two worth making impossible to read
     past. A failed reconciliation and a negative figure are findings,
     not decoration.
8. Colour and type belong to the brand kit (§10), not to the template.
   The injected block defines `--brand-ink`, `--brand-paper`,
   `--brand-primary`, `--brand-accent`, `--brand-muted`, `--brand-rule`,
   `--brand-font-heading`, `--brand-font-body` and
   `--brand-font-numeric` on every render, whether or not this ledger
   has a kit — an unset field falls back to a built-in value. Write the
   chrome against those variables rather than hardcoding a hex code, and
   a later `update_brand_kit` rebrands every report with no template
   edit at all. The block lands ahead of the template's own CSS, so a
   rule the template declares later still wins where it needs to.
9. Resubmitting the default document does not turn customization off: it
   pins a frozen copy of the default as the ledger's own template,
   `source` reading `"custom"` from then on, because documents have no
   delete and there is no true un-customize.
10. The built-in default is the designed chrome: an A4 statement drawn
    in the brand variables — a serif masthead (the logo when the kit
    has one, else the brand name), the period and basis lines, tabular
    figures in the numeric face, an accountant's rules above totals and
    a double rule under the closing figure, landscape pages for the
    working papers and compared statements. Don't build an edit on
    markup you remember: `get_report_template` is what the ledger
    actually has right now.

## 10. The brand kit: one brand, read before it is written

1. A ledger has ONE brand kit, and every report `render_report` produces
   is styled by it: a name, six colours, three font stacks, up to three
   embedded font files, a logo and a footer line. `get_brand_kit` shows
   it — `source` (`"default"` until the ledger writes one, `"custom"`
   after), `version`, and a `brand` block with every field. `source:
   "default"` with null fields means the built-in values apply, never
   that something is missing or broken.
2. The three font files and the logo come back summarised as their media
   type and decoded byte size, never as base64. So you can see that a
   logo is set and how big it is, and you cannot read one back out —
   which is also why §10.8's merge semantics matter.
3. Start by reading whatever the human actually has: logo files, a brand
   guidelines PDF or docx, a stylesheet or CSS variables, a website, a
   deck. Pull typed values out of it — six hex colours, the heading and
   body faces, the name as it should print, a footer line — and note
   which of them the material genuinely settles and which you inferred.
4. CONFIRM the palette and the type with the human BEFORE writing
   anything. Show the mapping field by field: which colour is ink, which
   is paper, which is primary, which is the rule; which stack is heading
   and which is body. Then wait. This rebrands every future document, so
   a guess here is not a small one.
5. The fields, with the grammar each is held to:
   - `name` — printed at the top of every report, at most 120
     characters. Unset, it falls back to the invoicing seller name when
     one is configured, else the ledger slug.
   - `colour_ink`, `colour_paper`, `colour_primary`, `colour_accent`,
     `colour_muted`, `colour_rule` — six hex digits with a leading `#`
     (`#a04e26`). No named colours, no `rgb()`, no `var()`.
   - `font_heading`, `font_body`, `font_numeric` — CSS font stacks of
     letters, digits, spaces, commas, quotes and hyphens, at most 200
     characters each (`Georgia, "Times New Roman", serif`).
   - `font_file_heading`, `font_file_body`, `font_file_numeric` —
     `data:font/woff2;base64,…` URIs, at most 200KB decoded each.
   - `logo` — a `data:image/svg+xml;base64,…` or
     `data:image/png;base64,…` URI, at most 128KB decoded.
   - `footer` — one line at the foot of every report, at most 200
     characters. It is text, not markup: a `{{TOKEN}}` written here
     prints literally and is never substituted.
6. The logo is converted by you, from the file the human supplied:
   base64 it and prefix the media type. SVG is preferred — it prints
   crisply at any size — and PNG is accepted. An SVG is refused whole if
   it carries `<script`, an `on…=` event handler, `javascript:`,
   `<foreignObject`, `<use`, a protocol-relative `//` reference, any
   `http:`/`https:` reference, or ANY numeric character reference: write
   the character itself rather than `&#…;` (named entities such as
   `&amp;` stay fine). A PNG has to be a real PNG — the file signature
   is checked. A logo over the cap is re-exported smaller, at the same
   aspect and legibility, and the refusal names the byte size it came to
   rather than echoing the URI. Prefer the transparent-background version
   of a logo: the defaults print on the printer's paper, not on the kit's
   paper colour, so a logo that carries its own paper-coloured background
   prints as a box on white.
   A FILE EXPORTED FROM A DRAWING PROGRAM (Inkscape, Illustrator) is
   accepted as it comes, as long as its only `http:` references are
   namespace declarations — `xmlns="…"`, `xmlns:inkscape="…"`,
   `xmlns:sodipodi="…"` — and a leading `<!DOCTYPE svg …>` naming a DTD
   URL, with no `[…]` entity subset in it. Those are identifiers nothing
   ever fetches, and they are blanked before the scan; a doctype that
   declares entities is a payload rather than an identifier, so it is
   scanned like anything else. Every other position still counts, so the export's
   own metadata is what usually needs a hand: strip the `<metadata>`
   block, any `sodipodi:namedview` element, and the comments — an
   editor's "created with" comment carries its home page as a plain
   `http://` URL and is refused for it. A quick clean-and-retry beats
   guessing which line the refusal meant.
7. Embedded fonts come only from files the human supplies. Never fetch a
   face, never embed one the human has not licensed for this, and never
   convert something you were not given. woff2 only. An embedded face is
   prepended to its slot's stack and the stack stays the fallback, so
   set `font_file_heading` and `font_heading` together rather than
   relying on either alone.
8. `update_brand_kit` MERGES. A field you pass is set, a field named in
   `clear` is removed and falls back to the built-in value, and a field
   you mention neither way is left exactly as it was — so changing one
   colour never disturbs the logo, which is just as well, since you
   could not resend it. When the same field is both given a value and
   named in `clear` in one call, `clear` wins and the field is unset.
   `clear` refuses a name outside the field list above.
9. A refused kit writes NOTHING: the whole submission is validated
   first, every violation comes back at once as `brand.invalid`, and the
   stored kit is left exactly as it was. The response summarises the kit
   as re-read from what was actually written, so read it back rather
   than assuming the merge you intended is the merge that landed.
10. Where the human's material settles nothing for a field, leave that
    field unset and SAY SO: the built-in neutral value applies to it,
    and the reports still render. Never invent a hex code to fill a
    slot, and never substitute a lookalike stack for a licensed
    typeface without telling the human that is what you did.
11. Then render one sample of EACH of the four reports with
    `render_report` and look at all four. A kit that reads well on a
    P&L can fail on the trial balance's four numeric columns or the
    aging's bucket grid. Iterate one `update_brand_kit` call at a time —
    contrast on the rule and muted colours, a logo that swamps the
    masthead, figures in a proportional face — and re-render. `brand`
    and `brand_version` on each render say which kit produced it.
12. The invoice document has its own template and its own restyle loop
    (qunei-invoicing §4); the kit here is the reports' brand.

## 11. Files on request

1. **Document formats** (docx/pptx/xlsx): if the client offers its own
   document-authoring skills, compose the report through those and say
   which one you used. This skill has no document-writing tool of its
   own. For a statement the human will send on, prefer §8's rendered
   HTML — it is already the finished document, and its PDF is one print
   dialogue away.
2. **CSV**: write the file directly — a CSV is just the query result
   restated as comma-separated rows, no client skill required.
3. **HTML dashboard**: fill `dashboard-template.html`'s placeholders
   (`{{ENTITY}}`, `{{PERIOD}}`, `{{PL_ROWS}}`, `{{CASH_POSITION}}`,
   `{{TOP_MOVERS}}`, `{{PENDING_DRAFTS}}`, `{{GENERATED_AT}}`) from this
   session's query results and write the filled copy next to the books.
   `{{PL_ROWS}}` and `{{TOP_MOVERS}}` are row-fragment insertion points —
   see the HTML comment beside each token in the template for the exact
   `<tr>` shape expected. The template is self-contained (inline CSS, no
   JS, no external requests) — keep it that way; don't add a script tag
   or a remote font/asset link when filling it in. This is the ephemeral
   management view: a scan for whoever runs the business, regenerated
   whenever it is asked for, and not the document a bank or an
   accountant receives. That one is §8's.
4. When the ledger has a brand kit, take the dashboard's palette from
   `get_brand_kit` rather than leaving it on the shipped colours. In the
   COPY you write (never in the template itself), set the `:root`
   custom properties from the kit: `--bg` from `colour_paper`, `--fg`
   from `colour_ink`, `--heading` from `colour_primary`, `--muted` from
   `colour_muted`, `--accent` from `colour_accent`, `--border` from
   `colour_rule`, and `--font-serif`/`--font-sans` from
   `font_heading`/`font_body`. `--card-bg` has no brand field to draw
   from and stays on the dashboard's shipped value. Leave a property
   alone where the kit leaves its field unset, and if you recolour the
   light palette either recolour the `prefers-color-scheme: dark` block
   to match or drop it — never leave the two disagreeing. The dashboard
   has no logo slot; the brand mark belongs on §8's rendered report.
5. **Artifact publishing**: when the client supports it, publish the
   filled dashboard (or a `render_report` document) as an Artifact
   instead of, or alongside, writing it to disk.
6. Every number still comes from §1's query tools whatever the format —
   with §8 as the one exception worth naming: a `render_report` document
   is built server-side from its own query, so its figures are the
   query's own and there is nothing in it for you to fill in, restate or
   check by hand.

See qunei-month-review for the periodic scan this skill often renders
the output of, and qunei-bookkeeping for the session rules (briefing
first, query tools for money questions) this skill inherits.
