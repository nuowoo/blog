
Responsiveness is one of three components of [Materialize's Trust pillar of product value](https://materialize.com/blog/operational-attributes/#trust), the other two being [freshness](https://materialize.com/blog/freshness/) and [consistency](https://materialize.com/blog/operational-consistency/).
While being fresh and consistent is fundamental, operational work suffers if each intervention is a 15 minute deployment away.
We all want to live in world where our operational logic is fully baked, but the reality is that things change and interactivity matters.
Moreover, operational work is often inherently interactive: responding to user or operator queries that are not known ahead of time.
For these reasons, among others, systems must be responsive to be a trustworthy part of your operational layer.

Different architectures have different visions for how work gets done, which leads to different responsiveness characteristics.
The conventional cloud data warehouse pulls stale data from cloud storage and re-evaluates your query, each time from scratch and at some cost.
Dataflow engines generally re-flow the streams that define their inputs, which happens at high throughput but still takes time to cover the volume of data.
Caches and microservices generally nail responsiveness, though without much to say about consistency or freshness.
The caveats make none of these alternatives especially satisfying.

Responsiveness is about more than just promptly providing a response: the response needs to be valuable and actionable.
Systems can trivially respond with inconsistent, stale, or unhelpful results ("nothing yet, boss"), but we understand that this doesn't yet provide value.
They can promptly respond to interventions with confirmation of initiation ("just starting, boss"), but this doesn't mean any work will soon be done.
Responsiveness provides value when the response has meaning, which we believe is captured by consistency and freshness (which is why we covered them first!).
A responsive system must promptly provide a *meaningful* response; otherwise it is just entertainment.

In this post we'll dive into how Materialize makes commands responsive, from the structure it exploits in both data and queries, through the technical underpinnings, up to an example of responsive, fresh, and consistent results for non-trivial operational work involving multi-way joins.

## Responsiveness in Materialize

In Materialize, responsiveness is about minimizing the time between an issued command and Materialize's consistent, fresh responses (to the operator, or to downstream consumers).

Achieving responsiveness is about much more than just programming hard to make computers go fast. 
It is about preparing and organizing information ahead of time so that when commands arrive we have the answers (nearly) at hand.
When `SELECT` commands arrive, from easy `LIMIT 1`s to hard multi-way `JOIN`s, we want to minimize the time required before Materialize can provide the result.
When users create indexes, materialized views, and sinks, we want to minimize the time before those assets are operational.
In each case, we want to identify and exploit structure in the data and the commands to make subsequent work fast.

We also try to program rly hard, but the gains really come from the preparation instead.

### Data Structure: Change Data Capture and Snapshot Roll-ups

Materialize uses [change data capture](https://en.wikipedia.org/wiki/Change_data_capture) (CDC) as a way to represent continually changing data.
Importantly, while CDC presents itself as a stream of events, it has the special structure that they always "roll up" to a snapshot data set.
One can interpret and operate on CDC data as if a snapshot followed by changes, without needing to retain and review the historical detail of a raw stream.
This is an example of "data structure" that will allow us to do something more clever than continually re-evaluating over all data we've ever seen.

The CDC structure gives us a guiding principle for how to organize information: organize the snapshot and maintain it as it changes.
Materialize durably records CDC updates, but continually compacts them to maintain a concise snapshot of input data.
Materialize builds indexes over both input data and data derived through views, and maintains them as the data change.
Materialize responds with snapshot data, but follows it with CDC updates that call out the changed data explicitly.
Any tricks we can use for snapshots of data are in scope for Materialize, as long as we can extend them to *maintained* results.

The superpower of CDC and roll-ups is that we know that queries have a correct and concise answer, and we can prepare our data to answer them ahead of time.

### Query Structure: Data Parallelism

A great deal of the value in SQL's `SELECT` command is how it draws out of complex questions the *independence* of the rows of the data.
A `WHERE` or `HAVING` clause applies row-by-row; the result on one row does not affect the result on another row.
A `JOIN` clause finds rows that match on key columns, whose results are independent of rows that do not match on these columns.
A `GROUP BY` clause produces aggregates for each key, each output independent of rows with other keys.
It is this query structure, the identified *independence*, that enables much of modern data processing optimization.

Materialize's storage plane records CDC streams and maintains them as snapshots and changelogs, serving them up to other parts of the system.
When it does serve them up, it does so in response to requests, and these requests usually have valuable context that can improve its performance.
If a user requires only recent data, e.g. a `WHERE row.time > mz_now()`, the storage layer can return a subset of records that might pass this test.
If a user requires only a subset of columns, e.g. a projection, the storage layer could (but does not yet) return only those columns
If a user needs only limited results, e.g. a `LIMIT 1`, the storage layer can stop as soon as the needed number is met.
These are each techniques from cloud data warehouses on static data, but generalize to changing data for the same SQL idioms. 

Materialize's compute plane builds and maintains indexes over both input data and data derived from SQL views.
These indexes are on key columns, or key expressions, and ensure that one can look up all records that match a certain key.
They allow queries with `WHERE key = literal` or `WHERE key IN (lit1, lit2, lit3)` to dive directly to the relevant results, in milliseconds, rather than scan anything.
They also enable `JOIN`s that equate the key columns to do so immediately, rather than needing to rescan and reorganize the input.
These indexes are continually maintained, providing interactive access without sacrificing freshness or consistency as might an independent cache.

Finally, Materialize's serving plane takes advantage of independence among the SQL commands themselves. 
While Materialize must put the commands in *some* order, Materialize can see which commands can execute concurrently and does so.
Materialize tracks the available timestamps for each input and derived view (their "freshness"), and uses this information in determining the best order.
When consistency or freshness is not as important to you as as responsiveness, Materialize provides tools (e.g. `SERIALIZABLE` isolation) to help navigate the trade-offs.

Materialize takes advantage of existing SQL idioms you already know and expect, to provide a responsive experience.

## A Worked Example: Auctions

Let's take a quick look at a workload that highlights Materialize's *responsiveness* in the face of a non-trivial workload.
We'll mostly deal with interactive queries, but the implications apply just as well to deployed dataflows into indexes, materialized views, and sinks.

Our [guided tutorial](https://materialize.com/docs/get-started/quickstart/) is based around an auction load generator, which contains among other things continually evolving auctions and bids.
One common query you might want to support is "for each auction I (a user) have bid in, how many other users have outbid me?"
This both calls out auctions you are currently winning, and gives a sense for the level of competition in other auctions.
However, it is not immediately obvious how best to support this sort of query interactively.

Let's start by writing some views defining the logic we'll want.
As it turns out, the views themselves will not need to change much as we explore different ways to dial in their responsiveness.

```sql
-- All bids for auctions that have not closed.
CREATE VIEW active_bids AS
SELECT bids.*
FROM bids, auctions 
WHERE bids.auction_id = auctions.id
  AND auctions.end_time > mz_now() 
  AND bids.bid_time + INTERVAL '10 seconds' > mz_now();
```

```sql
-- Number of times each buyer is outbid in each auction.
CREATE VIEW out_bids AS
SELECT a1.buyer, a1.auction_id, COUNT(*)
FROM active_bids AS a1, 
     active_bids AS a2
WHERE a1.auction_id = a2.auction_id
  AND a1.amount < a2.amount
  AND a1.buyer != a2.buyer
GROUP BY a1.buyer, a1.auction_id;
```

A first approach could be to perform the work from scratch each time a user asks.
This is roughly what would happen if you tried to serve the application out of your data warehouse.
While it works, doing so is all sorts of scary, and isn't even all that responsive.
```sql
-- From-scratch evaluation of `out_bids` with a predicate applied.
SELECT * FROM out_bids WHERE buyer = <buyer_id>;
```
Materialize can push down the `mz_now()` temporal filters to the storage layer, reducing the amount of data that must be processed.
However, we still need to collect and organize the data, which is unavoidable work to produce the correct count.
On the plus side, we have no ongoing cost other than the storage layer maintaining `bids` and `auctions`.
On Materialize just now, this took between 100 and 300 milliseconds to re-run (with `SERIALIZABLE` isolation).

A second approach could be to materialize the whole of `out_bids`, maintaining each count for each user and auction.
This is roughly what you'd get if you set up a stream processor, and produced the results to some serving or caching layer.
While it also works, you'll end up spending a fair bit maintaining data you may not need, and you won't even get consistency by the end.
```sql
-- Index `out_bids` by the `buyer` column, for fast look-up.
CREATE INDEX out_bids_idx ON out_bids (buyer);
-- Random access to the index by the buyer id.
SELECT * FROM out_bids WHERE buyer = <buyer_id>;
```
This approach is very responsive, reading the result directly out of an index. 
However, there is a maintenance cost: any new bid to an auction means updates for all counts that it exceeds.
On Materialize just now, this took consistently 20 milliseconds to re-run (with `SERIALIZABLE` isolation).
Were I to increase the input load, I would need to quickly increase the instance size in order to keep up.

A third approach is to index the intermediate `active_bids`, on both the `buyer` and `auction_id` columns.
This is neither what you'd get in a cloud data warehouse or in a stream processor; it seems unique to Materialize.
```sql
-- Index `active_bids` by the `buyer` and `auction_id` columns.
CREATE INDEX active_bids_idx1 ON active_bids (buyer);
CREATE INDEX active_bids_idx2 ON active_bids (auction_id);
-- Allow Materialize to cleverly use the indexes in live joins.
SELECT * FROM out_bids WHERE buyer = <buyer_id>;
```
In this case Materialize will plan a `JOIN` query that uses the indexes and returns in interactive timescales.
Informally, the query plan will start from `<buyer_id>` and pull all relevant auction identifiers from the first index, then use the second index to translate auction identifiers into the bids on those auctions, then count those records that satisfy the predicate on bid values.
We only touch the records we are interested in, and maintaining indexes on `active_bids` takes much less effort than maintaining all of `out_bids`.
The counts are instead produced at query time, showing a neat hybridization of pre-computation and query time computation.
On Materialize just now, this took consistently 30 milliseconds to re-run (with `SERIALIZABLE` isolation).
Were I to increase the input load, I would also need to increase the instance size, but not nearly as much.

---

If you'd like to explore any of these query plans in Materialize, just put an `EXPLAIN` in front of the `SELECT` command.
The plans of the second and third approaches are very approachable, whereas the first (re-execution) is a whole screenful.
But actually, taking a moment with each of them is probably very helpful, 

---

These three approaches to addressing a task show off several of the ways Materialize provides a responsive experience.
The storage layer can minimize data retrieved, the compute layer can maintain results in indexes and use them to fuel interactive joins, the adapter layer can choose between them based on available assets.
These mechanism take advantage of structure in the data and structure in the queries, keeping the right information up to date with input changes.
Importantly, each of them provide identical output, as responsiveness does not come at the expense of consistency or freshness.

## Responsiveness and Operational Agility

Responsiveness is about the ability to do new things quickly.
To answer new questions, or set up new ongoing workflows, quickly.
To interactively probe and live-diagnose problems, with SQL queries not just key lookups, quickly.
Responsiveness speaks to the *agility* of your operational layer.

Operational tools that cannot respond quickly with actionable output are inherently clumsy and problematic.
You, your team, or your users will work around them, giving up on hard-won consistency, freshness, or both.
By the same token, being *meaningfully responsive* is about more than providing a prompt placeholder response.
Operational systems need to be ready with the information you need, and be poised to correcctly implement the operational work you require.

If responsiveness and operational agility sound exciting to you, we invite you to try out Materialize for yourself.
Our [guided tutorial](https://www.materialize.com/docs/get-started/quickstart/) builds up the auction data sources described above, and includes demonstrations of consistency.
If you'd like to try out Materialize on larger volumes of your own data, reach out about doing a [Proof of Concept](https://materialize.com/trial/) with us!

<!-- ##{"timestamp":1696914000}## -->