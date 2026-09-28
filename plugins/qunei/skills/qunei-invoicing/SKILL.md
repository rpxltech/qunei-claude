---
name: qunei-invoicing
description: Use when issuing, tracking, crediting, or chasing customer invoices in a Qunei entity — drafting and issuing invoices, recording payments, credit notes, rendering the invoice document, and restyling its template.
---

# Qunei Invoicing

The customer-invoice lifecycle end to end: configure once, draft and issue
with the human's explicit go-ahead, then track payments, credit notes, and
overdue chasing against what `get_invoice` and `list_invoices` actually
return. Substance is frozen the moment an invoice is issued — corrections
happen through a credit note or a void, never a hand edit — but
presentation stays free: re-render the same invoice from its own records
as many times, in as many formats, as anyone asks.

## 1. Configure before anything else

1. Call `get_briefing` first, per qunei-bookkeeping's session rule, and
   check whether its `ar` key is present. Its absence means this entity
   has never run `configure_invoicing` — every other invoicing tool
   refuses with `invoicing.not-configured` (hint: run it) until that
   happens.
2. Configuring is a short conversation, not a silent setup call: ask the
   human which account should receive this entity's receivables. Every
   country template (`nz-company`, `au-company`, `uk-company`,
   `us-company`) already ships `Assets:Accounts-Receivable` for this; the
   `global` template does not — `configure_invoicing` refuses an
   unrecognized `ar_account` with `account.unknown`, and that error's own
   hint says exactly this. A global-template entity therefore needs
   `add_account` for a new asset account first, confirmed with the human
   exactly like any other new account, per qunei-bookkeeping §2.
3. Once there's an `ar_account`, call `configure_invoicing` with it — and,
   in the same call, whatever else the human wants to set now: numbering
   prefixes (`prefix`/`credit_prefix`, defaulting to `INV-`/`CN-`),
   default `terms`, the seller block that prints on every issued
   invoice (`seller_name`, `seller_tax_number`, `seller_address`,
   `payment_instructions`), and how its dates print (`date_format`:
   `dd/mm/yyyy` for day first or `mm/dd/yyyy` for month first). Ask the
   human which date style their customers expect; until it is set,
   invoices follow the ledger's locale, month first for `en-US` and day
   first everywhere else.
4. It's safe to call `configure_invoicing` again later to change one
   field — every omitted field keeps its current value, so fixing the
   payment instructions next month doesn't mean re-stating the AR
   account.

## 2. Draft, confirm, then issue — never unprompted

1. A new customer needs declaring before it can be invoiced: `add_customer`
   (slug, name, and whatever else the human has on hand — email, address,
   tax number, terms); `list_customers` shows who's already on file.
   `draft_invoice` refuses an undeclared customer with `customer.unknown`,
   whose hint names exactly those two tools.
2. `draft_invoice` builds the draft and returns a computed totals preview
   (`subtotal`, `tax_by_code`, `total`) alongside its substance — the
   exact figures an eventual issue would post. The draft's own `number`
   is null; nothing has been assigned yet.
3. Show the human a compact totals table before anything else happens:
   line items, subtotal, tax, total — and say plainly that the number
   isn't assigned yet, never a placeholder that could be mistaken for the
   real one.
4. `render_invoice` also works on a still-unissued draft, referenced by
   its ULID (a draft has no number yet for `ref` to resolve by) — it
   shows the actual client-facing document, masthead-numbered "Draft",
   before anything is issued. §3 covers rendering in full; offer this
   alongside the totals table above when the human wants to see the real
   layout, not just the figures, before confirming.
5. Wait for explicit confirmation. A wanted change goes through
   `update_invoice`, then the totals table again — don't issue against a
   stale preview. An abandoned draft goes through `discard_invoice`;
   never leave it sitting unmentioned, the same disposition discipline
   qunei-code-statement applies to a held-back draft.
6. Only once confirmed: `issue_invoice`. This is what assigns the legal
   number — the next integer in its prefix's series — and posts the
   entry to the entity's ledger.
7. `issue_invoice` is always allowed regardless of the entity's posting
   policy: there's no separate "approve this issue" step the way
   `post_entries` has one, because issuing an invoice IS the approval act
   (the same exemption `approve_drafts` carries). That describes what the
   server will let you do, not what you should do without asking — step
   5's go-ahead still comes first, every time.

## 3. Rendering: one `render_invoice` call, delivered as written

