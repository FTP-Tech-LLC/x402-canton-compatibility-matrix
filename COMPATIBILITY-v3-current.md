# Compatibility Matrix

What versions of the runtime, the Canton platform, the Daml contracts,
the x402 wire protocol, and the published packages work together. This is
the reference for "will version X of the client talk to version Y of the
facilitator on network Z", and the checklist to re-run before any major
platform upgrade.

Snapshot date: 2026-07-05 (transfer-factory-only; allocation-direct
removed). The versions below are what the production deployment is built
and run against, not just what the manifests request.

## 1. Runtime and toolchain

| Layer | Required | Verified in production |
|---|---|---|
| Node.js | `>=22` (root `engines`) | v22.23.1 on `node:22-alpine` |
| Package manager | pnpm 9.12.0 (pinned via `packageManager`, installed by corepack) | same — the facilitator `Dockerfile` also corepack-pins `pnpm@9.12.0` |
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
| Canton participant | 3.5.x line | production runs a `3.5.1-snapshot` build (documented in `docs/STACK.md`; not pinned in any manifest); the JSON Ledger API v2 surface used is stable across the 3.4.x and 3.5.x lines |
| Daml SDK (contracts) | 3.4.11 | pinned in every `daml.yaml` and in CI; minimum 3.4.11 — `3.4.0` does not exist |
| Daml-LF target | 2.1 (`--target=2.1`) | required for the Canton 3.4/3.5 line |
| OpenJDK (to build a DAR) | 21 | Canton 3.4+ will not start on 17 |

### Daml packages

