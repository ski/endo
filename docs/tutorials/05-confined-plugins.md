---
title: 05-confined-plugins
group: Documents
category: Guides
---

# The Plugin That Can't Escape: Hardened Compartments

Tutorials 1 through 4 were all code you wrote yourself — you knew what was in the module, you decided what capabilities to pass. The interesting question is what happens when the module comes from someone else. A third-party library. A downloaded plugin. Code you are willing to run but do not fully trust.

The answer in Endo is: you run it in a Hardened JavaScript Compartment, you pass it exactly the capabilities it needs, and nothing else. It cannot reach `fetch`, `process`, the filesystem, or anything it was not given. It cannot mutate shared objects because every primordial — `Array.prototype`, `Object.prototype`, everything — was frozen before the module loaded. Even if the plugin is adversarial, its blast radius is bounded by what you handed it.

This tutorial makes that concrete. You will load a plugin that tries to escape its sandbox in three different ways, watch each attempt fail, and then see the same plugin do its legitimate work through an attenuated capability.

## What you'll need

- Tutorial 1 setup: the daemon running, `my-car` installed, `valet-key` installed (the one that only unlocks).
- The `endo-car-tutorial/` project directory with `@endo/far` and `@endo/harden` installed.
- About 20 minutes.

## 1. The compartment, briefly

When you call `endo make plugin.js`, the daemon bundles `plugin.js` and its local imports, then runs the bundle inside a Hardened JavaScript Compartment. The compartment has:

- All standard JavaScript built-ins — `Array`, `Object`, `Promise`, `Map`, `Set`, and so on — but they are frozen. No one can modify their prototypes.
- The endowments you passed via `--powers`: a guest agent object that the plugin receives as its `powers` argument.
- `E`, `Far`, `makeExo`, `M`, `console`, `harden`, `TextEncoder`, `TextDecoder`, `URL` — the Endo toolkit.

The compartment does NOT have:

- `fetch`
- `process` (or any Node.js global)
- `require`
- `globalThis.XMLHttpRequest`
- Any access to your filesystem, network, or other processes

This is not a blocklist. Those things simply do not exist inside the compartment, the same way a browser's `window.document` does not exist inside a Node.js script. There is nothing to block; the capability was never granted.

## 2. Write the plugin

Create `reporter.js` in `endo-car-tutorial/`:

```js
import harden from '@endo/harden';
import { E } from '@endo/far';

// ---- Escape attempt 1: network access ----
try {
  const res = await fetch('https://example.com');
  console.log('fetch worked:', res.status); // should not print
} catch (err) {
  console.log('fetch blocked:', err.message);
}

// ---- Escape attempt 2: process / environment ----
try {
  const secret = process.env.HOME;
  console.log('process.env worked:', secret); // should not print
} catch (err) {
  console.log('process blocked:', err.message);
}

// ---- Escape attempt 3: prototype mutation ----
try {
  Array.prototype.evil = () => 'pwned';
  const x = [].evil();
  console.log('prototype mutation worked:', x); // should not print
} catch (err) {
  console.log('prototype mutation blocked:', err.message);
}

// ---- The actual work ----
export const make = powers =>
  harden({
    async report() {
      const result = await E(powers).unlock();
      return `Car status: ${result}`;
    },
  });
```

The three escape attempts run at module load time — before `make` is even called. That way you see the failures immediately when the plugin is instantiated.

## 3. Install with no powers

Install the plugin with `--powers NONE` — no capabilities at all:

```sh
endo make reporter.js --name reporter --powers NONE
```

The daemon's log (your second terminal running `endo log -f`) should show three lines from the plugin:

```
fetch blocked: fetch is not defined
process blocked: process is not defined
prototype mutation blocked: Cannot add property evil, object is not extensible
```

All three escape attempts failed cleanly. The plugin is running — its module-level code executed — but it had no path out of its compartment.

Now call `report()`:

```sh
endo eval 'E(r).report()' r:reporter
```

This throws because `powers` is the least-authority object (no methods). That's expected — the plugin declared it needs a car but we gave it nothing. The point is that the three escape attempts were already blocked before we even called the method.

## 4. Install with the valet key

Remove the old formula and reinstall with the valet key as powers:

