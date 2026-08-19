Database Administration terminology is broad, and the same concept often has a different name or implementation in SQL Server, PostgreSQL, MongoDB, MySQL, Oracle, and IBM Db2.

Below is an **end-to-end DBA terminology reference**, organized from architecture → storage → objects → security → transactions → backup/recovery → HA/DR → performance → maintenance → troubleshooting.

## 1. Core Database Architecture

| #  | DBA Concept         | SQL Server              | PostgreSQL                  | MongoDB                      | MySQL                          | Oracle                        | IBM Db2              |
| -- | ------------------- | ----------------------- | --------------------------- | ---------------------------- | ------------------------------ | ----------------------------- | -------------------- |
| 1  | Database engine     | Database Engine         | PostgreSQL Server           | `mongod`                     | `mysqld`                       | Oracle Database               | Db2 Database Manager |
| 2  | Server process      | SQL Server Service      | `postgres`                  | `mongod`                     | `mysqld`                       | Oracle Instance               | Db2 Instance         |
| 3  | Instance            | SQL Server Instance     | PostgreSQL Cluster/Instance | `mongod` Instance            | MySQL Instance                 | Oracle Instance               | Db2 Instance         |
| 4  | Database            | Database                | Database                    | Database                     | Database / Schema              | Database                      | Database             |
| 5  | Schema              | Schema                  | Schema                      | Usually collection namespace | Database/Schema                | Schema = user-owned namespace | Schema               |
| 6  | Main config         | `sp_configure` / config | `postgresql.conf`           | `mongod.conf`                | `my.cnf` / `my.ini`            | SPFILE / PFILE                | DBM / DB CFG         |
| 7  | Connection endpoint | TCP / Named Pipes       | TCP / Unix socket           | TCP                          | TCP / Unix socket              | Listener / Service            | TCP/IP               |
| 8  | Default port        | 1433                    | 5432                        | 27017                        | 3306                           | 1521 listener                 | 50000 commonly       |
| 9  | System metadata     | System catalogs         | System catalogs             | System collections/commands  | `information_schema` / `mysql` | Data Dictionary               | System Catalog       |
| 10 | Namespace           | Database.Schema.Object  | Database.Schema.Object      | Database.Collection          | Database.Table                 | Schema.Object                 | Schema.Object        |

---

# 2. Storage Terminology

| Concept              | SQL Server      | PostgreSQL        | MongoDB                | MySQL                     | Oracle               | Db2                           |
| -------------------- | --------------- | ----------------- | ---------------------- | ------------------------- | -------------------- | ----------------------------- |
| Physical data file   | MDF/NDF         | Relation files    | WiredTiger files       | InnoDB tablespace/files   | Datafile             | Tablespace containers/storage |
| Primary data file    | MDF             | N/A               | N/A                    | N/A                       | Datafile             | Container                     |
| Secondary file       | NDF             | N/A               | N/A                    | N/A                       | Additional datafile  | Container                     |
| Transaction log      | LDF             | WAL               | Journal/Oplog*         | Redo log / Binary log     | Online Redo Log      | Transaction Logs              |
| Storage logical unit | Filegroup       | Tablespace        | Collection             | Tablespace                | Tablespace           | Tablespace                    |
| Page                 | 8 KB Page       | Usually 8 KB Page | WiredTiger Page        | Usually 16 KB InnoDB Page | Database Block       | Page                          |
| Extent               | 8 pages / 64 KB | Extent internally | Internal               | Extent                    | Extent               | Extent                        |
| Data block           | Page            | Page              | Page                   | Page                      | Oracle Block         | Page                          |
| Temporary storage    | `tempdb`        | Temp files/tables | Temp/internal storage  | Temp tablespace           | TEMP tablespace      | Temp tablespace               |
| Large object         | LOB             | TOAST             | BSON document/GridFS   | BLOB/TEXT                 | LOB                  | LOB                           |
| Compression          | Row/Page        | TOAST/compression | WiredTiger compression | InnoDB compression        | Advanced Compression | Row/adaptive compression      |

* MongoDB's oplog is primarily a replication operation log, while journaling supports durability.

---

# 3. Database Object Terminology

| Object            | SQL Server   | PostgreSQL                | MongoDB                                       | MySQL                                                     | Oracle            | Db2        |
| ----------------- | ------------ | ------------------------- | --------------------------------------------- | --------------------------------------------------------- | ----------------- | ---------- |
| Table             | Table        | Table                     | Collection equivalent                         | Table                                                     | Table             | Table      |
| Row               | Row          | Row/Tuple                 | Document                                      | Row                                                       | Row               | Row        |
| Column            | Column       | Column                    | Field                                         | Column                                                    | Column            | Column     |
| Record identifier | PK           | PK                        | `_id`                                         | PK                                                        | PK                | PK         |
| View              | View         | View                      | View                                          | View                                                      | View              | View       |
| Materialized view | Indexed View | Materialized View         | Materialized pattern / aggregation output     | Limited/native alternatives                               | Materialized View | MQT        |
| Index             | Index        | Index                     | Index                                         | Index                                                     | Index             | Index      |
| Stored procedure  | Procedure    | Procedure                 | Server-side patterns/functions differ         | Procedure                                                 | Procedure         | Procedure  |
| Function          | Function     | Function                  | JS functions limited/not equivalent           | Function                                                  | Function          | Function   |
| Trigger           | Trigger      | Trigger                   | Change Streams often used for event reactions | Trigger                                                   | Trigger           | Trigger    |
| Sequence          | Sequence     | Sequence                  | Usually counters/application logic            | Sequence in newer variants differs; AUTO_INCREMENT common | Sequence          | Sequence   |
| Identity          | IDENTITY     | Identity/Sequence         | `_id`                                         | AUTO_INCREMENT                                            | Identity/Sequence | Identity   |
| Synonym           | Synonym      | No direct core equivalent | N/A                                           | N/A                                                       | Synonym           | Alias      |
| Constraint        | Constraint   | Constraint                | Schema validation                             | Constraint                                                | Constraint        | Constraint |

