# ClickHouse Notes

Notes about ClickHouse.

## Partitions ([ref](https://clickhouse.com/docs/optimize/partitioning-key))

- 1 insert == 1 part PER partition

    If the insert has data belonging to >1 partition, then 1 insert == >1 data part.

    Will cause negative impact to merging, need to do more work.

    Only can merge WITHIN the same partition.

- Performance negative impact

    If query need to scan across multiple partitions, need to read from more data parts.

- Partition key cardinality

    LOW cardinality (100 - 1,000)

- MinMax index auto created on the columns used by partition key

    Can reduce the data scanned, if the query contains the partition key colum.

- To check how many partitions read

    ```sql
    EXPLAIN indexes = 1
    SELECT avg(price)
    FROM transactions_partition
    WHERE contract_date < '2020-10-01'
    SETTINGS enable_filesystem_cache = 0
    ```

    ```
        ┌─explain────────────────────────────────────────────────────────────┐
     1. │ Expression ((Project names + Projection))                          │
     2. │   Aggregating                                                      │
     3. │     Expression (Before GROUP BY)                                   │
     4. │       Expression                                                   │
     5. │         ReadFromMergeTree (property.transactions_partition)        │
     6. │         Indexes:                                                   │
     7. │           MinMax                                                   │
     8. │             Keys:                                                  │
     9. │               contract_date                                        │
    10. │             Condition: (contract_date in (-Inf, 18535])            │
    11. │             Parts: 3/61                                            │
    12. │             Granules: 3/61                                         │
    13. │           Partition                                                │
    14. │             Keys:                                                  │
    15. │               toYYYYMM(contract_date)                              │
    16. │             Condition: (toYYYYMM(contract_date) in (-Inf, 202010]) │
    17. │             Parts: 3/3                                             │
    18. │             Granules: 3/3                                          │
    19. │           PrimaryKey                                               │
    20. │             Condition: true                                        │
    21. │             Parts: 3/3                                             │
    22. │             Granules: 3/3                                          │
        └────────────────────────────────────────────────────────────────────┘
    ```

    Read from 3 partitions

    ```
    MinMax
        Keys:
            contract_date
        Condition: (contract_date in (-Inf, 18535])
        Parts: 3/61
        Granules: 3/61
    ```

- To check how many partitions in the table

    ```sql
    SELECT countDistinct(_partition_value) AS partition
    FROM transactions_partition
    ORDER BY partition ASC;
    ```

    ```
       ┌─partition─┐
    1. │        61 │
       └───────────┘
    ```

## Query Parallelism ([ref](https://clickhouse.com/docs/optimize/query-parallelism))

- To show the current max thread setting.
    
    This is only the hard upper limit.

    ⚠️ actual query maybe less than this value if never hit the threshold

    - `merge_tree_min_rows_for_concurrent_read`
    - `merge_tree_min_bytes_for_concurrent_read`

    ```sql
    SELECT getSetting('max_threads');
    
    SELECT getSetting('merge_tree_min_rows_for_concurrent_read');
    
    SELECT getSetting('merge_tree_min_bytes_for_concurrent_read');
    ```

- To show how many parallel processing lanes the query will actually use

    ```sql
    EXPLAIN PIPELINE
    SELECT
    ...
    FROM
    my_table_name;
    ```

    Result

    ```
    ...   
    (ReadFromMergeTree)
    MergeTreeSelect(pool: PrefetchedReadPool, algorithm: Thread) × 30
    ```

## Two Array Columns (Map)

[PR docs](https://github.com/ClickHouse/ClickHouse/pull/72517)

### Setup

1. 1 column is to store the key, other column is to store the value.

    ```sql
    CREATE TABLE testing_array (
        `pid` String,

        `keys` Array(String),
        `values` Array(String),

        `random_keys` Array(String),
        `random_values` Array(String),

        `sorted_key_value` Array(Tuple(String, String)) MATERIALIZED arraySort(arrayZip(random_keys, random_values)),
        `sorted_keys` Array(String) ALIAS arrayMap(x -> x.1, sorted_key_value),
        `sorted_values` Array(String) ALIAS arrayMap(x -> x.2, sorted_key_value),

        `sorted_keys_mat` Array(String) MATERIALIZED arrayMap(x -> x.1, sorted_key_value),
        `sorted_values_mat` Array(String) MATERIALIZED arrayMap(x -> x.2, sorted_key_value)
    ) ENGINE = MergeTree
    ORDER BY pid;
    ```

2. Insert sample data and randomise the key order.

    ```sql
    INSERT INTO testing_array
    SELECT
        'abc',
        ['LineId', 'Time', 'Level', 'Content', 'EventId', 'EventTemplate'] AS keys,
        [toString(LineId), Time, Level, Content, EventId, EventTemplate] AS values,
        arrayMap(x -> x.1, shuffled) AS random_keys,
        arrayMap(x -> x.2, shuffled) AS random_values
    FROM (
        SELECT
            LineId, Time, Level, Content, EventId, EventTemplate,
            arrayShuffle(arrayZip(
                ['LineId', 'Time', 'Level', 'Content', 'EventId', 'EventTemplate'],
                [toString(LineId), Time, Level, Content, EventId, EventTemplate]
            )) AS shuffled
        FROM file('Apache_2k.log_structured.csv')
    );
    ```

### Results

| setup | time (sec) | memory (kib) |
| - | - | - |
| random order | 0.005 | 59.57 |
| sorted order (alias) | 0.013 | 62.90 |
| sorted order (materialised) | 0.003 | 62.80 |

- Query using the random order columns.

    ```sql
    SELECT count()
    FROM testing_array
    WHERE (random_values[indexOf(random_keys, 'Content')]) ILIKE '%m%'
    FORMAT `NULL`
    SETTINGS enable_filesystem_cache = 0

    Query id: 922abe7a-faaa-440c-add2-58ecd82a4cef

    Ok.

    0 rows in set. Elapsed: 0.005 sec. Processed 2.00 thousand rows, 576.76 KB (390.80 thousand rows/s., 112.70 MB/s.)
    Peak memory usage: 59.57 KiB.
    ```

- Query using the sorted order alias columns.

    ```sql
    SELECT count()
    FROM testing_array
    WHERE (sorted_values[indexOfAssumeSorted(sorted_keys, 'Content')]) ILIKE '%m%'
    FORMAT `NULL`
    SETTINGS enable_filesystem_cache = 0

    Query id: 30442161-538f-4862-94ce-6a92840488b8

    Ok.

    0 rows in set. Elapsed: 0.013 sec. Processed 2.00 thousand rows, 560.76 KB (148.64 thousand rows/s., 41.67 MB/s.)
    Peak memory usage: 62.90 KiB.
    ```

- Query using the sorted order materialised columns.

    ```sql
    SELECT count()
    FROM testing_array
    WHERE (sorted_values_mat[indexOfAssumeSorted(sorted_keys_mat, 'Content')]) ILIKE '%m%'
    FORMAT `NULL`
    SETTINGS enable_filesystem_cache = 0

    Query id: 2d57123a-a420-4d9d-8c74-86cb22bd7f60

    Ok.

    0 rows in set. Elapsed: 0.003 sec. Processed 2.00 thousand rows, 576.76 KB (756.85 thousand rows/s., 218.26 MB/s.)
    Peak memory usage: 62.80 KiB.
    ```



- Why sorted alias column take longer than random order?

- Why sorted alias/materialised column take more memory than random order?

- What other settings to use for benchmarking? (Format null/enable_filesystem_cache = 0)
