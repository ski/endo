---
title: 03-per-grantee-wrappers
group: Documents
category: Guides
---

# One Key Per Valet: Per-Grantee Wrappers and Independent Revocation

In [tutorial 2](https://suhail.ski/posts/mailing-bob-the-start-fob-capabilities-across-the-wire) Alice revoked the start fob by removing it from the synced store. That killed every downstream copy at once — a blunt instrument. What if she wants to cut Bob's access without affecting Charlie's?

The answer is that she should never have handed Bob and Charlie the same object. She should have minted a separate wrapper for each of them — same underlying capability, different forwarder, independent kill switch. This tutorial builds that pattern. It runs entirely in a single daemon; no two-daemon setup required. The concept is simple enough that the network would only be noise.

## What you'll build

A `makeRevocableForwarder` factory that wraps any capability with an on/off switch. Alice calls it twice — once for Bob, once for Charlie — and hands each of them their own forwarder pointing at the same Car. She revokes Bob's. Bob's copy dies. Charlie's still works. The Car is untouched throughout.

## What you'll need

- Tutorial 1 completed: you have the daemon running, the Car installed, and Bob as a guest.
- The `endo-car-tutorial/` project directory from tutorial 1, with `@endo/far` installed.
- About 20 minutes.

## 1. The forwarder factory

Inside `endo-car-tutorial/`, create `revocable.js`:

```js
import harden from '@endo/harden';
import { E, Far } from '@endo/far';

export const make = () =>
  Far('RevocableFactory', {
    wrap(target) {
      let alive = true;

      const forwarder = Far('Forwarder', {
        async start() {
          if (!alive) throw new Error('revoked');
          return E(target).start();
        },
        async unlock() {
          if (!alive) throw new Error('revoked');
          return E(target).unlock();
        },
      });

      const revoker = Far('Revoker', {
        revoke() {
          alive = false;
        },
      });

      return harden({ forwarder, revoker });
    },
  });
```

Install the `@endo/harden` package alongside `@endo/far` in your project:

```sh
npm install @endo/harden
```

A few things worth noting before you move on.

The factory exposes `wrap(target)`, which returns two objects: a `forwarder` that delegates method calls to `target`, and a `revoker` that flips the internal `alive` flag. Once `revoker.revoke()` is called, every subsequent call through the forwarder throws. The target never changes — it has no idea the forwarder exists.

`alive` is a plain boolean in the closure. It is not an Endo formula, not a pet name, not persisted. It lives only in the worker process that evaluates this module. That has one implication worth stating plainly: if the daemon restarts, the forwarder is recreated from its formula, `alive` resets to `true`, and the revocation is gone. For persistent revocation you would store a flag in a durable store and load it on init. This tutorial keeps it simple, but carry that caveat in mind.

**Sidebar — why two separate Far objects?** Alice needs to keep `revoker` and give Bob only `forwarder`. If both lived on the same object, Bob could call `revoke()` himself and deny service to everyone else — including Charlie. Separating them means Alice literally cannot hand Bob the thing that turns off his own access, even by accident. Least authority, again, applied to the revocation mechanism itself.

## 2. Install the factory

From `endo-car-tutorial/`:

```sh
endo make revocable.js --name revocable-factory
```

Check that it's there:

```sh
endo list
# my-car  revocable-factory
```

The factory is now a formula in Alice's daemon, addressable by pet name. Only Alice has it; no guest can see it unless she explicitly sends it to them.

## 3. Make sure the Car and guests are set up

If you're continuing from a fresh daemon, re-run the setup from tutorial 1:

```sh
endo make car.js --name my-car
endo mkguest bob bob-agent
endo mkguest charlie charlie-agent
```

If Bob and Charlie already exist from a previous session, a second `endo mkguest` on the same name will fail — just skip it. Confirm both guests are visible:

```sh
endo list
# bob  bob-agent  charlie  charlie-agent  my-car  revocable-factory
```

## 4. Mint two independent forwarders

Alice wraps the Car once for Bob and once for Charlie. Each call to `wrap` creates a completely separate pair of forwarder + revoker objects:

```sh
endo eval 'E(f).wrap(c)' f:revocable-factory c:my-car --name bobs-bundle
endo eval 'E(f).wrap(c)' f:revocable-factory c:my-car --name charlies-bundle
```

Each bundle is an object with two fields: `forwarder` and `revoker`. Pull them out into their own pet names:

```sh
endo eval 'E(b).forwarder' b:bobs-bundle     --name bobs-fob
endo eval 'E(b).revoker'   b:bobs-bundle     --name bobs-revoker
endo eval 'E(b).forwarder' b:charlies-bundle --name charlies-fob
endo eval 'E(b).revoker'   b:charlies-bundle --name charlies-revoker
```

Alice now holds four pet names:

- `bobs-fob` — goes to Bob
- `bobs-revoker` — Alice keeps this; it's Bob's kill switch
- `charlies-fob` — goes to Charlie
- `charlies-revoker` — Alice keeps this; it's Charlie's kill switch

You should now see all of them in `endo list`. Bob and Charlie have none of them yet.

## 5. Hand each guest their fob

```sh
endo send bob 'Your start fob. @bobs-fob.'
endo send charlie 'Your start fob. @charlies-fob.'
```

Bob adopts his:

```sh
endo inbox --as bob-agent
# 0. "HOST" sent "Your start fob. @bobs-fob."
endo adopt --as bob-agent 0 bobs-fob
```

Charlie adopts his:

```sh
endo inbox --as charlie-agent
# 0. "HOST" sent "Your start fob. @charlies-fob."
endo adopt --as charlie-agent 0 charlies-fob
```

Confirm each guest has exactly one capability:

```sh
endo list --as bob-agent
# bobs-fob

endo list --as charlie-agent
# charlies-fob
```

Neither guest has the revoker. Neither guest has the Car. Neither guest has any visibility into the other's namespace.

## 6. Both drive

Bob:

```sh
endo eval --as bob-agent 'E(f).start()' f:bobs-fob
# vroom!
```

Charlie:

```sh
endo eval --as charlie-agent 'E(f).start()' f:charlies-fob
# vroom!
```

Two guests, two forwarders, one underlying Car, everyone happy.

## 7. Revoke Bob — and only Bob

Alice calls Bob's revoker:

```sh
endo eval 'E(r).revoke()' r:bobs-revoker
```

Bob tries to drive:

```sh
endo eval --as bob-agent 'E(f).start()' f:bobs-fob
# Error: revoked
```

Charlie tries to drive:

```sh
endo eval --as charlie-agent 'E(f).start()' f:charlies-fob
# vroom!
```

The Car:

```sh
endo eval 'E(c).start()' c:my-car
# vroom!
```

Bob's access is gone. Charlie's is not. The Car is untouched. Alice revoked one grantee without touching anyone else, because each grantee had their own object with their own internal state.

**Sidebar — composability.** You can stack these layers freely. An attenuated wrapper (tutorial 1 — exposes only `unlock`) around a revocable forwarder gives you attenuation and revocation independently. A revocable forwarder around an attenuated wrapper gives the same. A factory that mints revocable attenuated wrappers gives you both for each grantee. The layers compose because they are all just objects — `Far` references whose methods close over whatever state they need. There is no framework magic, no central registry, no ACL table. The policy is the object graph.

## 8. What the factory itself is

`revocable-factory` is itself a capability. Right now only Alice has it. If she granted it to Bob, Bob could mint his own forwarders — wrapping anything he holds, issuing sub-capabilities to his own guests. If Alice wanted to limit that, she could attenuate the factory before sending it: wrap it in a new Far object that only allows wrapping a specific set of targets, or that limits the number of forwarders that can be minted. The pattern recurses without limit.

This is one of the structural differences between capability systems and traditional access control: policy is expressed by what you hand out and how you wrap it, not by what you write in a central database. Every wrapper is a policy decision. Every attenuation is a policy decision. They compose.

## 9. Tear down

```sh
endo stop
endo purge
```

## What you now know

- **One capability per grantee.** Sharing the same capability object with multiple people means revocation is all-or-nothing. Mint a separate wrapper per grantee, and each kill switch is independent.
- **Separation of forwarder and revoker.** The thing you hand out and the thing that turns it off should never be the same object. Separating them means the grantee cannot revoke their own access (or anyone else's) unless you explicitly grant them the revoker.
- **Composability is structural.** Attenuation, revocation, and delegation all compose because they are all just objects. Stack them in any order. The pattern works at any depth.
- **Persistence caveat.** The `alive` flag in this tutorial lives in the worker process. A daemon restart resets it. For persistent revocation, store the flag durably — in a pet store entry, a persistent formula, or an external store.

## What's next

In tutorial 4, Bob doesn't just receive capabilities — he makes a request. Alice can approve or reject it, and the result is a capability Bob can use. This is the Endo mailbox and form system: the conversation layer that sits above the object layer, useful when the capability Bob needs doesn't exist yet and has to be provisioned on demand.