---

# 4. Keys and Constraints

| Term                  | Meaning                                         |
| --------------------- | ----------------------------------------------- |
| Primary Key           | Uniquely identifies a row                       |
| Foreign Key           | Links records between tables                    |
| Unique Key            | Prevents duplicate values                       |
| Candidate Key         | Column(s) capable of uniquely identifying a row |
| Composite Key         | Key containing multiple columns                 |
| Natural Key           | Business value used as key                      |
| Surrogate Key         | Artificial ID generated for identification      |
| CHECK Constraint      | Enforces a condition                            |
| NOT NULL              | Prevents NULL values                            |
| DEFAULT               | Supplies default value                          |
| Referential Integrity | Keeps relationships between tables valid        |
| Cascading Delete      | Deletes dependent records automatically         |
| Cascading Update      | Updates related key values                      |
| MongoDB `_id`         | Mandatory unique document identifier            |

---

# 5. Transaction Terminology

| Concept     | SQL Server                     | PostgreSQL      | MongoDB                                 | MySQL/InnoDB    | Oracle                | Db2                                    |
| ----------- | ------------------------------ | --------------- | --------------------------------------- | --------------- | --------------------- | -------------------------------------- |
| Transaction | Transaction                    | Transaction     | Transaction                             | Transaction     | Transaction           | Unit of Work                           |
| Commit      | COMMIT                         | COMMIT          | commitTransaction                       | COMMIT          | COMMIT                | COMMIT                                 |
| Rollback    | ROLLBACK                       | ROLLBACK        | abortTransaction                        | ROLLBACK        | ROLLBACK              | ROLLBACK                               |
| Savepoint   | SAVE TRANSACTION               | SAVEPOINT       | Different transaction API               | SAVEPOINT       | SAVEPOINT             | SAVEPOINT                              |
| Concurrency | Locking + row versioning       | MVCC            | Document concurrency + transactions     | MVCC            | MVCC/read consistency | Locking/MVCC capabilities              |
| Deadlock    | Deadlock                       | Deadlock        | Write conflict/deadlock-like contention | Deadlock        | Deadlock              | Deadlock                               |
| Isolation   | Isolation Level                | Isolation Level | Read/Write Concern + transactions       | Isolation Level | Isolation Level       | Isolation Level                        |
| Undo        | Version store / log mechanisms | MVCC tuples     | WiredTiger transaction mechanisms       | Undo Log        | Undo Tablespace       | Transaction logs/versioning mechanisms |
| Redo        | Transaction Log                | WAL             | Journal                                 | Redo Log        | Redo Log              | Transaction Log                        |

---

# 6. ACID Terminology

| Term        | Meaning                                             | DBA Importance           |
| ----------- | --------------------------------------------------- | ------------------------ |
| Atomicity   | All transaction operations succeed or fail together | Prevents partial updates |
| Consistency | Transaction leaves data valid                       | Protects integrity       |
| Isolation   | Concurrent transactions don't improperly interfere  | Controls concurrency     |
| Durability  | Committed data survives failure                     | Critical for recovery    |

---

# 7. Isolation Levels

| Isolation        | SQL Server | PostgreSQL                             | MySQL                 | Oracle                              | Db2                               |
| ---------------- | ---------- | -------------------------------------- | --------------------- | ----------------------------------- | --------------------------------- |
| Read Uncommitted | Yes        | Accepted but behaves as Read Committed | Yes                   | No direct equivalent                | UR                                |
| Read Committed   | Yes        | Default                                | Yes                   | Default                             | CS roughly comparable             |
| Repeatable Read  | Yes        | Yes                                    | Default InnoDB        | No direct ANSI setting              | RS                                |
| Serializable     | Yes        | Yes                                    | Yes                   | Yes                                 | RR roughly strongest Db2 level    |
| Snapshot         | Yes        | MVCC-based isolation                   | Consistent reads/MVCC | Serializable/read consistency model | Currently Committed/MVCC features |

Isolation semantics are **not perfectly interchangeable** between engines, even where names match.

---

# 8. Locking Terminology

| Term            | Purpose                                           |
| --------------- | ------------------------------------------------- |
| Shared Lock     | Allows reads while restricting conflicting writes |
| Exclusive Lock  | Protects data being modified                      |
| Update Lock     | SQL Server lock used during update paths          |
| Intent Lock     | Indicates lower-level locking                     |
| Row Lock        | Locks individual rows                             |
| Page Lock       | Locks database page                               |
| Table Lock      | Locks whole table                                 |
| Schema Lock     | Protects object definition                        |
| Metadata Lock   | Protects metadata/DDL operations                  |
| Lock Escalation | Converts many smaller locks to larger lock        |
| Blocking        | Session waits for another session                 |
| Deadlock        | Sessions wait cyclically                          |
| Deadlock Victim | Transaction selected for rollback                 |
| Lock Timeout    | Maximum permitted lock wait                       |
| Latch           | Lightweight internal synchronization              |
| Mutex           | Internal mutual-exclusion mechanism               |

---

# 9. MVCC / Versioning

