---
title: 07-toward-agoric
group: Documents
category: Guides
---

# What the Garage Is Missing: Toward a Blockchain Capability Marketplace

By the end of tutorial 6 you have a working capability broker. Alice runs a garage. She provisions per-grantee forwarders to plugins. She can revoke any plugin's access in a single call. She can emergency-stop everything at once. The object-capability model is working: the policy is in the object graph, not in a database, and it composes cleanly.

But look at the garage honestly and you will notice five things it does not do.

## 1. Trust is personal, not contractual

When Bob receives a start fob from Alice's garage, he is trusting Alice. Not the garage code — Alice. She wrote the garage. She controls the daemon it runs on. She can stop the daemon at midnight, wipe the formula files, and Bob's fob dies with no recourse. Nothing in the capability model prevents this.

For Alice giving access to a family member or a trusted colleague, this is fine. For Alice running a commercial service — charging Bob a monthly fee for guaranteed access to her fleet of cars — it is not fine. Bob needs to know that Alice's ability to revoke is governed by a rule he agreed to, not by Alice's mood.

This is what smart contracts exist for. A smart contract is code that runs on a blockchain — a machine that neither Alice nor Bob controls, that enforces the terms both parties signed up for, and that cannot be stopped by one party unilaterally. The garage becomes a contract. Alice's discretionary revocation becomes a term: "access expires after 30 days unless payment is received." The rule is enforced by the chain, not by Alice's goodwill.

## 2. Payment and access are not atomic

Right now Alice provisions a fob and separately collects payment through some other system. These two actions are not connected. Alice could take payment and forget to provision. Bob could receive the fob and dispute the charge. There is no way to make "Bob gets the fob if and only if Alice receives payment" true without a trusted third party holding the transaction.

ERTP — the Electronic Rights Transfer Protocol, Agoric's financial primitive layer — solves this. In an ERTP-based system, access is a digital asset (a "payment"), and the exchange is atomic: either both sides complete or neither does. Bob offers his USDC; Alice offers a fob; the exchange happens or it does not. No trusted intermediary. No credit card chargeback. No "Alice forgot to send the key."

## 3. The marketplace does not exist

Bob wants access to a car. He does not know Alice. How does he find her garage? Right now the answer is: out of band. Alice posts on a website, Bob emails her, they negotiate. The capability system has nothing to say about discovery.

A marketplace contract changes this. Alice lists her fleet with a price and a duration. Bob searches the marketplace, finds Alice's listing, and purchases a fob directly — all on-chain, all trustless. The marketplace is itself a capability broker: it holds Alice's offer, Bob's payment, and releases each to the other party when the conditions are met.

## 4. The fob cannot be traded

Bob received a start fob for Alice's car. Can he sell it to Charlie? Right now, no — or rather, he can give it away (delegation is unstoppable, as tutorial 2 explains), but he cannot sell it in a way that is atomic and trustless. And Alice might not want it transferred at all — she provisioned the fob specifically for Bob, and her revocation strategy assumes Bob is the only holder.

ERTP has a native concept of transferability. Some assets are fungible and freely tradeable (tokens). Others are non-fungible and represent a specific right (an access ticket, a subscription, a permit). The designer of the asset chooses the rules. If Alice mints Bob's access as a non-transferable ERTP payment, it cannot move to Charlie. If she mints it as transferable, Bob can sell it on a secondary market and the buyer gets Alice's real fob — because the fob IS the asset. There is nothing to transfer separately.

## 5. Revocation is Alice's word, not the chain's

Related to point 1 but worth stating separately: in tutorial 3, revocation is Alice flipping a boolean in a closure. It works perfectly in the single-daemon world. In a multi-party world with real money at stake, "trust Alice's closure" is not good enough.

On Agoric, revocation can be governed by on-chain rules. Bob's access expires at a timestamp encoded in a smart contract. The contract itself calls the garage's revoke method when the timestamp passes — not Alice, not a cron job on Alice's server. The revocation is auditable, predictable, and enforceable by anyone who reads the contract source code before paying.

## The questions tutorials 8 and beyond will answer

Each of these gaps points at a specific part of Agoric's stack. Here is the map:

**Tutorial 8 — The Car as a Digital Asset.** What does it mean to represent access rights as ERTP tokens? How do you mint a non-fungible "start fob" that carries its own revocation terms? What is the relationship between an ERTP payment and the capability it represents?

**Tutorial 9 — The Garage as a Zoe Contract.** How do you port the garage from tutorial 6 to a Zoe smart contract? What does Alice's `provision()` method look like when it is an on-chain offer? What does Bob's side of the transaction look like? How do the terms get enforced without Alice being online?

**Tutorial 10 — The Marketplace.** How do you build a contract that connects buyers and sellers of capability-backed assets? How does a "list your car fleet" offer work? How does discovery happen? What prevents the marketplace itself from being a single point of failure?

**Tutorial 11 — The Wallet and Petnames On-Chain.** How does Bob see Alice's contract in his wallet? How does the wallet's petname system map to the three-layer model from tutorial 1? What does approving a transaction look like — and why is it just `endo resolve` with a UI?

**Tutorial 12 — The Mobile Wallet.** Now the mobile question makes sense. Bob's phone holds petnames for on-chain capabilities. It connects to an Agoric node via the chain's RPC endpoint. SES runs in a browser WebView (PWA or Capacitor). The synced pet store is the wallet's local state replica. Revocation is enforced by the chain, not by Alice's daemon.

## What Agoric is, from here

If you have read tutorials 1 through 6, you already understand Agoric's programming model. The differences are:

- The daemon is a blockchain (Agoric's **SwingSet** kernel replays a deterministic log instead of reading formula files from disk)
- The worker is a **vat** — same single-threaded isolation, different word
- `endo make` is `E(zoe).install(bundle)` — same bundling, same compartment
- The pet store is the **wallet's petname system** — exact same three layers
- The garage is **Zoe** — same capability broker pattern
- The forwarder is an **ERTP payment** — same attenuation, now financially meaningful
- CapTP is the Agoric inter-vat protocol — same wire format, now securing financial transactions

The code uses the same `E()`, `Far`, `harden`, and `makeExo` you have been using all session. The mental model is identical. The only thing that changed is what is underneath: a blockchain that neither party controls, enforcing rules that both parties read before signing.

That is what tutorial 8 will show you.
