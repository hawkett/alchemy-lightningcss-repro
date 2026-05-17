# alchemy + lightningcss bundling repro

Minimal reproduction of an [alchemy](https://alchemy.run) v2 bundling failure that surfaces in any pnpm workspace using vite 7 + tailwind v4 (or any other path that makes pre-1.32 lightningcss reachable).

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

Two-app pnpm workspace:

- **`apps/worker/`** — uses alchemy. Pins `vite: "7.3.1"` as a devDep (no explicit lightningcss anywhere).
- **`apps/styled/`** — has `@tailwindcss/postcss@4.1.18` as a devDep. Nothing else. The `styled` app's `lightningcss` is never imported by `apps/worker/` source — but tailwind v4 brings it in via `@tailwindcss/node` (which has lightningcss `1.30.2` as a regular dep at this tailwind version), and pnpm satisfies vite 7's optional lightningcss peer from that.

The trigger is structural:

1. **Vite 7 has `lightningcss` as an optional peer dep** (`^1.21.0`). When something else in the workspace declares lightningcss (directly or transitively), pnpm wires it into vite's resolution.
2. **Lightningcss < 1.32.0 contains `require('../pkg')`** — a runtime-conditional WASM-fallback that only executes if `CSS_TRANSFORMER_WASM=1` is set. The `pkg` directory only ships in the separate `lightningcss-wasm` package, not in standard installs.
3. **Rolldown does static analysis** — it doesn't know the require is dead code; it just sees the string `'../pkg'` and tries to resolve a file that doesn't exist.
4. **Alchemy's `Cloudflare.state()` reconciliation path bundles its own state-store Worker source**, which transitively reaches `loadVite()` (`await import("vite")`). Rolldown follows it, walks into vite's static reference to lightningcss, hits the `../pkg` line, fails.

This is the "I didn't declare lightningcss anywhere, why is my deploy broken?" surprise — typical of pnpm workspaces where one app has tailwind v4 and another, unrelated app uses alchemy.

### Why vite 7 specifically

Vite 8 (released 2026-03-12) moved lightningcss from an optional peer to a regular dependency AND bumped the minimum version to `^1.32.0` — the version that incidentally dropped the `../pkg` line as part of [an unrelated visitor-API refactor](https://github.com/parcel-bundler/lightningcss/pull/1170). So workspaces that have already moved to vite 8 don't hit this — vite 8 pulls in a fixed lightningcss directly.

Real-world workspaces stuck on vite 7 typically have it because of a peer constraint from another dep (e.g., `@shopify/hydrogen-react@2025.10.0` only accepted `vite: ^5.1.0 || ^6.2.1`, which pnpm satisfies with vite 7 via the path of least resistance), or because the lockfile hasn't been refreshed since vite 8 came out. Pinning `vite: 7.3.1` here mimics that situation.

### Why this specific tailwind version

`@tailwindcss/postcss@4.1.18` brings in `lightningcss@1.30.2` (which still has the `../pkg` line). The latest `@tailwindcss/postcss@4.3.0+` brings in `lightningcss@1.32.0` (no `../pkg`). The repro pins to `4.1.18` to keep the bug reproducible — any tailwind v4 version below `4.3.0` would work the same way.

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