| Database   | Main terminology                                           |
| ---------- | ---------------------------------------------------------- |
| SQL Server | Row Versioning / Version Store / Snapshot Isolation / RCSI |
| PostgreSQL | MVCC / Tuple Versions / `xmin` / `xmax`                    |
| MongoDB    | WiredTiger concurrency / snapshot transactions             |
| MySQL      | InnoDB MVCC / Undo Logs / Read View                        |
| Oracle     | Read Consistency / Undo                                    |
| Db2        | Currently Committed / locking and versioning features      |

**MVCC = Multi-Version Concurrency Control.**

Instead of forcing every reader to wait for a writer, the database can maintain versions that allow consistent reads.

---

# 10. Logging Terminology

| Concept                | SQL Server          | PostgreSQL   | MongoDB                | MySQL               | Oracle              | Db2             |
| ---------------------- | ------------------- | ------------ | ---------------------- | ------------------- | ------------------- | --------------- |
| Transaction log        | Transaction Log     | WAL          | Journal                | Redo Log            | Redo Log            | Transaction Log |
| Replication/change log | Transaction Log/CDC | WAL          | Oplog                  | Binary Log          | Redo/Archive Log    | Logs            |
| Archived logs          | Log backups         | WAL Archive  | Oplog retention        | Binary Logs         | Archived Redo Logs  | Archive Logs    |
| Log sequence           | LSN                 | LSN          | optime/timestamp       | Log position / GTID | SCN + redo sequence | LSN             |
| Log shipping           | Log Shipping        | WAL Shipping | Replica Set sync model | Binlog replication  | Data Guard          | HADR            |

---

# 11. Backup Terminology

| Concept                | SQL Server             | PostgreSQL                              | MongoDB                               | MySQL                 | Oracle           | Db2                  |
| ---------------------- | ---------------------- | --------------------------------------- | ------------------------------------- | --------------------- | ---------------- | -------------------- |
| Full backup            | Full                   | Base Backup                             | Snapshot/backup                       | Full Backup           | Full Backup      | Full Backup          |
| Incremental            | Differential           | Incremental capabilities/tool dependent | Incremental depending backup platform | Incremental via tools | Incremental RMAN | Incremental          |
| Log backup             | Transaction Log Backup | WAL Archive                             | Oplog-based mechanisms                | Binlog                | Archive Log      | Log Archive          |
| Physical backup        | Native backup          | `pg_basebackup`                         | Filesystem/snapshot                   | Physical backup       | RMAN             | Backup image         |
| Logical backup         | BACPAC/scripts         | `pg_dump`                               | `mongodump`                           | `mysqldump`           | Data Pump        | `db2move` / export   |
| Restore                | RESTORE                | Restore/base backup                     | `mongorestore`                        | Restore               | RMAN RESTORE     | RESTORE              |
| Point-in-time recovery | PITR                   | PITR                                    | PITR with appropriate backup tooling  | PITR                  | PITR             | Rollforward Recovery |

---

# 12. Recovery Terminology

| Term           | Meaning                                                          |
| -------------- | ---------------------------------------------------------------- |
| RPO            | Recovery Point Objective — acceptable amount of data loss        |
| RTO            | Recovery Time Objective — acceptable recovery duration           |
| PITR           | Point-in-Time Recovery                                           |
| Crash Recovery | Automatic recovery following abnormal shutdown                   |
| Roll Forward   | Reapply logged transactions                                      |
| Rollback       | Undo incomplete/unwanted transaction                             |
| Redo           | Reapply committed changes                                        |
| Undo           | Reverse changes                                                  |
| Recovery Log   | Log used during recovery                                         |
| Checkpoint     | Flushes/coordinates dirty data and establishes recovery progress |
| Restore        | Return backup data                                               |
| Recovery       | Make restored database operational/consistent                    |
| Failover       | Move workload to another server                                  |
| Failback       | Return workload to original/preferred server                     |

---

# 13. High Availability / Disaster Recovery

| Concept       | SQL Server         | PostgreSQL                         | MongoDB                     | MySQL                              | Oracle            | Db2                            |
| ------------- | ------------------ | ---------------------------------- | --------------------------- | ---------------------------------- | ----------------- | ------------------------------ |
| HA technology | Always On AG / FCI | Streaming Replication + HA tooling | Replica Set                 | Group Replication / InnoDB Cluster | RAC / Data Guard  | HADR / pureScale               |
| Primary       | Primary Replica    | Primary                            | Primary                     | Primary                            | Primary           | Primary                        |
| Secondary     | Secondary Replica  | Standby                            | Secondary                   | Secondary                          | Standby           | Standby                        |
| Replication   | AG/Replication     | Streaming/Logical                  | Replica Set                 | Async/Semi-sync/Group Replication  | Data Guard        | HADR                           |
| Cluster       | WSFC               | External HA stack                  | Replica Set/Sharded Cluster | InnoDB Cluster                     | RAC               | pureScale                      |
| Failover      | Automatic/Manual   | HA-manager dependent               | Election                    | Group/InnoDB Cluster mechanisms    | RAC/Data Guard    | HADR                           |
| Read replica  | Readable Secondary | Hot Standby                        | Secondary Reads             | Read Replica                       | Active Data Guard | Standby-read capabilities vary |

---

# 14. Replication Terminology

| Term                 | Meaning                                                             |
| -------------------- | ------------------------------------------------------------------- |
| Primary              | Main writable node                                                  |
| Secondary            | Replicated copy                                                     |
| Publisher            | Source of replicated data                                           |
| Subscriber           | Receiver of replicated data                                         |
| Distributor          | SQL Server replication intermediary                                 |
| Physical Replication | Copies low-level storage/log changes                                |
| Logical Replication  | Replicates logical data changes                                     |
| Synchronous          | Commit waits for replica acknowledgement according to configuration |
| Asynchronous         | Primary does not wait for remote durable acknowledgement            |
| Replication Lag      | Secondary is behind primary                                         |
| Log Shipping         | Ships transaction/WAL logs                                          |
| CDC                  | Change Data Capture                                                 |
| Change Tracking      | Tracks changed rows/keys                                            |
| Oplog                | MongoDB operations log                                              |
| WAL                  | Write-Ahead Log                                                     |
| Binlog               | MySQL Binary Log                                                    |
| Redo                 | Oracle change log                                                   |
| LSN                  | Log Sequence Number                                                 |
| SCN                  | Oracle System Change Number                                         |
| GTID                 | Global Transaction Identifier                                       |

