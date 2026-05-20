---
title: 02-mailing-keys
group: Documents
category: Guides
---

# Mailing Bob the Start Fob: Capabilities Across the Wire

In [tutorial 1](https://suhail.ski/posts/alices-car-key-capabilities-attenuated), Bob was a guest inside Alice's daemon. Different namespace, same process. This time he moves out. Bob gets his own daemon — his own state directory, his own socket, his own everything — and Alice has to hand him capabilities across a wire. The mental model from tutorial 1 doesn't change. A capability is still an unforgeable reference; pet names are still local; attenuation is still just wrapping. What changes is the substrate that carries the reference from one daemon to the other.

This tutorial walks through Endo's invitation protocol, the synced pet store that pairs two peers, the wire-side delivery of a capability, and the question of revocation — what it means, what it doesn't mean, and why Alice can cut the line but cannot stop Bob from making copies before she does.

## What you'll need

- Tutorial 1 completed (you have the mental model and the `endo` CLI installed somewhere).
- The `kriskowal-llm-synced-pet-store` branch of [ski/endo](https://github.com/ski/endo) checked out and built. The synced pet store machinery this tutorial relies on lives on that branch and is not in the published `@endo/cli` from npm yet.
- Two terminal windows for the live work, plus one more if you want to tail logs.
- About 40 minutes.

## 0. Build from source

The published `@endo/cli` does not yet have the synced pet store. Clone the branch and build the workspace:

```sh
git clone https://github.com/ski/endo.git endo-synced
cd endo-synced
git checkout kriskowal-llm-synced-pet-store
corepack enable
yarn
```

The install takes a couple of minutes and pulls about 600 MB of dependencies. When it finishes, alias the CLI binary so the rest of this tutorial reads naturally:

```sh
alias endo="$PWD/packages/cli/bin/endo"
```

(On Windows under Git Bash, the launcher is a `.cjs` file; invoke it via `node $PWD/packages/cli/bin/endo.cjs ...` or rely on the `endo.cmd` shim that comes with the package.)

Confirm:

```sh
endo --version
```

You should see a 2.x version printed.

## 1. Two daemons, two terminals

Endo's daemon listens on a single socket per user. To run two side by side, you point each one at a different state directory and a different socket. The cleanest way is to export environment variables in each shell before starting the daemon.

In your first terminal — call this Alice's terminal:

```sh
export ENDO_HOME=$HOME/endo-alice
mkdir -p "$ENDO_HOME"/{state,run}
export XDG_STATE_HOME="$ENDO_HOME/state"
export XDG_RUNTIME_DIR="$ENDO_HOME/run"
endo start
endo ping
```

In your second terminal — Bob's:

```sh
export ENDO_HOME=$HOME/endo-bob
mkdir -p "$ENDO_HOME"/{state,run}
export XDG_STATE_HOME="$ENDO_HOME/state"
export XDG_RUNTIME_DIR="$ENDO_HOME/run"
endo start
endo ping
```

Both should print `ok`. Each terminal now talks to its own daemon. Confirm they are isolated:

```sh
# Alice's terminal
endo list
# (empty)

# Bob's terminal
endo list
# (empty)
```

Two empty namespaces, talking to two different daemons. Note: keep these environment variables exported for the entire tutorial. Every `endo` command in Alice's terminal needs them pointed at Alice's home, and likewise for Bob. A new terminal that doesn't export them will silently talk to your default daemon — which is neither.

> **Windows note.** Endo's daemon on Windows uses named pipes, not Unix sockets, and the pipe path is derived from `USERNAME` rather than from `XDG_RUNTIME_DIR`. To get two daemons side by side on the same Windows user account, set `ENDO_SOCK` explicitly to a different named-pipe path for each (e.g. `\\?\pipe\suhai-Endo-alice\captp0.pipe` and `\\?\pipe\suhai-Endo-bob\captp0.pipe`). The state directory is still controlled by `XDG_STATE_HOME` on every platform, so the example above stays accurate except for the socket override.

## 2. Re-mint the Car (Alice's side only)

Alice needs the same `Car` from tutorial 1 in her new, fresh state directory. From Alice's terminal, in a project directory with `@endo/far` installed:

```sh
endo make car.js --name my-car
endo eval 'E(c).unlock()' c:my-car   # click! unlocked
endo eval 'E(c).start()' c:my-car    # vroom!
```

Bob's terminal has no car. `endo list` in Bob's terminal is still empty. The two daemons currently know nothing about each other.

## 3. Alice invites Bob

In Alice's terminal:

```sh
endo invite bob > /tmp/invitation.txt
```

`endo invite` creates a one-shot capability — a sealed envelope that carries Alice's identity and a single-use slot for whoever opens it. The output, a long opaque locator string, is what Alice has to physically deliver to Bob. Out-of-band: email, Signal, scribbled on a napkin. The locator is not secret in the cryptographic sense (it's reachable by URL), but it IS single-use: once someone accepts it, it cannot be accepted again.

Look at the file if you like:

```sh
cat /tmp/invitation.txt
```

It will be a single line that starts with `endo:` or similar — a locator URL the recipient's daemon can dial.

## 4. Bob accepts

In Bob's terminal:

```sh
cat /tmp/invitation.txt | endo accept alice
```

(The `accept` command reads the locator from stdin, not as an argument, which makes pipe-friendly out-of-band delivery natural.)

The argument `alice` is the pet name Bob chooses for whoever sent the invitation. From Bob's perspective, the host on the other end of this connection will henceforth be known as `alice`. Alice's actual identity (the cryptographic key under the hood) is opaque; the pet name is Bob's nickname for it.

Now look at both namespaces:

```sh
# Alice's terminal
endo list
# bob

# Bob's terminal
endo list
# alice
```

Each side has a new pet name pointing at the other peer. These names point at something more interesting than just a peer reference, though.

## 5. The synced pet store, revealed

On this branch of Endo, accepting an invitation does more than introduce two peers. It also creates a paired "synced pet store" on each side — a shared map that converges between Alice and Bob without either party being the server. Either side can add and remove entries; both sides eventually agree on what the map contains.

From Alice's terminal, look at the synced store named `bob`:

```sh
endo eval 'E(s).list()' s:bob
```

You should see one entry — Bob's handle, written by the invitation flow itself as proof-of-introduction. (The exact name will depend on which side was the grantor; in this case it's whatever name Alice used in `endo invite bob`.)

