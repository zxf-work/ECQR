# ECQR: Evidence-Carrying Query Results

> **Supplementary material:** [`ecqr_supplement.pdf`](ecqr_supplement.pdf).
> It holds the proofs, a worked example, the complexity derivation, the
> benchmark construction and annotation protocol, the full result
> tables, and the interface audit. "Supp." below points into it.

## What this is

A database agent runs queries and turns the results into claims. A
query can execute correctly and still support less than the claim says.
Take a reachability query that computes 343 rows. The interface
delivers the first 100 rows and marks the result as truncated. Citing
one of those rows is fine. Saying "100 reachable accounts" is wrong.
A second failure is mixing evidence read at an older database state
with the current state.

With ECQR, a query result travels with a descriptor of what executed:
a digest of the delivered rows, the query domain, the basis (the
database state the query read), whether execution and delivery were
complete, and capabilities such as an exact cardinality. A verifier
that does not depend on the backend checks each typed claim against
that descriptor. The claim types are membership, scalar, exact count,
complete set, exists, not-exists and top-k, and each can be pinned to a
basis. The verdict is SUPPORTED or a named reason, such as
`UNSUPPORTED_COMPLETENESS_NOT_CERTIFIED`.

There are two backends. TGMS is a bi-temporal graph store with an agent
layer. SQL engines (SQLite, DuckDB, PostgreSQL) produce descriptors
through the SQL adapter. The same verifier judges both.

## A first example

The running example above, on an in-memory SQLite database. The page
and an unlimited count of the same query are read in one transaction,
and the SQL adapter turns them into a descriptor.

```python
import sqlite3

from tgms.evidence import CompleteSet, ExactCount, Membership, verify
from tgms.evidence.adapter_sql import build_sql_ecqr
from tgms.evidence.sql_snapshot import consistent_page_and_count

# 343 accounts are reachable; the interface delivers at most 100 rows.
con = sqlite3.connect(":memory:")
con.execute("CREATE TABLE reachable (account INTEGER PRIMARY KEY)")
con.executemany("INSERT INTO reachable VALUES (?)", [(i,) for i in range(343)])
con.commit()

query = "SELECT account FROM reachable"
page_sql = query + " ORDER BY account LIMIT 100"

# The page and an unlimited count of the same query, in one read transaction.
rows, total, _ = consistent_page_and_count(con, query, page_sql, engine="sqlite")
ecqr = build_sql_ecqr(rows=rows, sql=query, store_id="demo", engine="sqlite",
                      total_count=total, limited=True)
print("delivered", ecqr.scope.rows_returned, "of", ecqr.scope.exact_cardinality,
      "| delivery complete:", ecqr.scope.delivery_complete)

claims = {
    "account 7 is reachable": Membership(value=7),
    "343 accounts are reachable": ExactCount(n=343),
    "100 accounts are reachable": ExactCount(n=100),
    "these 100 are all of them": CompleteSet(members=[r[0] for r in rows]),
}
for text, claim in claims.items():
    print(f"{text:28} {verify(claim, ecqr, {'rows': rows}).verdict.name}")
```

Run it with `uv run python example.py` after the setup below. It prints:

```text
delivered 100 of 343 | delivery complete: False
account 7 is reachable       SUPPORTED
343 accounts are reachable   SUPPORTED
100 accounts are reachable   UNSUPPORTED_VALUE_MISMATCH
these 100 are all of them    UNSUPPORTED_COMPLETENESS_NOT_CERTIFIED
```

A delivered row is a witness even on a truncated page; a count or a
complete set needs evidence the page alone does not give. More cases
(pinned bases, top-k, the TGMS adapter) are in
[`tests/test_evidence.py`](tests/test_evidence.py),
[`tests/test_portability.py`](tests/test_portability.py) and
[`tests/test_topk.py`](tests/test_topk.py).

## Repository layout