1. Once issued (or credited — `ref` also accepts a credit note's number,
   e.g. `CN-7`), call `render_invoice` once and use exactly what comes
   back. Its `html` is the finished, ready-to-send customer document —
   deliver it as written, or print it to PDF client-side; the pixels are
   pinned by the HTML, so there's no layout left to do. Never hand-edit
   the returned markup, never hand-author invoice markup of your own,
   and never recompute or restate a figure — every number on the page
   came from the same `Totals` math that posted the entry, not a fresh
   qty × unit worked out by hand.
2. `source` and `template_version` on that same payload are how an agent
   tells which document a customer actually got: `source` reads
   `"default"` for an entity that has never customized its template and
   `"custom"` once it has (§4), and `template_version` is that custom
   template's own version number — `nil` whenever `source` is
   `"default"`, since the built-in default has no version of its own.
   Read them off this call rather than assuming.
3. `number` on this payload is the masthead text as printed — "Draft" on
   an unissued draft, the real number otherwise — which is not always
   the same string `get_invoice` reports for that same file: a RESERVED
   draft (a number already assigned, the issuing entry not yet posted)
   still prints "Draft" here, deliberately, so a customer is never
   handed a document carrying a number issuing could still fail to
   finalize. `get_invoice` has the stored number; this call has the
   printed one — never assume the two match.
4. A credit note's heading, and its own `Credits INV-1042`-style line
   naming the invoice it credits, are already correct in the returned
   HTML — nothing to detect from `invoice.credits` and relabel by hand
   the way this section used to ask for.
5. `render_invoice` produces HTML only. For a `.docx`/`.pptx` instead,
   compose with the client's own document-authoring skill, the
   qunei-reports pattern, pulling figures from a plain `get_invoice` call
   rather than recomputing them. Substance is frozen at issue, but
   rendering is not: call `render_invoice` again any time presentation
   needs to change — a lost copy, a different format, a freshly
   restyled template.
6. A tax row is labelled with the tax's name and rate as a customer reads
   it — `GST 15%` — never with the ledger's own tax-code slug. The name
   is the slug's first segment, upcased, and the rate is the code's
   declared fraction as a percentage, so a code named `gst-standard`
   prints "GST 15%" and `vat-reduced` at 0.125 prints "VAT 12.5%". When
   you declare a tax code, put the tax's name before the first hyphen.

## 4. Restyling: a template edit, never a per-render option

1. Restyling changes what every future `render_invoice` call produces —
   for every invoice, every customer, starting with the very next
   render. There is no per-invoice or per-render style override, by
   owner ruling: byte-consistent invoices are what makes a document
   recognizable as genuinely from this business rather than a convincing
   forgery, and that consistency is the reason this module exists at
   all. A request for "just this one invoice in a different style" has
   no tool behind it — say so rather than improvising one.
2. The loop is always the same three calls: `get_invoice_template` reads
   the entity's current document — its own customization if it has one,
   else the built-in default, `source` saying which — then edit the
   returned `html` directly, then `update_invoice_template` with the
   result, which reports the new version number: the same figure a
   subsequent `render_invoice` call reports back as `template_version`.
   Confirm the change actually took with a real `render_invoice` call
   afterwards and look at the result — never report a restyle done on
   the strength of the write alone.
3. `update_invoice_template` validates the WHOLE document before writing
   anything. Any rule broken anywhere refuses the entire submission and
   changes nothing — the previous template, default or custom, is left
   exactly as it was. The error is `template.invalid`, and it names
   every violation at once rather than one at a time, so fix all of them
   and resubmit the complete document; there's no partial-apply to build
   on. One exception: an oversized document is refused for its size
   alone, before anything else is even checked — see §4.4.
