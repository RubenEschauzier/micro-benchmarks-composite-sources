# Evaluation plan: FedShop axes and micro-benchmarks

Status: agreed 2026-11-25. Replaces [DESIGN.md](DESIGN.md) and [PLAN.md](PLAN.md) of 2026-11-19. Since then the
engine gained:
* the store drop (claimed triples are dropped from the store after link extraction);
* extended blocks (a star together with the star hanging off it), now sent as the star and the star with its hops;
* bound composites for constant-subject seeds.

The measured gains also came mostly from earlier answers rather than from saved join work. The section
[What changed](#what-changed-from-the-2026-11-19-design) lists which old axes were kept, merged or dropped.

## Configurations

| name | what it is | where it runs |
|---|---|---|
| **traversal only** | link traversal, no delegation | FedShop: the default point and coverage 0 only, as the floor |
| **pattern-only** | traversal + every triple pattern delegated to the pod's endpoint | everywhere: the baseline |
| **derived full** | traversal + composite sources (stars and chains of 2, star-first), whole composites | micro-benchmarks |
| **derived full + extended blocks** | as derived full, with extended blocks | micro-benchmark B3 only |
| **derived full + extended blocks + bound composites** | the final configuration | FedShop axes, if the full instance-0 run of 2026-11-25 confirms it; otherwise derived full |

The micro-benchmarks leave bound composites out: they compare plain composites with pattern-only.

## Part A: FedShop axes

Each axis varies its knob around the **default point** of the instance-0 experiment
(`~/jbr-experiment-fedshop-discovery-instance0` on the VM, `input/discovery-overrides.yaml`):

| setting | default |
|---|---|
| `members.shops`, `members.review_sites` | 20, 20 |
| `members.offers` (per shop), `members.reviews` (per site), `members.reviews_per_reviewer` | 1,000, 5,000, 100 |
| `members.pool` (products sold by shops) | 10,000 |
| `brands.count`, `brands.products` (over all brands), `brands.min_products` | 400, 30,000, 20 |
| `services.brands`, `services.shops`, `services.review_sites` | 1, 1, 1 |
| `known.ratio` | 0.5 |
| queries | instance 0 of the 16 templates |

### A1. Coverage

* **Knob:** the share f of *all discoverable* endpoints: `services.brands = services.shops = services.review_sites = f`,
  with `services.select: data` (a seeded random order per kind, so a larger share keeps every endpoint of a smaller
  one, and 25% is a random quarter of the data, not the largest pods).
* **Values:** f ∈ {0, 0.25, 0.5, 0.75, 1}.
* **The known members** (half the shops and review sites) always keep their endpoint: they are the federation the
  query starts from (`publishing.py`: `service = known or chosen`). So f = 0 is "known federation + traversal", not
  pure traversal. Pure traversal at 0 would need a generator change, which is not planned.
* **Expectation:** at f = 0 the derived configuration is as fast as pattern-only (the overhead, which is the number to
  report); the gain grows with f.
* **Generator work:** none; the shares exist.

### A2. Federation size

* **Knob:** the number of pods, with every pod the same size: scale `members.shops`, `members.review_sites` and
  `brands.count` together.
* **Values:** ×0.5, ×1, ×2, and ×4 only if ×2 stays within the timeouts and memory. Shops 10/20/40/80, brands
  200/400/800/1,600.
* **Scaled along**, so that the data per pod and the links between pods stay the same:
  * `members.pool` and `brands.products`, so shops per product and products per brand stay constant;
  * nothing per pod: `members.offers`, `members.reviews`, `members.reviews_per_reviewer` and `brands.min_products`
    stay at their defaults.
* **What it stresses:** composite requests (one per pod per block), discovery, data volume.
* **Note:** brands are about 95% of the pods, so pod count and shop data grow together. This is the realistic "the
  federation grows" question, as in FedShop's own batches.

### A3. Pod size

* **Knob:** the data per pod, with the number of pods fixed.
* **Values:** ×0.5, ×1, ×2, (×4) of `members.offers`, `members.reviews` and `brands.products` (products per brand).
* **Scaled along:**
  * `members.reviews_per_reviewer`, so the number of reviewer pods (reviews ÷ reviews per reviewer) stays the same;
  * `brands.min_products`, so the spread over brands keeps its shape.
* **Fixed:** the number of shops, review sites and brands; `members.pool`.
* **What it stresses:** rows per composite (q12 already gets 30,000 rows for 1 answer), documents per pod, what the
  store drop saves.

### Running and comparing

* **One jbr experiment per dataset** (a copy of the instance-0 experiment with its own overrides), with the
  configurations as combinations. Combinations share one generated data set (`commonGenerated`), datasets do not.
* **Replications:** 3 measured + 1 warm-up (the default point keeps its 6).
* **Timeouts:** raise the query timeout from 180 s at ×2 and ×4, and watch Virtuoso memory and disk there.
* **Queries are regenerated per dataset.** The constants differ per point, so speedups are compared within a point
  (against pattern-only), per template, with the result counts recorded.
* **Datasets:**
  * A1: 4 new;
  * A2 and A3: 2 new each, plus 1 each at ×4;
  * so 8–10 datasets, the default point shared. About 2–7 hours each at 2 configurations, so 2–3 days of VM time.

## Part B: micro-benchmarks on synthetic data

### B0. Irrelevant data, measured on FedShop

The engine parses and indexes every triple of every fetched document, not only the triples matching a query
pattern. So synthetic data without irrelevant triples would make ingesting unrealistically cheap and the relative
gain too large.

**Measured** with `analyse-derived-resource-data/notes/composite-investigation/scripts/measure_relevant_share.py`, on
the instance-0 data and the documents each run of the bound experiment fetched. A triple is relevant when it matches
at least one pattern of the query.

| query | fetched documents | triples | relevant | share |
|---|---|---|---|---|
| q12 | 5,282 | 202,788 | 23,118 | 11.3% |
| q13 | 4,578 | 203,018 | 23,661 | 12.0% |
| q14 | 3,304 | 161,572 | 27,309 | 16.6% |
| q16 | 3,136 | 166,081 | 27,402 | 16.4% |

Over all 16 templates (static: the documents holding a relevant triple), the median share is 15.3%. Two lookups are
lower (q02 4.1%, q09 2.7%) and two are higher (q08 63%, q10 35%).

Where the irrelevant triples are:

| query | of the irrelevant triples: same subject as a relevant triple | other subjects |
|---|---|---|
| q12 | 59% | 41% |
| q13 | 64% | 36% |
| q14 | 92% | 8% |
| q16 | 97% | 3% |

Almost no fetched document lacks a relevant triple (at most 2 of about 3,000 per run): broad patterns such as
`rdfs:label` and `rdf:type` touch nearly every document.

**Rule for the synthetic data:**
* **15% relevant triples**: the triples the query's patterns match, and that the composite and pattern-only arms
  ask the endpoint for.
* **85% irrelevant**: about three quarters extra properties of the subjects the query touches, and one quarter
  subjects it never touches, both in the documents that hold the relevant triples. No documents with only
  irrelevant data.
* **Irrelevant triples add no links:** their objects are literals, or IRIs in the same document. That keeps them from
  changing what traversal fetches; they only cost parsing, indexing and memory.
* **Their predicates** come from a vocabulary of their own, so they match no query pattern.

The share can be checked once at 5% and 50% on one point of B1, to see how much it moves the result.

### B1. Data and serving

* **Generator:** a small Python generator that writes the synthetic pods directly, in the output layout of
  fedshop-discovery: `web/<pod>/*.nq`, each storage root's `.meta` with `pim:Storage` and `void:sparqlEndpoint`,
  `publishers.json`, the endpoint graph files, `serving.json`, and `workload/qNN/instance_0/plain.sparql`. The
  existing serving (Solid server, Virtuoso, counting proxy) and runner (`up.sh` and `run_queries_stream.py` from the
  bound experiment) then work unchanged.
* **The generator also writes** the reference answer of every query, and the characterisation: matches per pattern,
  the composite's rows, the relevant share.
* **Pods:** P pods, each with N star subjects, grouped into documents of a fixed number of subjects (as FedShop's
  product pages). Every pod has an endpoint; every pod is seeded (no discovery in the micro-benchmarks).
* **Sizes to calibrate first** (P, N, subjects per document), so that pattern-only takes roughly 10–60 s at the
  middle point: long enough to measure, short enough for 5 replications.
* **Replications:** 5 measured + 1 warm-up per cell.

### B2. Selectivity: by a constant, and by the join

Two series over the same selectivities σ (star solutions ÷ star subjects), σ ∈ {1, 10⁻¹, 10⁻², 10⁻³}. Arms:
pattern-only and derived full.

| series | template | how σ is set | held fixed |
|---|---|---|---|
| **constant** | `?s :c <v> . ?s :p1 ?a . ?s :p2 ?b . ?s :p3 ?d` | `<v>` is a value held by σN subjects | `:p1`–`:p3` on 50% of the subjects each, including all those with `<v>` |
| **join** | `?s :p1 ?a . ?s :p2 ?b . ?s :p3 ?d . ?s :p4 ?e` (no constants) | the overlap of the four predicates: σN subjects have all four, no other subject has all four | each predicate on 50% of the subjects, at every σ |

* **Why two series:** with a constant, one pattern returns few rows on its own, so the adaptive join can start from
  it, and pattern-only's local work stays moderate. With join selectivity no single pattern shows it: pattern-only
  fetches, indexes and joins every match of every pattern, at every σ.
* **Join series, construction:** since every pattern's match count is the same at every point, pattern-only does the
  same work (triples, documents, requests) at every σ, and only the composite's result size changes. The difference
  between the arms is then the join alone.
* **Expectation:**
  * no gain at σ = 1;
  * the gain grows as σ shrinks, in first and median answers;
  * clearly larger for the join series than for the constant series at equal σ.

### B3. Hop locality

* **Template:** a star plus the hop hanging off its object:
  `?o :p1 ?a . ?o :p2 ?b . ?o :ref ?l . ?l :q ?x`.
* **Knob:** the share h of `?l` that lies in another pod than `?o`.
* **Values:** h ∈ {0, 0.25, 0.5, 0.75, 1}.
* **Arms:** pattern-only, derived full, derived full + extended blocks.
* **What it tests:** the star / star + hop split. A row whose hop leaves the pod falls back to the plain star.
* **Expectation:**
  * extended blocks are never worse than derived full;
  * the gain is largest at h = 0 and gone at h = 1.
  * FedShop sits at h = 0, which is why this could not be seen there.

### Metrics (every part)

| metric | source |
|---|---|
| first, median and last answer, total time, answers, correctness against the reference | runner (FedShop: jbr; micro: `run_queries_stream.py` with the generator's reference) |
| requests and bytes per kind (document, storage root, SPARQL) | counting proxy |
| engine CPU time and peak memory | container stats |
| composite rows, blocks, whole/bound decisions | engine trace (`DERIVED_BOUND_TRACE=1`) |

## Order of work

1. **A1 coverage:** overrides only, on the VM.
2. **A3 pod size, then A2 federation size:** overrides only (×0.5, ×2 first; ×4 last).
3. **The micro generator** (B1), calibrated on the middle point of B2.
4. **B2, join series first, then the constant series.**
5. **B3 hop locality.**

## What changed from the 2026-11-19 design

| old axis | now | why |
|---|---|---|
| S2 join work / gain | kept as B2, in two series | still the cleanest test of how selective a composite must be |
| M2 coverage | kept as A1, on FedShop itself | the generator has the shares; no extraction needed |
| S1 size at fixed documents | merged into A3 | A3 is the realistic version |
| M1 number of storages | merged into A2 | same question |
| S3 latency, depth, breadth | dropped | latency judged the least informative; layouts measure traversal, not composites |
| M4 hops between storages | dropped | measures traversal depth |
| S4 star-first vs chain-first | dropped | with an endpoint, the S4 template now becomes one extended composite |
| M3 star + chain across storages | replaced by B3 | anchors, base dedup and the split now answer its open question |
| "no pattern-only arm" | reversed | pattern-only is the strongest baseline |
| CPU model (composites only save join work) | replaced | they also save indexing and memory (store drop), and mostly bring answers earlier |
| raw-build extraction (`micro instantiate`) | replaced by a direct generator | hop locality and join selectivity need data that varies, not only constants |

Still open from the old design: the `constantCount` tie-break in `ActorDerivedResourceProposeStars.ts` is still
always 1 (operator precedence); fix it before the runs.
