---
title: 09-zoe
group: Documents
category: Guides
---

# The Garage as a Smart Contract: Zoe

Tutorial 8 gave the car's access right a proper financial form — an ERTP payment with supply controlled by a mint, transferable between purses, and verifiable by an issuer. But it left two gaps open from the bridge chapter: payment and access are not yet atomic, and there is still no trustless marketplace. Both gaps come from the same root problem — Alice and Bob still have to trust each other to complete their half of any exchange.

Zoe closes both gaps. It is Agoric's smart contract host: a piece of infrastructure that holds both sides of a proposed exchange in escrow and releases them simultaneously, or releases nothing. Alice cannot take Bob's USDC and disappear. Bob cannot take Alice's fob without paying. Either the swap completes in full, or every asset goes back to its original owner. This is called **offer safety** — a guarantee enforced by the host, not by the parties.

This tutorial shows what the garage looks like as a Zoe contract, how offers work, and why the guarantee is structural rather than legal.

## What you will need

Zoe runs on the Agoric chain. There is no standalone runner. To follow along locally you will need:

```sh
# Node.js 18.18+, yarn, and Docker
yarn create @agoric/dapp car-shop
cd car-shop
yarn install
yarn start:docker    # starts a local Agoric chain in Docker
```

If you only want to read the contract code for now, you do not need Docker. The pattern is worth understanding even before you run it.

## The shape of a Zoe contract

A Zoe contract is a JavaScript module that exports a `start` function:

```js
const start = (zcf, privateArgs) => {
  // contract logic here
  return harden({
    creatorFacet,   // returned to whoever called startInstance
    publicFacet,    // accessible to anyone via E(zoe).getPublicFacet(instance)
  });
};
export { start };
```

`zcf` is the Zoe Contract Facet — the internal interface the contract uses to create invitations, inspect offers, and move assets. `privateArgs` carries anything the deployer needs to pass in without making public (like a reference to a mint).

Compare this to tutorial 6's garage:

```js
export const make = powers => {
  // garage logic here
  return Far('Garage', { provision, revoke, audit });
};
```

The shape is almost identical: a factory function receives its capabilities and returns a facet. The difference is that `zcf` is the capability broker — it is Zoe, not Alice's daemon — and the assets it manages are on-chain ERTP assets, not in-process forwarders.

## The car shop contract

This contract lets Alice list a CarAccess fob for a price, and Bob buy it atomically:

```js
import { atomicTransfer } from '@agoric/zoe/src/contractSupport/atomicTransfer.js';
import { AmountMath } from '@agoric/ertp';

/**
 * carAccessShop: Alice lists a CarAccess fob for sale at a fixed price.
 * Bob buys it by making a matching offer. Zoe ensures the swap is atomic.
 */
const start = (zcf) => {
  // The contract knows about two asset types: CarAccess and Price.
  // These are declared via issuerKeywordRecord in startInstance.

  // creatorFacet: returned to Alice (the shop owner).
  const creatorFacet = {
    // Alice makes an offer: she gives a fob, wants a price.
    makeSellerInvitation: () =>
      zcf.makeInvitation(
        sellerOfferHandler,
        'list CarAccess fob for sale',
      ),
  };

  // publicFacet: available to anyone.
  const publicFacet = harden({
    // Bob gets an invitation to buy.
    makeBuyerInvitation: () =>
      zcf.makeInvitation(buyerOfferHandler, 'buy CarAccess fob'),
  });

  let sellerSeat;

  // Alice's offer handler: she puts her fob into escrow.
  const sellerOfferHandler = (seat) => {
    sellerSeat = seat;
    return 'fob listed — waiting for buyer';
  };

  // Bob's offer handler: if his offer matches Alice's, swap atomically.
  const buyerOfferHandler = (buyerSeat) => {
    const sellerProposal = sellerSeat.getProposal();
    const buyerProposal  = buyerSeat.getProposal();

    // Verify the proposals match: what the seller wants = what the buyer gives,
    // and what the buyer wants = what the seller gives.
    AmountMath.isEqual(
      sellerProposal.want.Price,
      buyerProposal.give.Price,
    ) || zcf.fail('price mismatch');

    AmountMath.isEqual(
      sellerProposal.give.CarAccess,
      buyerProposal.want.CarAccess,
    ) || zcf.fail('fob mismatch');

    // Move assets atomically: seller gets the price, buyer gets the fob.
    atomicTransfer(
      zcf,
      sellerSeat, buyerSeat,
      { CarAccess: sellerProposal.give.CarAccess },
      { Price: sellerProposal.want.Price },
    );

    sellerSeat.exit();
    buyerSeat.exit();

    return 'swap complete';
  };

  return harden({ creatorFacet, publicFacet });
};

export { start };
```

