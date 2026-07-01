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

## 3. Why a migration is required (and what it must do)

The stuck epochs already retry every unbonding cycle and keep failing. Retrying alone can't fix
it, for two reasons:

1. **The retry targets the phantom validators.** The undelegation split still points at
   cosmostation/chorus_one/pryzmstakedrop, which hold ~0 on-chain, so the host rejects it. Until
   the phantom `Delegation` records are reconciled to on-chain, every retry fails. (The
   shares-safety fix does not help here — that's the INJ rounding case, not a zero-delegation
   target.)
2. **Clearing the phantom cratered the rate unless the liquid is credited.** Removing 614.4k from
   `TotalDelegations` drops the redemption rate ~13% instantly, because the rate never counted the
   623.8k liquid — the phantom was the only thing keeping that balance in the rate.

So the migration's essential job is exactly **two accounting corrections**: reconcile the phantom
records, and credit the stranded liquid. Everything else (paying redeemers, burning stTokens) is
then done by the existing, battle-tested unbonding machinery — no per-epoch logic required.

## 4. Forward fixes (shipped as separate PRs)

1. **Shares-rounding safety** in `GetUnbondingICAMessages` — buffers full drains of slashed
   validators so the host stops rejecting them. Fixes Injective and prevents recurrence.
2. **`undelegation_failed` event** in `HandleFailedUndelegation` — makes a stuck retry loop
   alertable instead of silent (both chains).
3. **`CheckDelegationRecordsConsistent`** in BeginBlocker — log-only internal accounting check.

## 5. Recommended migration (v33): reconcile records, let the queue drain

Two state edits in the upgrade handler; the existing machinery does the rest.

1. **Reconcile the phantom.** For each of the 3 validators, set `Delegation` to its on-chain value
   (cosmostation → 0, chorus_one → 0, pryzmstakedrop → 60,662.8) and decrement `TotalDelegations`
   by the total (614,423.1). Re-derive these from live state at migration time.
2. **Credit the stranded liquid.** Create a `DELEGATION_QUEUE` deposit record for the delegation
   ICA's liquid balance (~623,806.6), so it is counted (`undelegatedBalance`) and re-delegated by
   the normal delegate flow.

Net backing change ≈ **+9.4k** (previously-uncounted reinvestment rewards) → the redemption rate
is preserved (a hair higher). Correct, not coincidental.

After that, no custom per-epoch code: the stuck epochs retry, now find the phantom validators at
0 capacity so the split targets real validators, succeed (with the shares-safety fix), unbond
fresh, and the normal `UndelegateCallback` burns stTokens and pays redeemers. No manual burn,
no manual sweep, no per-epoch allocation.

Trade-off vs. Appendix A: the already-unbonded redeemers (epochs 1345/1346) wait one more
unbonding period, and ~614k is re-delegated then re-undelegated (churn). Accepted in exchange for
using only well-tested flows instead of novel fund-moving migration code — the right call for
live funds.

### Must confirm on the fork replay (the riskiest assumption)
- **A manually-created `DELEGATION_QUEUE` deposit record actually gets delegated against the
  pre-existing liquid** in the delegation account. The normal path assumes tokens arrived via
  deposit → IBC transfer → delegate; a record created directly against balance already sitting in
  the delegation ICA may not be picked up. If it isn't, the migration must instead issue the
  delegate ICA directly (or trigger the transfer/sweep step). **Prove this on the fork before
  anything else.**
- Once the phantom is cleared, the stuck epochs produce a valid undelegation split (real
  validators) and complete end-to-end (unbond → sweep → claim).

### Post-migration assertions (must all hold)
- For every OSMO validator: `Delegation == TokensFromShares(on-chain shares)`.
- `TotalDelegations == Σ validator.Delegation`.
- Redemption-rate change within inner safety bounds (≈ 0).
- Delegation ICA liquid balance re-delegated (→ ~0 after the delegate lands).
- Every stuck redemption eventually reaches `CLAIMABLE` and pays its user the recorded amount.

## 6. Do NOT
- Do **not** run `calibrate`/slash on the phantom without also crediting the liquid — it drops
  `TotalDelegations` ~614k with no offset, tanking the rate ~13% and likely tripping the
  safety-bound halt. (`CalibrateDelegation` is blocked anyway by `CalibrationThreshold = 5000`.)
- Do **not** clear the phantom without confirming the stuck records will still unbond (they do,
  once the split targets real validators) — otherwise redeemers are neither paid from liquid nor
  re-unbonded.

## 7. Prevention (process)
- Ramp validator weights toward 0 over several unbonding periods, not in one upgrade.
- Dry-run every rebalance against mainnet state and push the resulting undelegations through
  `ValidateUnbondAmount` before shipping.
- Monitor: delegation ICA holding non-trivial liquid; tracked-vs-on-chain delegation divergence
  (needs periodic ICQ); the new `undelegation_failed` event.

## Appendix A — alternative: pay executed redeemers directly from the liquid

Faster for the oldest redeemers (no extra unbonding wait) but requires more custom fund-moving
migration code, so it is **not** recommended unless sparing those redeemers ~14 days outweighs the
added risk.

It relies on a per-epoch split that is derivable **without an archive**: the executed volume
equals the phantom (614,423.1 OSMO — already completed unbonding, now the liquid), attributed
oldest-epoch-first because `UpdateHostZoneUnbondingsAfterUndelegation` decrements records "starting
from the earliest record". Replaying that against each record's current `native_tokens_to_unbond`:

| Epoch | to_unbond (OSMO) | cumulative | classification |
|------:|-----------------:|-----------:|----------------|
| 1345 | 559,031.6 | 559,031.6 | executed → pay from liquid |
| 1346 | 49,330.7 | 608,362.3 | executed → pay from liquid |
| 1354 | 7,066.7 | 615,429.0 | boundary — 6,060.8 from liquid / 1,005.9 unbond |
| 1355,1358,1361,1362,1371,1374,1381,1384 | 61,936.7 | 676,365.7 | never executed → unbond fresh |

For the executed epochs: decrement the phantom `Delegation` + `TotalDelegations`, advance the
record to `EXIT_TRANSFER_QUEUE` (so the sweep pays from liquid), and burn the stTokens. Rate is
preserved because `amount = stTokens × rate`, so numerator −amount and denominator −amount/rate
keep `N/D` constant. Never-executed epochs and the liquid surplus are handled as in §5.
