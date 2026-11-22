# L3 — What deleting an agent's memory actually removes

Preregistered study of memory deletion in LLM agents. Three registered iterations plus a
carrier-sensitivity replication. Everything reported in the paper can be recomputed from the
per-episode judgements and traces shipped in the evidence bundle.

- Paper (PDF): [`paper/main.pdf`](paper/main.pdf) — 13 pages
- Preregistration (frozen before expansion): [`PREREG_L3.md`](PREREG_L3.md)
- Deviation log (every departure, with the evidence that forced it): [`DEVIATION_L3.md`](DEVIATION_L3.md)
- Recompute commands: [`REPRODUCE_CHECKS.md`](REPRODUCE_CHECKS.md)
- Reproducibility notes: [`REPRODUCIBILITY.md`](REPRODUCIBILITY.md)

## What the study found

The registered hypothesis was that deleting a memory entry from an agent's plan would reduce
stale-plan following. It did not, in the registered sense: the measured effect is carried by the
prompt wording, not by the deletion, and the deletion is inert once the prompt is held fixed. The
paper reports the negation with the same detail as a confirmation: the three registrations, the
power analysis that set the sample floor, the deviations that the data forced, and the
carrier-sensitivity replication that isolates what the deletion actually removes.

The registered cost clause did not hold either. The deletion arm spends 1.1479x the tokens of the
retention arm (clause ceiling 1.10); the mechanism is that recovering from a stale plan requires
finishing the work while a failing arm stops early.

## Evidence bundle

`l3-evidence-20261122.tar.gz` — 3012 files, per-episode judgements, traces, state files,
preregistrations, deviation log, and the paper source.

Verify before trusting it:

```bash
mkdir -p /tmp/l3-evidence-verify && cd /tmp/l3-evidence-verify
tar xzf /path/to/l3-evidence-20261122.tar.gz
sha256sum -c /path/to/SHA256SUMS    # 3012 lines, all OK
```

`evidence_manifest.json` carries the archive's own sha256 and byte count; the release page
carries the same values so the archive can be checked against a source outside itself.

## Recomputing the headline numbers

Every number in the paper is produced by `recompute_stats.py` from the run reports, not typed by
hand. See [`REPRODUCE_CHECKS.md`](REPRODUCE_CHECKS.md) for the exact commands, expected outputs,
and the three integrity gates (failure-mode, experiment-provenance, figure-trace) that the
manuscript passes before release.

## License and citation

Text and figures: CC BY 4.0. Code and evidence: CC0 1.0. Cite via [`CITATION.cff`](CITATION.cff).

```bibtex
@misc{liu2026l3,
  title  = {What deleting an agent's memory actually removes: evidence from three preregistered iterations},
  author = {Liu, Zewen},
  year   = {2026},
  note   = {ORCID 0009-0003-2981-9888},
  url    = {https://github.com/aidless/l3-stale-plan}
}
```
