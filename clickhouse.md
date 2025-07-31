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

  - count query

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
  
  - query log

    ```sql
    hostname:                               fcd2d8cd3c64
    type:                                   QueryFinish
    event_date:                             2025-07-30
    event_time:                             2025-07-30 00:53:20
    event_time_microseconds:                2025-07-30 00:53:20.989193
    query_start_time:                       2025-07-30 00:53:20
    query_start_time_microseconds:          2025-07-30 00:53:20.984546
    query_duration_ms:                      4
    read_rows:                              2000
    read_bytes:                             576756
    written_rows:                           0
    written_bytes:                          0
    result_rows:                            1
    result_bytes:                           136
    memory_usage:                           61000
    current_database:                       log_testing
    query:                                  SELECT count()
    FROM testing_array
    WHERE (random_values[indexOf(random_keys, 'Content')]) ILIKE '%m%'
    FORMAT NULL SETTINGS enable_filesystem_cache = 0;
    formatted_query:
    normalized_query_hash:                  4913069487316396775
    query_kind:                             Select
    databases:                              ['log_testing']
    tables:                                 ['log_testing.testing_array']
    columns:                                ['log_testing.testing_array.random_keys','log_testing.testing_array.random_values']
    partitions:                             ['log_testing.testing_array.all']
    projections:                            []
    views:                                  []
    exception_code:                         0
    exception:
    stack_trace:
    is_initial_query:                       1
    user:                                   default
    query_id:                               922abe7a-faaa-440c-add2-58ecd82a4cef
    address:                                ::ffff:172.18.0.1
    port:                                   63372
    initial_user:                           default
    initial_query_id:                       922abe7a-faaa-440c-add2-58ecd82a4cef
    initial_address:                        ::ffff:172.18.0.1
    initial_port:                           63372
    initial_query_start_time:               2025-07-30 00:53:20
    initial_query_start_time_microseconds:  2025-07-30 00:53:20.984546
    interface:                              1
    is_secure:                              0
    os_user:                                reuben
    client_hostname:                        reubens-MacBook-Pro.local
    client_name:                            ClickHouse client
    client_revision:                        54477
    client_version_major:                   25
    client_version_minor:                   6
    client_version_patch:                   1
    script_query_number:                    1
    script_line_number:                     1
    http_method:                            0
    http_user_agent:
    http_referer:
    forwarded_for:
    quota_key:
    distributed_depth:                      0
    revision:                               54501
    log_comment:
    thread_ids:                             [3031,3039,738,807,814,803,782,737,3037,766,788,84]
    peak_threads_usage:                     12
    ProfileEvents:                          {'Query':1,'SelectQuery':1,'InitialQuery':1,'QueriesWithSubqueries':1,'SelectQueriesWithSubqueries':1,'FileOpen':1,'ReadBufferFromFileDescriptorReadBytes':92112,'ReadCompressedBytes':76588,'CompressedReadBufferBlocks':2,'CompressedReadBufferBytes':384756,'OpenedFileCacheMisses':1,'OpenedFileCacheMicroseconds':1,'IOBufferAllocs':2,'IOBufferAllocBytes':319002,'ArenaAllocChunks':1,'ArenaAllocBytes':4096,'FunctionExecute':6,'MarkCacheHits':1,'QueryConditionCacheMisses':1,'CreatedReadBufferOrdinary':1,'DiskReadElapsedMicroseconds':48,'NetworkSendElapsedMicroseconds':70,'NetworkSendBytes':3768,'GlobalThreadPoolJobs':11,'LocalThreadPoolExpansions':10,'LocalThreadPoolShrinks':10,'LocalThreadPoolThreadCreationMicroseconds':213,'LocalThreadPoolJobs':10,'SelectedParts':1,'SelectedPartsTotal':1,'SelectedRanges':1,'SelectedMarks':1,'SelectedMarksTotal':1,'SelectedRows':2000,'SelectedBytes':576756,'RowsReadByMainReader':2000,'FilteringMarksWithPrimaryKeyMicroseconds':1,'WaitMarksLoadMicroseconds':7,'ContextLock':54,'RWLockAcquiredReadLocks':1,'PartsLockHoldMicroseconds':14,'RealTimeMicroseconds':22241,'UserTimeMicroseconds':4414,'SystemTimeMicroseconds':699,'SoftPageFaults':300,'OSCPUWaitMicroseconds':29,'OSCPUVirtualTimeMicroseconds':5111,'OSWriteBytes':4096,'OSReadChars':97199,'OSWriteChars':4033,'ThreadPoolReaderPageCacheHit':2,'ThreadPoolReaderPageCacheHitBytes':92112,'ThreadPoolReaderPageCacheHitElapsedMicroseconds':48,'SynchronousReadWaitMicroseconds':51,'LogTrace':13,'LogDebug':6,'LoggerElapsedNanoseconds':201128,'InterfaceNativeSendBytes':3768,'ConcurrencyControlSlotsGranted':1,'ConcurrencyControlSlotsAcquired':9,'ConcurrencyControlSlotsAcquiredNonCompeting':1,'FilterTransformPassedRows':583,'FilterTransformPassedBytes':2000}
    Settings:                               {'enable_filesystem_cache':'0','parallel_replicas_for_cluster_engines':'0'}
    used_aggregate_functions:               ['count','max','min']
    used_aggregate_function_combinators:    []
    used_database_engines:                  []
    used_data_type_families:                []
    used_dictionaries:                      []
    used_formats:                           []
    used_functions:                         ['arrayMap','arrayElement','ilike','indexOf','tupleElement']
    used_storages:                          []
    used_table_functions:                   []
    used_executable_user_defined_functions: []
    used_sql_user_defined_functions:        []
    used_row_policies:                      []
    used_privileges:                        ['SELECT(random_values, random_keys) ON log_testing.testing_array']
    missing_privileges:                     []
    transaction_id:                         (0,0,'00000000-0000-0000-0000-000000000000')
    query_cache_usage:                      None
    asynchronous_read_counters:             {}
    ```

  - explain actions

    ```sql
    EXPLAIN actions = 1
    SELECT count()
    FROM testing_array
    WHERE (random_values[indexOf(random_keys, 'Content')]) ILIKE '%m%'
    SETTINGS enable_filesystem_cache = 0

    Query id: ff2431de-65d2-456f-8ef8-be3d867646c2

        ┌─explain────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
     1. │ Expression ((Project names + Projection))                                                                                                                                                                                                                  │
     2. │ Actions: INPUT :: 0 -> count() UInt64 : 0                                                                                                                                                                                                                  │
     3. │ Positions: 0                                                                                                                                                                                                                                               │
     4. │   Aggregating                                                                                                                                                                                                                                              │
     5. │   Keys:                                                                                                                                                                                                                                                    │
     6. │   Aggregates:                                                                                                                                                                                                                                              │
     7. │       count()                                                                                                                                                                                                                                              │
     8. │         Function: count() → UInt64                                                                                                                                                                                                                         │
     9. │         Arguments: none                                                                                                                                                                                                                                    │
    10. │   Skip merging: 0                                                                                                                                                                                                                                          │
    11. │     Expression (Before GROUP BY)                                                                                                                                                                                                                           │
    12. │     Positions:                                                                                                                                                                                                                                             │
    13. │       Filter ((WHERE + Change column names to column identifiers))                                                                                                                                                                                         │
    14. │       Filter column: ilike(arrayElement(__table1.random_values, indexOf(__table1.random_keys, 'Content'_String)), '%m%'_String) (removed)                                                                                                                  │
    15. │       Actions: INPUT : 0 -> random_values Array(String) : 0                                                                                                                                                                                                │
    16. │                INPUT : 1 -> random_keys Array(String) : 1                                                                                                                                                                                                  │
    17. │                COLUMN Const(String) -> 'Content'_String String : 2                                                                                                                                                                                         │
    18. │                COLUMN Const(String) -> '%m%'_String String : 3                                                                                                                                                                                             │
    19. │                FUNCTION indexOf(random_keys :: 1, 'Content'_String :: 2) -> indexOf(__table1.random_keys, 'Content'_String) UInt64 : 4                                                                                                                     │
    20. │                FUNCTION arrayElement(random_values :: 0, indexOf(__table1.random_keys, 'Content'_String) :: 4) -> arrayElement(__table1.random_values, indexOf(__table1.random_keys, 'Content'_String)) String : 2                                         │
    21. │                FUNCTION ilike(arrayElement(__table1.random_values, indexOf(__table1.random_keys, 'Content'_String)) :: 2, '%m%'_String :: 3) -> ilike(arrayElement(__table1.random_values, indexOf(__table1.random_keys, 'Content'_String)), '%m%'_String) UInt8 : 4 │
    22. │       Positions: 4                                                                                                                                                                                                                                         │
    23. │         ReadFromMergeTree (log_testing.testing_array)                                                                                                                                                                                                      │
    24. │         ReadType: Default                                                                                                                                                                                                                                  │
    25. │         Parts: 1                                                                                                                                                                                                                                           │
    26. │         Granules: 1                                                                                                                                                                                                                                        │
        └────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
    ```

