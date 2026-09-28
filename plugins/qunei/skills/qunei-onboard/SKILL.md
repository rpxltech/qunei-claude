---
name: qunei-onboard
description: Use when a Qunei workspace has no ledger yet, the human asks to set up their books or start fresh, or when inviting users to the workspace, archiving a ledger, or explaining a billing state or billing refusal.
---

# Qunei Onboard

Guided setup for a workspace's first ledger, plus the workspace-level
manners that come with it: naming ledgers explicitly, inviting users,
archiving, and billing states. Setup is a conversation before it is a
sequence of tool calls: interview the human fully, confirm the plan,
then execute it exactly as confirmed.

## 1. Where you are: the hosted arrival

1. Qunei is hosted and self-serve. The human signed up at qunei.ai,
   verified their email, and brought you in one of three ways: the
   Qunei plugin (Claude on a paid plan, or Claude Code), which
   carries these skills and the connection; Qunei added as a custom
   connector on Claude's free plan, with its skills uploaded by hand;
   or, for tools that cannot sign in, an MCP config carrying a personal
   access token. By the time this skill runs, the connection already
   exists, and there is no web setup wizard to send them to. You are
   the setup — the product calls you the workspace's scribe: you record
   it, the human reviews it. With the plugin, its `start` skill runs the
   first conversation (the greeting, the questions, the first
   statement) and asks only its own questions, taking from §2 just the
   setup for their country and the settings to confirm; §2's full
   interview is for setups started any other way. This skill's rules
   still bind it: confirm before creating, never invent, and §6's rule
   that settings are fixed at creation.
2. Vocabulary for everything you say: a **workspace** is the account
   (logins and billing); a **ledger** is one set of books inside it —
   what Xero calls an organisation. The tools' `entity` parameter
   takes a ledger's slug.
3. In a fresh workspace there is no ledger yet, so there is nothing to
   brief on — run the interview below, and call `get_briefing` right
   after `init_entity` (qunei-bookkeeping's own session-start
   exception). In a workspace that already has ledgers, brief first;
   the briefing's `billing` section (§10) tells you where the
   workspace stands.

## 2. Interview first — no tools yet

Ask through all of these before calling anything:

1. **Entity kind**: company, sole trader, or other.
2. **Country, currency and year end**: the country decides the
   template, and a country template brings that country's currency,
   financial year end, date style, starter chart of accounts and tax
   codes:
   - New Zealand: `nz-company` (NZD, 31 March, `en-NZ`; GST codes
     `gst-standard` at 15%, `gst-zero`, `gst-exempt`).
   - Australia: `au-company` (AUD, 30 June, `en-AU`; GST codes `gst` at
     10%, `gst-free`, `input-taxed`, `bas-excluded`).
   - United Kingdom: `uk-company` (GBP, 31 March, `en-GB`; VAT codes
     `vat-standard` at 20%, `vat-reduced` at 5%, `vat-zero`,
     `vat-exempt`; rent carries no default code, since commercial rent
     is exempt unless the landlord has opted to tax).
   - United States: `us-company` (USD, 31 December, `en-US`; no tax
     codes, since sales tax varies by state: it posts straight to
     `Liabilities:Sales-Tax-Payable`).
   - Anywhere else: `global`, a minimal chart with no tax codes and one
     `base` book, which needs the ledger's own currency, year end and
     locale (§4.1). For the locale, pick the date order the business
     writes: `en-US` for month first, otherwise the nearest of `en-NZ`,
     `en-AU` or `en-GB`.
   Confirm the year end and currency with the human even on a country
   template: a business whose year ends on another day, or that keeps
   its books in another currency, passes its own. Every country template
   builds a single `tax` book on the `base` layer.
3. **Bank accounts**: every bank account to add, by name/purpose (e.g.
   "ANZ everyday checking", "savings"). Note that every template already
   seeds one generic `Assets:Bank` (code 100) — ask whether that covers
   the first account or whether it should be renamed in spirit (a new,
   more specific account added instead; `add_account` cannot rename the
   seeded one).
