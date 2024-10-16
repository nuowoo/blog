
Let's take a look at a fundamental problem in economics, with applications to doing business: matching up producers and consumers of some abstract resource, in a way that appeals to all of the participants.

Imagine we have a set of producers and a set of consumers, each of whom wants to be matched to one member of the opposite type, and each of them have some (not neccesarily shared) preference for the other.
The problem was initially presented in the language of "stable marriage", but it applies to any pairings where the participants have opinions about those they might be paired with.
The framing has also been applied to matching hospital residents with hospitals, application clients with server capacity, and in this post hungry engineers and their lunching options.
You should be able to apply it to a variety of settings, most fruitfully when the matched things come with a rich variety of opinions about each other.

To spill the beans, there already is an algorithm for [stable matching](https://en.wikipedia.org/wiki/Stable_marriage_problem), and we're just going to implement it in recursive SQL.
You might not have thought of SQL as a language for *algorithms*, and conventional SQL is certainly very limited in this respect.
However, recursive SQL can be a great fit, and when it is there's no reason not to just lean on the existing approaches!

### Stable Matching in SQL

We will work off of a table `prefs` that will store the mutual preferences between pairs of producer and consumer.
Not every pair needs to be represented here, and any pairs that are absent will just be taken to be non-viable.
We'll call producers and consumers by `name1` and `name2`, respectively, which aren't very evocative but are easier to type.
Each pair will have integer preferences `pref1` and `pref2` for each other, where smaller numbers mean higher preference (imaging them as a ranking).

```sql
-- Each entry indicates a potential connection between `name1` and `name2`.
-- Each has a numerical preference for this, where we'll take smaller to be better.
-- The goal is to match up `(name1, name2)` pairs where each prefers the other over
-- any other "stable" pairing (someone else who likes them back enough not to leave).
CREATE TABLE prefs(name1 TEXT, pref1 INT, name2 TEXT, pref2 INT);
```

Our goal is to pull out a subset of `prefs` where each `name1` and `name2` occur at most once.
Also, we shouldn't leave behind any pairing in which each prefers the other more than the pair they were assigned.
That second part is where the algorithm comes in.

Of course, we'll want some example preferences to work with.
Let's start with some hungry engineers and food options.
Thematically, let's imagine that each human prefers the foods based on their own unaccountable tastes, and the food options (restaurants) prefer the humans based on their distance (because each's price doesn't vary as a function of the human, but the delivery cost does).

Here's some made up data that will show off what we are trying to do.

```sql
-- Imagine people have a preference for foods that idk is based on its price.
-- Imagine restaurants have a preference for people based on their distance.
INSERT INTO prefs VALUES
('frank',  4, 'ramen', 1),  -- frank needs food, and ramen likes him best
('arjun',  1, 'ramen', 3),  -- arjun lovel ramen, but it is unrequited.
('arjun',  3, 'sushi', 4),  -- arjun can tolerate sushi; they prefer him to nikhil.
('nikhil', 1, 'sushi', 5);  -- nikhil is too far away to safely enjoy sushi.
```

If we study the data (and the comments) we will find that one stable matching is 
```
 name1 | pref1 | name2 | pref2 
-------+-------+-------+-------
 arjun |     3 | sushi |     4
 frank |     4 | ramen |     1
(2 rows)
```
Nikhil doesn't get lunch in this story, which is too bad, but is a demonstration of the constraints: not everyone gets what they want.
Arjun also doesn't get what he wants, which is ramen, because it isn't stable: the ramen-ya would just hit Frank up and they'd do lunch instead.
It turns out there aren't other stable matchings for this data, but in general there can be many.

How do we arrive at a stable matching?
Fortunately, way back in 1962, [Gale and Shapley proposed](https://web.archive.org/web/20170925172517/http://www.dtic.mil/get-tr-doc/pdf?AD=AD0251958) an algorithm to do just that.
In one variant: each producer proposes to satisfy their favorite consumer, each consumer definitively rejects all but the best proposal, and spurned proposers repeat with their next best options, until the rejections stop or they run out of options.

It's pretty much recursion, isn't it? 
And moreover, each of the steps are pretty easy SQL.
Let's write them down!

```sql
-- Iteratively develop proposals and rejections.
WITH MUTUALLY RECURSIVE
    -- Pairings that have yet not been explicitly rejected.
    active(name1 TEXT, pref1 INT, name2 TEXT, pref2 INT) AS (
        SELECT * FROM prefs
        EXCEPT ALL
        SELECT * FROM rejects
    ),
    -- Each `name1` proposes to its favorite-est `name2`.
    proposals(name1 TEXT, pref1 INT, name2 TEXT, pref2 INT) AS (
        SELECT DISTINCT ON (name1) *
        FROM active
        ORDER BY name1, pref1, name2, pref2
    ),
    -- Each `name2` tentatively accepts the proposal from its favorite-est `name1`
    tentative(name1 TEXT, pref1 INT, name2 TEXT, pref2 INT) AS (
        SELECT DISTINCT ON (name2) *
        FROM proposals
        ORDER BY name2, pref2, name1, pref1
    ),
    -- Proposals that are not accepted become definitively rejected.
    rejects(name1 TEXT, pref1 INT, name2 TEXT, pref2 INT) AS (
        SELECT * FROM rejects
        UNION ALL
        SELECT * FROM proposals
        EXCEPT ALL
        SELECT * FROM tentative
    )
-- The tentative accepts become real accepts!
SELECT * FROM tentative
```

Each of these steps--proposal, tentative acceptance, and rejection--follow the written description up above.
The behavior of the `WITH MUTUALLY RECURSIVE` block is to evaluate each term in order, then repeat from the top, until they stop changing.
It's worth a moment reading and maybe re-reading the SQL to convince yourself that there is at least some relationship to the written plan.

If we run the query, we get the result up above.
```
 name1 | pref1 | name2 | pref2 
-------+-------+-------+-------
 arjun |     3 | sushi |     4
 frank |     4 | ramen |     1
(2 rows)
```

These results are great to see, but we are here to *maintain* computation, as input data change.
We can also [`SUBSCRIBE`](https://materialize.com/docs/sql/subscribe/) to the query, and then modify the input to see some output changes.

Each subscribe starts with a snapshot, and it should be (and is) the answer just up above.
```
1702997600437	 1	arjun	3	sushi	4
1702997600437	 1	frank	4	ramen	1
```
To remind you, or introduce you, `SUBSCRIBE` produces output whose first column is the timestamp of some update event, followed by a change in count (here `1` for both records), followed by payload columns matching what you'd see from a `SELECT` query.

At this point, let's introduce the possibility that Frank would happily eat a sandwich instead of ramen.
```
materialize=> insert into prefs values ('frank', 2, 'sando', 3);
```
As soon as I press enter, a bunch of changes spill out of the subscription:
```
1702997625810	 1	arjun	1	ramen	3
1702997625810	-1	arjun	3	sushi	4
1702997625810	 1	frank	2	sando	3
1702997625810	-1	frank	4	ramen	1
1702997625810	 1	nikhil	1	sushi	5
```
How do we read this? 
Arjun has a shuffle where he gains a matching with ramen and yields his sushi seat.
Frank switches to a sandwich from ramen.
And Nikhil gets lunch! 
Sushi isn't happy about it, mind you, but lunch occurs for all producers and consumers.

Importantly, there is one timestamp (`1702997625810`), indicating that all five changes happen atomically, at exactly the same moment.
Neither producer nor consumer will be over-committed, even for a moment, on account of Materialize doesn't screw around with consistency and correctness.

### Generalizing Stable Matching

Let's imagine that each restaurant can serve more than one person, and instead has an integer "capacity".
What do we need to change about our process?
Let's introduce tables `producer_capacity` and `consumer_capacity`, which each hold a name and an integer capacity.

```sql
-- Each producer and consumer have an integer number of matches they can participate in.
CREATE TABLE producer_capacity(name TEXT, cap INT);
CREATE TABLE consumer_capacity(name TEXT, cap INT);
```

What we need to tweak about the algorithm is that each producer proposes at their top `cap` opportunities, and each consumer tentatively accepts their top `cap` proposals.

Where above we have fragments that look like so, to pick the top singular opportunity,
```sql
    -- Each `name1` "proposes" to its favorite-est `name2`.
    proposals(name1 TEXT, pref1 INT, name2 TEXT, pref2 INT) AS (
        SELECT DISTINCT ON (name1) *
        FROM active
        ORDER BY name1, pref1, name2, pref2
    ),
```
we'll want to update these to pick the top `cap` opportunities:
```sql
    -- Each `name1` "proposes" to its `cap` favorite-est `name2`.
    proposals(name1 TEXT, pref1 INT, name2 TEXT, pref2 INT) AS (
        SELECT lat.* FROM producer_capacity, 
        LATERAL (
            -- pick out the best `cap` opportunities
            SELECT * FROM active
            WHERE active.name1 = producer_capacity.name
            ORDER BY active.pref1
            LIMIT producer_capacity.cap
        ) lat
    ),
```
This new SQL is a bit more complicated than the old SQL, but the `LATERAL` join allows us to invoke `LIMIT` with an argument that depends on `cap` rather than a limit of exactly one that `DISTINCT ON` provides.

We'll need to do the same thing for our tentative accepts, using `consumer_capacity`.
```sql
    -- Each `name2` tentatively "accepts" the proposal from its favorite-est `name1`
    tentative(name1 TEXT, pref1 INT, name2 TEXT, pref2 INT) AS (
        SELECT lat.* FROM consumer_capacity, 
        LATERAL (
            -- pick out the best `cap` proposals
            SELECT * FROM proposals
            WHERE proposals.name2 = consumer_capacity.name
            ORDER BY proposals.pref2
            LIMIT consumer_capacity.cap
        ) lat
    ),
```

With unit capacities we'll see the same results as before. 
However, let's introduce Nikhil to ramen, which it turns out he likes.
```
materialize=> insert into prefs values ('nikhil', 1, 'ramen', 2);
```
This has some immediate consequences for our subscription to the matching.
I restarted it because we need to pick up the new query with capacities, but the new snapshot put us right back where we were before.
```
1703011622743	-1	arjun	1	ramen	3
1703011622743	 1	arjun	3	sushi	4
1703011622743	 1	nikhil	1	ramen	2
1703011622743	-1	nikhil	1	sushi	5
```
This dislodges Arjun, who is now back on the sushi plan, because the ramen folks are fully occupied. 
But only because they are occupied.
Let's update their capacity to two, which should give Arjun a seat.
```
materialize=> update consumer_capacity set cap = 2 where name = 'ramen';
```
```
1703011679155	 1	arjun	1	ramen	3
1703011679155	-1	arjun	3	sushi	4
```
And, to rattle things a bit more let's imagine the sandwich shop is sold out and their capacity drops down to zero.
```
materialize=> update consumer_capacity set cap = 0 where name = 'sando';
```
```
1703011883207	-1	arjun	1	ramen	3
1703011883207	 1	arjun	3	sushi	4
1703011883207	-1	frank	2	sando	3
1703011883207	 1	frank	4	ramen	1
```
Poor Arjun is just getting bounced around. 
He decides he really wants some ramen, and offers a cash incentive which updates their preference for him dramatically. 
We'll model this by just tweaking their preference directly.

```
materialize=> update prefs set pref2 = 1 where name1 = 'arjun' and name2 = 'ramen';
```
```
1703012011622	 1	arjun	1	ramen	1
1703012011622	-1	arjun	3	sushi	4
1703012011622	-1	nikhil	1	ramen	2
1703012011622	 1	nikhil	1	sushi	5
```
And Arjun is back on ramen and Nikhil is back on sushi.

### Recursive SQL and Doing Business

There are lots of changes the input may experience, many of which lead to changed output.
Like in life, the world changes around you and you may need to promptly update your plans for the world.
Materialize and recursive SQL are here to make sure you are always looking at the correct output, moment by moment.

We've seen an example of using SQL for one problem that is fundamental in economics: stable matching (with capacities).
This certainly isn't the only problem in economics, nor even the most significant business problem you'll have, but it does show off a potentially new use of recursive SQL to solve the problem.
Other problems, similar and different, have natural solutions with recursive SQL that you might not have imagined, and you wouldn't be able to access with vanilla SQL.

<!-- ##{"timestamp":1702962000}## -->