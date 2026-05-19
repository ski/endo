# Synced Pet Store

The synced pet store is a CRDT-backed name → locator map shared between two
Endo daemons. It is the storage substrate for the capabilities that Alice and
Bob agree to share with each other after one of them accepts the other's
invitation.

This document is the introductory tour. Code lives in
[`src/synced-pet-store.js`](src/synced-pet-store.js); the formula plumbing is
in [`src/daemon.js`](src/daemon.js); type definitions are in
[`src/types.d.ts`](src/types.d.ts). Tests are
[`test/synced-pet-store.test.js`](test/synced-pet-store.test.js) (unit) and
[`test/synced-pet-store-integration.test.js`](test/synced-pet-store-integration.test.js)
(two-daemon end-to-end).

## What problem it solves

A regular `pet-store` is local-only: it maps pet names to formula identifiers
inside a single daemon. It has no notion of who else might want to see those
names, and no way to converge if two parties edit the same map.

When two daemons form a peer relationship via invite/accept, they each need
the *same* view of "the names Alice has granted Bob" and "the names Bob has
disclaimed." Either party may add a tombstone (deletion) at any time. The
sync may be delayed (offline, partition). When both sides reconnect, they
must end up agreeing without anyone needing to be "the server."

That is what `synced-pet-store` provides.

## The CRDT

Each entry is a triple:

```js
{
  locator: string | null,  // null = tombstone
  timestamp: number,       // Lamport clock value
  writer: string,          // node id that produced this entry
}
```

The merge rule for two competing entries `a` and `b` for the same key is, in
order:

1. **Higher timestamp wins.** Lamport-clock ordering. Every local write
   increments `localClock` by one; merging a remote state advances
   `localClock` to at least the maximum timestamp observed.
2. **Tombstone bias on tie.** If timestamps are equal, a `locator: null`
   entry beats a non-null one. Revocation is sticky.
3. **Lexicographically greater `writer` wins.** Deterministic tiebreak when
   both sides wrote at the same Lamport tick from different nodes.

See `mergeEntry` and `mergeState` in
[`src/synced-pet-store.js`](src/synced-pet-store.js).

This is a **last-writer-wins map with tombstones** — a single-key LWW
register, lifted to a key-value map. The tombstone bias makes revocation
*monotone*: once a name is removed, no concurrent write at the same logical
time can resurrect it.

## Roles: grantor and grantee

Each replica is created with a role:

- The **grantor** is the side that issued the invitation. They are the only
  side allowed to `write()` new name → locator entries. In the Alice-invites-Bob
  flow, Alice's replica is the grantor.
- The **grantee** is the side that accepted. They can `remove()` entries
  (disclaim a granted capability) but cannot `write()` new ones. Bob's
  replica is the grantee.

Either side can remove. Only one side can add. This is enforced at the
in-process API level (`write()` throws on a grantee) and reflected in the
CRDT only by who happens to call which method — the merge rules themselves
are symmetric.

## Lifecycle: invite, accept, sync

A two-daemon walkthrough, distilled from
[`test/synced-pet-store-integration.test.js`](test/synced-pet-store-integration.test.js):

1. **Alice invites Bob.** `E(hostA).invite('bob')` produces an invitation
   formula. Alice's daemon eagerly formulates a `synced-pet-store` in the
   `grantor` role and writes the guest-handle locator into it under the pet
   name `'bob'`. The synced store's formula id (not the raw handle's id) is
   what gets written into Alice's local pet store under `bob`.
2. **Bob accepts.** `E(hostB).accept(invitationLocator, 'alice')`. The
   invitation's `accept` handler runs on Alice's side, returns Alice's
   synced-store formula number. Bob's host then formulates its own paired
   `synced-pet-store` in the `grantee` role, with `remoteStoreNumber`
   pointing at Alice's, and writes the grantee store under `alice` in Bob's
   local pet store.
3. **Both sides now have a synced replica** addressable as `hostA.lookup('bob')`
   and `hostB.lookup('alice')`. Calling `.list()`, `.write()`, `.remove()`,
   etc. on either of these is the API surface for granted capabilities.
