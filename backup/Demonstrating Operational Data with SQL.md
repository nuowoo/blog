Databases, Big Data, and Stream Processors have long had the property that it can be hard to *demonstrate* their value, like in a demo setting.
Databases coordinate the work of multiple teams of independent workers, and don't shine when there is just one user.
Big Data systems introduce scalable patterns that can be purely overhead when the data fit on a single laptop.
Stream Processors aim to get the lowest of end-to-end latencies, but do nothing of any consequence on static data.
These systems demonstrate value when you have variety, volume, and velocity, and most demo data sets have none of these.

Materialize, an operational data warehouse backed by scalable streaming systems, has all three of these challenges!

Fortunately, Materialize is powerful enough to synthesize its own operational data for demonstration purposes.
In this post, we'll build a recipe for a generic live data source using standard SQL primitives and some Materialize magic.
We'll then add various additional flavors: distributions over keys, irregular validity, foreign key relationships.
It's all based off of Materialize's own [auction load generator](https://materialize.com/docs/sql/create-source/load-generator/#auction), but it's written entirely in SQL and something that I can customize as my needs evolve.

The thing I find most amazing here is that with just SQL you can create *live* data. 
Data that comes and goes, changes, and respects invariants as it does.
And that the gap between your idea for live data and making it happen is just typing some SQL.

### My Motivation: Materialize

Materialize has a few product beats it wants to hit when we demo it, derived from our product principles.

* **Responsiveness**: Materialize should be able to get back to you ASAP, even with lots of data involved.
* **Freshness**: Materialize should reflect arbitrary updates almost immediately, even through complex logic.
* **Consistency**: Materialize's outputs should always reflect a consistent state, even across multiple users and views.

We want to get folks to that "aha!" moment where they realize that Materialize is like no other technology they know of.
Until that moment, Materialize could just be a trenchcoat containing Postgres, Spark, and Flink stacked according to your preferences.

Of course, different contexts connect for different users.
Some folks think about transactions and fraud and want to see how to get in front of that.
Others have users of their own, and know that sluggish, stale, inconsistent results are how they lose their users, and want to feel the lived experience.
Many users won't believe a thing until the data looks like their data, with the same schemas and data distributions, and the same business logic.
These are all legitimate concerns, and to me they speak to the inherent *heterogeneity* involved in demonstrating something.

I want to be able to demonstrate Materialize more **effectively**, which is some amount tied up in demonstrating it more **flexibly**.

As a personal first, I'm going to try telling the story in reverse order, Memento-style.
We'll start with the outcomes, which I hope will make sense, and then figure out how we got there, and eventually arrive at the wall of SQL that makes it happen.
It does mean we'll need some suspension of disbelief as we go, though; bear with me!
I do hope that whichever prefix you can tolerate makes sense and is engaging, and am only certain that if we started with the SQL it would not be.

The outine is, roughly:

1.  [Demonstrating Materialize with auction data](https://github.com/frankmcsherry/blog/blob/master/posts/2024-05-19.md#demonstrating-materialize)

    We'll work through Materialize's quick start to show off `auctions` and `bids` data, and give a feel for what we need to have our live data do.
    We're going to hit the beats of responsiveness, freshness, and consistency along the way.

2.  [Building an Auction loadgen from unrelated live data](https://github.com/frankmcsherry/blog/blob/master/posts/2024-05-19.md#auction-data-from-changing-moments)

    Here we'll build live views that define `auctions` and `bids`, starting from a live view that just contains recent timestamps.
    We'll see how to turn largely nonsense data into plausible auctions and bids, through the magic of pseudorandomness.

3.  [Building live random data from just SQL](https://github.com/frankmcsherry/blog/blob/master/posts/2024-05-19.md#operational-data-from-thin-air)

    Starting from nothing more than SQL, we'll create a live view that Materialize can maintain containing recent moments as timestamps.
    As time continually moves forward, those moments continually change.

4.  [All the SQL](https://github.com/frankmcsherry/blog/blob/master/posts/2024-05-19.md#appendix-all-the-sql) Really, just SQL.

Feel more than welcome to leap to the sections that interest you most.
I recommend starting at the beginning, though!

### Demonstrating Materialize

Let's sit down with Materialize and some live auction data and see if we can't hit the beats of responsiveness, freshness, and consistency.
The story is borrowed from our own quickstart, but by the end of it we'll find we've swapped out the quickstart's built-in load generator.

Materialize's [`AUCTION` load generator](https://materialize.com/docs/sql/create-source/load-generator/#auction) populates `auctions` and `bids` tables.
Their contents look roughly like so:
```
materialize=> select * from auctions;
 id | seller |        item        |          end_time          
----+--------+--------------------+----------------------------
  2 |   1592 | Custom Art         | 2024-05-20 13:43:16.398+00
  3 |   1411 | City Bar Crawl     | 2024-05-20 13:43:19.402+00
  1 |   1824 | Best Pizza in Town | 2024-05-20 13:43:06.387+00
  4 |   2822 | Best Pizza in Town | 2024-05-20 13:43:24.407+00
  ...
(4 rows)
```
```
materialize=> select * from bids;
 id | buyer | auction_id | amount |          bid_time          
----+-------+------------+--------+----------------------------
 31 |    88 |          3 |     67 | 2024-05-20 13:43:10.402+00
 10 |  3844 |          1 |     59 | 2024-05-20 13:42:56.387+00
 11 |  1861 |          1 |     40 | 2024-05-20 13:42:57.387+00
 12 |  3338 |          1 |     97 | 2024-05-20 13:42:58.387+00
 ...
```

We will root around in this data, as it changes, and show off Materialize as something unlike other data tools.
Specifically we'll want to show off responsiveness, freshness, and consistency, which we'll do in that order.
However, the point is that you get them all at the same time, rather than one at a time, and by the end we should be able to see all three at once.

#### Beat 1: Responsiveness

Materialize is able to respond immediately, even to complex queries over large volumes of data.
Let's start by looking at the data, counting the number of auctions and the number of bids.
```
materialize=> select count(*) from auctions;
 count 
-------
 86400
(1 row)

Time: 52.580 ms
```
```
materialize=> select count(*) from bids;
  count   
----------
 10994252
(1 row)

Time: 8139.897 ms (00:08.140)
```
It's almost 100k auctions, and over 10M bids across them.
The specific numbers will make more sense when we get to the generator, but some of you may already recognize 86,400.
Ten seconds to count ten million things is not great, but this is running on our smallest instance (`25cc`; roughly 1/4 of a core).
Also, we aren't yet using Materialize's super-power to *maintain* results.

Materialize maintains computed results in indexes, created via the `CREATE INDEX` command.
```sql
-- Maintain bids indexed by id.
CREATE INDEX bids_id ON bids (id);
```

When we want to find a specific bid by id, this can be very fast .
```
materialize=> select * from bids where id = 4;
 id | buyer | auction_id | amount |        bid_time        
----+-------+------------+--------+------------------------
  4 |   228 |    6492730 |    149 | 2024-06-19 13:57:50+00
(1 row)

Time: 19.711 ms
```
Inspecting the query history (a feature in Materialize's console) we can see it only took 5ms for the DB, and the additional latency is between NYC and AWS's us-east-1.
This really is just a look-up into a maintained index, admittedly only on `bids` rather than some sophisticated query.

You can build indexes on any collection of data, not just raw data like `bids`.
We could build an index on `SELECT COUNT(*) FROM bids` to make that fast too, for example.
Instead, let's go straight to the good stuff.

Here's a view that determines which auctions are won by which bids.
```sql
-- Determine auction winners: the greatest bid before expiration.
CREATE VIEW winning_bids AS
  SELECT DISTINCT ON (auctions.id) bids.*,
    auctions.item,
    auctions.seller
  FROM auctions, bids
  WHERE auctions.id = bids.auction_id
    AND bids.bid_time < auctions.end_time
    AND mz_now() >= auctions.end_time
  ORDER BY auctions.id,
    bids.amount DESC,
    bids.bid_time,
    bids.buyer;
```

Directly querying this view results in a not-especially-responsive experience
```
materialize=> select auction_id, buyer, amount from winning_bids limit 5;
 auction_id | buyer | amount 
------------+-------+--------
        217 |    41 |    252
       3328 |   209 |     55
      19201 |   147 |    255
      18947 |    34 |    254
       7173 |   143 |      5
(5 rows)

Time: 87428.589 ms (01:27.429)
```
We are grinding through all the bids from scratch when you select from a view, because the view only explains what query you want to run.
A view by itself doesn't cause any work to be done ahead of time.

However, we can create indexes on `winning_bids`, and once they are up and running everything gets better.
We are going to create two indexes, on the columns `buyer` and `seller`, for future storytelling reasons.
```sql
-- Compute and maintain winning bids, indexed two ways.
CREATE INDEX wins_by_buyer ON winning_bids (buyer);
CREATE INDEX wins_by_seller ON winning_bids (seller);
```
The auctions aren't faster to magic in to existence than the original query was, so we'll have to wait a moment for them to hydrate.
Once this has happened, you get responsive interactions with the view.
```
materialize=> select auction_id, buyer, amount from winning_bids limit 5;
 auction_id | buyer | amount 
------------+-------+--------
    7647534 |     0 |    254
    6568079 |     0 |    239
   10578840 |     0 |    254
   14208479 |     0 |    249
   15263465 |     0 |    199
(5 rows)

Time: 61.283 ms
```
Rather than grind over the ten million or so bids to find winners, the ~80,000 results are maintained and its easy to read the first five.
Moreover, the results are all immediately up to date, rather than being fast-but-stale.
Let's hit that **freshness** beat now!

<!-- 
In addition, our indexes set us up for responsive ad-hoc queries.
Here's an example where we look for "auction flippers": folks who are both buyers and sellers of the same item at increased amounts:
```sql
-- Look for users who re-sell their winnings
CREATE VIEW potential_flips AS
  SELECT w2.seller,
         w2.item AS item,
         w2.amount AS seller_amount,
         w1.amount AS buyer_amount
  FROM winning_bids w1,
       winning_bids w2
  WHERE w1.buyer = w2.seller
    AND w2.amount > w1.amount
    AND w1.item = w2.item;
```

We have enough auctions that some folks will be both buyers and sellers, and for some fraction of them its the same item for an increased price.
```
materialize=> select count(*) from potential_flips;
 count 
-------
  9755
(1 row)

Time: 602.481 ms
```
```
materialize=> select seller, count(*) from potential_flips group by seller order by count(*) desc limit 5;
 seller | count 
--------+-------
  42091 |     7
  42518 |     6
  10529 |     6
  39840 |     6
  49317 |     6
(5 rows)

Time: 678.330 ms
```

This is now pretty interactive, using scant resources, over enough data and through complex views that to start from scratch would be exhausting.
However, maintained indexes keep intermediate results up to date, and you get the same results as if re-run from scratch, just without the latency. -->

#### Beat 2: Freshness

All of this auction data is synthetic, and while it changes often the show is pretty clearly on rails.
That is, Materialize knows ahead of time what the changes will be.
You want to know that Materialize can respond fast to *arbitrary* changes, including ones that Materialize doesn't anticipate.

We need **interaction**!

Let's create a table we can modify, through our own whims and fancies.
Our modifications to this table, not part of the load generator, will be how we demonstrate the speed at which Materialize updates results as data change.
```sql
-- Accounts that we might flag for fraud.
CREATE TABLE fraud_accounts (id bigint);
```

Let's look at a query that calls out the top five accounts that win auctions.
We'll subscribe to it, meaning we get to watch the updates as they happen.
```sql
-- Top five non-fraud accounts, by auction wins.
COPY (SUBSCRIBE TO (
  SELECT buyer, count(*)
  FROM winning_bids
  WHERE buyer NOT IN (SELECT id FROM fraud_accounts)
  GROUP BY buyer
  ORDER BY count(*) DESC, buyer LIMIT 5
)) TO STDOUT;
```
This produces first a snapshot and then a continual stream of updates.
In our case, the updates are going to derive from our manipulation of `fraud_accounts`.
```
1718981380562	1	7247	7
1718981380562	1	17519	7
1718981380562	1	27558	7
1718981380562	1	20403	7
1718981380562	1	16584	7
```
The data are not really changing much, on account of the winners all having the same counts.
But, this is actually good for us, because we can see what happens when we force a change.

At this point, let's insert the record `17519` into `fraud_accounts`.
```
-- Mark 17519 as fraudulent
1718981387841	-1	17519	7
1718981387841	1	32134	7
```
We can do the same with `16584`, and then `34985`.
```
-- Mark 16584 as fraudulent
1718981392977	1	34985	7
1718981392977	-1	16584	7
-- Mark 34985 as fraudulent
1718981398158	1	35131	7
1718981398158	-1	34985	7
```
Finally, let's remove all records from `fraud_accounts` and we can see that we return back to the original state.
```
-- Remove all fraud indicators.
1718981403087	-1	35131	7
1718981403087	1	17519	7
1718981403087	-1	32134	7
1718981403087	1	16584	7
...
```
That `34985` record isn't mention here because it only showed up due to our other removals.
We don't hear about a change because there is no moment when it is in the top five, even transiently.
That is a great lead-in to Materailize's **consistency** properties!

#### Beat 3: Consistency

All the freshness and responsiveness in the world doesn't mean much if the results are incoherent.
Materialize only ever presents actual results that actually happened, with no transient errors.
When you see results, you can confidently act on them knowing that they are real, and don't need further second to bake.

Let's take a look at consistency through the lens of account balances as auctions close and winning buyers must pay sellers.
```sql
-- Account ids, with credits and debits from auctions sold and won.
CREATE VIEW funds_movement AS
  SELECT id,
         SUM(credits) AS credits,
         SUM(debits) AS debits
  FROM (
    SELECT seller AS id, amount AS credits, 0 AS debits
    FROM winning_bids
    UNION ALL
    SELECT buyer AS id, 0 AS credits, amount AS debits
    FROM winning_bids
  )
  GROUP BY id;
```

These balances derive from the same source: `winning_bids`, and although they'll vary from account to account, they should all add up.
Specifically, if we get the total credits and the total debits, they should 100% of the time be exactly equal.
```sql
-- Discrepancy between credits and debits.
SELECT SUM(credits) - SUM(debits) 
FROM funds_movement;
```
This query reports zero, 100% of the time.
We can `SUBSCRIBE` to the query to be notified of any change.
```
materialize=> COPY (SUBSCRIBE (
    SELECT SUM(credits) - SUM(debits) 
    FROM funds_movement
)) TO STDOUT;

1716312983129	1	0
```
This tells us that starting at time `1716312983129`, there was `1` record, and it was `0`.
You can sit there a while, and there will be no changes.
You could also add the `WITH (PROGRESS)` option, and it will provide regular heartbeats confirming that second-by-second it is still zero.
The credits and debits always add up, and aren't for a moment inconsistent.

We can set up similar views for other assertions.
For example, every account that has sold or won an auction should have a balance.
A SQL query can look for violations of this, and we can monitor it to see that it is always empty.
If it is ever non-empty, perhaps there are bugs in the query logic, its contents are immediately actionable: 
there is a specific time where the inputs evaluated to an invariant-violating output, and if you return to that moment you'll see the inputs that produce the bad output.

The consistency extends across multiple independent sessions.
The moment you get confirmation that the insert into `fraud_accounts`, you can be certain that no one will see that account in the top five non-fraudulent auction winners.
This guarantee is called "strict serializability", that the system behaves as if every event occurred at a specific time between its start and end, and is the strongest guarantee that databases provide.

#### Demo over!

That's it!
We've completed the introduction to Materialize, and used auction data to show off responsiveness, freshness, and consistency.
There's a lot more to show off, of course, and if any of this sounded fascinating you should swing by https://materialize.com/register/ to spin up a trial environment.

However, in this post we will continue to unpack how we got all of that `auctions` and `bids` data in the first place!

### Auction Data from Changing Moments

Where do the `auctions` and `bids` data come from?
You can get them from our load generator, but we're going to try and coax them out of raw SQL.
We're going to start with something we haven't introduced yet, but it's a view whose content looks like this
```sql
-- All seconds within the past 24 hours.
CREATE VIEW moments AS
SELECT generate_series(
    now() - '1 day'::interval + '1 second'::interval,
    now(),
    '1 second'
) moment;
``` 

Unpacking this, `moments` contains rows with a single column containing a timestamp.
Whenever we look at it, the view contains those timestamps at most one day less than `now()`.
It should have at any moment exactly 86,400 records present, as many as `auctions` up above.

Importantly, this view definition will not actually work for us.
You are welcome to try it out, but you'll find out that while it can be *inspected*, it cannot be *maintained*.
We'll fix that by the end of the post, but it will need to wait until the next section.
For the moment, let's assume we have this view and the magical ability to keep it up to date.

These "moments" are not auction data, though.
How do we get from moments to auctions and bids?

The `auctions` and `bids` collections look roughly like so:
```
materialize=> select * from auctions;
 id | seller |        item        |          end_time          
----+--------+--------------------+----------------------------
  2 |   1592 | Custom Art         | 2024-05-20 13:43:16.398+00
  3 |   1411 | City Bar Crawl     | 2024-05-20 13:43:19.402+00
  1 |   1824 | Best Pizza in Town | 2024-05-20 13:43:06.387+00
  4 |   2822 | Best Pizza in Town | 2024-05-20 13:43:24.407+00
  ...
(4 rows)
```
```
materialize=> select * from bids;
 id | buyer | auction_id | amount |          bid_time          
----+-------+------------+--------+----------------------------
 31 |    88 |          3 |     67 | 2024-05-20 13:43:10.402+00
 10 |  3844 |          1 |     59 | 2024-05-20 13:42:56.387+00
 11 |  1861 |          1 |     40 | 2024-05-20 13:42:57.387+00
 12 |  3338 |          1 |     97 | 2024-05-20 13:42:58.387+00
 ...
```

Auctions have a unique id, a seller id, an item description, and an end time.
Bids have a unique id (no relation), a buyer id, an auction id, the amount of the bid, and the time of the bid.

The `seller`, `item`, `buyer`, and `amount` fields are all random, within some bounds.
As a first cut, we'll think about just using random values for each of the columns.
Where might we get randomness, you ask?
Well, if *pseudo*-randomness is good enough (it will be), we can use cryptographic hashes of the moments.
```sql
-- Extract pseudorandom bytes from each moment.
CREATE VIEW random AS
SELECT moment, digest(moment::text, 'md5') as random
FROM moments;
```
Let's start with bytes from `random` to populate columns, and we'd have a first cut at random data.
Columns like `auctions.item` are populated by joining with a constant collection (part of the generator), but `id` and `seller` could just be random.
The `end_time` we'll pick to be a random time up to 256 minutes after the auction starts.
```sql
-- Totally accurate auction generator.
CREATE VIEW auctions_core AS
SELECT 
    moment,
    random,
    get_byte(random, 0) + 
    get_byte(random, 1) * 256 + 
    get_byte(random, 2) * 65536 as id,
    get_byte(random, 3) +
    get_byte(random, 4) * 256 as seller,
    get_byte(random, 5) as item,
    -- Have each auction expire after up to 256 minutes.
    moment + (get_byte(random, 6)::text || ' minutes')::interval as end_time
FROM random;
```
We've clearly made some calls about how random each of these should be, and those calls influence what we'll see in the data.
For example, we've established at most 65,536 sellers, which lines up fine with our 86,400 auctions at any moment; some sellers will have multiple auctions and many will not.
Auctions are open for a few hours on average, close out but linger, and then vanish after 24 hours.
If we want to change any of these, perhaps to add more distinct items, or keep auctions running longer, or to skew the distribution over sellers, we can!

Similarly, the columns of `bids` are also pretty random, but columns like `auction_id` and `bid_time` do need to have some relationship to `auctions` and the referenced auction.
We'll build those out in just a moment, but have a bit more tidying to do for `auctions` first.

#### Adding Custom Expiration

Our auctions wind down after some random amount of time, but they are not removed from `auctions` for three hours.
Thematically we can think of this as auctions whose winners have been locked in, but whose accounts have not yet been settled.

If we want the auction to vanish from `auctions` at this time it closed, we could accomplish this with a temporal filter:
```sql
WHERE mz_now() < end_time
```
As soon as we reach `end_time` the auction would vanish from `auctions`.

This is a very helpful pattern for load generators that want to control when data arrive and when it departs, in finer detail than "a twenty four hour window".
For example, one could randomly generate `insert_ts` and `delete_ts`, and then use
```sql
-- Create an event that is live for the interval `[insert_ts, delete_ts]`.
WHERE mz_now() BETWEEN insert_ts AND delete_ts
```
This pattern allows careful control of when events *appear* to occur, by holding them back until `mz_now()` reaches a value, and then retracting them when it reaches a later value. 

#### Making More Realistic Data

Our random numbers for `item` aren't nearly as nice as what the existing load generator produces.
However, we can get the same results by putting those nice values in a view and using our integer `item` to join against the view.
```sql
-- A static view giving names to items.
CREATE VIEW items (id, item) AS VALUES
    (0, 'Signed Memorabilia'),
    (1, 'City Bar Crawl'),
    (2, 'Best Pizza in Town'),
    (3, 'Gift Basket'),
    (4, 'Custom Art');
```

Now when we want to produce an actual auction record, we can join against items like so
```sql
-- View that mirrors the `auctions` table from our load generator.
CREATE VIEW auctions AS
SELECT id, seller, items.item, end_time
FROM auctions_core, items
WHERE auction.item = items.id;
```

We've now got a view `auctions` that mirrors what Materialize's load generator produces, at least superficially.

#### Introducing Foreign Key Constraints

Each bid in `bids` references an auction, and we are unlikely to find an extant auction if we just use random numbers for `auction_id`.
We'd like to base our `bids` on the available auctions, and have them occur at times that make sense for the auction.

We can accomplish this by deriving the bids for an auction from `auctions` itself.
We will use some available pseudorandomness to propose a number of bids, and then create further pseudorandomness to determine the details of each bid.
```sql
CREATE VIEW bids AS
-- Establish per-bid records and pseudorandomness.
WITH prework AS (
    -- Create `get_byte(random, 6)` many bids for each auction, 
    -- each with their own freshly generated pseudorandomness.
    SELECT 
        id as auction_id,
        moment as auction_start,
        end_time as auction_end,
        digest(random::text || generate_series(1, get_byte(random, 6))::text, 'md5') as random
    FROM auctions_core
)
SELECT
    get_byte(random, 0) +
    get_byte(random, 1) * 256 +
    get_byte(random, 2) * 65536 as id,
    get_byte(random, 3) AS buyer,
    auction_id,
    get_byte(random, 4)::numeric AS amount,
    auction_start + (get_byte(random, 5)::text || ' seconds')::interval as bid_time
FROM prework;
```

We now have a pile of bids for each auction, with the compelling property that when the auction goes away so too do its bids.
This gives us "referential integrity", the property of foreign keys (`bids.auction_id`) that their referent (`auction.id`) is always valid.

And with this, we have generated the `auctions` and `bids` data that continually change, but always make sense.

There are several other changes you might want to make!
For example, random bids means that auctions stop changing as they go on, because new random bids are unlikely to beat all prior bids.
You could instead have the bids trend up with time, to keep the data interesting.
But, the changes are pretty easy to roll out, and just amount to editing the SQL that defines them.

Let's pause for now on noodling on ways we could make the data even more realistic.
Up next we have to unpack how we got that `moments` view in the first place.
Once we've done that, you are welcome to go back to playing around with load generator novelties and variations!

### Operational Data from Thin Air

Our `auctions` and `bids` data was based on a view `moments` that showed us all timestamps within the past three hours.
We saw how we could go from that to pretty much anything, through extracted pseudorandomness.

We used a view that seemed maybe too easy, that looked roughly like so:
```sql
-- Generate a sliding window over timestamp data.
-- Arguments: <volume>, <velocity>
SELECT moment,
FROM generate_series(
    '1970-01-01 00:00:00+00', 
    '2099-01-01 00:00:00+00', 
    <velocity>
) moment
WHERE now() BETWEEN moment AND moment + <volume>;
```

This example uses `generate_series` to produce moments at which events will occur.
The `<velocity>` argument chooses the step size of the `generate_series` call, and locks in the cadence of updates.
The `<volume>` argument controls for how long each record lingers, and sets the steady state size.
The result is a sliding window over random data, where you get to control the volume and velocity.

We used `'1 second'` for the velocity and `'1 day'` for the volume.

Now, while you can *type* the above, it won't actually run properly if you press enter.
The query describes 130 years of data, probably at something like a one second update frequency (because you wanted live data, right?).
I don't even know how to determine how many records this is accurately based on all the leap-action that occurs.
Moreover, you won't be able to materialize this view, because `now()` prevents materializations.

To actually get this to work, we'll have to use some clever tricks.
The coming subsections are a sequence of such tricks, and the punchline will be "it works!", in case that saves you any time.

#### Clever trick 1: using `mz_now()`

Our first clever trick is to move from `now()` to `mz_now()`.
These are very similar functions, where the `now()` function gets you the contents of the system clock, and `mz_now()` gets you the transaction time of your command.
The main difference between the two is that we can materialize some queries containing `mz_now()`, unlike any query containing `now()`.

```sql
-- Generate a sliding window over timestamp data.
SELECT moment,
FROM generate_series(
    '1970-01-01 00:00:00+00', 
    '2099-01-01 00:00:00+00', 
    '1 second'
) moment
--    /------\---- LOOK HERE!
WHERE mz_now() BETWEEN moment AND moment + '1 day';
```
This very simple change means that Materialize now has the ability to keep the query up to date.
Materialize has a feature called ["temporal filters"](https://materialize.com/docs/transform-data/patterns/temporal-filters/) that allows `mz_now()` in `WHERE` clauses, because we are able to invert the clause and see the moment (Materialize time) at which changes will occur.

Unfortunately, the implementation strategy for keeping this view up to date still involves first producing all the data, and then filtering it (we don't have any magical insight into `generate_series` that allows us to invert its implementation).
But fortunately, we have other clever tricks available to us.

#### Clever trick 2: Hierachical Generation

The problem above is that we generate all the data at once, and then filter it.
We could instead generate the years of interest, from them the days of interest, from them the hours of interest, then minutes of interest, then seconds of interest, and finally milliseconds of interest.
In a sense we are generating *intervals* rather than *moments*, and then producing moments from the intervals.

Let's start by generating all the years we might be interested in.
We start with all the years we might reasonably need, and a `WHERE` clause that checks for intersection of the interval (`+ '1 year'`) and the extension by volume (`+ '1 day'`).
```sql
-- Each year-long interval of interest
CREATE VIEW years AS
SELECT * 
FROM generate_series(
    '1970-01-01 00:00:00+00', 
    '2099-01-01 00:00:00+00', 
    '1 year') year
WHERE mz_now() BETWEEN year AND year + '1 year' + '1 day';
```
This view does not have all that many years in it. 
Roughly 130 of them.
Few enough that we can filter them down, and get to work on days.

At this point, we'll repeatedly refine the intervals by subdividing into the next granularity.
We'll do this for years into days, but you'll have to use your imagination for the others.
We have all the SQL at the end, so don't worry that you'll miss out on that.
```sql
-- Each day-long interval of interest
CREATE VIEW days AS
SELECT * FROM (
    SELECT generate_series(
        year, 
        year + '1 year' - '1 day'::interval, 
        '1 day') as day
    FROM years
)
WHERE mz_now() BETWEEN day AND day + '1 day';
```
We'll repeat this on to a view `seconds`, and stop there.

Although we could continue to milliseconds, experience has been that it's hard to demo things changing that quickly through SQL.
Lines of text flow past like the Matrix, and all you can really see is that there is change, not what the change is.

Unfortunately, there is a final gotcha.
Materialize is too clever by half, and if you materialize the `seconds` view, it will see that it is able to determine the entire 130 year timeline of the view, history and future, and record it for you.
At great expense.
These declarative systems are sometimes just too smart.

#### Clever trick 3: An empty table

We can fix everything by introducing an empty table.

The empty table is only present to ruin Materialize's ability to be certain it already knows the right answer about the future.
We'll introduce it to each of our views in the same place, and its only function is to menace Materialize with the possibility that it *could* contain data.
But it won't.
But we wont tell Materialize that.

```sql
-- Each day-long interval of interest
CREATE VIEW days AS
SELECT * FROM (
    SELECT generate_series(
        year, 
        year + '1 year' - '1 day'::interval, 
        '1 day') as day
    FROM years
    -- THIS NEXT LINE IS NEW!!
    UNION ALL SELECT * FROM empty
)
WHERE mz_now() BETWEEN day AND day + '1 day';
```

With these tricks in hand, we now have the ability to spin it up and see what it looks like.

```sql
CREATE DEFAULT INDEX ON days;
```

We'll want to create the same default indexes on our other views: `hours`, `minutes`, and `seconds`.
Importantly, we want to create them in this order, also, to make sure that each relies on the one before it.
If they did not, we would be back in the world of the previous section, where each would read ahead until the end of time (the year 2099, in this example).

#### Finishing touches

As a final bit of housekeeping, we'll want to go from intervals back to moments, with some additional inequalities.
```sql
-- The final view we'll want to use.
CREATE VIEW moments AS
SELECT second AS moment FROM seconds
WHERE mz_now() >= second
  AND mz_now() < second + '1 day';
```
The only change here is the `mz_now()` inequality, which now avoids `BETWEEN` because it has inclusive upper bounds.
The result is now a view that always has exactly 24 * 60 * 60 = 86400 elements in it.
We can verify this by subscribing to the changelog of the count query:
```sql
-- Determine the count and monitor its changes.
COPY (
    SUBSCRIBE (SELECT COUNT(*) FROM moments) 
    WITH (progress = true)
)
TO stdout;
```
This reports an initial value of 86400, and then repeatedly reports (second by second) that there are no additional changes.
```
materialize=> COPY (
    SUBSCRIBE (SELECT COUNT(*) FROM moments) 
    WITH (progress = true)
)
TO stdout;
1716210913609	t	\N	\N
1716210913609	f	1	86400
1716210914250	t	\N	\N
1716210914264	t	\N	\N
1716210914685	t	\N	\N
1716210915000	t	\N	\N
1716210915684	t	\N	\N
1716210916000	t	\N	\N
1716210916248	t	\N	\N
1716210916288	t	\N	\N
1716210916330	t	\N	\N
1716210916683	t	\N	\N
^CCancel request sent
ERROR:  canceling statement due to user request
materialize=> 
```
All rows with a second column of `t` are "progress" statements rather than data updates.
The second row, the only one with a `f`, confirms a single record (`1`) with a value of `86400`.

Yeah, that's it! The only thing left is to read a wall of text containing all the SQL.
Actually, I recommend bouncing up to the start of the post again, and confirming that the pieces fit together for you.
It's also a fine time to [try out Materialize](https://materialize.com/register/), the only system that can run all of these views. 

### Appendix: All the SQL

```sql
CREATE TABLE empty (e TIMESTAMP);

-- Supporting view to translate ids into text.
CREATE VIEW items (id, item) AS VALUES
    (0, 'Signed Memorabilia'),
    (1, 'City Bar Crawl'),
    (2, 'Best Pizza in Town'),
    (3, 'Gift Basket'),
    (4, 'Custom Art');

-- Each year-long interval of interest
CREATE VIEW years AS
SELECT * 
FROM generate_series(
    '1970-01-01 00:00:00+00', 
    '2099-01-01 00:00:00+00', 
    '1 year') year
WHERE mz_now() BETWEEN year AND year + '1 year' + '1 day';

-- Each day-long interval of interest
CREATE VIEW days AS
SELECT * FROM (
    SELECT generate_series(year, year + '1 year' - '1 day'::interval, '1 day') as day
    FROM years
    UNION ALL SELECT * FROM empty
)
WHERE mz_now() BETWEEN day AND day + '1 day' + '1 day';

-- Each hour-long interval of interest
CREATE VIEW hours AS
SELECT * FROM (
    SELECT generate_series(day, day + '1 day' - '1 hour'::interval, '1 hour') as hour
    FROM days
    UNION ALL SELECT * FROM empty
)
WHERE mz_now() BETWEEN hour AND hour + '1 hour' + '1 day';

-- Each minute-long interval of interest
CREATE VIEW minutes AS
SELECT * FROM (
    SELECT generate_series(hour, hour + '1 hour' - '1 minute'::interval, '1 minute') AS minute
    FROM hours
    UNION ALL SELECT * FROM empty
)
WHERE mz_now() BETWEEN minute AND minute + '1 minute' + '1 day';

-- Any second-long interval of interest
CREATE VIEW seconds AS
SELECT * FROM (
    SELECT generate_series(minute, minute + '1 minute' - '1 second'::interval, '1 second') as second
    FROM minutes
    UNION ALL SELECT * FROM empty
)
WHERE mz_now() BETWEEN second AND second + '1 second' + '1 day';

-- Indexes are important to ensure we expand intervals carefully.
CREATE DEFAULT INDEX ON years;
CREATE DEFAULT INDEX ON days;
CREATE DEFAULT INDEX ON hours;
CREATE DEFAULT INDEX ON minutes;
CREATE DEFAULT INDEX ON seconds;

-- The final view we'll want to use .
CREATE VIEW moments AS
SELECT second AS moment FROM seconds
WHERE mz_now() >= second
  AND mz_now() < second + '1 day';

-- Extract pseudorandom bytes from each moment.
CREATE VIEW random AS
SELECT moment, digest(moment::text, 'md5') as random
FROM moments;

-- Present as auction 
CREATE VIEW auctions_core AS
SELECT 
    moment,
    random,
    get_byte(random, 0) + 
    get_byte(random, 1) * 256 + 
    get_byte(random, 2) * 65536 as id,
    get_byte(random, 3) +
    get_byte(random, 4) * 256 as seller,
    get_byte(random, 5) as item,
    -- Have each auction expire after up to 256 minutes.
    moment + (get_byte(random, 6)::text || ' minutes')::interval as end_time
FROM random;

-- Refine and materialize auction data.
CREATE MATERIALIZED VIEW auctions AS
SELECT auctions_core.id, seller, items.item, end_time
FROM auctions_core, items
WHERE auctions_core.item % 5 = items.id;

-- Create and materialize bid data.
CREATE MATERIALIZED VIEW bids AS
-- Establish per-bid records and randomness.
WITH prework AS (
    SELECT 
        id AS auction_id,
        moment as auction_start,
        end_time as auction_end,
        digest(random::text || generate_series(1, get_byte(random, 5))::text, 'md5') as random
    FROM auctions_core
)
SELECT 
    get_byte(random, 0) + 
    get_byte(random, 1) * 256 + 
    get_byte(random, 2) * 65536 as id, 
    get_byte(random, 3) +
    get_byte(random, 4) * 256 AS buyer,
    auction_id,
    get_byte(random, 5)::numeric AS amount,
    auction_start + (get_byte(random, 6)::text || ' minutes')::interval as bid_time
FROM prework;
```
<!-- ##{"timestamp":1716094800}## -->

