# Contract redeploy runbook

A pool redeploy touches **five places that must move together**. The last
redeploy updated the code but left `README.md` stale, so follow this order every
time. Skipping a step produces either a silent runtime failure or an invisible
documentation drift.

> Read this before deploying. Changing `CONTRACTS.pool` also **wipes every
> user's local note store** — see [Consequences](#consequences).

## The five update sites

| # | Site | What changes |
|---|---|---|
| 1 | `frontend/src/lib/stellar/config.ts` | `CONTRACTS` — `pool`, `verifier`, `aspMembership`, `aspNonMembership`, `token`, `deployer` |
| 2 | `README.md` | The "Addresses (Stellar Testnet)" table must list every `CONTRACTS` address |
| 3 | `docs/DEPLOYMENT.md` | Only if the deployment story or the endpoint list changes |
| 4 | `frontend/public/engine/js/*_bg.wasm` | The embedded `"deploymentLedger":<ledger>` value |
| 5 | `frontend/src/app/api/rpc/route.ts` | `NEW_DEPLOYMENT_LEDGER` |

`CHANGELOG.md` gets an entry too, describing the new addresses and the OPFS
reset.

## Ordered procedure

1. **Deploy the contracts** with the Soroban CLI (pool, verifier, ASP
   membership, ASP non-membership). Note the resulting addresses and the ledger
   at which the new pool went live.

2. **Update `CONTRACTS`** in `frontend/src/lib/stellar/config.ts` with the new
   addresses. The token SAC address normally does not change.

3. **Patch the WASM deployment ledger.** The vendored bundles have the old
   deployment ledger baked in; run:

   ```bash
   cd frontend
   node scripts/patch-wasm-ledger.mjs
   ```

   The script reads the latest testnet ledger from Soroban RPC and rewrites
   `"deploymentLedger":\d{7}` in `frontend/public/engine/js/web_bg.wasm`,
   `prover-worker_bg.wasm` and `storage-worker_bg.wasm` to
   `latest - 1000`. It aborts if a replacement would change a file's length.

4. **Set `NEW_DEPLOYMENT_LEDGER`** in `frontend/src/app/api/rpc/route.ts` to the
   same ledger the WASM was patched to. If the two disagree, the proxy rewrites
   `startLedger` to a range that does not match the bundle and the
   out-of-range loop the proxy was written to fix comes back.

5. **Update the README address table** so it lists every address in `CONTRACTS`
   verbatim. CI fails the `docs-drift` job otherwise.

6. **Add a `CHANGELOG.md` entry** naming the new addresses and the Merkle-level
   change, and noting the OPFS reset below.

## Verify

```bash
cd frontend
pnpm lint                 # Biome check
pnpm build                # Next build
node -e "const s=require('fs').readFileSync('src/lib/stellar/config.ts','utf8');console.log(s.match(/[CG][A-Z2-7]{55}/g))"
grep -o '"deploymentLedger":[0-9]\{7\}' public/engine/js/web_bg.wasm | head -1
grep -n 'NEW_DEPLOYMENT_LEDGER' src/app/api/rpc/route.ts
```

Then load the app on testnet and run one Shield followed by one Private
Transfer: the first deposit registers the user in the new ASP tree and may take
up to a minute while the chain syncs.

## Consequences

- **OPFS reset (user-visible).** `maybeResetStorage` in
  `frontend/src/engine/index.ts` compares the stored pool id with
  `CONTRACTS.pool`; on a mismatch it wipes OPFS and clears every
  `zStellar:asp-registered:*` flag. Every user therefore loses their local note
  cache and must re-derive keys (one Freighter signature) and re-register on
  their next deposit. This is by design — old notes belong to the old pool — but
  it must be announced.
- **Explorer links and docs go stale.** Any README or explorer link pointing at
  the previous pool or verifier address is dead after the redeploy.
- **In-flight transactions fail.** Anything signed against the old pool cannot
  be submitted once `CONTRACTS.pool` changes.
