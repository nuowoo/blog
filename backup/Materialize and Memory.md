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

<!-- ##{"timestamp":1703048400}## -->
