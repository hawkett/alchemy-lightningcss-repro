# alchemy + lightningcss bundling repro

Minimal reproduction of an [alchemy](https://alchemy.run) v2 bundling failure in pnpm monorepos where lightningcss is reachable in the workspace.

**TL;DR**: running `alchemy cloudflare bootstrap` (or any operation that triggers state-store reconciliation) fails at the rolldown bundling step with `Could not resolve '../pkg' in lightningcss/node/index.js`, because alchemy bundles its own state-store Worker source (`lib/Cloudflare/StateStore/Api.js` → `lib/Cloudflare/Workers/Worker.js` → `loadVite()` → vite → lightningcss).

## The bug

Same shape as [rolldown/tsdown#212](https://github.com/rolldown/tsdown/issues/212) — rolldown statically follows vite's reference into `lightningcss/node/index.js`, which contains a dead-code WASM fallback `require('../pkg')` that doesn't resolve.

The chain alchemy hits:

```
[UNRESOLVED_IMPORT] Could not resolve '../pkg' in lightningcss/node/index.js
imported by:
  - vite/dist/node/chunks/node.js
  - vite/dist/node/index.js
  - lib/Cloudflare/Workers/Worker.js
  - lib/Cloudflare/StateStore/Api.js
  - virtual:alchemy-entry
```

The rolldown maintainer's prescribed fix for this shape ([sxzz](https://github.com/rolldown/tsdown/issues/212#issuecomment-2869371776)) is to mark `lightningcss` (and `fsevents`) as `external` in the bundler config.

## Reproduce

Requires a Cloudflare account + API token.

```bash
pnpm install
cd apps/worker
CLOUDFLARE_API_TOKEN=<your-token> \
CLOUDFLARE_ACCOUNT_ID=<your-account-id> \
  pnpm run bootstrap-repro
```

The `bootstrap-repro` script runs:

```
alchemy cloudflare bootstrap --worker-name alchemy-state-store-lightningcss-repro
```

(Custom `--worker-name` so this doesn't conflict with an existing state-store on your account.)

You should see the `Could not resolve '../pkg'` error, with the chain shown above.

## Why it triggers here but not elsewhere

The bundler only reaches into lightningcss's source if **lightningcss is resolvable somewhere in the workspace**. This repo arranges that via `apps/styled/`, which has `@tailwindcss/postcss` as a devDependency — tailwind v4 pulls lightningcss in through `@tailwindcss/node`. The `styled` app's lightningcss is never imported in `apps/worker/` source, but pnpm's workspace-wide resolution makes it reachable when bundling.

In alchemy's own dev workspace (or any workspace without tailwind v4 / another lightningcss-pulling dep), the bundler still walks the same `loadVite()` code path but lightningcss resolution simply fails-silently / tree-shakes, and the bundle succeeds. The trigger is **workspace lightningcss reachability**, not anything specific to the user's alchemy config.

### The `pnpm.overrides.lightningcss` pin is load-bearing

The root `package.json` pins `pnpm.overrides.lightningcss: "1.30.1"`. **The repro depends on this pin** — empirically, removing it causes pnpm to resolve to a different lightningcss arrangement where the alchemy bundle path no longer reaches into lightningcss's source, and the bundle succeeds. The failing chain still goes through vite 8 either way; the difference is in how pnpm wires up lightningcss relative to the bundle's resolution context. We haven't fully traced the mechanism, but the pin is required to reproduce.

## Cleanup

Bootstrap fails during the bundling phase (before any Cloudflare API writes), so **no resources are created on your CF account**. The only artifact is `apps/worker/.alchemy/state/CloudflareStateStore/alchemy-state-store-lightningcss-repro/` with partial state files locally. Delete that directory to fully reset:

```bash
rm -rf apps/worker/.alchemy
```

## Notes

- Alchemy version pinned to `2.0.0-beta.39`. Effect + `@effect/platform-node` pinned to `4.0.0-beta.66`.
- The same chain trips for `state: Cloudflare.state()` deploys when the state-store is in **"owner" mode** (local state-store state files present) — alchemy reconciles the state-store on every deploy, which bundles its own state-store source the same way. The `bootstrap` command is the cleanest way to demonstrate the bundle failure standalone.
- A user-side workaround for the `Cloudflare.state()` reconciliation case is to delete `apps/<your-app>/.alchemy/state/CloudflareStateStore/` so alchemy treats the state-store as opaque external infrastructure (uses the auth token from `~/.alchemy/credentials/` without reconciliation). Doesn't help with `bootstrap` — fresh accounts can't avoid creating a state-store somehow.
- The proper fix lives in alchemy's rolldown config: `external: ['lightningcss', 'fsevents']` per the upstream rolldown maintainer's recommendation. That would protect every bundling path inside alchemy regardless of what's reachable in the consuming workspace.