From Bob's terminal:

```sh
endo eval 'E(s).list()' s:alice
```

You should see a corresponding entry from the other direction. The two stores are mirrors of each other; what's on one is on the other (modulo the network catching up).

**Sidebar — what a synced entry looks like.** Each entry in the synced store is a triple: a "locator" (the capability), a "Lamport timestamp" (a logical clock value), and a "writer" (which side authored it). Conflicts — two writes to the same name from both sides — resolve by higher timestamp wins, with a tombstone bias on ties (a deletion beats a concurrent write at the same logical instant), and a lexicographic tiebreak as the last resort. The result is a last-writer-wins map with sticky deletions, which is the right shape for "names of capabilities the two of us share."

## 6. Mint the start-fob

Alice wants to grant Bob the engine, and only the engine. In tutorial 1 she built a valet key that exposed `unlock`. This time she builds the opposite: a start fob that exposes `start`, and nothing else. The pattern is the same — wrap the underlying capability in a Far object that only re-exports the methods you want shared.

From Alice's terminal:

```sh
endo eval 'Far("StartFob", { start: () => E(c).start() })' c:my-car --name start-fob
```

Sanity check:

```sh
endo eval 'E(f).start()' f:start-fob
# vroom!
```

This is a new, distinct capability from the car itself. It has its own formula and its own pet name. Crucially, this means it can be revoked independently of the car. The car will still be in Alice's namespace and still drive; only the start fob will be cut.

## 7. Hand the start fob across the wire

In tutorial 1 Alice sent Bob a capability via `endo send`, a message in Bob's inbox that he had to `adopt` into his namespace. That flow still works across daemons, but the synced pet store gives us a cleaner alternative: write the fob directly into the shared map, and Bob's side will see it.

First, Alice needs the start fob's locator — the long unforgeable string the daemon uses to refer to it under the hood:

```sh
endo locate start-fob
# endo://...
```

Then she writes it into the synced store with Bob:

```sh
LOC=$(endo locate start-fob)
endo eval "E(s).write('start-fob', '$LOC')" s:bob
```

(In real deployment the locator passes between the daemons over CapTP and you don't shell-quote it; for the tutorial we use a shell variable so it's easy to inspect.)

Verify that Alice sees the entry:

```sh
endo eval 'E(s).list()' s:bob
# [ 'bob', 'start-fob' ]
```

Bob will see it too once the two stores have synced. On this branch, after a brief moment:

```sh
# Bob's terminal
endo eval 'E(s).list()' s:alice
# [ 'alice-handle', 'start-fob' ]
```

## 8. Bob drives

