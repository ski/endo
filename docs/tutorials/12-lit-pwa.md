---
title: 12-lit-pwa
group: Documents
category: Guides
---

# The Mobile Wallet: Lit, Web Components, and Offline Capabilities

The series started with a Car object in a terminal. It ends with Bob's phone, running Lit and Web Components, holding a CarAccess token offline, connecting to the Agoric chain over a VPN, and calling `start()` on a capability that lives on a blockchain.

The mental model is exactly the same as tutorial 1. The substrate has changed — daemon process → blockchain node → browser JavaScript runtime — but the pattern has not. Capabilities are unforgeable references. Pet names are local. Attenuation is wrapping. The `E()` call that fires across a Cloudflare Worker binding fires equally well across a WebSocket to an Agoric RPC endpoint.

This tutorial builds the PWA wallet. It is a progressive web app — installable on iOS and Android, works offline for reads, syncs when connected — built with Lit + Vite + Web Components and backed by the CRDT synced pet store from tutorial 2.

## The four structural correspondences

Before writing any code, observe what maps to what.

**Web component tag name = pet name.** `customElements.define('car-fob', CarFob)` registers a name that is meaningful to this app, in this context, chosen by the developer. It is not the on-chain identity of the CarAccess brand. It is a local nickname — exactly the pet name from tutorial 1.

**Private field = the closure from tutorial 3.** The `CarFob` element holds its CapTP reference in a private field (`#fob`). Nothing outside the class can read or modify it — not the parent page, not a sibling component, not a script injected by a rogue extension. The private field is the closure that protected the car reference inside the revocable forwarder.

**Shadow DOM = confinement boundary.** The shadow DOM isolates the element's styles and internal structure. A stylesheet in the parent page cannot reach inside. A script in the parent page cannot imperatively mutate the element's internals. This is not as strong as an SES compartment, but it is the closest Web Platform primitive to it, and it composes with SES.

**`start()` method on the element = attenuated interface.** The element only exposes `start()`. It does not expose the raw CapTP reference. It does not expose `unlock()` unless the element's author added it. The public interface is whatever the element declares — same attenuation logic as the valet key from tutorial 1.

## Project setup

```sh
npm create vite@latest bob-wallet -- --template lit
cd bob-wallet
npm install ses @endo/init @endo/far
npm install @agoric/react-components  # for wallet bridge types
npm run dev
```

## 1. Lockdown first, everything else second

Edit `src/main.js` — this file runs before any component code:

```js
// src/main.js

// Lockdown MUST run before any other import that relies on hardened objects.
// It freezes all JavaScript primordials: Array.prototype, Object.prototype,
// Date.now (returns NaN after lockdown), Math.random (throws), and so on.
// No supply-chain attack can add properties to shared prototypes after this.
import 'ses/lockdown';
lockdown({
  errorTaming: 'safe',
  overrideTaming: 'severe',
});

// Everything else loads after the perimeter is established.
import './components/car-fob.js';
import './components/wallet-panel.js';
import './app.js';
```

One import, one call, before anything else. Every Web Component, every Lit element, every third-party library loaded after this line runs inside a hardened environment. Prototype mutation fails. `Date.now()` returns `NaN`. `Math.random()` throws. The compartment boundary is the entire page.

## 2. The CarFob web component

