
Consistency is one facet of Materialize's "Trust" pillar, the others being responsiveness and freshness.
It turns out that being super responsive and ultra fresh doesn't amount to much if the results don't make any sense.
The last thing you need in your operational data plane is a layer that introduces chaos and confusion, even if it is fast and scalable.
*Especially* if it is fast and scalable.

Many popular platforms ultimately bring weak consistency properties.
We've discussed in [our product principles post](https://materialize.com/blog/operational-attributes/) how caches and bespoke microservices are one way to get both responsiveness and freshness, but at the expense of consistency.
But even internally consistent platforms, like some stream processors and data warehouses, often end up wrapped in caches and serving layers for operational work. 
Their consistency properties largely go out the window at that point, and it becomes your job to make sure that these systems operate as intended.

At Materialize we believe consistency is at the heart of the value that a database provides.
The *order* that a database introduces is why you use one, rather than a heap of JAR files pointed at various Kafka topics.
For those of you with a heap of JAR files and Kafka topics, this post is for you.

Informally, consistency speaks to Materialize *appearing* to simply process commands and events in the order they happen in the real world.
While the reality is that no scalable data platform does anything nearly so simple, responsible platforms don't let that become your problem.
Materialize is a responsible platform, and it opts you in to the strongest consistency guarantees we know of: [strict serializability](https://jepsen.io/consistency/models/strict-serializable).
Although powerful, these database guarantees needs to be extended from command-response operation (pull) to streaming operation (push), as Materialize supports both concurrently.

In this post we will unpack Materialize's consistency guarantees, show them happening in a [playground environment](https://materialize.com/register/), and help you probe and evaluate the consistency properties of other tools you may be using for your operational work.


### Consistency a la Databases

Ironically perhaps, the term "consistency" means many different things to folks in the databases, distributed systems, and big data spaces.
For a helpful introduction I recommend [the Jepsen page on consistency models](https://jepsen.io/consistency).
The tl;dr there is that [strict serializable](https://jepsen.io/consistency/models/strict-serializable) is what you wish were the case: all interactions are applied in an order that tracks the order they happened in the real world.
The other, weaker models introduce semantic anomalies in the interest of avoiding performance anomalies (up to and including database unavailability).
That doesn't mean the other models are inherently bad, but they are certainly spookier and require more expertise on your part.

Materialize supports both [strict serializable]((https://jepsen.io/consistency/models/strict-serializable)) and [serializable](https://jepsen.io/consistency/models/serializable) operation.
Serializability still requires interactions be applied in some order, but the order doesn't need to match the real world;
for example, you could be served stale results in order to see them faster than if you waited for the results to catch up to their fresh inputs.
We start you off with strict serializability so that you aren't surprised by the apparent mis-orderings of (non-strict) serializability, and then teach you about the latter if you believe you need to squeeze more performance out of Materialize and can absorb the potential confusion.

However, definitions like strict serializability and serializability only apply to systems that accept commands and provide responses.
There are other dimensions to consistency as we move into the world of streamed inputs, maintained views, and streamed outputs.
Let's dive into those now!

### Consistency in Materialize

Although Materialize fits the mold of an interactive SQL database, and provides the guarantees of one, it has additional streaming touchpoints.
Input data can be provided by external sources like Kafka and Postgres, which do not "transact" against Materialize.
Materialized views are kept always up to date, as if they are refreshed instantaneously on each data update.
Output data can be provided to external sinks like Kafka, as streams of events rather than sequences of transactions.
We need to speak clearly about how Materialize's consistency guarantees integrate with these features.

These three concerns lie at the heart of an operational data warehouse, whose outputs and actions must faithfully represent business logic applied to their inputs.
Without this guarantee, it is not entirely clear what an operational platform will and will not do on your behalf.

---

Although things sound like they might be about to get more complicated, I think they actually get *easier*, by getting more specific about how we maintain consistency in Materialize.

Materialize uses a concurrency control mechanism called [Virtual Time](https://materialize.com/blog/virtual-time-consistency-scalability/).
Every command and data update get assigned a virtual timestamp, and then Materialize applies these operations in the order of these timestamps. 
Although there is some subtlety to how we *assign* the timestamps to operations, once that step is done the system behaves in what we think is an largely unsurprising and thoroughly consistent manner.
Not only will Materialize behave as if all operations happen in *some* order, as required by serializability, *we can even show you what that order is*.

---

Properly prepared, let's now dive in to each of the three concerns above, which I'll call here input consistency, internal consistency, and output consistency.


#### Input Consistency

Materialize draws streamed input data from external sources, like Kafka and PostgreSQL.
Ideally, Materialize would assign timestamps to updates that exactly track the moments of change in the upstream data.
In practice, these sources are often insufficiently specific about their changes, and Materialize instead "reclocks" their sequence of states into its own virtual time.
When it does so, it assigns timestamps that aim to be consistent with the source itself.

Materialize durably records its timestamp assignment in auxiliary sources, as changing collections that at each time record the progress through the source so far.

PostgreSQL sources move forward using a "log sequence number", and you can see the current time and current log sequence number with the following query, where `pg_source_progress` just happened to be the name of the progress source.
```
materialize=> select mz_now(), * from pg_source_progress;
        mz_now |         lsn
---------------+-------------
 1695659907060 | 11695622984
(1 row)
```

Kafka is more complicated. Each topic is comprised of an unbounded number of partitions, each of which moves forward through integer offsets. 
Rather than a single `lsn`, each time has an association between partition ids and offsets, including a `0` for all partitions that have not yet come into existence.
The selection reports not a single number, but an offset for ranges of partitions.
```
materialize=> select mz_now(), * from kafka_source_progress;
        mz_now | partition |   offset
---------------+-----------+----------
 1695659699912 |     [0,0] | 40166616
 1695659699912 |     [1,1] | 40781940
 1695659699912 |     [2,2] | 40472272
 1695659699912 |      (2,) |        0
(4 rows)
```

When Materialize reclocks these sources into its own timestamps, it aims to maintain consistency with the inputs.
Specifically, it maintains the order of events in the underlying sources, it respects transaction boundaries when it is aware of them, and it could (but currently does not) transact against the upstream source to ensure that all writes are immediately visible.
Let's explore each of these properties.

Most streamed sources have a notion of order, in some cases a total order like PostgreSQL's replication log, and in some cases a weaker order like Kafka's partitioned topics.
Materialize's timestamp assignment should (and does) respect this order, so that you see a plausible database state.
Materialize records for each virtual timestamp the coordinates in the input order that describe the subset of data available at that timestamp. 
A new data update is assigned the first timestamp whose coordinates contain the update.
As long as the recorded coordinates move forward along the order as times increase, the revealed states of the data also move forward following the order.

For PostgreSQL we can verify that repeated inspection of the progress source shows an advancing timestamp and an advancing log sequence number.
```
materialize=> select mz_now(), * from pg_source_progress;
        mz_now |         lsn
---------------+-------------
 1695659907060 | 11695622984
(1 row)
materialize=> select mz_now(), * from pg_source_progress;
        mz_now |         lsn
---------------+-------------
 1695659910061 | 11695624104
(1 row)
materialize=> select mz_now(), * from pg_source_progress;
        mz_now |         lsn
---------------+-------------
 1695659911994 | 11695624568
(1 row)
```

Many streamed sources reveal transactional boundaries, such as PostgreSQL's replication log.
Kafka itself supports "transactional writes" but does not reveal the transaction boundaries to readers; you would need to use Debezium configured with a transaction topic to provide transaction information with it.
For PostgreSQL, Materialize assigns identical timestamps to all updates associated with the same transaction.
This ensures that other operations either see all or none of the updates in any transaction.

Finally, having written something to an upstream system (and received confirmation) you might like to be certain it is now available and reflected in Materialize.
This can be achieved by transacting against the upstream system for each timestamp we produce, but is not currently done by Materialize.
We think we should do it, however, and you should expect systems that can provide this level of fidelity to external data sources.

Timestamp assignment is the moment Materialize introduces order to its often inconsistent sources of data. 
It is also the moment we are able to be precise about the consistency properties we are able to maintain, and which we will need to invent.

#### Internal Consistency

Materialize has streaming internals, and uses them to continually keep various materialized views up to date.
Even with careful timestamps on input updates, with all the updates in motion through the streaming internals there is the real possibility that Materialize might reveal inconsistent results.
Inconsistent or transiently incorrect results are unacceptable for operational work; at best you have to stall your operational plane to sort things out, and at worst you may take irrevocable incorrect actions.

Many stream processors have the baffling property that their outputs need not correspond to any specific input.
This comes under the name of [eventual consistency](https://en.wikipedia.org/wiki/Eventual_consistency), which allows systems to be transiently incorrect as long as their inputs continue to change.
Inputs change pretty much always for stream processors, that's why you use them, leaving several popular systems with no specific consistency properties.
For an excellent overview, [Jamie Brandon's post on "internal consistency"](https://www.scattered-thoughts.net/writing/internal-consistency-in-streaming-systems/) evaluates this property for ksqlDB, Flink's Table API, and Materialize (and finds chaos in the non-Materialize entrants).

Materialize continually produces **specific** and **correct** outputs for its timestamped inputs.
Anything else is a bug.

We can see this in a playground environment using a query like Jamie used in his post.
Our [guided tutorial](https://materialize.com/docs/get-started/quickstart/) sets up a source of auction transactions, with buyers and sellers and bids.
Although many things change continually, we would hope that the sum of all credits through sales match the sum of all debits through sales.
They should always be exactly identical, and if even for a moment they are not that would be a bug in Materialize.

```sql
-- Maintain the credits due to each account.
CREATE MATERIALIZED VIEW credits AS
SELECT seller, SUM(amount) AS total
FROM winning_bids
GROUP BY seller;

-- Maintain the credits owed by each account.
CREATE MATERIALIZED VIEW debits AS
SELECT buyer, SUM(amount) AS total
FROM winning_bids
GROUP BY buyer;

-- Maintain the net balance for each account.
CREATE VIEW balance AS
SELECT 
    coalesce(seller, buyer) as id, 
    coalesce(credits.total, 0) - coalesce(debits.total, 0) AS total
FROM credits FULL OUTER JOIN debits ON(credits.seller = debits.buyer);

-- This will always equal zero.
SELECT SUM (total) FROM balance;
```

Importantly, nothing about the above example relies on the views being created in the same session, by the same person, team, or even running on the same physical hardware.
Materialize will ensure that `credits`, `debits`, and `balance` always track exactly the correct answer for the timestamped input, and will always have a net balance of zero.

To assess internal consistency for systems, Materialize and others, it can help to write views that track *invariants* of your data. 
If there is something you know should always hold, for example that the net balances are zero, then you can observe the results and watch for a result that violates the invariant.

You can similarly be certain that when you see a result that it corresponds to the correct answer on a specific input. 
For example, if you want to notify those users whose balance is below 100, the following view is certain to only report users for which it *actually happened*.

```sql
SELECT mz_now(), * FROM balance WHERE total < -100
```

The `mz_now()` column will report the exact time at which the input data yielded a low balance.

All results Materialize produces are the specific answers to the query on the input data as it existed at the query time.

#### Output Consistency

Finally, having both ingested and maintained results, Materialize needs to speak clearly about its results to external systems.
We saw just above that a `SELECT` query can use `mz_now()` to learn the specific moment at which query results were correct.
However, the full power of Materialize unlocks when you connect its views as streaming outputs onward to downstream applications or systems.
How does Materialize speak clearly and unambiguously to these streaming consumers?

Materialize connects to three different types of downstream consumer, but as we will see it follows identical principles for each.
Materialize can return streamed changelogs for views in a standard SQL session using its [`SUBSCRIBE`](https://materialize.com/docs/sql/subscribe/) command.
It can also stream those same changelogs on to external systems, like Kafka and RedPanda, using its [`CREATE SINK`](https://materialize.com/docs/sql/create-sink/) command.
Finally, Materialize also commonly writes data back to *itself*, to fan out to other users and uses, through its [`CREATE MATERIALIZED VIEW`](https://materialize.com/docs/sql/create-materialized-view/) command.
Although different types of endpoints, all three communicate the same information: exactly what changed in a view and exactly when did those changes happen.

To communicate clearly Materialize follows certain rules for its changelogs.
Each changelog begins at a specific timestamp with the collection snapshot at that timestamp.
Each record changes only once for each timestamp, and that timestamp is explicitly recorded with the change.
Each timestamp is regularly indicated to be complete, even when no changes occur.
These properties remove ambiguity about what the changes were, when they happened, and whether there are any more coming for any given timestamp.

Let's take a peek using the `SUBSCRIBE` command, simply watching the count of the number of auctions that have been won.

```
materialize=> copy (
    subscribe (select count(*) from winning_bids) 
         with (progress = true)
) to stdout;
```

I pressed `ENTER` between blocks of returned results to suggest at the live experience, and added comments to these lines that describe the *preceding* block of responses.

```
1695653291958	t	\N	\N
-- Timestamp of initial snapshot
1695653291958	f	1	38549
1695653293090	f	-1	38549
1695653293090	f	1	38550
1695653298001	t	\N	\N
-- Initial snapshot and immediate change
1695653299001	t	\N	\N
1695653299105	t	\N	\N
1695653299105	f	-1	38550
1695653299105	f	1	38551
1695653300001	t	\N	\N
-- Brief break before next change
1695653301001	t	\N	\N
1695653302001	t	\N	\N
1695653303001	t	\N	\N
...
-- Nothing happens for a while.
```

The columns of each returned row are: first the timestamp in milliseconds since 1970, second "is this a watermark", third the change in the cardinality of the record, and finally the payload columns of the record itself.
Watermark records indicate only the forward progress of times, that all future timestamps will be at least so large, and have null values for columns other than the timestamp.

There are four blocks of output to unpack.
1. The first and immediate block of output is the "initial snapshot timestamp" progress message, which tells us the time the initial snapshot of the `SUBSCRIBE` will reflect.
2. The second block of output includes the snapshot first. As the snapshot requires spinning up a dataflow (`winning_bids` is a non-materialized view), some additional input changes happen before we have the snapshot, and we report their output changes as well.
3. The next block is now live and reports a new update just as it happens, from `38550` to `38551`, and confirms that there are no further changes at that time.
4. The last block reports multiple seconds proceeding for which the count does not change.

These blocks each report the correct `COUNT(*)` output at the exact times the inputs change. 
Materialize will wait until it is certain of the exact updates for a time, including that they are durably committed, before reporting them.

Although other destinations differ from `SUBSCRIBE`, each have access to an ongoing stream of precise information detailing exactly what changed, when it changed, and whether more changes are due.
This information communicates to consumers the moment a change has certainly occurred, giving them the confidence to act immediately.

## Consistency and Operational Confidence

Consistency is critical on operational workflows because there are actions that need to be taken.
Many of these actions have consequences, and if they are directly driven by an inconsistent platform it is up to you to diagnose and debug any resulting glitchy behavior.
These glitches have consequences too, some of which can be corrected after the fact and some of which cannot.
Operational platforms provide value in part by introducing and maintaining consistency for you, avoiding unintended actions and their consequences.

Materialize specifically provides strict serializability, and extends this to its streaming ingestion, transformation, and onward communication.
This guarantee means Materialize behaves *as if* it applied all commands in an order that matches how they happened in the real world.
In reality Materialize is massively concurrent, but it absorbs this complexity and presents as a surprisingly capable single operator.

If this resonates with you, especially if you have heaps of JAR files and Kafka topics, we invite you to try out Materialize for yourself.
Our [guided tutorial](https://www.materialize.com/docs/get-started/quickstart/) builds up the auction data sources described above, and includes demonstrations of consistency.
If you'd like to try out Materialize on larger volumes of your own data, reach out about doing a [Proof of Concept](https://materialize.com/trial/) with us!

<!-- ##{"timestamp":1695099600}## -->