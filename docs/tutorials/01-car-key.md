---
title: 01-car-key
group: Documents
category: Guides
---

# Alice's Car Key: Capabilities, Attenuated

By the end of this tutorial you will have built a `Car` object with two methods, wrapped it in a "valet key" that only unlocks, handed that key to a confined guest named Bob, and watched Bob try (and fail) to drive away. That sequence is the whole spine of Endo: capabilities are objects, you name them locally, and you can always hand out a narrower version — what the people who work on this for a living call an attenuated capability. "Attenuated" just means "deliberately weakened." Decaf espresso. The kids' menu. The valet key you hand to someone you'd rather not see do donuts in the parking lot.

## What you'll need

- Node.js 20 or later.
- Two terminal windows. One for the daemon's log, one for everything else.
- About 25 minutes.

## 1. Install Endo and start the daemon

Install the published CLI:

```sh
npm install -g @endo/cli
```

If you are working from a clone of the Endo repository instead of the published package, run `yarn` at the repo root and then alias the binary: `alias endo=$PWD/packages/cli/bin/endo`.

Start the daemon:

```sh
endo start
```

In your second terminal, tail the log so you can watch the daemon react to each command as you run it:

```sh
endo log -f
```

Confirm the daemon is alive:

```sh
endo ping
```

You should see `ok`. The daemon is a long-running background process that keeps your capabilities alive between commands; everything else in this tutorial talks to it through a Unix domain socket (or, on Windows, a named pipe).

## 2. Create a project directory

Endo's daemon is a long-running shared resource on your machine, but the files you write (like `car.js` in the next step) live in a normal Node.js project on your filesystem. When you ask the daemon to instantiate `car.js`, the CLI bundles the file using your project's `node_modules` to resolve imports — so the imports in your file need to be installable packages.

Make a fresh directory and initialise it:

```sh
mkdir endo-car-tutorial
cd endo-car-tutorial
npm init -y
```

Open the generated `package.json` and add `"type": "module"` so that plain `.js` files are treated as ES modules. You should end up with something like:

```json
{
  "name": "endo-car-tutorial",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  }
}
```

Install the one dependency you'll need:

```sh
npm install @endo/far
```

`@endo/far` is the package that gives you `Far` and `E` — the two building blocks of every capability you'll write. You should now have an `endo-car-tutorial/` directory containing `package.json`, `package-lock.json`, and a `node_modules/` folder.

Stay in this directory for the rest of the tutorial. Every `endo` command from here on assumes this is your current working directory.

## 3. Define the Car

Inside `endo-car-tutorial/`, create a file called `car.js`:

```js
import { Far } from '@endo/far';

export const make = () => {
  let locked = true;
  return Far('Car', {
    unlock() {
      locked = false;
      return 'click! unlocked';
    },
    start() {
      if (locked) throw new Error("can't start: doors are locked");
      return 'vroom!';
    },
  });
};
```

That is the entire Car. A factory function `make` returns an object with two methods. `Far('Car', { ... })` is the load-bearing call: it marks the returned object as a "far reference" — a thing whose methods are safe to invoke from across confinement boundaries, and which Endo will let you pass around between agents. You should now have a 15-line file that exports a `make` function.

**Sidebar — where did `harden` go?** Endo's hardening rule is "freeze deeply, so no one can mutate this object from under you." `Far()` hardens its return value for you, so you do not have to call `harden()` by hand on the result. If you forgot the `Far` (or wrote a plain object without freezing it), anyone holding a reference could rewrite the methods — including the threat we are about to describe. The framework gives you a sharp default: when in doubt, wrap with `Far`.

## 4. Install the car as a capability

From inside `endo-car-tutorial/` (the directory you created in step 2), ask the daemon to instantiate `car.js` and remember the result under the name `my-car`:

```sh
endo make car.js --name my-car
```

The path `car.js` is resolved relative to your current working directory, and the bundler walks up from there looking for `node_modules` — which is why you have to run this command from the project directory, not from your home folder.

List your pet names to confirm:

```sh
endo list
```

You should see `my-car` in the list. What you have just created is a "formula" — a persistent recipe for the Car that survives daemon restarts — plus a "pet name" in your personal namespace that points at it. Three layers, worth pausing on:

- **Pet name** (`my-car`): your local nickname. Yours alone.
- **Formula ID**: a long, unguessable hex string the daemon uses to refer to the recipe. See it with `endo locate my-car`.
- **Live object**: the actual `Car` instance, alive in a worker process, reachable through the formula.

**Sidebar — why "pet name"?** Not whimsical: it's a [Mark Miller term](https://en.wikipedia.org/wiki/Petname) from capability-systems research, contrasted with globally-scoped names like DNS hostnames or URLs that everyone resolves the same way (and that can be squatted or guessed). A pet name is yours alone — chosen by you, meaningful to you, useless to anyone else. Endo's three-layer model (pet name → formula → live object) makes that middle layer a first-class subsystem with a name of its own. **Don't confuse it with "vat"**, which is a different ocap term for an isolated execution domain. Endo's analog of a vat is what it calls a "worker" — each formula runs in one.

## 5. Use the car

Send a method call:

```sh
endo eval 'E(c).unlock()' c:my-car
```

You should see `click! unlocked`. Now start the engine:

```sh
endo eval 'E(c).start()' c:my-car
```

You should see `vroom!`.

A few words on what just happened. The string after `endo eval` is a tiny JavaScript program that runs inside a Hardened JavaScript Compartment. The argument `c:my-car` says "inside that program, the name `c` should be bound to whatever the pet name `my-car` refers to." `E(c).unlock()` is "eventual send": it sends an `unlock` message to whatever `c` is and returns a promise for the result. You write `E(c).method()` instead of `c.method()` because in general the target might live in a different process or on another machine; eventual send abstracts that distance.

