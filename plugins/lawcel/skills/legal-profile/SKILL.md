---
name: legal-profile
description: Use when working in a codebase connected to Lawcel and the code could settle a legal or compliance fact about the product. Triggers - the user mentions Lawcel, compliance, privacy policy, terms of service, DPA, cookie policy, GDPR, subprocessors, the legal profile, or an open compliance case; or the work touches a third-party SDK or vendor, analytics or tracking, cookies, personal data fields, account deletion, data retention or cleanup jobs, export or portability, consent, email/SMS/push channels, auth providers, payments, logging, or where data is stored and processed. Also use once at the start of substantive work in an unfamiliar repo to check whether Lawcel has open profile questions the codebase can answer.
---

# Answering Lawcel's legal-profile questions from the codebase

Lawcel keeps a company's legal documents (privacy policy, terms of service, DPA, cookie
policy) accurate as its product changes. Those documents are generated from a **legal
profile**: a structured set of factual claims about the product. Which vendors receive user
data. How long deleted accounts are retained. What the app emails people about. Which
regions data lives in.

Most of those facts are in the codebase. None of them are in the heads of the people who
have to keep the documents correct. That gap is what you are here to close: you are already
reading this repository, so you are the cheapest accurate source of truth the company has.

The tools in this plugin are read-only, except that you can **propose** answers to profile
questions. You cannot create cases, edit documents, or change the profile.

## Do this first, once per session

Call `list_open_profile_questions`. It is cheap, read-only, and org-scoped — it returns only
the questions Lawcel has already decided are answerable from code (`codebase_derivable` or
`mixed`). Questions requiring human knowledge (jurisdiction, DPO identity, vendor contracts)
are never in this list, by design.

- Empty list → nothing to do. Say nothing, carry on with the user's actual task.
- Non-empty → tell the user what you found and **offer**, in one or two lines. Do not start
  a twenty-minute investigation uninvited, and do not bury it three paragraphs down.

> Lawcel has 4 open profile questions for this workspace, and 3 look answerable from this
> repo (third-party data sharing, account-deletion timeline, notification channels). Want me
> to work through them? I'll cite file and line for every claim, and everything goes to you
> for review before it touches a document.

If the user says yes, work through the questions one at a time. If they say no, drop it and
do not re-offer in the same session.

Do not re-poll the list on every turn. Re-poll only if you submitted answers (the list
changes), the session has run long and drifted onto a new task, or the user asks.

If the tools are not connected, say so plainly and stop. Do not fabricate the question list.

## The evidence bar

Every answer you submit needs at least one `{ path, line?, comment? }` reference, and the
server rejects submissions without one. But passing that check is not the bar. The bar is:

**A reviewer who opens the files you cited, and reads nothing else, would reach the same
conclusion.**

Practically that means:

| Answer from | Not from |
|---|---|
| The call site where a vendor SDK is initialized | Its presence in `package.json` / `requirements.txt` |
| The migration or schema that defines the column | A TypeScript interface that mirrors it |
| The cron/queue job that actually deletes rows | A `deleteUser()` that only sets `deleted_at` |
| The config value in effect, plus where it is set | A default that an env var overrides in production |
| Application code | `test/`, `fixtures/`, `examples/`, `docs/`, vendored third-party code |
| Code on the default branch | A feature branch, or your own uncommitted edits |
| Live code | Commented-out or unreachable code |

And two rules with no exceptions:

- **A dependency is not a usage.** `posthog-js` in `package.json` tells you somebody
  installed it. Find where it is initialized and whether that path runs in production. If you
  cannot, that is a `needs_human_review`, not a hedged answer.
- **Convention is never evidence.** "Most SaaS apps retain backups for 30 days" is a guess
  wearing a fact's clothes. If the number is not in a file, you do not have it.

Cite paths relative to the repo root. Include a line number whenever there is a specific line
worth pointing at. Never cite a file you did not open.

## Answer shape

Each question comes back with metadata — use it:

- `profilePath` — where the answer lands in the profile (e.g. `thirdParties`,
  `userControls.accountDeletion.timeline`). It tells you the *grain* of answer wanted.
