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


<!-- ##{"timestamp":1716094800}## -->

