---
title: 04-mailbox-and-requests
group: Documents
category: Guides
---

# Bob Makes a Request: The Endo Mailbox and Forms

In tutorials 1 through 3, Alice was always the one pushing capabilities to Bob. She decided what to mint, when to send it, and what to revoke. Bob was passive: he received, he used, he waited for Alice to decide.

That is one valid pattern. But it is not the only one. Sometimes the capability Bob needs does not exist yet, or its creation requires human judgement at runtime. Sometimes Bob is a program that needs to declare its requirements and then wait. For these cases, Endo provides a mailbox: a structured way for guests to make requests of their host, and for the host to approve or deny them.

This tutorial builds that flow end to end. You will see a request travel from Bob to Alice, Alice approve it, and Bob use the resulting capability — all without Alice having to predict Bob's needs ahead of time.

## What you'll build

Bob sends Alice a request for a start fob. Alice sees it in her inbox. Alice approves it by resolving the request with a capability she already has. Bob's command unblocks and prints the result. Then you will try the rejection path, and see how a program handles both outcomes.

## What you'll need

- Tutorial 1 setup: the daemon running, `my-car` installed, `bob` and `bob-agent` as pet names.
- Tutorial 3 setup: the `revocable-factory` installed (for minting a dedicated fob for Bob).
- About 20 minutes.

## 1. Alice mints a fob to grant

Alice prepares a dedicated start fob for Bob using the revocable factory from tutorial 3. She does not send it to Bob yet — she just has it ready to grant:

```sh
endo eval 'E(f).wrap(c)' f:revocable-factory c:my-car --name bobs-bundle
endo eval 'E(b).forwarder' b:bobs-bundle --name bobs-fob
endo eval 'E(b).revoker'   b:bobs-bundle --name bobs-revoker
```

Alice holds `bobs-fob` and `bobs-revoker`. Bob has neither.

## 2. Bob makes a request

