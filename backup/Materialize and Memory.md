Materialize keeps your SQL views up to date as the underlying data change.
The value Materialize provides comes from how promptly it reflects new data, but its *cost* comes from the computer resources needed to achieve this.
While we often talk about the value Materialize provides, and work continually to improve it, we are also hard at work continually reducing the cost.
This work has had marked impact recently, and it felt like a great time to tell you about it, and the reductions in cost. 

Materialize maintains your source and derived data (e.g. any materialized view), durably in economical cloud storage.
However, to promptly maintain views and serve results we want to use much more immediately accessible storage.
This storage, memory or as we'll see soon local disk, acts as a cache that must be fast, but needn't be durable.
And of course, we would all like it to be as economical as possible.

We've been dialing down the amount of "overhead" associated with each intermediate maintained record in Materialize.
We started some months ago at roughly 96 bytes of overhead (we will explain why), and we are now closing in on between 0 and 16 bytes of overhead, depending.
This first wave of results have already seen many users memory requirements reduced by nearly 2x.
Moreover, we've laid the groundwork for further improvements, through techniques like spill-to-disk, columnar layout, and compression.
This further work comes at the cost of CPU cycles, but for the moment CPU cycles are abundant (and elastic) in a way that bytes of memory are not.

In this post we'll map out where we started, detail the relatively simple steps we've taken to effectively reduce the overhead, and sketch the future we've opened up with some help from Rust.

### The Fundemantals of Remembered Things

Materialize models all data as relational rows, each of which has some number of columns, each of which contains one of a few different types of data.
Over time the rows come and go, each changing their multiplicity through what we call "updates": triples `(data, time, diff)`.
Each update indicates a row `data` that at some moment `time` experiences a change `diff` in its multiplicity.
These changes are often `+1` (insertion) or `-1` (deletion), or a mix of two or more (updates).

Materialize maintains *indexed state* by viewing each `data` as a pair `(key, val)`, where `key` are some signified columns and `val` the remaining columns.
When you create an index on a collection of data, you specify columns by which you hope to access the data; those columns define `key` and `val` for each `data`.
We regularly want to fetch the history of some `key`: the associated `val`s and the `(time, diff)` changes they have undergone.

The abstract data type we use maps from `key` to `val` to a list of `(time, diff)` pairs.
In Rust you might use the `HashMap` type to support this abstraction:
```rust
/// Map from key, to value, to a list of times and differences.
type Indexed<K, V, T, D> = HashMap<K, HashMap<V, Vec<(T, D)>>>;
```

For various reasons we won't actually want to use `HashMap` itself, and instead prefer other data structures that provide different performance characteristics.
For example, we are interested in minimizing the number and size of allocations, and optimizing for both random and sequential read and write throughput.

### A First Evolution, circa many years ago

Differential dataflow's fundamental data structures are thusfar based on sorted lists.
All of differential dataflow's historical performance, which has been pretty solid, has been based on [the perhaps surprising efficiency of sorted memory access](https://github.com/frankmcsherry/blog/blob/master/posts/2015-08-15.md).
You may have thought we were going to impress you with exotic improvements on Rust's `HashMap` implementation, but we are going to stay with sorted lists.

In the context of space efficiency, sorted lists have a compelling property that Rust's `HashMap` does not have: you can append multiple sorted lists into one larger list, and only need to record the boundaries between them.
This reduces the per-key, and per-value overhead to something as small as an integer.
You do miss out on some random access performance, but you also gain on sequential access performance.
For the moment though, we're interested in space efficiency.

To store the map from `key` to `val` to list of `(time, diff)` updates, differential dataflow uses roughly three vectors:
```rust
/// Simplification, for clarity.
struct Indexed<K, V, T, D> {
    /// key, and the start of its sorted run in `self.vals`.
    keys: Vec<(K, usize)>,
    /// val, and the start of its sorted run in `self.upds`.
    vals: Vec<(V, usize)>,
    /// lots and lots of updates.
    upds: Vec<(T, D)>,
}
```

Each key is present once, in sorted order. 
The `usize` offset for each key tells you where to start in the `vals` vector, and you continue until the offset of the next key or the end of the vector.
The `usize` offset for each value tells you where to start in the `upds` vector, and you continue until the offset of the next value or the end of the vector.

