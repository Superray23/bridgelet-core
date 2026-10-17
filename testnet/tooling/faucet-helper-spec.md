# Spec: Faucet Helper — Fund and Initialise a Fresh Test `EphemeralAccount` in One Step

## Status

**Spec only. Nothing in this document is implemented.** See `testnet/tooling/decision-log.md` for why this directory holds specifications rather than runnable scripts.

## Problem

Every manual testnet experiment starts with the same three-step dance, and every newcomer gets it slightly wrong:

1. Create a Stellar account and **fund it with Friendbot** (XLM is required before the account exists on-ledger, and before it can pay any fee).
2. `stellar contract deploy` a new `ephemeral_account` WASM instance.
3. `stellar contract invoke ... initialize` with five arguments — two of which (`expiry_ledger` and `recovery_address`) are easy to get wrong and **cannot be corrected afterwards**.

Then, to do anything useful, a fourth step: fund the new contract with a SEP-41 token, because `record_payment()` only records bookkeeping — if the account has no balance, the eventual sweep fails with `TransferFailed`.

That is four commands, roughly ten seconds of RPC latency, and two irreversible parameters. A tester running this fifty times is fifty opportunities to type a bad `recovery_address`.

A single helper that combines Friendbot funding and `EphemeralAccount::initialize()` into one step — optionally including SEP-41 token funding — removes this class of error entirely.

> Funding behaviour, Friendbot limits, and the workarounds for throttling are documented in `testnet/config/friendbot.md` (and summarized for quick account funding in `testnet/faucet/README.md`). That document is the reference; this is a proposal for a tool layered on top of it.

## Scope

**In scope:** create → fund → deploy → initialise, with correct-by-construction parameter handling, and print a summary an integrator can copy.

**Out of scope:**

- Recording payments. The tester chooses the asset and amount; guessing would be worse than useless.
- Sweeping. Different tool, different risk profile.
- Managing Friendbot rate limits beyond surfacing them clearly.
- Anything that touches contracts not owned by this deployment.

---

## Interface

### Invocation

```bash
bridgelet-faucet [options]
```

### Options

| Option | Default | Purpose |
|---|---|---|
| `--expiry <ledgers \| 5m \| 1h \| 1d \| 1w>` | `1h` | Expiry, by ledger count or human duration. See below. |
| `--recovery <G...>` | caller's address | Recovery address. **Immutable after init** — confirm before deploying. |
| `--admin <G...>` | caller's address | Upgrade admin. Immutable after init. |
| `--controller <C...>` | shared `SweepController` ID | `authorized_controller`. Must be a real controller. |
| `--creator <G...>` | caller's address | `creator`; also authorises `initialize()`. |
| `--identity <name>` | caller's active identity | Stellar identity to operate as. |
| `--token <C...>:<amount>` | none | Optional SEP-41 funding, repeatable. |
| `--wasm <path>` | built `ephemeral_account.wasm` | WASM to deploy. |
| `--dry-run` | off | Print the full plan and exit. Deploys nothing. |
| `--json` | off | Machine-readable output. |
| `--rpc-url <URL>` | `https://soroban-testnet.stellar.org` | RPC endpoint. |

### Output — human

```text
✔ funded          testnet-harness  GABC…  (10000 XLM)
✔ deployed        CDEF…
  recovery        GABC…          (your address)
  controller      CBEU…          (shared SweepController)
  admin           GABC…
  expiry_ledger   17894512       (+720 ≈ 1h from 17893792)
✔ initialized     expiry_ledger 17894512

  ephemeral account   CDEF…
  recovery address    GABC…
  controller          CBEU…
  admin               GABC…
  expires at ledger   17894512  (~1h)

Copy this block into your test:
  EPHEMERAL_ID=CDEF…
  RECOVERY_ADDRESS=GABC…
  SWEEP_CONTROLLER_ID=CBEU…
```

The copyable block matters. The point is not that the helper runs the commands; it is that it **emits the resulting identifiers in a form that can be pasted into a test without retyping them.**

### Output — JSON

```json
{
  "identity": "testnet-harness",
  "ephemeral_id": "CDEF...",
  "recovery_address": "GABC...",
  "authorized_controller": "CBEU...",
  "admin": "GABC...",
  "expiry_ledger": 17894512,
  "funding": [
    { "asset": "CA3D5...", "amount": "2500000000" }
  ]
}
```

Note `amount` as a **string**. Token amounts are `i128`; JSON numbers lose precision above 2^53.

### Exit codes

