

### To list rows, where any field contains the `search query` as a substring.

- We needed substring search across a huge transactions table. For context, one of our client's DB has nearly 80M rows in a single month's partition. This write-up walks through the thought process behind the implementation, the approaches we considered, and the tradeoffs between storage, query speed, and efficiency.


My experience with SQL was mostly around DML, from the application layer with huge abstractions of ORM. I had hardly peeled a single layer below `SELECT .. FROM .. WHERE`.
My weapon of choice initially was naïve `LIKE`, which worked just fine with my 10-row synthetically seeded `Orders` table for my fifth semester e-commerce project.

*Side note: Ended up with `LIKE` anyways.*

Initially, the table had an extra column `searchable` of type `tsvector` which was used to query  against the searched tokens with `tsquery`. A `tsvector` value is a normalized, storage efficient, collection of `lexemes`, optimized for text search in PostgreSQL.

Example:
```
SELECT to_tsvector(
  'english',
  'Chat, We cooked? Or Are we being cooked? AI is cooking? What is happening chat?'

);

----------OUTPUT-------------
'ai':9 'chat':1,15 'cook':3,8,11 'happen':14
```

Here's what happened:
- Deduplication: `chat`, `cook` was only stored once
- Lexemes were normalized to merge different variants of the same word: `cooked` and `cooking` in this case.
- Stop words like `Or`, `Are`, `Is` were removed.

`tsvector` operator is a hugely celebrated feature for text search because with proper indexing, searching even in huge documents becomes sufficiently fast. If your users frequently look for the token 'shake' in Shakespeare's novels, then wohoo, there you go, but our users searched with substrings like `05067` in fields that looked like `PSW UPI-7175926-050674899-CTZNNPKA K94210543`

here's what `tsvector` provides for cases like this

```
SELECT to_tsvector(
  'english',
  'PSW UPI-7175926-050674899-CTZNNPKA K94210543'
);
---
'-050674899':4 '-7175926':3 'ctznnpka':5 'psw':1 'k94210543':6 'upi':2
```

To get any relevant results with `tsquery` you had to search with a block of code (like `k94210543`). But, a `lexeme` lost its meaning when used with machine generated identifier like texts. This approach even with the smartest tokenization, was quickly deemed unhelpful and had to be refactored for substring matching. Oh !! `LIKE`, I am coming for you.

