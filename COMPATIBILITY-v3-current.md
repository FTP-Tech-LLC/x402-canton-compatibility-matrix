# Compatibility Matrix

What versions of the runtime, the Canton platform, the Daml contracts,
the x402 wire protocol, and the published packages work together. This is
the reference for "will version X of the client talk to version Y of the
facilitator on network Z", and the checklist to re-run before any major
platform upgrade.

Last updated: 2026-10-23. Package versions are npm `latest` on that date;
wire fields match the live `GET https://facilitator.ftptech.xyz/supported`.
The section 1 toolchain rows were last checked on 2026-07-05.

## 1. Runtime and toolchain

| Layer | Required | Verified in production |
|---|---|---|
| Node.js | `>=22` (root `engines`) | v22.23.1 on `node:22-alpine` |
| Package manager | pnpm 9.12.0 (pinned via `packageManager`, installed by corepack) | same; the facilitator `Dockerfile` also corepack-pins `pnpm@9.12.0` |
| PostgreSQL | 16.x | `postgres:16` on the mainnet-standalone box and pay-proxy; `postgres:16-alpine` on the devnet/testnet + local/prod facilitator composes |
| Docker base image | `node:22-alpine` (build + runtime stages) | same |
| TypeScript | 5.6 | 5.6.3 |
| Test runner | vitest 2.1 | 2.1.8 |
| Lint | eslint 9 / typescript-eslint 8, `--max-warnings 0` | same |

The one Node exception: `@ftptech/canton-x402-mcp` declares `node >=18`
because MCP hosts install it standalone. Everything else inherits the
root `>=22`.

## 2. Canton platform, Daml, and Splice

| Component | Version | Notes |
|---|---|---|
| Canton participant | 3.6 | the primary participant (node 2) runs Canton 3.6 since 2026-10-21; node 1 is the secondary, its version is unconfirmed. Canton 3.6 disables `/v2/updates/transaction-by-id` and the top-level ACS `filter`/`verbose` fields; 3.7 removes them. Anything that calls a participant directly needs `@ftptech/x402-canton-ledger` 0.6.2 or later (section 7) |
| Splice | 0.9.3 | primary participant and the sv-2 Scan. Splice 0.9.0 removed Scan v0/v1 `holdings/state`; the facilitator reads Scan v2 |
| Daml SDK (contracts) | 3.4.11 | pinned in every `daml.yaml` and in CI; minimum 3.4.11 (`3.4.0` does not exist) |
| Daml-LF target | 2.1 (`--target=2.1`) | required for the Canton 3.4/3.5 line |
| OpenJDK (to build a DAR) | 21 | Canton 3.4+ will not start on 17 |

### Daml packages

The `transfer-factory` money path uses **no custom DAR**. It settles
entirely through Splice token-standard interfaces plus the Amulet
`TransferPreapproval`, all resolved at runtime via `#package-name`
references, so the stack survives Splice package upgrades without a
recompile. (The legacy `x402-direct` DAR that backed `allocation-direct`
has been retired: its `daml.yaml` is gone; only stale build artifacts
linger under `daml/x402-direct/`.)

The Daml SDK / LF / JDK rows above are the toolchain for the repo's
remaining auxiliary DARs (the provider / marker-sim / escrow / cip56-test
packages under `packages/daml*` and `daml/`), none of which the money
path uploads.

At runtime the transfer-factory path binds these Splice interfaces
(suffix-matched against the Scan-resolved templateIds), never a
repo-built package:

| Interface (via `#package-name`) | Used for |
|---|---|
| `Splice.Api.Token.TransferInstructionV1:TransferInstruction` | the settled transfer; the payer's relay-prepared `TransferFactory_Transfer` resolves through this interface |
| `Splice.AmuletRules:AmuletRules` → choice `AmuletRules_CreateTransferPreapproval` | merchant self-provisions its `TransferPreapproval` |
| `Splice.Amulet:Amulet` | payer holdings + the facilitator's fee inputs |
| `Splice.Amulet:TransferPreapproval` | the live merchant preapproval that lets the incoming transfer resolve `direct` |

