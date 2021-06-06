[select-query]: ../../imgs/seqscan_select_query.png
[select-result]: ../../imgs/seqscan_select_result.png
[select-ctid]: ../../imgs/seqscan_select_ctid.png
[ctid-gif]: ../../imgs/seqscan_ctid.gif
[seqscan-execution-plan]: ../../imgs/seqscan_execution_plan_en-us.png
[license-cc]:../../imgs/license-cc.png

# Explicando PostgreSQL
## 🔍 Sequential Scan (Leitura sequencial) - Lendo o disco em ordem
> hiding in [postgres/src/backend/executor/nodeSeqscan.c](github.com/postgres/postgres/blob/master/src/backend/executor/nodeSeqscan.c)

A query like:

![select-query]

may return a result set  like:

![select-result]

which can get you wondering: what is causing the (sometimes seemingly random) sorting of the results from a SELECT with no _ORDER BY_?

Data from the tables are stored in blocks (or pages) - usually of 8k of data by default - and rows (usually referred to as "tuples") that can be identified by their **CTID**:

![select-ctid]

The CTID is a tuple in the format "(<block number>, <row offset>)" that identifies data inside a page file. Adding the hidden system CTID column  to the query reveals the source of the ordering:

![ctid-gif]

CTIDs may not be contiguous because whenever a row is deleted the tuple slot becomes invalid ( or "dead" ) until a vacuum is run and the space is marked as reusable. The same happens when a row is updated, as the old row is marked as deleted and a new copy of the modified data is written in whatever slot is marked as available.

## What is a Sequential Scan?
The simplest way to read a table from a disk is simply to fetch the rows in the order they are stored. It requires minimum overhead, but is associated with poor performance in a lot of cases where filtering, ordering or grouping is involved.

Example output using "EXPLAIN (VERBOSE, ANALYSE, BUFFERS)":
```
-> Seq Scan on <table_name> (cost=<min_cost>..<max_cost> rows=<estimated_rowcount> width=<row_size>)
  Output: <result_columns>
  Buffers: shared hit=<cached_blocks>
```
## 📋 Fields:
- "Output": The columns returned from the step. 
Requires VERBOSE
- "Buffers": Information about cached data and reading operations.
Requires BUFFERS

## ✅ It's OK to see this step if
- **Your query is trying to read the entire table**. If no sorting or filtering is needed, just returning the rows in the order they're stored in the disk requires basically no overhead.
- **The available indexes won't discard enough rows**. Similar to the previous point, fetching almost every block from the disk by looking at an index that isn't able to filter many rows may cause unnecessary overhead.
- **The table is really small**. Loading a few blocks directly to memory may be faster than reading from a index.

## ❎ It's a problem to see this step when
- **Your query should return a few rows from a large table using a filter**. This may indicate that an important index is not working or simply doesn't exist.
- **The execution plan's row count estimate is really off**. In some cases, Postgres may be completely unaware of how big a table actually is. This is a sign that the table hasn't been vacuumed and analysed properly. 'autovacuum' and 'autoanalyse' settings should also be reviewed system wide or for the specific table. 

## ℹ️ Extra info
- **VACUUM FULL** will cause the table to have contiguous CTIDs because it rewrites the data to the disk. But the original order is kept.
- **CLUSTER** will reorder the table on the disk by rewriting it sorted by a given index.

## Exemple
```sql
SELECT
  country_name
FROM
  country
WHERE
  country_name ILIKE 'b%'
ORDER BY
  country_id;
```

results in the following execution plan:

```
Sort  (cost=2.15..2.21 rows=26 width=16)
  Sort Key: country_id
  ->  Seq Scan on country  (cost=0.00..1.54 rows=26 width=16)
        Filter: ((country_name)::text ~~* 'b%'::text)
```
we can think of this plan as a DAG:

![seqscan-execution-plan]

---

![license-cc]  
This work is licensed under a Creative Commons Attribution-ShareAlike 4.0 International License.