---

# 15. Index Terminology

| Index Concept     | SQL Server                 | PostgreSQL                             | MongoDB                      | MySQL                  | Oracle                                      | Db2                  |
| ----------------- | -------------------------- | -------------------------------------- | ---------------------------- | ---------------------- | ------------------------------------------- | -------------------- |
| Primary structure | Clustered Index often used | Heap by default                        | Collection storage + indexes | Clustered PK in InnoDB | Heap table normally                         | Table + indexes      |
| Secondary         | Nonclustered               | B-tree etc.                            | Secondary Index              | Secondary Index        | B-tree                                      | Index                |
| Unique            | Unique Index               | Unique Index                           | Unique Index                 | Unique Index           | Unique Index                                | Unique Index         |
| Composite         | Composite                  | Multicolumn                            | Compound                     | Composite              | Composite                                   | Composite            |
| Filtered          | Filtered Index             | Partial Index                          | Partial Index                | Different alternatives | Function-based/partial approaches differ    | Index features vary  |
| Covering          | INCLUDE                    | INCLUDE                                | Covered Query                | Covering Index         | Index-only access depending structure/query | Index-only access    |
| Full text         | Full-Text Index            | Full Text Search indexes/configuration | Text Index/Search features   | FULLTEXT               | Oracle Text                                 | Text Search          |
| Spatial           | Spatial Index              | GiST/SP-GiST + PostGIS commonly        | Geospatial Index             | SPATIAL                | Spatial Index                               | Spatial capabilities |

---

# 16. Index Problems

| Term             | Meaning                                                  |
| ---------------- | -------------------------------------------------------- |
| Fragmentation    | Index pages become less optimally organized              |
| Bloat            | PostgreSQL/index structures occupy unnecessary space     |
| Index Rebuild    | Recreate index                                           |
| Index Reorganize | SQL Server lighter index maintenance                     |
| REINDEX          | PostgreSQL rebuild index                                 |
| Missing Index    | Potentially useful index absent                          |
| Unused Index     | Index consumes resources without useful workload benefit |
| Duplicate Index  | Multiple indexes provide overlapping structures          |
| Selectivity      | How effectively values narrow rows                       |
| Cardinality      | Number/distribution of distinct values                   |
| Index Scan       | Reads significant/all index entries                      |
| Index Seek       | Targets specific key ranges                              |
| Table Scan       | Reads table broadly                                      |
| Bitmap Scan      | PostgreSQL/Oracle-style bitmap access                    |
| Index-Only Scan  | Query satisfied largely from index                       |

---

# 17. Query Processing Terminology

| Term                 | Meaning                                                              |
| -------------------- | -------------------------------------------------------------------- |
| SQL Parser           | Validates/parses SQL syntax                                          |
| Optimizer            | Chooses execution strategy                                           |
| Execution Plan       | Operations used to execute query                                     |
| Estimated Plan       | Optimizer's predicted plan                                           |
| Actual Plan          | Plan with runtime information                                        |
| Cost                 | Optimizer's relative resource estimate                               |
| Cardinality Estimate | Estimated rows                                                       |
| Predicate            | Filtering condition                                                  |
| Join                 | Combines data sets                                                   |
| Nested Loop          | Join strategy                                                        |
| Hash Join            | Hash-based join                                                      |
| Merge Join           | Sorted-input join                                                    |
| Sort                 | Orders rows                                                          |
| Aggregate            | Calculates grouped/summary results                                   |
| Scan                 | Broad read operation                                                 |
| Seek                 | Targeted index access                                                |
| Spool                | Temporary intermediate storage                                       |
| Parallelism          | Uses multiple workers/threads                                        |
| Plan Cache           | Stores reusable execution plans                                      |
| Parameter Sniffing   | SQL Server plan affected by parameter values seen during compilation |
| Statistics           | Data distribution metadata used by optimizer                         |

---

# 18. Statistics Terminology

| SQL Server            | PostgreSQL             | MongoDB                        | MySQL            | Oracle                    | Db2                           |
| --------------------- | ---------------------- | ------------------------------ | ---------------- | ------------------------- | ----------------------------- |
| Statistics            | Statistics             | Query planner/index statistics | Statistics       | Optimizer Statistics      | Statistics                    |
| UPDATE STATISTICS     | ANALYZE                | Internal/planner mechanisms    | ANALYZE TABLE    | `DBMS_STATS`              | RUNSTATS                      |
| Auto Stats            | Autovacuum/autoanalyze | Automatic mechanisms           | Persistent stats | Automatic statistics jobs | Automatic statistics features |
| Histogram             | Histogram              | Statistics/planner metadata    | Histogram        | Histogram                 | Distribution Statistics       |
| Cardinality Estimator | Planner                | Query Planner                  | Optimizer        | CBO                       | Optimizer                     |

---

# 19. Performance Terminology