- `schemaHint` — the shape expected (e.g. "List of vendor names with purpose, data shared,
  role, jurisdiction").
- `inputType` / `selectOptions` — when present, the answer must match. For a select, submit
  one of the given options verbatim, not a paraphrase.
- `status` — `open`, or `skipped` if the user already deferred this question once. A skipped
  question is worth *more* effort, not less: the human bounced off it, so a well-grounded
  answer is the most valuable thing you can hand them. It is not a signal to stay away.

Write plain prose. A paragraph is usually right. State what is true and where it is true;
do not editorialize about legal implications, and do not soften a fact you actually verified.

One question, one `submit_profile_answer` call. Never fold several questions into one answer.

## Worked examples

### 1. A question the code settles

> **Which third-party services do you share user data with?** (e.g. Stripe, Google Analytics,
> Sentry) — `profilePath: thirdParties`

Bad, and rejected by a careful reviewer:

> We share data with Stripe, PostHog, Sentry and Resend.
> Evidence: `package.json:31`

Four vendors, one line of evidence, and that line proves only that four packages are
installed. It does not say what each receives, and one of them may be dead.

What to do instead: for each candidate, find the initialization, then find what is passed to
it. Then submit:

> Four external processors receive user data in production:
>
> - **Stripe** — payment processing. Receives customer email and org name at checkout;
>   card data goes directly to Stripe and never reaches our servers (Checkout is hosted).
> - **Resend** — transactional email. Receives recipient email address and message body.
> - **Sentry** — error monitoring. Receives the authenticated user's id and email, attached
>   as user context on every captured exception.
> - **PostHog** — product analytics. Receives user id, org id and page-view events. Loaded
>   only when `PUBLIC_POSTHOG_KEY` is set, which it is in the production env template.
>
> No other outbound SDK sends user-identifiable data. Google Analytics appears in
> `docs/legacy-setup.md` but has no call site in the application.
>
> Evidence:
> - `src/lib/server/stripe.ts:14` — `new Stripe(env.STRIPE_SECRET_KEY)`
> - `src/routes/api/checkout/+server.ts:47` — `customer_email` passed to Checkout
> - `src/lib/server/email.ts:9` — Resend client; `:52` sends `to`, `subject`, `html`
> - `src/hooks.client.ts:22` — `Sentry.setUser({ id, email })`
> - `src/lib/analytics.ts:11` — PostHog init guarded by `PUBLIC_POSTHOG_KEY`
> - `.env.production.example:18` — `PUBLIC_POSTHOG_KEY` set

Every claim maps to a reference. The negative claim about Google Analytics is stated *and*
sourced, because "we checked and it is not used" is exactly the kind of fact a policy
depends on.

### 2. A question the code only half settles

> **When a user requests account deletion, how long before their data is fully deleted?** —
> `profilePath: userControls.accountDeletion.timeline`, `schemaHint: "e.g. '30 days',
> '90 days', 'immediately'"`

You find `deleteAccount()` in `src/lib/server/users.ts:88` setting `deleted_at = now()`. You
grep for anything that consumes `deleted_at` and find only query filters. No purge job, no
retention constant, nothing in the scheduler config.

The tempting answer is "30 days" because that is what everyone does. That answer would be
invented, and it would end up in a published privacy policy as a promise the product does not
keep. Instead:

`submit_profile_needs_human_review` with:

> Account deletion is a soft delete only. `src/lib/server/users.ts:88` sets `deleted_at` and
> every read path filters on it, but nothing in the repo purges the rows — no cron entry, no
> queue job, no retention constant. So the codebase cannot tell you the actual deletion
> timeline; it may be a manual or infrastructure-level process. Backup retention would also
> need to be included in the answer and is not visible here either.

That note is worth more to the reviewer than a number would be: it tells them what is true,
what is missing, and that there may be a gap between the policy and the product.

### 3. A question that looks answerable and is not

> **Which communication channels do you use to contact users?** (email, push, SMS, in-app) —
> `profilePath: communications.channels`

You find a Twilio client and an SMS helper. You are one grep from answering "email and SMS".

Then you check the call sites: the SMS helper is referenced only from
`scripts/backfill-notify.ts`, a one-off script, and from a test. Nothing in the request path
calls it. Twilio credentials are absent from the production env template.

Answer what you verified, and say what you found about the rest:

> Email is the only channel in production use. All user-facing sends go through
> `src/lib/server/email.ts` (Resend) and cover transactional messages only — invites,
> password resets, case notifications — with no marketing sends in the codebase. An SMS
> helper exists but is called only from a one-off backfill script and a test, and the
> production env template has no Twilio credentials. No push or in-app notification code.
>
> Evidence:
> - `src/lib/server/email.ts:52` — the single `resend.emails.send` call site
> - `src/lib/server/notify/sms.ts:18` — Twilio helper
> - `scripts/backfill-notify.ts:40` — only non-test caller
> - `.env.production.example` — no `TWILIO_*` keys

The difference between this and "email and SMS" is one grep for call sites. That grep is the
job.

## When the codebase does not settle it

Reach for `submit_profile_needs_human_review` — with a note — whenever any of these is true:

- The fact lives outside the code: a contract, an infrastructure setting, a business decision,
  an internal policy document.
- You found the mechanism but not the value (a retention job whose window comes from an env
  var you cannot see).
- Two plausible readings exist and the code does not choose between them.
- The code contradicts itself, or contradicts something you saw in `list_documents`.
- You would have to say "probably", "typically", or "presumably" to write the answer.

Write the note as if the reader has not seen the repo: what you looked for, where you looked,
what you found, and what is still missing. A flagged question with a good note is a **success**.
It is strictly better than a plausible-sounding guess, and the reviewer's time is the scarce
resource here.

Never leave a question silently unanswered. Answer it, or flag it.

## What happens to your answers

Every answer lands in `pending_review`. Nothing you submit is applied to the profile, and
nothing reaches a legal document, until a human at the company opens it in the Lawcel
dashboard and accepts it. Your evidence references are shown alongside the answer so the
reviewer can click straight to the files.

This is why the evidence bar is where it is. A wrong answer is not a wrong string in a
database — it is a false statement in a published privacy policy that real users read and
regulators can act on. The review step exists to catch that, but a reviewer who has learned
your answers need re-checking from scratch is a reviewer you have cost time rather than saved.

Close the loop with the user in one line — how many answers you submitted, how many you
flagged, and that review happens in Lawcel:

> Submitted 3 answers and flagged 1 for you (account-deletion timeline — the repo only does a
> soft delete). They're waiting for review in Lawcel; nothing changes in your documents until
> you accept them.

## The other tools

`list_cases` and `list_documents` are read-only lookups. Reach for them when:

- **The user asks about compliance state** — "do we have any open compliance issues?", "what
  did Lawcel flag on that PR?", "what does our privacy policy actually say about X?".
  `list_cases` returns the 50 most recently updated cases with status, risk score and the
  documents each affects. `list_documents` returns full current document text.
- **You are about to assert what a policy says.** Read it with `list_documents` rather than
  paraphrasing from memory or from a stale copy in the repo.
- **A profile answer needs a cross-check.** If the code says one thing and the published
  document says another, that discrepancy belongs in your answer or your review note — it is
  precisely the drift Lawcel exists to catch.
- **You just made a change with obvious compliance surface** — added a vendor SDK, a new
  personal-data column, a new outbound integration. Mention it and offer to check whether the
  documents already cover it. Do not silently open an investigation.

Neither tool changes anything, and there is no tool that creates a case — do not tell the
user you have filed or opened anything.

## Failure modes, in the order they actually happen

1. **Answering from `package.json`.** The single most common one. A dependency list is a
   list of things somebody once installed.
2. **Guessing a number.** Retention windows, deletion timelines, log rotation. If the number
   is not in a file, flag it.
3. **Silently over-scoping.** Being asked what data you share and answering what data you
   *collect*. Re-read the question and the `profilePath` before you write.
4. **Answering `mixed` questions as if they were purely technical.** For a `mixed` question,
   state the code fact and scope it to the code ("the application deletes X after 30 days,
   per `<file:line>`"). Do not restate it as a company-wide commitment — that is the
   reviewer's call to make, not yours.
5. **Skipping the offer.** Doing the work unasked is as wrong as not doing it. Ask first.
6. **Treating a `skipped` question as out of bounds.** It is the opposite: it is the one the
   human most needs help with.
7. **Reporting an answer as applied.** It is pending review. Say pending review.

## If something is not working

- **The tools are missing entirely** — the MCP server has not connected. `/mcp` shows its
  state; connecting opens a browser window where the user signs in to Lawcel and approves.
- **The browser step says agent access is not on this plan** — connecting a coding agent
  requires a paid Lawcel plan. Tell the user; there is nothing to retry.
- **A call returns 401** — the connection was revoked (Connections → MCP in Lawcel) or the
  token expired. Re-approving through `/mcp` fixes it.
- **`submit_profile_answer` returns "not open"** — somebody answered it first. Re-list and
  move on.
- **"not answerable by an agent"** — the question is `org_knowledge` and human-only. Do not
  try to answer it another way.
