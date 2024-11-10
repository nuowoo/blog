
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


If responsiveness and operational agility sound exciting to you, we invite you to try out Materialize for yourself.
Our [guided tutorial](https://www.materialize.com/docs/get-started/quickstart/) builds up the auction data sources described above, and includes demonstrations of consistency.
If you'd like to try out Materialize on larger volumes of your own data, reach out about doing a [Proof of Concept](https://materialize.com/trial/) with us!

<!-- ##{"timestamp":1696914000}## -->