## 3. x402 wire protocol

| Property | Value |
|---|---|
| x402 version | 2 |
| Scheme | `exact` (only) |
| Headers | `PAYMENT-REQUIRED` (402 challenge), `PAYMENT-SIGNATURE` (payment), `PAYMENT-RESPONSE` (settlement). The v1 `X-PAYMENT` header is not supported |
| 402 challenge | `accepts[]` carried in the `PAYMENT-REQUIRED` header (base64 JSON) |
| `accepts[].extra` | `assetTransferMethod`, `feePayer`, `synchronizerId`, `instrumentId {admin, id}`, `executeBeforeSeconds`, `memo` (optional) |
| Payload carriage | inline only (`/supported` advertises `carriages: ["inline"]`) |
| Payload fields | `assetTransferMethod`, `preparedTransaction` (base64 gzip), `preparedTxHash`, `signature`, `hashingSchemeVersion` (optional). There is no `payer` field; the facilitator derives the payer from the signed transfer. `payer` appears only in the `/verify` and `/settle` responses |
| Asset | `CC` (also accepts `canton-coin` and the `<admin>::Amulet` form) or a CIP-56 token as `<admin>::<id>` plus `extra.instrumentId` (section 3.1) |
| Amount encoding | atomic integers on the wire, 10^10 units per token for every instrument (1 CC = `"10000000000"`, 0.0025 USDCx = `"25000000"`) |
| Header size | every hop must accept a `PAYMENT-SIGNATURE` header of at least 16 KiB (a real one is about 8-10 KiB). nginx: `large_client_header_buffers 4 16k;` |