The `transfer-factory` money path uses **no custom DAR**. It settles
entirely through Splice token-standard interfaces plus the Amulet
`TransferPreapproval`, all resolved at runtime via `#package-name`
references, so the stack survives Splice package upgrades without a
recompile. (The legacy `x402-direct` DAR that backed `allocation-direct`
has been retired — its `daml.yaml` is gone; only stale build artifacts
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
| `Splice.Api.Token.TransferInstructionV1:TransferInstruction` | the settled transfer — the payer's relay-prepared `TransferFactory_Transfer` resolves through this interface |
| `Splice.AmuletRules:AmuletRules` → choice `AmuletRules_CreateTransferPreapproval` | merchant self-provisions its `TransferPreapproval` |
| `Splice.Amulet:Amulet` | payer holdings + the facilitator's fee inputs |
| `Splice.Amulet:TransferPreapproval` | the live merchant preapproval that lets the incoming transfer resolve `direct` |

## 3. x402 wire protocol

| Property | Value |
|---|---|
| x402 version | 2 |
| Scheme | `exact` (only) |
| Envelope field names | `assetTransferMethod`, `feePayer`, `payer` (canonical) |
| Asset symbol | `CC` (also accepts `canton-coin` and the `<admin>::Amulet` form) |
| Amount encoding | atomic integers on the wire (1 CC = 10^10) |
| 402 challenge | `accepts[]` carried in the `PAYMENT-REQUIRED` header (base64 JSON) |

### Scheme-name history (the breaking boundary)

The scheme name changed once. This is the single most important
compatibility line for anyone integrating a pinned client:

| Wire era | Scheme string | Package range |
|---|---|---|
| Legacy | `exact-canton` | core/client 0.2.x and earlier |
| Current | `exact` | core/client 0.3.0 and later |

A client on the `exact-canton` string will not match a facilitator that
advertises only `exact`, and vice versa. Both sides must be on the same
era. Production advertises `exact` only.

## 4. Payment method (`assetTransferMethod`)

The facilitator speaks ONE method: **`transfer-factory`**. The legacy
`allocation-direct`, `external-party-amulet-rules` (v1) and
`cip56-transfer-factory` methods have been removed from the type union —
`transfer-factory` is the sole `exact`-scheme settlement path.

| Method | Sponsored gas | Client nonce | Merchant preapproval | Settle mechanism |
|---|---|---|---|---|
| `transfer-factory` | yes (facilitator relays) | none (per-holding + `executeBefore` deadline) | yes | payer signs a relay-prepared `TransferFactory_Transfer`; the facilitator relays it (interactive ExecuteSubmission) in ONE tx — no custom DAR, no custody |

The merchant MUST hold a live `TransferPreapproval`. It self-provisions
one with `canton-agent-wallet preapproval` — its OWN key, single
controller (`actAs=[party]`, `provider = receiver = party`), no operator
token — so the incoming transfer resolves `direct` and settles in a
single facilitator-relayed transaction. Without it the transfer resolves
to a two-step Pending and `/settle` fails closed with
`invalid_exact_canton_preapproval_missing`.

`/supported` advertises `["transfer-factory"]` **unconditionally**. The
settle path itself is gated by `CANTON_X402_TF_ENABLED`; the separate
`CANTON_X402_ADVERTISE_TF` flag exists in config but the current
`/supported` handler does not read it (advertisement is not gated).

## 5. Networks

Auth is enforced per network. The faucet is a single env-gated feature
(`CANTON_X402_FAUCET_ENABLED`, default OFF), NOT a per-network code branch
— the faucet column below is intended availability, not a hard switch.

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

| Package | Version | Depends on (internal) | Role |
|---|---|---|---|
| `x402-canton-core` | 0.6.0 | (none) | wire types for the `exact` scheme |
| `x402-canton-ledger` | 0.1.1 | (none) | JSON Ledger API v2 + Scan client |
| `x402-canton-client` | 0.6.0 | core, ledger | buyer-side payment scheme |
| `x402-canton-express` | 0.2.1 | core | merchant middleware for Express |
| `x402-canton-next` | 0.2.1 | core | merchant middleware for Next.js |
| `canton-agent-wallet` | 0.6.1 | ledger, client | self-custody wallet CLI |
| `canton-x402-mcp` | 0.5.0 | agent-wallet | MCP server wrapping the wallet |

Third-party pins live in the package that actually uses them (the doc
used to attribute all of these to the facilitator — most are wrong):

- **facilitator** (private, not published): `fastify ^5.1.0`, `pg ^8.21.0`, `undici ^7.1.0`, `pino ^9.5.0`, `prom-client ^15.1.3`.
- **agent-wallet**: `@canton-network/core-tx-visualizer` and `@canton-network/core-ledger-proto` at `~1.4.0` (tilde — patch drift only), `undici ^7.1.0`.
- **mcp**: `@modelcontextprotocol/sdk ^1.29.0`, `zod ^3.25.76`.
- **pay-proxy** (private): shares `fastify ^5.1.0`, `pg ^8.21.0`, `undici ^7.1.0`.

The facilitator and pay-proxy services are not published to npm; they
ship as Docker images built from the same workspace.

### Client-to-facilitator compatibility

| Client (`x402-canton-client`) | Facilitator wire | Compatible |
|---|---|---|
| 0.3.0+ (`exact`) | advertises `exact` | yes |
| 0.3.0+ (`exact`) | advertises only `exact-canton` | no |
| 0.2.x (`exact-canton`) | advertises `exact` | no |
| 0.2.x (`exact-canton`) | advertises `exact-canton` | yes |

Rule of thumb: keep the client on 0.3.0 or later against any current
facilitator. The middleware packages (`express`, `next`) follow the same
`exact` boundary through their `core` dependency.

## 7. JSON Ledger API v2 surface

The facilitator + ledger client use a small, stable slice of the JSON
Ledger API v2. The endpoint-specific body shapes matter and are easy to
get wrong.

| Endpoint | Body shape / note | Used for |
|---|---|---|
| `/v2/state/ledger-end` | offset read | ledger-end before an interactive execute / preapproval |
| `/v2/state/active-contracts` | filter by template/interface | ACS reads (verify, fee-input selection) |
| `/v2/interactive-submission/prepare` | prepare | transfer-factory pay + merchant self-preapproval prepare |
| `/v2/interactive-submission/execute` | execute — requires `deduplicationPeriod` (`{Empty:{}}`) | transfer-factory settle relay + self-preapproval commit |
| `/v2/updates/transaction-by-id` | one-tx read of the payer's own projection | the settle funds-moved gate (a **participant** read, not Scan) |
| `/v2/commands/submit-and-wait-for-transaction` | wrapped: `{commands: {commandId, actAs, commands, ...}}` | non-interactive submit (legacy facilitator-as-provider preapproval create) |

Not on the money path anymore: the Scan `getUpdateById` read
(`/api/scan/v2/updates/...`) that backed the removed cip56 verify path has
no live caller. And `submit-and-wait-for-transaction-tree` is not used
anywhere — it is listed only as an upgrade tripwire (deprecated in 3.4,
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
- The money path ships **no custom DAR** — it binds Splice interfaces by
  `#package-name`, so it rides Splice package upgrades without a
  recompile. Re-vet only if a Splice interface's package name or a bound
  choice signature (`AmuletRules_CreateTransferPreapproval`,
  `TransferInstruction`) changes. The auxiliary DARs still target LF 2.1
  across the Canton 3.4.x/3.5.x lines.
- A scheme-name or envelope-field change is a breaking change and gets a
  major bump on `core` plus a coordinated facilitator redeploy, because
  it moves the `exact` boundary in section 3.
