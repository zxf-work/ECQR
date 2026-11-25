# Reproducing the results: reference

This page holds the details behind the "Reproducing the results"
section of the [README](../README.md): optional packages, the external
data and how to pin it, every experiment with its script and receipt,
and how the cluster runs were made.

"Supp." refers to the supplementary material,
[`ecqr_supplement.pdf`](../ecqr_supplement.pdf).

## Contents

1. [Repository layout in detail](#repository-layout-in-detail)
2. [Setup and optional packages](#setup-and-optional-packages)
3. [Data](#data)
4. [Experiments](#experiments)
5. [Regenerating numbers, tables and figures](#regenerating-numbers-tables-and-figures)
6. [Cluster runs](#cluster-runs)

## Repository layout in detail

| path | contents |
|---|---|
| `tgms/evidence/` | ECQR core: descriptor (`ecqr.py`), typed claims (`claims.py`), verifier (`verify.py`); backend adapters (`adapter_tgms.py`, `adapter_sql.py`); SQL page/count snapshot reads (`sql_snapshot.py`); EvidenceBench fault cells (`faultbench.py`, `faultbench_topk.py`) |
| `tgms/agent/` | agent layer: plan IR, executor, claim verifier gate, certified renderer |
| `tgms/eval/` | task-suite generation (two-lane gold), the frozen task-run harness (`tgms eval run`), SQL baselines (`b6`, `b6e`) |
| `tgms/` (other) | the TGMS store, temporal operator algebra, MCP tool surface, CLI |
| `crates/` | native storage engine (Rust, built as `tgms._engine`) |
| `configs/` | operator manual and few-shot plans read by the planner; campaign configs of the frozen task runs (`campaign/m8-*.yaml`) and of the operator-level ablation (`campaign/m6-*.yaml`) |
| `benchmarks/oracle-v3/`, `benchmarks/oracle-v3.1/` | frozen task suites of the three SNAP datasets (fixed 116-draw universes, two-lane gold) |
| `benchmarks/frozen-v2/` | the predecessor suites of the same datasets, kept for the record |
| `benchmarks/results-v1/` | run receipts: every reported number is computed from these files |
| `external_workloads/` | BIRD and LDBC freezes (`FREEZE.md`, `MANIFEST.yaml`), annotations, adjudication records, the BIRD agent run (`bird/agent_run/`), the truncation probe (`probe/`), and their scripts |
| `scripts/` | experiment drivers (`eval_*.py`, `m8_tables.py`, `gate_counterfactual.py`), Slurm examples (`*.slurm`), and the paper-output generators (`paper_*.py`) |
| `tests/` | tests of the evidence layer and of the modules it uses |

## Setup and optional packages

Requirements: Python 3.12, [uv](https://docs.astral.sh/uv/), and a Rust
toolchain (stable; the engine is a PyO3 extension built by maturin).

```bash
uv sync                      # builds the native engine, installs runtime + dev dependencies
uv run pytest tests/         # evidence-layer tests
```

`uv sync` installs the dev group, which includes DuckDB, pytest, and
hypothesis. Further packages are needed only by specific experiments:

| package | used by |
|---|---|
| `uv sync --extra agent` (litellm, fastmcp) | the frozen task runs (`tgms eval run`) |
| `sqlglot==30.15.0` | BIRD annotation, BIRD agent configuration, truncation probe |
| `pgserver==0.1.4` and `psycopg` (or `psycopg2`) | PostgreSQL legs of the snapshot stress test and of the probe; the PostgreSQL snapshot test runs only with `TGMS_PG_TESTS=1` |
| `transformers` | token counts in `scripts/eval_unsupported_composition.py` (tokenizer of `Qwen/Qwen2.5-14B-Instruct-AWQ`) |
| vLLM 0.11.0, in a separate environment | serving the model for every run that calls an LLM |

Install the single-purpose packages with `uv pip install <package>`.

## Data

Neither the BIRD databases nor the SNAP graphs are redistributed here.

### BIRD Mini-Dev (SQLite, 500 SELECT questions)

- Source: github.com/bird-bench/mini_dev at commit
  `b3d4bcbbae9a96934ad812551eb400c7a3b23c12`, and the Hugging Face
  dataset `birdsql/bird_mini_dev` at revision
  `f65faf4ae3b638c1fa6df1d3370c8d92c8366301`.
- The databases come from the package `minidev_0703.zip` linked from the
  mini_dev README (sha256
  `aeb211c0e39010bbdae3838bb5e8bd27dc446ed77495b1709f85ccc9bf67f2be`).
- Unzip it under `external_workloads/bird/package/` so that each
  database is at
  `external_workloads/bird/package/minidev/MINIDEV/dev_databases/<db_id>/<db_id>.sqlite`,
  or pass the `dev_databases` directory with `--db-root` / `--dbroot`.
- The per-database sha256 values are pinned in
  `external_workloads/MANIFEST.yaml` (`bird.database_sha256`).
- The frozen question set itself is committed
  (`external_workloads/bird/bird_500_select_sqlite.jsonl`, pinned in the
  same manifest).

### SNAP temporal graphs

Download from snap.stanford.edu/data into a directory passed to the
loader (`tgms/data/loaders.py`, conventionally `data_raw/`). The loader
writes a `<file>.sha256` sidecar on first download and checks it on
every later load. To enforce the values the paper's runs used, create
the sidecars before the first load:

```bash
echo "<sha256>  <file>" > data_raw/<file>.sha256
```

These values were recorded by the loader when the files were
downloaded; the freeze files do not pin them.

| dataset | files | sha256 | events |
|---|---|---|---:|
| MathOverflow (`sx-mathoverflow`) | `sx-mathoverflow-a2q.txt.gz` | `1e160dcd3e685f7467f4869451899bae0299016080ae994895a900b57812ca78` | 506,550 (all three files) |
| | `sx-mathoverflow-c2q.txt.gz` | `ee99942bc234cb3596e93a5c49ce8e3bd82af6dcca4adb687f287b2eef5f0a3e` | |
| | `sx-mathoverflow-c2a.txt.gz` | `bdfa7b71ba564c8b9307ae9892ab7138c35ac5a0e4a285cb9021d1e6598a27a4` | |
| SuperUser (`sx-superuser`) | `sx-superuser-a2q.txt.gz` | `1facf774af2a644c0c230a4c9d8994b2071a9b24bea9f40af5fe6f6ada3557c7` | 1,443,339 (all three files) |
| | `sx-superuser-c2q.txt.gz` | `10c7be82d96add560dfa04415d51727032635d0f7456686bcde4b0664b865236` | |
| | `sx-superuser-c2a.txt.gz` | `e8dbcfdd6e62e8984694ee905e5d4a0b09f6af23afe81a1eb695f7e981800494` | |
| wiki-talk (`wiki-talk`) | `wiki-talk-temporal.txt.gz` | `2053b1f5bb3c46320f8e4da91aa6d7d79ec8d78e2c8cc99cede609c0c375f33e` | 7,833,140 |

### Building the stores

- A store is built with
  `tgms.data.loaders.ingest_dataset(name, data_dir, store_path, backend=...)`.
- `tgms tasks` injects the synthetic correction stream
  (`tgms.eval.tasks.inject_corrections`) when it generates a suite.
- `tgms memory build` builds the planner's memory database.
- `tgms replay <store>/eventlog.jsonl --store <replica> --backend duckdb`
  builds the SQL replica with transaction times preserved.

The campaign configs expect these paths:

| what | path |
|---|---|
| operator store | `stores/<dataset>` |
| DuckDB replica | `stores/<dataset>-duckdb` |
| memory database | `stores/<dataset>/memory.sqlite` |
| suite | `stores/suite-<dataset>/suite-v3.json` (the committed copy is `benchmarks/oracle-v3/suite-<dataset>.json`) |

**Canonical stores.** The canonical stores the paper used (SNAP events
plus the injected corrections, built on the cluster at the commits
recorded in each suite) are not redistributed; wiki-talk alone has
about 7.8M events. A store rebuilt from the same raw files has the same
valid-time history, but transaction times and derived identifiers are
assigned at ingest. The frozen suites' gold for belief-time questions
therefore holds against the canonical stores, or against replicas
replayed from their event logs, not against an independent re-ingest.

### LDBC SNB

The read templates are extracted from the official query
specifications. The three source repositories are pinned in
`external_workloads/MANIFEST.yaml` (`ldbc`):

| repository | commit | check out at |
|---|---|---|
| ldbc_snb_docs | `b2269610f433da72e7c97041f01680aae369a903` | `external_workloads/ldbc/docs` |
| ldbc_snb_interactive_v1_impls | `11db98cc2ba14c33492f6c0c34e68c8be7e22e5f` | `external_workloads/ldbc/interactive_v1` |
| ldbc_snb_bi | `47dd38b40844ecdb0e42e5a610c369535304786d` | `external_workloads/ldbc/bi` |

The extracted manifest and the annotations are committed, so only
re-extraction needs these checkouts. No LDBC data set is used; nothing
LDBC-related is executed.

## Experiments

Receipts are under `benchmarks/results-v1/` unless a path is given.
"Cluster" marks runs that were made as Slurm jobs; the `.slurm` files
are the job scripts used (see [Cluster runs](#cluster-runs)). An empty
Supp. cell means the supplementary material has no table or section for
that result.

| result | Supp. | script(s) | receipt(s) | needs |
|---|---|---|---|---|
| EvidenceBench fault matrix (fault x claim matrix, errors by fault family) | §6 Fig. 4; §7 Table 1 | `scripts/eval_fault_matrix.py` | `eval-fault-matrix.json` | CPU only |
| Deterministic baseline checkers and composition cells | §9 Tables 4 and 5 | `scripts/eval_baseline_checkers.py`, `scripts/eval_baseline_checkers_v2.py` | `eval-baseline-checkers.json` (two checkers), `eval-baseline-checkers-v2.json` (five checkers, before the executor's basis-compatibility fix), `eval-baseline-checkers-v3.json` (same, after the fix) | CPU only |
| Top-k EvidenceBench cells | §10 Table 6 | `scripts/eval_topk_cells.py` | `eval-topk-cells.json` | CPU only |
| Frozen task runs (three datasets, four configurations, three seeds) | §8 (audit-mode accuracy) | `tgms eval run --config configs/campaign/m8-<dataset>-{tgms,sql}.yaml` (job script `scripts/itiger_job.slurm`), then `scripts/m8_tables.py --runs runs --out benchmarks/results-v1/m8` | `m8/m8-tables.json`, `m8/m8-tables.md` | GPU + vLLM 0.11.0 serving `Qwen/Qwen2.5-14B-Instruct-AWQ` (temperature 0, 16,384-token context); the SNAP stores; cluster |
| Replay control: the gate applied to the ungated runs | §8 | `scripts/gate_counterfactual.py --runs runs` | `eval-gate-counterfactual.json` | CPU; the task-run rows under `runs/` |
| Unsupported-claim composition, plan depth, context tokens | | `scripts/eval_unsupported_composition.py --runs runs` | `eval-unsupported-composition.json` | CPU; the task-run rows; `transformers` for the tokenizer |
| Propagation ablation over the frozen task runs | §9 | `scripts/eval_propagation_ablation.py --runs runs` | `eval-propagation-ablation.json` | CPU; the task-run rows |
| Model configurations (read by `paper_numbers.py`, not printed in the paper) | | `configs/campaign/m8-*-{7b,32b,32bfp16}.yaml`, then `scripts/m8_tables.py` | `m8-7b/`, `m8-32b/`, `m8-32bfp16/` | GPU + vLLM with `Qwen/Qwen2.5-7B-Instruct`, `Qwen/Qwen2.5-32B-Instruct-AWQ`, `Qwen/Qwen2.5-32B-Instruct`; cluster |
| BIRD gold validation, claim annotation, multiplicity audit | §5; §7 Table 2; §13 | `external_workloads/scripts/validate_bird_gold.py`, `annotate_bird_sql.py`, `audit_bird_multiplicity.py` | `eval-bird-gold.json`, `eval-bird-annotation.json`, `eval-bird-multiplicity.json`; `external_workloads/bird/*.jsonl` | CPU; BIRD databases; `sqlglot` |
| BIRD agent configuration | §7 Table 2; §13 | `external_workloads/scripts/run_bird_agent.py` (job script `external_workloads/scripts/bird_agent.slurm`); `--replay-from external_workloads/bird/agent_run` re-scores the recorded SQL without model calls | `eval-bird-agent.json`; per-question records in `external_workloads/bird/agent_run/` | GPU + vLLM 0.11.0 serving `Qwen/Qwen2.5-14B-Instruct-AWQ` (replay: CPU); BIRD databases; `sqlglot`; cluster |
| BIRD infrastructure check with gold SQL in place of the agent; official BIRD EX of the same 500 statements | §5 (protocol validation); §13 | the agent harness with gold SQL substituted; the official Mini-Dev evaluator (`evaluation_ex.py` of the pinned mini_dev commit), not scripted here | `eval-bird-oracle-smoke.json`, `eval-bird-official-ex.json` | CPU; BIRD databases |
| LDBC coverage | §7 Table 3 | `external_workloads/scripts/extract_ldbc_templates.py`, `annotate_ldbc_coverage.py` | `eval-ldbc-coverage.json`; `external_workloads/ldbc/ldbc_read_templates.jsonl`, `coverage_annotation.jsonl` | CPU; LDBC repositories only for re-extraction |
| Top-k routes on BIRD; static LDBC top-k judgment | §12 Tables 8 and 9 | `scripts/eval_bird_topk.py --db-root <dev_databases>` (v1), with `--boundary` (v3); the LDBC judgment is static and has no script | `eval-bird-topk.json`, `eval-bird-topk-v2.json`, `eval-bird-topk-v3.json`, `eval-ldbc-topk.json`; `external_workloads/ldbc/topk_expressibility.jsonl` | CPU; BIRD databases (replay of recorded agent SQL); cluster for v2/v3 |
| Verifier scaling, descriptor bytes | | `scripts/eval_verifier_scaling.py` | `eval-verifier-scaling.json` | CPU only; cluster node |
| Evidence overhead, SQL certificate cost | | `scripts/eval_evidence_overhead.py --sql-db <SuperUser DuckDB replica>` | `evidence-overhead-itiger.json` | CPU; the SuperUser replica; cluster node |
| Basis semantics under live correction | §15 Fig. 5 | `tests/test_live_basis.py` | the test itself | CPU only |
| SQL page/count snapshot stress test, pre-registered in `external_workloads/probe/SNAPSHOT_FREEZE.md` | §11 Table 7 | `scripts/eval_sql_snapshot.py` (job script `scripts/sql_snapshot.slurm`), `tests/test_sql_snapshot.py` | `eval-sql-snapshot.json` (SQLite, DuckDB), `eval-sql-snapshot-pg.json` (PostgreSQL) | CPU; DuckDB; `pgserver` for PostgreSQL; cluster |
| Truncation probe without ECQR (baseline rows), pre-registered in `external_workloads/probe/PROBE_FREEZE.md` | §14; §16 Table 11 (baseline rows) | `external_workloads/probe/setup_probe.py`, `run_probe.py`, `score_probe.py` (job scripts `probe.slurm`, `probe_pg_only.slurm`) | `eval-trunc-probe-setup.json`, `eval-trunc-probe.json`, scorer checks `eval-trunc-probe-oracle-diligent.json` and `eval-trunc-probe-oracle-enumerating.json`; `external_workloads/probe/probe_manifest*.jsonl` | GPU + vLLM 0.11.0 serving `Qwen/Qwen2.5-14B-Instruct-AWQ`; BIRD databases; `sqlglot`; `pgserver` for the PostgreSQL leg; cluster |
| Truncation probe under the ECQR gate, pre-registered in `external_workloads/probe/PROBE_ECQR_FREEZE.md` | §16 Table 11 | `external_workloads/probe/run_probe_ecqr.py`, `score_probe_ecqr.py` (job script `probe_ecqr.slurm`) | `eval-trunc-probe-ecqr.json` | GPU + vLLM 0.11.0 serving `Qwen/Qwen2.5-14B-Instruct-AWQ`; BIRD databases; `sqlglot`; cluster |
| Two-lane suites, oracle coverage | §4 Fig. 3 | `tgms tasks` (suite generation) | `benchmarks/oracle-v3/`, `benchmarks/oracle-v3.1/` | CPU; the SNAP stores |
| Every number in the text | | `scripts/paper_numbers.py`, `scripts/paper_macros.py` | `paper_numbers.json` | standard library only |

`paper_numbers.json` also carries the admission-guardrail operating
points (`guardrail`, from `eval-guardrail-frontier-d087.json` and
`guard-frontier-itiger-scaled.json`) and the operator-level ablation
(`m6_frontier`). The per-operator time coefficients of
guardrail-policy-v1, which every task run uses
(`tgms/temporal/guardrails.py`), were measured in
`eval-guardrail-frontier.json`.

The main paper's conceptual figures and Supp. Figs. 1, 2, 3 and 5 are
drawn by hand and carry no measured data.

## Regenerating numbers, tables and figures

These commands need only the Python standard library (3.12 or later;
`scripts/paper_macros.py` uses f-string syntax that Python 3.11
rejects) and read only committed files. Run them from the repository
root:

```bash
OUT=$(mktemp -d)
cp scripts/paper_palette.tex "$OUT/preamble-figures.tex"   # figure colors
python3 scripts/paper_macros.py         --outdir "$OUT"   # pn-macros.tex and tab-*.tex
python3 scripts/paper_figures.py        --outdir "$OUT"   # fig-*.tex (reads pn-macros.tex)
python3 scripts/paper_bird_funnel.py    --outdir "$OUT"   # fig-bird-funnel.tex
python3 scripts/paper_bird_census.py    --outdir "$OUT"   # tab-bird-census.tex
python3 scripts/paper_ldbc_table.py     --outdir "$OUT"   # tab-ldbc-coverage.tex
python3 scripts/paper_fault_families.py --outdir "$OUT"   # tab-fault-families.tex
ls "$OUT"
```

- **Order.** `paper_macros.py` must run first. The figure generators
  read `pn-macros.tex` from the same directory and refuse to write a
  figure whose receipt disagrees with a macro printed beside it.
- **Placeholder files.** `paper_figures.py` also writes a few files that
  only say a figure was merged into another (`fig-bird-census.tex`,
  `fig-efficiency.tex`, `fig-reasons.tex`, `fig-topk.tex`).
- **`paper_numbers.json`.** `scripts/paper_numbers.py --out <file>`
  rebuilds `paper_numbers.json` from the receipts. Two parts need the
  raw task-run rows under `runs/`, which are not committed:
  `m6_frontier` and `frozen_2x2.changed_outcome_audit`. Without `runs/`
  the script marks them as absent, so write its output to a new file
  rather than over the committed `paper_numbers.json`; every other
  section reproduces exactly.

## Cluster runs

Runs marked "cluster" were Slurm jobs on the University of Memphis
iTiger cluster:

- partition `bigTiger`;
- GPU jobs on RTX 6000 Ada, RTX 5000 Ada, or H100 nodes;
- driver CUDA 12.6, vLLM 0.11.0 with torch 2.8.0+cu126 and transformers
  4.57.1 in a separate environment.

The `.slurm` files are those job scripts and contain the cluster's
absolute paths (`/project/xzhang12/...`). To adapt one, change the
partition, the `--gres` request, the paths to the repository, the
Python environment, the vLLM environment, the Hugging Face cache, and
the BIRD `dev_databases` directory; the commands themselves need no
change.

Every LLM-calling run uses one served model at temperature 0: seed 0
for the external workloads and the model configurations, seeds 0, 1, 2
for the main frozen task runs. The prompt and repair templates of the
BIRD agent configuration are pinned by sha256 in
`external_workloads/FREEZE.md`.

CPU-only receipts record their host in their manifest; some were
measured on a laptop (Apple M-series), the rest on cluster CPU nodes.
