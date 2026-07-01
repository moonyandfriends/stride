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

## 3. Per-epoch allocation (derived from current state — no archive required)

The three numbers do not reconcile 1:1 (stuck 676.4k ≠ liquid 623.8k ≠ phantom 614.4k) because
some stuck epochs **executed but lost their ack** (produced liquid) while others **never executed**
(error-ack, no liquid). This split can be derived **without any archive**, from two facts:

1. **Executed volume = the phantom = 614,423.1 OSMO.** The phantom is exactly the on-chain
   drainage that was never decremented from records; those tokens have already completed unbonding
   and are the stranded liquid. (The only amount still unbonding on-chain is a separate ~522 OSMO
   in-flight to 2026-07-07 — not part of this.)
2. **Attribution is oldest-epoch-first.** `UpdateHostZoneUnbondingsAfterUndelegation` decrements
   records "in a cascading fashion starting from the earliest record", so the correct
   reconciliation is exactly what that callback *would* have done with the executed amount: apply
   614,423.1 OSMO to the stuck epochs oldest-first.

Replaying that (using each record's current `native_tokens_to_unbond`):

| Epoch | to_unbond (OSMO) | cumulative | classification |
|------:|-----------------:|-----------:|----------------|
| 1345 | 559,031.6 | 559,031.6 | executed → pay from liquid |
| 1346 | 49,330.7 | 608,362.3 | executed → pay from liquid |
| 1354 | 7,066.7 | 615,429.0 | boundary — 6,060.8 from liquid / 1,005.9 unbond |
| 1355,1358,1361,1362,1371,1374,1381,1384 | 61,936.7 | 676,365.7 | never executed → unbond fresh |

Result: **614,423.1 OSMO** owed to the oldest redemptions is paid from the liquid; **~61,943 OSMO**
(newest) is unbonded fresh; the **~9,384 OSMO** liquid surplus is reinvestment rewards → re-delegate.

Optional simplification: pay only whole epochs from liquid (1345+1346 = 608,362.3), treat 1354
onward as unbond-fresh, and roll the 6,060.8 boundary remainder into the re-delegated surplus. This
avoids splitting an epoch's user-redemption records; slightly more unbonds fresh.

> Re-read `native_tokens_to_unbond` / `st_tokens_to_burn` for each epoch at migration time (values
> are current-state) and re-derive the phantom per validator, then assert the totals below.

## 4. Forward fixes (shipped as separate PRs)

1. **Shares-rounding safety** in `GetUnbondingICAMessages` — buffers full drains of slashed
   validators so the host stops rejecting them. Fixes Injective and prevents recurrence.
2. **`undelegation_failed` event** in `HandleFailedUndelegation` — makes a stuck retry loop
   alertable instead of silent (both chains).
3. **`CheckDelegationRecordsConsistent`** in BeginBlocker — log-only internal accounting check.

## 5. OSMO state migration (v33 upgrade handler) — algorithm

Inputs (all from §2/§3, current-state — no archive): the per-epoch executed/never-executed split
from the oldest-first table, and the per-validator phantom.

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