```sh
endo remove reporter
endo make reporter.js --name reporter --powers valet-key
```

The same three failures appear in the log. Now call `report()`:

```sh
endo eval 'E(r).report()' r:reporter
# Car status: click! unlocked
```

The plugin unlocked the car through the capability you gave it and returned a string. It did not start the engine — not because of a rule that said "don't call start", but because the valet key simply does not have a `start` method. The attenuation from tutorial 1 is still doing its job, now composing with the compartment's isolation.

Try calling start explicitly through the plugin's powers:

```sh
endo eval 'E(r).tryStart()' r:reporter
# Error: target has no method "start"
```

(The plugin does not expose `tryStart` — that would throw a different error. But if you added it and tried `E(powers).start()` inside the plugin, it would get the same "no method" error. The compartment does not add any new restriction here; the capability was already attenuated.)

## 5. What the compartment and the capability together achieve

You now have two independent security layers working together:

- **The capability** (valet key) limits what the plugin can do with the car. Attenuation is the outer boundary — the plugin's access was bounded before it even ran.
- **The compartment** limits what the plugin can do with the rest of the world. Isolation is the inner boundary — the plugin cannot reach anything that was not explicitly given to it.

Either layer alone would be incomplete. Capability without compartment: the plugin could call `fetch()` and exfiltrate data even if it could only unlock the car. Compartment without capability: the plugin is isolated but could still start the engine if you gave it the full car. Together, they are orthogonal: the capability shapes the interface, the compartment contains the blast radius.

**Sidebar — frozen primordials and supply-chain attacks.** One of the most common JavaScript attack vectors is prototype pollution: a malicious package somewhere in your dependency tree adds a property to `Array.prototype` or `Object.prototype`, and every object in the process is now affected. In a Hardened JavaScript environment, this is impossible — the primordials were frozen at lockdown time, before any untrusted code ran. `Array.prototype.evil = ...` throws. No auditing required; the invariant is structural.

**Sidebar — `Date.now()` and `Math.random()`.** After lockdown, `Date.now()` returns `NaN` and `Math.random()` throws. This is intentional: these are covert channels. A sufficiently creative adversary can use timing information or random seeds to infer things about the host system. In a strict Hardened JavaScript environment, determinism is the default. If your plugin legitimately needs randomness or current time, you pass it a clock or an RNG capability explicitly — and you control the resolution.

## 6. Bundling and what it means for imports

When the daemon bundles `reporter.js`, it walks the static import graph and includes every local module the file imports. Imports from installed npm packages in your project's `node_modules` are also included, but only if the bundler can resolve them. Imports of Node.js built-ins (`fs`, `path`, `crypto`, etc.) will fail at bundle time or be treated as empty modules — they are not available inside the compartment.

This means: a plugin that `import`s `fs` will fail to load, not fail silently at runtime. The error surfaces early, at instantiation, before any damage can be done.

It also means: if a dependency of your plugin somewhere deep in `node_modules` tries to call `require('child_process')`, it will get an error when the plugin loads. The compartment boundary is the whole bundle, not just the top-level file.

## 7. Tear down

```sh
endo stop
endo purge
```

## What you now know

- **Hardened compartments contain blast radius.** A plugin running in a compartment cannot reach `fetch`, `process`, the filesystem, or any other capability it was not explicitly given. There is no blocklist; the capability was simply never granted.
- **Frozen primordials block prototype pollution.** After lockdown, no module can add properties to shared prototypes. Supply-chain attacks that rely on prototype mutation fail at the point of mutation.
- **Attenuation and confinement compose.** The capability limits what the plugin can do with its endowments. The compartment limits what the plugin can do with everything else. They are orthogonal and independent.
- **Bundling surfaces import errors early.** A plugin that imports unavailable modules fails at instantiation, not silently at runtime.
- **`Date.now()` and `Math.random()` are disabled by default.** Pass them explicitly as capabilities if the plugin legitimately needs them.

## What's next

You have now seen the full capability stack: objects, attenuation, revocation, per-grantee wrappers, requests, and confined execution. The natural next step is to assemble these pieces into a small but real plugin system — a host that manages a catalog of capabilities, accepts plugins from untrusted sources, routes their requests through an approval workflow, and revokes access when they misbehave. That is tutorial 6.
