
Freshness is one of three components of [Materialize's Trust pillar of product value](https://materialize.com/blog/operational-attributes/#trust), the other two being responsiveness and [consistency](https://materialize.com/blog/operational-consistency/).
Operational work is fundamentally about promptly reacting to and reflecting events in the real world. 
And the real world, famously, waits for no one.
Every moment your operational layer isn't up to date represents missed opportunity as the real world moves on.

And believe it or not, staying up to date is only the tip of the operational iceberg.

Materialize uses SQL not only to query the present, but also to describe how it should respond to future events.
Your operational work shifts from being a repeated sequence of imperative SQL commands to declarative SQL views that describe your business logic.
This allows Materialize to accept responsibility for ongoing operational work, and to act autonomously where appropriate.
And it allows *you* to declaratively specify much of your operational layer, avoiding a tangle of scripts, cron jobs, and baling twine.

In this post we'll unpack how Materialize views freshness, see how it introduces autonomy at different moments, and call out the work you currently do that it can do for you instead.
We'll build up to an end-to-end demonstration borrowing from our [guided tutorial](https://materialize.com/docs/get-started/quickstart/).

## Freshness in Materialize

At the heart of freshness in Materialize is autonomous proactive work, done in response to the arrival of data rather than waiting for a user command.
User commands still exist, and Materialize promptly responds to them too, but many of the commands set up ongoing work rather than one-off work.
The proactive ongoing work spans data ingestion, view and index maintenance, and onward streaming outputs.
All of this work aims to minimize the time from data updates to their reflection in indexes (for querying) and output streams (for action).

In addition to acting proactively, we need to carefully consider the work we choose to do.
One can't simply re-do all work on each data update; we'll end up continually behind rather than at all ahead.
Ideally, we would do the *same* work as for batch processing, only performed eagerly (as the updates arrive) rather than lazily (once the batch completes).
This principle ensures that we remain throughput-competitive with batch systems, while minimizing the latency for data updates.

Let's examine the proactive work across Materialize's ingestion, computation, and output layers.

### Autonomy in Ingestion

Materialize draws input data from [sources](https://materialize.com/docs/sql/create-source/): tables maintained by external systems that Materialize should faithfully reflect.
Examples include PostgreSQL databases (through their replication log) and Kafka topics.
Materialize continually monitors these external systems, and receives data updates the first moment the systems make them available.

As Materialize receives data updates it timestamps them and commits them to its own durable storage.
The storage layer uses an append-friendly changelog format that does not need to rewrite existing data.
Log compaction happens in the background, off of the critical path and without impeding data ingestion.
Updates are available to users and their uses as soon as the timestamped data are durably committed to the OLTP database containing Materialize's storage metadata.

This ongoing work pulls data in as soon as Materialize has access to it, and attempts to do as little as possible to make it durable and then reveal it to users.
The result is continual freshness of ingested data, always as current as upstream systems have presented it.

### Autonomy in Computation

Many operational systems record data updates promptly, and then invite you to query it.
While useful, that invitation stops short of any consequent operational work that needs to be done.
If you have business logic that depends on those changed data, you'd really like to see the changes in the *outputs* rather than the *inputs*.
You'd like someone to *maintain* your business logic for you.

Materialize's maintenance of views and indexes is driven by [differential dataflow](https://github.com/TimelyDataflow/differential-dataflow), a compute engine specifically designed to minimize the end-to-end latency of data updates.
Differential dataflow provides carefully implemented data-parallel operators (e.g. `map`, `reduce`, `join`) and Materialize translates your SQL into a dataflow of these operators.
To read more about the implementation of these atomic operators, and the properties of differential dataflow generally, we recommend [the VLDB paper on Shared Arrangements](http://www.vldb.org/pvldb/vol13/p1793-mcsherry.pdf).

Even with differential dataflow, Materialize needs to carefully construct dataflows to ensure that updates happen both promptly and efficiently.
A not-uncommon pattern in other systems with shallower incremental view maintenance (IVM) support is that they fall back to expensive implementations when queries stray outside of the range of SQL the system's IVM supports.
Materialize uses the same engine to both evaluate queries and to incrementally maintain them, so it doesn't have exceptions to its IVM support.

Let's look at three examples of SQL that can be challenging to maintain in other systems: supporting updates and deletions, correlated subqueries, and recursion.

SQL aggregations `MIN` and `MAX` are not hard to maintain incrementally when you only insert data, but life gets much harder when you update or delete input data.
Your continued deletions (imagine implementing a priority queue) can eventually make any input record become the correct answer.
Materialize ensures this happens both correctly and promptly by performing aggregation in a tree, and leaving this tree structure behind as the state to maintain. 
The same construction applies equally well to maintaining views containing `ORDER BY .. LIMIT ..` clauses.

```sql
-- You can *retract* arbitrary rows from `input_tbl`,
-- and can make any input row become the correct answer.
SELECT key_col, MIN(col1), MAX(col2), ..
FROM input_tbl
GROUP BY key_col;
```
When `input_tbl` is append-only, either because its source is append-only or because this is a one-off query, Materialize is able to use the leaner implementation that keeps only the results for each `key_col`.
When `input_tbl` can change arbitrarily, Materialize prepares to minimize the update time for any changes, including retractions.

SQL has the concept of "correlated subquery" which behave as if you you were to issue a new query for each record in some table.
Similarly, SQL's `LATERAL` join keyword allows you to manually correlate subqueries. 
For example, 
```sql
SELECT * FROM
    input_tbl,
    LATERAL (
        -- As if re-queried for each row in `input_tbl`.
        SELECT col1, col2... FROM other_tbl
        WHERE other_tbl.key_col = input_table.key_col
          AND other_tbl.val_col > input_table.val_col
        ORDER BY other_tbl.ord_col LIMIT k
    )
```
Materialize rewrites all queries to be free of subqueries in a process called decorrelation ([described here by Neumann and Kemper](https://cs.emis.de/LNI/Proceedings/Proceedings241/383.pdf)).
This way, Materialize is able to incrementally maintain arbitrary correlated subqueries.

SQL allows you to write recursive queries with `WITH RECURSIVE`.
This powerful construct is often vexxing, and we are unaware of other systems that are able to incrementally maintain anything like it for general queries.
Fortunately, differential dataflow supports recursive natively, and Materialize supports incremental evaluation and maintenance through its (slightly different) [`WITH MUTUALLY RECURSIVE`](https://materialize.com/docs/sql/recursive-ctes/#details) construct.

Not all of Materialize's dataflows are flawless.
Window functions in particular are challenging to support in their full generality, as they allow rich computation and aren't as easily eliminated as are correlated subqueries.
However they, like any other limitations, are being actively pursued and should only improve!

Although there is a lot to know here, Materialize's computation layer is continually working to maintain your SQL views and indexes as the underlying data change.
This is all in pursuit of freshness, pushing data updates through business logic proactively, both to be ready with fresh indexed results and to communicate them onward.

### Autonomy in Query Serving

The most common mode of interaction with a SQL system, the `SELECT` query, isn't great from the perspective of freshness.
You are required to repeatedly ask the system for results, and when there is a change you need to be the one to notice it.

Materialize adds a new command, [`SUBSCRIBE`](https://materialize.com/docs/sql/subscribe/), which like `SELECT` gives you the answer to your query, but then continues with a stream of timestamped updates that tell you about changes to those results as soon as they happen.
The `SUBSCRIBE` command allows you to build fresh applications without continually hammering the systems with polling `SELECT` statements.

Materialize also has the concept of a [SINK](https://materialize.com/docs/sql/create-sink/), which is roughly the output complement to an input `SOURCE`: it pushes the information of a `SUBSCRIBE` on to an external system, such as a Kafka topic.
Downstream systems can listen to these sinks to see updates to maintained views as soon as they happen.

Let's see `SUBSCRIBE` in action, using an example from our [guided tutorial](https://materialize.com/docs/get-started/quickstart/). 
Specifically, we'll head to ["Step 3: See results change!"](https://materialize.com/docs/get-started/quickstart/#step-3-see-results-change), in case you'd like to follow along.
In this example we have a large, continually changing view `winning_bids` of auction winners, some of which may correspond to fraudulent accounts.
We introduce a new table on the side, `fraud_accounts`, and want to monitor the top non-fraudent auction winners, written
```sql
SUBSCRIBE TO (
  SELECT buyer, count(*)
  FROM winning_bids
  WHERE buyer NOT IN (SELECT id FROM fraud_accounts)
  GROUP BY buyer
  ORDER BY 2 DESC LIMIT 5
);
```
We can look at the output and take any of the top buyers and (perhaps unfairly) flag them as fraudulent by inserting them into `fraud_accounts`.
 Perhaps we investigate and clear them, then deleting them from `fraud_accounts`. 
 Each action results in an immediate update to the `SUBSCRIBE` output.
The example demonstrates each of the layers, ingesting updates promptly from both tables and sources, moving the updates through an `ORDER BY .. LIMIT` dataflow with a (non-correlated) subquery, and surfacing output updates as soon as they occur.

The `SUBSCRIBE` and `SINK` constructs allow Materialize to serve fresh results as soon as they happen.
Users and applications are not required to anticipate changes, nor poll the system on a tight cadence.

## Freshness and Operational Autonomy

An operational layer wants to be able to connect the dots from input updates and events, through business logic, on to downstream systems that can take the appropriate actions.
To achieve this one must build autonomy into each of the layers of ingestion, computation, and serving.
If any of these layers aren't fully autonomous, you or code acting on your behalf will have to poke them into action on some regular basis.
You'll also likely be responsible for interpreting the results and determining if they merit propagating onward.

Materialize specifically allow you to install operational business logic that keeps its results up to date and allows others to take action the moment results change.
It does this by making its internal components update autonomously and proactively, as updates to data occur.
Materialize can absorb end-to-end responsibility for this operational work, framed as SQL views.

If freshness and operational autonomy sound exciting to you, we invite you to try out Materialize for yourself.
Our [guided tutorial](https://www.materialize.com/docs/get-started/quickstart/) builds up the auction data sources described above, and includes demonstrations of consistency.
If you'd like to try out Materialize on larger volumes of your own data, reach out about doing a [Proof of Concept](https://materialize.com/trial/) with us!

<!-- ##{"timestamp":1695963600}## -->