The contract is about 60 lines. Everything else is Zoe doing the work: holding the assets, enforcing offer safety, and distributing payouts.

## Deploying and using the contract

Once your local chain is running:

```sh
# Bundle and deploy the contract
agoric deploy scripts/deploy.js
```

The deploy script calls `E(zoe).install(bundle)` to register the contract code, then `E(zoe).startInstance(installation, issuerKeywordRecord, terms)` to create a running instance.

```js
// deploy.js (simplified)
const installation = await E(zoe).install(carShopBundle);

const { creatorFacet, publicFacet, instance } = await E(zoe).startInstance(
  installation,
  {
    CarAccess: carAccessIssuer,  // issuer from the mint Alice created
    Price:     istIssuer,        // IST — Agoric's stable token
  },
);
```

The keywords `CarAccess` and `Price` map the contract's internal names to the real issuers on the chain.

## Alice lists her fob

Alice mints a CarAccess payment (from tutorial 8), then makes a seller offer:

```js
const fobAmount  = AmountMath.make(carAccessBrand, harden(['fob-001']));
const fobPayment = carAccessMint.mintPayment(fobAmount);
const priceAmount = AmountMath.make(istBrand, 50n);

const sellerInvitation = await E(creatorFacet).makeSellerInvitation();

const sellerSeat = await E(zoe).offer(
  sellerInvitation,
  harden({ give: { CarAccess: fobAmount }, want: { Price: priceAmount } }),
  harden({ CarAccess: fobPayment }),
);

console.log(await E(sellerSeat).getOfferResult());
// 'fob listed — waiting for buyer'
```

Alice's fob is now held in escrow by Zoe. She cannot retrieve it except by having her offer cancelled — and the contract does not provide a cancel path, which is intentional. She listed it; the contract holds her to it.

## Bob buys

Bob has 50 IST (Agoric's stable coin). He makes a buyer offer:

```js
const buyerInvitation = await E(publicFacet).makeBuyerInvitation();

const buyerSeat = await E(zoe).offer(
  buyerInvitation,
  harden({ give: { Price: priceAmount }, want: { CarAccess: fobAmount } }),
  harden({ Price: bobIstPayment }),
);

console.log(await E(buyerSeat).getOfferResult());
// 'swap complete'

// Bob's payout contains the fob.
const { CarAccess: bobFobPayment } = await E(buyerSeat).getPayouts();

// Alice's payout contains the price.
const { Price: alicePricePayment } = await E(sellerSeat).getPayouts();
```

The swap happened atomically. Alice received the IST. Bob received the fob. Neither could cheat the other — there was no moment where one party had both assets.

## The offer safety guarantee

The critical property is what happens when the proposals do not match. If Bob offers 40 IST instead of 50:

- The `buyerOfferHandler` calls `zcf.fail('price mismatch')`
- Zoe returns Bob's 40 IST to him immediately
- Alice's fob stays in escrow — her offer is still live, waiting for a matching buyer
- No assets were lost, no partial state was committed

This is the gap from tutorial 7 closed. In the tutorial-6 garage, "payment and access are not atomic" meant Alice could provision a fob before receiving payment, or Bob could pay without getting the fob. In Zoe, the contract never sees the assets until both sides are in escrow. The swap either happens or it does not.

## The map: Zoe ↔ tutorials 6 and 8

| Zoe | Tutorial concept |
|---|---|
| `E(zoe).install(bundle)` | `endo make garage.js` |
| `E(zoe).startInstance(installation, ...)` | Instantiating the garage with a specific car |
| `creatorFacet` | The garage's private interface (Alice's `audit()`, `revoke()`) |
| `publicFacet` | The garage's public interface (anyone can call `makeBuyerInvitation`) |
| Invitation | A single-use capability to participate in one specific offer |
| Proposal (`give`, `want`) | The terms of an offer — Bob's `request` from tutorial 4, but structured |
| `atomicTransfer` | Tutorial 3's `revocable forwarder` — but now it swaps two assets at once |
| `seat.exit()` | Tutorial 3's `revoker.revoke()` — closes out the offer |
| Offer safety | The guarantee the garage could not provide — neither party can be cheated |

The code is structurally identical to what you have been writing. The difference is the guarantee: instead of "Alice pinky-swears she will provision Bob's fob if he pays," the contract provides a mathematical guarantee enforced by Zoe.

## What's next

The garage is now a smart contract. The atomic swap is working. What it is not yet is a marketplace — Alice's single listing is not discoverable, and Bob had to know her contract's address in advance. Tutorial 10 adds a marketplace contract: a listing board where multiple sellers can post offers, buyers can browse and accept, and the whole thing is on-chain and trustless.
