# Reproducibility statement

All numbers reported in the paper are recomputed from per-episode carriers (judgements and traces) by scripts shipped with the project. This file is the human-readable map of that process; the authoritative command list with expected readings is `REPRODUCE_CHECKS.md` at the project root.

## 1. Environment

- Python 3.12 (stdlib `urllib` for the model client; no local GPU or deep-learning framework is required).
- Model channel: a local Magpie-compatible OpenAI gateway at `http://127.0.0.1:3425/v1`, accessed through a reverse tunnel on the compute host. The task uses chat completions with `temperature = 0` and a 1600-token output cap; the three preregistered iterations expose two tools (`stat`, `write`), and the carrier-sensitivity replication adds a `retrieve` tool that reads the version mapping from a corpus while leaving the resource identifiers in the system prompt.
- Cards and code live in `l3_pilot/` of the project. The compute host runs the orchestration layer only; the analysis and all gates run locally.

## 2. Frozen analysis layer

Six files are frozen by the preregistration (full SHA-256 in `PREREG_L3.md` §1; prefix check here):

| file | sha256 prefix |
|---|---|
| `judge.py` | `0e09d9e1123236ad82bd` |
| `engine.py` | `46e99ba8dd9a6f82d84e` |
| `cards.py` | `cea89d856b3d2c1e5000` |
| `cards_agent.py` | `53e7d8f92408e2d7422d` |
| `agent_arm.py` | `6239b5d32d578122d9b1` |
| `power_l3.py` | `2fe1f54210f2848ff826` |

`python3 formal_runner.py --verify-provenance` re-checks these hashes and exits non-zero on any mismatch. The three orchestration runners (`formal_runner.py`, `formal_runner_mt.py`, `factorial_3x2.py`) and the v2/v3 runners are not frozen; their identity at the time each preregistration took effect is recorded in the deviation log and, for v3, in `L3_V3_PREREG.md` §4b.

## 3. Experimental carriers

| run | directory | records | role |
|---|---|---|---|
| Iteration 1 (packaged) | `runs/l3-runA-50x2x3/` | 300 | confounded contrast + factorial decomposition |
| Iteration 2 (minimax) | `runs/l3-v2-sc-40-minimax/` | 80 | identifier-confirmed main batch |
| Iteration 2 (kimi) | `runs/l3-v2-sc-40-kimi-lowspeed/` | 80 | second confirmation |
| Iteration 2 (workbuddy) | `runs/l3-v2-sc-40-workbuddy2/` | 80 | zero-transport-loss confirmation |
| Iteration 3 (minimax) | `runs/l3-v3-sc-200-minimax/` | 400 | clean main experiment (159 effective pairs) |
| Carrier-sensitivity (workbuddy) | `runs/l3-sa-sc-200/` | 306 | retrieval-corpus replication, stopped at 306/400 (rate limits) |
| Carrier-sensitivity pilot | `runs/l3-sa-sc-40/` | 80 | 40-card precursor (reported in `DEVIATION_L3.md` D24) |
| Truncation repro | `runs/l3-truncation-repro/` | 32 | 400-token cap causal chain |

Each run directory holds one JSON per episode under `judgements/` and `traces/`, plus `state.json` and `report.json`. `python3 verify_evidence_chain.py --run-dir <dir>` confirms that the reported record count matches the number of carrier files and recomputes the incomplete/loss rates; it exits non-zero on mismatch.

## 4. Recomputing every headline number

From `l3_pilot/`:

```bash
python3 summarize_v3_main.py runs/l3-v3-sc-200-minimax   # Table 4 numbers
python3 paper_v2/scripts/make_v3_evidence.py            # -> paper_v2/evidence/v3_main.json
python3 summarize_sa.py --run-dir runs/l3-sa-sc-200      # Table 5 numbers -> verification log
python3 summarize_sa.py --selftest                       # 4/4: smoke + tamper + stratified + RED direction
python3 resummarize_sa.py --selftest                     # 2/2: rebuild-from-carrier checked against registered report
python3 paper_v2/scripts/make_tables.py                 # -> paper_v2/paper/tables/tab_*.tex (cells/paired/writes/v3/sa)
python3 paper_v2/recompute_stats.py                     # -> paper_v2/stats.json
python3 paper_v2/scripts/verify_batches.py              # cross-check raw vs rejudged
```

Expected iteration-3 readings: keep 200 episodes, del 200, 159 effective pairs, keep stale 2/159, del stale 53/159, paired difference 32.0755 pp, McNemar exact p = 1.199e-14, cost ratio 1.0177 (≤ 1.10 bound), both arms delivering all files (199/200 and 200/200), deletion-arm pre-injection accepted writes 1481.

Expected carrier-sensitivity readings (306 records, 12 transport losses excluded): 147 episodes per arm, 146 effective pairs, keep stale 10/146 versus del 79/146, paired difference 47.2603 pp, McNemar p = 3.726e-18, token ratio 1.2622, delivery 147/147 versus 135/147; stratified row (deletion arm wrote before invalidation, n = 97) 5 versus 30, +25.77 pp, p = 4.649e-06, token ratio 1.2452. The `resummarize_sa.py` tool rebuilds `runs/l3-sa-sc-200/report.json` from the 306 judgement/trace carriers (the runner was killed before writing it) and marks it `resummarized_from_carriers: true`; it is checked against the 40-card registered report before touching the new batch.

## 5. Gates (all must pass; each has an injected-violation RED test)

```bash
python3 l3_pilot/power_l3.py --check
python3 l3_pilot/formal_runner.py --verify-provenance
python3 l3_pilot/v3_runner.py --selftest            # 7/7
python3 l3_pilot/sa_runner.py --selftest            # 8/8 (retrieval carrier tool)
python3 l3_pilot/verify_evidence_chain.py --selftest
python3 scripts/research_failure_mode_gate.py --decl ... --paper ... --root . --final --prior ...
python3 scripts/experiment_provenance_gate.py  --decl ... --paper ... --root . --final --prior ...
python3 scripts/figure_trace_gate.py            --manifest ... --paper ... --root . --final --prior ...
bash ~/.dsh/skills/arxiv-submission/scripts/arxiv_package_check.sh paper_v2/paper l3-v6
```

Detailed commands, expected outputs, and every deviation (D1–D30, including the confound discoveries, the measurement-artifact corrections, and the carrier-sensitivity replication) are in `REPRODUCE_CHECKS.md` and `DEVIATION_L3.md`.