Open a second terminal (Bob's). From Bob's perspective, ask Alice for a start fob:

```sh
endo request 'Please grant me a start fob.' --as bob-agent --name my-fob
```

This command blocks. Bob is waiting. Alice has not responded yet. You should see the cursor hanging — the daemon is holding Bob's request open, waiting for Alice to act.

Leave Bob's terminal open and switch to Alice's.

## 3. Alice checks her inbox

```sh
endo inbox
```

You should see something like:

```
0. "bob" requested "Please grant me a start fob."
```

The message number is `0`. Alice can resolve or reject it.

## 4. Alice approves

Alice grants Bob access by resolving the request with the `bobs-fob` capability she prepared in step 1:

```sh
endo resolve 0 bobs-fob
```

Switch back to Bob's terminal. The `endo request` command unblocks and prints:

```
[object Object]
```

(Endo prints a representation of the resolved value — the forwarder object. The exact format varies. What matters is that the command unblocked and did not throw.)

Bob now has `my-fob` in his namespace:

```sh
endo list --as bob-agent
# my-fob
```

Bob drives:

```sh
endo eval --as bob-agent 'E(f).start()' f:my-fob
# vroom!
```

**What just happened under the hood.** The `endo request` call on Bob's side is an eventual send to Bob's agent: `E(bobAgent).request('HOST', 'Please grant me a start fob.', 'my-fob')`. The daemon created a pending request formula and put a message in Alice's inbox. When Alice called `endo resolve 0 bobs-fob`, the daemon resolved Bob's promise with the `bobs-fob` capability and wrote it into Bob's pet store under `my-fob`. The request formula is now settled; it survives a daemon restart in its resolved state.

## 5. The rejection path

Reset: remove the fob from Bob's namespace so he is empty again:

```sh
endo remove my-fob --as bob-agent
```

Bob makes another request. In Bob's terminal:

```sh
endo request 'Please grant me a start fob.' --as bob-agent --name my-fob
```

Alice's inbox will now show message `1` (the numbering is cumulative). This time Alice rejects:

```sh
endo reject 1 'Not today.'
```

Bob's terminal throws:

```
Error: Not today.
```

The command exits with an error. `my-fob` does not appear in Bob's namespace — a rejected request does not write anything. Bob's code needs to handle this case.

## 6. Handling both outcomes in code

The CLI is convenient for interactive testing, but real guests are usually programs. Here is how the same request/resolve/reject flow looks in a worklet — a module loaded via `endo make` that runs in a persistent worker.

Create `requester.js` in your project directory:

```js
import harden from '@endo/harden';
import { E } from '@endo/far';
import { makeExo } from '@endo/exo';
import { M } from '@endo/patterns';

export const make = powers => {
  // Request a start fob from the host. This returns a promise that resolves
  // when the host calls endo resolve, or rejects when the host calls endo reject.
  // The third argument ('my-fob') is the pet name to store it under if granted.
  const fobP = E(powers).request(
    'HOST',
    'Please grant me a start fob.',
    'my-fob',
  );

  return makeExo(
    'Requester',
    M.interface('Requester', {}, { defaultGuards: 'passable' }),
    {
      async drive() {
        let fob;
        try {
          fob = await fobP;
        } catch (err) {
          return `Access denied: ${err.message}`;
        }
        return E(fob).start();
      },
    },
  );
};
```

Install it with a guest agent as its powers so it can make requests on Bob's behalf:

```sh
endo make requester.js --name requester --powers bob-agent
```

Check Alice's inbox — the requester fired its request immediately on startup:

```sh
endo inbox
# 2. "bob" requested "Please grant me a start fob."
```

Alice resolves:

```sh
endo resolve 2 bobs-fob
```

Now call the requester's `drive` method:

```sh
endo eval 'E(r).drive()' r:requester
# vroom!
```

Call it again — the fob is settled, the request does not repeat:

```sh
endo eval 'E(r).drive()' r:requester
# vroom!
```

Restart the daemon:

```sh
endo restart
endo eval 'E(r).drive()' r:requester
# vroom!
```

The request is resolved. The formula persists. No new inbox message appears after restart because the promise is already settled — Endo remembers.

Now see what happens when Alice revokes the fob while the requester holds it:

```sh
endo eval 'E(r).revoke()' r:bobs-revoker
endo eval 'E(r).drive()' r:requester
# Access denied: revoked
```

The requester's `drive` method catches the error and returns a message instead of throwing. The try/catch in `drive` is the standard pattern for capability code that must handle denied access gracefully.

**Sidebar — the request as a promise.** `E(powers).request(...)` returns a promise immediately, before the host has done anything. The worklet can pipeline off that promise right away. The `fobP` variable holds a promise that the worklet captures at startup. Calling `E(fobP).start()` would send the `start` message to whatever `fobP` eventually resolves to — that pipelining is what makes eventual send composable at scale. In this tutorial we `await fobP` inside `drive()` to keep the error handling readable, but you can just as easily pipeline: `return E(await fobP).start()`.

## 7. Following requests in real time

If Alice is at her terminal waiting for requests from multiple guests, she can follow her inbox:

```sh
endo inbox --follow
```

This streams new messages as they arrive. Each line shows the message number, sender, and description. Alice can resolve or reject from a separate terminal while `inbox --follow` streams.

## 8. Tear down

```sh
endo stop
endo purge
```

## What you now know

- **Requests flow from guest to host.** `E(powers).request('HOST', description, name)` puts a message in the host's inbox and returns a promise that settles when the host acts.
- **Resolve grants a capability.** `endo resolve <N> <pet-name>` fulfils the promise with whatever capability lives under that pet name, writes it into the guest's namespace under the requested name, and unblocks the guest.
- **Reject denies without granting.** `endo reject <N> <message>` rejects the promise. Nothing is written to the guest's namespace. The guest's code should catch the rejection.
- **Settled requests survive restarts.** The formula is persisted in its resolved or rejected state. A restarted daemon does not re-fire pending requests for formulas that are already settled.
- **The request is a promise you can pipeline.** You do not have to await it immediately. You can send messages to the eventual result and Endo will queue them until the host acts.

## What's next

In tutorial 5, we step back from individual capabilities and look at the whole object: a confined plugin that declares its own needs, receives exactly what it asks for, and cannot reach anything else. This is Endo's plugin model — the thing that lets you run arbitrary third-party code without giving it access to the rest of your system.
