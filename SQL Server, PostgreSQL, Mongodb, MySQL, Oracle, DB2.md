# SQL Server, PostgreSQL, MongoDB, MySQL, Oracle & DB2 — DBA Terminology

## 1. Core Database Architecture

| # | DBA Concept | SQL Server | PostgreSQL | MongoDB | MySQL | Oracle | DB2 |
|---:|---|---|---|---|---|---|---|
| 1 | Database engine | Database Engine | PostgreSQL Server | `mongod` | `mysqld` | Oracle Database | Db2 DB Manager |
| 2 | Server process | SQL Server Service | `postgres` | `mongod` | `mysqld` | Oracle Instance | Db2 Instance |
| 3 | Instance | SQL Instance | PG Cluster / Instance | `mongod` Instance | MySQL Instance | Oracle Instance | Db2 Instance |
| 4 | Database | Database | Database | Database | Database / Schema | Database | Database |
| 5 | Schema | Schema | Schema | Collection namespace | Database / Schema | User namespace | Schema |
| 6 | Main config | `sp_configure` | `postgresql.conf` | `mongod.conf` | `my.cnf` / `my.ini` | SPFILE / PFILE | DBM / DB CFG |
| 7 | Connection | TCP / Named Pipes | TCP / Unix socket | TCP | TCP / Unix socket | Listener / Service | TCP/IP |
| 8 | Default port | 1433 | 5432 | 27017 | 3306 | 1521 | 50000* |
| 9 | Metadata | System catalogs | System catalogs | System collections | `information_schema` | Data Dictionary | System Catalog |
| 10 | Namespace | DB.Schema.Object | DB.Schema.Object | DB.Collection | DB.Table | Schema.Object | Schema.Object |

\* Db2 port is configurable; 50000 is commonly used.

## 2. Storage Terminology

| # | DBA Concept | SQL Server | PostgreSQL | MongoDB | MySQL | Oracle | DB2 |
|---:|---|---|---|---|---|---|---|
| 1 | Data file | MDF / NDF | Relation files | WiredTiger files | InnoDB files | Datafile | Container |
| 2 | Primary file | MDF | N/A | N/A | System tablespace | Datafile | Container |
| 3 | Secondary file | NDF | N/A | N/A | Tablespace file | Datafile | Container |
| 4 | Transaction log | LDF / T-Log | WAL | Journal | Redo Log | Redo Log | Transaction Log |
| 5 | Change log | T-Log | WAL | Oplog | Binary Log | Redo / Archive | Logs |
| 6 | Logical storage | Filegroup | Tablespace | Collection | Tablespace | Tablespace | Tablespace |
| 7 | Page / block | 8 KB Page | Usually 8 KB | WT Page | Usually 16 KB | DB Block | Page |
| 8 | Extent | 64 KB | Internal | Internal | Extent | Extent | Extent |
| 9 | Temp storage | `tempdb` | Temp files | Temp storage | Temp tablespace | TEMP | Temp tablespace |
| 10 | Large object | LOB | TOAST | BSON / GridFS | BLOB / TEXT | LOB | LOB |
| 11 | Compression | Row / Page | TOAST / methods | WiredTiger | InnoDB | Advanced Comp. | Row / Adaptive |

## 3. Database Objects

| # | DBA Concept | SQL Server | PostgreSQL | MongoDB | MySQL | Oracle | DB2 |
|---:|---|---|---|---|---|---|---|
| 1 | Table | Table | Table | Collection | Table | Table | Table |
| 2 | Row | Row | Row / Tuple | Document | Row | Row | Row |
| 3 | Column | Column | Column | Field | Column | Column | Column |
| 4 | Primary ID | PK | PK | `_id` | PK | PK | PK |
| 5 | View | View | View | View | View | View | View |
| 6 | Materialized view | Indexed View | Materialized View | Materialized pattern | No native MV | Materialized View | MQT |
| 7 | Index | Index | Index | Index | Index | Index | Index |
| 8 | Procedure | Procedure | Procedure | App/server logic | Procedure | Procedure | Procedure |
| 9 | Function | Function | Function | Limited JS* | Function | Function | Function |
| 10 | Trigger | Trigger | Trigger | Change Stream* | Trigger | Trigger | Trigger |
| 11 | Sequence | Sequence | Sequence | Counter pattern | AUTO_INCREMENT* | Sequence | Sequence |
| 12 | Identity | IDENTITY | Identity / Seq. | `_id` | AUTO_INCREMENT | Identity / Seq. | Identity |
| 13 | Synonym | Synonym | No direct core eq. | N/A | N/A | Synonym | Alias |
| 14 | Constraint | Constraint | Constraint | Validation | Constraint | Constraint | Constraint |

\* Not a direct relational equivalent.

## 4. Keys and Constraints