| path | contents |
|---|---|
| `tgms/evidence/` | ECQR core: descriptor (`ecqr.py`), claims (`claims.py`), verifier (`verify.py`), TGMS and SQL adapters, EvidenceBench cells |
| `tgms/agent/` | agent layer: plan IR, executor, claim verifier gate, certified renderer |
| `tgms/eval/` | task-suite generation, the frozen task-run harness (`tgms eval run`), SQL baselines |
| `tgms/` (other) | the TGMS store, temporal operator algebra, MCP tool surface, CLI |
| `crates/` | native storage engine (Rust, built as `tgms._engine`) |
| `benchmarks/results-v1/` | run receipts: every reported number is computed from these files |
| `benchmarks/oracle-v3/`, `benchmarks/oracle-v3.1/` | frozen task suites of the three SNAP datasets |
| `benchmarks/frozen-v2/` | the predecessor suites of the same datasets, kept for the record |
| `external_workloads/` | BIRD and LDBC freezes, annotations, the BIRD agent run, the truncation probe, and their scripts |
| `configs/` | planner manual and few-shot plans; campaign configs of the task runs |
| `scripts/` | experiment drivers (`eval_*.py`), Slurm job scripts, and the generators of numbers, tables and figures (`paper_*.py`) |
| `tests/` | tests of the evidence layer and of the modules it uses |
| `docs/` | [`REPRODUCING.md`](docs/REPRODUCING.md): data, every experiment, cluster notes |

## Getting started

You need Python 3.12, [uv](https://docs.astral.sh/uv/), and a stable
Rust toolchain (the engine is a PyO3 extension built by maturin).

```bash
uv sync                          # builds the native engine, installs runtime + dev dependencies
uv pip install sqlglot==30.15.0  # needed by the top-k tests and the BIRD scripts
uv run pytest tests/
```

Some experiments need further packages; see
[docs/REPRODUCING.md](docs/REPRODUCING.md#setup-and-optional-packages).

## Reproducing the results, at three levels of effort

**(a) Regenerate every number, table and figure from the committed
receipts.** Standard library only (Python 3.12 or later); takes
seconds. Run from the repository root:

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

`paper_macros.py` must run first; the other generators read its output.

**(b) Rerun the CPU-only experiments.** No model and no external data.
The outputs go to a new directory; compare them with
`benchmarks/results-v1/`:

```bash
OUT=$(mktemp -d)
uv run python scripts/eval_fault_matrix.py         --json "$OUT/eval-fault-matrix.json"
uv run python scripts/eval_baseline_checkers_v2.py --json "$OUT/eval-baseline-checkers-v3.json"
uv run python scripts/eval_topk_cells.py           --json "$OUT/eval-topk-cells.json"
uv run python scripts/eval_verifier_scaling.py     --json "$OUT/eval-verifier-scaling.json"
uv run python scripts/eval_sql_snapshot.py --engines sqlite-wal,sqlite-rollback,duckdb \
    --workdir "$OUT/snap" --receipt "$OUT/eval-sql-snapshot.json"
uv run pytest tests/test_live_basis.py             # basis semantics under live correction
```

The snapshot stress test runs 1,000 trials per block and takes about
12 minutes; the other steps take seconds. Its PostgreSQL leg
(`--engines postgres`, receipt `eval-sql-snapshot-pg.json`) needs
`pgserver` and `psycopg`.

**(c) Rerun the runs that call an LLM.** These need a GPU, vLLM 0.11.0
serving `Qwen/Qwen2.5-14B-Instruct-AWQ`, and external data (BIRD
Mini-Dev databases, SNAP graphs) that this repository does not
redistribute. [docs/REPRODUCING.md](docs/REPRODUCING.md) gives the data
pins, the store-building steps and the cluster job scripts.

Every result, with its script, its receipt in `benchmarks/results-v1/`,
its place in the supplementary material, and what it needs to run, is
listed in [docs/REPRODUCING.md](docs/REPRODUCING.md#experiments).

## Provenance

This repository is a subset of github.com/zxf-work/tgms at commit
`bd4a9b12` (branch `agent-interface`), restricted to what the
experiments use. Every file is byte-identical to that commit, including
every receipt and frozen input, except: this README, `.gitignore`,
`scripts/paper_palette.tex`, the `authors` field of `pyproject.toml`,
and the two added files `ecqr_supplement.pdf` and
`docs/REPRODUCING.md`.

## License

Apache-2.0; see [LICENSE](LICENSE).