| Term               | What DBA watches                |
| ------------------ | ------------------------------- |
| CPU Utilization    | Processor pressure              |
| Memory Pressure    | Insufficient available memory   |
| Disk Latency       | Slow storage                    |
| IOPS               | Storage operations/sec          |
| Throughput         | Amount of data processed        |
| Network Latency    | Communication delay             |
| Query Duration     | Execution time                  |
| Wait Time          | Time waiting on resources       |
| Blocking           | Sessions waiting on locks       |
| Deadlocks          | Circular locking dependency     |
| Cache Hit Ratio    | Reads served from memory/cache  |
| Buffer Cache       | Cached database pages           |
| Plan Cache         | Cached query plans              |
| Temp Usage         | Temporary workspace consumption |
| Connection Count   | Active/open connections         |
| Transactions/sec   | Transaction workload            |
| Batch Requests/sec | SQL Server workload rate        |
| QPS                | Queries per second              |
| TPS                | Transactions per second         |

---

# 20. Memory Terminology

| SQL Server         | PostgreSQL                         | MongoDB                        | MySQL                       | Oracle                      | Db2                  |                          |
| ------------------ | ---------------------------------- | ------------------------------ | --------------------------- | --------------------------- | -------------------- | ------------------------ |
| Buffer Pool        | Shared Buffers                     | WiredTiger Cache               | InnoDB Buffer Pool          | Buffer Cache                | Bufferpools          |                          |
| Plan Cache         | Plan/prepared statement mechanisms | Plan Cache                     | Statement/metadata caches   | Shared Pool                 | Package Cache        |                          |
| Query workspace    | `work_mem`                         | Working memory/cache           | Sort/Join buffers           | PGA                         | Sort/statement heaps |                          |
| Server memory      | Max Server Memory                  | Shared memory + process memory | WiredTiger + process memory | Buffer Pool + server memory | SGA                  | Instance/DB memory       |
| Per-session memory | Query grants/session memory        | `work_mem` etc.                | Connection/operation memory | Per-thread buffers          | PGA                  | Agent/application memory |

---

# 21. SQL Server-Specific DBA Terms

| Term            | Purpose                                              |
| --------------- | ---------------------------------------------------- |
| MDF             | Primary database data file                           |
| NDF             | Secondary data file                                  |
| LDF             | Transaction log file                                 |
| Filegroup       | Logical data-file grouping                           |
| tempdb          | Global temporary/system workspace                    |
| master          | System database                                      |
| model           | Template database                                    |
| msdb            | Agent, backup history and other operational metadata |
| Resource DB     | Read-only system object storage                      |
| SQL Agent       | Job scheduler                                        |
| SSMS            | Administration GUI                                   |
| T-SQL           | SQL Server SQL dialect                               |
| DMVs            | Dynamic Management Views                             |
| Extended Events | Diagnostics/event framework                          |
| Query Store     | Query history/performance tracking                   |
| Always On AG    | Availability Groups                                  |
| FCI             | Failover Cluster Instance                            |
| Log Shipping    | DR/log-copy technology                               |
| Replication     | Data distribution technology                         |
| RCSI            | Read Committed Snapshot Isolation                    |
| DBCC            | Database consistency/maintenance commands            |
| CHECKDB         | Integrity checking                                   |
| SPID            | Session/process identifier                           |
| Wait Stats      | SQL Server wait diagnostics                          |

---

# 22. PostgreSQL-Specific DBA Terms

| Term                  | Purpose                                                  |
| --------------------- | -------------------------------------------------------- |
| Cluster               | Collection of databases managed by one PostgreSQL server |
| `postgres`            | Main server process                                      |
| WAL                   | Write-Ahead Log                                          |
| `pg_wal`              | WAL directory                                            |
| VACUUM                | Reclaims/reuses space and maintains MVCC health          |
| VACUUM FULL           | Rewrites table to reclaim space                          |
| Autovacuum            | Automatic vacuum/analyze system                          |
| ANALYZE               | Updates planner statistics                               |
| Dead Tuple            | Obsolete row version                                     |
| Bloat                 | Excess table/index space                                 |
| TOAST                 | Large-value storage mechanism                            |
| Tablespace            | Alternate filesystem storage location                    |
| `pg_hba.conf`         | Client authentication rules                              |
| `postgresql.conf`     | Main configuration                                       |
| `pg_stat_activity`    | Session/activity monitoring                              |
| `pg_stat_statements`  | Query statistics extension                               |
| Streaming Replication | Physical replication                                     |
| Logical Replication   | Publisher/subscriber replication                         |
| Hot Standby           | Read-capable standby                                     |
| `pg_basebackup`       | Physical base backup                                     |
| `pg_dump`             | Logical backup                                           |
| `pg_restore`          | Logical restore                                          |

---

# 23. MongoDB-Specific DBA Terms

| Term            | Purpose                                 |
| --------------- | --------------------------------------- |
| Document        | BSON record                             |
| Collection      | Group of documents                      |
| BSON            | Binary JSON representation              |
| `_id`           | Unique document identifier              |
| `mongod`        | Database server process                 |
| `mongos`        | Sharded-cluster router                  |
| Replica Set     | HA/replication group                    |
| Primary         | Main writable member                    |
| Secondary       | Replicated member                       |
| Arbiter         | Voting member without data copy         |
| Oplog           | Replication operations log              |
| WiredTiger      | Default storage engine                  |
| Journal         | Durability/recovery log                 |
| Sharding        | Horizontal data partitioning            |
| Shard           | Data-bearing partition                  |
| Config Server   | Stores sharded-cluster metadata         |
| Shard Key       | Determines data distribution            |
| Chunk/Range     | Unit/range of sharded data distribution |
| Balancer        | Redistributes ranges between shards     |
| Read Preference | Chooses eligible read members           |
| Read Concern    | Controls read consistency guarantees    |
| Write Concern   | Controls write acknowledgement          |
| Change Stream   | Watches data changes                    |
| GridFS          | Storage convention for large files      |