| # | Term | Meaning |
|---:|---|---|
| 1 | Primary Key | Uniquely identifies a row |
| 2 | Foreign Key | Links related records |
| 3 | Unique Key | Prevents duplicate values |
| 4 | Candidate Key | Can uniquely identify a row |
| 5 | Composite Key | Key using multiple columns |
| 6 | Natural Key | Business value used as a key |
| 7 | Surrogate Key | Artificial/generated identifier |
| 8 | CHECK | Enforces a condition |
| 9 | NOT NULL | Prevents NULL values |
| 10 | DEFAULT | Supplies a default value |
| 11 | Referential Integrity | Keeps relationships valid |
| 12 | Cascade Delete | Deletes dependent records |
| 13 | Cascade Update | Updates related key values |
| 14 | MongoDB `_id` | Unique document identifier |

## 5. Transaction Terminology

| # | DBA Concept | SQL Server | PostgreSQL | MongoDB | MySQL | Oracle | DB2 |
|---:|---|---|---|---|---|---|---|
| 1 | Transaction | Transaction | Transaction | Transaction | Transaction | Transaction | Unit of Work |
| 2 | Commit | COMMIT | COMMIT | Commit txn | COMMIT | COMMIT | COMMIT |
| 3 | Rollback | ROLLBACK | ROLLBACK | Abort txn | ROLLBACK | ROLLBACK | ROLLBACK |
| 4 | Savepoint | SAVE TRAN | SAVEPOINT | No direct eq. | SAVEPOINT | SAVEPOINT | SAVEPOINT |
| 5 | Concurrency | Locks + versions | MVCC | WiredTiger | MVCC | Read consistency | Locks / versions |
| 6 | Deadlock | Deadlock | Deadlock | Write conflict | Deadlock | Deadlock | Deadlock |
| 7 | Isolation | Isolation level | Isolation level | Read/write concern | Isolation level | Isolation level | Isolation level |
| 8 | Undo | Version/log mechanisms | MVCC tuples | WT transactions | Undo Log | Undo | Logs/versioning |
| 9 | Redo | T-Log | WAL | Journal | Redo Log | Redo Log | T-Log |

## 6. ACID Terminology

| # | Term | Meaning | DBA Importance |
|---:|---|---|---|
| 1 | Atomicity | All or nothing | Prevents partial updates |
| 2 | Consistency | Data remains valid | Protects integrity |
| 3 | Isolation | Transactions do not improperly interfere | Controls concurrency |
| 4 | Durability | Committed data survives failure | Supports recovery |

## 7. Isolation Levels

| # | Level | SQL Server | PostgreSQL | MongoDB | MySQL | Oracle | DB2 |
|---:|---|---|---|---|---|---|---|
| 1 | Read Uncommitted | Yes | Acts as RC | Read concern model | Yes | No direct eq. | UR |
| 2 | Read Committed | Yes | Default | Read concern model | Yes | Default | CS* |
| 3 | Repeatable Read | Yes | Yes | Snapshot txn model | Default InnoDB | No direct ANSI setting | RS |
| 4 | Serializable | Yes | Yes | Transaction semantics | Yes | Yes | RR* |
| 5 | Snapshot/versioned | Snapshot / RCSI | MVCC | Snapshot reads | Consistent read | Read consistency | Currently Committed |

\* Db2 isolation names and semantics are not exact ANSI name-for-name matches.

## 8. Locking Terminology

| # | Term | Purpose |
|---:|---|---|
| 1 | Shared Lock | Allows compatible reads |
| 2 | Exclusive Lock | Protects modified data |
| 3 | Update Lock | SQL Server update-path lock |
| 4 | Intent Lock | Signals lower-level locks |
| 5 | Row Lock | Locks individual row |
| 6 | Page Lock | Locks database page |
| 7 | Table Lock | Locks whole table |
| 8 | Schema Lock | Protects object definition |
| 9 | Metadata Lock | Protects metadata / DDL |
| 10 | Lock Escalation | Converts many small locks to a larger lock |
| 11 | Blocking | Session waits on another |
| 12 | Deadlock | Cyclic session dependency |
| 13 | Deadlock Victim | Transaction selected for rollback |
| 14 | Lock Timeout | Maximum lock wait |
| 15 | Latch | Lightweight internal synchronization |
| 16 | Mutex | Mutual-exclusion mechanism |

## 9. MVCC / Versioning

| # | SQL Server | PostgreSQL | MongoDB | MySQL | Oracle | DB2 |
|---:|---|---|---|---|---|---|
| 1 | Row Versioning | MVCC | WiredTiger concurrency | InnoDB MVCC | Read Consistency | Currently Committed |
| 2 | Version Store | Tuple Versions | Snapshot transactions | Undo Logs | Undo | Versioning features |
| 3 | Snapshot / RCSI | `xmin` / `xmax` | Snapshot reads | Read View | Consistent reads | Lock/version model |

**MVCC:** Multi-Version Concurrency Control.

## 10. Logging Terminology

