# learn-elasticsearch

Twitter thread: https://twitter.com/madawei2699/status/1541215807228637186

## ES Version

commit hash: b6721943443ce4f21781f051ba6a856f8a1cdd68
Twitter
tag: v8.3.1

repo: https://github.com/madawei2699/elasticsearch

## Project Structure

```bash
tree server/src/main/java/org/elasticsearch/ -L 1 -d
for d in server/src/main/java/org/elasticsearch/*/ ; do (cd "$d" && echo "$d" && cloc --vcs git --md); done
```

```text
server/src/main/java/org/elasticsearch/
├── action 76240L # Actions that Elasticsearch can take either on the data stored on disk or on other nodes.
│   ├── admin
│   ├── bulk
│   ├── datastreams
│   ├── delete
│   ├── explain
│   ├── fieldcaps
│   ├── get
│   ├── index
│   ├── ingest
│   ├── main
│   ├── resync
│   ├── search
│   ├── support
│   ├── termvectors
│   └── update
├── bootstrap 3028L
├── client 2383L
├── cluster 51592L # High availability implemented by cluster.
│   ├── ack
│   ├── action
│   ├── block
│   ├── coordination
│   ├── desirednodes
│   ├── health
│   ├── metadata
│   ├── node
│   ├── routing
│   └── service
├── common 46580L # Common utils like IoC by inject, logging, filesystem, lucene, metrics and network.
│   ├── blobstore
│   ├── breaker
│   ├── bytes
│   ├── cache
│   ├── cli
│   ├── collect
│   ├── component
│   ├── compress
│   ├── document
│   ├── filesystem
│   ├── geo
│   ├── hash
│   ├── inject
│   ├── io
│   ├── logging
│   ├── lucene
│   ├── metrics
│   ├── network
│   ├── path
│   ├── recycler
│   ├── regex
│   ├── settings
│   ├── text
│   ├── time
│   ├── transport
│   ├── unit
│   ├── util
│   └── xcontent
├── discovery 1161L
├── env 1861L
├── gateway 4499L
├── health 756L
├── http 1729L
├── immutablestate 28L
├── index 87209L # ES index implementation.
│   ├── analysis
│   ├── bulk
│   ├── cache
│   ├── codec
│   ├── engine
│   ├── fielddata
│   ├── fieldvisitor
│   ├── flush
│   ├── get
│   ├── mapper
│   ├── merge
│   ├── query
│   ├── recovery
│   ├── refresh
│   ├── reindex
│   ├── search
│   ├── seqno
│   ├── shard
│   ├── similarity
│   ├── snapshots
│   ├── stats
│   ├── store
│   ├── termvectors
│   ├── translog
│   └── warmer
├── indices 14632L # Global Index settings for ES index.
│   ├── analysis
│   ├── breaker
│   ├── cluster
│   ├── fielddata
│   ├── recovery
│   └── store
├── ingest 4614L
├── lucene 2437L
├── monitor 4288L
├── node 1944L
├── persistent 2393L
├── plugins 2334L
├── readiness 171L
├── repositories 7148L
├── rest 12674L # Exposes Elasticsearch functionality over RESTful HTTP.
│   └── action
├── script 7774L
├── search 91813L # ES search feature returning hits that match the query defined in the request.
│   ├── aggregations
│   ├── builder
│   ├── collapse
│   ├── dfs
│   ├── fetch
│   ├── internal
│   ├── lookup
│   ├── profile
│   ├── query
│   ├── rescore
│   ├── runtime
│   ├── searchafter
│   ├── slice
│   ├── sort
│   ├── suggest
│   └── vectors
├── shutdown 69L
├── snapshots 7153L
├── tasks 1822L
├── threadpool 1314L
├── timeseries 492L
├── transport 9538L
├── upgrades 1370L
├── usage 41L
└── watcher 459L
```

## Design

### Cluster

File: docs/reference/high-availability/cluster-design.asciidoc

## Module

![](https://img.bmpi.dev/7f8d8e56-a68e-5a52-bce7-5caddb48ed49.png)

## Note

### How to write test (./TESTING.asciidoc)

#### Good practices

##### What kind of tests should I write?

Unit tests are the preferred way to test some functionality: most of the time
they are simpler to understand, more likely to reproduce, and unlikely to be
affected by changes that are unrelated to the piece of functionality that is
being tested.

The reason why `ESSingleNodeTestCase` exists is that all our components used to
be very hard to set up in isolation, which had led us to having a number of
integration tests but close to no unit tests. `ESSingleNodeTestCase` is a
workaround for this issue which provides an easy way to spin up a node and get
access to components that are hard to instantiate like `IndicesService`.
Whenever practical, you should prefer unit tests.

Many tests extend `ESIntegTestCase`, mostly because this is how most tests used
to work in the early days of Elasticsearch. However the complexity of these
tests tends to make them hard to debug. Whenever the functionality that is
being tested isn't intimately dependent on how Elasticsearch behaves as a
cluster, it is recommended to write unit tests or REST tests instead.

In short, most new functionality should come with unit tests, and optionally
REST tests to test integration.

##### Refactor code to make it easier to test

Unfortunately, a large part of our code base is still hard to unit test.
Sometimes because some classes have lots of dependencies that make them hard to
instantiate. Sometimes because API contracts make tests hard to write. Code
refactors that make functionality easier to unit test are encouraged. If this
sounds very abstract to you, you can have a look at
https://github.com/elastic/elasticsearch/pull/16610[this pull request] for
instance, which is a good example. It refactors `IndicesRequestCache` in such
a way that:
 - it no longer depends on objects that are hard to instantiate such as
   `IndexShard` or `SearchContext`,
 - time-based eviction is applied on top of the cache rather than internally,
   which makes it easier to assert on what the cache is expected to contain at
   a given time.

#### Bad practices

##### Use randomized-testing for coverage

In general, randomization should be used for parameters that are not expected
to affect the behavior of the functionality that is being tested. For instance
the number of shards should not impact `date_histogram` aggregations, and the
choice of the `store` type (`niofs` vs `mmapfs`) does not affect the results of
a query. Such randomization helps improve confidence that we are not relying on
implementation details of one component or specifics of some setup.

However it should not be used for coverage. For instance if you are testing a
piece of functionality that enters different code paths depending on whether
the index has 1 shards or 2+ shards, then we shouldn't just test against an
index with a random number of shards: there should be one test for the 1-shard
case, and another test for the 2+ shards case.

##### Abuse randomization in multi-threaded tests

Multi-threaded tests are often not reproducible due to the fact that there is
no guarantee on the order in which operations occur across threads. Adding
randomization to the mix usually makes things worse and should be done with
care.
