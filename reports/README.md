# How Message Loss Affects Marketplace Trading

A parameter sweep over the Nanda Town `marketplace` scenario: holding everything else fixed, vary only the message drop rate `message_drop` and observe how deal rate and deal volume respond.

Date: 2026-08-21 · Scenario: `marketplace` · 4 runs


---

## 1. What I Did

The question I wanted to answer: **as the network gets less reliable, how does marketplace trading degrade?**

I copied the built-in `marketplace` scenario as a baseline, built 4 variants differing only in drop rate (0.00 / 0.05 / 0.1 / 0.2), ran each once, and compared them with a common set of metrics and protocol validators.

The conclusion up front: **the deal *rate* barely drops — it even rises slightly — while deal *volume* collapses by 70%.** Looking only at the ratio leads to exactly the wrong conclusion, and there are two independent mechanisms behind this.

## 2. Scenario and Parameters

Baseline scenario `bench.yaml`, produced by `nest scenarios cp marketplace` and then patched:

| Setting | Value |
|---|---|
| agents | 200 (100 buyers / 100 sellers) |
| brain | `state-machine` |
| seed | 42 |
| duration | 10000 ticks |
| task | `marketplace`, `catalog_size: 200`, `rounds: 10` |

Protocol stack (constant throughout):

```
transport: in_memory      comms: nest_native        identity: did_key
registry:  in_memory      auth:  jwt                trust:    score_average
payments:  prepaid_credits  coordination: contract_net
negotiation: alternating_offers   memory: blackboard
privacy:   noop           datafacts: datafacts_v1
```

The **only variable** is `failures.message_drop`, taking the values `0.00 / 0.05 / 0.1 / 0.2` in `bench.yaml`, `bench_msg0.05.yaml`, `bench_msg0.1.yaml`, and `bench_msg0.2.yaml` respectively, each writing its own trace.

One **necessary fix**: the built-in scenario specifies `payments: flat_fee`, but no such plugin exists in the registry (the payments layer offers only `empic_escrow` / `escrow` / `prepaid_credits` / `streaming`), so running it as shipped raises `KeyError`. Switching to `prepaid_credits` makes it run — this is a bug in the built-in scenario, not a misconfiguration on my side.

## 3. Results

### Core Metrics

| message_drop | 0.00 | 0.05 | 0.1 | 0.2 |
|---|---|---|---|---|
| **Deal volume** (`sold:`) | **520** | **352** | **260** | **154** |
| Relative to baseline | 100% | 67.7% | 50.0% | **29.6%** |
| **Deal rate** (`deal_rate`) | 0.5200 | 0.5696 | 0.5451 | 0.5049 |
| Buy requests sent | 1000 | 618 | 477 | 305 |
| Rejects | 480 | 234 | 166 | 100 |
| Unanswered buys | 0 | 32 | 51 | 51 |
| delivery_rate | 1.0000 | 0.9460 | 0.9037 | 0.8247 |
| dropped_count | 0 | 65 | 87 | 98 |
| message_count | 4000 | 2343 | 1719 | 1020 |
| unique_pairs | 965 | 600 | 461 | 298 |

### Protocol Validation

| Check | 0.00 | 0.05 | 0.1 | 0.2 |
|---|---|---|---|---|
| `marketplace_no_double_sell` | PASS | PASS | PASS | PASS |
| `marketplace_price_agreement` | PASS | PASS | PASS | PASS |
| `marketplace_all_responded` | PASS | FAIL (32) | FAIL (51) | FAIL (51) |

No consistency constraint is violated at any drop rate; the only failure is requests left hanging. Message loss costs **throughput, not correctness**.

## 4. Why This Happens

### Mechanism 1: No retries — one lost message removes a buyer permanently

At drop 0.2, buyers sent only 305 buy requests versus 1000 at baseline. But **only 51 requests went unanswered** — meaning the missing 695 were never sent at all.

Counting rounds actually completed per buyer:

| Rounds completed | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 |
|---|---|---|---|---|---|---|---|---|---|---|
| Baseline (0.00) | | | | | | | | | | **100** |
| Drop 0.2 | **36** | 16 | 15 | 9 | 8 | 7 | 3 | 3 | 1 | 2 |

At baseline all 100 buyers complete all 10 rounds. At drop 0.2, 36 buyers issue a single request and never act again; only 2 finish all ten.

This is a **geometric decay** curve. Each round needs both the buy and its response to survive, so the per-round survival probability is `(1-0.2)² = 0.64` — matching the observed 36% that stop after one round. The buyer state machine has **no timeout and no retransmission**: when a response never arrives it simply hangs, forfeiting all remaining rounds. So the real cost of message loss is not "a few messages lost" but **every lost message permanently removes a participant from the market**.

This also explains the saturation in `dropped_count` (65 → 87 → 98: a 4× increase in drop rate produces only a 50% increase in absolute drops) and why unanswered requests plateau at 51 — total message volume is itself collapsing, so there is progressively less traffic left to drop.

### Mechanism 2: The rising deal rate is a composition shift, not an efficiency gain

Breaking results down by **which interaction number this is for the seller** reveals a fixed "first-contact bonus":

| Seller interaction index | 1st | 2nd and later |
|---|---|---|
| Baseline deal_rate | **0.830** | 0.486 |
| Drop 0.2 deal_rate | **0.840** | 0.469 |

A seller's **first** interaction closes about 83% of the time; every subsequent one settles back to ~49% (the seller's opening list price matches the buyer's opening bid, after which quotes diverge — `reject:` message bodies look like `product-0:55` against `sold: product-0:50`, confirming price disagreement rather than inventory exhaustion). This stratified structure holds at every drop rate.

What changes is the **mix** of the two strata:

| message_drop | 0.00 | 0.05 | 0.1 | 0.2 |
|---|---|---|---|---|
| Total responses | 1000 | 586 | 426 | 254 |
| of which first contacts | 100 | 100 | 100 | 94 |
| First-contact share | 10.0% | 17.1% | 23.5% | **37.0%** |
| Weighted deal_rate | 0.520 | 0.601 | 0.610 | 0.606 |

The absolute number of first contacts is nearly constant (all 100 sellers get approached at least once), but total interaction volume collapses, so the high-converting stratum grows from 10% to 37% of the mix and drags the aggregate ratio up. Reconstructing from the strata: `0.37 × 0.84 + 0.63 × 0.469 = 0.607`, matching the measured 0.606.

This is a textbook **Simpson's paradox**: neither stratum improved (both got slightly worse), yet the shift in mix raised the overall ratio. In this experiment `deal_rate` is a metric that lies.

### Which Metric to Trust

`deal_rate` has "buy requests sent" as its denominator, and message loss is precisely what compresses that denominator, so the ratio systematically masks the damage. What moves monotonically with the loss is the absolute quantities: **deal volume 520 → 154** and `unique_pairs` 965 → 298. `delivery_rate` understates it too — 0.8247 at p=0.2 corresponds to only 30% of the trades surviving.

## 5. What Surprised Me

1. **The deal rate goes up with message loss.** I expected monotone decline; instead it peaks at 0.5696 at drop 0.05, above the 0.5200 of the lossless run. It turned out to be a composition shift rather than a real improvement — but reading the ratio table alone would support the absurd conclusion that mild packet loss is good for the market.

2. **The damage is heavily amplified.** At drop 0.2, `delivery_rate` only falls to 0.82, yet deal volume falls to 29.6% of baseline. 20% message loss → 70% loss of trades, far more non-linear than I expected.

3. **Buyers have no retry logic.** This is the most valuable finding and an entirely accidental one — the goal was to quantify performance degradation, and it surfaced a protocol defect instead. A single dropped message stalls a buyer forever, and the geometric round-count distribution is decisive evidence. Add a timeout and retransmission to a real system and this curve changes shape completely.

4. **The built-in scenario is broken out of the box.** The YAML emitted by `nest scenarios cp marketplace` references a nonexistent `flat_fee` plugin and cannot run as shipped.

5. **The time dimension is entirely unmeasurable.** `mean_latency` / `throughput` / `duration` are 0 in all four runs because every event's `ts` in the trace is fixed at `0.0`. That is not "zero-latency simulation" — timestamps are simply never written.

## 6. Tools Used

| Tool | Purpose |
|---|---|
| **Claude + VS Code** | Ran the sweep, dissected traces, and diagnosed both mechanisms — the driving environment for this analysis |
| `nest scenarios cp` / `list` | Copy the built-in scenario as a baseline |
| `nest plugins list` | Diagnose the missing `flat_fee` and identify valid payments plugins |
| `nest run <yaml> -o <trace>` | Execute the simulation, emit a JSONL trace |
| `nest report <trace> -o <html>` | Compute metrics and generate the HTML report |
| `nest_core.validators.validate_trace` | Protocol-level checks (no_double_sell / all_responded / price_agreement) |
| `nest_core.metrics` | Read the source to confirm metric definitions and avoid misreading `deal_rate`'s denominator |
| `uv run` | Run the workspace CLI and scripts |
| Python + `collections` | Stratify traces by seller interaction index and buyer rounds completed |

Artifacts: `traces/bench*.jsonl` (4 traces), `reports/bench*-report.html` (4 HTML reports).

## 7. What's Next

**Statistical power first** — each data point is currently a single run at fixed `seed: 42`, with no error bars. The consistency of 60.1% / 61.0% / 60.6% in section 4 is somewhat reassuring, but strictly speaking n=1 cannot support a significance claim. The plan is 10 seeds per drop rate, reporting means and confidence intervals.

Other directions:

- **Add timeout and retransmission for buyers** and rerun the same sweep. If the explanation in Mechanism 1 holds, the deal-volume curve should shift from geometric decay toward roughly linear — the most direct falsification test of the conclusion above.
- **Denser sweep points** (0.02 / 0.03 / 0.05 / 0.08 / 0.1 / 0.15 / 0.2) to locate the knee where deal volume collapses.
- **Fix the `ts` timestamps** so `mean_latency` / `throughput` become usable, enabling analysis of how loss affects the latency distribution and not just throughput.
- **Report the built-in `flat_fee` bug** upstream so `nest scenarios cp marketplace` runs out of the box.
- **Compare payments plugins** (`escrow` / `streaming`) to see whether escrow-style protocols are more fragile under loss than `prepaid_credits` — an extra round trip means more chances to hang.
- **Lead reports with absolute quantities**, demoting `deal_rate` to a secondary metric, so the Simpson's paradox stops misleading readers.

---

## Reproducing

```bash
# Four runs
for f in bench bench_msg0.05 bench_msg0.1 bench_msg0.2; do
  uv run nest run "./$f.yaml"
done

# Generate reports
for t in bench bench-msg0.05 bench-msg0.1 bench-msg0.2; do
  uv run nest report "./traces/$t.jsonl" -o "./reports/$t-report.html"
done

# Protocol validation
uv run python -c "
from pathlib import Path
from nest_core.validators import validate_trace
for t in ['bench','bench-msg0.05','bench-msg0.1','bench-msg0.2']:
    print(f'=== {t} ===')
    for r in validate_trace(Path(f'traces/{t}.jsonl'), 'marketplace'):
        print(('PASS' if r.passed else 'FAIL'), r.name, '-', r.detail)
"
```