| # | DBA Concept | SQL Server | PostgreSQL | MongoDB | MySQL | Oracle | DB2 |
|---:|---|---|---|---|---|---|---|
| 1 | Transaction log | T-Log | WAL | Journal | Redo Log | Redo Log | T-Log |
| 2 | Replication log | T-Log / CDC | WAL | Oplog | Binlog | Redo / Archive | Logs |
| 3 | Archived logs | Log backups | WAL Archive | Oplog retention | Binlogs | Archive Logs | Archive Logs |
| 4 | Log sequence | LSN | LSN | Optime | Position / GTID | SCN | LSN |
| 5 | Log shipping | Log Shipping | WAL Shipping | Replica sync | Binlog repl. | Data Guard | HADR |

## 11. Backup Terminology

| # | DBA Concept | SQL Server | PostgreSQL | MongoDB | MySQL | Oracle | DB2 |
|---:|---|---|---|---|---|---|---|
| 1 | Full backup | Full | Base Backup | Backup / Snapshot | Full | Full | Full |
| 2 | Incremental | Differential | Tool/version dependent | Tool dependent | Tool dependent | RMAN Incremental | Incremental |
| 3 | Log backup | T-Log Backup | WAL Archive | Oplog-based | Binlog | Archive Log | Archive Log |
| 4 | Physical backup | Native backup | `pg_basebackup` | Snapshot | Physical backup | RMAN | Backup image |
| 5 | Logical backup | BACPAC / scripts | `pg_dump` | `mongodump` | `mysqldump` | Data Pump | `db2move` / export |
| 6 | Restore | RESTORE | Base restore | `mongorestore` | Restore | RMAN RESTORE | RESTORE |
| 7 | PITR | Yes | Yes | With backup tooling | Yes | Yes | Rollforward |

## 12. Recovery Terminology

| # | Term | Meaning |
|---:|---|---|
| 1 | RPO | Acceptable data-loss window |
| 2 | RTO | Acceptable recovery duration |
| 3 | PITR | Point-in-Time Recovery |
| 4 | Crash Recovery | Recovery after abnormal shutdown |
| 5 | Roll Forward | Reapply logged changes |
| 6 | Rollback | Undo changes |
| 7 | Redo | Reapply changes |
| 8 | Undo | Reverse changes |
| 9 | Recovery Log | Log used for recovery |
| 10 | Checkpoint | Establishes recovery progress |
| 11 | Restore | Return data from backup |
| 12 | Recovery | Make restored DB consistent/usable |
| 13 | Failover | Move workload to another node |
| 14 | Failback | Return workload to preferred node |

## 13. High Availability / Disaster Recovery

| # | DBA Concept | SQL Server | PostgreSQL | MongoDB | MySQL | Oracle | DB2 |
|---:|---|---|---|---|---|---|---|
| 1 | HA technology | AG / FCI | Streaming Repl. + HA | Replica Set | InnoDB Cluster | RAC / Data Guard | HADR / pureScale |
| 2 | Primary | Primary Replica | Primary | Primary | Primary | Primary | Primary |
| 3 | Secondary | Secondary Replica | Standby | Secondary | Secondary | Standby | Standby |
| 4 | Replication | AG / Replication | Streaming / Logical | Replica Set | Group / Async | Data Guard | HADR |
| 5 | Cluster | WSFC | External HA stack | Sharded / Replica Set | InnoDB Cluster | RAC | pureScale |
| 6 | Failover | Auto / Manual | HA-tool dependent | Election | Cluster mechanisms | RAC / DG | HADR |
| 7 | Read replica | Readable Secondary | Hot Standby | Secondary reads | Read Replica | Active Data Guard | Standby-read varies |

## 14. Replication Terminology

| # | Term | Meaning |
|---:|---|---|
| 1 | Primary | Main writable node |
| 2 | Secondary | Replicated copy |
| 3 | Publisher | Replication source |
| 4 | Subscriber | Replication receiver |
| 5 | Distributor | SQL Server replication intermediary |
| 6 | Physical Replication | Copies physical/log changes |
| 7 | Logical Replication | Replicates logical data changes |
| 8 | Synchronous | Commit waits per acknowledgement policy |
| 9 | Asynchronous | Primary does not wait for remote durable ack |
| 10 | Replication Lag | Replica is behind primary |
| 11 | Log Shipping | Ships transaction/WAL logs |
| 12 | CDC | Change Data Capture |
| 13 | Change Tracking | Tracks changed keys/rows |
| 14 | Oplog | MongoDB operations log |
| 15 | WAL | Write-Ahead Log |
| 16 | Binlog | MySQL Binary Log |
| 17 | Redo | Oracle change log |
| 18 | LSN | Log Sequence Number |
| 19 | SCN | Oracle System Change Number |
| 20 | GTID | Global Transaction Identifier |

## 15. Index Terminology