---

# 24. MySQL-Specific DBA Terms

| Term                 | Purpose                               |
| -------------------- | ------------------------------------- |
| `mysqld`             | Database server                       |
| InnoDB               | Default transactional storage engine  |
| MyISAM               | Older/nontransactional storage engine |
| Buffer Pool          | InnoDB cache                          |
| Redo Log             | Crash recovery                        |
| Undo Log             | MVCC/rollback                         |
| Binary Log           | Replication/PITR                      |
| Relay Log            | Replica receives source binlog events |
| GTID                 | Global Transaction Identifier         |
| AUTO_INCREMENT       | Automatic numeric ID                  |
| `my.cnf`             | Linux/Unix configuration              |
| `my.ini`             | Windows configuration                 |
| Performance Schema   | Performance instrumentation           |
| `information_schema` | Metadata views                        |
| Group Replication    | HA replication technology             |
| InnoDB Cluster       | MySQL HA solution                     |
| `mysqldump`          | Logical backup                        |
| Slow Query Log       | Slow SQL diagnostics                  |

---

# 25. Oracle-Specific DBA Terms

| Term            | Purpose                                     |
| --------------- | ------------------------------------------- |
| Instance        | SGA + background processes                  |
| Database        | Physical database files                     |
| SID             | Instance identifier                         |
| Service Name    | Client connection service                   |
| SGA             | Shared memory                               |
| PGA             | Process/session work memory                 |
| Shared Pool     | SQL/data dictionary caching                 |
| Buffer Cache    | Data-block cache                            |
| Redo Log        | Records changes                             |
| Archive Log     | Archived redo                               |
| Control File    | Database structural metadata                |
| Datafile        | Physical data storage                       |
| Tablespace      | Logical storage                             |
| Undo Tablespace | Read consistency/rollback                   |
| TEMP            | Temporary operations                        |
| SCN             | System Change Number                        |
| RMAN            | Backup/recovery manager                     |
| Data Pump       | Logical import/export                       |
| Data Guard      | Standby/DR platform                         |
| RAC             | Multi-instance clustered database           |
| ASM             | Automatic Storage Management                |
| AWR             | Workload performance repository             |
| ASH             | Active Session History                      |
| ADDM            | Automatic performance diagnostics           |
| CDB             | Container Database                          |
| PDB             | Pluggable Database                          |
| Listener        | Client network connection listener          |
| TNS             | Oracle networking terminology/configuration |

---

# 26. IBM Db2-Specific DBA Terms

| Term            | Purpose                                   |
| --------------- | ----------------------------------------- |
| Db2 Instance    | Database manager environment              |
| Database        | Database                                  |
| Tablespace      | Logical storage                           |
| Bufferpool      | Memory cache                              |
| Container       | Tablespace physical storage concept       |
| DBM CFG         | Database Manager configuration            |
| DB CFG          | Database configuration                    |
| Transaction Log | Recovery log                              |
| Active Log      | Currently active transaction logs         |
| Archive Log     | Archived transaction logs                 |
| HADR            | High Availability Disaster Recovery       |
| pureScale       | Clustered scale/availability architecture |
| RUNSTATS        | Update statistics                         |
| REORG           | Reorganize table/index data               |
| REORGCHK        | Reorganization analysis                   |
| BIND            | Creates package from application SQL      |
| REBIND          | Recreates existing package                |
| Package Cache   | Cached SQL/packages                       |
| Explain         | Execution plan analysis                   |
| Locklist        | Lock memory configuration                 |
| Unit of Work    | Transaction                               |
| Rollforward     | Recovery using transaction logs           |

---

# 27. Security Terminology

| Concept                 | SQL Server                | PostgreSQL                               | MongoDB               | MySQL                                 | Oracle                        | Db2                         |
| ----------------------- | ------------------------- | ---------------------------------------- | --------------------- | ------------------------------------- | ----------------------------- | --------------------------- |
| Authentication identity | Login                     | Role/User                                | User                  | User                                  | User                          | OS/external/user mechanisms |
| Authorization grouping  | Database Role/Server Role | Role                                     | Role                  | Role                                  | Role                          | Role/Authority              |
| Permission              | GRANT                     | GRANT                                    | Privilege/Role        | GRANT                                 | GRANT                         | GRANT                       |
| Remove permission       | REVOKE                    | REVOKE                                   | Revoke role/privilege | REVOKE                                | REVOKE                        | REVOKE                      |
| Encryption in transit   | TLS                       | SSL/TLS                                  | TLS                   | TLS                                   | Native Network Encryption/TLS | TLS                         |
| Encryption at rest      | TDE                       | External/storage solutions or extensions | Encryption at Rest    | Enterprise/storage encryption options | TDE                           | Native/storage encryption   |
| Auditing                | SQL Audit                 | Logging/extensions                       | Auditing              | Audit plugin                          | Unified Auditing              | Audit facility              |

---

# 28. Common Security Terms Every DBA Should Know

| Term                  | Meaning                                     |
| --------------------- | ------------------------------------------- |
| Authentication        | Who are you?                                |
| Authorization         | What are you allowed to do?                 |
| RBAC                  | Role-Based Access Control                   |
| Least Privilege       | Give only required permissions              |
| Encryption at Rest    | Encrypt stored database data                |
| Encryption in Transit | Encrypt network traffic                     |
| TLS                   | Secure network protocol                     |
| Certificate           | Cryptographic identity                      |
| Key Management        | Management of encryption keys               |
| Data Masking          | Hide sensitive values                       |
| Dynamic Masking       | Mask data during query output               |
| Auditing              | Track security/database activity            |
| Service Account       | Account used by database services           |
| Privileged Account    | High-level administrative account           |
| Secret                | Password/token/key                          |
| Rotation              | Periodically replace credentials/keys       |
| SQL Injection         | Application attack through unsafe SQL input |