4. The rules an edit has to hold to, concretely:
   - Every `{{TOKEN}}` must sit in ordinary text content — never in an
     attribute, a tag name, a `<style>` block, or an HTML comment.
     `<td>{{DESCRIPTION}}</td>` is right; `<td class="{{DESCRIPTION}}">`,
     `style="width:{{TOTAL}}"`, a token written inside `<style>...</style>`,
     a token used as a tag name, or a token left in `<!-- {{TOTAL}} -->`
     are all refused. Reason worth remembering while editing: the server
     escapes each value for a text position when it fills the template,
     so a token anywhere else could let a customer's own name or note
     become live markup on the document.
   - The document must be fully self-contained: no `<script>`, no
     `@import`, no CSS escape sequence, and no `http:` or `https:`
     reference anywhere in it — the bare scheme is enough to refuse it,
     no `//` required, so even `https:qunei.ai` in a footer note is
     caught. A logo or custom font is a quoted `data:` URI —
     `url("data:image/png;base64,...")` in CSS, or
     `src="data:image/svg+xml;base64,..."` on an `<img>` — never a
     linked file. An SVG logo works this same way, as a
     `data:image/svg+xml` URI; a bare `<svg>` element does not, refused
     on sight exactly like `<script>`. CSS is function-allowlisted too,
     so a visual-effect function like `filter:`/`blur()`/`drop-shadow()`/
     `backdrop-filter`, or a 3D transform like
     `scale3d()`/`rotate3d()`/`translateZ()`, is refused by name, while
     `rgba()`, `linear-gradient()`, `box-shadow`, 2D
     `transform: rotate()/scale()/translate()`, `calc()`, and `var()`
     all work. The refusal names the exact function, so when a restyle
     hits one, swap in a supported effect and move on.
   - The whole document is capped at 256KB, and that cap is checked
     FIRST, alone, before anything else in this list — an oversized
     document is refused for its size and nothing else, so a missing
     token or an external reference sitting in that same document stays
     unreported until it's made small enough to be checked at all. A
     base64-embedded logo is the likely way to hit this cap by accident,
     so keep an eye on size while embedding one.
   - The vocabulary is fixed: 13 top-level tokens that must each appear
     somewhere in the document, one optional token that may appear any
     number of times including zero (`{{BRAND_LOGO}}`, §4.6), plus two
     repeating blocks —
     `{{#LINE_ROWS}}…{{/LINE_ROWS}}` and `{{#TAX_ROWS}}…{{/TAX_ROWS}}` —
     each of which is required exactly once, carrying its own per-item
     tokens inside. `get_invoice_template` shows the complete, current
     set in place — read it there rather than recalling this list from
     memory or an old render. The row markup inside each block is
     otherwise ordinary, editable HTML — add a column, restyle a cell,
     reorder them — only the repetition itself is mechanical.
5. Submitting the default template's own HTML back through
   `update_invoice_template` does not turn customization off. It pins a
   frozen copy of the default as the entity's own template from that
   point on — `source` reads `"custom"` afterwards, same as any other
   edit — because documents have no delete. There is no true
   un-customize.
6. The ledger also carries ONE brand kit: a name, six colours, three
   font stacks, up to three embedded font files, a logo and a footer
   line, set through qunei-reports §10 and readable at any time with
   `get_brand_kit`. It styles the reports `render_report` produces AND
   every invoice rendered here. The kit's `<style id="brand">` block is
   injected into the head of every `render_invoice` document, always,
   whether or not this ledger ever set a kit: unset fields fall back to
   built-in values, so the variables below always resolve.
   `render_invoice` reports which kit produced the bytes as `brand` and
   `brand_version`, exactly the way it reports the template as `source`
   and `template_version`.
   The shipped default template reads those variables, so a kit
   restyles an unedited invoice exactly as it restyles the statements —
   the same faces, colours and rules as the default report chrome — and
   prints the kit's logo in the masthead when the kit has one, above
   the seller's name, which a tax invoice always prints. What a
   CUSTOMISED template can do:
   - Use the kit's values as CSS variables anywhere in its own `<style>`
     block: `var(--brand-ink)`, `--brand-paper`, `--brand-primary`,
     `--brand-accent`, `--brand-muted`, `--brand-rule`,
     `--brand-font-heading`, `--brand-font-body`, `--brand-font-numeric`.
     The block is injected ahead of the template's own rules, so a rule
     the template declares later still wins.
   - Carry `{{BRAND_LOGO}}` in a text position, normally once, in the
     masthead. It is the ONE optional token in the vocabulary: legal any
     number of times including zero, so every template written before
     the kit existed still validates untouched, and it prints the kit's
     logo, or nothing at all when no kit sets one.
   A logo or typeface embedded directly in the template as a `data:` URI
   per §4.4 still works and is independent of the kit. Read the kit
   before restyling either way: the invoice and the statements should
   read as the same business, and the kit is where that business already
   wrote its colours down.

## 5. Payments: match first, then record

1. Before recording anything, find the right invoice: `list_invoices`
   filtered by `customer` and status (`open`/`part-paid`), or
   `overdue_only` when that's the question — never record against a
   number recalled from memory or one that only loosely matches the
   deposit in front of you.