| # | DBA Concept | SQL Server | PostgreSQL | MongoDB | MySQL | Oracle | DB2 |
|---:|---|---|---|---|---|---|---|
| 1 | Main structure | Clustered often | Heap default | Collection + indexes | Clustered PK | Heap normally | Table + indexes |
| 2 | Secondary | Nonclustered | B-tree etc. | Secondary | Secondary | B-tree | Index |
| 3 | Unique | Unique Index | Unique Index | Unique Index | Unique Index | Unique Index | Unique Index |
| 4 | Composite | Composite | Multicolumn | Compound | Composite | Composite | Composite |
| 5 | Filtered/partial | Filtered | Partial | Partial | Alternatives | Conditional methods | Varies |
| 6 | Covering | INCLUDE | INCLUDE | Covered Query | Covering Index | Index-only access | Index-only access |
| 7 | Full text | Full-Text | FTS | Text / Search | FULLTEXT | Oracle Text | Text Search |
| 8 | Spatial | Spatial Index | GiST/PostGIS | Geospatial | SPATIAL | Spatial Index | Spatial features |

## 16. Index Problems

| # | Term | Meaning |
|---:|---|---|
| 1 | Fragmentation | Less optimal index organization |
| 2 | Bloat | Unnecessary table/index space |
| 3 | Index Rebuild | Recreate index |
| 4 | Index Reorganize | Lighter SQL Server maintenance |
| 5 | REINDEX | PostgreSQL index rebuild |
| 6 | Missing Index | Useful index may be absent |
| 7 | Unused Index | Index has little/no workload value |
| 8 | Duplicate Index | Indexes overlap |
| 9 | Selectivity | Ability to narrow rows |
| 10 | Cardinality | Distinct-value count/distribution |
| 11 | Index Scan | Broad index read |
| 12 | Index Seek | Targeted index access |
| 13 | Table Scan | Broad table read |
| 14 | Bitmap Scan | Bitmap-based access |
| 15 | Index-Only Scan | Query served mainly from index |

## 17. Query Processing Terminology

| # | Term | Meaning |
|---:|---|---|
| 1 | Parser | Parses/validates SQL |
| 2 | Optimizer | Selects execution strategy |
| 3 | Execution Plan | Operations used to run query |
| 4 | Estimated Plan | Predicted plan |
| 5 | Actual Plan | Runtime plan information |
| 6 | Cost | Relative optimizer estimate |
| 7 | Cardinality Estimate | Estimated rows |
| 8 | Predicate | Filter condition |
| 9 | Join | Combines data sets |
| 10 | Nested Loop | Join strategy |
| 11 | Hash Join | Hash-based join |
| 12 | Merge Join | Sorted-input join |
| 13 | Sort | Orders rows |
| 14 | Aggregate | Calculates summaries/groups |
| 15 | Scan | Broad read |
| 16 | Seek | Targeted access |
| 17 | Spool | Temporary intermediate storage |
| 18 | Parallelism | Multiple workers/threads |
| 19 | Plan Cache | Cached execution plans |
| 20 | Parameter Sniffing | SQL Server compile-value plan behavior |
| 21 | Statistics | Optimizer distribution metadata |

## 18. Statistics Terminology

| # | DBA Concept | SQL Server | PostgreSQL | MongoDB | MySQL | Oracle | DB2 |
|---:|---|---|---|---|---|---|---|
| 1 | Statistics | Statistics | Statistics | Planner stats | Statistics | Optimizer Stats | Statistics |
| 2 | Update stats | UPDATE STATISTICS | ANALYZE | Internal | ANALYZE TABLE | `DBMS_STATS` | RUNSTATS |
| 3 | Auto stats | Auto Stats | Autoanalyze | Automatic | Persistent stats | Auto stats jobs | Auto features |
| 4 | Histogram | Histogram | Histogram | Planner metadata | Histogram | Histogram | Distribution Stats |
| 5 | Cardinality | CE | Planner | Query Planner | Optimizer | CBO | Optimizer |

## 19. Performance Terminology

| # | Term | DBA Focus |
|---:|---|---|
| 1 | CPU Utilization | Processor pressure |
| 2 | Memory Pressure | Available-memory pressure |
| 3 | Disk Latency | Storage response time |
| 4 | IOPS | I/O operations/sec |
| 5 | Throughput | Data processed/sec |
| 6 | Network Latency | Network delay |
| 7 | Query Duration | SQL execution time |
| 8 | Wait Time | Resource wait time |
| 9 | Blocking | Lock waits |
| 10 | Deadlocks | Cyclic locking |
| 11 | Cache Hit Ratio | Reads served from cache |
| 12 | Buffer Cache | Cached DB pages |
| 13 | Plan Cache | Cached query plans |
| 14 | Temp Usage | Temporary workspace |
| 15 | Connections | Active/open connections |
| 16 | TPS | Transactions/sec |
| 17 | QPS | Queries/sec |
| 18 | Batch Requests/sec | SQL Server workload rate |

## 20. Memory Terminology

| # | DBA Concept | SQL Server | PostgreSQL | MongoDB | MySQL | Oracle | DB2 |
|---:|---|---|---|---|---|---|---|
| 1 | Data cache | Buffer Pool | Shared Buffers | WT Cache | InnoDB Buffer Pool | Buffer Cache | Bufferpools |
| 2 | Plan cache | Plan Cache | Prepared-plan model | Plan Cache | Statement caches | Shared Pool | Package Cache |
| 3 | Query memory | Memory Grants | `work_mem` | Working memory | Sort/Join buffers | PGA | Sort heaps |
| 4 | Server memory | Max Server Memory | Shared + process | WT + process | Server memory | SGA | Instance/DB memory |
| 5 | Session memory | Session/query memory | `work_mem` etc. | Operation memory | Per-thread buffers | PGA | Agent/app memory |

