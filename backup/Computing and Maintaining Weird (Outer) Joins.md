[Differential dataflow](https://github.com/TimelyDataflow/differential-dataflow) has a single join operator: `join`.
It takes two input collections, and for each `(key, val1)` and `(key, val2)` in the inputs it produces `(key, (val1, val2))` in the output.
This makes `join` a "binary equijoin", where it fishes out exactly the exact matches on `key`.
This restriction is important, and powerful: when either input experiences a change, the `key` of the change is what directs us to the (other) input records that will help us produce the appropriate output change.
However, there are other "joins" in the larger relational data world, and we need to support them as well.

In this post we'll build up an implementation of a **multi-way outer equijoin**.
We'll start small, but arrive at the best way I know how to build these beasts out of existing parts.
Along the way, we'll 
    get an introduction to how differential dataflow works, 
    develop several ways to use it to implement joins of various stripes, and
    deploy these techniques together to take on the outer-est of (equi-)joins.

Amazingly, to me at least, we end up needing to understand how to efficiently implement multi-way joins of sums of terms.
That is, how to efficiently implement
```
(A0 + A1 + A2) ⋈ (B0 + B1 + B2) ⋈ (C0 + ...) ⋈ ...
```
To be honest, I can't recall this pattern from my database education (such as it was), and I'd love any tips or pointers about where else this shows up.
If you get to the end and it all checks out as old-hat for you, I'd love to know about it!

### Differential Dataflow and the Binary Equijoin `join`

Differential dataflow is a framework for computing and then maintaining functions over continually changing volumes of data.
It manipulates *updates* to data, written as triples `(data, time, diff)` and indicating that at `time` the number of occurrences of `data` changes by `diff`.
Differential dataflow provides primitive operators like `map`, `filter`, `join`, `reduce`, and `iterate`, which users compose to build more complex functions.
Each operator translates input updates into the output updates that would result from continually re-evaluating the operator at every time. 
Similarly, the composed dataflow of operators similarly produces output updates that correspond exactly to continual reevaluation on the changing inputs.

The `join` operator applies to two input collections, for which their `data` have the shape `(key, _)`: pairs of some common "key" type and potentially unrelated "value" types.
The intended output is a tuple `(key, (val1, val2))` for each pair of inputs that have a matching `key`.
The output updates can be derived from first principles, but with enough head-scratching one can conclude that each pair of updates with matching key produces one output update:

```   
    update1: ((key, val1), time1, diff1)     -- First input
    update2: ((key, val2), time2, diff2)     -- Second input
-> 
    ((key, (val1, val2)),  max(time1, time2),  diff1 * diff2)
     \-- output data --/   \----  time ----/   \--  diff --/
```

We can respond to each input update by iterating over the updates in the *other* input with the same key, and use the rule above.
There are smarter ways to do this, consider for example the second update introducing and retracting a record before `time1`: we would produce two outputs that exactly cancel.
In any case, we'll need to retain *some* information about each input, ideally arranged by `key` so that these updates can be efficiently retrieved.

Differential dataflow has a primitive called an "arrangement", which is both a stream of updates and a maintained indexed form of their accumulation.
An arrangement translates a stream of updates into a sequence of indexed "batches" of updates, each of which are indexed by `key`.
It also maintains a collection of these batches that serve as an indexed roll-up of the accumulated updates, using a structure analogous to a [log-structured merge-tree](https://en.wikipedia.org/wiki/Log-structured_merge-tree).
Arrangements are the primary mechanism to maintain "state" as a dataflow runs, and specifically are what `join` uses: each input to `join` must be an arrangement, and if they are not then they will be arranged for you.

### Technique 1: Shared Arrangements

A key advantage to using arrangements is that they can be [*shared*](http://www.vldb.org/pvldb/vol13/p1793-mcsherry.pdf).
Arranged data can be used by any number of dataflows, avoiding the cost of an additional redundant arrangement.
As an example, imagine we have a collection of link data `(source, target)`, and we would like to compute and maintain those identifiers within three steps of some query set `query`.
If the data are arranged, say in an arrangement named `links`, we could write
```rust
// Join `query` against `links` three times, giving
// the identifiers three steps away from each query.
query.map(|query| (query, query))
     .join(links).map(|(step0, (query, step1))| (step1, query))
     .join(links).map(|(step1, (query, step2))| (step2, query))
     .join(links).map(|(step2, (query, step3))| (step3, query))
```
This fragment would naively require six arrangements, two for each `join` invocation.
However, we are able to re-use the `links` arrangement at no cost, and instead only introduce three arrangements, corresponding to the number of steps (0, 1, and 2) out from `query`.
These new arrangements can be substantially smaller than `links`, and the amount of work required to compute and maintain the results can be trivial even when `links` is enormous.

### Technique 2: Functional Joins

This one is a bit of cheat, in that by the end of it you may not be sure it is even a join.

There are times, and we will see them coming up, where we want to join not against *data* but against a *function*.
For example, perhaps we have a collection of `(line, text)` of pairs of integers and strings, and we would like to split each `text` into the words it contains.
One way to do this is with `join`: the first input is our lines of text, and the second input is the quite large collection of pairs `(text, (pos, word))` each indicating a word that can be found in `text`.

Rather than hope to implement this with `join`, because we couldn't hope to maintain the second collection, we could implement this with the `flat_map` operator instead.
```rust
// Convert each `text` into the words it contains.
lines.flat_map(|(line, text)| 
    text.split_whitespace()
        .enumerate()
        .map(|(pos, word)| (line, text.clone(), (pos, word.to_owned())))
)
```

At this point you may be wondering why we have called this a "functional join" rather than a "flat map".
You are not wrong that `flat_map` is the best way to implement this.
However, we will need to prepare ourselves to see this pattern in joins, and understand that it is one way to implement something that may present as a `join`.
Each input record results in zero or many output records, determined by some key fields in the record.

### Technique 3: Multi-way Joins

Even managing a single join can be challenging, but invariably folks actually want to perform multiple joins at once.
Recall our `query` and `links` example, from just up above
```rust
// Join `query` against `links` three times, giving
// the identifiers three steps away from each query.
query.map(|query| (query, query))
     .join(links).map(|(step0, (query, step1))| (step1, query))
     .join(links).map(|(step1, (query, step2))| (step2, query))
     .join(links).map(|(step2, (query, step3))| (step3, query))
```
This performs three joins, and introduces new arrangements for the left inputs of each of the three `join` calls.
We argued that this could be small if `query` is small, and also if each of the intermediate results are small.
But if this isn't the case, then they might be large, and we might end up maintaining quite a lot of information.

Let's take a different example that might not be so easy.
Imagine you start with a collection `facts` of raw data, and you want to enrich it using dimesion tables that translate foreign keys like "user id" into further detail.
The additional detail may result in further keys you want to unpack, like addresses, zipcodes, and the sales agents they map to.
```rust
// Enrich facts with user, address, and sales agent information.
facts.map(|fact| (fact.user_id, fact)).join(users).map( .. )
     .map(|fact| (fact.addr_id, fact)).join(addrs).map( .. )
     .map(|fact| (fact.zipcode, fact)).join(agent).map( .. )
```
Lots and lots of data pipelines have this sort of enrichment in them, in part because "normalized" database best practices are to factor apart this information.
Unfortunately, stitching it back together efficiently is an important part of these best practices.

For this query, we may have arrangements of `users`, `addrs`, `agent`.
However, we are unlikely to have arrangements of the left inputs to each of the `join`s.
The very first left input, `facts` keyed by `user_id`, is plausibly something we might have pre-arranged, but the other two result from the query itself.
Naively implemented, we'll create second and third arrangements of enriched `fact` data, which can be really quite large.

Fortunately, there is a trick for multiway joins that I have no better name for than ["delta joins"](https://github.com/TimelyDataflow/differential-dataflow/tree/master/dogsdogsdogs).
The gist is that rather than plan a multiway join as a sequence (or tree) of binary joins, as done in System R, you describe how the whole join will vary as a function of each input.
You can get this derivation by expanding out our derivation for binary joins, in terms of input updates, to multiple inputs.
You then independently implement each of these response functions for each input as best as you can and then compose their results.

For example, our query above joins four relations: `facts`, `users`, `addrs`, and `agent`, subject to some equality constraints.
When `facts` changes, we need to look up enrichments in `users`, then `addrs`, then `agent` to find the change to enriched facts.
When `agent` changes, we need to find the affected `addrs`, then `users`, then `facts`, in order to update the enrichment of existing facts.

```
-- rules for how to react to an update to each input.
-- elided: equality constraints for each join (⋈).
d_query/d_facts = d_facts ⋈ users ⋈ addrs ⋈ agent
d_query/d_users = d_users ⋈ addrs ⋈ agent ⋈ facts
d_query/d_addrs = d_addrs ⋈ agent ⋈ users ⋈ facts
d_query/d_agent = d_agent ⋈ addrs ⋈ users ⋈ facts
```
The overall changes to `query` result from adding together these update rules.

What's different above is that each of the `d_term ⋈` joins are *ephemeral*: no one needs to remember the `d_` part of the input.
Each of these rules are implementable with what differential dataflow calls a `half_join`: an operator that responds to records in one input by look-ups into a second, and which does not respond to changes to the second input.
The `half_join` operator needs an arrangement of its second input, but not of its first input.

This pattern has different arrangement requirements than the sequence of binary `join` operators.
Each collection needs an arrangement by those attributes by which it may be interrogated.
In the example above, the required arrangements end up being:

1. input `facts` arranged by `user_id`,
2. input `users` arranged by `user_id` and also by `addr_id`,
3. input `addrs` arranged by `addr_id` and also by `zipcode`,
4. input `agent` arranged by `zipcode`.

This ends up being six arrangements, just like before, but they are all arrangements we might reasonably have ahead of time.
The *incremental* arrangement cost of the query can be zero, if these arrangement are all pre-built.

### Boss Battle: Left Outer Joins

An "outer" join is a SQL construct that is much like an standard ("inner") join except that any records that "miss", i.e. do not match any other records, are still produced as output but with `NULL` values in columns we hoped to populate.
Outer joins are helpful in best-effort joins, where you hope to enrich some data, but can't be certain you'll find the enrichment and don't want to lose the input data if you cannot.

For example, consider our `facts`, `users`, `addrs`, and `agent` scenario just above.
What would happen if there is a `user_id` that does not exist in `users`, or a `addr_id` that does not exist in `addrs`, or a `zipcode` that does not exist in `agent`?
Written as a conventional join, we would simply drop such records on the floor and never speak of them.
Sometimes that is the right thing to do, but often you want to see the data along with any *failures* to find the enrichments.

If we take our example from above but use `LEFT JOIN` instead of `join`, we will keep even facts that do not match `users`, `addrs`, or `agent`.
```sql
-- Enrich facts with more data, but don't lose any.
facts LEFT JOIN users ON (facts.user_id = users.id)
      LEFT JOIN addrs ON (users.addr_id = addrs.id)
      LEFT JOIN agent ON (addrs.zipcode = agent.zc)
```

There are also `RIGHT` and `FULL` joins, which respectively go in the other direction (e.g. output users that match no facts, with null fact columns) and in both directions (all bonus records that would be added to a `LEFT` or `RIGHT` join).
We are only going to noodle on `LEFT` joins, though the noodling should generalize just fine.

To implement left joins, we'll need to find a way to express them in terms of the tools we have.
Those tools are .. the operators differential dataflow provides; things like `map`, `filter`, `join`, and `reduce` (no `iterate`. NO!).

### Step one: turn LEFT JOINs into JOINs

When we left join two collections, some records match perfectly as in an inner join, and some do not.
What do we have to add to the results of the inner join to get the correct answer?
Specifically, any keys that might be present in the first input, but are not present in the second input, could just be added to the second input with `NULL` values.

```sql
-- Some facts exactly match some entry in users.
SELECT * FROM facts INNER JOIN users ON (facts.user_id = users.id)
-- Some facts totally miss, but need to match something.
UNION ALL
SELECT facts.*, NULL FROM facts
WHERE facts.user_id NOT IN (SELECT id FROM users)
```
This construction keeps the `INNER JOIN` pristine, but adds in `facts` extended by `NULL`s for any fact whose `user_id` is not found in `users`.
Although not totally clear, `NOT IN` results in a join between `facts` and distinct `users.id`.
This approach feels good, re-uses arrangements on `facts` and `users`, and is pretty close to what Materialize does for you at the moment.

However, this technique is not great for multiway outer joins.
We need access to the left input (here: `facts`) to complete the outer join, and generally that input is the result of the outer join just before this one.
If we need to have that answer to form this query fragment, we don't have a story for how they all become one multiway inner join.
Likewise, Materialize currently plans a multiway outer join as a *sequence* of fragments like above that *involve* inner joins, but are not *an* inner join.

### Step two: Multiway LEFT JOINS into Multiway JOINs

Let's take the intuition above and see if we can preserve the join structure.
We want to produce a SQL fragment that is at its root just an inner join.
We will need to be careful that it should rely on base tables, not its direct inputs (what?).

Let's start and we'll see where we get.

First, let's rewrite the above fragment in a way that looks more like *one* inner join.
One one side we have `facts`, and on the other side .. at least `users` but also some other stuff?
For a first cut, that "other stuff" is .. the `user_id`s in `facts` but not in `users`?
We could add those rows to `users`, with `NULL` values in missing columns, and see what we get!

As it turns out we get totally the wrong answer. 
Best intentions, of course, but the wrong answer.
I believe the right answer is expressed roughly this way, in SQL:

```sql
-- Some facts exactly match some entry in users.
SELECT facts.*, users.* 
FROM facts INNER JOIN users ON (facts.user_id = users.id)
-- Some facts totally miss, but could match something.
UNION ALL
WITH absent(id) AS (
    SELECT user_id FROM facts 
    EXCEPT 
    SELECT id FROM users
)
SELECT facts.*, NULL 
FROM facts INNER JOIN absent ON (facts.user_id = absent.id)
-- Some facts have NULL `user_id` and refuse to be joined.
UNION ALL
SELECT facts.*, NULL
FROM facts WHERE facts.user_id IS NULL
```

We do grab the `absent` keys, but importantly we produce `NULL` in their key columns.
We also need to deal with potentially null `user_id` values, which we do in the third clause, because SQL's `NULL` values do not equal themselves.
Again, best intentions, I'm sure.

The good news is that we have framed the SQL in a way that looks like (taking some notational liberties):
```
  facts ⋈ users
+ facts ⋈ absent    -- with null outputs
+ facts ⋈ NULLs     -- only for null user_id
```
Each of the three joins are slightly different, but they all have the property that `facts` arranged by `user_id` is enough for them.
We can and will now factor out `facts` from these three terms, which puts us in a position to write our multiway left join as:

```
-- Left join of facts, users, addrs, and agent.
facts ⋈ (users + absent(users) + NULL)
      ⋈ (addrs + absent(addrs) + NULL)
      ⋈ (agent + absent(agent) + NULL)
```
This is starting to look a bit more like the joins over sums of terms advertised in the beginning of the post.
For the moment, we are just going to add together the terms, though.

There is quite a lot unsaid here, and the nature of the ⋈ varies a bit for each of the terms in parentheses.
You do have to populate the `absent(foo)` collections with values from base relations, rather than their immediate inputs.
And fortunately, SQL notwithstanding, differential dataflow *does* equate NULL with itself, and everything works out just fine.
Materialize [recently merged](https://github.com/MaterializeInc/materialize/pull/24345) an approach that looks like this for multiway outer joins.
It's early days, but we'll soon start exploring how this work for folks with stacks of left joins.

But the story doesn't end here. 
Somewhat stressfully, this approach takes existing inputs `facts`, `users`, `addrs`, and `agent` and .. fails to use any of their pre-existing arrangements.
It has some other performance issues as well.

### Step three: Rendering JOINs of UNIONs

The last step, or next step at least .. perhaps not the last, is to render these query plans efficiently.
At the moment we have no better plan than to treat the augmented collections as new collections, arrange them, and join them.
Roughly like so:
```
-- Left join of facts, users, addrs, and agent.
with users_aug as (users + absent(users) + NULL)
with addrs_aug as (addrs + absent(users) + NULL)
with agent_aug as (agent + absent(agent) + NULL)
facts ⋈ users_aug ⋈ addrs_aug ⋈ agent_aug
```

These `_aug` collections are as big (somewhat bigger) than their unaugmented counterparts, and it feels somewhat bad to re-arrange them.
It feels bad that despite pre-arranging `users`, `addrs`, and `agent` we can re-use none of them.
It feels bad that all `NULL` values will be routed to a single worker just to find out that they map to `NULL`; lots of work for no surprise.

However, we can get around all of these bad feels with some dataflow shenanigans.
Unfortunately, they are shenanigans that as far as I can tell neither Materialize nor SQL can describe.

We have two strategies for evaluating multiway joins: as a sequence of binary joins, and using delta join rules.
The shenanigans are easier with the sequence of binary joins, so let's start there.
We are going to do something as simple as re-distributing over the `⋈` operator, performing each join the way we want.
We then add up the results of each step of the join rather than adding up the inputs to each step of the join.

```
-- Left join of facts, users, addrs, and agent.
step0 = facts;
step1 = step0 ⋈ users + step0 ⋈ absent(users) + step0 ⋈ NULL;
step2 = step1 ⋈ addrs + step1 ⋈ absent(addrs) + step1 ⋈ NULL;
step3 = step2 ⋈ agent + step2 ⋈ absent(agent) + step2 ⋈ NULL;
step3
```
Each of these ⋈ operators are slightly different. 
The first ⋈ in each row is the traditional equijoin.
The second ⋈ in each row is an equijoin that projects away matched keys and puts `NULL` in their place.
The third ⋈ in each row only matches nulls.

However, in each line we only have to arrange non-null `stepx` and determine and arrange `absent(foo)`. 
We can re-use existing arrangements of `users`, `addrs`, and `agent`.
The join with `NULL` can be implemented as a `flat_map` rather than by co-locating all null records for a `join` (omg finally explained).

In actual fact, we can implement this in both SQL and Materialize, but in doing so we'll lose the multiway join planning benefit of avoiding intermediate arrangements.
We will need to arrange `stepx` for each `x`, and the nice folks with stack of left joins 30+ deep (yes, seriously) will be sitting on 30x as much data as they feel they should.

To recover the benefits, let's grab the delta join construction from way up above. 
I'll use `_aug` suffixes to remind us that it isn't going to be as easy as joining against the pre-arranged collections.
```
-- rules for how to react to an update to each input.
-- elided: equality constraints for each join (⋈).
d_query/d_facts     = d_facts     ⋈ users_aug ⋈ addrs_aug ⋈ agent_aug
d_query/d_users_aug = d_users_aug ⋈ addrs_aug ⋈ agent_aug ⋈ facts
d_query/d_addrs_aug = d_addrs_aug ⋈ agent_aug ⋈ users_aug ⋈ facts
d_query/d_agent_aug = d_agent_aug ⋈ addrs_aug ⋈ users_aug ⋈ facts
```
Ignore for the moment the fact that `d_users_aug` is complicated (an update to `users` may induce the opposite update to `absent(users)`).
Each line up above describes a sequence of `half_join` applications, which like `join` also distributes over `+`.

```
  d_query/d_facts    
= d_facts ⋈ users_aug ⋈ addrs_aug ⋈ agent_aug
= ( 
    d_step0 = d_facts;
    d_step1 = d_step0 ⋈ users + d_step0 ⋈ absent(users) + d_step0 ⋈ NULL;
    d_step2 = d_step1 ⋈ addrs + d_step1 ⋈ absent(addrs) + d_step1 ⋈ NULL;
    d_step3 = d_step2 ⋈ agent + d_step2 ⋈ absent(agent) + d_step2 ⋈ NULL;
    d_step3
)
```
Each time we need to do a `half_join`, we can unpack the right argument to it and conduct the half join as we see fit.
We can either `half_join` with a pre-existing arrangement, `half_join` with a new arrangement of absent values, or `flat_map` some `NULL` values into place.

Writing the whole thing out is exhausting, especially for 30-deep stacks of left joins.
Fortunately this is something computers are good at.
Much like it now seems that they may be good at computing and maintaining deep stacks of left equijoins.

### What's next?

There is an expressivity gap to close between SQL/Materialize and differential dataflow.
I'm not aware of a SQL-to-SQL rewrite that gets us the desired implementation, because we cannot afford to distribute the joins out across the unions, and SQL does not have a `half_join` operator.
We're pondering options now, including expanding our lowest level IR to reflect e.g. half joins, and tweaking the renderer to recognize the idiom of joins between sums of terms.
There will certainly be some amount of measurement as we try and assess the remaining gap, and draw down the amount of time and resources spent on outer joins.

I have a concurrent effort to spread the gospel of [referential integrity](https://en.wikipedia.org/wiki/Referential_integrity) so that we can turn those outer joins to inner joins.
You can understand how in weakly consistent systems you'd need the outer joins to cover for inconsistencies, but do you need it in a strongly consistent system like Materialize?

Of course, if you've read this far you have an obligation to fill me in on what I've missed about all of this.
Is there an easier transform, one that doesn't end up joining terms that are themselves sums of useful constituents?
Do you have an exciting use case for maintaining stacks of outer joins, and you've been burned before?
Do reach out in these cases, and [take Materialize for a spin](https://materialize.com/register/) (though, if you have stacks of 30+ left joins, please reach out for some personal attention).

<!-- ##{"timestamp":1710738000}## -->