```js
// src/components/car-fob.js
import { LitElement, html, css } from 'lit';
import { E } from '@endo/far';

class CarFob extends LitElement {
  // The CapTP reference lives here and nowhere else.
  // JavaScript private fields are not accessible outside the class body —
  // not via the DOM, not via Object.getOwnPropertyDescriptor,
  // not by any script in the parent page.
  #fob = null;
  #status = 'idle';

  static properties = {
    locator: { type: String },  // The on-chain locator for this fob
  };

  static styles = css`
    :host { display: block; padding: 1rem; border: 1px solid #ddd; border-radius: 8px; }
    button { background: #2a2a2a; color: white; border: none; padding: 0.5rem 1rem; border-radius: 4px; cursor: pointer; }
    .status { font-size: 0.8rem; color: #666; margin-top: 0.5rem; }
  `;

  // The element's public API: start the car.
  // The caller cannot bypass this to get the raw #fob reference.
  async start() {
    if (!this.#fob) {
      this.#status = 'not connected';
      this.requestUpdate();
      return;
    }
    this.#status = 'starting…';
    this.requestUpdate();
    try {
      const result = await E(this.#fob).start();
      this.#status = result;
    } catch (err) {
      this.#status = `error: ${err.message}`;
    }
    this.requestUpdate();
  }

  // Called by the wallet panel once it resolves the locator to a live ref.
  setFobReference(fob) {
    this.#fob = fob;
    this.#status = 'ready';
    this.requestUpdate();
  }

  render() {
    return html`
      <div>
        <strong>CarAccess fob</strong>
        <button @click=${() => this.start()}>Start engine</button>
        <div class="status">${this.#status}</div>
      </div>
    `;
  }
}

customElements.define('car-fob', CarFob);
```

The element is 50 lines. The important property: `#fob` is in a private field. If a rogue script in the page tries `document.querySelector('car-fob')['#fob']`, it gets `undefined`. The reference is as protected as a closure in the same-process model.

## 3. Connect to the chain and resolve the locator

The wallet panel component handles the chain connection and passes resolved references into the fob elements:

```js
// src/components/wallet-panel.js
import { LitElement, html } from 'lit';
import { E } from '@endo/far';
import { makeCapTP } from '@endo/captp';

class WalletPanel extends LitElement {
  #bootstrap = null;
  #purses = [];
  #ws = null;

  connectedCallback() {
    super.connectedCallback();
    this.#connect();
  }

  async #connect() {
    // Connect to the Agoric chain gateway (or your Endo daemon) over WebSocket.
    // In production, this goes through Tailscale — the phone and the node
    // are on the same VPN, so the WS endpoint is reachable without exposing
    // a public port.
    const ws = new WebSocket('ws://agoric-node.local:26657/websocket');
    this.#ws = ws;

    const { getBootstrap } = makeCapTP('mobile-wallet', {
      send: (msg) => ws.send(JSON.stringify(msg)),
    });

    ws.onmessage = (event) => {
      const msg = JSON.parse(event.data);
      // route incoming CapTP messages
    };

    this.#bootstrap = getBootstrap();

    // Resolve CarAccess fobs stored in IndexedDB (the offline replica)
    const storedFobs = await this.#loadOfflineFobs();
    for (const { elementId, locator } of storedFobs) {
      const element = this.renderRoot.querySelector(`#${elementId}`);
      if (element) {
        const fob = await E(this.#bootstrap).fetch(locator);
        element.setFobReference(fob);
      }
    }
  }

  async #loadOfflineFobs() {
    // The synced pet store from tutorial 2, stored in IndexedDB.
    // When online: merge from remote daemon.
    // When offline: read from the local CRDT replica.
    const db = await openIndexedDB('bob-wallet', 1, (db) => {
      db.createObjectStore('fobs', { keyPath: 'petName' });
    });
    return getAllFromStore(db, 'fobs');
  }

  render() {
    return html`
      <slot></slot>
    `;
  }
}

customElements.define('wallet-panel', WalletPanel);
```

## 4. The app shell

```html
<!-- index.html -->
<!doctype html>
<html>
  <head>
    <title>Bob's Wallet</title>
    <link rel="manifest" href="/manifest.json" />
    <script type="module" src="/src/main.js"></script>
  </head>
  <body>
    <wallet-panel>
      <car-fob id="alice-fob" locator="endo://..."></car-fob>
    </wallet-panel>
  </body>
</html>
```

The `locator` attribute carries the on-chain reference. The `wallet-panel` resolves it to a live CapTP object and passes it to the `car-fob` via `setFobReference()`. The element's private field holds the reference from that point on.

## 5. The offline layer: IndexedDB + the synced pet store

The synced pet store from tutorial 2 is a CRDT that two peers can merge without coordination. In the mobile context:

- **Online**: the phone connects to the Agoric node (or Endo daemon) over Tailscale, calls `mergeRemoteState()` to pick up any changes, and `acknowledgeRemoteClock()` to let the other side prune old tombstones.
- **Offline**: the phone reads from its local replica in IndexedDB. If Alice revoked a fob while the phone was offline, the tombstone will propagate the next time it reconnects.

```js
// src/sync.js — simplified synced pet store over IndexedDB

export async function syncWithRemote(localStore, remoteStoreRef) {
  // Merge remote state into local.
  const remoteState = await E(remoteStoreRef).getState();
  const remoteClock  = await E(remoteStoreRef).getLocalClock();
  await localStore.mergeRemoteState(remoteState, remoteClock);

  // Ack so the remote can prune tombstones.
  const localClock = localStore.getLocalClock();
  await E(remoteStoreRef).acknowledgeRemoteClock(localClock);

  // Persist local state to IndexedDB.
  await saveToIndexedDB('bob-wallet', 'fobs', localStore.getState());
}
```

When the service worker intercepts a navigation while offline, it serves the shell from its cache. The wallet panel loads, reads from IndexedDB, and resolves whatever locators are in the local synced store. Method calls to dead fobs (revoked by Alice while the phone was offline) will fail with "access revoked" when the phone next connects and the tombstone propagates. Until then, the phone does not know — it holds what it last synced.

This is not a bug. It is the intended behaviour of a system that trades consistency for availability. The fob might have been revoked. The phone cannot know until it connects. When it does, the CRDT merge delivers the truth.

## 6. Install to home screen

Add `manifest.json`:

```json
{
  "name": "Bob's Wallet",
  "short_name": "Wallet",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#1a1a1a",
  "theme_color": "#2a2a2a",
  "icons": [
    { "src": "/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

Register the service worker in `main.js`, after lockdown:

```js
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('/sw.js');
}
```

The service worker caches the shell (`index.html`, the compiled JS, the CSS) on install and serves it from cache when offline. Dynamic data — the on-chain locators, the CRDT state — lives in IndexedDB and is never served by the service worker.

## 7. Tailscale as the transport

Do not expose the Agoric node or the Endo daemon to the public internet. Install Tailscale on both the node (the machine running the chain) and the phone. The phone reaches the node at its Tailscale IP, which is only reachable from authenticated devices on your tailnet. The WebSocket endpoint never needs a public port, and the traffic is encrypted end-to-end.

For the Endo daemon from tutorial 2, the RUNBOOK equivalent for mobile is:

```sh
# On the machine running the daemon:
ENDO_ADDR=0.0.0.0:8920 \
  ENDO_GATEWAY_ALLOWED_CIDRS="100.64.0.0/10" \
  endo start
```

`100.64.0.0/10` is the Tailscale IP range. The phone connects to `ws://100.x.y.z:8920/` — only reachable through the VPN.

## The full circle

In tutorial 1, Alice created `my-car` by running `endo make car.js --name my-car`. The daemon assigned a formula ID. She called it by a pet name. The Car ran in a worker.

In tutorial 12, Bob's phone has a `<car-fob>` web component. The tag name is `car-fob` — his pet name for this capability, meaningful to this app, invisible to anyone else. The private field `#fob` holds the CapTP reference — the unforgeable identity, resolved from the on-chain locator the wallet gave him after he bought the token in tutorial 10. The reference points at a vat running on the Agoric chain — the live object, the thing that can actually start the engine.

Three layers. Pet name, reference, live object. Tutorial 1 to tutorial 12, the model did not change. The substrate evolved from a local daemon to a blockchain; the abstraction held.

That is what Endo is. That is what Agoric is. That is what the series was about.
