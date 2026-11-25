## Microbenchmarks to test composite resource delegation failure modes and advantages

We will test 4 axes:
- How the varying the size of queried sources and thus the delegated join changes performance characteristics
- Selectivity of the delegated join and how it changes performance
- Spread of data queried (depth / breadth) how many links to traverse untill all query relevant data is found
- How delegation is done: Either star-first or chain-first, where we can have a star - chain query where either
the chain is very selective (of size 3) or one where the star is selective and chain of size 2 attached is not


We want to showcase where delegating composite resources helps and where it doesn't.

For you claude: Please investigate the querying-derived-resources directory. Also check out the /papers or /project/papers
with the correct paper (federated..) to see the draft so far.

Few open questions:
1. Should we use SolidBench.js or the Fedshop-discovery
2. How to create data so it is a quantifiable micro benchmark
3. Should we test speedup of pattern-only vs composite or use composite-only vs default (with adaptive join). We want to isolate
   the impact of delegating sub-queries, pattern queries seems not as interesting.