---

# 29. DBA Maintenance Terminology

| Task              | SQL Server                        | PostgreSQL            | MongoDB                      | MySQL                         | Oracle                        | Db2                                    |
| ----------------- | --------------------------------- | --------------------- | ---------------------------- | ----------------------------- | ----------------------------- | -------------------------------------- |
| Integrity         | DBCC CHECKDB                      | Checks/tooling        | Validation                   | CHECK TABLE                   | DBVERIFY/RMAN validation      | INSPECT                                |
| Stats             | UPDATE STATISTICS                 | ANALYZE               | Planner mechanisms           | ANALYZE TABLE                 | DBMS_STATS                    | RUNSTATS                               |
| Index maintenance | REBUILD/REORGANIZE                | REINDEX               | Rebuild indexes where needed | ALTER/OPTIMIZE depending goal | ALTER INDEX REBUILD           | REORG                                  |
| Space maintenance | Shrink rarely/restricted use      | VACUUM                | Compact/resync strategies    | OPTIMIZE                      | Segment/tablespace management | REORG                                  |
| Scheduler         | SQL Agent                         | cron/pg_cron/external | External/Atlas automation    | Event Scheduler/external      | DBMS_SCHEDULER                | Administrative Task Scheduler/external |
| Backup validation | RESTORE VERIFYONLY + test restore | Test restore          | Restore testing              | Restore testing               | RMAN VALIDATE                 | Restore validation/testing             |

---

# 30. Monitoring / Troubleshooting Terms

| Term                  | DBA Question                                       |
| --------------------- | -------------------------------------------------- |
| Session               | Who is connected?                                  |
| Active Session        | Who is currently working?                          |
| Long-Running Query    | What SQL has been running too long?                |
| Blocking Session      | Who is blocking others?                            |
| Blocked Session       | Who is waiting?                                    |
| Deadlock              | Which transactions conflict cyclically?            |
| Wait Event            | What is a session waiting for?                     |
| CPU Pressure          | Is CPU saturated?                                  |
| Memory Pressure       | Is database memory constrained?                    |
| I/O Pressure          | Is storage the bottleneck?                         |
| Log Growth            | Is transaction log expanding unexpectedly?         |
| Replication Lag       | How far behind is replica?                         |
| Connection Saturation | Are connection limits being reached?               |
| Disk Full             | Is database storage exhausted?                     |
| Slow Query            | Which query is consuming excessive time/resources? |
| Plan Regression       | Did execution plan become worse?                   |
| Cache Miss            | Is required data frequently absent from cache?     |
| Timeout               | Did operation exceed configured wait time?         |
| Error Log             | What errors/warnings did engine report?            |

---

# 31. Capacity Planning Terms

| Term            | Meaning                               |
| --------------- | ------------------------------------- |
| Database Size   | Total allocated/used database storage |
| Data Growth     | Rate data volume increases            |
| Log Growth      | Rate transaction logs increase        |
| Free Space      | Remaining available capacity          |
| Autogrowth      | Automatic file expansion              |
| Max Size        | Configured storage limit              |
| Disk Capacity   | Total filesystem/storage capacity     |
| IOPS            | I/O operations per second             |
| Throughput      | MB/s or similar data transfer rate    |
| Latency         | Time required for I/O                 |
| CPU Cores       | Processing capacity                   |
| RAM             | Memory capacity                       |
| Connections     | Concurrent database connections       |
| TPS             | Transactions per second               |
| QPS             | Queries per second                    |
| Peak Load       | Maximum workload period               |
| Baseline        | Normal performance reference          |
| Growth Forecast | Expected future capacity requirement  |

---

# 32. Data Modeling Terminology

| Term             | Meaning                                        |
| ---------------- | ---------------------------------------------- |
| Entity           | Business object                                |
| Attribute        | Property/column                                |
| Relationship     | Connection between entities                    |
| One-to-One       | 1:1 relationship                               |
| One-to-Many      | 1:N relationship                               |
| Many-to-Many     | M:N relationship                               |
| Normalization    | Reduce redundancy                              |
| Denormalization  | Add redundancy for performance/access patterns |
| 1NF              | First Normal Form                              |
| 2NF              | Second Normal Form                             |
| 3NF              | Third Normal Form                              |
| BCNF             | Boyce-Codd Normal Form                         |
| Star Schema      | Dimensional warehouse model                    |
| Snowflake Schema | Normalized dimensional model                   |
| Fact Table       | Measures/events                                |
| Dimension Table  | Descriptive attributes                         |
| OLTP             | Transaction-oriented workload                  |
| OLAP             | Analytical workload                            |
| Data Warehouse   | Analytical data platform                       |
| Data Mart        | Subject-specific analytical store              |

---

# 33. Partitioning and Sharding

| Concept                 | Meaning                                       |
| ----------------------- | --------------------------------------------- |
| Partitioning            | Divide one logical table into smaller pieces  |
| Horizontal Partitioning | Split by rows                                 |
| Vertical Partitioning   | Split by columns                              |
| Range Partitioning      | Partition according to value ranges           |
| List Partitioning       | Partition according to value lists            |
| Hash Partitioning       | Hash determines partition                     |
| Partition Key           | Column(s) determining partition               |
| Partition Pruning       | Optimizer skips irrelevant partitions         |
| Sharding                | Distribute data across servers                |
| Shard Key               | Determines target shard                       |
| Resharding              | Change shard distribution/key strategy        |
| Hot Partition           | Partition receiving disproportionate workload |
| Hot Shard               | Shard receiving disproportionate workload     |

