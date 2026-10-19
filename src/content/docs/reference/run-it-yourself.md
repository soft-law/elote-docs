---
title: Run it yourself
description: Build and test the program, run a local validator and the demo script, run the app end to end, and the environment variables each part reads.
---

The program and the demo script are in [soft-law/elote-contracts](https://github.com/soft-law/elote-contracts) (npm at the root); the app is `soft-law/elote-web` and these docs `soft-law/elote-docs` (both Bun). The hackathon submission, with the app and the Meteora launch, is frozen at https://github.com/soft-law/elote-baby/tree/colosseum-2026.

## Toolchain

| Tool | Version |
|---|---|
| Rust | 1.89.0 (`rust-toolchain.toml`) |
| Anchor CLI | 1.1.2 |
| Solana CLI | 3.1.x |
| Node | 20.18 or later |
| Bun | for `elote-web`, the agreements server and these docs |

## Build and test the program

```sh
NO_DNA=1 anchor build            # target/deploy/elote_pool.so and target/idl/elote_pool.json
cargo test -p elote_pool         # unit tests and LiteSVM integration tests against the built .so
npm install
npm run codama                   # regenerates clients/js/src/generated from the IDL
npm run check:client             # fails if the committed client differs from the IDL
npm run typecheck
```

The integration tests run the compiled program in LiteSVM, which moves the clock by months to test installments, late interest and collection.

## Local validator

```sh
mkdir -p .demo-keys && test -f .demo-keys/admin.json || solana-keygen new --no-bip39-passphrase -s -o .demo-keys/admin.json
solana-test-validator --reset \
  --bpf-program BGdJQxxJoHiLH2VS6HAGHe5kQuKKq9dwrPeX7DEEwYyM target/deploy/elote_pool.so \
  --mint "$(solana-keygen pubkey .demo-keys/admin.json)"
```

## Demo script

```sh
npx tsx scripts/demo-pool.ts --cluster local  [--agreement-hash <64 hex>] [--sponsor] [--schedule]
npx tsx scripts/demo-pool.ts --cluster devnet [--agreement-hash <64 hex>] [--sponsor]
```

The script creates a test token and a pool, then runs two investors (10,000 and 5,000 tMXN), a 6,000 line at 12% with the given agreement hash, the draw, the borrower's repayment of principal and the sponsor's payment of the 720 interest, then a partial redemption. It prints every signature with an explorer link and a table of NAV, supply and value per share before and after.

| Flag | Effect |
|---|---|
| `--agreement-hash` | Hash recorded on the line; a placeholder if absent. |
| `--sponsor` | The sponsor co-signs the opening and is named on the line. |
| `--schedule` | Also opens a 600 line in six installments, misses one past its due date and grace, and repays it with the next: late interest first, then interest, then principal. |
| `--interval-seconds` | Seconds between installments. One month by default; 20 seconds on local, since a validator's clock cannot be moved forward. |
| `--grace-seconds` | The pool's grace period. 0 by default; 5 with `--schedule` on local. |

Wallets are generated into `.demo-keys/` (ignored by git). Only addresses are printed.

## Pool app

The app is the `elote-web` repository (private). It depends on the published client
`@softlaw/elote-pool-client`, so it needs no checkout of the program to build.

```sh
git clone git@github.com:soft-law/elote-web.git && cd elote-web
bun install
cp .env.example .env.local   # then fill in the VITE_* values
bun run dev                  # http://localhost:5173, the app at /app
bun test tests
bun run build
```

### End to end against a local validator

```sh
ELOTE_CONTRACTS_DIR=/path/to/elote-contracts PLAYWRIGHT_FROM=/path/with/playwright bun run e2e
```

`scripts/e2e.mjs` in `elote-web` starts a throwaway validator with the built program, runs the demo script, gives a fresh test wallet test tokens and an approved line, builds the app against that pool, and drives Chromium with Playwright: connect, pool figures, deposit 100 tMXN, redeem 10 shares, activity, draw and repay 50 on the wallet's line, then each page at 375 px. It stops every process on exit, also on Ctrl-C, and deletes the test build. Needs `solana-test-validator` on `PATH` with port 8899 unused, an `elote-contracts` checkout with the program built and `npm install` done (`ELOTE_CONTRACTS_DIR`, default `../contracts`), Bun, and Playwright with Chromium.

## Agreements server

The agreements server is Soft.Law's `softlaw-backend` (private), which serves the faucet, the
agreement summaries and the MCP endpoint for every Soft.Law product. Its setup, API keys and
environment are documented in that repository.

## Environment variables

Names only; see each `.env.example`. Never put a key in a `VITE_*` variable: every value ends up in the browser.

| Part | Variable | Meaning |
|---|---|---|
| Demo script | `ELOTE_RPC` | Overrides the cluster's RPC URL. |
| | `ELOTE_ADMIN_KEYPAIR`, `ELOTE_INVESTOR_A_KEYPAIR`, `ELOTE_INVESTOR_B_KEYPAIR`, `ELOTE_BORROWER_KEYPAIR`, `ELOTE_SPONSOR_KEYPAIR` | Paths to keypair files instead of `.demo-keys/`. |
| Pool app | `VITE_RPC_URL` | Solana RPC the app reads and sends through. |
| | `VITE_POOL_ADDRESS` | The pool. |
| | `VITE_ASSET_MINT` | The tMXN address; the app refuses a pool holding another asset. |
| | `VITE_API_URL` | Base URL of the agreements server (faucet and agreement summaries). |
| | `VITE_EXPLORER_CLUSTER` | `devnet`, `testnet`, `mainnet-beta` or `localnet`. |
| | `VITE_TEST_WALLET`, `VITE_TEST_WALLET_SECRET` | Test builds only (the e2e run). Leave unset in real builds. |
| Agreements server | see `softlaw-backend` | The faucet key is never the pool admin key. |

## Deployment order

1. Build, tests green, client check clean.
2. Deploy the program to devnet with a dedicated upgrade key, then publish the IDL.
3. Run the demo script on local, then once on devnet; keep that pool and test token.
4. Host the agreements server and faucet.
5. Point the web app at the pool, the test token and the server.

Program and upgrade keypairs are never committed (`target/` is ignored).
