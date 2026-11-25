# Implementation plan: micro-benchmarks

Status: **superseded** by [EVALUATION-PLAN.md](EVALUATION-PLAN.md) (2026-11-25). Draft of 2026-11-19. Goes with [DESIGN.md](DESIGN.md), which says *what* is measured; this file says *how* it is built. Items marked **(verify)** are assumptions not yet checked.

## Principles

1. **Restrict the data, not the engine.** An instance serves only the storages in its scope. The engine traverses whatever it finds, so nothing is needed in the engine to keep traversal inside the scope.
2. **One operation for every family: instantiate.** Every experiment is a *dataset instance*: a few storages, extracted from one raw build, optionally sub-sampled and re-laid-out. On top of that come *query instances*: a template plus constants. All families, from S1 to M4, are lists of these two things.
3. **Vary constants, not data.** Within one dataset instance, selectivity is set by searching the data for constants that hit a target, never by generating new data. A grid (S2, S4) is one dataset plus many query instances.
4. **Edit data sparingly.** There are only two edits. Both restate facts that already hold in the raw data, and both are applied at instantiation and recorded in the instance (see [Edits](#edits)).
5. **Self-contained.** One Python project in this directory, one CLI (`micro`), no git submodules, no changes to fedshop-discovery. The only work outside this directory is in the engine (configs and the chain-first actor), which the engine needs anyway.

## Directory layout

```
experiments-micro-benchmarks/
  DESIGN.md  PLAN.md  README.md
  pyproject.toml              # uv project: click, pyyaml, pyoxigraph, aiohttp, psutil, matplotlib
  config/
    raw-build.yaml            # fedshop-discovery overrides for the one raw build
    css-config.json           # copied from fedshop-discovery (file backend, .meta support)
    docker-compose.yml        # one Virtuoso container for all endpoints
    engines.yaml              # arm name -> engine config path (T, C-SF, C-CF)
  queries/                    # templates with named constant slots
    star.rq  star-leftover.rq  star-chain.rq  chain4.rq  s4.rq  m3.rq  m4-d{1,2,3}.rq
  families/                   # one file per family: datasets, query grids, arms, latency
    S1.yaml S2.yaml S3.yaml S4.yaml M1.yaml M2.yaml M3.yaml M4.yaml
  micro/                      # the Python package
    cli.py                    # micro raw | instantiate | search | serve | run | analyse | family
    rdf.py                    # read/write N-Triples, IRI helpers
    extract.py                # scope + sampling -> triples per storage
    layouts.py                # per-resource, product-pages, bucketed, nested
    edits.py                  # the two edits
    publish.py                # documents + storage roots (.meta) + endpoint graphs
    stats.py                  # characterisation with pyoxigraph
    search.py                 # constant search for grid targets
    serve.py                  # CSS + Virtuoso + proxy for one dataset instance
    proxy.py                  # counting/logging/delaying reverse proxy
    run.py                    # engine endpoint per arm, executions, measurements
    analyse.py                # tables and plots
  work/                       # gitignored: raw/, datasets/, queries/, results/
```

## Step 0: engine checks (before writing code)

These decide whether the design holds. Each is a short experiment against the existing small build.

| Check | Why | How |
|---|---|---|
| How seeds are passed (context `sources`, or IRIs in the query) | The runner must start traversal at the storage roots in scope | read the jbr experiment's runner (`FedShopDiscoveryRunner.ts`) |
| C discovers an endpoint from a storage root's `.meta` (`void:sparqlEndpoint`) alone | Extraction writes these files itself | one star query on one shop |
| The partition is logged | DESIGN's metrics need it; replace the leftover `console.log` | read the partition actor |
| Whether delegated units are joined by hash join on full results, or by bind join (one unit's bindings pushed into the other's endpoint query) | Decides what S4 can show. With full results only, SF and CF move about the same amount of data, and S4 will show little difference. | log the endpoint queries of one star + arm query under C |
| The HTTP endpoint's cache invalidation flag (`--invalidateCache`) | One engine per (dataset, arm) instead of a fresh engine per run | Comunica docs / `bin/http.js --help` |

## Step 1: the raw build

One fedshop-discovery build, used read-only by everything after it. Only the steps that need no Docker run: `catalog`, `members`, `publish`.

```bash
uv run --project ../../fedshop-discovery fedshop-discovery --overrides config/raw-build.yaml \
    catalog  ../../fedshop-discovery/experiments/bsbm/snakefile/config_small.yaml
# same for members, publish
```

`micro raw` wraps these three calls and copies `publishers.json` and the per-publisher graph files into `work/raw/`.

`config/raw-build.yaml`:

| Key | Value | Why |
|---|---|---|
| `members.shops` | 16 | M1 needs up to k = 16 shops |
| `members.offers` | 5000 | The small build has about 20 offers per shop, far too little to be CPU-bound. 5000 gives selectivities down to 10⁻³ (5 offers). |
| `members.review_sites`, `members.reviews` | 4, 2000 | Enough for M4's hops; nothing else uses them |
| `members.pool` | 200 | About 25 offers per listing, so chains have a range of selectivities |
| `layout.shops`, `layout.review_sites` | all per-resource | Raw layout; `micro` re-lays out at instantiation |
| `services.*` | 1.0 | Every publisher has an endpoint in the raw build; an instance decides which ones it advertises |
| `links.*` | 0 | No seeAlso links between storages |
| `markets` | off | |
| brand-record placements | off | |

Everything this file fixes is a generation parameter, not a data edit. **(verify)** that WatDiv generates 16 × 5000 offers in reasonable time, and the spread of offers per listing.

## Step 2: dataset instances (`micro instantiate`)

A dataset spec (inline in a family file, or standalone YAML):

```yaml
id: s1-n1000-b64
storages:
  - publisher: vendor3
    sample: {offers: 1000}            # optional: the first n offers in raw order (a random sample, since
                                      # WatDiv numbers offers randomly); listings and the vendor record
                                      # follow the offers that are kept
    layout: {kind: bucketed, documents: 64}
    endpoint: true                    # advertise void:sparqlEndpoint in the storage root
  - publisher: producer12
    layout: {kind: per-resource}
    endpoint: true
edits: [listing-vendor, listing-types]
```

`micro instantiate <spec>` writes `work/datasets/<id>/`:

```
web/<publisher>/...              # one file per document, plus the storage root's .meta
graphs/<publisher>.nt            # what that publisher's endpoint serves
dataset.json                     # the spec, plus counts: triples, entities, documents per storage
seeds.json                       # storage roots: the seeds of every query on this dataset
```

The steps, all in plain Python over N-Triples:

1. **Extract** (`extract.py`): read the raw graphs of the publishers in scope; apply `sample`.
2. **Edit** (`edits.py`): apply the listed edits.
3. **Lay out** (`layouts.py`): map every entity of a storage (a subject IRI in its namespace) to a document IRI, and rewrite the IRIs everywhere in the dataset (subjects and objects, documents and graphs). Layouts:
   - `per-resource`: one document per entity;
   - `product-pages`: a listing and its offers in one document;
   - `bucketed` (N): entities spread over N documents by their number, `<storage>/data/<i mod N>#<local>`;
   - `nested` (d, b): `<storage>/data/<i_1>/…/<i_d>#<local>`, with fanout b per level. `bucketed` is `nested` with d = 0.
4. **Publish** (`publish.py`): write one file per document (the subject triples of its entities); write each storage root's `.meta` (`pim:Storage`, and `void:sparqlEndpoint <port>` when `endpoint: true`); write the endpoint graphs. Containers come for free: the Solid server's file backend lists directories (`ldp:contains`).

**What the restriction does:** IRIs that point outside the scope (for example a listing's `fs:listingOf <brand product>` when the brand is not in scope) are on the same host, but nothing serves them. They give a fast 404, the same in both arms, and the proxy counts them under their own kind.

**(verify)** the file naming and content types that css-config.json expects (copy the convention from fedshop-discovery's `fragment.py`).

### Edits

Two edits, both copying facts that already hold in the raw data:

| Edit | Triples added | Why | Used by |
|---|---|---|---|
| `listing-vendor` | `<listing> bsbm:vendor <vendor record>`, one per listing | Gives a chain of 3 inside one shop (offer → listing → vendor record). Without it every chain leaves the shop after the listing. | star-chain type |
| `listing-types` | `<listing> rdf:type <T>` for each type of the listed product (copied from the brand's graph) | Restores what the discovery variant dropped from listings. Product types form a hierarchy, broad to narrow, so `?l rdf:type <T>` gives a whole range of selectivities on the listing side. | S4 chain selectivity, S2 shop half of the chain of 4 |

## Step 3: query instances (`micro search`)

A query template has named constant slots:

```sparql
# s4.rq: star of 3 + chain of 2 sharing ?o bsbm:product ?l
SELECT * WHERE {
  ?o {{star_pred}} {{star_value}} .  ?o bsbm:price ?pr .  ?o bsbm:product ?l .
  ?l rdf:type {{chain_type}} .
}
```

Each query instance carries a **target** per delegated unit (a selectivity: solutions of the unit ÷ offers in scope). `micro search` fills the slots:

1. Load the dataset's graphs into an in-memory pyoxigraph store.
2. Per slot, list the candidates with their counts in one query: `SELECT ?v (COUNT(*) AS ?n) WHERE { <unit with ?v in the slot> } GROUP BY ?v`. Candidate predicates per slot come from the template, for example `star_pred ∈ {rdf:type, bsbm:deliveryDays, bsbm:validFrom, bsbm:validTo}`. `?o rdf:type bsbm:Offer` is the selectivity-1 candidate, so the query shape stays the same at every level.
3. Per target, pick the candidate closest in log scale. Accept it within a factor 1.5. Among equally close candidates, prefer one that gives the full query at least one result.
4. Run the full query and every sub-BGP each partition can delegate, and record the **achieved** values next to the targets.
5. Write `work/queries/<id>/query.rq`, `reference.json` (the answer and its hash) and `stats.json` (the characterisation of DESIGN.md: matches per pattern, solutions per unit, gain, local join input per partition, leftovers, query type).

Targets that no candidate reaches within the factor are reported in `search-report.csv` and left out. They are not filled by editing data.

## Step 4: serving (`micro serve <dataset>`)

- **Solid server:** CSS over `web/` with `config/css-config.json`, on an internal port.
- **Endpoints:** one Virtuoso container (`config/docker-compose.yml`). `serve` clears it, loads `graphs/<publisher>.nt` into one graph per publisher, and creates one virtual endpoint per publisher with `endpoint: true`, on its own internal port. It uses the same isql statements as fedshop-discovery's `store.py` (about 30 lines, copied rather than imported).
- **Proxy** (`proxy.py`, aiohttp): owns the public ports (documents on 3000, one per endpoint). Per request it:
  - forwards to CSS or the endpoint;
  - optionally waits a fixed delay first (the latency factor, the same for documents and SPARQL);
  - appends `{t, port, path, kind, status, bytes}` to a JSON-lines log, where `kind` is document, root, sparql or 404.
- **Resource sampling:** `serve` samples the CPU time of CSS and the Virtuoso container (psutil and the cgroup) for the run log.

Serving happens once per dataset. All query instances and arms on that dataset run against the same servers (warm servers).

## Step 5: running (`micro run <family>`)

Per dataset: serve it. Per arm: start one engine HTTP endpoint (`node <engine>/bin/http.js -c <config> --invalidateCache`). Per query instance: 1 warm-up, then 5 measured executions. Per execution, the runner records:

| Field | How |
|---|---|
| total time, first and last arrival, result count, result hash | client side, streaming SPARQL JSON results |
| correct | hash equals `reference.json` |
| requests and bytes per kind | proxy log lines between start and end |
| **tail** | last arrival − time of the last document request in the proxy log (same clock, the same machine) |
| engine CPU time | `/proc/<pid>/stat` utime + stime of the endpoint process, before and after |
| server CPU time | from `serve`'s sampler |
| partition | from the engine log (Step 0) |

Results go to `work/results/<family>/<dataset>/<query>/<arm>.jsonl`. A timeout (default 300 s) is recorded as such and excluded from the metrics. The engine endpoint is pinned to fixed cores (`taskset`), and the servers to others.

## Step 6: the families as files

Each family file lists datasets, query instances (as targets, filled by `search`), arms and latencies. `micro family S4` runs instantiate → search → run → analyse for that family. How each family maps to datasets and query instances:

| Family | Datasets | Query instances per dataset |
|---|---|---|
| S1 | the largest shop at offer samples n ∈ {500, 1000, 2000, 3000, 5000}, × {bucketed N, per-resource} | 4 types at one fixed selectivity (target 0.01) |
| S2 | the largest shop at 2000 offers, bucketed N | 4 types × 6 gain targets, plus the type-specific knobs |
| S3 | the same shop and sample, × 7 layouts | 4 types at the highest S2 gain; latency 0 and 80 ms are runs, not datasets |
| S4 | **one** dataset: the largest shop at 5000 offers, bucketed N | the 4 × 4 grid below |
| M1 | k ∈ {1, 2, 4, 8, 16} shops sharing a product, 2000 offers each | 4 types at the S2 middle gain |
| M2 | the k = 8 dataset with endpoints advertised for a fraction f of the shops | same as M1 |
| M3 | 3 shops, each with the brands of its listed products | m3.rq, SF |
| M4 | shop → brand → review site → reviewer storages, depth d | m4-d.rq |

### The S4 grid

S4 is one dataset and one template (`s4.rq`), so the grid differs only in the constants. Two targets:

- **star selectivity** σ_star ∈ {1, 10⁻¹, 10⁻², 10⁻³}, set through `star_pred`/`star_value` (`?o rdf:type bsbm:Offer` for 1, `bsbm:deliveryDays` for about 10⁻¹, `bsbm:validTo` dates for the smaller values);
- **chain selectivity** σ_chain ∈ {1, 10⁻¹, 10⁻², 10⁻³}, set through `chain_type`: a root product type for 1, narrower types for the smaller values (from the `listing-types` edit).

That makes 16 cells, run with T, C-SF and C-CF. The plot is a heatmap of time(C-CF) ÷ time(C-SF) over the grid, plus a line plot of the same ratio against log(σ_star ÷ σ_chain). The DESIGN hypothesis is that CF wins above the diagonal (chain more selective) and SF below it. If Step 0 finds that delegated units are joined on full results only, expect the ratio to stay near 1 everywhere, which is then itself the result.

## Step 7: analysis (`micro analyse`)

One table per family (median and spread of the 5 runs per cell, timeouts, wrong answers), and the plots of DESIGN.md: speedup T ÷ C per axis with one panel per query type, the tail of T as the regime indicator, and the S4 heatmap. Plots read only `work/results/` and `work/queries/*/stats.json`.

## Order of work and milestones

1. **Step 0 checks.** Write the findings into DESIGN.md.
2. **Raw build + `instantiate` + `serve`** for one dataset (one shop, per-resource). Milestone: T answers the star query, and its result hash matches the reference from pyoxigraph.
3. **The C arm on the same dataset.** Milestone: C answers correctly; the proxy log shows the SPARQL requests; the partition is logged.
4. **`search` + the S4 grid** (one dataset, 16 cells). This is the first full family, because it needs no new layout.
5. **Layouts** (`bucketed`, `nested`, `product-pages`), then S1–S3.
6. **Multi-storage datasets**, then M1–M4.
7. **Analysis and plots** throughout; final tables last.

Engine work, outside this directory, in parallel with steps 2–4: the C configs (stars + chain proposers, with SF and with CF), the chain-first partition actor, the `constantCount` fix, and the partition log.
