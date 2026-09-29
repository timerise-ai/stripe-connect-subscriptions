# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.11] - 2026-09-29

Fix release, from scoring the prompt-1 agent eval runs against 0.1.10.

### Fixed
- Settlement writes the escrow holds, the reserve and the vendor order's status for every vendor order on every
  run (`settlement.md`). Written only beside a newly created seller leg, they were skipped on resume after a
  crash, and never written for a seller not yet onboarded, whose transfer the retry sweep later funded with
  nothing to release it. `escrow_holds.order_item_id` and the new `reserves.vendor_order_id` are unique, the
  two store inserts ignore a duplicate, and `release-escrow` waits for a funded seller leg. Apps built from
  earlier versions should add the migration, the insert-or-ignore and the release condition, and create
  holds for any paid vendor order that has none.

### Changed
- The quick start names the seams (store implementation, auth, validation, audit) and says settlement,
  retry, onboarding and money logic are copied, not redesigned; a payout hold sets both escrow constants; the
  handover is in the final message.
- `provenance.md` records the fix under *Added*; the ledger has eighteen entries.

## [0.1.10] - 2026-09-29

Fix release, from scoring the prompt-1 agent eval runs against 0.1.9.

### Fixed
- Connected accounts are set to manual payouts during onboarding (`connect-accounts.md`). On Stripe's
  automatic schedule the connected balance was paid out to the seller's bank before the escrow hold
  released, and the payout gate never ran. Apps built from earlier versions should add the
  `balanceSettings.update` call to their onboarding and run it once for every existing connected account.
- `distribute` ranks remainders by magnitude (`money.md`). A negative total handed the leftover grain to the
  smallest share, so a prorated reversal did not mirror the positive split. Apps built from earlier
  versions should copy the new sort; the money module now has ten tests.

### Changed
- The quick start in `SKILL.md` says to copy the templates and `money.test.ts` verbatim, names `stripe`,
  `vitest` and `pg` as the module's dependencies, maps a payout hold to the escrow window, and lists what to
  hand over to the operator. `adaptation.md`, `settlement.md` and `data-model.md` say the same in place.
- `STRIPE_COUNTRIES` ships as the platform's own country instead of a placeholder.
- `operations.md` lists `CRON_SECRET` with the other env variables and says what `.env.example` holds.
- `provenance.md` records both fixes under *Added*.

## [0.1.9] - 2026-09-29

Brings the repository to the current skill standard. The money model, the templates' logic and the
non-negotiables are unchanged from 0.1.8.

### Added
- `CLAUDE.md`, the editing conventions for an agent working on the skill itself.
- `evals/prompts.md`, the three prompts the agent evals run, and
  `.github/workflows/agent-eval.yml`, the index's eval caller, copied verbatim from the standard.
- `adaptation.md` opens its seam section with the full seam table.

### Changed
- `SKILL.md` keeps only the headings of the standard: the Adaptation Contract table moves to
  `adaptation.md`, which the quick start names as the seam contract. The frontmatter description names the
  `PaymentsStore` seam.
- README: the intro is three paragraphs, the manual clone sits under *Manual install*, the file table lists
  every file in the repository, the non-negotiables use the same wording as the hard rules, and
  *Contributing* points to `CLAUDE.md`.
- Every em-dash, en-dash, arrow and other non-ASCII symbol in the repository's markdown is rewritten in
  plain punctuation, and every diagram is redrawn in ASCII.

### Fixed
- Code blocks that named no destination now do: `subscriptions.md`, `settlement.md`,
  `reconciliation.md`, `store.md`, `connect-accounts.md` and the schema in `data-model.md`.
- The money module lives at `lib/payments/money.ts`, where `stripe-adapter.md` already imported it from;
  `money.md` named `lib/money.ts`.
- Older changelog entries no longer use the standard's banned words.

## [0.1.8] - 2026-09-21

Wording release. The skill content is unchanged from 0.1.7.

### Added