| Code | Meaning | Typical CI reaction |
|---|---|---|
| `0` | Success (account funded, deployed, and initialised) | Proceed with testing |
| `1` | Contract returned an error (`InvalidExpiry`, `AlreadyInitialized`, etc.) | Inspect contract status; abort run |
| `2` | Usage error (invalid flag syntax, malformed address/contract ID) | Fix configuration |
| `3` | RPC unreachable or timed out | Retry with exponential backoff per `testnet/config/rpc-fallback.md` |
| `4` | Friendbot throttled (HTTP 429) | Backoff for 60 seconds or rotate proxy/IP per `testnet/config/friendbot.md` |
| `5` | Prerequisite missing (WASM file not found, identity not found) | Run `scripts/build.sh` or configure test identity |

---

## Behaviour

### Stage 1 — Fund

```bash
stellar keys create --global "$IDENTITY"        # if it does not exist
stellar keys fund "$IDENTITY" --network testnet # Friendbot
```

A Stellar account must exist and be funded on-ledger before it can appear as a contract argument or pay a fee. If the identity already exists and is funded, skip — but **check** rather than assume, because an unfunded account fails later at `initialize()` with a confusing auth error.

Report Friendbot throttling distinctly from other failures. It means "slow down and retry", not "something is wrong". The limits and the workarounds are in `testnet/config/friendbot.md`.

### Stage 2 — Deploy

```bash
stellar contract deploy --wasm "$WASM" --network testnet --source "$IDENTITY"
```

If the WASM is missing, say so and print the build command. Do **not** silently build or substitute a different artefact — deploying the wrong WASM produces an account that behaves unexpectedly much later.

### Stage 3 — Initialise

Five arguments, all required, all in one call:

```bash
stellar contract invoke --id "$EPH_ID" --network testnet --source "$IDENTITY" -- initialize \
  --creator "$CREATOR" \
  --expiry_ledger "$EXPIRY" \
  --recovery_address "$RECOVERY" \
  --authorized_controller "$CONTROLLER" \
  --admin "$ADMIN"
```

`initialize()` sets every one of these exactly once. A second call returns `AlreadyInitialized` (1) and there is **no setter for any of them**. The two most consequential:

- **`recovery_address`** — the destination of the whole recovery path. Wrong means funds are unrecoverable by you.
- **`expiry_ledger`** — must be strictly in the future (`InvalidExpiry`, code 5) or the account is dead on arrival; must be far enough out that ledger timing does not surprise you.

The helper's real value is refusing to let either be wrong silently. If `--recovery` was not passed, default to the caller's own address and **echo it prominently** — a recovery address you did not consciously choose is the kind of thing nobody notices until it matters.

### `--expiry` parsing

| Input | Ledgers added | Approximate wall time |
|---|---|---|
| `5m` | 60 | ~5 min |
| `1h` | 720 | ~1 hr |
| `1d` | 17280 | ~24 hr |
| `1w` | 120960 | ~7 days |
| `3600` (raw) | 3600 | ~5 hr |

Read the live ledger first — `stellar network current-ledger` — and add. Never derive the expiry from a local clock; `expiry_ledger` is a **ledger sequence number**, and `is_expired()` uses `current >= expiry`, inclusive, so the boundary ledger is already expired. The arithmetic and its rationale are in `testnet/config/expiry-ledger-testing.md`.

### Stage 4 — Optional token funding

```bash
--token CA3D5…:2500000000
```

`record_payment()` is **bookkeeping only** — it does not move value. Without an actual balance, the sweep later reaches `transfers::execute_transfers()` and reverts with `TransferFailed`, taking the whole multi-asset sweep down with it. Offering `--token` here prevents the most common "it initialised fine but the sweep fails" report.

Do not call `record_payment()` automatically. The tester picks the asset and amount.

### `--dry-run`

Print every command, every resolved address, and the computed `expiry_ledger`, then exit without touching the network. This is how you check a `--recovery` value before committing to it.

---

## Failure Handling

| Stage | Failure | Handling |
|---|---|---|
| Fund | Friendbot throttled | Report as throttle with a retry hint; do not treat as fatal |
| Fund | Account exists, unfunded | Re-fund, continue |
| Deploy | WASM not found | Stop; print the build command |
| Deploy | Insufficient XLM | Report balance and required amount |
| Init | `InvalidExpiry` (5) | Computed expiry ≤ current ledger; recompute and retry once |
| Init | `AlreadyInitialized` (1) | Should be impossible on a fresh deploy; report loudly, as it means a hash/deploy collision |
| Init | auth failure | Caller did not authorise. The identity must be the `creator` |
| Init | `NotInitialized` (2) | Deployed but not initialised — check the WASM is really `ephemeral_account` |
| Token fund | Insufficient balance | Name the asset; do not retry blindly |
| Any | RPC unreachable | Distinct exit code from contract errors |