## 21. SQL Server-Specific DBA Terms

| # | Term | Purpose |
|---:|---|---|
| 1 | MDF | Primary data file |
| 2 | NDF | Secondary data file |
| 3 | LDF | Transaction log file |
| 4 | Filegroup | Logical data-file grouping |
| 5 | tempdb | Temporary/system workspace |
| 6 | master | Core system database |
| 7 | model | Database template |
| 8 | msdb | Agent/backup operational metadata |
| 9 | Resource DB | Read-only system objects |
| 10 | SQL Agent | Job scheduler |
| 11 | SSMS | Administration GUI |
| 12 | T-SQL | SQL Server SQL dialect |
| 13 | DMVs | Dynamic Management Views |
| 14 | Extended Events | Diagnostics framework |
| 15 | Query Store | Query/performance history |
| 16 | Always On AG | Availability Groups |
| 17 | FCI | Failover Cluster Instance |
| 18 | Log Shipping | DR/log-copy technology |
| 19 | Replication | Data distribution |
| 20 | RCSI | Read Committed Snapshot Isolation |
| 21 | DBCC | Maintenance/consistency commands |
| 22 | CHECKDB | Integrity checking |
| 23 | SPID | Session/process ID |
| 24 | Wait Stats | Wait diagnostics |

## 22. PostgreSQL-Specific DBA Terms

| # | Term | Purpose |
|---:|---|---|
| 1 | Cluster | Databases managed by one server |
| 2 | `postgres` | Main server process |
| 3 | WAL | Write-Ahead Log |
| 4 | `pg_wal` | WAL directory |
| 5 | VACUUM | MVCC cleanup/reuse |
| 6 | VACUUM FULL | Table rewrite/space reclaim |
| 7 | Autovacuum | Automatic vacuum/analyze |
| 8 | ANALYZE | Planner statistics |
| 9 | Dead Tuple | Obsolete row version |
| 10 | Bloat | Excess table/index space |
| 11 | TOAST | Large-value storage |
| 12 | Tablespace | Alternate storage location |
| 13 | `pg_hba.conf` | Client authentication |
| 14 | `postgresql.conf` | Main configuration |
| 15 | `pg_stat_activity` | Session monitoring |
| 16 | `pg_stat_statements` | Query statistics |
| 17 | Streaming Replication | Physical replication |
| 18 | Logical Replication | Logical pub/sub |
| 19 | Hot Standby | Read-capable standby |
| 20 | `pg_basebackup` | Physical base backup |
| 21 | `pg_dump` | Logical backup |
| 22 | `pg_restore` | Logical restore |

## 23. MongoDB-Specific DBA Terms

| # | Term | Purpose |
|---:|---|---|
| 1 | Document | BSON record |
| 2 | Collection | Group of documents |
| 3 | BSON | Binary JSON format |
| 4 | `_id` | Unique document ID |
| 5 | `mongod` | Database server |
| 6 | `mongos` | Sharded-cluster router |
| 7 | Replica Set | HA/replication group |
| 8 | Primary | Writable member |
| 9 | Secondary | Replicated member |
| 10 | Arbiter | Voting non-data member |
| 11 | Oplog | Replication operations log |
| 12 | WiredTiger | Storage engine |
| 13 | Journal | Durability/recovery log |
| 14 | Sharding | Horizontal distribution |
| 15 | Shard | Data-bearing partition |
| 16 | Config Server | Sharded-cluster metadata |
| 17 | Shard Key | Data distribution key |
| 18 | Chunk / Range | Sharded data range |
| 19 | Balancer | Redistributes ranges |
| 20 | Read Preference | Read-member selection |
| 21 | Read Concern | Read consistency |
| 22 | Write Concern | Write acknowledgement |
| 23 | Change Stream | Watches data changes |
| 24 | GridFS | Large-file storage convention |

## 24. MySQL-Specific DBA Terms

| # | Term | Purpose |
|---:|---|---|
| 1 | `mysqld` | Database server |
| 2 | InnoDB | Default transactional engine |
| 3 | MyISAM | Older nontransactional engine |
| 4 | Buffer Pool | InnoDB data cache |
| 5 | Redo Log | Crash recovery |
| 6 | Undo Log | MVCC/rollback |
| 7 | Binary Log | Replication/PITR |
| 8 | Relay Log | Replica event log |
| 9 | GTID | Global Transaction ID |
| 10 | AUTO_INCREMENT | Generated numeric ID |
| 11 | `my.cnf` | Unix/Linux config |
| 12 | `my.ini` | Windows config |
| 13 | Performance Schema | Performance instrumentation |
| 14 | `information_schema` | Metadata views |
| 15 | Group Replication | HA replication |
| 16 | InnoDB Cluster | HA solution |
| 17 | `mysqldump` | Logical backup |
| 18 | Slow Query Log | Slow SQL diagnostics |

