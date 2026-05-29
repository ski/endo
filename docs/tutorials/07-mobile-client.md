---
title: 07-mobile-client
group: Documents
category: Guides
---

# Endo on Mobile: Capabilities Across the Wire, Offline

A question that comes up naturally after building the smart garage: can a phone be a peer in a capability system? Can Bob's mobile app hold a reference to Alice's Car, call `E(fob).start()` from a React Native screen, and have the method execute on Alice's daemon across the network?

The short answer is yes, and less work is required than you might expect. This tutorial explains why, shows the gateway connection that makes it work, and introduces the piece that makes mobile capability systems viable in the real world: an offline-capable local replica of the capability namespace, backed by the CRDT synced pet store from [tutorial 2](https://suhail.ski/posts/mailing-bob-the-start-fob-capabilities-across-the-wire).

## The architecture

The Endo daemon runs a unified HTTP/WebSocket gateway. Any client that can open a WebSocket and speak the CapTP binary protocol can hold capability references and call methods on them. The daemon does not care whether the client is another daemon, a browser, or a mobile app — the protocol is the same.

```
[Phone — React Native]
  @endo/init (SES/lockdown)
  @endo/far  (E, Far)
  @endo/captp (CapTP over WebSocket)
       |
       | WebSocket
       |
[Desktop / Server]
  Endo daemon + gateway (ENDO_ADDR=0.0.0.0:8920)
  Capability formulas live here
  my-car, garage, etc.
```

The phone does not run the daemon. It does not spawn worker processes or manage formula files. It is a CapTP client — it holds references to remote capabilities and calls methods on them via eventual send.

## What you'll need

- Tutorial 6 setup: the garage installed on your daemon, `assistant-fob` or similar in your namespace.
- Node.js 20, for running the client-side demo in this tutorial.
- For the React Native section: familiarity with Expo or React Native CLI.
- About 40 minutes.

## 1. Open the daemon gateway to the network

By default the gateway only accepts connections from localhost. To accept from a phone on the same WiFi network, bind to all interfaces and whitelist your local network:

```sh
ENDO_ADDR=0.0.0.0:8920 \
  ENDO_GATEWAY_ALLOWED_CIDRS="192.168.0.0/16,10.0.0.0/8,100.64.0.0/10" \
  endo start
```

`192.168.0.0/16` covers typical home WiFi. `10.0.0.0/8` covers most corporate LANs. `100.64.0.0/10` covers Tailscale — recommended for production: run the daemon and phone on the same Tailscale network and you get encrypted transport for free.

Verify the gateway is up:

```sh
curl http://localhost:8920/
# Endo Gateway
```

Find your machine's local IP address (e.g. `192.168.1.42`). The phone will connect to `ws://192.168.1.42:8920/`.

> **Security note.** These options control which IP addresses can open a WebSocket connection. They do not add authentication or encryption. For anything beyond local development, put the daemon behind a VPN (Tailscale is the simplest) so only authorised devices can reach the port at all.

## 2. Pre-provision a capability for the mobile client

Before the phone can do anything useful, Alice needs to provision it a capability via the garage. From Alice's terminal, create a full grant for the mobile app:

```sh
endo eval 'E(g).provision("mobile-app", "full")' g:garage --name mobile-fob
```

Get the formula ID of the forwarder — this is what the phone will use to locate the capability on the daemon:

```sh
endo locate mobile-fob
# endo:abc123...   (a long formula locator)
```

Copy this locator. The phone needs it to fetch the capability from the gateway.

## 3. A gateway client (Node.js demo)

Before writing the React Native app, verify the gateway connection with a standalone Node.js script. Create `gateway-client.js` outside your `endo-car-tutorial/` directory (it has its own dependencies):

```sh
mkdir endo-gateway-client
cd endo-gateway-client
npm init -y
npm install @endo/init @endo/far @endo/captp @endo/stream ws
```

Create `client.js`:

```js
import '@endo/init';
import { E } from '@endo/far';
import { makeMessageCapTP } from '@endo/captp';
import { makePipe, mapWriter, mapReader } from '@endo/stream';
import { makePromiseKit } from '@endo/promise-kit';
import WebSocket from 'ws';

// Replace with your daemon's address and the formula locator from step 2.
const GATEWAY = 'ws://localhost:8920/';
const FORMULA_LOCATOR = process.argv[2]; // pass on command line

const { promise: cancelled, reject: cancel } = makePromiseKit();

const socket = new WebSocket(GATEWAY);

await new Promise((resolve, reject) => {
  socket.on('open', resolve);
  socket.on('error', reject);
});

console.log('Connected to gateway');

const [reader, sink] = makePipe();

socket.on('message', (bytes, isBinary) => {
  if (isBinary) sink.next(bytes);
});
socket.on('close', () => sink.return(undefined));

const writer = {
  async next(bytes) {
    socket.send(bytes, { binary: true });
    return { done: false, value: undefined };
  },
  async return() {
    socket.close();
    return { done: true, value: undefined };
  },
  async throw(err) {
    socket.close();
    return { done: true, value: err };
  },
  [Symbol.asyncIterator]() { return this; },
};

const encode = msg => new TextEncoder().encode(JSON.stringify(msg));
const decode = bytes => JSON.parse(new TextDecoder().decode(bytes));

const { getBootstrap } = makeMessageCapTP(
  'mobile-client',
  mapWriter(writer, encode),
  mapReader(reader, decode),
  cancelled,
  undefined,
);

const bootstrap = getBootstrap();

// Fetch the capability by its formula locator.
const fob = await E(bootstrap).fetch(FORMULA_LOCATOR);
console.log('Got capability:', fob);

// Use it.
const result = await E(fob).start();
console.log('Result:', result);

cancel(new Error('done'));
socket.close();
```

Run it with the locator from step 2:

```sh
node --experimental-vm-modules client.js "endo:abc123..."
# Connected to gateway
# Got capability: [object Object]
# Result: vroom!
```

The method called `start()` on a formula that lives on the daemon. The phone — or in this case, the Node.js process standing in for the phone — has no knowledge of how the Car is implemented. It just holds a reference and sends messages.

## 4. React Native: what changes and what does not

The gateway client code above is almost exactly what you would write in a React Native app. The differences are packaging, not protocol.

**What works on React Native out of the box:**
- `@endo/init` — calls `lockdown()` on the JS engine. Tested by Agoric on both Hermes (Android) and JavaScriptCore (iOS).
- `@endo/far` — pure JavaScript, no Node.js dependencies.
- `@endo/captp`, `@endo/marshal`, `@endo/patterns` — pure JavaScript.
- `WebSocket` — React Native's global `WebSocket` works as a drop-in for the `ws` package used above. Remove the `ws` import; `WebSocket` is already in scope.

**What needs attention:**
- Remove the `import WebSocket from 'ws'` line. React Native provides `WebSocket` as a global, exactly as browsers do.
- Replace `import '@endo/init'` with `import '@endo/init/pre.js'` and call `lockdown()` explicitly at app startup (in your root `index.js`), before importing anything that uses SES. Lockdown must run once, before the JS engine has evaluated any untrusted code.
- Configure your Metro bundler (`metro.config.js`) to resolve Node.js built-in shims for any transitive dependency that tries to import `crypto`, `stream`, etc. The `@endo/*` packages themselves do not need Node.js builtins, but bundler warnings can surface from indirect deps.
- The `TextEncoder`/`TextDecoder` globals are available in React Native 0.71+. If you are on an older version, polyfill them.

**Your app startup (`index.js`):**

```js
import { lockdown } from 'ses';

lockdown({
  // 'safe' is the recommended mode for apps that load third-party plugins.
  // 'unsafe' is easier to get started with if you control all the code.
  errorTaming: 'safe',
  overrideTaming: 'severe',
});

import { AppRegistry } from 'react-native';
import App from './App';
import { name as appName } from './app.json';

AppRegistry.registerComponent(appName, () => App);
```

After that, the gateway client code from step 3 works unchanged — swap `new WebSocket(url)` for the global, remove the `ws` import, and you are done.

## 5. The offline problem and the synced pet store

There is one gap in the architecture so far: the phone needs the formula locator out-of-band (Alice emailed it, pasted it into a config file, or hard-coded it). More importantly, if the daemon is unreachable — the phone is on a plane, the home network is down — `E(fob).start()` will not work because the capability lives on the remote daemon.

This is where the synced pet store from tutorial 2 becomes the key infrastructure piece. Instead of the phone fetching a formula locator by ID, Alice and the phone establish a peer relationship via invitation. Each side gets a synced pet store — a CRDT replica that holds the phone's granted capabilities. The phone can read its namespace offline. When it reconnects, the CRDT merges any changes from both sides.

```
[Phone — local state]                    [Daemon — remote state]
  SyncedPetStore (grantee replica)  ←→   SyncedPetStore (grantor replica)
  { 'my-fob': locator }                   { 'my-fob': locator }
  Stored on-device                        Stored in daemon formula files

  When online: sync via CapTP gateway
  When offline: read from local replica
```

Alice revokes the phone's access by writing a tombstone to the synced store on her side. The next time the phone syncs (reconnects to the network), `mergeRemoteState()` delivers the tombstone. The phone's local lookup returns `undefined`. The capability is gone — even if the phone was holding a cached reference, the forwarder on the daemon side is already dead.

The offline capability store is not fully implemented in this tutorial — it requires a persistent storage layer on the device (AsyncStorage in React Native, IndexedDB in a browser). But the CRDT logic is already in `@endo/daemon/src/synced-pet-store.js`. The phone-side store is the same `makeSyncedPetStore` function with a mobile-appropriate `filePowers` implementation (using React Native's file system API instead of Node.js `fs`).

## 6. What a production mobile client looks like

Putting it all together, a production mobile Endo client would have three layers:

**Transport** — a WebSocket connection to the daemon gateway, opened when the app is in the foreground and the network is available. The app attempts to reconnect automatically when it returns from background.

**Capability namespace** — a local synced pet store, persisted to device storage, that holds the set of capabilities Alice has granted to this device. The pet store syncs with the daemon when connected. The app reads from it whenever it needs a capability.

**UI** — shows the list of capabilities from the local store, calls methods on them via `E(locator)`, handles errors gracefully when the daemon is unreachable.

The result is an app that:
- Works offline for read operations (checking what capabilities are available)
- Calls remote methods when connected
- Picks up grants and revocations automatically when it syncs
- Cannot be given more authority than the daemon's grantor replica contains

## 7. Tear down

When you are done testing:

```sh
endo stop
endo purge
```

Remember to revoke the mobile-app grant:

```sh
endo eval 'E(g).revoke("mobile-app")' g:garage
```

## What you now know

- **The Endo daemon has a WebSocket gateway.** Any client that speaks CapTP over WebSocket can hold references to remote capabilities and call methods on them via `E()`. No daemon required on the client side.
- **SES runs on mobile JS engines.** Hermes (Android) and JavaScriptCore (iOS) both support `lockdown()`. Agoric has deployed this in production.
- **The React Native client is the same code.** Remove the `ws` import, use the global `WebSocket`, call `lockdown()` before anything else. The rest is unchanged.
- **The synced pet store is the offline layer.** A CRDT replica on the device holds the capability namespace. Revocations propagate on reconnect. The phone can work offline and stay in sync with the daemon.
- **Tailscale is the simplest secure transport.** Put both the daemon and the phone on a Tailscale network. You get encrypted transport, device-level authentication, and no firewall rules to manage.

## What's next

The seven tutorials have taken you from a single object in a single daemon to a mobile app holding capabilities across a network, with offline support and automatic revocation. The Endo ecosystem has more: OCapN (the capability transport protocol being standardised for cross-language interoperability), the Familiar desktop shell, and Agoric's smart contract platform, which uses the same SES/HardenedJS substrate for production financial contracts. The patterns you have learned here scale to all of it.