4. **Usual suppliers/payees**: 5–10 names the entity deals with
   regularly, and which account category each maps to (e.g. "AWS →
   Expenses:General", "main landlord → Expenses:Rent").
5. **Invoicing** (optional): does this entity invoice customers? A yes
   here means `configure_invoicing` once the setup below is confirmed
   and done — a global-template entity needs an AR account added via
   `add_account` first (every country template already ships one), and
   the human chooses whether invoice dates read day first or month
   first. That's a separate conversation with its own manners; see
   qunei-invoicing for it rather than folding it into the plan this
   skill confirms next.
6. **GST or VAT registration**: is this entity registered?
   - New Zealand: registered for GST with Inland Revenue? A yes here
     means `configure_nz_gst` once the setup below is confirmed and done
     — the `nz-company` template already ships the three canonical tax
     codes, `global` gets them seeded by that same call. Its own
     basis/frequency/cycle-end conversation belongs to qunei-nz-gst, not
     here.
   - Australia (GST, the ATO) and the United Kingdom (VAT, HMRC): the
     template ships the codes, but there is no return pack for either
     yet, so the answer becomes a convention (§4.4), which every
     briefing carries and qunei-bookkeeping §8 reads. Record it either
     way, for example `Registered for GST: record taxable amounts net,
     with GST split to Liabilities:GST and the net line tagged tax:gst`,
     or `Not registered for VAT: record amounts gross, with no tax
     tags`.

## 3. Confirm the plan before executing anything

1. Restate back to the human: entity kind, chosen template, currency,
   financial year end, the slug you'll use, every account you're about
   to add, and every payee rule you're about to seed.
2. Get an explicit go-ahead. Do not call `init_entity` or any tool that
   follows until the human has confirmed the plan as stated.

## 4. Execute in order

1. `init_entity` with the confirmed slug and template. On a country
   template, add `currency` or `fy_end` only where the human's differ
   from the template's. On `global`, pass `currency`, `fy_end` and
   `locale`: all three are required, and the call refuses with
   `settings.required` without them.
2. `add_account` once per named bank account beyond the template's
   seeded `Assets:Bank` — use a distinct code and a `Namespace:Name` form
   (e.g. `Assets:Bank-Savings`).
3. `update_account` for any mapping tweaks the human explicitly asked
   for in the interview (report mapping, tax default, archived flag) —
   never for account name or code, which this tool can't change anyway.
4. `append_convention` once per payee rule learned in the interview
   (one call per rule, per qunei-conventions).
5. Finish with `get_briefing` and a plain-language summary of what now
   exists: the ledger, its accounts, and the conventions you seeded.

## 5. Never invent beyond the interview

1. Do not add an account, tax code, or convention the human didn't name.
   No speculative "just in case" accounts.
2. If a need surfaces mid-setup that wasn't covered in the interview
   (an unexpected account, an ambiguous payee category), stop and ask —
   don't fold a guess into the plan you already confirmed.

## 6. Settings are fixed at creation: get them right first

1. `init_entity` takes the ledger's `currency` (an ISO code the
   registry holds), `fy_end` (MM-DD, a day that exists every year, so
   a year ending in February is `02-28`) and `locale` (`en-NZ`,
   `en-AU`, `en-GB` or `en-US`, which decides whether its reports'
   dates read day first or month first). It checks all three before
   creating anything and refuses with a code naming the fix. No tool
   changes them afterwards, and a currency is fixed for good once
   entries exist, so settle them with the human in §2 before §4.
2. After `init_entity`, check what actually landed: its response
   echoes the ledger's `settings`, and `get_briefing` reports
   `entity.currency` and `entity.fy_end`. If they don't match what the
   human confirmed, tell them plainly and log it with
   `append_open_item` — do not silently proceed as if it matches, and
   do not invent a settings-editing call that doesn't exist.
3. If you're running in local mode (§11) with filesystem access to this
   ledger's `documents/` folder, you may also tell the human they can
   hand-edit `settings.txt` themselves — it's plain `key value` lines,
   the same format the template shipped (e.g. `currency AUD`). You never
   write this file yourself; only the human edits it, and only after you
   tell them how. Once they say they've made the change, re-run
   `get_briefing` to confirm it landed. A hosted workspace has no
   filesystem for anyone to hand-edit — there, the `append_open_item`
   in §6.2 is the whole fallback.

## 7. Name the ledger in every write

1. In a workspace where more than one ledger is visible to you, every
   write call must name the ledger — pass `entity:` explicitly; the
   server refuses the default-ledger fallback with
   `entity.explicit-required`, by design. That refusal is the rule
   working, not an error to route around.
2. When the conversation has not made the target ledger obvious, ask
   the human which ledger before naming one — never guess a ledger
   into a write because it was the last one touched or a configured
   default.