## 25. Oracle-Specific DBA Terms

| # | Term | Purpose |
|---:|---|---|
| 1 | Instance | SGA + background processes |
| 2 | Database | Physical database files |
| 3 | SID | Instance identifier |
| 4 | Service Name | Client connection service |
| 5 | SGA | Shared memory |
| 6 | PGA | Process/session memory |
| 7 | Shared Pool | SQL/dictionary cache |
| 8 | Buffer Cache | Data-block cache |
| 9 | Redo Log | Change records |
| 10 | Archive Log | Archived redo |
| 11 | Control File | Structural metadata |
| 12 | Datafile | Physical data storage |
| 13 | Tablespace | Logical storage |
| 14 | Undo Tablespace | Undo/read consistency |
| 15 | TEMP | Temporary operations |
| 16 | SCN | System Change Number |
| 17 | RMAN | Backup/recovery manager |
| 18 | Data Pump | Logical import/export |
| 19 | Data Guard | Standby/DR |
| 20 | RAC | Multi-instance cluster |
| 21 | ASM | Storage management |
| 22 | AWR | Workload repository |
| 23 | ASH | Active Session History |
| 24 | ADDM | Performance diagnostics |
| 25 | CDB | Container Database |
| 26 | PDB | Pluggable Database |
| 27 | Listener | Network listener |
| 28 | TNS | Oracle networking |

## 26. IBM DB2-Specific DBA Terms

| # | Term | Purpose |
|---:|---|---|
| 1 | Db2 Instance | DB manager environment |
| 2 | Database | Database |
| 3 | Tablespace | Logical storage |
| 4 | Bufferpool | Memory cache |
| 5 | Container | Physical tablespace storage |
| 6 | DBM CFG | DB Manager configuration |
| 7 | DB CFG | Database configuration |
| 8 | Transaction Log | Recovery log |
| 9 | Active Log | Active transaction logs |
| 10 | Archive Log | Archived logs |
| 11 | HADR | HA/DR |
| 12 | pureScale | Clustered scale/HA |
| 13 | RUNSTATS | Update statistics |
| 14 | REORG | Reorganize data/indexes |
| 15 | REORGCHK | Reorganization analysis |
| 16 | BIND | Create package |
| 17 | REBIND | Recreate package |
| 18 | Package Cache | Cached SQL/packages |
| 19 | Explain | Execution-plan analysis |
| 20 | Locklist | Lock memory configuration |
| 21 | Unit of Work | Transaction |
| 22 | Rollforward | Log-based recovery |

## 27. Security Terminology

| # | DBA Concept | SQL Server | PostgreSQL | MongoDB | MySQL | Oracle | DB2 |
|---:|---|---|---|---|---|---|---|
| 1 | Identity | Login | Role / User | User | User | User | User / external |
| 2 | Role | Server/DB Role | Role | Role | Role | Role | Role / Authority |
| 3 | Grant | GRANT | GRANT | Role/Privilege | GRANT | GRANT | GRANT |
| 4 | Revoke | REVOKE | REVOKE | Revoke role | REVOKE | REVOKE | REVOKE |
| 5 | Transit encryption | TLS | SSL/TLS | TLS | TLS | TLS / NNE | TLS |
| 6 | At-rest encryption | TDE | Storage/extensions | Encryption at Rest | Enterprise/storage | TDE | Native/storage |
| 7 | Auditing | SQL Audit | Logging/extensions | Auditing | Audit plugin | Unified Audit | Audit facility |

## 28. Common Security Terms

| # | Term | Meaning |
|---:|---|---|
| 1 | Authentication | Who are you? |
| 2 | Authorization | What can you do? |
| 3 | RBAC | Role-Based Access Control |
| 4 | Least Privilege | Only required permissions |
| 5 | Encryption at Rest | Encrypt stored data |
| 6 | Encryption in Transit | Encrypt network traffic |
| 7 | TLS | Secure transport protocol |
| 8 | Certificate | Cryptographic identity |
| 9 | Key Management | Encryption-key management |
| 10 | Data Masking | Hide sensitive values |
| 11 | Dynamic Masking | Mask query output |
| 12 | Auditing | Track activity |
| 13 | Service Account | DB service identity |
| 14 | Privileged Account | Administrative identity |
| 15 | Secret | Password/token/key |
| 16 | Rotation | Replace credentials/keys |
| 17 | SQL Injection | Unsafe-input SQL attack |

## 29. DBA Maintenance Terminology