- Query using the sorted order alias columns.

  - count query

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

  - query log

    ```sql
    hostname:                               fcd2d8cd3c64
    type:                                   QueryFinish
    event_date:                             2025-07-30
    event_time:                             2025-07-30 00:54:12
    event_time_microseconds:                2025-07-30 00:54:12.607737
    query_start_time:                       2025-07-30 00:54:12
    query_start_time_microseconds:          2025-07-30 00:54:12.595427
    query_duration_ms:                      12
    read_rows:                              2000
    read_bytes:                             560756
    written_rows:                           0
    written_bytes:                          0
    result_rows:                            1
    result_bytes:                           136
    memory_usage:                           64408
    current_database:                       log_testing
    query:                                  SELECT count()
    FROM testing_array
    WHERE (sorted_values[indexOfAssumeSorted(sorted_keys, 'Content')]) ILIKE '%m%'
    FORMAT `NULL`
    SETTINGS enable_filesystem_cache = 0;
    formatted_query:
    normalized_query_hash:                  3631270350271597700
    query_kind:                             Select
    databases:                              ['log_testing']
    tables:                                 ['log_testing.testing_array']
    columns:                                ['log_testing.testing_array.sorted_key_value']
    partitions:                             ['log_testing.testing_array.all']
    projections:                            []
    views:                                  []
    exception_code:                         0
    exception:
    stack_trace:
    is_initial_query:                       1
    user:                                   default
    query_id:                               30442161-538f-4862-94ce-6a92840488b8
    address:                                ::ffff:172.18.0.1
    port:                                   63372
    initial_user:                           default
    initial_query_id:                       30442161-538f-4862-94ce-6a92840488b8
    initial_address:                        ::ffff:172.18.0.1
    initial_port:                           63372
    initial_query_start_time:               2025-07-30 00:54:12
    initial_query_start_time_microseconds:  2025-07-30 00:54:12.595427
    interface:                              1
    is_secure:                              0
    os_user:                                reuben
    client_hostname:                        reubens-MacBook-Pro.local
    client_name:                            ClickHouse client
    client_revision:                        54477
    client_version_major:                   25
    client_version_minor:                   6
    client_version_patch:                   1
    script_query_number:                    1
    script_line_number:                     1
    http_method:                            0
    http_user_agent:
    http_referer:
    forwarded_for:
    quota_key:
    distributed_depth:                      0
    revision:                               54501
    log_comment:
    thread_ids:                             [726,3034,3035,806,3036,3044,3038,84,797,812,764,741,2985,719]
    peak_threads_usage:                     14
    ProfileEvents:                          {'Query':1,'SelectQuery':1,'InitialQuery':1,'QueriesWithSubqueries':1,'SelectQueriesWithSubqueries':1,'FileOpen':1,'ReadBufferFromFileDescriptorReadBytes':46056,'ReadCompressedBytes':30002,'CompressedReadBufferBlocks':1,'CompressedReadBufferBytes':368756,'OpenedFileCacheMisses':1,'OpenedFileCacheMicroseconds':2,'IOBufferAllocs':2,'IOBufferAllocBytes':415002,'ArenaAllocChunks':1,'ArenaAllocBytes':4096,'FunctionExecute':24,'MarkCacheHits':1,'QueryConditionCacheMisses':1,'CreatedReadBufferOrdinary':1,'DiskReadElapsedMicroseconds':64,'NetworkSendElapsedMicroseconds':182,'NetworkSendBytes':3830,'GlobalThreadPoolJobs':13,'LocalThreadPoolExpansions':12,'LocalThreadPoolShrinks':12,'LocalThreadPoolThreadCreationMicroseconds':140,'LocalThreadPoolJobs':12,'SelectedParts':1,'SelectedPartsTotal':1,'SelectedRanges':1,'SelectedMarks':1,'SelectedMarksTotal':1,'SelectedRows':2000,'SelectedBytes':560756,'RowsReadByMainReader':2000,'FilteringMarksWithPrimaryKeyMicroseconds':4,'WaitMarksLoadMicroseconds':10,'ContextLock':58,'ContextLockWaitMicroseconds':1,'RWLockAcquiredReadLocks':1,'PartsLockHoldMicroseconds':23,'RealTimeMicroseconds':23493,'UserTimeMicroseconds':7786,'SystemTimeMicroseconds':5276,'SoftPageFaults':366,'OSCPUWaitMicroseconds':12,'OSCPUVirtualTimeMicroseconds':13058,'OSWriteBytes':4096,'OSReadChars':51971,'OSWriteChars':4193,'ThreadPoolReaderPageCacheHit':1,'ThreadPoolReaderPageCacheHitBytes':46056,'ThreadPoolReaderPageCacheHitElapsedMicroseconds':64,'SynchronousReadWaitMicroseconds':67,'LogTrace':13,'LogDebug':6,'LoggerElapsedNanoseconds':2336087,'InterfaceNativeSendBytes':3830,'ConcurrencyControlSlotsGranted':1,'ConcurrencyControlSlotsAcquired':11,'ConcurrencyControlSlotsAcquiredNonCompeting':1,'FilterTransformPassedRows':583,'FilterTransformPassedBytes':2000}
    Settings:                               {'enable_filesystem_cache':'0','parallel_replicas_for_cluster_engines':'0'}
    used_aggregate_functions:               ['count','max','min']
    used_aggregate_function_combinators:    []
    used_database_engines:                  []
    used_data_type_families:                []
    used_dictionaries:                      []
    used_formats:                           []
    used_functions:                         ['arrayMap','indexOfAssumeSorted','arrayElement','ilike','tupleElement']
    used_storages:                          []
    used_table_functions:                   []
    used_executable_user_defined_functions: []
    used_sql_user_defined_functions:        []
    used_row_policies:                      []
    used_privileges:                        ['SELECT(sorted_values, sorted_keys) ON log_testing.testing_array']
    missing_privileges:                     []
    transaction_id:                         (0,0,'00000000-0000-0000-0000-000000000000')
    query_cache_usage:                      None
    asynchronous_read_counters:             {}
    ```

  - explain actions

    ```sql
    EXPLAIN actions = 1
    SELECT count()
    FROM testing_array
    WHERE (sorted_values[indexOfAssumeSorted(sorted_keys, 'Content')]) ILIKE '%m%'
    SETTINGS enable_filesystem_cache = 0

        ┌─explain────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
     1. │ Expression ((Project names + Projection))                                                                                                                                                                                                                  │
     2. │ Actions: INPUT :: 0 -> count() UInt64 : 0                                                                                                                                                                                                                  │
     3. │ Positions: 0                                                                                                                                                                                                                                               │
     4. │   Aggregating                                                                                                                                                                                                                                              │
     5. │   Keys:                                                                                                                                                                                                                                                    │
     6. │   Aggregates:                                                                                                                                                                                                                                              │
     7. │       count()                                                                                                                                                                                                                                              │
     8. │         Function: count() → UInt64                                                                                                                                                                                                                         │
     9. │         Arguments: none                                                                                                                                                                                                                                    │
    10. │   Skip merging: 0                                                                                                                                                                                                                                          │
    11. │     Expression (Before GROUP BY)                                                                                                                                                                                                                           │
    12. │     Positions:                                                                                                                                                                                                                                             │
    13. │       Filter ((WHERE + (Change column names to column identifiers + Compute alias columns)))                                                                                                                                                               │
    14. │       Filter column: ilike(arrayElement(__table1.sorted_values, indexOfAssumeSorted(__table1.sorted_keys, 'Content'_String)), '%m%'_String) (removed)                                                                                                      │
    15. │       Actions: INPUT : 0 -> sorted_key_value Array(Tuple(String, String)) : 0                                                                                                                                                                              │
    16. │                COLUMN Const(Function) -> x Tuple(String, String) -> tupleElement(x, 1_UInt8) Function(Tuple(String, String) -> String) : 1                                                                                                                 │
    17. │                COLUMN Const(Function) -> x Tuple(String, String) -> tupleElement(x, 2_UInt8) Function(Tuple(String, String) -> String) : 2                                                                                                                 │
    18. │                COLUMN Const(String) -> 'Content'_String String : 3                                                                                                                                                                                         │
    19. │                COLUMN Const(String) -> '%m%'_String String : 4                                                                                                                                                                                             │
    20. │                FUNCTION arrayMap(x Tuple(String, String) -> tupleElement(x, 1_UInt8) :: 1, sorted_key_value : 0) -> arrayMap(x Tuple(String, String) -> tupleElement(x, 1_UInt8), sorted_key_value) Array(String) : 5                                      │
    21. │                FUNCTION arrayMap(x Tuple(String, String) -> tupleElement(x, 2_UInt8) :: 2, sorted_key_value :: 0) -> arrayMap(x Tuple(String, String) -> tupleElement(x, 2_UInt8), sorted_key_value) Array(String) : 1                                     │
    22. │                FUNCTION indexOfAssumeSorted(arrayMap(x Tuple(String, String) -> tupleElement(x, 1_UInt8), sorted_key_value) :: 5, 'Content'_String :: 3) -> indexOfAssumeSorted(__table1.sorted_keys, 'Content'_String) UInt64 : 0                         │
    23. │                FUNCTION arrayElement(arrayMap(x Tuple(String, String) -> tupleElement(x, 2_UInt8), sorted_key_value) :: 1, indexOfAssumeSorted(__table1.sorted_keys, 'Content'_String) :: 0) -> arrayElement(__table1.sorted_values, indexOfAssumeSorted(__table1.sorted_keys, 'Content'_String)) String : 3 │
    24. │                FUNCTION ilike(arrayElement(__table1.sorted_values, indexOfAssumeSorted(__table1.sorted_keys, 'Content'_String)) :: 3, '%m%'_String :: 4) -> ilike(arrayElement(__table1.sorted_values, indexOfAssumeSorted(__table1.sorted_keys, 'Content'_String)), '%m%'_String) UInt8 : 0 │
    25. │       Positions: 0                                                                                                                                                                                                                                         │
    26. │         ReadFromMergeTree (log_testing.testing_array)                                                                                                                                                                                                      │
    27. │         ReadType: Default                                                                                                                                                                                                                                  │
    28. │         Parts: 1                                                                                                                                                                                                                                           │
    29. │         Granules: 1                                                                                                                                                                                                                                        │
        └────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
    ```

