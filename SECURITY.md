# Security Policy

zStellar is a privacy payments application built on Stellar Testnet. It is
**unaudited research software: testnet only, never use it with real funds.**
Report anything you find, but assume no bug bounty, SLA, or safe-harbour
guarantee beyond the constraints below.

## Supported versions

Only the `main` branch of [zstellar-labs/zstellar](https://github.com/zstellar-labs/zstellar)
is maintained. There are no supported release branches.

## Reporting a vulnerability

Please report privately, not in a public issue:

- Preferred: GitHub's private vulnerability reporting —
  https://github.com/zstellar-labs/zstellar/security/advisories/new
- Alternatively, open a minimal public issue asking a maintainer to open a
  private channel, without describing the vulnerability itself.

Include the affected file and line, the impact, and a minimal reproduction. Do
not include a working exploit against a funded account. We aim to acknowledge a
report within a few days; this is a volunteer project, so please be patient.

## Threat model

- **Testnet only.** The contract addresses in `frontend/src/lib/stellar/config.ts`
  point at Stellar Testnet. Do not deploy this code against mainnet without an
  independent audit; the privacy guarantees are inherited from Nethermind's
  Stellar Private Payments PoC and are not verified here.
- **The relayer key is money.** `/api/relay` signs and pays for Private Transfer
  and Private Withdraw submissions with a server-side Stellar secret
  (`RELAYER_SECRET`). Whoever holds that secret can spend the relayer account's
  balance. On testnet the loss is testnet XLM only — treat the key as a real
  secret anyway.
- **The relayer is public.** `/api/relay` is an unauthenticated route: it is
  rate-limited and origin-restricted, but it still signs on behalf of any
  same-origin caller. If you find a way to make it sign something outside the
  pool `transact` call, that is a valid report.
- **Client-side proving.** Note keys, amounts, and blindings are derived from a
  Freighter signature and never leave the device; proofs are produced in a Web
  Worker with OPFS-backed storage. A bug that leaks these secret inputs is
  high severity.

## Secrets handling

- `RELAYER_SECRET` is a Stellar secret seed (`S...`). It is server-only — it has
  no `NEXT_PUBLIC_` prefix, so Next.js never inlines it into the browser bundle,
  and it is read only by `frontend/src/app/api/relay/route.ts`.
- The secret is written to `frontend/.env.local` by
  `frontend/scripts/setup-relayer.mjs`. That file is gitignored; **never commit
  it, never force-add it, and never paste the seed into source, comments, docs,
  issues, or chat.** Set it as a secret environment variable on the host instead
  (see `docs/DEPLOYMENT.md`).
- CI runs a `secret-scan` job that fails when a `RELAYER_SECRET=S...`-shaped
  value or a bare `S<55 base32 chars>` seed appears in a tracked file.
- If a seed is ever committed or shared, rotate it: generate a new keypair with
  `node scripts/setup-relayer.mjs`, fund it, update
  `NEXT_PUBLIC_RELAYER_ADDRESS` and the host's `RELAYER_SECRET`, and stop using
  the old account.
- `NEXT_PUBLIC_RELAYER_ADDRESS`, `NEXT_PUBLIC_STELLAR_RPC_URL`,
  `NEXT_PUBLIC_STELLAR_HORIZON_URL` and `STELLAR_RPC_UPSTREAM` are not secrets.
  The full variable list is in `frontend/.env.example`.

## Never use real funds

`frontend/public/engine/DISCLAIMER.txt` states the PoC constraint that zStellar
inherits: this is research and educational software, unaudited, and it must not
be used to move real value. Any deployment against mainnet is out of scope for
this policy and needs independent security review and key-management work first.