---

## Ordering and Idempotency

The order is not arbitrary: fund → deploy → initialise → token-fund. Each stage depends on the previous one having landed on-chain, so the helper must **wait for confirmation between stages** rather than firing them optimistically.

A helper that does not wait produces accounts that initialise against an unconfirmed deploy, which fails in a way that looks like a contract bug.

**Not idempotent.** Every run creates a new Stellar account and a new contract instance. There is no "reuse" mode, deliberately: `EphemeralAccount` is single-use by design (one payment per asset, one sweep ever, immutable recovery address), so reusing one instance across experiments is a reliable source of confusing failures. Reuse the funded *Stellar* account, not the contract instance.

---

## Ethics on a Shared Deployment

The testnet deployment is shared. This helper creates state, so:

- **Never call `upgrade()`.** Admin-gated and destructive for every integrator.
- **Never call `update_authorized_destination()`.** Creator-gated and it changes routing for every account on that controller.
- **Do not sweep or expire the long-lived accounts** in `testnet/registry/known-test-accounts.md`.
- **Fund only your own accounts.**
- **Prefer short expiries** so your accounts retire rather than accumulating.
- Do not add a batch mode. `AccountFactory::batch_initialize` exists for programmatic fan-out and **swallows per-account error detail** (`success: false, error: None`) — see `testnet/runbooks/account-factory-batch-failure.md`. A one-account helper that is obviously correct is more useful than a batch helper that fails invisibly.

The broader set of shared-state hazards is in `testnet/security/known-testnet-abuse-patterns.md`.

---

## Non-Goals

| Not doing | Why |
|---|---|
| Running or deploying in this repository | This is a specification for external consumer/tester tooling, not an in-repo deployment script (`scripts/` handles repo builds/CI) |
| Automatic SEP-41 payment recording (`record_payment`) | Tester selects asset and amount; automated bookkeeping without tester intent creates confusing test state |
| Automated sweeping | Sweeping is the responsibility of `SweepController` and consumer payout workers; separating funding/init from sweeping preserves distinct failure domains |
| Bypassing Friendbot rate limits | Friendbot limits are enforced network-wide by the Stellar Development Foundation; helper surfaces 429 errors cleanly rather than attempting abusive evasion |
| Modifying or sweeping existing long-lived accounts | Preserves shared test accounts documented in `testnet/registry/known-test-accounts.md` |
| Batch account deployment wrapping `AccountFactory` | `AccountFactory::batch_initialize` drops per-account error diagnostics (`error: None`); one-step helper prioritizes clear error attribution for single accounts |
| Reusing `EphemeralAccount` instances | Ephemeral accounts are single-use by design (one payment per asset, single sweep lifecycle); reusing contract instances introduces state contamination |

---

## Integration

### Manual Tester Workflow

Manual testers previously had to execute and coordinate multiple disjoint commands:

```bash
# Old manual workflow (multiple commands, manual arithmetic, copy-paste risk):
stellar keys fund test-identity --network testnet
CURRENT_LEDGER=$(curl -s https://soroban-testnet.stellar.org -X POST -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"getLatestLedger"}' | jq .result.sequence)
EXPIRY=$((CURRENT_LEDGER + 720))
EPH_ID=$(stellar contract deploy --wasm target/wasm32-unknown-unknown/release/ephemeral_account.wasm \
  --source test-identity --network testnet)
stellar contract invoke --id "$EPH_ID" --source test-identity --network testnet -- initialize \
  --creator "$CREATOR_ADDR" \
  --expiry_ledger "$EXPIRY" \
  --recovery_address "$RECOVERY_ADDR" \
  --authorized_controller "$SWEEP_CONTROLLER_ID" \
  --admin "$ADMIN_ADDR"

# New single-step workflow with bridgelet-faucet:
bridgelet-faucet --expiry 1h --recovery "$RECOVERY_ADDR" --controller "$SWEEP_CONTROLLER_ID"
```

### Script and Harness Integration (JSON Output)

Automated test harnesses or local bash scripts can consume the machine-readable output directly:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Provision fresh ephemeral account in one step
SETUP_JSON=$(bridgelet-faucet --json --expiry 30m --recovery "$RECOVERY_ADDR")

