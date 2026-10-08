# YouTube Trending Videos – Case Study

A hands-on project to apply database concepts learned along the way, using the Kaggle YouTube Trending Videos dataset on **Supabase (PostgreSQL)**. This README works as a running log: each step is documented as it is completed, to track progress and showcase the skills learned.

## Tech Stack

- **Supabase** (managed PostgreSQL), connected to this GitHub repository
- **Kaggle** (data source)
- **Python** (data ingestion)
- **SQL** (schema design, partitioning, performance analysis)

## Concepts Covered So Far

Distributed databases · OLAP vs OLTP · Partitioning · Join algorithms · Database storage · Normalization · SQL fundamentals · B-tree indexes · Hashing (and its types)

---

## Progress Log

### Step 1 – Ingest the data (Kaggle → Supabase)

Connected Kaggle to Supabase and used a Python script to load the dataset into a single staging table, `staging_videos`.

```python
# Add your ingestion script here (Kaggle download + load into Supabase)
```

### Step 2 – Create a partitioned table

Created a second table, `staging_videos_new`, using **declarative list partitioning on `country`**.

```sql
CREATE TABLE staging_videos_new (
    video_id               TEXT,
    trending_date          TEXT,
    title                  TEXT,
    channel_title          TEXT,
    category_id            TEXT,
    publish_time           TEXT,
    tags                   TEXT,
    views                  TEXT,
    likes                  TEXT,
    dislikes               TEXT,
    comment_count          TEXT,
    comments_disabled      TEXT,
    ratings_disabled       TEXT,
    video_error_or_removed TEXT,
    country                TEXT
) PARTITION BY LIST (country);
```

### Step 3 – Create one partition per country

```sql
CREATE TABLE staging_videos_ca PARTITION OF staging_videos_new FOR VALUES IN ('CA');
CREATE TABLE staging_videos_de PARTITION OF staging_videos_new FOR VALUES IN ('DE');
CREATE TABLE staging_videos_fr PARTITION OF staging_videos_new FOR VALUES IN ('FR');
CREATE TABLE staging_videos_gb PARTITION OF staging_videos_new FOR VALUES IN ('GB');
CREATE TABLE staging_videos_in PARTITION OF staging_videos_new FOR VALUES IN ('IN');
CREATE TABLE staging_videos_jp PARTITION OF staging_videos_new FOR VALUES IN ('JP');
CREATE TABLE staging_videos_kr PARTITION OF staging_videos_new FOR VALUES IN ('KR');
CREATE TABLE staging_videos_mx PARTITION OF staging_videos_new FOR VALUES IN ('MX');
CREATE TABLE staging_videos_ru PARTITION OF staging_videos_new FOR VALUES IN ('RU');
CREATE TABLE staging_videos_us PARTITION OF staging_videos_new FOR VALUES IN ('US');
```

A `DEFAULT` partition catches any unexpected country value so inserts never fail:

```sql
CREATE TABLE staging_videos_other PARTITION OF staging_videos_new DEFAULT;
```

### Step 4 – Move the existing data

```sql
INSERT INTO staging_videos_new
SELECT * FROM staging_videos;
```

### Step 5 – Measure the performance difference

Same query against both tables, using `EXPLAIN (ANALYZE, BUFFERS)`:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM staging_videos WHERE country = 'US';

EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM staging_videos_new WHERE country = 'US';
```

**Results**

| | `staging_videos` (non-partitioned) | `staging_videos_new` (partitioned) |
|---|---|---|
| Scan | Seq Scan on the whole table | Seq Scan on `staging_videos_us` only |
| Rows returned | 40,949 | 40,949 |
| Rows removed by filter | 334,993 | none (other countries never read) |
| Buffers | hit=3,501, read=22,201 | hit=2,388 |
| Planning time | 6.4 ms | 27.6 ms |
| **Execution time** | **~2,790 ms** | **~1,728 ms** |

**Takeaway:** partition pruning let Postgres skip every partition except `US`, so it read far fewer pages and avoided filtering ~335k irrelevant rows. Execution time dropped by roughly **38%** (about **1.6x faster**).

**Caveats worth noting**

- The non-partitioned run shows 22,201 disk reads, while the partitioned run was served entirely from cache. Part of the gap may come from cache warmth, so the fair test is to run each query several times and compare steady-state timings.
- Planning time is higher on the partitioned table, since the planner has to evaluate partition bounds. This is negligible here but grows with the number of partitions.
- Row width differs (484 vs 429 bytes), so the two tables are not byte-for-byte identical in layout.

---

## Next Steps

- [ ] Re-run the benchmark with warm cache on both tables and record averages
- [ ] Add a B-tree index on `country` (and on other filter columns) and compare against partitioning
- [ ] Convert the `TEXT` columns to proper types (`INTEGER`, `TIMESTAMP`, `BOOLEAN`)
- [ ] Normalize the schema (channels, categories, videos, trending snapshots)
- [ ] Compare join algorithms (nested loop, hash join, merge join) with `EXPLAIN ANALYZE`
- [ ] Build OLAP-style aggregate queries and analyze them

*This log will keep growing as the journey continues.*
