# Week 12: More Indexes, Full Text Search, JSON Search

**Announcements:**

1. The last lab:
    1. Lots of bad grades because:
        1. The indexes you provided would not speed up the queries.
            (I gave 0/4 for this error.)
        1. You didn't use the EXPLAIN command to verify that your queries actually use the index.
    1. My solution used `reverse` + `lower` + `LIKE` + `text_pattern_ops`
    1. If you used `pg_tgrm`+GIN index:
        1. You answered the question as asked, so I gave full credit
        1. But it wasn't the intended solution
        1. It's not as fast as the btree solution
        1. My guess is that most of you who used this solution just randomly copied stuff from ChatGPT
        1. Expect to explain to me how this works in your final exam

1. No lab this week

    1. Due to datascience capstone presentations
    1. A simple assignment posted to github issues: <https://github.com/mikeizbicki/cmc-csci143>

1. Lots of good github issues questions; I believe everything was resolved in office hours that is still open, but if not, then let me know.

1. Thursday will be a final review session.

    Today we will finish going over material.

1. Non-graduating students can sign up for final exam timeslots: <https://docs.google.com/spreadsheets/d/1axeCxjA8i8koFtmPWlzSbos108D8XAtXbGEkiU63E30/edit#gid=0>

## Lecture Notes

<img src=index-all-the-things-meme.jpg width=400px />

**Overview:**

Required readings:
1. ~~Hash index: <https://habr.com/ru/company/postgrespro/blog/442776/>~~
1. GIN index: <https://habr.com/ru/company/postgrespro/blog/448746/>
1. RUM index: <https://habr.com/ru/company/postgrespro/blog/452116/>
1. ~~GIST Index: <https://habr.com/ru/company/postgrespro/blog/444742/>~~
1. ~~SP-GIST Index: <https://habr.com/en/companies/postgrespro/articles/446624/>~~
1. ~~BRIN Index: <https://habr.com/en/companies/postgrespro/articles/452900/>~~

For your homework, you have to use GIN/RUM indexes to speed up:
1. full text search
1. JSONB search

<!--
### Hash Index

1. reference: <https://habr.com/ru/company/postgrespro/blog/442776/>

1. btree index (this class) is to balanced binary search tree (data structures)

   as hash index (this class) is to hash tables (data structures)

    1. trees have O(log n) worst case lookup/insert/delete operations
    1. hash tables have O(1) amortized average case lookup/insert/delete operations
    1. Python dictionaries use hash tables for their implementation

1. used for:
    1. very rarely
    1. it can sometimes be marginally faster than a btree for equality search, but it supports much fewer operations, and so btree tends to be more practical
1. limitations:
    1. only support equality search
    1. does not sort results
    1. cannot decrease in size when items deleted
    1. no support for index only scans (only hash values are stored in the index, and so the table must always be checked for hash collisions)
    1. does not support multicolumn indexes
    1. older versions of postgresql have significantly more limitations
1. runtime (index scan and bitmap index scan):
    1. table pages accessed = `O(k)`, same as b-tree
    1. index pages accessed = `O(k/b)`, negligibly less than b-tree `O(log_b n + k/b)`
        1. the buffer cache optimization causes b-tree index pages accessed to be effectively `O(k/b)` instead of `O(k/b + log_b n)`
            1. store the most recently accessed pages in memory
            1. top levels of a btree are accessed frequently, so they'll be in the buffer cache
            1. `shared_buffer_size` controls how many pages will be stored in memory
            1. these pages are shared by all indexes/tables
            1. makes lookups MUCH faster by avoiding disk access,
               makes insert/update/delete MUCH more complicated to implement with ACID guarantees
            1. famous joke:
               > There are 2 hard problems in computer science: cache invalidation, naming things, and off by one errors
            1. there's actually dozens of caches like this that postgres uses, and how caches work is typically covered in operating systems and computer architecture class
        1. buffer cache reference: <https://habr.com/en/company/postgrespro/blog/491730/>
    1. comparison operations = `O(k)` less than b-tree
    1. must consider the runtime of hashing,
       which can be large for large datatypes