The upstream x402 spec (`scheme_exact_canton.md`, x402-foundation/x402
#2634) covers Canton Coin only. CIP-56 tokens are an FTP facilitator
extension carried in `extra.instrumentId`.

### 3.1 Instruments on MainNet

All instruments use 10 decimals. Do not use 6-decimal USDC units.

| Symbol | Admin | Id | In `/supported` |
|---|---|---|---|
| CC | `DSO::1220b1431ef217342db44d516bb9befde802be7d8899637d290895fa58880f19accc` | `Amulet` | implied, not listed |
| USDCx | `decentralized-usdc-interchain-rep::12208115f1e168dd7e792320be9c4ca720c751a02a3053c7606e1c1cd3dad9bf60ef` | `USDCx` | admin + instrumentId |
| CBTC | `cbtc-network::12205af3b949a04776fc48cdcc05a060f6bda2e470632935f375d1049a8546a3b262` | `CBTC` | admin + instrumentId |
| USDXLR | `excellar-issuer::12203d1e36930ee0e3fbb898add7e222a47ae9d2a5f0f6187e3a446ea32f871ce2ca` | `USDXLR` | admin + instrumentId |
| cETH | `rails-cethMain-1::12200350ba6e96e3b701c3048b5aa013a8c1c08833e8ebf54339cff581055c29003a` | `cETH` | admin + instrumentId |
| eXAG | `ember-silver::1220ad9c097ae75192b4716854fa413aa8582c86a62b16714663cdab55e025cc79c4` | `eXAG` | admin only |
| eXAU | `ember-gold::1220ad9c097ae75192b4716854fa413aa8582c86a62b16714663cdab55e025cc79c4` | `eXAU` | admin only |

A payer paying in a token uses `canton-agent-wallet pay --asset <SYM>` and
must list the instrument in `CANTON_AGENT_PAYABLE_INSTRUMENTS`
(`<admin>|<id>,...`). See section 6 for the minimum versions.

### Wire history (the breaking boundaries)

| Date | Change | Package boundary |
|---|---|---|
| 2026-06-29 | scheme `exact-canton` renamed to `exact` | core/client 0.3.0 |
| 2026-07-09 | `payload.payer` dropped (a legacy `payer` is ignored) | client 0.6.1 |
| 2026-08-23 | inline carriage only; the stash `submissionRef` carriage removed | core/client/agent-wallet 1.0.0 |

A client on the `exact-canton` string will not match a facilitator that
advertises only `exact`. A client that sends the stash carriage is
refused with `invalid_exact_canton_missing_proof`. Production advertises
`exact` with the inline carriage only.

## 4. Payment method (`assetTransferMethod`)

The facilitator speaks ONE method: **`transfer-factory`**. The legacy
`allocation-direct`, `external-party-amulet-rules` (v1) and
`cip56-transfer-factory` methods have been removed from the type union;
`transfer-factory` is the sole `exact`-scheme settlement path.

| Method | Sponsored gas | Client nonce | Merchant preapproval | Settle mechanism |
|---|---|---|---|---|
| `transfer-factory` | yes (facilitator relays) | none (per-holding + `executeBefore` deadline) | yes | payer signs a relay-prepared `TransferFactory_Transfer`; the facilitator relays it (interactive ExecuteSubmission) in ONE tx; no custom DAR, no custody |

The merchant MUST hold a live `TransferPreapproval`. It self-provisions
one with `canton-agent-wallet preapproval`: its OWN key, single
controller (`actAs=[party]`, `provider = receiver = party`), no operator
token, so the incoming transfer resolves `direct` and settles in a
single facilitator-relayed transaction. Without it the transfer resolves
to a two-step Pending and `/settle` fails closed with
`invalid_exact_canton_preapproval_missing`.

`/supported` advertises `transferMethods: ["transfer-factory"]` and
`carriages: ["inline"]`.

## 5. Networks

Auth is enforced per network. The faucet is a single env-gated feature
(`CANTON_X402_FAUCET_ENABLED`, default OFF), NOT a per-network code branch;
the faucet column below is intended availability, not a hard switch.

| Network | `transfer-factory` | Faucet (intended) | Auth (enforced) |
|---|---|---|---|
| LocalNet | yes | n/a | unsafe HMAC |
| DevNet | yes | yes | OIDC |
| TestNet | yes | yes | OIDC |
| MainNet | yes | no | OIDC |

## 6. Published packages

All under the `@ftptech` scope on npm, Apache-2.0. Internal dependencies
use the `workspace:^` protocol, which resolves to a caret range at publish
time so patch releases propagate to fresh installs.

| Package | Version | Depends on (internal, published range) | Role |
|---|---|---|---|
| `x402-canton-core` | 1.4.0 | (none) | wire types for the `exact` scheme |
| `x402-canton-ledger` | 0.6.2 | (none) | JSON Ledger API v2 + Scan client |
| `x402-canton-client` | 1.2.2 | core ^1.4.0, ledger ^0.6.0 | buyer-side payment scheme |
| `x402-canton-express` | 0.2.6 | core ^1.2.0 | merchant middleware for Express |
| `x402-canton-next` | 0.2.6 | core ^1.2.0 | merchant middleware for Next.js |
| `canton-agent-wallet` | 2.4.0 | core ^1.4.0, client ^1.2.1, ledger ^0.5.1 | self-custody wallet CLI and payer library (`makePayingFetch`) |
| `canton-x402-mcp` | 2.3.0 | agent-wallet ^2.3.0 | MCP server wrapping the wallet |

Third-party pins live in the package that actually uses them:

- **facilitator** (private, not published): `fastify ^5.1.0`, `pg ^8.21.0`, `undici ^7.1.0`, `pino ^9.5.0`, `prom-client ^15.1.3`.
- **agent-wallet**: `@canton-network/core-tx-visualizer` and `@canton-network/core-ledger-proto` at `~1.4.0` (tilde: patch drift only), `undici ^7.1.0`.
- **ledger**: `@canton-network/core-tx-visualizer ~1.4.0`.
- **mcp**: `@modelcontextprotocol/sdk ^1.29.0`, `zod ^3.25.76`.
- **pay-proxy** (private): shares `fastify ^5.1.0`, `pg ^8.21.0`, `undici ^7.1.0`.

The facilitator and pay-proxy services are not published to npm; they
ship as Docker images built from the same workspace.

### Minimum versions against the current facilitator

| Use | Minimum | Why |
|---|---|---|
| Any payment | `x402-canton-client` 1.0.0, `canton-agent-wallet` 1.0.0, `canton-x402-mcp` 1.0.0 | inline carriage only; older clients default to the removed stash carriage |
| Merchant middleware | `x402-canton-express` / `x402-canton-next` 0.2.3 | earlier versions miss the payment-gate and settle-replay fixes |
| CIP-56 token payments | `canton-agent-wallet` 1.4.3 (core 1.2.1) | earlier versions refuse to sign current registry-token transfers |
| CBTC | `canton-agent-wallet` 2.3.2 and core 1.3.1 | CBTC changed its reward beneficiary on 2026-10-14 |
| Direct participant access on Canton 3.6 | `x402-canton-ledger` 0.6.2 | `eventFormat` ACS reads and `update-by-id` |

The wallet and MCP talk only to the relay, so the ledger floor applies to
code that calls a participant directly.

## 7. JSON Ledger API v2 surface

The facilitator + ledger client use a small slice of the JSON Ledger
API v2. The endpoint-specific body shapes matter and are easy to get
wrong. The shapes below are the ones that work on Canton 3.6.

| Endpoint | Body shape / note | Used for |
|---|---|---|
| `/v2/state/ledger-end` | offset read | ledger-end before an interactive execute / preapproval |
| `/v2/state/active-contracts` | filter inside `eventFormat` (`filtersByParty`, `verbose`), accepted since Canton 3.3; the top-level `filter`/`verbose` fields are rejected on 3.6. Large parties are read in `?limit=N` slices (200 per read). Ledger 0.6.1 or later | ACS reads (verify, fee-input selection) |
| `/v2/interactive-submission/prepare` | prepare | transfer-factory pay + merchant self-preapproval prepare |
| `/v2/interactive-submission/execute` | execute; requires `deduplicationPeriod` (`{Empty:{}}`) | transfer-factory settle relay + self-preapproval commit |
| `/v2/updates/update-by-id` | `TRANSACTION_SHAPE_ACS_DELTA` with a wildcard filter per requesting party by default; `TRANSACTION_SHAPE_LEDGER_EFFECTS` for full effects. Replaces `/v2/updates/transaction-by-id`, which Canton 3.6 disables. Ledger 0.6.2 or later | the settle funds-moved gate (a **participant** read, not Scan) |
| `/v2/commands/submit-and-wait-for-transaction` | wrapped: `{commands: {commandId, actAs, commands, ...}}` | non-interactive submit (legacy facilitator-as-provider preapproval create) |

Not on the money path anymore: the Scan `getUpdateById` read
(`/api/scan/v2/updates/...`) that backed the removed cip56 verify path has
no live caller. And `submit-and-wait-for-transaction-tree` is not used
anywhere; it is listed only as an upgrade tripwire (deprecated in 3.4,
removed in 3.5, and common in older Canton snippets, so any external
script still calling it breaks on a 3.5 participant). Run the ledger-API
compat checklist in [MAINTENANCE-PLAYBOOK.md](MAINTENANCE-PLAYBOOK.md)
before any participant major upgrade.

## 8. Support and upgrade policy

- Published packages follow semver. Internal dependencies use the
  `workspace:^` protocol (caret at publish); a minor bump of `core`
  reaches dependents on their next install.
- `@canton-network/*` are tilde-pinned (`~1.4.0`) in `agent-wallet`:
  patch drift only, no automatic minor upgrades.
- The money path ships **no custom DAR**: it binds Splice interfaces by
  `#package-name`, so it rides Splice package upgrades without a
  recompile. Re-vet only if a Splice interface's package name or a bound
  choice signature (`AmuletRules_CreateTransferPreapproval`,
  `TransferInstruction`) changes. The auxiliary DARs still target LF 2.1
  across the Canton 3.4.x/3.5.x lines.
- A scheme-name or envelope-field change is a breaking change and gets a
  major bump on `core` plus a coordinated facilitator redeploy, because
  it moves the `exact` boundary in section 3.