3. With exactly one ledger visible, the fallback still works and
   `entity:` may be omitted — but naming it costs nothing and keeps
   working the day a second ledger appears.

## 8. Inviting users

1. `invite_user` invites a person to the workspace by email. Grants and
   roles are named at invite time: `grants` is one `{entity, role}`
   object per ledger they should reach — role `viewer` (read
   everything, write nothing) or `member` (full bookkeeping). Ledgers
   not granted stay invisible to them; accepting never widens beyond
   what the invite named.
2. Billing-owner-only: only the workspace's billing owner can invite —
   anyone else is refused with `role.write-denied`. Relay that refusal
   plainly and name the recovery: the billing owner sends the invite
   (this tool, or the web users page — same rules either way).
3. Caps exist, and refusals name the cap that was hit: at most one
   pending invite per address, plus pending and daily caps. The accept
   link goes only to the invitee's inbox and expires in 7 days; your
   response is the pending-invite summary and never carries the link —
   don't promise the human a link to forward.
4. Interview before inviting, §2's manners: which ledgers, which role
   on each, confirmed back before the call. An accountant usually
   starts as `viewer`; upgrade to `member` when they actually keep the
   books.

## 9. Archiving a ledger

1. `archive_entity` archives a ledger; `restore: true` un-archives it.
   Archiving is how the bill goes down — billing counts unarchived
   ledgers, so the next invoice drops — while the books stay readable:
   the ledger's files are never touched, and everyone with access can
   still read it forever by naming its slug explicitly. It disappears
   from ledger lists and refuses writes; nothing is ever deleted.
2. Billing-owner-only, and `entity:` is REQUIRED — this tool never
   falls back to a default ledger. Confirm the slug with the human and
   restate the consequences (writes stop, bill drops, reads survive)
   before calling. Never archive on your own initiative.

## 10. Billing states, as you see them

1. `get_briefing`'s `billing` section reports the workspace's state:
   `{state: "trialing", trial_days_left: N}`, `{state: "active"}`,
   `{state: "grace", grace_deadline: "...Z"}`, or
   `{state: "read_only"}` — absent means there is nothing to relay.
   Answer billing questions from it, never from memory.
   A `subscribed: true` key means the workspace holds a live Stripe
   subscription even when the state reads `trialing` (a subscriber whose
   first payment falls due when the trial ends), so answer "am I
   subscribed?" from that key, not from the state name.
2. During grace — the last payment failed — writes still work, and
   every successful write carries a `billing_warning` key naming the
   deadline writes stop at. Relay it to the human once, clearly, and
   keep working; don't repeat the warning on every call.
3. After lapse, write tools refuse with `billing.read-only`, and the
   refusal names the recovery: the billing Portal on the dashboard.
   Reads, reports, `clone_entity` and `export_entity` keep working
   forever — nobody is ever locked out of their own books. Creating a
   new ledger on a lapsed workspace refuses with
   `billing.create-blocked`, naming the same recovery. Point the billing
   owner at the Portal; don't retry the write.
4. When the human wants a copy of their books, in any billing state,
   `export_entity` returns a ten-minute link to a zip of the whole
   ledger: the plain-text journal, every document with its history, and
   every attachment. Save it where they ask (for example
   `curl -o <filename> '<url>'`) or give the link to them alone; it
   needs no sign-in, so treat it like a password. The Download link on
   their workspace page gives the same zip.
5. Connection facts worth relaying when they come up: completing a
   password reset disconnects claude.ai/Cowork connections (the human
   must reconnect and re-consent there), while agent personal access
   tokens keep working. The workspace page lists the assistants
   connected by signing in and disconnects any one of them without a
   reset. A personal access token can be given an end date when it is
   made; once it passes, the token stops working and the human makes a
   new one on the workspace page.

## 11. Local mode: the secondary path

Qunei also runs local-first: `bin/qunei-connect` installs the skills
and a stdio MCP server over a filesystem substrate, no hosted account
involved. The interview, plan, and execution above apply verbatim
there. The differences: `invite_user` and `archive_entity` are
hosted-only and refuse gracefully in local mode, billing does not
exist, and §6.3's `settings.txt` hand-edit is available because the
human actually has the files.

See qunei-bookkeeping for session discipline once the ledger exists,
qunei-conventions for adding more payee rules after this session ends,
qunei-invoicing for the configure_invoicing conversation when the human
said yes to invoicing above, and qunei-nz-gst for the configure_nz_gst
conversation when the human said yes to GST registration above.