1. The syntax for creating a hash index is
   ```
   CREATE INDEX tweet_tags_idx_hash ON tweet_tag USING hash (tag);
   ```
-->

### (Some) Difficulties of Full Text Search

Consider the following query that finds tweets containing the word `corona`:
```
SELECT id_tweets
FROM tweets
WHERE text LIKE '%corona%';
```

1. Recall that the `%` in a `LIKE`/`ILIKE` clause is a wildcard character that matches any string, much like the glob `*` in the shell.

1. Btree indexes cannot speed up this query.

1. If we only had a single wildcard at the end of the query,
   then we could use a btree index.
   For example, the following condition:
   ```
   WHERE text LIKE 'corona%'
   ```
   can be rewritten as
   ```
   WHERE text >= 'corona' AND text < 'coronb'
   ```
   and btree indexes support inequality search efficiently.
   <!--
   A hash index still would not be able to speed up this query because it only supports exact equality search.
   -->

   > **NOTE:**
   > Recall that you need to use the `text_pattern_ops` access method in order to use a LIKE operator,
   > and you use the default access method in order to use the >= and < operators.

1. **QUESTION:**
    Can we use a btree to speed up searches that end with `corona`?

    **ANSWER:** Yes, but you must be careful.

    The where clause
    ```
    WHERE text LIKE '%corona'
    ```
    cannot be speed up with a btree index.

    Instead, you must move the wildcard to the end of the string with the reverse function:
    ```
    WHERE reverse(text) LIKE reverse('%corona')
    ```
    and create the following index
    ```
    CREATE INDEX ON tweets(reverse(text) text_pattern_ops);
    ```

1. **QUESTION:**
   If we replace `LIKE` with `ILIKE` in the above queries,
   how does that affect the performance of a btree index?

   *ANSWER:*
   It's bad.
   Recall that `ILIKE` is a case-insensitive search.
   So
   ```
   WHERE text ILIKE 'corona%'
   ```
   is equivalent to
   ```
   WHERE (text >= 'corona' AND text < 'coronb')
      OR (text >= 'Corona' AND text < 'Coronb')
      OR (text >= 'cOrona' AND text < 'cOronb')
      OR (text >= 'COrona' AND text < 'COronb')
      OR (text >= 'coRona' AND text < 'coRonb')
      OR (text >= 'cORona' AND text < 'cORonb')
      OR (text >= 'CORona' AND text < 'CORonb')
      ...
   ```
   if there are `n` letters in your word, then there are `O(n^2)` different OR clauses in the equivalent query.
   Each of those individually can be sped up with a b-tree,
   but there's still a LOT of queries to try.

   > **IMPORTANT:**
   > Some databases (i.e. SQLite) default to case insensitive search with `LIKE`,
   > but postgres maintains the case (in)sensitivity of (I)LIKE for performance reasons.

1. These "one-sided" wildcard queries are not useful for full text search,
   but they are useful when finding hashtags about coronavirus.
   For example, the following query finds all tweets with hashtags that start with `#corona`,
   and a btree index can be used to speed it up.
   ```
   SELECT id_tweets
   FROM tweet_tags
   WHERE tag LIKE '#corona%'
   ```

### Full Text Search (FTS) outside Postgres

<img src=8s90ho7un6i51.jpg width=400px />

1. There is no SQL standard for FTS

