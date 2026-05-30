---
title: 10-marketplace
group: Documents
category: Guides
---

# The Listing Board: An On-Chain Capability Marketplace

Tutorial 9 built an atomic swap. Alice listed a CarAccess fob, Bob bought it, Zoe guaranteed the exchange. But there was a gap: Bob had to know Alice's contract address in advance. He could not browse a list of available fobs, compare prices, or discover that Alice's garage existed at all. There was no marketplace — only a private agreement dressed up in a smart contract.

This tutorial builds the marketplace. It is a single on-chain contract that acts as a listing board: sellers post offers, buyers browse them, and every match settles atomically through the same Zoe offer-safety guarantee from tutorial 9. Neither side trusts the other; both trust the contract.

## What you'll build

A `carAccessMarket` contract that lets multiple sellers list CarAccess fobs at different prices, exposes a public listing feed that any buyer can read, and routes each purchase through an atomic swap. Alice posts a listing. Bob browses it. Bob buys. Charlie posts a different listing at a different price. The board shows both. Either buyer can take either listing without knowing the other exists.

## What you'll need

- Tutorials 8 and 9 completed: you understand ERTP payments and the Zoe offer/seat/proposal model.
- A local Agoric chain running (`yarn start:docker`).
- About 40 minutes.

## 1. The marketplace contract

The listing board is a Map of active listings. Each listing holds an open seller seat — escrowing the seller's CarAccess fob — and the seller's proposal (how much IST they want). When a buyer takes a listing, the contract matches the buyer's seat against the stored seller seat and calls `atomicTransfer`.

```js
import { atomicTransfer } from '@agoric/zoe/src/contractSupport/atomicTransfer.js';
import { AmountMath } from '@agoric/ertp';

/**
 * carAccessMarket: a listing board for CarAccess tokens.
 *
 * Sellers call publicFacet.makeListingInvitation() to post a listing.
 * Buyers call publicFacet.getListings() to browse, then
 * publicFacet.makePurchaseInvitation(listingId) to buy.
 */
const start = (zcf) => {
  /** @type {Map<string, { sellerSeat: ZCFSeat, give: Amount, want: Amount }>} */
  const listings = new Map();
  let nextId = 0n;

  // Seller posts a listing: they give CarAccess, want Price.
  const handleSeller = (sellerSeat) => {
    const { give, want } = sellerSeat.getProposal();
    const listingId = String(nextId++);

    listings.set(listingId, {
      sellerSeat,
      give: give.CarAccess,
      want: want.Price,
    });

    // Return the listing id so the seller can share it or cancel later.
    return listingId;
  };

  // Buyer takes a listing: they give Price, want CarAccess.
  const makePurchaseHandler = (listingId) => (buyerSeat) => {
    const listing = listings.get(listingId);
    if (!listing) {
      buyerSeat.fail(new Error(`Listing ${listingId} not found`));
      return;
    }

    const { sellerSeat, give: sellerCarAccess, want: sellerPrice } = listing;
    const buyerProposal = buyerSeat.getProposal();

    // Verify the buyer is offering the right price and wanting the right asset.
    AmountMath.isGTE(buyerProposal.give.Price, sellerPrice) ||
      zcf.fail(`Insufficient payment: want ${sellerPrice.value} IST`);

    AmountMath.isGTE(buyerProposal.want.CarAccess, sellerCarAccess) ||
      zcf.fail(`CarAccess mismatch`);

    // Atomic swap: seller gets Price, buyer gets CarAccess.
    atomicTransfer(
      zcf,
      sellerSeat, buyerSeat,
      { CarAccess: sellerCarAccess },
      { Price: sellerPrice },
    );

    sellerSeat.exit();
    buyerSeat.exit();
    listings.delete(listingId);

    return 'purchase complete';
  };

  const publicFacet = harden({
    makeListingInvitation: () =>
      zcf.makeInvitation(handleSeller, 'post a CarAccess listing'),

    makePurchaseInvitation: (listingId) =>
      zcf.makeInvitation(
        makePurchaseHandler(listingId),
        `buy CarAccess listing ${listingId}`,
      ),

    // Returns an array of { listingId, give, want } for UI display.
    getListings: () =>
      harden(
        [...listings.entries()].map(([id, { give, want }]) => ({
          listingId: id,
          give,
          want,
        })),
      ),
  });

  return harden({ publicFacet });
};

export { start };
```

The listing board is about 70 lines. No `creatorFacet` — the market has no owner. Any seller can list, any buyer can buy, and the contract enforces the rules for both without needing to trust either.

## 2. Deploy the marketplace

Once your local chain is running:

```sh
agoric deploy scripts/deploy-market.js
```

The deploy script installs the contract and captures the `instance` and `publicFacet` handles.

```js
// deploy-market.js (simplified)
const installation = await E(zoe).install(marketBundle);

const { publicFacet, instance } = await E(zoe).startInstance(
  installation,
  {
    CarAccess: carAccessIssuer,
    Price:     istIssuer,
  },
);
```

