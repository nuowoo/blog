

This past year Team Materialize struck out to do each day of 2023's [Advent of Code](https://adventofcode.com/2023), an annual programming event with thought-provoking problems that you are encouraged to approach from non-standard directions.
We figured we'd try and use SQL for the whole thing.

SQL is a bold choice because it is meant for querying data, and not as much for general computation.
Several of the problems call for interesting algorithms, specific data structures, and some flexibility.
However, Materialize's core thesis is that you can do so much more with SQL that just query your data.
If you want to move operational logic from bespoke code into SQL, you'll need to be able to express that logic.
And so, Advent of Code was a great opportunity to stretch our legs, and fingers, and see just how much logic fits into SQL.

### Preliminaries

There's a lot of content in the month's problems.
There are 49 problems, and although there is some overlap really there is too much to say about all of them.
We aren't going to recount each of the problems, the whimsical backstories, and the shape of the problem inputs.
We'll try and flag some surprising moments, though, and you should dive into those problems if you are keen (they can each be done on their own).

I (Frank) wrote all of my solutions using Materialize's `WITH MUTUALLY RECURSIVE` even when recursion was not required.
This just helped me start writing, as the blocks allow you to just start naming subqueries and writing SQL.

My solutions all had the same skeletal structure:
```sql
WITH MUTUALLY RECURSIVE

    -- Parse the problem input into tabular form.
    lines(line TEXT) AS ( .. ),

    -- SQL leading up to part 1.
    part1(part1 BIGINT) AS ( .. ),

    -- SQL leading up to part 2.
    part2(part2 BIGINT) AS ( .. ) 

SELECT * FROM part1, part2;
```

As mentioned, we won't always need recursion.
However, we often do use recursion, and may even need it.
We'll call this out, as the use (and ease) of recursion in SQL was one of the main unlocks.

### Week one

