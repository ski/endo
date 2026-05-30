---
title: 11-wallet
group: Documents
category: Guides
---

# The Wallet: Petnames on the Blockchain

In tutorial 1, Alice's daemon tracked three layers: a pet name she chose, a formula ID that was the stable unforgeable reference, and the live object behind it. She called the Car `my-car`. The daemon held the formula. The Car ran in a worker.

The Agoric wallet is the same three-layer model, applied to a blockchain.

When Bob buys a CarAccess token from the marketplace in tutorial 10, he does not receive a formula file. He receives an ERTP payment. He deposits it into a purse. His wallet displays it under whatever petname he chooses — "My CarAccess", "Alice's Fleet Fob", anything. The petname is local to him, chosen by him, invisible to Alice. The brand is the on-chain ERTP identity, stable and unforgeable. The balance in the purse is the live capability.

This tutorial shows what that looks like from the user's side: a React dapp that connects to the wallet, queries the marketplace listing board, and routes Bob's purchase through the wallet's offer-approval UI so he can see exactly what he is giving and what he will receive before he signs.

## What you'll build

A minimal marketplace dapp. It reads active listings from the on-chain contract (tutorial 10), shows them in a React component, and when Bob clicks "Buy" it calls the Agoric wallet to present the offer for approval. Bob sees the proposal — give 35 IST, want CarAccess fob-charlie-001 — and approves or cancels. If he approves, Zoe executes the atomic swap and his wallet balance updates.

## What you'll need

- Tutorial 10 completed: the marketplace contract deployed on a local Agoric chain.
- Node.js 20 and a React project scaffolded with Vite.
- About 35 minutes.

## 1. The project structure

```sh
npm create vite@latest car-market-dapp -- --template react
cd car-market-dapp
npm install @agoric/react-components @agoric/ertp
npm run dev
```

`@agoric/react-components` is the official Agoric dapp library. It provides `AgoricProvider` (the root context) and the `useAgoric()` hook, which returns the user's wallet connection, purse balances, and a `makeOffer` function.

## 2. Wrap the app in AgoricProvider

```jsx
// main.jsx
import { AgoricProvider } from '@agoric/react-components';
import App from './App';

const CHAIN_CONFIG = {
  chainName: 'agoriclocal',
  rpc: 'http://localhost:26657',
  api: 'http://localhost:1317',
};

export default function Root() {
  return (
    <AgoricProvider chains={[CHAIN_CONFIG]}>
      <App />
    </AgoricProvider>
  );
}
```

`AgoricProvider` establishes a connection to the chain's VStorage, polls for wallet state updates, and makes everything available via the `useAgoric()` hook.

## 3. Show the wallet balance

```jsx
// WalletPanel.jsx
import { useAgoric } from '@agoric/react-components';

export function WalletPanel() {
  const { purses, walletConnection } = useAgoric();

  if (!walletConnection) {
    return <button onClick={walletConnection?.connect}>Connect Wallet</button>;
  }

  return (
    <div>
      <h2>My Wallet</h2>
      {purses?.map(({ brandPetname, displayInfo, currentAmount }) => (
        <div key={brandPetname}>
          <strong>{brandPetname}</strong>:{' '}
          {String(currentAmount.value)} {displayInfo?.assetKind}
        </div>
      ))}
    </div>
  );
}
```

`purses` is an array of objects where `brandPetname` is the user's chosen name for that asset type — "IST", "My CarAccess", or whatever they named it when they first received the token. The wallet does not show raw brand IDs. It shows what the user named the thing.

This is the three-layer model made visible. The `brandPetname` ("My CarAccess") is Bob's pet name. Behind it is the brand (the on-chain ERTP identity). Behind the brand is the actual balance in the purse.

## 4. Read listings from the marketplace

The marketplace contract's `publicFacet.getListings()` is available on-chain. From a dapp, you reach it via the contract instance that was recorded in the chain's VStorage at deploy time:

```jsx
// useListings.js
import { useAgoric } from '@agoric/react-components';
import { useEffect, useState } from 'react';
import { E } from '@agoric/eventual-send';

export function useListings(publicFacet) {
  const [listings, setListings] = useState([]);

  useEffect(() => {
    if (!publicFacet) return;

    const poll = async () => {
      const result = await E(publicFacet).getListings();
      setListings(result);
    };

    poll();
    const id = setInterval(poll, 5000);
    return () => clearInterval(id);
  }, [publicFacet]);

  return listings;
}
```

`E(publicFacet).getListings()` is the same eventual-send from tutorial 1 — `E()` abstracts the distance. Here the distance is the blockchain. The method executes in the contract's vat; the result comes back as a promise.

## 5. Present a listing for purchase