| # | DBA Task | SQL Server | PostgreSQL | MongoDB | MySQL | Oracle | DB2 |
|---:|---|---|---|---|---|---|---|
| 1 | Integrity | CHECKDB | Checks/tooling | Validation | CHECK TABLE | DBVERIFY/RMAN | INSPECT |
| 2 | Statistics | UPDATE STATISTICS | ANALYZE | Planner mechanisms | ANALYZE TABLE | DBMS_STATS | RUNSTATS |
| 3 | Index maint. | REBUILD/REORG | REINDEX | Rebuild if needed | ALTER/OPTIMIZE | INDEX REBUILD | REORG |
| 4 | Space maint. | File management | VACUUM | Compact/resync* | OPTIMIZE* | Segment mgmt. | REORG |
| 5 | Scheduler | SQL Agent | cron / pg_cron | External / Atlas | Event Scheduler | DBMS_SCHEDULER | Admin Task Scheduler |
| 6 | Backup validation | VERIFY + restore test | Restore test | Restore test | Restore test | RMAN VALIDATE | Restore test |

## 30. Monitoring / Troubleshooting Terms

| # | Term | DBA Question |
|---:|---|---|
| 1 | Session | Who is connected? |
| 2 | Active Session | Who is working now? |
| 3 | Long Query | What is running too long? |
| 4 | Blocking Session | Who is blocking? |
| 5 | Blocked Session | Who is waiting? |
| 6 | Deadlock | Which transactions conflict? |
| 7 | Wait Event | What is the session waiting on? |
| 8 | CPU Pressure | Is CPU saturated? |
| 9 | Memory Pressure | Is memory constrained? |
| 10 | I/O Pressure | Is storage the bottleneck? |
| 11 | Log Growth | Is the log growing unexpectedly? |
| 12 | Replication Lag | How far behind is replica? |
| 13 | Connection Saturation | Are connection limits reached? |
| 14 | Disk Full | Is storage exhausted? |
| 15 | Slow Query | Which query is expensive? |
| 16 | Plan Regression | Did the plan become worse? |
| 17 | Cache Miss | Is data missing from cache? |
| 18 | Timeout | Did operation exceed wait limit? |
| 19 | Error Log | What errors/warnings occurred? |

## 31. Capacity Planning Terms

| # | Term | Meaning |
|---:|---|---|
| 1 | Database Size | Allocated/used storage |
| 2 | Data Growth | Data growth rate |
| 3 | Log Growth | Log growth rate |
| 4 | Free Space | Remaining capacity |
| 5 | Autogrowth | Automatic file expansion |
| 6 | Max Size | Configured storage limit |
| 7 | Disk Capacity | Total storage capacity |
| 8 | IOPS | I/O operations/sec |
| 9 | Throughput | Data transfer rate |
| 10 | Latency | I/O response time |
| 11 | CPU Cores | Processing capacity |
| 12 | RAM | Memory capacity |
| 13 | Connections | Concurrent connections |
| 14 | TPS | Transactions/sec |
| 15 | QPS | Queries/sec |
| 16 | Peak Load | Maximum workload |
| 17 | Baseline | Normal performance reference |
| 18 | Growth Forecast | Expected future capacity |

## 32. Data Modeling Terminology

| # | Term | Meaning |
|---:|---|---|
| 1 | Entity | Business object |
| 2 | Attribute | Property / column |
| 3 | Relationship | Entity connection |
| 4 | One-to-One | 1:1 |
| 5 | One-to-Many | 1:N |
| 6 | Many-to-Many | M:N |
| 7 | Normalization | Reduce redundancy |
| 8 | Denormalization | Add redundancy for access/performance |
| 9 | 1NF | First Normal Form |
| 10 | 2NF | Second Normal Form |
| 11 | 3NF | Third Normal Form |
| 12 | BCNF | Boyce-Codd Normal Form |
| 13 | Star Schema | Dimensional model |
| 14 | Snowflake Schema | Normalized dimensional model |
| 15 | Fact Table | Measures/events |
| 16 | Dimension Table | Descriptive attributes |
| 17 | OLTP | Transaction workload |
| 18 | OLAP | Analytical workload |
| 19 | Data Warehouse | Analytical platform |
| 20 | Data Mart | Subject-specific analytics store |

## 33. Partitioning and Sharding

| # | Term | Meaning |
|---:|---|---|
| 1 | Partitioning | Split logical table into pieces |
| 2 | Horizontal Partitioning | Split rows |
| 3 | Vertical Partitioning | Split columns |
| 4 | Range Partitioning | Split by value range |
| 5 | List Partitioning | Split by value list |
| 6 | Hash Partitioning | Hash selects partition |
| 7 | Partition Key | Determines partition |
| 8 | Partition Pruning | Skip irrelevant partitions |
| 9 | Sharding | Distribute data across servers |
| 10 | Shard Key | Determines shard |
| 11 | Resharding | Change shard distribution |
| 12 | Hot Partition | Overloaded partition |
| 13 | Hot Shard | Overloaded shard |

## 34. Migration / Upgrade Terminology

