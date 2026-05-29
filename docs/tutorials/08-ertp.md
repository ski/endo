---
title: 08-ertp
group: Documents
category: Guides
---

# The Car as a Digital Asset: ERTP

Tutorial 7 named five gaps in the garage. This tutorial closes the first three: trust is personal not contractual, payment and access are not atomic, and the fob cannot be traded. All three gaps trace back to the same missing piece — the car's access right is an object in Alice's daemon, but it is not an asset. It has no intrinsic value, no verifiable supply, and no way to transfer without relying on Alice to cooperate.

ERTP — the Electronic Rights Transfer Protocol — is Agoric's answer. It is a small set of JavaScript primitives that turn any right into a tradeable asset, with supply controlled by a mint that cannot be forged, and a transfer model that does not require trusting the other party.

This tutorial introduces ERTP locally, in plain Node.js, with no blockchain. You will build a CarAccess mint, issue a payment to Bob, let Bob transfer it to Charlie, and burn a revoked payment. Every concept maps to something you already know from tutorials 1 through 3.

## The four primitives

ERTP has four core objects. Each one has a counterpart in the tutorial series.

**Mint** — the exclusive authority to create new supply of an asset. Only the holder of the mint can issue new units. No one else can forge a payment. This is Alice from tutorial 3: she is the only one who can call `provision()` on the garage. The mint IS the minting authority — if you have the mint object, you can mint; if you do not, you cannot. Object identity is the only access control.

**Issuer** — the public-facing side of the mint. It can validate payments (tell you whether a payment is genuine and unspent), create empty purses, and burn assets. It cannot create new supply. This is the attenuated valet key from tutorial 1: it exposes a subset of the mint's interface — the subset that is safe to share publicly.

**Brand** — the identity of an asset type. It is a stable, unforgeable object that lets anyone check "is this payment for CarAccess or for something else?" It is a pet name for an asset type — globally meaningful, not just local like an Endo pet name, because it is an object reference that travels with every payment.

