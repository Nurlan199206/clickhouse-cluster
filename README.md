3 nodes clickhouse

* OS: Ubuntu 20.04 LTS

* ClickHouse version: 21.7.4 revision 54449
* zookeeper version: 3.6.3

```
192.168.0.51
192.168.0.52
192.168.0.53
```


3 nodes zookeeper

* CentOS 8

```
192.168.0.54
192.168.0.55
192.168.0.56
```
--------------------------------------------------------------------------------------------------------------------------------
Testing distributed table

first check cluster: 

```select * from system.clusters

SELECT *
FROM system.clusters

Query id: 00f924f7-e9d5-415d-82c7-f306abba0284

┌─cluster────────────────────┬─shard_num─┬─shard_weight─┬─replica_num─┬─host_name────┬─host_address─┬─port─┬─is_local─┬─user────┬─default_database─┬─errors_count─┬─slowdowns_count─┬─estimated_recovery_time─┐
│ perftest_1shards_3replicas │         1 │            1 │           1 │ 192.168.0.51 │ 192.168.0.51 │ 9000 │        1 │ default │                  │            0 │               0 │                       0 │
│ perftest_1shards_3replicas │         2 │            1 │           1 │ 192.168.0.52 │ 192.168.0.52 │ 9000 │        0 │ default │                  │            0 │               0 │                       0 │
│ perftest_1shards_3replicas │         3 │            1 │           1 │ 192.168.0.53 │ 192.168.0.53 │ 9000 │        0 │ default │                  │            0 │               0 │                       0 │
└────────────────────────────┴───────────┴──────────────┴─────────────┴──────────────┴──────────────┴──────┴──────────┴─────────┴──────────────────┴──────────────┴─────────────────┴─────────────────────────┘

3 rows in set. Elapsed: 0.004 sec.
```

1) on all nodes ```create DATABASE test on cluster perftest_1shards_3replicas```


2) on all nodes ```CREATE TABLE test.visits_local
(
    id UInt64,
    duration Float64,
    url String,
    created DateTime
)
ENGINE = MergeTree()
PRIMARY KEY id
ORDER BY id```


3) on all nodes ```CREATE TABLE test.visits_all AS test.visits_local
ENGINE = Distributed('perftest_1shards_3replicas', 'test', visits_local, rand());```


4) insert on 1 & 2 nodes
```INSERT INTO test.visits_all VALUES (1, 10.5, 'http://example.com', '2019-01-01 00:01:01');```


5) run query on node 03 
```
SELECT *
FROM test.visits_all

Query id: 7b001487-3633-4099-9271-76af5c3135f4

┌─id─┬─duration─┬─url────────────────┬─────────────created─┐
│  1 │     10.5 │ http://example.com │ 2019-01-01 00:01:01 │
│  1 │     10.5 │ http://example.com │ 2019-01-01 00:01:01 │
└────┴──────────┴────────────────────┴─────────────────────┘

2 rows in set. Elapsed: 0.016 sec.
```
