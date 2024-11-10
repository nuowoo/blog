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


<!-- ##{"timestamp":1716094800}## -->