Bob looks up the start fob in his synced store with Alice, and uses it:

```sh
# Bob's terminal
endo eval 'E(s).lookup("start-fob")' s:alice
# (prints the locator)
```

The `lookup` returns a locator string, which Bob can pass into `E()` directly by giving it a name in eval scope:

```sh
endo eval 'E(s).lookup("start-fob")' s:alice --name fob
endo eval 'E(f).start()' f:fob
# vroom!
```

Bob has driven Alice's car, from his own daemon, through nothing but the capability she gave him. He still has no reference to the car. He still cannot unlock it (Alice didn't grant `unlock` — only `start`). He can only do the one thing the fob does.

Sit with this for a second. The fob travelled across a CapTP connection between two processes. The method invocation on Bob's side resulted in an eventual-send message routed back to Alice's daemon, where the actual `Car` object lives. The car's `start()` ran in Alice's worker, and the result came back to Bob. All of that machinery is hidden behind one line — `E(f).start()` — which is exactly the same shape as the in-process call in tutorial 1.

## 9. The aha: revocation

Alice has changed her mind. She no longer wants Bob to be able to start the engine. From Alice's terminal:

```sh
endo eval 'E(s).remove("start-fob")' s:bob
```

This writes a tombstone into the synced store — a entry with `locator: null`, marked with a fresh Lamport timestamp. The tombstone propagates to Bob's side via the same sync that delivered the original entry.

From Bob's terminal:

```sh
endo eval 'E(s).list()' s:alice
# [ 'alice-handle' ]
endo eval 'E(s).lookup("start-fob")' s:alice
# undefined
```

The start fob is gone from Bob's view. If he kept a reference to it in his eval scope (say, the `fob` name he created above), and he tries to use it:

```sh
endo eval 'E(f).start()' f:fob
# error — the underlying formula is no longer reachable
```

The exact error depends on whether the daemon has garbage-collected the formula yet, but the practical effect is the same: Bob's fob is dead.

Importantly, Alice's car is untouched:

```sh
# Alice's terminal
endo eval 'E(c).start()' c:my-car
# vroom!
```

The car was never the thing being shared. The start fob was. Revoking the fob does not revoke the car. This is why we built the fob as a distinct capability in step 6 — so that the unit of revocation matches the unit of grant.

**Sidebar — delegation is unstoppable.** Before Alice revoked, suppose Bob had passed the fob to Charlie. (`endo send charlie 'Here is @fob.'`, assuming Bob also had a peer relationship with Charlie.) Alice cannot prevent this. In a capability system, delegation is structural: if Bob can use it, Bob can pass it on. What Alice can do is what she just did: revoke the source. When the fob's tombstone propagates, every downstream copy — Bob's, Charlie's, and any further hops — dies along with it. Revocation cascades. Delegation prevention does not exist. The right defence against worrying about delegation is not to try to forbid it, but to grant narrow capabilities that you are willing to see propagate, and to mint per-grantee wrappers when you want independent revocation. We will build the per-grantee wrapper pattern in tutorial 3.

## 10. Tear down

When you're done, stop both daemons:

```sh
# Alice's terminal
endo stop

# Bob's terminal
endo stop
```

If you want a clean slate before the next tutorial:

```sh
# in each terminal
endo purge
```

Or just delete the home directories:

```sh
rm -rf $HOME/endo-alice $HOME/endo-bob
```

## What you now know

- **Daemons are peer hosts.** Each daemon owns its own pet store, its own workers, and its own connections. Two daemons on the same machine are no more connected than two daemons on different continents; what binds them is the invitation protocol.
- **Invitations are one-shot capabilities.** An invitation is a sealed envelope: a single-use slot for someone to introduce themselves to you. Once accepted, it cannot be accepted again. The locator can be passed in the clear; the freshness, not the secrecy, is what gates the introduction.
- **The synced pet store is the shared name space.** When two peers establish a relationship, each gets a synced store named after the other. Writes from either side propagate; deletions propagate as tombstones; conflicts resolve under a deterministic CRDT.
- **Revocation cuts the source, not the leaves.** Alice cannot pluck specific copies of a capability out of the network. She can only cut the formula that minted them, and watch every downstream reference die together.

## What's next

In tutorial 3, we'll address the obvious follow-up: Alice wants to revoke Bob's access specifically, without affecting Charlie's. This is the per-grantee wrapper pattern. We'll build a tiny attenuator factory that mints a fresh wrapper per grantee, with a kill switch that only takes out that one grantee's copy. The same Far + closure tooling from tutorial 1; new pattern.
