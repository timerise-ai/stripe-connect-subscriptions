---
prompts:
  - prompt: "Add marketplace payments with Stripe Connect: a buyer pays once for items from several sellers, each seller gets their share minus our 10% fee, paid out after a 7-day hold."
    stack: Stripe
  - prompt: Bill each seller on our platform a monthly fee on their saved card, with retries and emails when the card fails.
    stack: Stripe
  - prompt: One of our connected accounts looks healthy in Stripe but has never received a payout. Find out why.
    stack: Stripe
---

# Prompts

What an operator types after installing this skill, in their own words. An agent eval installs the skill
into an empty Next.js app, gives the agent one of these prompts and no further help, then type-checks, builds
and tests the result; the first prompt runs before every release. The results are the other files in this
folder. Section 10 of [STANDARD.md](https://github.com/timerise-ai/skills/blob/main/STANDARD.md) says how a
run is made. The prompts and the newest runs are on
[the skill's page](https://timerise.ai/skills/stripe-connect-subscriptions) on timerise.ai.