- Query using the sorted order materialised columns.

  - count query

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

  - query log

    ```sql
    hostname:                               fcd2d8cd3c64
    type:                                   QueryFinish
    event_date:                             2025-07-30
    event_time:                             2025-07-30 00:55:06
    event_time_microseconds:                2025-07-30 00:55:06.410975
    query_start_time:                       2025-07-30 00:55:06
    query_start_time_microseconds:          2025-07-30 00:55:06.408492
    query_duration_ms:                      2
    read_rows:                              2000
    read_bytes:                             576756
    written_rows:                           0
    written_bytes:                          0
    result_rows:                            1
    result_bytes:                           136
    memory_usage:                           64312
    current_database:                       log_testing
    query:                                  SELECT count()
    FROM testing_array
    WHERE (sorted_values_mat[indexOfAssumeSorted(sorted_keys_mat, 'Content')]) ILIKE '%m%'
    FORMAT `NULL`
    SETTINGS enable_filesystem_cache = 0;
    formatted_query:
    normalized_query_hash:                  16039326560965710386
    query_kind:                             Select
    databases:                              ['log_testing']
    tables:                                 ['log_testing.testing_array']
    columns:                                ['log_testing.testing_array.sorted_keys_mat','log_testing.testing_array.sorted_values_mat']
    partitions:                             ['log_testing.testing_array.all']
    projections:                            []
    views:                                  []
    exception_code:                         0
    exception:
    stack_trace:
    is_initial_query:                       1
    user:                                   default
    query_id:                               2d57123a-a420-4d9d-8c74-86cb22bd7f60
    address:                                ::ffff:172.18.0.1
    port:                                   63372
    initial_user:                           default
    initial_query_id:                       2d57123a-a420-4d9d-8c74-86cb22bd7f60
    initial_address:                        ::ffff:172.18.0.1
    initial_port:                           63372
    initial_query_start_time:               2025-07-30 00:55:06
    initial_query_start_time_microseconds:  2025-07-30 00:55:06.408492
    interface:                              1
    is_secure:                              0
    os_user:                                reuben
    client_hostname:                        reubens-MacBook-Pro.local
    client_name:                            ClickHouse client
    client_revision:                        54477
    client_version_major:                   25
    client_version_minor:                   6
    client_version_patch:                   1
    script_query_number:                    1
    script_line_number:                     1
    http_method:                            0
    http_user_agent:
    http_referer:
    forwarded_for:
    quota_key:
    distributed_depth:                      0
    revision:                               54501
    log_comment:
    thread_ids:                             [785,3025,2979,768,792,3007,747,813,780,751,84,3028,787,718]
    peak_threads_usage:                     14
    ProfileEvents:                          {'Query':1,'SelectQuery':1,'InitialQuery':1,'QueriesWithSubqueries':1,'SelectQueriesWithSubqueries':1,'FileOpen':1,'ReadBufferFromFileDescriptorReadBytes':59678,'ReadCompressedBytes':30104,'CompressedReadBufferBlocks':2,'CompressedReadBufferBytes':384756,'OpenedFileCacheMisses':1,'OpenedFileCacheMicroseconds':1,'IOBufferAllocs':2,'IOBufferAllocBytes':319002,'ArenaAllocChunks':1,'ArenaAllocBytes':4096,'FunctionExecute':6,'MarkCacheHits':1,'QueryConditionCacheMisses':1,'CreatedReadBufferOrdinary':1,'DiskReadElapsedMicroseconds':19,'NetworkSendElapsedMicroseconds':45,'NetworkSendBytes':3637,'GlobalThreadPoolJobs':13,'LocalThreadPoolExpansions':12,'LocalThreadPoolShrinks':12,'LocalThreadPoolThreadCreationMicroseconds':64,'LocalThreadPoolJobs':12,'SelectedParts':1,'SelectedPartsTotal':1,'SelectedRanges':1,'SelectedMarks':1,'SelectedMarksTotal':1,'SelectedRows':2000,'SelectedBytes':576756,'RowsReadByMainReader':2000,'WaitMarksLoadMicroseconds':11,'ContextLock':54,'RWLockAcquiredReadLocks':1,'PartsLockHoldMicroseconds':5,'RealTimeMicroseconds':12678,'UserTimeMicroseconds':2442,'SystemTimeMicroseconds':378,'SoftPageFaults':38,'OSCPUVirtualTimeMicroseconds':2815,'OSWriteBytes':4096,'OSReadChars':65626,'OSWriteChars':4075,'ThreadPoolReaderPageCacheHit':2,'ThreadPoolReaderPageCacheHitBytes':59678,'ThreadPoolReaderPageCacheHitElapsedMicroseconds':19,'SynchronousReadWaitMicroseconds':20,'LogTrace':13,'LogDebug':6,'LoggerElapsedNanoseconds':114999,'InterfaceNativeSendBytes':3637,'ConcurrencyControlSlotsGranted':1,'ConcurrencyControlSlotsAcquired':11,'ConcurrencyControlSlotsAcquiredNonCompeting':1,'FilterTransformPassedRows':583,'FilterTransformPassedBytes':2000}
    Settings:                               {'enable_filesystem_cache':'0','parallel_replicas_for_cluster_engines':'0'}
    used_aggregate_functions:               ['count','max','min']
    used_aggregate_function_combinators:    []
    used_database_engines:                  []
    used_data_type_families:                []
    used_dictionaries:                      []
    used_formats:                           []
    used_functions:                         ['arrayMap','indexOfAssumeSorted','arrayElement','ilike','tupleElement']
    used_storages:                          []
    used_table_functions:                   []
    used_executable_user_defined_functions: []
    used_sql_user_defined_functions:        []
    used_row_policies:                      []
    used_privileges:                        ['SELECT(sorted_values_mat, sorted_keys_mat) ON log_testing.testing_array']
    missing_privileges:                     []
    transaction_id:                         (0,0,'00000000-0000-0000-0000-000000000000')
    query_cache_usage:                      None
    asynchronous_read_counters:             {}
    ```

  - explain actions

    ```sql
    EXPLAIN actions = 1
    SELECT count()
    FROM testing_array
    WHERE (sorted_values_mat[indexOfAssumeSorted(sorted_keys_mat, 'Content')]) ILIKE '%m%'
    SETTINGS enable_filesystem_cache = 0

    Query id: 61b2aa9d-4008-49e2-875f-89f449bff1d3

        ┌─explain────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
     1. │ Expression ((Project names + Projection))                                                                                                                                                                                                                  │
     2. │ Actions: INPUT :: 0 -> count() UInt64 : 0                                                                                                                                                                                                                  │
     3. │ Positions: 0                                                                                                                                                                                                                                               │
     4. │   Aggregating                                                                                                                                                                                                                                              │
     5. │   Keys:                                                                                                                                                                                                                                                    │
     6. │   Aggregates:                                                                                                                                                                                                                                              │
     7. │       count()                                                                                                                                                                                                                                              │
     8. │         Function: count() → UInt64                                                                                                                                                                                                                         │
     9. │         Arguments: none                                                                                                                                                                                                                                    │
    10. │   Skip merging: 0                                                                                                                                                                                                                                          │
    11. │     Expression (Before GROUP BY)                                                                                                                                                                                                                           │
    12. │     Positions:                                                                                                                                                                                                                                             │
    13. │       Filter ((WHERE + Change column names to column identifiers))                                                                                                                                                                                         │
    14. │       Filter column: ilike(arrayElement(__table1.sorted_values_mat, indexOfAssumeSorted(__table1.sorted_keys_mat, 'Content'_String)), '%m%'_String) (removed)                                                                                              │
    15. │       Actions: INPUT : 0 -> sorted_values_mat Array(String) : 0                                                                                                                                                                                            │
    16. │                INPUT : 1 -> sorted_keys_mat Array(String) : 1                                                                                                                                                                                              │
    17. │                COLUMN Const(String) -> 'Content'_String String : 2                                                                                                                                                                                         │
    18. │                COLUMN Const(String) -> '%m%'_String String : 3                                                                                                                                                                                             │
    19. │                FUNCTION indexOfAssumeSorted(sorted_keys_mat :: 1, 'Content'_String :: 2) -> indexOfAssumeSorted(__table1.sorted_keys_mat, 'Content'_String) UInt64 : 4                                                                                     │
    20. │                FUNCTION arrayElement(sorted_values_mat :: 0, indexOfAssumeSorted(__table1.sorted_keys_mat, 'Content'_String) :: 4) -> arrayElement(__table1.sorted_values_mat, indexOfAssumeSorted(__table1.sorted_keys_mat, 'Content'_String)) String : 2 │
    21. │                FUNCTION ilike(arrayElement(__table1.sorted_values_mat, indexOfAssumeSorted(__table1.sorted_keys_mat, 'Content'_String)) :: 2, '%m%'_String :: 3) -> ilike(arrayElement(__table1.sorted_values_mat, indexOfAssumeSorted(__table1.sorted_keys_mat, 'Content'_String)), '%m%'_String) UInt8 : 4 │
    22. │       Positions: 4                                                                                                                                                                                                                                         │
    23. │         ReadFromMergeTree (log_testing.testing_array)                                                                                                                                                                                                      │
    24. │         ReadType: Default                                                                                                                                                                                                                                  │
    25. │         Parts: 1                                                                                                                                                                                                                                           │
    26. │         Granules: 1                                                                                                                                                                                                                                        │
        └────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
    ```

