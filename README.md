# OpenZeppelin Role Analyzer

A single-page, **fully client-side** tool that inspects the access-control and
ownership state of a smart contract on **Ethereum, Base, Polygon, or
Arbitrum**, using the public [BlockScout](https://docs.blockscout.com/) API
directly from the browser. No backend, no build step, no libraries.

The entire tool is one self-contained file: [`index.html`](./index.html)
(vanilla HTML + CSS + JS, including a from-scratch `keccak256`).

It understands four common privilege primitives:

- **AccessControl** — `bytes32` roles (`DEFAULT_ADMIN_ROLE`, `MINTER_ROLE`, …)
- **AccessManager** — `uint64` role IDs (`ADMIN_ROLE` = 0, `PUBLIC_ROLE` = max uint64)
- **Ownable** — single `owner`
- **Safe** (Gnosis Safe / Safe{Wallet}) — multisig `owners`, `threshold`, enabled
  `modules`, and `guard`

## What it does

1. **Roles declared by the contract (from the ABI, if available).**
   It fetches the verified ABI via BlockScout. For EIP-1967 / proxy contracts
   it resolves and pulls the **implementation** ABI (or its verified bytecode
   twin), then lists the `bytes32` role constants the interface exposes
   (`DEFAULT_ADMIN_ROLE`, `*_ROLE`, …).

2. **Current account → role assignments (from events).**
   It reconstructs who currently holds each role by replaying the relevant
   events since deployment:
   - AccessControl: `RoleGranted` / `RoleRevoked` / `RoleAdminChanged`
   - AccessManager: `RoleGranted` / `RoleRevoked` / `RoleAdminChanged` /
     `RoleGuardianChanged` / `RoleLabel` (the last supplies human-readable names)
   - Ownable: `OwnershipTransferred`
   - Safe: `SafeSetup` / `AddedOwner` / `RemovedOwner` / `ChangedThreshold` /
     `EnabledModule` / `DisabledModule` / `ChangedGuard` / `ChangedFallbackHandler`.
     The current owner set (rendered as **m-of-n**), enabled modules and guard are
     reconstructed from these. Initial owners come from `SafeSetup` (Safe's
     `setupOwners` emits no per-owner event), and the single address each event
     carries is read whether it is indexed (Safe ≥ 1.4) or in the log data.

   These events are decoded directly from their well-known `topic0` signatures,
   so it works even when the contract — or the implementation behind a proxy —
   is **not verified**.

3. **Audit log.**
   A chronological (oldest → newest) list of every role-update event —
   grants, revocations, admin/guardian changes, labels and ownership transfers —
   with the block, timestamp, a human-readable description, and a link to the
   transaction on the chain's canonical explorer (Etherscan / BaseScan /
   PolygonScan / Arbiscan).

AccessControl role hashes are labelled best-effort by matching against
`keccak256(name)` for a built-in dictionary of common OpenZeppelin role names,
any names found in the ABI, and any custom names you supply. AccessManager roles
are labelled from on-chain `RoleLabel` events (plus the well-known ADMIN/PUBLIC
roles). Unmatched roles are shown by their raw `bytes32` hash or `uint64` ID.

Queries to BlockScout are retried with backoff, and transient failures are
surfaced as a warning so an incomplete result is never mistaken for an empty one.

## Usage

Open the published page (or `index.html` locally), pick a network, paste a
contract address, and click **Analyze**.

Try the bundled example: contract
`0xF35f12c2556706b074d2dCeBF178DEB9003a358b` on **Base**. For a Safe multisig,
try `0xD23CCDc650b2a5709e0Eb3C7e68B9C77c20e361b` on **Base** (the Engineers
Multisig), which resolves to its current owners and threshold.

### Run locally

```bash
python3 -m http.server 8000
# then open http://localhost:8000
```

(Opening the file directly via `file://` also works — all calls are CORS-enabled
BlockScout endpoints.)

## Deployment

Pushes to `main` are published to **GitHub Pages** automatically by
[`.github/workflows/deploy-pages.yml`](./.github/workflows/deploy-pages.yml).

> **One-time setup:** in the repository **Settings → Pages**, set
> **Source = "GitHub Actions"**. The workflow cannot enable Pages by itself.

## Notes & limitations

- Holder lists come from event replay; a query that hits BlockScout's 1000-log
  limit for a single event type is flagged as potentially incomplete.
- Role labels are heuristic: a contract may define a role as something other
  than `keccak256("THE_NAME")`. The raw `bytes32` hash is always shown.
