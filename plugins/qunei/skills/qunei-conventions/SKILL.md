---
name: qunei-conventions
description: Use when a coding decision reveals a stable, reusable rule for a Qunei entity — a payee-to-account mapping, a recurring split, or another standing preference — either to consult one before coding a transaction or to record one just discovered.
---

# Qunei Conventions

Memory discipline for an entity's `conventions.md`: the append-only
record of standing decisions a bookkeeper has already made, so the same
judgment call isn't re-litigated every session. The server enforces
append-only (there is no edit or delete tool for this document); this
skill explains how to use that memory well.

## 1. Consult conventions before coding

1. Before deciding how to categorize a transaction, check whether the
   entity already has a rule for it. `get_briefing` includes a capped
   view of `conventions.md`; call `read_conventions` directly when you
   need the full, untruncated text (the briefing marks its copy
   `truncated: true` when it was cut).
2. If a matching rule exists, follow it. Do not re-derive a categorization
   from first principles when the entity has already settled it.

## 2. Recording a new rule

1. When coding a transaction reveals a stable rule worth reusing —
   "payee X always posts to account Y", "this kind of split always
   divides N ways" — append it with `append_convention`.
2. `append_convention` takes one `line` and prefixes it with today's date
   automatically; do not add your own date. One call, one rule.
3. If a single session surfaces several distinct rules, call
   `append_convention` once per rule rather than folding them into one
   line — each convention should be independently readable and
   greppable later.
4. Write the line so it stands alone: name the payee or pattern and the
   exact account, in a form a future session can match against without
   re-reading this conversation. Prefer "Payee 'Acme Freight' → account
   Expenses:Freight" over a vague summary of what you decided.

## 3. Conflicts go to the human

1. If a current transaction's evidence contradicts a rule already in
   `conventions.md`, that is a conflict, not an update you resolve on
   your own. Surface it to the human and ask which one governs.
2. Never silently override a standing convention because the current
   case looks like an exception. Exceptions are exactly what get lost
   without a human decision on record.

## 4. Never rewrite history

1. There is no tool to edit or delete a convention line, and that is
   intentional — `conventions.md` is a record of what was decided and
   when, not just what's currently true.
2. A correction to an old rule is a NEW line that supersedes the old one
   (e.g. "Payee 'Acme Freight' now → account Expenses:Logistics,
   supersedes 2026-01-10 entry"), appended via `append_convention`, not
   an edit to the original.
3. Flag superseded lines to the human — curating `conventions.md` (e.g.
   deciding when a document has accumulated enough dead rules to warrant
   a rewrite) is a human call, not something this skill automates.

See qunei-bookkeeping for when account resolution should consult this
document, and qunei-onboard for seeding the first conventions on a new
entity.