```jsx
// Listing.jsx
import { useAgoric } from '@agoric/react-components';
import { AmountMath } from '@agoric/ertp';

export function Listing({ listing, carAccessBrand, istBrand, marketInstance }) {
  const { makeOffer } = useAgoric();

  const handleBuy = () => {
    const purchaseInvitation = E(publicFacet).makePurchaseInvitation(listing.listingId);

    makeOffer(
      // The invitation to participate in this specific trade
      { source: 'contract', instance: marketInstance, publicInvitationMaker: 'makePurchaseInvitation', invitationArgs: [listing.listingId] },

      // The proposal: what Bob gives and wants
      {
        give: {
          Price: AmountMath.make(istBrand, listing.want.value),
        },
        want: {
          CarAccess: listing.give,
        },
      },

      // No extra offer args needed for this contract
      undefined,

      // Status callback
      (status) => {
        if (status.numWantsSatisfied === 1) {
          console.log('Purchase complete!');
        }
        if (status.error) {
          console.error('Purchase failed:', status.error);
        }
      },
    );
  };

  return (
    <div>
      <span>{String(listing.give.value[0])} — {String(listing.want.value)} IST</span>
      <button onClick={handleBuy}>Buy</button>
    </div>
  );
}
```

When Bob clicks "Buy", `makeOffer` routes the proposal through his wallet. The wallet presents a confirmation screen showing:

```
You are giving:  35 IST
You will receive: 1 CarAccess (fob-charlie-001)
```

Bob approves or cancels. If he approves, the wallet creates the payment from his IST purse, calls `E(zoe).offer(invitation, proposal, payment)` on his behalf, and monitors the outcome. Bob never writes a single line of contract interaction code. The dapp just says "here is what I want to trade and why"; the wallet handles the rest.

**Sidebar — why the wallet, not the dapp?** A malicious dapp could try to substitute a different contract, a different amount, or a different asset before calling `E(zoe).offer()`. The wallet is the user's agent — it is the entity that actually holds the IST purse and creates the payment. By routing through the wallet, the user sees exactly what will be submitted to the chain before it happens. The wallet's bridge facet is attenuated: the dapp can propose an offer, but it cannot execute one on the user's behalf without explicit approval. This is tutorial 1's valet key pattern, applied to financial transactions.

## 6. After the purchase: petnames on a blockchain

After Bob buys the fob, his wallet shows a new entry in his purse list. If he has never held CarAccess tokens before, the wallet will prompt him to add the issuer — and to give it a petname.

Bob types "Alice's Fleet Fob". From that moment:

- "Alice's Fleet Fob" is Bob's pet name — local to him, invisible to anyone else, stored in his wallet.
- The CarAccess brand is the on-chain identity — the same for every holder of CarAccess tokens, unforgeable, stable.
- The balance of 1 fob-charlie-001 in his purse is the live capability — what he can actually spend or transfer.

Alice's daemon from tutorial 1 called this a pet store. The Agoric wallet calls it a petname system. The mechanics are identical: a local map of names you chose, pointing at stable references you did not choose, backed by live objects that carry the actual authority.

The whole series has been building to this. The `my-car` pet name from tutorial 1 is `Alice's Fleet Fob` in tutorial 11. The formula ID is the brand. The worker process is the chain. The eventual send is `E()` whether the target is in the next process or the next block.

## 7. What the wallet protects Bob from

- **Asset substitution.** The wallet checks that the brand in the proposal matches the brand in the actual payment. A dapp cannot swap "give 35 IST" for "give 35 of something worthless" at execution time.
- **Offer bait-and-switch.** The invitation is verified against the contract instance the dapp specified. A dapp cannot redirect Bob's IST to a different contract.
- **Partial execution.** Zoe's offer safety guarantee (tutorial 9) means Bob either gets the CarAccess token or gets his 35 IST back. The wallet monitors the outcome and surfaces it.
- **Unauthorized spending.** The wallet holds the IST purse. The dapp has no direct access to it. The dapp can only propose; the user must approve.

This is attenuation (tutorial 1), per-grantee wrappers (tutorial 3), and offer safety (tutorial 9) composing at the wallet layer. The wallet is a capability broker — exactly like the garage from tutorial 6, but for the user's financial assets.

## What's next

Tutorial 12 puts all of this in a browser on Bob's phone. SES runs in a PWA (progressive web app) built with Lit + Web Components — the same stack as the Autopoet frontend. The wallet connection works the same way over the phone's browser. The synced pet store from tutorial 2 gives the PWA an offline replica of Bob's capability namespace. Revocations propagate when the phone reconnects. The capability model works at every scale, from a single object in a single process to a smartphone connected to a blockchain.
