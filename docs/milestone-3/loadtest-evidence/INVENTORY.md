# Milestone 3 — Load Test Evidence Inventory

Raw and derived results from the two load-test runs described in
[`../02-integration-test-report.md`](../02-integration-test-report.md) §3.3 (Load and soak
procedure), supporting that report's §5 (Results & logs) and §6 (Performance
characteristics).

Both runs executed against the **beta deployment on Cardano preprod** — real txbuilder,
real indexer, real Plutus V3 validators, real stablecoin transfers. Nothing about the
chain was mocked.

| Run | Profile | Window (UTC) | Orders |
|---|---|---|---|
| [`soak-constant-20260802/`](soak-constant-20260802/) | constant, target 38 orders/h, 8.7 h | 2026-08-03 04:42 → 13:25 | 304 |
| [`soak-burst-20260802/`](soak-burst-20260802/) | bursts of 10 every 30 min, 4.0 h | 2026-08-09 04:24 → 08:24 | 80 |

## Files (identical set in each run directory)

| File | What it contains | Produced by |
|---|---|---|
| `run-meta.json` | Run identity, profile, window, order counts | `loadtest:run` |
| `setup.json` | The run's vendors and licences (test data). The burst run was deliberately run against the constant run's vendors, so its copy is that same file and carries `runId: soak-constant-20260802` — see §7.3 of the integration test report. | `loadtest:setup` |
| `orders.jsonl` | One record per submitted order: sequence, HTTP status, submission latency, item count, order and royalty amounts | `loadtest:run` |
| `monitor.jsonl` | 60-second samples of health, order/batch status histograms, transaction census, and any alerts raised | `loadtest:monitor` |
| `timeline.jsonl` | Per-order lifecycle timestamps: intake, unlock, batched, IOU submitted/confirmed, settlement submitted/confirmed | `loadtest:collect` |
| `batches.csv` | Every batch with strategy, status, item and order counts, royalty total, and processing timestamps | `loadtest:collect` |
| `transactions.csv` | Every tracked transaction: type, status, transaction hash, submitted and confirmed times | `loadtest:collect` |
| `tx-census.csv` | Transaction counts grouped by type and status | `loadtest:collect` |
| `settlement-hourly.csv` | Confirmed settlement transactions per hour | `loadtest:collect` |
| `metrics.json` | Aggregated metrics: intake stats, per-stage latency percentiles, throughput, batch fill, log-derived event counts | `loadtest:report` |
| `metrics.csv` | Per-order stage durations, one row per order | `loadtest:report` |
| `summary.md` | Human-readable run summary: headline numbers, stage latency table, incidents, on-chain transaction inventory | `loadtest:report` |

Legends for the numeric enums used in the CSV and JSONL files:

- `orders.status` — 0 pending, 1 processed, 2 cancelled, 3 batched, 4 awaiting settlement
- `order_batches.status` — 0 pending, 1 ready, 2 processing, 3 completed, 4 failed
- `tracked_transactions.transaction_type` — 0 bucket create, 1 bucket remove, 2 batch IOU
  distribution, 3 settlement distribution, 4 bucket payout
- `tracked_transactions.status` — 0 submitted, 1 confirmed, 2 failed, 3 expired

## Scope and exclusions

- **Server logs are not included.** The runs captured full orchestrator and txbuilder logs
  (37 MB for the burst run alone); they were used for diagnosis and are not part of this
  package. Findings drawn from them are reported in §5.4 of the integration test report.
  One consequence: the `logEvents` block in `metrics.json` and the
  `Resubmissions / expiries / job errors` line in `summary.md` are derived by parsing those
  logs, so their zeros are absence of input rather than measurement — see §5.5.
- **Payout results are not included.** The payout phase is an operational step to drain
  vendor buckets between runs, not a performance characteristic in scope for Milestone 3.
  `payouts.jsonl` is therefore omitted, and the payout sections have been removed from
  `summary.md` and `metrics.json`.
- **Payout transactions still appear in `transactions.csv` and the transaction inventory**
  as `transaction_type = 4`. They are on-chain records belonging to the run's own vendors and
  buckets, drained after the orders had settled, so they postdate the order-arrival window
  shown above. They have been left intact rather than filtered.
- The `loadtest:collect` extracts are **scoped to their own run window**, so no file mixes
  data from the other run or from unrelated activity on the deployment. Two files are not
  produced by that stage and are not window-scoped: `monitor.jsonl`, which is written by a
  long-lived monitor session and in the constant run carries one entry from the start of the
  burst run (see §5.4 of the integration test report), and the payout rows described above.

## Reading the numbers

Three points from §6.1 of the integration test report are needed to interpret these files
correctly:

1. **The headline latency is `unlock → settled`**, not intake-anchored. Orders are held for
   an order-lock period before becoming batch-eligible — 24 hours in production, compressed
   to 30 seconds for these runs. That lock is the ORSY cancellation window, during which
   nothing irreversible happens, so excluding it makes the figure comparable across
   deployments.
2. **Most remaining latency is configured policy**, not computation: batching windows, job
   intervals, and on-chain confirmation thresholds. Actual compute is the ~200 ms intake
   and sub-second transaction builds.
3. **Throughput was not saturated.** The figures reflect the arrival rates chosen for each
   profile, not a measured capacity ceiling.

Configuration deviations from production are listed in §2.4 of the integration test report
and must accompany any published figure.
