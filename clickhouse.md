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
