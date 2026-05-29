---
title: tutorials
group: Documents
category: Guides
---

# Endo Tutorials

A progressive series for learning Endo from absolute zero to fair competence.

Each tutorial assumes the previous ones.
We build one mental model — capabilities as unforgeable references — and use
it to explain everything else, in increasingly capable settings.

## Who this is for

Programmers comfortable with modern JavaScript (ES modules, async/await,
factory functions) who want to understand what Endo is and how to use it.
You do not need prior exposure to capability-based security, SES, or
distributed systems.
The tutorials will introduce those ideas as you need them.

## What you will need

- Node.js, version 20 or later.
- A terminal, and the patience to keep two of them open.
- Roughly 20 to 40 minutes per tutorial, depending on how much you stop to
  inspect what just happened.

The tutorials install Endo's published CLI from npm, not the development
monorepo.
If you are working inside this repository instead, see
[CONTRIBUTING.md](../../CONTRIBUTING.md).

## The tutorials

1. **[Alice's Car Key: Capabilities, Attenuated](01-car-key.md)**
   Define a `Car` object with `unlock` and `start`, wrap it in a "valet key"
   facet that only unlocks, hand the key to a confined guest, and watch the
   guest try (and fail) to drive away.
   By the end you understand pet names, formulas, `Far`, `harden`, eventual
   send, and the principle of attenuation.
   → [Live on suhail.ski](https://suhail.ski/posts/alices-car-key-capabilities-attenuated)

2. **[Mailing Bob the Start Fob](02-mailing-keys.md)**
   Promote Bob from "a guest on Alice's daemon" to "his own daemon on his
   own machine," issue an invitation, and watch Alice deliver the `start`
   capability across the network.
   By the end you understand invitations, the synced pet store, CapTP, and
   the fact that revocation, not delegation, is the lever you actually hold.
   → [Live on suhail.ski](https://suhail.ski/posts/mailing-bob-the-start-fob-capabilities-across-the-wire)

Further tutorials will cover delegation and per-grantee revocation, the
compartment and lockdown layer underneath, and assembling everything into a
small but real plugin system.

## Conventions

Every tutorial follows the same shape so you always know where you are.

- **What you will build.**
  A two-sentence picture of the end state, before any commands.
- **Numbered steps.**
  Each step ends with a one-line _you should now see X_ or _you should now
  be able to Y_ checkpoint, so you can verify before moving on.
- **Sidebars** for the security stories — why `harden` exists, what an
  unhardened object lets a holder do, why delegation is unstoppable.
  Sidebars can be skipped on a first pass and returned to later.
- **Vocabulary is earned inline**, one sentence at a time, the first time
  each term becomes load-bearing.
  There is no upfront glossary.

If a checkpoint does not match what you see on your screen, stop and
investigate before going further.
The fastest way to learn Endo is to take the small confusions seriously
when they appear, instead of accumulating them until the model collapses
three steps later.
