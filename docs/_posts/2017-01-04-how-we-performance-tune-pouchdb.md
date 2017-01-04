---
layout: post

title: How we performance-tune PouchDB
author: Nolan Lawson

---

In PouchDB 6.1.1, there were substantial improvements to the performance of secondary index creation, as well as general improvements to the IndexedDB and in-memory adapters. In this post I'll talk about how I analyzed PouchDB and implemented these performance boosts.

### Measure twice, cut once

There's an old English idiom that says ["measure twice, cut once"](https://en.wiktionary.org/wiki/measure_twice_and_cut_once#English). This saying was originally a reference to carpentry, but I find it to be excellent advice for performance-tuning as well.  

When trying to improve the performance of any piece of code, proper measurements are your most valuable asset. If you can't accurately measure what "fast" means, then you can't tell if you're getting any faster.

PouchDB maintains [a simple set of benchmarks](https://github.com/pouchdb/pouchdb/tree/master/tests/performance) using [tape](https://github.com/substack/tape) as well as measurements based on `performance.mark()`, `performance.measure()`, and `performance.now()`, aka the [User Timing API](https://developer.mozilla.org/en-US/docs/Web/API/User_Timing_API). These APIs provide accurate timings as well as nice visualizations in the Chrome Dev Tools:

Previously we had been using `Date.now()`, but this API is no longer recommended for performance testing, because it's not as high-resolution (i.e. accurate) as `performance`. During the course of modernizing our measurements, I also built a standalone library called [marky](https://github.com/nolanlawson/marky), which makes it easier to use high-resolution APIs like `performance.mark()` and `measure()` while also falling back to `Date.now()` in older browsers.

Once you've got your measurements in place, next you need to make sure that you reduce variability in the tests. When I run performance tests, I always ensure:

1. **The Dev Tools have never been opened.** These add overhead to browsers. So for a fair assessment, it's best to completely close the Dev Tools, close the browser, and reopen it.
2. **The laptop is connected to a power outlet.** Browsers may change their behavior in battery mode. For instance, Edge throttles `setTimeout()` to 16 milliseconds rather than the standard 4.
3. **The tests are repeatable.** Performance tests should have enough iterations that you can see a clear, measurable difference between runs; you don't want to interpret statistical noise as a performance boost or regression. If you want to get really fancy, you can use something like [BenchmarkJS](https://benchmarkjs.com/) which does enough iterations to ensure statistical significance.

Another important consideration when running performance tests is to **test in multiple browsers**. I find many web developers have gotten into the habit of only ever testing in one browser (usually Chrome). However, browsers tend to differ enormously in their performance aspects. So if you only test in one browser, you can miss performance opportunities (e.g. because the one browser happens to have already optimized a particular scenario) or over-engineer based on idiosyncratic browser quirks (e.g. focusing only on [V8 deoptimizers](https://github.com/petkaantonov/bluebird/wiki/Optimization-killers) while ignoring other engines).

For PouchDB, it's especially crucial that we test in multiple browsers, because IndexedDB implementations vary wildly across the board. Chrome's version is implemented using [LevelDB](http://leveldb.org/), Firefox's and Safari's use [SQLite](https://sqlite.org/) (different implementations, though), and IE and Edge use [ESE](https://en.wikipedia.org/wiki/Extensible_Storage_Engine). In practice this leads to large and measurable differences between the various engines.

In the case of PouchDB, we also have to be careful to consider performance implications for non-IndexedDB databases, since we also support WebSQL, LevelDB (via [leveldown](https://github.com/level/leveldown)), in-memory (via [memdown](https://github.com/level/memdown)), and even exotic engines like [node-websql](https://github.com/nolanlawson/node-websql). However, IndexedDB is our primary target, since it's the most commonly-used adapter.

For this round of performance improvements, I chose to target a few different scenarios:

- Secondary index creation and querying, using IndexedDB
- Secondary index creation and querying, using in-memory (aka memdown)

I focused on secondary index creation and querying because, due to the way PouchDB is architected, this use case tends to hit lots of core PouchDB APIs, including `bulkDocs()`, `changes()`, and `allDocs()`. Furthermore, it's one of our slower scenarios, and one users frequently gripe about.

The reason I chose to focus on IndexedDB and in-memory was because 1) IndexedDB is our most important adapter, but 2) in-memory is extremely useful for running quick unit tests (e.g. via `pouchdb-server --in-memory`) and also helps isolate non-engine-specific issues, since it's pure JavaScript.