EPHEMERAL_ID=$(echo "$SETUP_JSON" | jq -r .ephemeral_id)
EXPIRY_LEDGER=$(echo "$SETUP_JSON" | jq -r .expiry_ledger)

echo "Ready for test run with contract: $EPHEMERAL_ID (expires at ledger $EXPIRY_LEDGER)"
```

### In CI Pipelines

For CI integration test runs, use `--dry-run` to validate parameter construction before spending testnet funds, and set a bounded expiry:

```bash
# Validate parameters in CI
bridgelet-faucet --dry-run --expiry 15m --recovery "$CI_RECOVERY_ADDR" --json

# Execute initialization in CI with timeout and distinct exit code handling
if ! OUTPUT=$(bridgelet-faucet --json --expiry 15m --recovery "$CI_RECOVERY_ADDR" --rpc-url "$RPC_URL"); then
  EXIT_CODE=$?
  if [ "$EXIT_CODE" -eq 4 ]; then
    echo "Friendbot rate-limited; retrying after backoff..."
    sleep 60
    OUTPUT=$(bridgelet-faucet --json --expiry 15m --recovery "$CI_RECOVERY_ADDR" --rpc-url "$RPC_URL")
  else
    echo "Faucet helper failed with exit code $EXIT_CODE"
    exit "$EXIT_CODE"
  fi
fi
```

---

## Quick Reference (added 2026-10-17)

One-step provisioning replaces the error-prone multi-step manual script:

```bash
bridgelet-faucet --expiry 1h --recovery <G_RECOVERY_ADDR> --controller <C_SWEEP_CONTROLLER>
```

### Quick Diagnostic & Parameter Checklist

| Parameter | Required | Immutable? | Rules to Remember |
|---|---|---|---|
| `creator` | Yes | After init | Authorizes the initialization transaction; must match signing identity |
| `expiry_ledger` | Yes | After init | Must be strictly > current on-chain ledger sequence (`InvalidExpiry` error 5 if `<= current`) |
| `recovery_address` | Yes | **Permanently** | Funds transfer here if account expires; cannot be changed after init |
| `authorized_controller` | Yes | After init | Only this contract can trigger sweeps; must be a valid `SweepController` ID |
| `admin` | Yes | After init | Upgrade authority; cannot be altered once initialised |

### Common Pitfalls & Solutions

1. **Friendbot HTTP 429 (`Friendbot throttled`)**: Wait 60 seconds before retrying, or configure a proxy/alternative testnet identity (see `testnet/config/friendbot.md`).
2. **`InvalidExpiry` (Code 5)**: Do not use wall-clock seconds. The tool reads `getLatestLedger` from the RPC and adds the offset (e.g. `+720` for ~1 hour).
3. **`AlreadyInitialized` (Code 1)**: Can only occur if attempting to re-initialize an existing contract instance. Fresh deployments always start uninitialized.
4. **`TransferFailed` during subsequent sweep**: `record_payment()` is bookkeeping only. Use `--token <C...>:<amount>` to supply real SEP-41 token balance prior to testing sweep execution.

---

## Open Questions

- Should it emit a Host.bot payment-request URL instead of calling Friendbot directly? Friendbot's canonical UX is the URL, and it avoids rate-limit interactions entirely.
- Should `--recovery` be mandatory with no default? The default is convenient; requiring it is safer. Lean toward mandatory, since the whole point is eliminating silent misconfiguration.
- Should the helper verify the WASM hash it is about to deploy against `testnet/registry/wasm-hash-reference.md`?
- Worth a `--json` schema stable enough to depend on from CI? If so, it needs a version field.

---

## Related Documentation

- `testnet/config/friendbot.md` — funding behaviour and rate limits
- `testnet/faucet/README.md` — quick testnet faucet guide and curl syntax
- `testnet/config/expiry-ledger-testing.md` — expiry arithmetic, and why not to use a clock
- `testnet/config/soroban-cli-identity-setup.md` — identity conventions used here
- `testnet/registry/known-test-accounts.md` — long-lived accounts not to disturb
- `testnet/runbooks/account-factory-batch-failure.md` — why batch creation is not wrapped
- `testnet/examples/create-and-fund-ephemeral-account.md` — the manual flow this replaces
- `docs/api-reference.md` — `initialize` parameter reference

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.1 | 2026-10-17 | Added Non-Goals table, Integration workflows, Quick Reference, corrected initialize argument count (5 args), and clarified Friendbot configuration references |
| 1.0 | 2026-09-26 | Initial faucet helper spec |
