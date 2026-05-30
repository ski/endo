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

3. **[One Key Per Valet: Per-Grantee Wrappers and Independent Revocation](03-per-grantee-wrappers.md)**
   Build a `makeRevocableForwarder` factory. Mint one wrapper per grantee.
   Revoke Bob's without touching Charlie's. The Car keeps running throughout.
   By the end you understand per-grantee wrappers, the forwarder/revoker
   separation, and why capability policy is structural rather than declarative.

4. **[Bob Makes a Request: The Endo Mailbox and Forms](04-mailbox-and-requests.md)**
   Bob asks Alice for a capability rather than waiting for her to push one.
   Alice sees the request in her inbox, approves or rejects it, and Bob's
   code handles both outcomes. Covers persistent settled requests, pipelining
   off a promise before the host acts, and following the inbox in real time.

5. **[The Plugin That Can't Escape: Hardened Compartments](05-confined-plugins.md)**
   Run untrusted third-party code in a Hardened JavaScript Compartment. Watch
   three escape attempts — `fetch`, `process.env`, prototype mutation — each
   fail cleanly. Then see the plugin do its legitimate work through an
   attenuated capability. Attenuation and confinement compose independently.

6. **[The Smart Garage: A Capability Broker](06-capability-broker.md)**
   Assemble tutorials 1–5 into a coherent system: a garage broker that
   provisions per-grantee forwarders to third-party plugins, audits active
   grants, revokes any plugin independently, and emergency-stops everything
   at once. Two plugins — one well-behaved, one adversarial — each get the
   access they were given and nothing more.

7. **[What the Garage Is Missing: Toward a Blockchain Capability Marketplace](07-toward-agoric.md)**
   A bridge chapter. Five things the garage from tutorial 6 cannot do: trust is
   personal not contractual, payment and access are not atomic, there is no
   marketplace, the fob cannot be traded, revocation is Alice's word not the
   chain's. Each gap maps to a specific part of Agoric's stack. Tutorials 8
   through 12 answer the questions this one poses.

8. **[The Car as a Digital Asset: ERTP](08-ertp.md)**
   Turn the car's access right into a tradeable asset using Agoric's Electronic
   Rights Transfer Protocol. Mint a non-fungible payment, deposit it into a
   purse, transfer it to Charlie without Alice's involvement, and burn a
   revoked payment. Every ERTP primitive maps directly to a concept from
   tutorials 1–3 — supply control, attenuation, and revocation, standardised.

9. **[The Garage as a Smart Contract: Zoe](09-zoe.md)**
   Port the garage to a Zoe smart contract on a local Agoric chain. Alice
   lists a CarAccess fob for 50 IST. Bob makes a matching offer. Zoe holds
   both sides in escrow and releases them atomically — neither party can
   cheat the other. The offer safety guarantee is structural, not legal.

10. **[The Listing Board: An On-Chain Capability Marketplace](10-marketplace.md)**
    Multiple sellers post CarAccess listings at different prices. Buyers browse
    the board, pick a listing, and buy atomically through Zoe. No discovery gap,
    no central operator, no trust required. Closes tutorial-7 gap 3.

11. **[The Wallet: Petnames on the Blockchain](11-wallet.md)**
    The Agoric wallet is tutorial 1's three-layer model applied to a blockchain.
    Bob buys a CarAccess token from the marketplace, approves the offer through
    the wallet UI, and the wallet displays it under his chosen petname. The
    dapp-wallet bridge is attenuation and per-grantee wrappers at the financial
    layer.

Further tutorials will cover the Lit + Web Components PWA mobile wallet.

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
