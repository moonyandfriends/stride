# Remediation spec: INJ/OSMO stuck unbondings after the v32 validator rebalance

Status: **draft for core-team review.** Touches user funds — do not execute any state migration without a full mainnet-fork replay.

## 1. Summary

After the **v32 upgrade (v32.0.0, 2026-04-24)**, `UpdateValidatorWeights` rebalanced weights
across every host zone, setting a large batch of previously-active validators to weight 0
(806 zeroed entries). Weight-0 validators are fully drained by the unbonding logic, and that
full-drain load surfaced **two independent latent bugs**:

- **Injective** — full-balance `MsgUndelegate` of a *slashed* validator overshoots on-chain
  shares (18-dec `SharesToTokensRate` rounds above the exact rate), so the host rejects it with
  `invalid shares amount` (ABCI 18). The atomic ICA reverts and strands every redemption in it.
- **Osmosis** — during the same window the delegation ICA channel was closing on packet
  timeouts; some undelegations **executed on Osmosis but their acks were lost**, so
  `UndelegateCallback` never decremented `Delegation` (phantom records) nor swept the returned
  tokens (stranded liquid) nor burned the stTokens.

## 2. Confirmed on-chain state (public-verifiable, as of 2026-07-01)

Injective (`injective-1`): delegations reconcile exactly with on-chain; no phantom. The only
issue is the shares-rounding overshoot → fixed forward by the shares-safety PR.

Osmosis (`osmosis-1`), delegation ICA `osmo1npfl…9zm5e`:

| Quantity | OSMO |
|---|---|
| `TotalDelegations` (tracked) | 4,780,238.5 |
| On-chain delegated | 4,165,815.5 |
| **Phantom (tracked − on-chain)** | **614,423.1** |
| — cosmostation `osmovaloper1clpqr4…88n0y4` | 258,638.3 (on-chain 0) |
| — chorus_one `osmovaloper15urq2d…9wh9czc` | 231,057.0 (on-chain 0) |
| — pryzmstakedrop `osmovaloper14n8pf9…amq3e5` | 124,727.8 (tracked 185,390.6 − on-chain 60,662.8) |
| **Stranded liquid in delegation ICA** | **623,806.6** |
| Liquid − phantom (≈ uncounted reinvestment rewards) | +9,383.6 |
| Stuck redemptions (11 epochs, `UNBONDING_RETRY_QUEUE`) | 682,426.5 |

Current retry pattern (9-day archive window): `MsgDelegate` 185/0, `MsgWithdrawDelegatorReward`
189/12, **`MsgUndelegate` 0 success / all fail**, targeting exactly the 3 phantom validators.

Key point: the redemption rate is currently ~correct **by coincidence** — the phantom
over-count in `TotalDelegations` (+614k) nearly offsets the liquid the rate ignores
(`UpdateRedemptionRateForHostZone` does not count the delegation ICA's raw balance). Any fix
must preserve this.

## 3. Forensic gap (why the migration can't be built from public data)

The three numbers do **not** reconcile 1:1 (stuck 682.4k ≠ liquid 623.8k ≠ phantom 614.4k),
because some stuck epochs **never executed** (error-ack, no liquid) while others **executed but
lost their ack** (produced liquid). Allocating the liquid to the right redeemers is per-epoch.

Public infra cannot resolve this: the public Stride archive's tx index only retains ~9 days
(`timeout_packet` on the OSMO delegation port = 0 in-window; oldest ack 2026-06-22), so the May
undelegations that produced the liquid are gone. **The core team must derive the per-epoch split
from an internal full archive + the icacallbacks callback data (packet → epoch mapping).**

### Required internal queries
For each stuck OSMO epoch record, determine the historical outcome of its undelegation ICA:
- **timeout ack** (or success ack whose callback errored) → executed on host → tokens are in the
  stranded liquid → this redemption is *already unbonded*.
- **error ack only, never a timeout/success** → never executed → still needs to unbond.

Cross-check against Osmosis archive `complete_unbonding` events crediting the delegation ICA
(amounts + completion times) to confirm the liquid attribution.

## 4. Forward fixes (shipped as separate PRs)

1. **Shares-rounding safety** in `GetUnbondingICAMessages` — buffers full drains of slashed
   validators so the host stops rejecting them. Fixes Injective and prevents recurrence.
2. **`undelegation_failed` event** in `HandleFailedUndelegation` — makes a stuck retry loop
   alertable instead of silent (both chains).
3. **`CheckDelegationRecordsConsistent`** in BeginBlocker — log-only internal accounting check.

## 5. OSMO state migration (v33 upgrade handler) — algorithm

Inputs (from §3 internal forensics): per stuck epoch, `executed` bool and the executed amount;
plus the per-validator phantom from §2.

Per stuck epoch:
- **Executed epochs** (tokens already in the liquid):
  1. Decrement the phantom validator `Delegation` and `TotalDelegations` by the executed amount.
  2. Advance the host-zone unbonding record to `EXIT_TRANSFER_QUEUE` (set `UnbondingTime` in the
     past) so the existing sweep pays redeemers from the liquid.
  3. Burn the corresponding stTokens.
  - Rate is preserved: numerator −amount and denominator −amount/rate are proportional at the
    current rate (amount = stTokens × rate), so `(N−amount)/(D−amount/rate) == N/D`.
- **Never-executed epochs** (tokens still in real delegations):
  1. Reconcile the phantom validators' `Delegation` down to on-chain (so the retry's split
     targets real validators), decrementing `TotalDelegations` by the same.
  2. Leave the record in the queue; with the shares-safety fix (PR 1) the next unbonding
     succeeds against real delegations. No stToken burn yet.

Residual liquid surplus (~9.4k reinvestment rewards): create a `DELEGATION_QUEUE` deposit record
so it's counted (`undelegatedBalance`) and re-delegated by the normal flow.

### Post-migration assertions (must all hold)
- For every OSMO validator: `Delegation == TokensFromShares(on-chain shares)`.
- `TotalDelegations == Σ validator.Delegation`.
- Redemption rate change is within inner safety bounds (should be ~0).
- Sum of amounts swept to the redemption account == sum owed to the paid redeemers.

## 6. Do NOT
- Do **not** run `calibrate`/slash on the OSMO phantom without the paired liquid handling — it
  drops `TotalDelegations` ~614k with no offset, tanking the rate ~13% and likely tripping the
  safety-bound halt. (`CalibrateDelegation` is blocked anyway by `CalibrationThreshold = 5000`.)
- Do **not** re-delegate the phantom-corresponding liquid to stakers — it is owed to redeemers.

## 7. Prevention (process)
- Ramp validator weights toward 0 over several unbonding periods, not in one upgrade.
- Dry-run every rebalance against mainnet state and push the resulting undelegations through
  `ValidateUnbondAmount` before shipping.
- Monitor: delegation ICA holding non-trivial liquid; tracked-vs-on-chain delegation divergence
  (needs periodic ICQ); the new `undelegation_failed` event.