- ProfileEvents

    ```sql
    WITH
        '922abe7a-faaa-440c-add2-58ecd82a4cef' AS qid1,
        '30442161-538f-4862-94ce-6a92840488b8' AS qid2
    SELECT
        event,
        any(value1) AS random,
        any(value2) AS sorted_alias,
        CASE
            WHEN random < sorted_alias THEN 'random'
            WHEN random > sorted_alias THEN 'alias sorted'
            ELSE ''
        END AS faster,
        abs(random - sorted_alias) AS difference,
        max(random, sorted_alias)/min(random, sorted_alias) AS difference_times
    FROM
    (
        SELECT
            arrayJoin(mapKeys(ProfileEvents)) AS event,
            ProfileEvents[event] AS value1,
            null AS value2
        FROM system.query_log
        WHERE query_id = qid1 AND type = 'QueryFinish'
        UNION ALL
        SELECT
            arrayJoin(mapKeys(ProfileEvents)) AS event,
            null AS value1,
            ProfileEvents[event] AS value2
        FROM system.query_log
        WHERE query_id = qid2 AND type = 'QueryFinish'
    )
    GROUP BY event
    ORDER BY faster, difference;
    ```

    ```
        ┌─event───────────────────────────────────────────┬─random─┬─sorted_alias─┬─faster───────┬─difference─┬───difference_times─┐
     1. │ RWLockAcquiredReadLocks                         │      1 │            1 │              │          0 │                  1 │
     2. │ SelectedPartsTotal                              │      1 │            1 │              │          0 │                  1 │
     3. │ FileOpen                                        │      1 │            1 │              │          0 │                  1 │
     4. │ Query                                           │      1 │            1 │              │          0 │                  1 │
     5. │ CreatedReadBufferOrdinary                       │      1 │            1 │              │          0 │                  1 │
     6. │ ArenaAllocChunks                                │      1 │            1 │              │          0 │                  1 │
     7. │ OSWriteBytes                                    │   4096 │         4096 │              │          0 │                  1 │
     8. │ SelectQueriesWithSubqueries                     │      1 │            1 │              │          0 │                  1 │
     9. │ LogDebug                                        │      6 │            6 │              │          0 │                  1 │
    10. │ MarkCacheHits                                   │      1 │            1 │              │          0 │                  1 │
    11. │ ArenaAllocBytes                                 │   4096 │         4096 │              │          0 │                  1 │
    12. │ SelectQuery                                     │      1 │            1 │              │          0 │                  1 │
    13. │ SelectedRanges                                  │      1 │            1 │              │          0 │                  1 │
    14. │ FilterTransformPassedBytes                      │   2000 │         2000 │              │          0 │                  1 │
    15. │ ConcurrencyControlSlotsGranted                  │      1 │            1 │              │          0 │                  1 │
    16. │ SelectedMarks                                   │      1 │            1 │              │          0 │                  1 │
    17. │ IOBufferAllocs                                  │      2 │            2 │              │          0 │                  1 │
    18. │ FilterTransformPassedRows                       │    583 │          583 │              │          0 │                  1 │
    19. │ SelectedRows                                    │   2000 │         2000 │              │          0 │                  1 │
    20. │ SelectedParts                                   │      1 │            1 │              │          0 │                  1 │
    21. │ InitialQuery                                    │      1 │            1 │              │          0 │                  1 │
    22. │ RowsReadByMainReader                            │   2000 │         2000 │              │          0 │                  1 │
    23. │ QueryConditionCacheMisses                       │      1 │            1 │              │          0 │                  1 │
    24. │ QueriesWithSubqueries                           │      1 │            1 │              │          0 │                  1 │
    25. │ SelectedMarksTotal                              │      1 │            1 │              │          0 │                  1 │
    26. │ OpenedFileCacheMisses                           │      1 │            1 │              │          0 │                  1 │
    27. │ LogTrace                                        │     13 │           13 │              │          0 │                  1 │
    28. │ ConcurrencyControlSlotsAcquiredNonCompeting     │      1 │            1 │              │          0 │                  1 │
    29. │ ContextLockWaitMicroseconds                     │   ᴺᵁᴸᴸ │            1 │              │       ᴺᵁᴸᴸ │               ᴺᵁᴸᴸ │
    30. │ ThreadPoolReaderPageCacheHit                    │      2 │            1 │ alias sorted │          1 │                  2 │
    31. │ CompressedReadBufferBlocks                      │      2 │            1 │ alias sorted │          1 │                  2 │
    32. │ OSCPUWaitMicroseconds                           │     29 │           12 │ alias sorted │         17 │ 2.4166666666666665 │
    33. │ LocalThreadPoolThreadCreationMicroseconds       │    213 │          140 │ alias sorted │         73 │ 1.5214285714285714 │
    34. │ CompressedReadBufferBytes                       │ 384756 │       368756 │ alias sorted │      16000 │ 1.0433891245159401 │
    35. │ SelectedBytes                                   │ 576756 │       560756 │ alias sorted │      16000 │  1.028532909144084 │
    36. │ OSReadChars                                     │  97199 │        51971 │ alias sorted │      45228 │  1.870254565045891 │
    37. │ ThreadPoolReaderPageCacheHitBytes               │  92112 │        46056 │ alias sorted │      46056 │                  2 │
    38. │ ReadBufferFromFileDescriptorReadBytes           │  92112 │        46056 │ alias sorted │      46056 │                  2 │
    39. │ ReadCompressedBytes                             │  76588 │        30002 │ alias sorted │      46586 │  2.552763149123392 │
    40. │ OpenedFileCacheMicroseconds                     │      1 │            2 │ random       │          1 │                  2 │
    41. │ LocalThreadPoolExpansions                       │     10 │           12 │ random       │          2 │                1.2 │
    42. │ LocalThreadPoolShrinks                          │     10 │           12 │ random       │          2 │                1.2 │
    43. │ GlobalThreadPoolJobs                            │     11 │           13 │ random       │          2 │ 1.1818181818181819 │
    44. │ ConcurrencyControlSlotsAcquired                 │      9 │           11 │ random       │          2 │ 1.2222222222222223 │
    45. │ LocalThreadPoolJobs                             │     10 │           12 │ random       │          2 │                1.2 │
    46. │ FilteringMarksWithPrimaryKeyMicroseconds        │      1 │            4 │ random       │          3 │                  4 │
    47. │ WaitMarksLoadMicroseconds                       │      7 │           10 │ random       │          3 │ 1.4285714285714286 │
    48. │ ContextLock                                     │     54 │           58 │ random       │          4 │ 1.0740740740740742 │
    49. │ PartsLockHoldMicroseconds                       │     14 │           23 │ random       │          9 │ 1.6428571428571428 │
    50. │ ThreadPoolReaderPageCacheHitElapsedMicroseconds │     48 │           64 │ random       │         16 │ 1.3333333333333333 │
    51. │ DiskReadElapsedMicroseconds                     │     48 │           64 │ random       │         16 │ 1.3333333333333333 │
    52. │ SynchronousReadWaitMicroseconds                 │     51 │           67 │ random       │         16 │ 1.3137254901960784 │
    53. │ FunctionExecute                                 │      6 │           24 │ random       │         18 │                  4 │
    54. │ InterfaceNativeSendBytes                        │   3768 │         3830 │ random       │         62 │ 1.0164543524416136 │
    55. │ NetworkSendBytes                                │   3768 │         3830 │ random       │         62 │ 1.0164543524416136 │
    56. │ SoftPageFaults                                  │    300 │          366 │ random       │         66 │               1.22 │
    57. │ NetworkSendElapsedMicroseconds                  │     70 │          182 │ random       │        112 │                2.6 │
    58. │ OSWriteChars                                    │   4033 │         4193 │ random       │        160 │  1.039672700223159 │
    59. │ RealTimeMicroseconds                            │  22241 │        23493 │ random       │       1252 │ 1.0562924328942045 │
    60. │ UserTimeMicroseconds                            │   4414 │         7786 │ random       │       3372 │ 1.7639329406434074 │
    61. │ SystemTimeMicroseconds                          │    699 │         5276 │ random       │       4577 │  7.547925608011445 │
    62. │ OSCPUVirtualTimeMicroseconds                    │   5111 │        13058 │ random       │       7947 │  2.554881627861475 │
    63. │ IOBufferAllocBytes                              │ 319002 │       415002 │ random       │      96000 │ 1.3009385521093912 │
    64. │ LoggerElapsedNanoseconds                        │ 201128 │      2336087 │ random       │    2134959 │ 11.614926812775943 │
        └─event───────────────────────────────────────────┴─random─┴─sorted_alias─┴─faster───────┴─difference─┴───difference_times─┘
    ```