| # | Term | Meaning |
|---:|---|---|
| 1 | Migration | Move DB/workload |
| 2 | Upgrade | Move to newer version |
| 3 | In-place Upgrade | Upgrade existing installation |
| 4 | Side-by-Side | Build new and migrate |
| 5 | Homogeneous | Same DB technology |
| 6 | Heterogeneous | Different DB engines |
| 7 | Schema Migration | Move/convert structures |
| 8 | Data Migration | Move records |
| 9 | Cutover | Switch production traffic |
| 10 | Rollback Plan | Return to old environment |
| 11 | Downtime | Application unavailable period |
| 12 | Minimal Downtime | Very short interruption |
| 13 | Compatibility | Feature/app compatibility |
| 14 | Validation | Confirm migrated data |
| 15 | Reconciliation | Compare source/target |
| 16 | CDC Migration | Capture ongoing changes |

## 35. Cloud DBA Terminology

| # | Term | Meaning |
|---:|---|---|
| 1 | IaaS | Infrastructure as a Service |
| 2 | PaaS | Platform as a Service |
| 3 | DBaaS | Database as a Service |
| 4 | Managed DB | Provider handles DBA operations |
| 5 | Read Replica | Read-only replica |
| 6 | Multi-AZ | Multiple availability zones |
| 7 | Region | Geographic cloud area |
| 8 | Availability Zone | Independent datacenter grouping |
| 9 | Snapshot | Point-in-time storage copy |
| 10 | Automated Backup | Scheduled platform backup |
| 11 | PITR | Point-in-time restore |
| 12 | Scale Up | Increase CPU/RAM |
| 13 | Scale Out | Add nodes |
| 14 | Serverless DB | Dynamic managed capacity |
| 15 | Provisioned IOPS | Configured storage performance |
| 16 | Encryption Key | Data-encryption key |
| 17 | KMS | Key Management Service |
| 18 | Private Endpoint | Private DB network access |

## 36. Core DBA Acronyms

| # | Acronym | Full Form |
|---:|---|---|
| 1 | DBA | Database Administrator |
| 2 | DBMS | Database Management System |
| 3 | RDBMS | Relational Database Management System |
| 4 | SQL | Structured Query Language |
| 5 | DDL | Data Definition Language |
| 6 | DML | Data Manipulation Language |
| 7 | DCL | Data Control Language |
| 8 | TCL | Transaction Control Language |
| 9 | ACID | Atomicity, Consistency, Isolation, Durability |
| 10 | MVCC | Multi-Version Concurrency Control |
| 11 | WAL | Write-Ahead Logging |
| 12 | LSN | Log Sequence Number |
| 13 | SCN | System Change Number |
| 14 | GTID | Global Transaction Identifier |
| 15 | CDC | Change Data Capture |
| 16 | HA | High Availability |
| 17 | DR | Disaster Recovery |
| 18 | RPO | Recovery Point Objective |
| 19 | RTO | Recovery Time Objective |
| 20 | PITR | Point-in-Time Recovery |
| 21 | OLTP | Online Transaction Processing |
| 22 | OLAP | Online Analytical Processing |
| 23 | ETL | Extract, Transform, Load |
| 24 | ELT | Extract, Load, Transform |
| 25 | IOPS | Input/Output Operations Per Second |
| 26 | TPS | Transactions Per Second |
| 27 | QPS | Queries Per Second |
| 28 | RBAC | Role-Based Access Control |
| 29 | TLS | Transport Layer Security |
| 30 | TDE | Transparent Data Encryption |
| 31 | PK | Primary Key |
| 32 | FK | Foreign Key |
| 33 | CTE | Common Table Expression |
| 34 | AG | Availability Group |
| 35 | FCI | Failover Cluster Instance |
| 36 | RAC | Real Application Clusters |
| 37 | RMAN | Recovery Manager |
| 38 | ASM | Automatic Storage Management |
| 39 | AWR | Automatic Workload Repository |
| 40 | HADR | High Availability Disaster Recovery |

## 37. End-to-End DBA Mental Model

| # | Area | Flow |
|---:|---|---|
| 1 | Architecture | Instance → Database → Schema → Object |
| 2 | Storage | Disk → Tablespace/Filegroup → File → Extent → Page → Row |
| 3 | Data Objects | Table/Collection → Row/Document → Column/Field → Key → Index |
| 4 | Transactions | BEGIN → Read/Write → Lock/MVCC → Log → COMMIT/ROLLBACK |
| 5 | Security | Authentication → Authorization → Role → Privilege → Audit |
| 6 | Performance | Query → Optimizer → Plan → CPU/Memory/I/O → Tuning |
| 7 | Maintenance | Statistics → Index → Vacuum/Reorg → Integrity → Capacity |
| 8 | Backup | Full/Base → Incremental/Diff → Logs → Restore → PITR |
| 9 | HA/DR | Primary → Replica/Standby → Replication → Failover → Failback |
| 10 | Monitoring | Sessions → Blocking → CPU → Memory → I/O → Logs → Lag |
| 11 | Lifecycle | Install → Configure → Secure → Monitor → Patch → Upgrade → Retire |

---

### GitHub Display Note

The cross-database tables intentionally keep these headers side by side:

**DBA Concept | SQL Server | PostgreSQL | MongoDB | MySQL | Oracle | DB2**

Cell text is kept compact to reduce horizontal width while preserving the comparison format.