2. `record_invoice_payment` posts the settlement against the matched
   invoice; omit `amount` to pay the full open balance. It refuses
   `invoice.overpayment` if the amount is more than what's actually
   owing, and `invoice.not-issued` if the invoice is still a draft.
3. Like `issue_invoice`, this is always allowed regardless of posting
   policy — it's a deliberate settlement act, not a batch of postings
   waiting on a review step.
4. This tool is convenience, not the only path: a receipt coded by hand
   through qunei-code-statement, tagged `invoice:<number>` on the AR
   posting, settles the exact same way. Status and balance are derived
   from that tag alone, never from which tool happened to post it.

## 6. Credit note or void — a judgment call

1. `void_invoice` reverses the issue entry outright. It only works when
   nothing has been paid or credited against the invoice yet —
   `invoice.void-blocked` otherwise, hinting at a credit note instead.
2. The decision rule: **sent → credit note; never sent → void.** A
   customer who has already seen the invoice, or logged it in their own
   accounts, is left holding a document that no longer matches yours if
   it's simply voided out from under them — a credit note leaves a paper
   trail they can reconcile against what they actually received, even in
   the rare case where the tool would still let you void it because
   nothing's been paid yet. Reach for void only when you're confident it
   never left this session.
3. `create_credit_note` defaults to crediting the entire original
   invoice's own lines — only legal the first time; once any credit note
   has ever been issued against that original, pass explicit `lines`
   instead, or it refuses `invoice.over-credit`.
4. It only drafts, exactly like `draft_invoice` — the same
   totals-preview-then-confirm dance from §2 applies before you
   `issue_invoice` the credit note itself. A credit note isn't real
   until it's issued, same as an invoice.

## 7. Chasing overdue invoices

1. Start with `aged_receivables` — always, before any chase text is
   written. One call answers "who owes what, and how late": every open
   invoice bucketed by days past its `due` date (`current`, `1-30`,
   `31-60`, `61-90`, `over-90`), grouped per customer and currency, each
   group carrying its own bucket totals. `at` defaults to today; pass it
   only when chasing as at some other date.
2. **The `over-90` column is the chase list.** Work it first, oldest due
   date down, and treat what sits there as a conversation with the human
   before a letter to the customer — an invoice three months past due is
   usually a dispute, a bad debt, or a posting error, not someone who
   forgot. `61-90` is the next call; `current` is nobody's chase. One
   thing sits in `current` that is not merely unpaid: an OVERPAID
   invoice (`credit: true`, a negative `balance`) is always bucketed
   `current` whatever its due date, while its `days_past_due` still
   states the true count. That is deliberate — a credit left in a
   due-date bucket cancels real overdue debt inside that bucket, and a
   chase list would come up empty while the money is genuinely late.
   Never chase one: it is money owed back. Raise it with the human
   instead, since it is usually a duplicate payment or one matched to
   the wrong invoice.
3. `list_invoices` with `overdue_only: true` — scoped to one customer
   when that's what was asked for — is where the per-invoice detail
   comes from once the aging has said who to chase. Never chase from a
   memory of last month's numbers.
4. For each one, draft chase text from its own `number`, `balance`, and
   `due` date: polite, factual, dated.
5. Hand the draft to the human for their own email or channel. Never
   send it yourself — this skill has no tool that delivers anything to a
   customer, on purpose; the human decides what leaves the building, and
   when.
6. A `customer` filter narrows the `customers` and `totals` blocks and
   nothing else: the `control` row keeps reconciling the whole ledger's
   receivables account, so under a filter its `aging_total` is larger
   than the groups you can see. That is deliberate — never quote the
   control row's figures as one customer's balance.
7. **`reconciles: false` on the control row is a bookkeeping finding,
   not a formatting problem.** It means the receivables account holds a
   balance the invoices do not account for: money booked straight to
   receivables without an invoice, a credit note tagged to the wrong
   number, an invoice posted twice. Report it in plain words, with the
   `difference` figure and the account named on the row, and hand it to
   qunei-bookkeeping or to the human. Never paper over it by re-adding
   the buckets yourself, and never chase a customer on the strength of
   an aging that doesn't reconcile.

See qunei-bookkeeping for the session manners this skill inherits
(briefing first, never guess an account, read a structured error before
retrying); qunei-code-statement for the receipt side of a tagged invoice
payment, coded from a bank statement rather than through §5 above; and
qunei-reports for the document-authoring pattern §3 uses for any format
`render_invoice` itself doesn't produce.