- Why sorted alias column take longer than random order?

  ```
  53. │ FunctionExecute                                 │      6 │           24 │ random       │         18 │                  4 │
      └─event───────────────────────────────────────────┴─random─┴─sorted_alias─┴─faster───────┴─difference─┴───difference_times─┘
  ```

  > M(FunctionExecute, "Number of SQL ordinary function calls (SQL functions are called on per-block basis, so this number represents the number of blocks).", ValueType::Number)

  https://github.com/ClickHouse/ClickHouse/blob/9f8540849b2bd5b2981297462208c890ad690321/src/Common/ProfileEvents.cpp#L79

  - random

    ```
    Actions: INPUT : 0 -> random_values Array(String) : 0
      INPUT : 1 -> random_keys Array(String) : 1
      COLUMN Const(String) -> 'Content'_String String : 2
      COLUMN Const(String) -> '%m%'_String String : 3
      FUNCTION indexOf(random_keys :: 1, 'Content'_String :: 2) -> indexOf(__table1.random_keys, 'Content'_String) UInt64 : 4
      FUNCTION arrayElement(random_values :: 0, indexOf(__table1.random_keys, 'Content'_String) :: 4) -> arrayElement(__table1.random_values, indexOf(__table1.random_keys, 'Content'_String)) String : 2
      FUNCTION ilike(arrayElement(__table1.random_values, indexOf(__table1.random_keys, 'Content'_String)) :: 2, '%m%'_String :: 3) -> ilike(arrayElement(__table1.random_values, indexOf(__table1.random_keys, 'Content'_String)), '%m%'_String) UInt8 : 4
    ```
  
  - sorted alias

    ```
    Actions: INPUT : 0 -> sorted_key_value Array(Tuple(String, String)) : 0
      COLUMN Const(Function) -> x Tuple(String, String) -> tupleElement(x, 1_UInt8) Function(Tuple(String, String) -> String) : 1
      COLUMN Const(Function) -> x Tuple(String, String) -> tupleElement(x, 2_UInt8) Function(Tuple(String, String) -> String) : 2
      COLUMN Const(String) -> 'Content'_String String : 3
      COLUMN Const(String) -> '%m%'_String String : 4
      FUNCTION arrayMap(x Tuple(String, String) -> tupleElement(x, 1_UInt8) :: 1, sorted_key_value : 0) -> arrayMap(x Tuple(String, String) -> tupleElement(x, 1_UInt8), sorted_key_value) Array(String) : 5
      FUNCTION arrayMap(x Tuple(String, String) -> tupleElement(x, 2_UInt8) :: 2, sorted_key_value :: 0) -> arrayMap(x Tuple(String, String) -> tupleElement(x, 2_UInt8), sorted_key_value) Array(String) : 1
      FUNCTION indexOfAssumeSorted(arrayMap(x Tuple(String, String) -> tupleElement(x, 1_UInt8), sorted_key_value) :: 5, 'Content'_String :: 3) -> indexOfAssumeSorted(__table1.sorted_keys, 'Content'_String) UInt64 : 0
      FUNCTION arrayElement(arrayMap(x Tuple(String, String) -> tupleElement(x, 2_UInt8), sorted_key_value) :: 1, indexOfAssumeSorted(__table1.sorted_keys, 'Content'_String) :: 0) -> arrayElement(__table1.sorted_values, indexOfAssumeSorted(__table1.sorted_keys, 'Content'_String)) String : 3
      FUNCTION ilike(arrayElement(__table1.sorted_values, indexOfAssumeSorted(__table1.sorted_keys, 'Content'_String)) :: 3, '%m%'_String :: 4) -> ilike(arrayElement(__table1.sorted_values, indexOfAssumeSorted(__table1.sorted_keys, 'Content'_String)), '%m%'_String) UInt8 : 0
    ```

  Each row for sorted alias, it needs to 
  - FUNCTION arrayMap(x Tuple(String, String) -> ...
  - FUNCTION arrayMap(x Tuple(String, String) -> ...

  These two extra steps make it slower.

- Why sorted alias/materialised column take more memory than random order?

- What other settings to use for benchmarking? (Format null/enable_filesystem_cache = 0)