4. **Sync** is initiated by either side calling, in a loop:
   - `getState()` and `getLocalClock()` on the local replica
   - `mergeRemoteState(remoteState, remoteClock)` on the peer
   - `acknowledgeRemoteClock(peerClock)` to flag what's been seen
   - optionally `pruneTombstones()` once both sides have acked past the
     tombstone's timestamp

The sync mechanism is currently driven explicitly by tests; no scheduled
gossip runs yet. Wiring the periodic exchange to the existing peer channel is
the next-obvious step (see `TODO.md` and `peer-retention-roots.md` in the
designs).

## Persistence layout

Each replica owns a directory under the daemon's state path. Inside:

- `clock.json` — `{ localClock, remoteAckedClock }`. Written atomically.
- `names/<petName>.json` — one file per CRDT entry, including tombstones.
  Written via `write-then-rename` through a hidden `.tmp.<hex>` sibling, so
  partial writes from a crash never present a half-formed JSON file. Stale
  `.tmp.*` files are swept on startup.

All writes go through a `makeSerialJobs()` queue, so one replica never has
two concurrent writers to the same file.

## How it plugs into the formula graph

`synced-pet-store` is a new formula type
(`packages/daemon/src/formula-type.js`) with two dependencies: the `peer`
formula (which keeps the store alive as long as the peer relationship is
alive) and a `store` dependency (currently aliased to the peer; the slot is
there for future split between transport and storage).

The formula's evaluator in `daemon.js` wraps the in-memory store with a
`Far('SyncedPetStore', ...)` so the remote peer can call its methods over
CapTP. The wrapper does two extra jobs the inner store doesn't:

- It maintains formula-graph **GC edges**. When a locally-owned formula id is
  written into the synced store, an edge is added so the local formula
  stays retained as long as the synced entry exists. On removal or merge,
  the edge is dropped (with a scan to make sure no other entry still
  references that formula).
- It translates `Set<string>` (from `mergeRemoteState`) to a plain array so
  it crosses the CapTP boundary, since `Set` is not passable.

On daemon startup, `synced-pet-store` formulas are rehydrated alongside
regular pet stores, and their local-id GC edges are reconstructed by
scanning the on-disk state.

## API summary

```js
interface SyncedPetStore {
  write(petName, locator): Promise<void>;       // grantor only
  remove(petName): Promise<void>;
  has(petName): boolean;
  lookup(petName): string | undefined;
  list(): PetName[];                            // sorted, no tombstones
  getState(): Record<string, SyncedEntry>;      // full state, incl. tombstones
  getLocalClock(): number;
  getRemoteAckedClock(): number;
  mergeRemoteState(state, clock): Promise<Set<string>>;
  acknowledgeRemoteClock(clock): Promise<void>;
  pruneTombstones(): Promise<string[]>;
  followChanges(): AsyncGenerator<{ key, entry }>;
}
```

`followChanges()` yields the current state first, then deltas — useful for a
UI or a reactive consumer.

## What it does *not* do (yet)

- **No automatic gossip.** Sync is manual: someone has to call
  `mergeRemoteState` from the outside. The integration tests do it inline.
- **No conflict surface for human resolution.** Conflicts collapse silently
  under the LWW rules.
- **No encryption or signing.** The transport (peer / CapTP) handles that
  layer; the store itself is in plaintext on disk.
- **`store` dependency is a placeholder.** The formula declares a `store`
  field separate from `peer`, but at present both fields point at the peer
  formula. The slot exists so a future change can detach storage from the
  peer channel.

## Where to start reading

- [`src/synced-pet-store.js`](src/synced-pet-store.js) — ~390 lines, the
  whole CRDT and persistence layer in one file. Read top-to-bottom.
- [`test/synced-pet-store.test.js`](test/synced-pet-store.test.js) — single-
  process unit tests covering merge rules, tombstones, persistence,
  pruning.
- [`test/synced-pet-store-integration.test.js`](test/synced-pet-store-integration.test.js) —
  two-daemon end-to-end. Read `'invite/accept creates synced-pet-store pair
  on both sides'` first, then `'synced stores converge via manual sync'`.
- `formulateSyncedPetStore` and the `'synced-pet-store'` formula evaluator
  in [`src/daemon.js`](src/daemon.js) — the integration points with the
  rest of the daemon.