1. Most FTS systems are not SQL-based (i.e. NoSQL)

    1. No ACID transactions 

    1. Examples:

        1. ElasticSearch
            1. Most popular FTS engine
            1. Reasonably powerful FTS, but cannot be combined with SQL

                <img src=elastic-drink.jpg width=300px >

            1. Requires installing a full service with similar complexity of postgres => difficult to configure

               <img src=one-does-not-ao1pfo.jpg width=300px />

            1. Can be embedded in postgres via [zombodb extension](https://github.com/zombodb/zombodb),
               but that's even more complicated
            1. No longer open source
                1. Originally licensed using Apache 2
                1. Elastic (the company) made money by offering their own hosted ElasticSearch solutions ($2 billion market cap)
                1. Amazon started offering ElasticSearch as a service on AWS
                1. In response, Elastic has relicensed the ElasticSearch library as SSPL
                    1. Created by MongoDB
                    1. Doesn't allow 3rd parties to offer paid hosting
                    1. Not an "open source license" officially approved by the Open Source Initiative (OSI)
                1. In response, Amazon has forked ElasticSearch, and will continue development of their forked version and offering it on AWS
                1. See:
                    1. https://opensourceconnections.com/blog/2021/01/15/is-elasticsearch-no-longer-open-source-software/
                    1. https://news.ycombinator.com/item?id=26780848
        1. Lucene / Solr
            1. Similar to Elastic, but came before, and no longer as popular
            1. Internally used by Elastic
        1. Groonga
            1. Embedded FTS engine => FTS library => "easy" to include in other projects 
            1. Good Asian language support
            1. Can be embedded in postgres via [pgroonga extension](https://github.com/pgroonga/pgroonga)
                1. Easy to do
                1. No ACID guarantees on the FTS indexes => crash means you must rebuild the index (hours/days for large datasets)
                1. Doesn't respect some postgres settings about memory/disk usage

    1. Advantages of postgres:
        1. Can use full SQL capabilites (i.e. joins)
        1. Postgres FTS generally considered superior to all other RDBMs FTS (MySQL, SQL Server, Oracle, etc.)

    1. Disadvantages of postgres:
        1. must store tables/indexes separately
            1. use about 2x the disk space... sort of... when using a NoSQL solution, you still have to store your original raw data somewhere
            1. be a bit slower for inserts/update/delete and reads that have to touch table pages
            1. there's active development around removing this restriction, and it'll probably land in postgres 14 or 15
        1. doesn't support as many ranking algorithms for search
            1. again, active development to fix this issue
            1. currently (with pspacy library), postgres supports more languages than any of the NoSQL FTS solutions
        1. recall:
            1. Postgres didn't use to support `JSONB`, and so MongoDB/CassandraDB/other NoSql solutions developed around this need
            1. Postgres implemented the first JSON+SQL engine (and is still by far the best JSON support in an RDBMS, see https://www.youtube.com/watch?v=tF3Lb2BvGpk&list=PLuJmmKtsV1dP8IGGH6Z_sYQKxfDqtoSLj&index=10)
            1. Now, Postgres is "better at being Mongo than Mongo"
                1. fully supports JSON
                1. it won't lose your data
                1. it can be much faster due to good indexing
                1. you can turn off ACID guarantees to get even faster
            1. Expect the same thing to happen to FTS NoSql engines over the next few years

    1. Lots of debate about how postgres compares to NoSQL solutions: 
        1. Reference:
            1. FTS in Postgres is good enough: http://rachbelaid.com/postgres-full-text-search-is-good-enough/
            1. discussion thread: https://news.ycombinator.com/item?id=12621950
        1. No one would think it's weird if you used any of these dedicated FTS engines for a project
        1. People who don't know postgres might think it's weird to use postgres, but most people who know postgres would think it's smart, especially if you're already using postgres
        1. We'll (eventually) talk about an extension to postgres called pspacy (that I'm currently developing) that makes postgres FTS even more powerful

### FTS in postgres

1. All of the basic theory we'll discus will apply to all other FTS solutions

1. Two important types:
    1. `tsvector` represents text that can be searched

        ```
        postgres=# SELECT to_tsvector('simple', 'this is a testing example');
                          to_tsvector                  
        -----------------------------------------------
         'a':3 'example':5 'is':2 'testing':4 'this':1
        ```

        If you specify a language, the `to_tsvector` function will perform:
        1. stemming: convert all words into their "root word"
            1. `example`, `examples`, `exemplary` all get converted to `exampl`
        1. stop word removal: remove short, non-semantic words

        This makes search more "accurate"

        ```
        postgres=# SELECT to_tsvector('english', 'this is a testing example');
             to_tsvector     
        ---------------------
         'exampl':5 'test':4
        ```

    1. `tsquery` represents the search query

        ```
        postgres=# SELECT to_tsquery('english', 'this & is & an & example & test');
            to_tsquery
        -------------------
         'exampl' & 'test'
        ```

        ```
        postgres=# SELECT to_tsquery('english', 'example | test');
            to_tsquery
        -------------------
         'exampl' | 'test'
        (1 row)
        ```

    1. The following query does English language text search for tweets containing the string `coronavirus`:
       ```
       SELECT text
       FROM tweets
       WHERE to_tsvector('english', text) @@ to_tsquery('english', 'coronavirus');
       ```
       The `@@` operator should be read as "contains".


    1. Reviewing the documentation may help with the homework assignment

       <img src=a5skfy5y88x11.jpg width=300px />

       References:

       1. <http://rachbelaid.com/postgres-full-text-search-is-good-enough/>

       1. <https://www.postgresql.org/docs/13/datatype-textsearch.html>

       1. <https://www.postgresql.org/docs/13/functions-textsearch.html>

    1. The GIN/RUM indexes can be used to speed up queries that use the `@@` operator, most commonly FTS.
       ```
       CREATE INDEX tweets_idx_fts on tweets USING gin(to_tsvector('english', text));
       ```

### JSONB Operations

Many new operators for querying `JSONB` type.

<img src=json.jpg width=300px />

Important operators/functions:
1. `jsonb_pretty`
1. `->` / `->>`
1. `jsonb_path_query`
1. `jsonb_array_elements`
1. `@@` (this is what can be sped up with an index)

References:
1. Postgres operators: <https://www.postgresql.org/docs/current/functions-json.html>
1. JSONPath/JSON Selector syntax tutorial: <https://dev.to/rezigned/an-intoduction-to-json-selector-1mho>
1. Postgres JSONPath + relation to GIN: <https://habr.com/en/company/postgrespro/blog/500440/>

### Indexes for the `@@` operator

GIN index (Generalized Inverted iNdex)
1. reference: <https://habr.com/ru/company/postgrespro/blog/448746/>
1. used for:
    1. full text search
    1. indexing `JSONB`
    1. indexing arrays
    1. basically, anything that uses the `@@`, `@>`, or `<@` operators
1. limitations:
    1. elements (i.e. lexemes) never deleted
    1. slow to modify (fast mode vs slow mode)
    1. does not store auxiliary information (fixed in rum index)
        1. position of lexemes in the document
        1. timestamp/pagerank of the document
        1. must recheck the table pages
    1. does not support index scan / index only scan;
       only supports bitmap scans;

       this implies that the `LIMIT` operation is not efficient

       `gin_fuzzy_search_limit` is an alternative

       (fixed in RUM index)

    1. does not sort the results (fixed in RUM index)
    1. does not support `CLUSTER`

RUM Index
1. Reference: <https://habr.com/ru/company/postgrespro/blog/452116/>
1. Like the GIN index, but:
    1. also let's you store "metainformation" (e.g. pagerank, timestamp) in the index
    1. results can be returned sorted according to the metainformation
    1. can perform index only scan on the metainformation
        1. speedup from fewer pages accessed
        1. speedup from LIMIT clauses
1. limitations:
    1. slower than the GIN index for insert/update
    1. uses more space than the GIN index (due to metainfo)
    1. not built-in to postgres, and must compile/install a separate extension

<!--
GIST Index
1. Reference: <https://habr.com/ru/company/postgrespro/blog/444742/>
1. We won't cover in detail
1. Faster inserts, slower lookups
    1. For most applications, fast lookups are much more important than fast inserts
    1. So GIST not often used in practice for FTS
-->