- `SKILL.md` closes with a line linking the
  [Timerise Skills](https://github.com/timerise-ai/skills) index, so an agent that
  has the skill loaded can find the sibling skills for neighbouring modules without
  leaving the entry point.

## [0.1.7] - 2026-09-02

Wording release. Templates and technical content are unchanged from 0.1.6.

### Changed
- README and `SKILL.md` describe the module by the properties the templates hold and
  what verifies them, in the frontmatter description, the intro, the critical facts,
  the hard rules and the non-negotiables. The audit record stays in
  `references/provenance.md`, linked from both.

## [0.1.6] - 2026-09-02

Documentation-only release. The skill itself, `SKILL.md` and `references/`, is
unchanged from 0.1.5.

### Changed
- `README.md` and `references/provenance.md`: the size figures of the earlier
  implementation (its file count and the span of its incident history) are gone,
  replaced by the shape of the module: the payments and billing module of a
  multi-vendor marketplace with an incident history behind it. Design parameters and
  the skill's own test count stay.

## [0.1.5] - 2026-09-02

Wording release. The origin and audit statements across the skill follow section 2 of
the skill standard; templates and technical content are unchanged from 0.1.4. The
repository history starts at this release.

### Changed
- Origin and audit wording across `SKILL.md`, `README.md` and `references/` now follows
  the skill standard: the reference point is the earlier implementation, stated
  in the standard's own words. The frontmatter description says the settlement
  internals are hardened against the defects the audit found.
- `provenance.md`: the onboarding-code section describes the withdrawn helper in the
  region rule's own terms, and the porting section is retitled for anyone fixing an
  existing implementation.

## [0.1.4] - 2026-09-02

Documentation-only release. The skill itself, `SKILL.md` and `references/`, is
unchanged from 0.1.3.

### Added
- README: badges under the title for the Agent Skills format, the skills.sh install, and
  Claude Code, Codex CLI and Gemini CLI compatibility.

### Changed
- README: the install leads with `npx skills add timerise-ai/stripe-connect-subscriptions`,
  which installs the skill into every skills-compatible agent it detects, with the `-a` form for named agents; the
  Claude Code clone moves under a *Manual install* heading. Activation gets its own
  heading, and a *Not this* table points neighbouring problems to the right skill or tool.
- README: *Usage* becomes *Activation*, *What it covers* becomes *What's inside*, *The five
  hard rules* becomes *The five non-negotiables*, and the *Versioning* section is dropped in
  favour of the current-release line and this changelog.
- README: the skill's origin is reworded. It was written by the engineers who built the
  module it describes; the reference point for `provenance.md` is the earlier
  implementation, named in the standard's words; the index is called Timerise Skills.
- README: every em-dash, arrow and en-dash in the prose is rewritten as a comma, colon,
  full stop or conjunction.

## [0.1.3] - 2026-09-01

Documentation only. The skill content is unchanged from 0.1.2.

### Added
- A one-command install via [skills.sh](https://www.skills.sh),
  `npx skills add timerise-ai/stripe-connect-subscriptions`, promoted above the
  manual clone, with the `-a` form for installing into named agents only.
- A "Codex CLI, Gemini CLI and other agents" section: nothing in the skill is
  Claude-specific, so it also loads from `~/.agents/skills`. Documents the
  symlink that lets one clone serve every agent, and the per-host invocation
  syntax and activation caveat.

### Changed
- The adaptation section now states that the `store` seam requires a
  **relational** backend, since the ledger relies on `sum()` over indexed rows, a
  unique constraint for webhook idempotency, and atomic conditional updates,
  and names the two reference implementations that ship: raw SQL/Postgres and
  the Supabase client.
- The intro calls the module an Agent Skill rather than a Claude Code skill,
  matching its actual portability.

### Fixed
- `"Funds can't be sent to accounts located in"` was listed as a trigger in
  `SKILL.md` but missing from the README's trigger vocabulary.
- The clone URL now carries the `.git` suffix.

## [0.1.2] - 2026-08-30

### Changed
- `LICENSE` names the legal entity, Timerise Sp. z o.o., matching every other
  skill in the [index](https://github.com/timerise-ai/skills).

## [0.1.1] - 2026-08-29

Documentation and licensing only. The skill content is unchanged from 0.1.0.

### Added
- `README.md` covering installation (personal and project scope), usage and
  trigger vocabulary, the architecture summary, the adaptation contract, and the
  versioning and contributing conventions.
- The five hard rules reproduced in the README, so the module's
  non-negotiables are visible without opening `SKILL.md`.
- `LICENSE`: MIT. The README declared MIT but no license text shipped, leaving
  the terms unenforceable for anyone cloning the skill.
- A link to the [Timerise skills index](https://github.com/timerise-ai/skills)
  and a note that the skill can also be invoked explicitly with
  `/stripe-connect-subscriptions`.

### Fixed
- Clone URLs in the README pointed at the `timerise-io` organisation, which does
  not host this repository; both now point at `timerise-ai`.

## [0.1.0] - 2026-08-15

Initial release of the stripe-connect-subscriptions skill.

### Added
- `SKILL.md` entry point covering the two money flows (Connect marketplace
  fan-out and platform subscription billing), the architecture diagram, six
  critical facts, five hard rules, the adaptation contract, and the reference
  directory mapping trigger keywords to reference files.
- `references/` topic files covering the money model (`architecture.md`),
  schema and RLS (`data-model.md`), the `PaymentsStore` contract (`store.md`),
  amounts and rounding (`money.md`), the Stripe client and dual webhook secrets
  (`stripe-adapter.md`), Connect onboarding and account state
  (`connect-accounts.md`), idempotent event receipt (`webhooks.md`), settlement
  fan-out with escrow and reserves (`settlement.md`), transfer retry and fee
  and clawback reconciliation (`reconciliation.md`), off-session billing and
  dunning (`subscriptions.md`), setup and runbook (`operations.md`), and the
  host-fitting seams (`adaptation.md`).
- `references/provenance.md` recording what the audit of the earlier
  implementation changed, what was kept deliberately, what was designed here (the
  `PaymentsStore` seam, the alerting list, the "leave behind" guidance), and
  which claims could not be verified.

### Fixed
Hardened against four defects found while auditing the earlier implementation:
- Crash-resume in settlement keyed the skip set on `vendorOrderId` alone, so a
  crash between the partner and seller transfer writes stranded the seller leg
  permanently. Templates ship two resume sets filtered by `destinationKind`.
- Subscription dunning deduped a failure on the intent id, which a network
  error or timeout never carries, burning two attempts on one declined charge.
  Templates treat a recorded failure with a null intent id as already-applied
  and adopt the id for later deliveries.
- The Stripe client pinned no `apiVersion`, so bumping the SDK silently shifted
  request and response shapes across money-carrying call sites. Templates pin
  it explicitly alongside `maxNetworkRetries`.
- Reconciliation sweeps were unbounded.
