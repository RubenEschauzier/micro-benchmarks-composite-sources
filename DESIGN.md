# Micro-benchmark design: when does composite delegation help?

Status: **superseded** by [EVALUATION-PLAN.md](EVALUATION-PLAN.md) (2026-11-25). Draft of 2026-11-19. Items marked **(verify)** are assumptions not yet checked against the code or data.

## Goal

Find out how much execution time delegating sub-queries to composite sources saves, and which properties of the data topology and the query decide it. Two phases:

1. **Single storage.** For one storage, when does delegating to its composite source beat evaluating the same patterns locally, and which properties of the data and the query decide it?
2. **Multiple storages.** Do the single-storage trends hold when a query spans several storages? Do the paper's claim-specific predictions (Section "Star- and chain-shaped sub-queries") show up?

The design is a **matrix**: every micro-benchmark (an *axis*: storage size, join work, topology, number of storages, coverage) is run for every **query type** (star, star + leftover, star-chain, chain). The axes ask *where* delegation pays off; the query types ask *for which shape of query*.

The micro-benchmarks are **hand-crafted query templates over FedShop-discovery data**. There is no separate data generator. Each query type is one template, and each axis is a list of constants and scopes for it. The constants are picked by a fixed rule (for example quantiles of a measured property), never by hand. Every instance is described by measured values, not assumed ones (see [Characterisation](#characterisation)).

## What delegation saves: the CPU model

The engine traverses the whole storage in both arms. C does not skip documents: it fetches the same documents as T, plus one request per delegated unit. So:

- **I/O is the same in both arms.** That covers fetching, the sequential round-trips through containers, and (roughly) parsing every document.
- **What C saves is local join work.** For each delegated unit the endpoint returns joined solutions. The engine no longer joins those patterns' matches itself.
- **What C pays:** discovering the endpoint, the endpoint's own join time, and moving the delegated solutions to the client.
- **Neither arm can finish before traversal ends**, because only then does the engine know it has seen all data.

So total time is roughly max(I/O time, local CPU time) plus whatever doesn't overlap. C beats T only when **T is CPU-bound**, meaning its local join work keeps running after the last document has arrived. When T is I/O-bound, C saves CPU time that was hidden behind the network anyway, and the speedup is about 1. Time to first result is the exception: delegated solutions can arrive before traversal ends.

The axes vary two things and look for where the regime flips:

- **I/O load**, set by topology: number of documents (breadth), container depth, number of storages, and network latency.
- **CPU load**, set by the query and the data: matches per pattern, number of patterns in a delegated unit, and how much the delegated join reduces them (the delegation gain).

The main plot is the speedup over the plane (I/O load, CPU load), with one panel per query type.

## Query types

A query type is defined by **how C partitions the query**: how many delegated units it gets, and what join work stays local. Only stars and chains of 2 are delegated (see [Delegated units](#delegated-units-are-stars-and-chains-of-2-never-the-whole-bgp)).

**The whole matrix uses the star-first partitioner (SF)**, and every instance is chosen so that SF's partition is the right one for it. The axes then measure delegation, not partitioning. How well the partitioner chooses is tested separately, in [S4](#s4-partitioner-star-first-vs-chain-first).

| Type | Template | Delegated units under SF | Local join work in C |
|---|---|---|---|
| **Star** | star on `?o` | 1 star | none |
| **Star + leftover** | star of 3 on `?o` + an arm of one pattern past it | 1 star (absorbs the arm's first pattern) | delegated solutions ⋈ matches of the leftover pattern |
| **Star-chain** | star of 3 on `?o` + a chain of 3 that shares one pattern with the star | 1 star + 1 chain of 2 | one join between the two units |
| **Chain** | chain of 4 | 2 chains of 2 | one join between the two units |

Templates, in a shop storage (offers, listings, the vendor record):

```sparql
# Star (s patterns, constants set the gain)
?o bsbm:deliveryDays <d> . ?o bsbm:price ?pr . ?o bsbm:validTo ?t . …

# Star + leftover: SF delegates the star of 3; ?l fs:listingOf ?p is left over
?o bsbm:deliveryDays <d> . ?o bsbm:price ?pr . ?o bsbm:product ?l .
?l fs:listingOf ?p .

# Star-chain: star of 3, chain of 3 sharing ?o bsbm:product ?l.
# SF delegates the star and the chain {?l bsbm:vendor ?v . ?v bsbm:country ?c}; nothing is left over
?o bsbm:deliveryDays <d> . ?o bsbm:price ?pr . ?o bsbm:product ?l .
?l bsbm:vendor ?v . ?v bsbm:country ?c .

# Chain of 4: shop half, then brand half
?o bsbm:product ?l . ?l fs:listingOf ?p .
?p bsbm:producer ?b . ?b rdfs:label ?bl .
```

Notes on the types:

- **Star + leftover vs star-chain:** both have a star of 3 whose last pattern starts an arm. With an arm of one pattern past the star, SF leaves that pattern over. With an arm of two patterns, SF delegates the arm as a chain of 2. In both, the star is the selective unit, since that is the case where star-first is the right choice.
- **The in-shop hop.** FedShop-discovery has no chain of 3 inside one storage. A listing has only `rdf:type` and `fs:listingOf`, so after `?o bsbm:product ?l` a chain leaves the shop. The `listing-vendor` edit therefore adds one triple per listing, `<listing> bsbm:vendor <vendor record>` (see [Data edits](#data-edits)). Then the star-chain's chain {vendor, country} has shop subjects only, and both units fall within the shop's claim. That chain returns one row per listing, so it is unselective, consistent with the star being the selective unit.
- **A leftover costs no extra I/O**, since the storage is traversed anyway. It brings join work back to the client, in proportion to its matches. So the leftover is `?l fs:listingOf ?p` (one match per listing), not a vendor pattern (one match per shop).
- **A chain of 2 is not a separate type.** Like a star, it is one delegated unit with no local join left, so at equal gain it should behave like a star. A chain only brings something new from 4 patterns on: two units that the engine joins itself. Chains of 2 sharing a pattern with a star appear only in S4.
- **The chain of 4 spans two storages.** Even with the in-shop hop, a shop has no chain of 4 of its own. Each half is claim-uniform, though: {product, listingOf} has shop subjects, {producer, label} has brand subjects. In Phase 1, its scope is the shop being varied plus **one fixed brand**, and the axis is applied to the shop. **(verify)** which split the partitioner picks. The middle chain {listingOf, producer} overlaps both halves. Picking it leaves two leftovers, and it crosses from the shop to the brand, whose endpoint has no shop data. Record the partition per run.
- **Why shops, not brands.** In a brand, the only arm or chain ends at the single brand record, so the leftover has one match and no chain of 3 exists. Offers have about 9 properties **(verify)**, so stars go up to about 8 patterns.
- **Constants keep the shape fixed.** Each unit has one constant slot. The selectivity-1 level uses a constant that matches everything (`?o rdf:type bsbm:Offer`, a root product type), so every level of a sweep has the same query shape.

## Restricting the data: dataset instances

The scope of an experiment is enforced by **the data, not the engine**. Every experiment runs on a *dataset instance*: the storages in scope, extracted from one raw FedShop-discovery build, and nothing else is served. The engine traverses whatever it finds, so no engine actor is needed to keep traversal inside the scope. (How it is built: [PLAN.md](PLAN.md).)

- **Raw build:** one FedShop-discovery build with larger shops (16 shops of about 5000 offers; the small build has about 20 per shop), every publisher with an endpoint, no seeAlso links, no markets, no brand-record placements. Everything else is derived from it.
- **Extraction:** a dataset instance names its storages, optionally a sample of a shop's offers (the first n, a random sample), a layout per storage, and whether each storage advertises its endpoint. Layouts are applied at extraction, so they need no change in FedShop-discovery: `per-resource`, `product-pages`, `bucketed` (N documents), `nested` (depth d, fanout b).
- **Seeds:** the storage roots in the instance.
- **Out-of-scope IRIs** (for example `fs:listingOf <brand product>` without the brand) are on the same host, but nothing serves them. They give a fast 404, the same in both arms, counted as a separate kind.
- **Reference answer and characterisation:** computed on the instance's own triples (pyoxigraph), so they see exactly what the engine can see.
- **Query instances:** a template plus constants. Within one dataset instance, selectivity is set by **searching the data for constants** that hit a target, never by generating data. A grid (S2, S4) is one dataset instance and many query instances.

### Data edits

Only two edits, both copying facts that already hold in the raw data, both applied at extraction and recorded in the instance:

| Edit | Triples added | Why |
|---|---|---|
| `listing-vendor` | `<listing> bsbm:vendor <vendor record>` | Gives a chain of 3 inside one shop (offer → listing → vendor record), needed for the star-chain type. Without it every chain leaves the shop after the listing. |
| `listing-types` | `<listing> rdf:type <T>` for each type of the listed product (copied from the brand) | Restores what the discovery variant dropped from listings. Product types run from broad to narrow, so `?l rdf:type <T>` gives a full range of chain selectivities (S4, S2). |

## Engine arms

Each arm is defined by which propose actors are enabled (`derived-resource-propose/actors/`). The same endpoint identify actor is enabled in all of them.

| Arm | Composite delegation | Propose actors | Config |
|---|---|---|---|
| **T** (default) | no | none | traversal + adaptive join (STeMS lottery) |
| **C** (composite) | yes | `actor-stars` + `actor-chain` | **missing** |

- **T vs C** is the main comparison: how much execution time does delegating sub-queries save?
- C runs with partitioner **SF** (star-first, exists) throughout the matrix. **CF** (chain-first, **missing**) appears only in S4, which compares the two.
- Single triple patterns are never delegated (no `actor-triple-pattern`). In C, a pattern outside every delegated unit is evaluated locally, exactly as in T.
- No pattern-only arm. In the setting of the paper, pattern delegation is faster by construction, so comparing against it would show nothing.

Per point on an axis, that gives **8 runs**: 4 query types × (T, C).

### Delegated units are stars and chains of 2, never the whole BGP

An endpoint accepts any BGP, but the engine only proposes stars and chains of at most 2 patterns (`maxChainLength: 2`), and it delegates them separately. That is required for correctness: under subject-namespace claims, a star is claim-uniform, and a chain of 2 has its claim status decided at its first pattern (paper §shapes). Delegating a larger BGP risks losing solutions that mix claimed and unclaimed triples. So even inside one storage the partitioner makes a real choice.

### Kept identical across arms

- **Network latency.** Today only the derived-resource configs import `actors-delay.json` (80 ms); the T baselines don't. Pick one of two fixes, and use it for every arm:
  - add a fixed delay in the FedShop counting proxy, so it applies to documents and endpoints alike; or
  - import the same HTTP delay into every engine config.

  Latency is also a factor of its own in S3: it is the cheapest way to move a query from CPU-bound to I/O-bound.
- **Server cache policy.** Choose cold (restart servers per run) or warm (one discarded warm-up, then N runs).
- **Bug fix applied to all arms:** the `constantCount` tie-break in `ActorDerivedResourceProposeStars.ts:66-68` is always 1 (operator precedence).

## Phase 1: single storage

Scope = one shop (plus one fixed brand for the chain of 4). S1–S3 are run for all four query types, with SF. S4 is the separate partitioner test.

### S1. Size of the data at a fixed number of documents

**The number of documents is held fixed.** With one document per offer, a larger shop has both more triples and more documents, so CPU load (join work over the matches) and I/O load (requests) grow together, and a trend in the speedup can't be pinned on either. S1 therefore fixes the documents and varies only the data. S3 does the opposite: it varies the documents and fixes the data. Together they give the two edges of the (I/O load, CPU load) plane.

- **Varies:** the amount of data, as samples of **one** shop (the largest): its first n offers, n ∈ {500, 1000, 2000, 3000, 5000}. Taking samples of one shop rather than different shops keeps the data distribution the same, so only the size changes.
- **Held fixed:**
  - **the number of documents, N.** Every sample is published with the `bucketed` layout: its entities are spread over N documents. The same N is used for every sample, and N ≤ 500 so no bucket is empty. **(decide)** N.
  - **selectivity.** The constant search picks constants per sample so that the target selectivity (0.01) holds at every size.
- **Reference series (realistic):** the same samples in the `per-resource` layout (one document per entity), so documents grow with the data. It shows where real data sits between the two clean edges, S1 (data only) and S3 (documents only).
- **Side effect to keep in mind:** at fixed N, documents grow with the data, so parsing time grows too. C does not save parsing, since it still fetches and parses every document. Part of the growing CPU load can't be saved, so the speedup rises less than the join work alone would suggest. The engine CPU time and the tail show how much of the growth is join work.
- **Path across the plane:** CPU load only (fixed-N series); I/O and CPU load together (reference series).
- **Hypothesis:**
  - *Star:* at fixed N, the I/O stays about the same (only document size grows), while T's join work grows with the matches. So T moves toward CPU-bound as the data grows, and the speedup grows with size. On small shops C loses, because discovering and calling the endpoint is a fixed cost that the saved join work doesn't cover. **The crossover point is the result.** In the reference series both arms also grow with the number of documents, so the speedup grows more slowly or stays flat. The gap between the two series is the effect of the document count.
  - *Star + leftover:* like star, but shifted toward 1, because the leftover's matches grow with the shop too.
  - *Star-chain:* the local join between the two units grows with the shop, because the unselective chain returns one row per listing.
  - *Chain:* the brand half is fixed, so only the shop half's saving grows with size.

### S2. Join work at fixed I/O

One dataset instance (the largest shop, 2000 offers, bucketed), so the documents are the same for every query instance. Varying the CPU load, only through the constants.

**Instances are chosen so that SF's partition is the right one**, because the partitioner is tested separately in S4. For the star + leftover and star-chain types this means the star is the selective unit. For the chain of 4 it means both halves are selective.

- **Delegation gain (all types):** constants chosen so the gain of the delegated units (matches per pattern ÷ solutions of the unit; the same definition as `analysis.json`) spans about 1× to 1000×. Pick by quantiles of the gain.
  - *Star, star + leftover, star-chain:* the gain is set on the star (through `<d>` and further constants).
  - *Chain:* both halves are selective and are swept together at about equal gain. The shop half is made selective through a constant on the offer side, the brand half through a constant on the brand record instead of `?bl` (**(verify)** which predicates the record has).
- **Type-specific knob**, one per type, each moving the work that only that type has:
  - *Star:* star size s ∈ {2, 4, 6, 8}, without constants. More patterns means more local joins in T, from the same documents.
  - *Star + leftover:* the leftover's matches that survive the local join, set through the star's selectivity (3 levels by quantile).
  - *Star-chain:* none beyond the gain sweep. The chain is fixed by the data (one vendor per shop).
  - *Chain:* two extra points with only one half selective (shop half only, brand half only). The partition is the same as in the sweep, so this does not test the partitioner. It measures the cost of the local join between two delegated units when one of them returns a large result.
- **Path across the plane:** CPU load only.
- **Hypothesis:** this is where C's saving is largest. The speedup grows with the gain and with the star size. At a gain of about 1, delegation saves no join work, and C loses on the endpoint's join time and on moving the solutions. For the two types with two units, the local join between them caps the saving. The speedup is largest when both units are selective, and when one unit is unselective it falls back toward what T does.

### S3. Topology inside one storage: breadth, depth, latency

S3 is the counterpart of S1: the data stays the same and the documents vary.

All layouts are applied when the dataset instance is extracted, so the same triples get different IRIs and documents, and the endpoints, reference answers and constants follow. `per-resource` and `product-pages` are flat, since the storage root lists all documents (`ldp:contains`). `nested` adds depth:

- **What it does:** entity IRIs become `vendor13/data/<i_1>/…/<i_d>#Offer77`. The bucket indices are derived from the entity number with fanout b.
- **Why this creates depth:** the Solid server lists only a container's direct children, so a traversing engine has to walk root → level 1 → … → document. That is d+1 sequential round-trips.

The axis:

- **Instance:** the S2 dataset's sample (largest shop, 2000 offers) under each layout; per type, the highest-gain query instance from S2, so there is join work to save. The constants are searched again per layout, because the IRIs change.
- **Breadth:** `per-resource` (one document per entity), `product-pages`, and `nested` with d = 1 and varying b.
- **Depth:** `nested` with d ∈ {1, 2, 3, 4} at a fixed number of documents.
- **Latency:** 0 and 80 ms per request.
- **Path across the plane:** I/O load only, since the triples and the join work are the same.
- **Hypothesis:** the speedup is largest with few, large documents and no latency, and drops toward 1 as documents, depth or latency grow and T becomes I/O-bound. Depth only matters with latency, because the round-trips are sequential. The types should flip to I/O-bound in the order of their saved join work: types that save less join work flip first.

### S4. Partitioner: star-first vs chain-first

The matrix fixes SF and picks instances where SF is right. S4 tests the choice itself, on the case where it matters: a star of 3 and a **chain of 2** that share one pattern.

**Where the decision comes from:** in `?s p1 ?a . ?s p2 ?b . ?s q1 ?x . ?x q2 ?y`, the pattern `?s q1 ?x` belongs to both the star and the chain.
- SF delegates the star {p1, p2, q1} and leaves `?x q2 ?y` over.
- CF delegates the chain {q1, q2} and the remaining star {p1, p2}.

**What decides the winner under the CPU model:** both partitions traverse the same documents. They differ in the **local join input**: the solutions of each delegated unit, plus the matches of each leftover pattern. Characterisation computes it per partition, and the partition with the smaller input should win.

- **Dataset:** one instance (the largest shop, all 5000 offers, bucketed, with the `listing-types` edit), at 0 ms latency, so the query is CPU-bound and the difference can show.
- **Template:** one shape for every cell; only the constants change.
  ```sparql
  ?o <star_pred> <star_value> . ?o bsbm:price ?pr . ?o bsbm:product ?l .   # star of 3
  ?l rdf:type <chain_type> .                                                # chain {product, rdf:type} shares ?o bsbm:product ?l
  ```
- **Grid of selectivities:** σ_star × σ_chain, each ∈ {1, 10⁻¹, 10⁻², 10⁻³}: 16 cells.
  - σ_star = solutions of the star ÷ offers, set by `star_pred`/`star_value`: `rdf:type bsbm:Offer` (1), `bsbm:deliveryDays` (about 10⁻¹), `bsbm:validTo` dates (smaller).
  - σ_chain = solutions of the chain ÷ offers, set by `chain_type`: a root product type (1) down to narrow types.
  - The constants come from a search over the dataset (closest candidate in log scale, accepted within a factor 1.5). The achieved values are recorded and used as the axes. Cells no constant reaches are reported and left out, not filled by editing data.
- **Arms:** T, C-SF, C-CF. Record the exact partition each arm chooses.
- **Plots:** a heatmap of time(C-CF) ÷ time(C-SF) over the grid, and the same ratio against log(σ_star ÷ σ_chain), the selectivity difference.
- **Hypothesis:** the ratio follows the selectivity difference. CF wins where the chain is more selective than the star, SF wins on the other side, and they tie on the diagonal. So the fixed star-first heuristic loses on half the grid. This is the paper's TODO "show star-only fails". **(verify, PLAN Step 0)** whether the engine joins delegated units by bind join. If it only joins their full results, SF and CF move about the same data, and the ratio stays near 1 everywhere. That would itself be the result.

## Phase 2: multiple storages

M1 and M2 run for all four query types. M3 and M4 are defined by a shape that crosses storages, so they belong to one type each.

### M1. Number of storages k

- **Instance:** each type's template over k shops that all sell a common product, k ∈ {1, 2, 4, 8, 16}. For the chain of 4, the scope also includes the brands the shops list.
- **Scope:** all storages seeded, so discovery stays out of the measurement.
- **Held fixed:** the matches per shop, roughly. Pick shops of similar size.
- **Hypothesis:** fetches from k servers run in parallel, but the engine's join work runs in one process. If so, T becomes more CPU-bound as k grows, and the speedup grows with k. If the engine's request parallelism is capped globally rather than per host **(verify)**, I/O grows with k too, and the speedup stays close to the single-storage value.

### M2. Coverage: how many storages have a composite

- **Instance:** M1 with k = 8. A fraction f ∈ {0, 0.25, 0.5, 0.75, 1} of the shops keep their endpoint.
- **Datasets:** the k = 8 dataset instance, extracted once per f, with `endpoint: true` for that fraction of the shops (the others' storage roots don't advertise one).
- **Hypothesis:** the saved join work is proportional to f, so in the CPU-bound regime the speedup grows with f. Answers stay correct when the plan mixes delegated and traversed storages. This is the deduplication case from the TODO list. For the chain of 4, a shop without an endpoint turns the shop half into local work, while the brand half stays delegated.

### M3. Star-chain across storages (paper prediction)

- **Type:** star-chain, but the chain crosses into another storage instead of staying in the shop:
  ```sparql
  ?o bsbm:deliveryDays <d> . ?o bsbm:price ?pr . ?o bsbm:product ?l .   # star of 3 on offers, in the shop
  ?l fs:listingOf ?p . ?p bsbm:producer ?b .                             # chain ?l → ?p → ?b: ?l is in the shop, ?p in a brand
  ```
  SF delegates the star and the chain {listingOf, producer}. That chain is not claim-uniform. **(verify)** what C does with it: the shop's endpoint has no producer data, so delegating it there would drop answers.
- **Scope:** one shop and the brands it lists.
- **Prediction (paper, §shapes):** the star falls entirely within the shop's claim, so delegating it could also save fetches from the shop's documents. The cross-storage chain is not claim-uniform: it saves join work but not fetches.
- **What the current engine can show:** only the join-work half. It traverses every storage in full, so no fetches are saved for either shape. The fetch half of the prediction needs an engine that stops traversing a storage whose claimed patterns are all delegated. That is out of scope here; note it in the paper as a limitation or future work.

### M4. Depth: hops between storages

- **Type:** chain, extended one storage hop at a time: shop offer → listing → brand product → review page at a review site → review. Depth d = 1, 2, 3.
- **Hypothesis:** each hop adds sequential discovery (I/O) but only one or two joins. So the speedup drops toward 1 with depth, the same effect as container depth in S3, but across storages.

## Characterisation

`micro search` (PLAN Step 3) records, per query instance and before any engine runs, from the dataset instance's own triples:

- the query type and the scope (storages, seeds);
- the I/O load: the documents in scope, the container depth, and the documents holding at least one match;
- the CPU load: matches per pattern within the scope, and the solutions of every sub-BGP each partition could delegate, and from those the delegation gain;
- per partition (T, SF, CF): the delegated units, the leftover patterns, and the **local join input** (solutions of the delegated units + matches of the leftovers; for T, the matches of all patterns);
- the reference answer and its hash.

These numbers become the x-axes of the plots. Constants are picked by searching these same numbers for the closest match to each target, so the selection is reproducible. The local join input is also a ready-made cost function for a cost-based partitioner.

## Metrics

| Metric | Source |
|---|---|
| total time, time to first result, arrival times | jbr / sparql-benchmark-runner |
| **tail**: time from the last document fetch to the last result. A large tail in T means T is CPU-bound; this is the measured regime indicator | last fetch timestamp from proxy or engine log **(verify which exists)** |
| engine CPU time ÷ wall time | proxy `resources` (or `/proc` sampling of the engine process) |
| correctness: recall, precision, result hash vs reference | existing `ExperimentFedShopDiscovery` |
| requests and bytes, per storage and per kind (document / root / sparql) | counting proxy `/stats` (also confirms both arms fetch the same documents) |
| CPU and memory: Solid server, endpoints | proxy `resources` |
| partition chosen, number of composite blocks, solutions per delegated unit | engine log (to add; replaces the leftover `console.log`) |

Replications: 5 per (instance × arm), with a fixed timeout. **(decide)** the cold/warm policy. CPU-bound runs are sensitive to machine noise, so pin the engine to fixed cores and run nothing else alongside.

## Size of the workload (estimate)

Every axis point costs 8 runs (4 types × T and C), × 5 replications.

| Axis | Points | Runs |
|---|---|---|
| S1 size | 5 shops × 2 series (fixed N, per-resource reference) | 400 |
| S2 gain | 6 gains | 240 |
| S2 type-specific knobs | star 4 sizes, leftover 3 levels, chain 2 one-sided points; 2 runs each | 40 + 30 + 20 = 90 |
| S3 topology | about 7 layouts × 2 latencies | 560 |
| S4 partitioner | 16 grid cells × (T, C-SF, C-CF) | 240 |
| M1 storages | 5 values of k | 200 |
| M2 coverage | 5 values of f | 200 |
| M3 | 3 shops × (T, C) | 30 |
| M4 | 3 depths × 2 instances × (T, C) | 60 |

About 2,050 runs, all on small data: two to three days of machine time. Servers are loaded once per dataset instance, and all query instances on it run against them. S3 is the largest share. If time is short, run the breadth points at 0 ms only, and keep both latencies for the depth points, where latency is what makes depth visible.

## What has to be built

See [PLAN.md](PLAN.md). Everything except the engine work lives in this directory as one Python project (`micro`), with no changes to FedShop-discovery. The engine work: the C configs (stars + chain proposers, with SF and with CF), the chain-first partition actor (subclass of `actor-derived-resource-partition-star-first`, override `tier()`), the `constantCount` fix, and a partition log.

## Open decisions

1. **Latency:** the proxy adds the same fixed delay to every request (documents and SPARQL), for every arm. S3 sweeps 0/80 ms. Which fixed value do the other axes use? 0 ms puts them in the CPU-bound regime, where C's saving is visible. 80 ms is closer to a real network.
2. **Cache policy:** the plan uses warm servers (loaded once per dataset instance) and one engine per (dataset, arm) with cache invalidation before each query. Is that acceptable, or should the servers be restarted per run?
3. **Discovery inside the micro-benchmarks:** M1–M3 seed every storage in scope. Should one axis let the engine discover the storages instead (seed one, find the rest by links), or is that left to the full benchmark?
4. **Cost-based partitioner:** add it as a third partitioner in S4 now, or only after the cost function exists?
5. **N for S1:** the fixed number of documents per sample (at most 500, the smallest sample). Alternatively, run two values of N (for example 16 and 256) instead of the per-resource reference series, to see whether the document count changes the size trend. That costs the same, but overlaps more with S3.
6. **Chain of 4 in Phase 1:** it needs a brand next to the shop. The proposal is to hold the brand fixed and apply the axis to the shop only. The alternative is to apply each axis to both storages at once.