---

# 34. Migration / Upgrade Terminology

| Term                    | Meaning                                       |
| ----------------------- | --------------------------------------------- |
| Migration               | Move database/workload                        |
| Upgrade                 | Move to newer database version                |
| In-place Upgrade        | Upgrade existing installation                 |
| Side-by-Side Upgrade    | Build new environment and migrate             |
| Homogeneous Migration   | Same database technology                      |
| Heterogeneous Migration | Different database engines                    |
| Schema Migration        | Move/convert structures                       |
| Data Migration          | Move actual records                           |
| Cutover                 | Switch production traffic                     |
| Rollback Plan           | Return to old environment                     |
| Downtime                | Period application unavailable                |
| Zero/Minimal Downtime   | Migration designed for little interruption    |
| Compatibility           | Application/database feature compatibility    |
| Validation              | Confirm migrated data is correct              |
| Reconciliation          | Compare source and target                     |
| CDC Migration           | Continuously capture changes during migration |

---

# 35. Cloud DBA Terminology

| Term                | Meaning                                           |
| ------------------- | ------------------------------------------------- |
| IaaS                | Infrastructure as a Service                       |
| PaaS                | Platform as a Service                             |
| DBaaS               | Database as a Service                             |
| Managed Database    | Cloud provider handles portions of DBA operations |
| Read Replica        | Read-only replicated database                     |
| Multi-AZ            | Deployment across availability zones              |
| Region              | Geographic cloud area                             |
| Availability Zone   | Independent datacenter grouping                   |
| Snapshot            | Point-in-time storage copy                        |
| Automated Backup    | Platform-scheduled backup                         |
| PITR                | Point-in-time restore                             |
| Scaling Up          | Increase CPU/RAM                                  |
| Scaling Out         | Add nodes                                         |
| Serverless Database | Capacity managed dynamically by service           |
| Provisioned IOPS    | Reserved/configured storage performance           |
| Encryption Key      | Key used for data encryption                      |
| KMS                 | Key Management Service                            |
| Private Endpoint    | Private network access to managed DB              |

---

# 36. Core DBA Acronyms — Quick Reference

| Acronym | Full Form                                     |
| ------- | --------------------------------------------- |
| DBA     | Database Administrator                        |
| DBMS    | Database Management System                    |
| RDBMS   | Relational Database Management System         |
| SQL     | Structured Query Language                     |
| DDL     | Data Definition Language                      |
| DML     | Data Manipulation Language                    |
| DCL     | Data Control Language                         |
| TCL     | Transaction Control Language                  |
| ACID    | Atomicity, Consistency, Isolation, Durability |
| MVCC    | Multi-Version Concurrency Control             |
| WAL     | Write-Ahead Logging                           |
| LSN     | Log Sequence Number                           |
| SCN     | System Change Number                          |
| GTID    | Global Transaction Identifier                 |
| CDC     | Change Data Capture                           |
| HA      | High Availability                             |
| DR      | Disaster Recovery                             |
| HA/DR   | High Availability / Disaster Recovery         |
| RPO     | Recovery Point Objective                      |
| RTO     | Recovery Time Objective                       |
| PITR    | Point-in-Time Recovery                        |
| OLTP    | Online Transaction Processing                 |
| OLAP    | Online Analytical Processing                  |
| ETL     | Extract, Transform, Load                      |
| ELT     | Extract, Load, Transform                      |
| IOPS    | Input/Output Operations Per Second            |
| TPS     | Transactions Per Second                       |
| QPS     | Queries Per Second                            |
| RBAC    | Role-Based Access Control                     |
| TLS     | Transport Layer Security                      |
| TDE     | Transparent Data Encryption                   |
| PK      | Primary Key                                   |
| FK      | Foreign Key                                   |
| CTE     | Common Table Expression                       |
| AG      | Availability Group                            |
| FCI     | Failover Cluster Instance                     |
| RAC     | Real Application Clusters                     |
| RMAN    | Recovery Manager                              |
| ASM     | Automatic Storage Management                  |
| AWR     | Automatic Workload Repository                 |
| HADR    | High Availability Disaster Recovery           |

## 37. End-to-End DBA Mental Model

A useful way to remember the entire DBA field is:

**Database Administration**

→ **Architecture**
Instance → Database → Schema → Object

→ **Storage**
Disk → Tablespace/Filegroup → File → Extent → Page/Block → Row/Document

→ **Data Objects**
Table/Collection → Row/Document → Column/Field → Key → Constraint → Index

→ **Transactions**
BEGIN → Read/Write → Lock/MVCC → Log → COMMIT/ROLLBACK

→ **Security**
Authentication → Authorization → Roles → Privileges → Encryption → Auditing

→ **Performance**
Query → Parser → Optimizer → Execution Plan → CPU/Memory/I/O → Waits → Tuning

→ **Maintenance**
Statistics → Index → Vacuum/Reorg → Integrity → Capacity

→ **Backup & Recovery**
Full/Base Backup → Incremental/Differential → Transaction/WAL/Archive Logs → Restore → PITR

→ **HA/DR**
Primary → Replica/Standby → Replication → Failover → Failback

→ **Monitoring**
Availability → Sessions → Blocking → Deadlocks → CPU → Memory → I/O → Storage → Logs → Replication Lag

→ **Lifecycle**
Install → Configure → Secure → Deploy → Monitor → Tune → Backup → Patch → Upgrade → Migrate → Decommission

This is the core terminology map I'd expect a **SQL Server / PostgreSQL / MongoDB / MySQL / Oracle / Db2 DBA or Database Architect** to understand.