## 6. The valet key (attenuation, the load-bearing idea)

Alice does not always want to grant the full Car. Sometimes she wants to lend out a key that can only unlock — for a valet, for a courier, for anyone she does not yet trust with the engine.

Build the valet key inline with `endo eval`:

```sh
endo eval 'Far("ValetKey", { unlock: () => E(c).unlock() })' c:my-car --name valet-key
```

You should see the new value printed. Confirm it shows up alongside `my-car`:

```sh
endo list
```

What you just did is attenuation. The valet key holds, in its closure, a reference to the underlying car. But it only exposes one method, `unlock`. The `start` method does not exist on the valet key, because no one wrote it there. That is the whole mechanism.

Test that the key works:

```sh
endo eval 'E(k).unlock()' k:valet-key
```

You should see `click! unlocked`.

Try to start the engine through the key:

```sh
endo eval 'E(k).start()' k:valet-key
```

You should get an error like `target has no method "start"`. The valet key is not the car. It is a narrower thing that happens to be able to ask the car to unlock.

**Sidebar — what `harden` is protecting you from.** Suppose you had written the Car as a plain object without `Far`:

```js
export const make = () => {
  let locked = true;
  return {
    unlock() { locked = false; return 'click!'; },
    start()  { if (locked) throw Error('locked'); return 'vroom!'; },
  };
};
```

Anyone who held a reference to this car could, in one line, do:

```js
car.unlock = () => car.start();
```

The next time Alice's own code called `car.unlock()`, the engine would fire. JavaScript objects are mutable by default; methods are just properties, and any property can be reassigned. Hardening the object freezes everything reachable from it so its behavior is fixed at the moment you minted it. `Far()` does this for you. And note: this threat compounds with delegation. In a capability system, if Alice gives Bob a reference, Bob can pass that reference to Charlie — Alice cannot prevent the sharing. An unhardened object can be mutated by anyone in the chain, and the mutation echoes back to whoever else holds a reference.

## 7. Spawn Bob

So far Alice has been talking to herself. To see attenuation do real work, introduce a second agent — a confined guest who is not Alice.

```sh
endo mkguest bob bob-agent
```

This creates two things on Alice's side, both new pet names: `bob` (a "handle" that appears in the "from" and "to" fields of messages) and `bob-agent` (Bob's "agent", which is the permission broker Alice uses to act on Bob's behalf for testing purposes).

Confirm Bob's namespace starts empty:

```sh
endo list --as bob-agent
```

You should see nothing. Bob has no pet names yet. He cannot reach `my-car`. He cannot even reach `valet-key`.

## 8. Hand Bob the key (and nothing else)

Send Bob a message containing the valet key:

```sh
endo send bob 'Here is your @valet-key.'
```

The `@valet-key` token is the way Endo embeds a capability reference in a human-readable message. Bob receives the message; the daemon stages the embedded reference for him to adopt under a name of his choosing.

Check Bob's inbox:

```sh
endo inbox --as bob-agent
```

You should see one message: `"HOST" sent "Here is your @valet-key."`.

Have Bob adopt the key into his own namespace:

```sh
endo adopt --as bob-agent 0 valet-key
```

(The `0` is the message number.)

Confirm Bob now has exactly one pet name:

```sh
endo list --as bob-agent
```

You should see `valet-key` and nothing else. Bob's universe contains one capability, the one Alice chose to deliver.

## 9. The aha moment

Bob unlocks the car:

```sh
endo eval --as bob-agent 'E(k).unlock()' k:valet-key
```

You should see `click! unlocked`.

Bob tries to start the engine through the key:

```sh
endo eval --as bob-agent 'E(k).start()' k:valet-key
```

You should see an error like `target has no method "start"`.

Bob tries to find the car directly:

```sh
endo eval --as bob-agent 'E(c).start()' c:my-car
```

You should see an error saying the pet name is unknown. Bob cannot reach `my-car` because it was never in his namespace. He has no way to guess it — formula IDs are unforgeable hex strings, and pet names are local. The only capability he holds is the valet key, and the valet key only unlocks.

Sit with this for a second. You did not write any access-control list. You did not configure any role. The reason Bob cannot start the engine is that the only object in his namespace that talks to the car does not have a `start` method on it. That is the entire mechanism. Principle of least authority, expressed in fifteen lines of JavaScript and one `endo send`.

## 10. Tear down

When you are done, stop the daemon:

```sh
endo stop
```

If you want to wipe state and start fresh next time:

```sh
endo purge
```

`purge` is destructive: it removes the persisted formulas and pet stores for every agent on this daemon. For a clean slate before the next tutorial, that is what you want.

## What you now know

- **Capabilities are objects.** A capability is just a hardened reference. If you have it, you can use it; if you don't, no amount of guessing will help.
- **Pet names are nicknames.** Each agent has their own pet store. `my-car` is meaningful to Alice and invisible to Bob. The thing that travels between agents is the reference, not the name.
- **Attenuation is wrapping.** To give someone a narrower capability, write an object that closes over the full capability and exposes only the methods you want shared. The security boundary is the lexical scope of the closure plus the hardening that prevents anyone from rewriting the wrapper.

## What's next

In tutorial 2, Bob moves out of Alice's daemon and into his own. Alice mails him an invitation, Bob accepts, and now Alice can grant Bob the `start` capability over the network — same mental model, longer wire. You will meet invitations, the synced pet store, and the question of revocation: Bob can pass the key to Charlie if he wants, and Alice cannot stop him — but she can cut the line.