**Payment** — a one-use object that carries some amount of an asset. Like a physical key: it exists, it can be handed to someone, and once it is deposited into a purse it is consumed and cannot be used again. This is the revocable forwarder from tutorial 3, but with two new properties: it is transferable (Bob can hand it to Charlie without Alice's involvement) and it is finite (the issuer can verify that it has not already been spent).

A **Purse** holds a persistent balance. Bob keeps his CarAccess balance in a purse. He can deposit payments into it and withdraw payments from it. This is Bob's pet store — a place to keep what he has been given.

## Install

```sh
mkdir endo-ertp-tutorial
cd endo-ertp-tutorial
npm init -y
```

Add `"type": "module"` to `package.json`. Then:

```sh
npm install @agoric/ertp @endo/init
```

`@agoric/ertp` pulls in the Agoric store and math libraries as dependencies. The install is heavier than tutorials 1–6 but straightforward.

## 1. Create the mint

Create `ertp-demo.js`:

```js
import '@endo/init';
import { makeIssuerKit, AssetKind, AmountMath } from '@agoric/ertp';

// Alice creates a mint for car access rights.
// AssetKind.SET means non-fungible: values are sets of named tokens,
// not interchangeable numbers. One 'fob-001' is not the same as 'fob-002'.
const { mint, issuer, brand } = makeIssuerKit('CarAccess', AssetKind.SET);

console.log('Mint created for brand:', brand.getAllegedName());
// CarAccess
```

Run it:

```sh
node ertp-demo.js
```

Three objects came out: `mint`, `issuer`, and `brand`. Alice holds all three right now. She will keep the `mint` private, share the `issuer` publicly, and let the `brand` travel freely — it is just an identity, not authority.

**The counterpart.** In tutorial 3, the garage held the car and minted forwarders. Here, `mint` IS the garage. The difference: the garage was Alice's custom code; the mint is a standard ERTP primitive with a well-understood interface that anyone can audit.

## 2. Mint a payment for Bob

Alice creates an amount — a description of "one CarAccess token named fob-001" — and mints a payment carrying that amount:

```js
// An amount describes what the payment will carry.
// For SET assets, the value is an array of named strings.
const fobAmount = AmountMath.make(brand, harden(['fob-001']));

// Mint creates a payment carrying that amount.
// This is synchronous — the payment object exists immediately.
const bobPayment = mint.mintPayment(fobAmount);

console.log('Minted payment:', issuer.getAmountOf(bobPayment));
// { brand: CarAccess, value: ['fob-001'] }
```

The payment exists. It has not been given to Bob yet — Alice is still holding the reference. The issuer can inspect it without consuming it: `issuer.getAmountOf(payment)` reads the amount and leaves the payment intact.

**The counterpart.** In tutorial 3, Alice called `E(garage).provision('bob', 'full')` to get a forwarder. Here she calls `mint.mintPayment(amount)` to get a payment. The difference: the forwarder delegated to the Car object; the payment represents the RIGHT to access without embedding the Car directly. The right and the mechanism are separated.

## 3. Bob puts it in a purse

Bob creates a purse (an empty balance holder for CarAccess tokens) and deposits Alice's payment:

```js
// Bob creates an empty purse for CarAccess assets.
const bobPurse = issuer.makeEmptyPurse();

// Deposit the payment into the purse.
// The payment is CONSUMED — it cannot be deposited again.
const depositedAmount = bobPurse.deposit(bobPayment);
console.log('Bob deposited:', depositedAmount);
// { brand: CarAccess, value: ['fob-001'] }

// Check the purse balance.
console.log('Bob balance:', bobPurse.getCurrentAmount());
// { brand: CarAccess, value: ['fob-001'] }

// The payment is now spent — trying to check it throws.
try {
  issuer.getAmountOf(bobPayment);
} catch (err) {
  console.log('Payment spent:', err.message);
}
```

The payment is consumed on deposit. There is no way to double-spend it. The issuer tracks which payments are live and which are spent — this is enforced by object identity, not by a ledger.

**The counterpart.** In tutorial 1, Bob's pet store held the valet key by name. Here, Bob's purse holds his CarAccess balance. The difference: the purse can hold multiple units (if Alice mints more fobs) and it knows the total amount.

## 4. Bob withdraws to transfer to Charlie

Bob wants to give Charlie access. He withdraws a payment from his purse and hands it to Charlie:

```js
// Bob withdraws the fob from his purse (creating a new payment).
const charliesAmount = AmountMath.make(brand, harden(['fob-001']));
const charliesPayment = bobPurse.withdraw(charliesAmount);

// Bob's purse is now empty.
console.log('Bob balance after withdrawal:', bobPurse.getCurrentAmount());
// { brand: CarAccess, value: [] }

// Charlie deposits it into her own purse.
const charliePurse = issuer.makeEmptyPurse();
charliePurse.deposit(charliesPayment);

console.log('Charlie balance:', charliePurse.getCurrentAmount());
// { brand: CarAccess, value: ['fob-001'] }
```

Bob transferred his access right to Charlie. Alice was not involved. No message to Alice's daemon, no `endo send`, no approval workflow. The payment traveled from Bob's purse to Charlie's.

This is the capability that did not exist in tutorial 3. There, Bob could pass a forwarder reference to Charlie (delegation), but Alice could revoke the whole thing at once. Here, Bob has performed a real transfer: the asset left his purse and entered Charlie's. He no longer has it.

**The counterpart.** This is tutorial 2's "delegation is unstoppable" — but now it is also transfer. Bob did not copy the fob; he moved it. His balance went to zero.

## 5. Alice burns a revoked payment

If Alice needs to destroy an unspent payment — say, she issued one by mistake — she can burn it:

```js
// Alice mints a second payment that she later decides to destroy.
const revokedFobAmount = AmountMath.make(brand, harden(['fob-002']));
const revokedPayment = mint.mintPayment(revokedFobAmount);

// Burn destroys the payment permanently.
const burnedAmount = issuer.burn(revokedPayment);
console.log('Burned:', burnedAmount);
// { brand: CarAccess, value: ['fob-002'] }

// The payment is now unspendable.
try {
  issuer.getAmountOf(revokedPayment);
} catch (err) {
  console.log('Payment burned:', err.message);
}
```

Burning is Alice's equivalent of tutorial 3's `revoke()`. The difference: `revoke()` flipped a boolean in a closure — it affected the forwarder but not the underlying Car. `burn()` destroys the asset entirely: the right is gone, not just the forwarder to it.

## 6. What ERTP cannot do yet (and what Zoe adds)

ERTP gives you trustworthy supply, transferability, and a burn mechanism. What it does not give you is atomicity between two parties. If Alice wants to exchange a CarAccess payment for Bob's USDC, there is still a sequencing problem: Alice sends first, or Bob sends first, and one of them has to trust the other to complete their half.

That is what Zoe adds — an escrow protocol that holds both sides of an exchange and releases them atomically. Neither party can be cheated. Tutorial 9 builds the garage as a Zoe contract and makes the car-for-payment exchange trustless.

## The map: ERTP ↔ tutorials 1–3

**Mint** — The grantor from tutorial 3: exclusive authority to issue. If you hold the mint, you can create new supply. If you don't, you can't.

**Issuer** — The attenuated mint: public interface, no issuance power. Equivalent to the valet key from tutorial 1 — a subset of the mint's authority, safe to share.

**Brand** — Asset identity: unforgeable, travels with every payment. A stable object reference that lets anyone check "is this payment for CarAccess?"

**Payment** — The per-grantee forwarder from tutorial 3, but now transferable and one-use. Once deposited it is consumed.

**Purse** — Bob's namespace from tutorial 1, but balance-aware. Holds persistent supply across multiple deposits and withdrawals.

**`mint.mintPayment`** — equivalent to `E(garage).provision('bob', 'full')`.

**`purse.deposit`** — equivalent to `endo adopt`.

**`purse.withdraw`** — no tutorial 1–3 equivalent. This is the new capability: extracting an asset for transfer without Alice's involvement.

**`issuer.burn`** — equivalent to `E(garage).revoke('bob')`. Destroys the asset permanently rather than just flipping a flag.

The concepts are the same. What ERTP adds is: a standard interface anyone can audit, financial-grade supply control, and the transfer semantics that delegation lacked.

## What's next

Tutorial 9 wraps the garage in a Zoe smart contract. Alice lists her CarAccess tokens with a price. Bob makes an offer — his USDC for her fob — and Zoe ensures the exchange is atomic: either both assets transfer or neither does. The garage becomes a market.