Unlike tutorial 9's single-seller contract, this one takes no `privateArgs` and no `terms` — the market has no configuration. The issuers are the only setup.

## 3. Alice posts a listing

Alice mints a CarAccess fob (as in tutorial 8), then posts it on the board for 50 IST:

```js
const fobAmount   = AmountMath.make(carAccessBrand, harden(['fob-alice-001']));
const fobPayment  = carAccessMint.mintPayment(fobAmount);
const priceAmount = AmountMath.make(istBrand, 50n);

const listingInvitation = await E(publicFacet).makeListingInvitation();

const aliceSeat = await E(zoe).offer(
  listingInvitation,
  harden({ give: { CarAccess: fobAmount }, want: { Price: priceAmount } }),
  harden({ CarAccess: fobPayment }),
);

const listingId = await E(aliceSeat).getOfferResult();
console.log('Alice listed as:', listingId);
// '0'
```

Alice's fob is now escrowed by Zoe. The listing is visible to anyone who calls `getListings()`.

## 4. Charlie posts a second listing

Charlie mints her own fob and lists it at a different price — 35 IST:

```js
const charlieFobAmount   = AmountMath.make(carAccessBrand, harden(['fob-charlie-001']));
const charlieFobPayment  = carAccessMint.mintPayment(charlieFobAmount);
const charliePrice       = AmountMath.make(istBrand, 35n);

const charlieSeat = await E(zoe).offer(
  await E(publicFacet).makeListingInvitation(),
  harden({ give: { CarAccess: charlieFobAmount }, want: { Price: charliePrice } }),
  harden({ CarAccess: charlieFobPayment }),
);

const charlieListingId = await E(charlieSeat).getOfferResult();
console.log('Charlie listed as:', charlieListingId);
// '1'
```

## 5. Bob browses the board

Bob calls `getListings()` and sees both offers:

```js
const listings = await E(publicFacet).getListings();
console.log(listings);
// [
//   { listingId: '0', give: { brand: CarAccess, value: ['fob-alice-001'] },
//                     want: { brand: IST, value: 50n } },
//   { listingId: '1', give: { brand: CarAccess, value: ['fob-charlie-001'] },
//                     want: { brand: IST, value: 35n } },
// ]
```

Bob picks listing `'1'` (Charlie's cheaper fob):

```js
const bobIst = /* Bob's 35 IST payment */;
const purchaseAmount = AmountMath.make(carAccessBrand, harden(['fob-charlie-001']));

const buyerSeat = await E(zoe).offer(
  await E(publicFacet).makePurchaseInvitation('1'),
  harden({
    give: { Price: charliePrice },
    want: { CarAccess: purchaseAmount },
  }),
  harden({ Price: bobIst }),
);

console.log(await E(buyerSeat).getOfferResult());
// 'purchase complete'

const { CarAccess: bobFob } = await E(buyerSeat).getPayouts();
```

Bob has Charlie's fob. Charlie has 35 IST. Alice's listing (`'0'`) is still open — no one bought it, and the board now shows only one entry:

```js
console.log(await E(publicFacet).getListings());
// [ { listingId: '0', give: ..., want: 50n IST } ]
```

## 6. What this closes

Look back at the bridge chapter (tutorial 7). Three gaps remain after tutorial 9:

- **Trust is personal not contractual** — closed by Zoe (tutorial 9). The contract enforces the terms; Alice cannot keep the IST and withhold the fob.
- **Payment and access are not atomic** — closed by Zoe (tutorial 9). Both transfer or neither does.
- **The marketplace does not exist** — closed by this tutorial. Sellers post publicly, buyers browse, and both parties interact with the same board without knowing each other.

Two gaps remain:

- **The fob cannot be traded** — Bob bought Charlie's fob, but he cannot sell it on the board unless Charlie's original issuance was a transferable ERTP asset (which it was — an ERTP payment deposited into a purse and withdrawn is fully transferable). To resell, Bob would simply post a new listing with his own purse's withdrawal. The primitives are already there.
- **Revocation is Alice's word not the chain's** — the contract does not enforce revocation terms. A seller can cancel their listing by exiting their seat before a buyer takes it. Subscription-based time-limited access (auto-revoke after 30 days) requires a clock capability in the contract terms, which tutorial 11 will cover when we build the wallet and petname layer on top.

## 7. Composing marketplaces

The market contract has no owner. Anyone can deploy one. Multiple independent markets for the same asset type can coexist — buyers choose which market to search, just as they choose which DEX to use on Ethereum. A market aggregator contract could watch multiple boards and route buyers to the best price.

This is the composability of the object-capability model applied to market structure. The market is a capability (the `publicFacet`). Passing it to an aggregator gives the aggregator read and listing access without any special permissions or governance — just object reference, just capability.

## What's next

Tutorial 11 builds the Agoric wallet interface on top: how users see their CarAccess tokens as petnames in their wallet, how they approve a purchase through the wallet's offer UI rather than calling `E(zoe).offer()` directly, and how the wallet's local petname system maps to the on-chain capability identifiers — completing the circle back to tutorial 1's three-layer model.
