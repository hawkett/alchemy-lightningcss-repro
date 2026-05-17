# alchemy + lightningcss bundling repro

Minimal reproduction of an [alchemy](https://alchemy.run) v2 bundling failure that surfaces in pnpm workspaces with a frontend framework constraining vite to ≤7 (e.g. SvelteKit ≤2.52, hydrogen-react ≤2026.4.1, older Storybook builders) alongside tailwind v4.

**TL;DR**: `alchemy cloudflare bootstrap` (or any operation that triggers state-store reconciliation) fails at the rolldown bundling step with `Could not resolve '../pkg' in lightningcss/node/index.js`, because alchemy bundles its own state-store Worker source (`lib/Cloudflare/StateStore/Api.js` → `lib/Cloudflare/Workers/Worker.js` → `loadVite()` → vite → lightningcss). Rolldown follows the vite → lightningcss chain statically and trips on a dead-code WASM fallback `require('../pkg')` inside lightningcss < 1.32.0.

## The bug

Same shape as [rolldown/tsdown#212](https://github.com/rolldown/tsdown/issues/212) — rolldown statically follows vite's reference into `lightningcss/node/index.js`, which contains a dead-code WASM fallback `require('../pkg')` that doesn't resolve.

The chain alchemy hits:

```
[UNRESOLVED_IMPORT] Could not resolve '../pkg' in lightningcss/node/index.js
imported by:
  - vite/dist/node/chunks/config.js
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

## The scenario this models

Two-app pnpm workspace, **no explicit vite declaration anywhere**:

- **`apps/worker/`** — uses alchemy. That's it. No vite pin.
- **`apps/styled/`** — has `@sveltejs/kit@2.52.0` and `@tailwindcss/postcss@4.2.1` as devDeps. Nothing else.

What happens at install time:

1. **`@sveltejs/kit@2.52.0` declares `vite: "^5.0.3 || ^6.0.0 || ^7.0.0-beta.0"` as a peer** — explicitly rejects vite 8. (SvelteKit 2.53.0+ widened to include vite 8, but 2.52.0 was the last release before that change, published 2026-02-15.)
2. **alchemy peers `vite: "^8.0.7"` (optional)** and its `@distilled.cloud/cloudflare-vite-plugin` dep peers `vite: "^7.0.0 || ^8.0.0"`. Both accept vite 7.
3. **pnpm has to pick a single vite version** that satisfies the most peers. Vite 7 is the only version that satisfies SvelteKit + cloudflare-vite-plugin simultaneously. Picks **`vite@7.3.1`**.
4. **`@tailwindcss/postcss@4.2.1` brings in `@tailwindcss/node@4.2.1`** which has `lightningcss@1.31.1` as a regular dep. Pnpm wires that lightningcss into vite 7's optional peer.
5. **Lightningcss 1.31.1 still contains `require('../pkg')`** — a runtime-conditional WASM-fallback that never executes at runtime, but rolldown does static analysis and trips on it.
6. **Alchemy's `Cloudflare.state()` reconciliation path bundles its own state-store Worker source**, which transitively reaches `loadVite()` (`await import("vite")`). Rolldown follows it, walks into vite's static reference to lightningcss, hits the `../pkg` line, fails.

This is the "I didn't declare vite or lightningcss anywhere, why is my deploy broken?" surprise — the kind of transitive-resolution mystery that's hard to debug without tracing the full peer chain.

### Why vite 7 gets picked (and how vite 8 fixed it upstream)

Vite 8 (released 2026-03-12) moved lightningcss from an optional peer to a regular dependency AND bumped the minimum version to `^1.32.0` — the version that incidentally dropped the `../pkg` line as part of [an unrelated visitor-API refactor](https://github.com/parcel-bundler/lightningcss/pull/1170). So workspaces on vite 8 don't hit this.

Workspaces stay on vite 7 when at least one dep's peer rejects vite 8. SvelteKit 2.52.0 is one example; older versions of `@shopify/hydrogen-react` (≤ 2026.4.1, `^5.1.0 || ^6.2.1`), `@sveltejs/kit` ≤ 2.52, older Storybook vite builders, and others fit the same pattern. Lockfile drift (workspace set up before vite 8 shipped) is another common path to the same end state.

### Why this specific tailwind version

`@tailwindcss/postcss@4.2.1` brings in `lightningcss@1.31.1` (which still has `../pkg`). `@tailwindcss/postcss@4.2.2+` bumped to `lightningcss@1.32.0` (no `../pkg`). The repro pins to `4.2.1` to keep the bug reproducible — any tailwind v4 version below `4.2.2` would work the same way.

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
- Another structural fix would be tightening `@distilled.cloud/cloudflare-vite-plugin`'s vite peer from `^7.0.0 || ^8.0.0` to just `^8.0.0`. Alchemy already peers `vite: ^8.0.7` at the top level, so the plugin allowing vite 7 is the loose link letting older vite leak into the alchemy bundle chain.
