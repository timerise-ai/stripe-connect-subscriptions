# stripe-connect-subscriptions

[![Agent Skills](https://img.shields.io/badge/Agent_Skills-open_format-059669)](https://agentskills.io)
[![skills.sh](https://img.shields.io/badge/skills.sh-npx_skills_add-059669)](https://www.skills.sh)
[![Claude Code](https://img.shields.io/badge/Claude_Code-compatible-059669)](https://docs.claude.com/en/docs/claude-code/skills)
[![Codex CLI](https://img.shields.io/badge/Codex_CLI-compatible-059669)](https://developers.openai.com/codex/skills)
[![Gemini CLI](https://img.shields.io/badge/Gemini_CLI-compatible-059669)](https://github.com/google-gemini/gemini-cli/blob/main/docs/cli/skills.md)

An [Agent Skill](https://agentskills.io) that teaches an agent to build **Stripe Connect marketplace money
movement** and **Stripe platform subscription billing** in a **Next.js App Router** app: two money flows on
one Stripe account. A marketplace charges the buyer once and fans the proceeds out to many merchants (Connect,
*separate charges and transfers*). A platform subscription charges those merchants a recurring fee
off-session (Billing). They share a client, a webhook pair and a money type.

The insight that shapes the whole skill: **the buyer's charge and the merchants' payouts fail independently.**
By the time settlement runs the money has already moved, so a rejected transfer must never fail the charge or
the webhook, and must stay re-drivable later without paying anyone twice.

Written by the engineers who have shipped this module. The earlier implementation it was audited against was
the payments and billing module of a multi-vendor marketplace on Stripe Connect. The templates hold four
properties end to end: every webhook is claimed once and re-run to completion after a crash, every transfer is
retried with adoption rather than re-sent, every seller is inside the platform's payout corridor before a
charge is split, and every fee, reversal and clawback reconciles to the ledger. The money module's ten tests
pin the rounding and distribution; the atomic claims in the store contract carry the rest. The record of what
the audit changed is in [`references/provenance.md`](references/provenance.md).

## Install

One command, via the [skills.sh](https://www.skills.sh) CLI, which installs the skill into every
skills-compatible agent it detects, including Claude Code, Codex CLI and Gemini CLI:

```bash
npx skills add timerise-ai/stripe-connect-subscriptions
```

Name the agents instead with `-a`, for example
`npx skills add timerise-ai/stripe-connect-subscriptions -a claude-code -a codex`.

### Manual install

Nothing here is Claude-specific: the skill is a plain [Agent Skills](https://agentskills.io) folder,
`SKILL.md` plus markdown references with no file that calls a model, so cloning it into an agent's skills
directory is all an install is. For Claude Code:

```bash
git clone https://github.com/timerise-ai/stripe-connect-subscriptions.git ~/.claude/skills/stripe-connect-subscriptions
```

To scope it to a single project instead, clone it into that project's `.claude/skills/` directory. For another
agent, clone into that agent's skills directory, or symlink the Claude Code copy so one `git pull` updates
every agent:

```bash
mkdir -p ~/.agents/skills
ln -s ~/.claude/skills/stripe-connect-subscriptions ~/.agents/skills/stripe-connect-subscriptions
```

Update the skill with `git pull` in its directory. The current release is **0.1.10**. See
[`CHANGELOG.md`](CHANGELOG.md). The [skills index](https://github.com/timerise-ai/skills) lists the other
Timerise Skills and how to install them all at once.

## Activation

The skill activates automatically when a task matches its description: building a split payment, onboarding a
connected account, debugging a webhook, or wiring off-session subscription billing; also on the vocabulary:
`transfer_group`, `source_transaction`, `application_fee_amount`, `acct_`, `whsec_`, `off_session`,
`SetupIntent`, Express onboarding, `account.updated`, *"the merchant never gets paid"*, or *"Funds can't be
sent to accounts located in"*. Invoke it explicitly with `/stripe-connect-subscriptions` in Claude Code,
`$stripe-connect-subscriptions` in Codex CLI, or from `/skills` in Gemini CLI.

Each host matches a task against the description its own way, so invoke the skill explicitly on a first run
rather than assuming it fired. Only `SKILL.md` is read up front; the `references/` files load on demand, so
the skill stays cheap in context until a topic is actually needed.

## What's inside

| File | Contents |
|---|---|
| `SKILL.md` | Entry point: when to use and when not to, the architecture, six critical facts, five hard rules, the quick start, and the reference directory |
| `README.md` | This file: the human-facing front door |
| `CHANGELOG.md` | Every release, newest first |
| `CLAUDE.md` | What this repository is and its editing conventions, for an agent editing the skill itself |
| `LICENSE` | MIT |
| `references/adaptation.md` | The seam contract with the host app: fitting it to your app, including which parts to leave behind |
| `references/architecture.md` | Why separate charges and transfers, and the ledger it implies |
| `references/data-model.md` | Schema, columns, RLS |
| `references/store.md` | The `PaymentsStore` data-access contract and its atomic claims |
| `references/money.md` | Amounts, rounding, largest-remainder distribution |
| `references/stripe-adapter.md` | Stripe client, pinned API version, dual webhook secrets |
| `references/connect-accounts.md` | Connect onboarding, `account.updated`, capability flags |
| `references/webhooks.md` | Receiving events idempotently (claims, not dedupe) |
| `references/settlement.md` | Settlement fan-out, escrow holds, rolling reserves |
| `references/reconciliation.md` | Transfer retry with adoption, fee and clawback reconciliation |
| `references/subscriptions.md` | Off-session charges, saved cards, dunning, suspension |
| `references/operations.md` | Env, setup order, crons, observability, troubleshooting |
| `references/provenance.md` | The audit record: what changed from the earlier implementation, what was kept, and what is unverified |
| `evals/` | The prompts an operator types after installing (`prompts.md`) and one file per agent eval: the skill installed into an empty Next.js app, one prompt, no help, then type-checked, built and tested |
| `.github/workflows/agent-eval.yml` | Runs the agent evals on every published release through the index's reusable workflow; the same in every skill |

The skill is server-side only and backend-agnostic behind one `store` object. It needs a **relational** store,
because the ledger relies on `sum()` over indexed rows, a unique constraint for webhook idempotency, and
atomic conditional updates for every state transition, and it ships two reference implementations side by
side: raw SQL/Postgres (portable to Drizzle, Prisma, Kysely) and the Supabase client. The `PaymentsStore` seam
was introduced when the skill was written and has never run as an abstraction, so its three atomic claims are
the places most likely to be implemented wrongly, and they are flagged as such.

## The five non-negotiables

These travel with the module and are never optional (they are the hard rules in `SKILL.md`):

1. **Never let a failed transfer fail the settlement.** Record the leg unfunded and continue. The charge has
   already succeeded, so the webhook must succeed too; the retry cron funds the leg later.
2. **Never treat a duplicate webhook insert as "already handled".** The insert is a *claim*; the retry after a
   delivery crashed mid-handling re-runs the handlers, so every event runs to completion exactly once.
3. **Never re-send a transfer without asking Stripe whether it already exists.** A leg recorded unfunded
   because the *response* was lost pays twice once Stripe's 24h idempotency window passes.
4. **Never derive settlement exclusivity from a status flag written at the end.** Two callers reach settlement
   routinely; an atomic leased claim is what makes inventory and transfers happen once.
5. **Never reverse more than a transfer's remaining headroom.** Stacked reversals, such as a gateway fee and
   then a refund, are rejected past the original amount.

Everything else is the host app's: ORM, auth, scheduler, vocabulary, UI.

## Adaptation

The host supplies the other half of each seam:

| Seam | The skill ships | The host supplies |
|---|---|---|
| Domain entities | `Tenant`, `Order`, `VendorOrder`, `Partner` | Its own vocabulary |
| Auth guard | An adapter signature per route | Clerk, NextAuth, Supabase, custom |
| Data access | A `PaymentsStore` contract + SQL | Its ORM or SDK |
| Background work | Crons and their cadence | Its scheduler |
| UI | Nothing | All of it |

## Not this

| Not this | Use instead |
|---|---|
| Single-seller checkout, no split | Stripe Checkout / Payment Element directly |
| PayPal Marketplace onboarding + payouts | A PayPal skill; only the seams are shared |
| Card UI, Payment Element styling | Your design system; this skill is server-side |

## Contributing

Issues and pull requests are welcome here. Pure markdown, with no build, lint or test step in this repository.
Code blocks name their destination on the first line, and every template is written to compile under `strict`
and `noUncheckedIndexedAccess`, so keep imports complete and types explicit. Claims in this skill are meant
to be verifiable: if you change a factual claim, say how you verified it, whether against the Stripe API, the
Stripe docs, or a reproduction.

Adding, removing or renaming a file in `references/` means updating the quick start and the reference
directory table in `SKILL.md`, the file table above, and any relative cross-links. The odd-looking parts of
the templates are there for reasons `references/provenance.md` records, and that ledger must stay truthful:
read it before simplifying anything, and add an entry for anything you change. Commits follow Conventional
Commits and releases follow [STANDARD.md](https://github.com/timerise-ai/skills/blob/main/STANDARD.md) in the
index; `CLAUDE.md` carries the full editing conventions.

## Part of the Timerise Skills

This is one of the [Timerise Skills](https://github.com/timerise-ai/skills): modules for **Next.js App
Router** apps written by our own senior engineers from the modules they have shipped, not synthetic, each
published as its own repository and indexed there. They share one layout, so an agent that has read one knows
how to read the next: a `SKILL.md` entry point, `references/` loaded on demand, and a seam contract carrying
the module's non-negotiables.

## Author

Built and maintained by [Timerise](https://timerise.ai).

## License

MIT. See [LICENSE](LICENSE).
