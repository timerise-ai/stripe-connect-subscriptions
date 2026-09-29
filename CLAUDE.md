# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

An [Agent Skill](https://agentskills.io) package: markdown only. There is no `package.json` here and nothing
in this repository executes. It teaches an agent to build two money flows on one Stripe account in a
**Next.js App Router** app: Connect marketplace settlement (separate charges and transfers, escrow, reserves,
payouts) and platform subscription billing (off-session charges on a saved card, with dunning).

Keep the two straight: the commands and code in `references/` describe the app the agent will generate, not
this repository. The `stripe listen` and cron `curl` calls in `operations.md`, the host probe in
`adaptation.md`, the SQL in `data-model.md` and `store.md` and the tests in `money.md` and `subscriptions.md`
all run in that generated app.

The skill was written by the engineers who have shipped this module; the earlier implementation it was
audited against was the payments and billing module of a multi-vendor marketplace on Stripe Connect.
`references/provenance.md` is the ledger of that audit: eighteen entries, split into what the audit fixed and
how the templates verify it, what was kept deliberately, and what was designed here and has never run in
production, plus the claims that could not be verified. That file is the rationale layer: read it before
"simplifying" anything.

## Structure

- `SKILL.md`: entry point, loaded whole on every activation, so it stays between 130 and 160 lines, the
  closing index line aside. The frontmatter `description` is the trigger surface; the body carries the
  architecture diagram, six **critical facts**, five **hard rules**, the quick-start order, the **reference
  directory table** mapping trigger keywords to files, and a closing line linking the skills index.
- `README.md`: the human-facing front door, in the section order of the skill standard: install, activation,
  the file table, the five non-negotiables, adaptation, the *Not this* table, contributing.
- `references/*.md`: one topic per file, loaded on demand. `adaptation.md` carries the seam contract and the
  rename table; `store.md` the `PaymentsStore` data-access contract and its three atomic claims;
  `architecture.md` why the money moves the way it does; `money.md`, `data-model.md`, `stripe-adapter.md`,
  `connect-accounts.md`, `webhooks.md`, `settlement.md`, `reconciliation.md`, `subscriptions.md` and
  `operations.md` one layer each; `provenance.md` the audit.
- `evals/`: `prompts.md` holds what an operator types after installing, in their words; the first prompt
  is the agent eval run before every release. Every other file there is one eval run: measured frontmatter
  that is never edited, then the notes of the person who ran it. Add a prompt rather than rewording one that
  has results. The procedure is section 10 of the index's STANDARD.md.
- `.github/workflows/agent-eval.yml`: the caller of the index's reusable eval workflow, run on every
  published release and on a maintainer's dispatch. It is copied verbatim from the standard and is the same
  in every skill; do not edit it, and never add a trigger on `push` or `pull_request`.

## Editing conventions

- **Code blocks name their destination on the first line** as a comment, for example
  `// lib/payments/settlement.ts` or `-- migrations/0001_payments.sql`. A block that continues a file already
  introduced in the same reference omits it. Keep imports complete and types explicit: every TypeScript
  block is written to compile under `strict` and `noUncheckedIndexedAccess`.
- **Identifiers are shared across files.** `PaymentsStore`, `stripeClient`, `resolveTransferSource`,
  `reversibleHeadroom`, `applyAccountUpdate`, `distribute`, `fn_claim_settlement`, `STRIPE_COUNTRIES`, and
  the env names `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `STRIPE_CONNECT_WEBHOOK_SECRET` and
  `CRON_SECRET` appear in several references. Rename in all of them or none.
- **Keep the three tables in sync** with `references/`: the reference directory in `SKILL.md`, the quick-start
  list in `SKILL.md`, and the file table in `README.md`. Links are relative: `[x.md](references/x.md)` from
  `SKILL.md`, `[x.md](x.md)` between references.
- **Do not remove the odd-looking parts.** A transfer rejection that does not throw, a webhook insert treated
  as a claim rather than a receipt, the retry sweep reusing the settlement idempotency key, a currency
  mismatch parked instead of converted, `account.updated` skipping the idempotency table, and separate
  country lists for payouts and addresses: each is a ledger entry. Check `provenance.md` before touching one.
- **The numbers that remain are load-bearing.** Stripe's 24-hour idempotency window, the `numeric(19,4)`
  money type, the 300-second settlement lease, three dunning attempts three days apart, the money module's
  ten tests and the ledger's eighteen entries. They are vendor facts, design parameters or counts of this
  repository. Do not restate them loosely and do not add new ones. Figures describing the earlier
  implementation's deployment do not appear anywhere.
- **Mark additions as additions.** Anything designed in the skill and never run in the earlier implementation,
  such as the `PaymentsStore` seam, belongs in the "Added" section of `provenance.md`, stated as such. The
  skill's credibility is that it distinguishes the two.
- **Plain punctuation.** No em-dash, en-dash, arrow, middle dot or smart quote anywhere, diagrams included:
  draw them in ASCII. Prose wraps at 110 columns.
- **Evals are not skill content.** A new prompt or an eval result is committed as `chore(evals): ...`,
  never causes a version bump and never rides in a release commit. The frontmatter of a result file is what
  was measured and is not edited; a failing run stays committed, and the fix is the next release.
- **Never present the non-negotiables as optional.** A failed transfer never fails the settlement, a duplicate
  webhook insert is never "already handled", a transfer is never re-sent without asking Stripe, settlement
  exclusivity comes from a leased claim, and no reversal exceeds the transfer's headroom. They are stated as
  hard rules in `SKILL.md` and as non-negotiables in `README.md`, in the same order; keep them that way
  everywhere.
- `Tenant`, `Order`, `VendorOrder` and `Partner` are meant to be renamed by the host, and the rename table in
  `adaptation.md` carries that procedure. Stripe's own terms (`transfer_group`, `source_transaction`,
  `acct_`) and the env var names are the authoring contract and are not renamed.