Before demeaning the `LIKE`, which now is a good friend of mine, let's understand its implementation. If you're a C whiz-kid, [here's the relevant source code.](https://github.com/postgres/postgres/blob/master/src/backend/utils/adt/like_match.c)
### How does `LIKE` work?
Conceptually, PostgreSQL’s `LIKE` implementation scans the input string and pattern using separate pointers. Literal characters are compared directly, `_` advances by one character, and `%` triggers scanning and recursive attempts against the remaining pattern.  My grandpa walked faster than this.

With my then knowledge, I would have written something like,

`.... WHERE description like '%search_query%' `

So simple, woof, am I a genius? Dunning Kruger can whoop my ass, I assumed my seniors never thought of it.

## Stumbling upon `pg_trgm` (The sauce)

`LIKE` was never the problem, the bottleneck was what came in and around it, quite literally, the *INPUTS*. With our previous query, the inputs were strings that had to be matched with two pointers approach, but what if we change the input as well as what it is matched against as well?

Enter trigram, a group of three consecutive characters taken from a string. Here's an example:
```
select show_trgm('Chat, We cooked? Or Are we being cooked? AI is cooking? What is happending chat?');

---OUTPUT---

{"  a","  b","  c","  h","  i","  o","  w"," ai"," ar"," be"," ch"," co"," ha"," is"," or"," we"," wh","ai ",app,are,"at ",bei,cha,coo,din,"ed ",ein,end,hap,hat,ing,"is ",ked,kin,ndi,"ng ",oke,oki,ook,"or ",pen,ppe,"re ","we ",wha}
```


As evident, with faster searching comes *nx* storage requirement.

Guess, which operator trigrams support for text searching? No, don't skip ahead, take a moment, wait, take one more second, ummmmm, yup!! the old, cute `LIKE` operator, a good friend of mine btw. But, this time it came with trigram around it. Hold on, my friend is not done yet, the comeback is stronger than the setbacks.

Now, I had a presentable strategy for the refactor, use `pg_trgm` with `GiST` index. Not much, but honest work. Anyways, with my trust-me-bro benchmark, I concluded 3GB of index for 15M rows to get substring matching in reasonable time was a no-brainer. So, I started bringing the refactor in the application layer.

## The Climax (Throwing away the sauce)

To get to this point, I had read enough PostgreSQL documentation and relevant articles to understand the character of indexing. So, while bringing my refactor to the application layer, I (with great help of my senior) realized that all the queries to the table had filters that drastically decreased the scope of the search. For example, a transaction is always scoped to the source, type of it, so naturally only transactions from that scope was a relevant result for the user.

If you have tried devising an indexing strategy ever, you know this is a perfect opportunity for a B-Tree composite index to shine.  My trust-me-bro benchmarks showed a 10x faster query results, with our newest, super-affordable index, 500MB in the table where trigram index was consuming 10GB. Now, coming back to our search, something weird started happening that I couldn't get my head around, the planner stopped using our trigram index.

I grew up in a household where shampoo bottles were refilled with water, so my heart shed a tear when my 10GB per partition row index was deemed useless by the planner. Ouch!! still hurts.

The query planner, brought me back to square one, because it applied `B-tree` index scan and used `LIKE '%query%'` predicate as a filter over those candidate rows instead of using the trigram index. It did so because the estimated cost to
- first, scan the index,
- then, combine on other predicates to filter,
- after that, fetch those rows
surpassed the benefit of faster lookup.
In other words, the `B-tree` index didn't help for substring search but it could cheaply find the small set of rows matching other fields. After that, applying `LIKE` as a filter was cheap enough.

A month passed, no complaints, no praises too but that's okay, until one day, I push through office door, a gust of wind whips my jacket. I narrow my eyes, start steps towards my seat, until a wave of breeze(internet) brings an envelope signed to Shashwat (a Slack message) steals my attention. A complaint, `the search has been performing poorly obstructing daily work.`

The plan was perfect, the stage was set for my engineering to shine, until another `EXPLAIN ANALYZE` query showed me my true colors. My trigram index didn't just come with storage constraints, it blew up the entire search function.

Here's what happened, the PostgreSQL planner, which was actively avoiding the trigram index altogether was now deciding to load both indexes, and perform bitmap `AND` operation. For debugging purpose, I dropped the trigram index altogether and ran the search query. The performance boost was, urmmm, so much better, how much better? slightly, but by how much? **65x better**. Yes, same query, when powered with a trigram index had execution time of 1.5 MINUTES, was dropped to 1.5 seconds with NO index.

Now, even without the trigram index, my `LIKE`, best friend btw, backed search runs faster than my grandpa who is on a wheelchair with two rocket boosters, all because it has a strong `B-Tree` companion which does all the heavy lifting.

Below, I have dropped some stats for reference, If you believe, the strategy had obvious loopholes, or have suggestions for me. hmu at [LinkedIn](https://www.linkedin.com/in/shashwat-poudel/).

```
select
  indexname as index_name,
  pg_size_pretty(pg_relation_size(indexrelid)) as index_size
from pg_stat_user_indexes
where relname = 'transactions_YYYY_MM';

                      index_name                       | index_size
-------------------------------------------------------+------------
 transactions_YYYY_MM_pkey                            | 2591 MB
 transactions_YYYY_MM_scope_idx                       | 580 MB
 transactions_YYYY_MM_searchable_idx                  | 10 GB
 transactions_YYYY_MM_field_two_idx                   | 117 MB
 transactions_YYYY_MM_field_one_idx                   | 125 MB
 transactions_YYYY_MM_field_three_idx                 | 125 MB
(6 rows)
```


## With Trigram Index


```
EXPLAIN ANALYZE
SELECT *
FROM transactions
WHERE txn_date = 'YYYY-MM-DD'
  AND source_id = 'SOURCE_NAME'
  AND data_type_id = 1
  AND description ILIKE '%SEARCH_QUERY%';

QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------
Bitmap Heap Scan on transactions_YYYY_MM transactions
  (cost=2952.55..2976.07 rows=21 width=838)
  (actual time=103132.213..103132.216 rows=1 loops=1)

  Recheck Cond:
    (
      (description ~~* '%SEARCH_QUERY%'::text)
      AND (data_type_id = 1)
      AND ((source_id)::text = 'SOURCE_NAME'::text)
      AND (txn_date = 'YYYY-MM-DD'::date)
    )

  Heap Blocks: exact=1

  -> BitmapAnd
       (cost=2952.55..2952.55 rows=21 width=0)
       (actual time=103132.179..103132.181 rows=0 loops=1)

       -> Bitmap Index Scan on transactions_YYYY_MM_searchable_idx
            (cost=0.00..150.65 rows=5951 width=0)
            (actual time=103095.044..103095.044 rows=4 loops=1)

            Index Cond:
              (description ~~* '%SEARCH_QUERY%'::text)

       -> Bitmap Index Scan on transactions_YYYY_MM_scope_idx
            (cost=0.00..2801.64 rows=208070 width=0)
            (actual time=36.741..36.741 rows=784149 loops=1)

            Index Cond:
              (
                (data_type_id = 1)
                AND ((source_id)::text = 'SOURCE_NAME'::text)
                AND (txn_date = 'YYYY-MM-DD'::date)
              )

Planning Time: 0.444 ms
Execution Time: 103132.762 ms
```




#### Dropping the Trigram index.
```
BEGIN;

DROP INDEX ix_transactions_searchable_gist_trgm;

EXPLAIN ANALYZE
SELECT *
FROM transactions
WHERE txn_date = 'YYYY-MM-DD'
  AND source_id = 'SOURCE_NAME'
  AND data_type_id = 1
  AND description ILIKE '%SEARCH_QUERY%';

QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------
Index Scan using transactions_YYYY_MM_scope_idx on transactions_YYYY_MM transactions
  (cost=0.56..223266.20 rows=21 width=838)
  (actual time=326.674..1579.087 rows=1 loops=1)

  Index Cond:
    (
      (data_type_id = 1)
      AND ((source_id)::text = 'SOURCE_NAME'::text)
      AND (txn_date = 'YYYY-MM-DD'::date)
    )

  Filter:
    (description ~~* '%SEARCH_QUERY%'::text)

  Rows Removed by Filter: 784148

Planning Time: 2.501 ms
Execution Time: 1579.112 ms
```

- The scoped B-tree index returned around **784,149 candidate rows**.
- The trigram index returned only **4 candidate rows**, but the trigram index scan itself took about **103 seconds**.
- PostgreSQL combined both indexes using `BitmapAnd`.
- Dropping the trigram index forced PostgreSQL into the simpler B-tree index scan.
- Filtering 784,148 rejected rows still took only about **1.58 seconds**, which was cheaper than touching the trigram index.


/Happy Engineering!!