[Day one](https://github.com/MaterializeInc/advent-of-code-2023/blob/main/week1/aoc_1201.md) was largely about text manipulation, specifically extracting numbers from text, and was well-addressed by using regular expressions to manipulate and search the text. 

[Day two](https://github.com/MaterializeInc/advent-of-code-2023/blob/main/week1/aoc_1202.md) was largely about aggregation: rolling up counts and maxima for games involving numbers of colored cubes; SQL did great here.

[Day three](https://github.com/MaterializeInc/advent-of-code-2023/blob/main/week1/aoc_1203.md) has inputs in grid form, where there can be interaction between multiple lines (with symbols above or below others). 
You are looking for runs of numerals, and I used `WMR` to track these down; reportedly you can also use regular expressions, but I was not clever enough for that!

[Day four](https://github.com/MaterializeInc/advent-of-code-2023/blob/main/week4/aoc_1225.md) introduced scratch cards where each line of input has some winners and losers. 
This was easy SQL until part two, in which winners give you other scratch cards, which have winners that give you other scratch cards, which .. you can see the recursion. 
Despite being wordy and complicated, the SQL isn't so bad:
```sql
    -- PART 2
    -- Each card provides a copy of the next `score` cards.
    expanded(card INT, score BIGINT) AS (
        SELECT * FROM matches
        UNION ALL
        SELECT
            matches.card,
            matches.score
        FROM
            expanded,
            matches,
            generate_series(1, expanded.score) as step
        WHERE
            expanded.card + step = matches.card
    ),
    part2(part2 BIGINT) AS ( SELECT COUNT(*) FROM expanded)
```
This would be tricky to do with non-recursive SQL, as the data itself tells us how to unfold the results.
Hooray for recursion!

[Day five](https://github.com/MaterializeInc/advent-of-code-2023/blob/main/week1/aoc_1205.md) was a bit of a bear.
It was the same day we were doing a Materialize on-site and we were all a bit distracted, but also it was pretty beefy. 
You first have to "route" various elements through a sequence of remappings, whose length is defined in the data.
You then have to expand that out to routing whole intervals (rather than elements), and .. there is just lots of potential for error.
I used recursive SQL to handle all the remapping, but other folks just expanded out their SQL for each of the (ten-ish) remappings.

[Day six](https://github.com/MaterializeInc/advent-of-code-2023/blob/main/week1/aoc_1206.md) was about whether you knew (or were willing to learn about) the [quadratic formula](https://en.wikipedia.org/wiki/Quadratic_formula).

[Day seven](https://github.com/MaterializeInc/advent-of-code-2023/blob/main/week1/aoc_1207.md) is about scoring poker hands, using some new rules for tie breaking. 
This was mostly SQL aggregation, as the numbers of each card in each hand largely determine the outcome, other than tie-breaking where I learned about the [`translate`](https://materialize.com/docs/sql/functions/#translate) function.

### Week two

[Day eight](https://github.com/MaterializeInc/advent-of-code-2023/blob/main/week2/aoc_1208.md) involved some graph navigation (recursion), and some mathematics.
The mathematics were of the form "notice that various things are relatively prime", and it was important to rely on SQL as a tool to support reasoning, as opposed to directly attacking the specified computation.
In this case, my problem called for 14,935,034,899,483 steps, and no tool is going to make direct simulation be the right answer.

[Day nine](https://github.com/MaterializeInc/advent-of-code-2023/blob/main/week2/aoc_1209.md) was a refreshing introduction to polynomials, and how if you take enough derivatives of them they end up at zero.
The task was to do this, repeatedly difference adjacent measurements, or adjacent differences, etc., until you get all zeros.
Then, integrate back up to get projections in the forward and reverse direction.
I used recursion here to accommodate the unknown degree of the polynomial (somewhere in the twenties).

[Day ten](https://github.com/MaterializeInc/advent-of-code-2023/blob/main/week2/aoc_1210.md) presents you with a grid of pipe (symbols `|`, `-`, `J`, `7`, `F`, and `L`), and questions about how long a loop of pipe is, and then how many cells are contained within it. The first part involved recursion, and I used it again for a dynamic programming solution to the second part.

[Day eleven](https://github.com/MaterializeInc/advent-of-code-2023/blob/main/week2/aoc_1211.md) presents a grid of "galaxies" and has you calculate the distance between pairs (the L1 or "Manhattan" distance, always the sum of absolute values of coordinate differences). 
Parts one and two were the same, but with different magnitudes of numbers.
No recursion here!

[Day twelve](https://github.com/MaterializeInc/advent-of-code-2023/blob/main/week2/aoc_1211.md) was about sequence alignment, matching partial observations with hard constraints.
Dynamic programming was a great solution here, using recursion.

[Day thirteen](https://github.com/MaterializeInc/advent-of-code-2023/blob/main/week2/aoc_1213.md) had grids of observations with the hypothesis that each is mirrored, horizontally or vertically, at some point that you need to find.
SQL and subqueries were a great way to validate hypothetical mirroring axes.

[Day fourteen](https://github.com/MaterializeInc/advent-of-code-2023/blob/main/week2/aoc_1214.md) was a treat, in that it used *nested* recursion: a `WMR` block within a `WMR` block.
The problem was simulation of rocks that roll in cardinal directions, changing the direction ninety degrees, and repeating.
Each simulation was recursive (rocks roll until they stop), and we were meant to repeat the larger progress a great many times (1,000,000,000 cycles).
The only bummer here was the amount of copy/paste re-use, as each of the four cardinal directions had different subqueries.

### Week three

[Day fifteen](https://github.com/MaterializeInc/advent-of-code-2023/blob/main/week3/aoc_1215.md) has you implement a hash function, and then a hash map.
Recursion was a handy way to walk through the input to be hashed, though the hash function was simple enough that you could have used math directly instead. 
The second part (the hash map) did not require recursion, as rather than simulate the operations you could leap to the final state you were looking for.

[Day sixteen](https://github.com/MaterializeInc/advent-of-code-2023/blob/main/week3/aoc_1216.md) was about bouncing light around in a grid, and seeing how many grid cells are illuminated.
The illumination process was classic recursive SQL, where you keep expanding `(row, col, dir)` triples until the set reaches a fixed point.
In the second part the light sources had an origin, which is just a fourth column to add, tracking the source of each ray of light.

[Day seventeen](https://github.com/MaterializeInc/advent-of-code-2023/blob/main/week3/aoc_1217.md) is a pathfinding problem, with constraints on how you move around the path (not too short or too long in any direction at once).
Classic recursive SQL to implement [Bellman-Ford](https://en.wikipedia.org/wiki/Bellman–Ford_algorithm).

[Day eighteen](https://github.com/MaterializeInc/advent-of-code-2023/blob/main/week3/aoc_1218.md) provides instructions of how a digger will move around, excavating earth, and asks you to calculate the area.
This is an opportunity to learn about the [Trapezoid formula](https://en.wikipedia.org/wiki/Shoelace_formula#Trapezoid_formula) for computing the area as the addition and subtraction of trapezoid areas.

[Day nineteen](https://github.com/MaterializeInc/advent-of-code-2023/blob/main/week3/aoc_1219.md) sneakily introduces you to [binary space partitioning](https://en.wikipedia.org/wiki/Binary_space_partitioning), where rules based on inequality tests route you to new rules, until eventually you reach some rule that says "accept" or "reject".
This was all pretty easy, except for a substantial amount of SQL overhead related to the various symbols and characters and coordinates all of which required their own columns.

[Day twenty](https://github.com/MaterializeInc/advent-of-code-2023/blob/main/week3/aoc_1220.md) presents you with the simulation of an asynchronous circuit, and this is the day that almost broke me.
Mechanically the SQL isn't that complicated, but *debugging* the SQL was a real challenge.
It got done over the course of a quite long train ride into the evening.

[Day twenty-one](https://github.com/MaterializeInc/advent-of-code-2023/blob/main/week3/aoc_1221.md) was another example of some (recursive) SQL for grid exploration, followed by some mathematics.
In this case the grid exploration was standard, determining reachable locations on the grid, and then the math was quadratic extrapolation from a sequence of measurements (to something too large to actually evaluate, an answer of 621,289,922,886,149 reachable states).

### Week four

The last week was shorter, but also culminated in some pretty exciting problems and techniques.

[Day twenty-two](https://github.com/MaterializeInc/advent-of-code-2023/blob/main/week4/aoc_1222.md) had shapes made of cubes falling into a well, and coming to rest on others (or the ground).
There were then questions about how many pieces are load bearing, and also for each load bearing piece how many others would fall if they were removed.
Dropping the pieces used recursive SQL, determining the load bearing pieces did not, but then scoring the load bearing pieces again required recursion.

[Day twenty-three](https://github.com/MaterializeInc/advent-of-code-2023/blob/main/week4/aoc_1223.md) is a classic example of finding the "longest path" in a directed graph.
This is a relatively easy problem when the input is acyclic (part one), and it is NP-hard when the input may have cycles (part two).
Part one was a mostly vanilla recursive SQL query, and part two encoded the 32 prior state options in a large integer and just did a lot of work.

[Day twenty-four](https://github.com/MaterializeInc/advent-of-code-2023/blob/main/week4/aoc_1224.md) had most folks reach for a numerical solver, something like Mathematica or z3.
That is less easy in SQL, and I needed to learn some math instead (specifically how to find the intersection of two line segments).
Although part two seemed quite complex, it ended up being relatively easy when you realize a few simplifications (an added dimension that can be ignored until the end, allowing you to re-use part one).

[Day twenty-five](https://github.com/MaterializeInc/advent-of-code-2023/blob/main/week4/aoc_1225.md) asked for a minimum graph cut (of three edges).
This is a standard optimization problem, but rather than try to implement the [Stoer-Wagner algorithm](https://en.wikipedia.org/wiki/Stoer–Wagner_algorithm) I went with something from my PhD thesis: partitioning the graph based on the [Fiedler vector](https://en.wikipedia.org/wiki/Algebraic_connectivity#Fiedler_vector).
It turns out this gave the right answer on the first try, and the holidays were saved!

## Conclusions

The exercise was certainly helpful and informative, on multiple levels.

First, it really reinforced for me that `WITH MUTUALLY RECURSIVE` is a very valuable tool to have access to when faced with a new problem.
Often your problem is a bunch of joins and reductions, but when it isn't you are immediately in a bit of a pickle.
In most cases, algorithmic challenges immediately gave way to recursive SQL.

That being said, there's clearly an accessibility gap when reaching for recursive SQL.
I find the idioms approachable, but I've spent a while working with data-parallel algorithms, and have seen several of the tricks.
There's still plenty of work to do before the casual SQL author feels comfortable with recursive SQL.

Second, the majority of my time was spent *debugging* rather than authoring.
This is a classic challenge with declaritive languages, who go from input program to output data in often inscrutable ways.
I borrowed some techniques from [debugging Datalog](https://yanniss.github.io/DeclarativeDebugging.pdf), but ideally the system itself would help me with this (and several research systems do provide integrated lineage).

Debugging the logic of SQL queries only gets harder when the data are changing underneath you.
Techniques like spot checking data become infeasible when the data changes faster than you can observe records that are meant to line up.
Materialize should help in these cases, with maintained diagnostic views that represent assertions, or better violations thereof, whose contents spell out records that at some moment violated something that was meant to be true.
Materialize's `SUBSCRIBE` provides a full account of these views, reporting records that existed even for a moment, where anything other than "always empty" represents an error in your SQL (or your assertions).

Third, using Materialize in new and weird ways shook out several bugs.
We've already fixed them.
Dogfooding your own product, especially in surprising contexts, is a great way to forcibly increase your test coverage.
Issues ranged from the silly ("why would you name a table `count`?") to the abstruse (doubly nested recursive SQL blocks), but they spilled out in the early days and became less frequent as the weeks went on.

Finally, the main conclusion was that it was all possible.
Despite substantial anxiety about whether and when we would need to bail out, defeated, the whole project did work out.
We were able to express a rich variety of computational tasks as data-driven SQL both expressed and maintained by Materialize.
<!-- ##{"timestamp":1704171600}## -->