The data structure supports high throughput sequential reads and writes, random access reads through binary search on keys, and random access writes through a [log-structure merge-tree](https://en.wikipedia.org/wiki/Log-structured_merge-tree) idiom (although perhaps "merge-list" is more appropriate).

The overhead is one `usize` for each key, and another `usize` for each distinct `(key, val)` pair.
You have three allocations, rather than a number proportional to the number of keys or key-value pairs.
The overhead seems pretty small, until we perform a more thorough accounting.

### A More Thorough Accounting

Although Materialize maintains only two `usize` (each 8 bytes) beyond the `K`, `V`, `T`, and `D` information it needs for updates, there is more overhead behind the scenes.

In Materialize both `K` and `V` are `Row` types, which are variable-length byte sequences encoding column data.
In Rust a `Vec<u8>` provides a vector of bytes, and takes 24 bytes in addition to the binary data itself.
In fact we have used a 32 byte version that allows for some amount of in-line allocation, but meant that the minimum sizes of `K` plus `V` is 64 bytes, potentially in addition to the binary row data itself.

Both `T` and `D` are each 8 byte integers, because there are many possible times, and many possible copies of the same record.
Adding these together, we get an overhead accounting of
```
key offset:  8 bytes
val offset:  8 bytes
key row:    32 bytes
val row:    32 bytes
time:        8 bytes
diff:        8 bytes
--------------------
overhead    96 bytes
```
The minimum buy-in for each update is 96 bytes.
These 96 bytes may cover no actual row data, and can just be pure overhead.

### Optimization

Fortunately, the more thorough accounting leads us to a clearer understanding of opportunities.
Every byte that is not actual binary payload is in play as optimization potential.
Let's discuss a few of the immediate opportunities.

#### Optimizing `(Time, Diff)` for Snapshots

Materialize first computes and then maintains SQL views over your data.
A substantial volume of updates describe the data as it initially exists, an initial "snapshot", before changes start to happen.
As changes happen we continually roll them up into the snapshot, so even a live system has a great deal of "snapshot" updates.

The snapshot updates commonly have `(time, diff)` equal to `(now, 1)`.
That is, each `(key, val)` pair in the snapshot exists "right now", and just once. 
This provides an opportunity for bespoke compression: if a `(time, diff)` pair repeats we are able to avoid writing it down repeatedly.
In fact, we can sneak this in at zero overhead by taking advantage of a quirk in our `usize` offsets: they *should* always strictly increase to indicate ranges of updates, because empty ranges should not be recorded, but we can use a repetition (a non-increase) to indicate that the preceding updates should be reused as well.

This typically saves 16 bytes per update for the snapshot, and brings us down to 80 bytes of overhead.
```
key offset:  8 bytes
val offset:  8 bytes
key row:    32 bytes
val row:    32 bytes
--------------------
overhead:   80 bytes
```

#### Optimizing `Row` representation

Although we have a 32 byte `Row` we could get by with much less.
Just like we appended lists and stored offsets to track the bounds, we could append lists of bytes into one large `Vec<u8>` and maintain only the `usize` offsets that tell us where each sequence starts and stops.

This takes us from 32 bytes with the option for in-line allocation, to 8 bytes without that option.
This applies twice, once to each of `key` and `val`.
Moreover, we avoid an *allocation* for each `key` and `val`, which evades some further unaccounted overhead in and around memory management.
We now have four offsets, two for each of `key` and `val`, which will be important next.
```
key offset:     8 bytes
val offset:     8 bytes
key row offset: 8 bytes
val row offset: 8 bytes
-----------------------
overhead:      32 bytes
```

#### Optimizing `usize` Offsets

Our `usize` offsets take 8 bytes, but rarely get large enough to need more than 4 bytes.
This is because we end up "chunking" our data to manageable sizes, and those chunk sizes rarely exceed 4GB, for which a `u32` would be sufficient.
Rather than use a `Vec<usize>` to store these offsets, we can first use a `Vec<u32>`, and should we exceed 4 billion-ish we can cut-over new elements to a `Vec<u64>`.

This shaves the four `usize` offsets down from 32 bytes to 16 bytes, in most cases.
```
key offset:     4 bytes
val offset:     4 bytes
key row offset: 4 bytes
val row offset: 4 bytes
-----------------------
overhead:      16 bytes
```

Going even further, these offsets often have very simple structure.
When there is exactly one value for each key (e.g. as in a primary key relationship) the key offsets are exactly the sequence 0, 1, 2, ...
When considering the snapshot, the value offsets are all zero (recall that repetitions indicate repeated `(time, diff)` pairs).
When the binary slices have the same length (e.g. for fixed-width columns) the corresponding row offsets are the integer multiples of this length.
Each of these cases can be encoded by a single "stride" and a length, using two integers in total rather than any per element.

These further optimizations can bring the 16 bytes of overhead down, all the way to zero when stars align.

### Further Optimization and Future Work

With nearly zero overhead, you may be surprised to learn that we are not yet done.
But in fact, there is still opportunity to further reduce cost!

#### Paging Binary Data to Disk

Materialize, by way of differential dataflow, performs its random accesses in a way that [resembles sequential scans](https://github.com/frankmcsherry/blog/blob/master/posts/2015-08-15.md) (essentially: batching and sorting look-ups before they happen).
This means that putting binary payload data on secondary storage like disk is not nearly as problematic as it would be were we to access it randomly, as in a hash map.
Disk is obviously substantially cheaper than memory, and it provides the opportunity to trade away peak responsiveness for some cost reduction.

In fact we've recently done this, backing in-memory allocations with disk allocations that Linux can spill to if it feels memory pressure.
Expect a post in the near future talking about the design and implementation of this paging layer.

Our experience so far is that initial snapshot computation experiences almost no degradation (the batch disk accesses are sequential scans), and once up and running update volumes are often low enough volume that local SSD accesses do not prevent timely results.
The local disks are ephemeral caches, and don't come at the same cost as more durable options like cloud block storage.


#### Columnar Compression

Rust has some [handy mechanisms](https://blog.rust-lang.org/2022/10/28/gats-stabilization.html) that allow us to interpose code between the binary data for each row and the SQL logic that will respond to the row data.
Our logic expects each row only as a sequence of `Datum` column values, and doesn't require an actual contiguous `[u8]` binary slab.
This allows us some flexibility in how we record each row, potentially as a `[u8]` sequence, but also potentially re-ordered, transformed, or compressed.

Cloud Data Warehouses often record their data in [columns](https://en.wikipedia.org/wiki/Column-oriented_DBMS), rather than rows, to improve their space efficiency while sacrificing performance for random access.
We don't want to sacrifice too much random access, but we can employ several of the same compression tricks.
In particular, we are able to sneak in various techniques, from [entropy coding](https://en.wikipedia.org/wiki/Entropy_coding) like Huffman and ANS, to [dictionary coding](https://en.wikipedia.org/wiki/Dictionary_coder) which often works well on denormalized relational data.
Moreover, we can apply these techniques column-by-column, as columns often exhibit more commonality than do rows.

The benefits of compression depend greatly on the nature of the data itself, and come at a non-trivial CPU overhead, but would unlock significant space savings and further opportunities.

#### Query Optimization

A final, evergreen opportunity is to continue to reduce the amount of information we need to maintain, independent of how it is represented.
Materialize's optimizer pays specific attention to the amount of information maintained, which distinguishes it from most query optimizers that aim primarily to reduce query time.
How and where we maintain state is very much under our control, and something we still have many opportunities to improve.

### Wrapping Up

Materialize provides value through the information it maintains, at the expense of maintaining intermediate state in scarce and costly storage (at least, relative to cloud blob storage).
The cost of the storage can't be overlooked, and driving it down makes the provided value net out positive for even more use cases.
In the limit, we'll get you to expect everything to be always up to date, because why shouldn't it be? 

The cost optimizations described above are all live in Materialize now.
It would be interesting to invite you to see the before and after, but actually we'd love to introduce you to the after, and let you see the next tranche of improvements as they happen.
To try out Materialize, sign up at https://materialize.com/register/!

<!-- ##{"timestamp":1703048400}## -->