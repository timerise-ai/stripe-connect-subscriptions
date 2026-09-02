# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
- README: the install leads with `npx skills add timerise-ai/stripe-connect-subscriptions`, which installs the skill
  into every skills-compatible agent it detects, with the `-a` form for named agents; the
  Claude Code clone moves under a *Manual install* heading. Activation gets its own
  heading, and a *Not this* table points neighbouring problems to the right skill or tool.
- README: *Usage* becomes *Activation*, *What it covers* becomes *What's inside*, *The five
  hard rules* becomes *The five non-negotiables*, and the *Versioning* section is dropped in
  favour of the current-release line and this changelog.
- README: the skill's origin is reworded. It was written by the engineers who built the
  module it describes; the reference point for `provenance.md` is the earlier
  implementation rather than "the source"; the index is called Timerise Skills.
- README: every em-dash, arrow and en-dash in the prose is rewritten as a comma, colon,
  full stop or conjunction.

## [0.1.3] - 2026-09-01

Documentation only — the skill content is unchanged from 0.1.2.

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
  **relational** backend — the ledger relies on `sum()` over indexed rows, a
  unique constraint for webhook idempotency, and atomic conditional updates —
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

Documentation and licensing only — the skill content is unchanged from 0.1.0.

### Added
- `README.md` covering installation (personal and project scope), usage and
  trigger vocabulary, the architecture summary, the adaptation contract, and the
  versioning and contributing conventions.
- The five hard rules reproduced in the README, so the module's
  non-negotiables are visible without opening `SKILL.md`.
- `LICENSE` — MIT. The README declared MIT but no license text shipped, leaving
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
- `references/provenance.md` recording what was extracted from the source
  module, what was kept deliberately, what was designed here (the
  `PaymentsStore` seam, the alerting list, the "leave behind" guidance), and
  which claims could not be verified.

### Fixed
Hardened against four defects found while auditing the source module:
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
