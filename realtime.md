Recently, replication lag increased because an application team executed a large bulk data load. This generated WAL records faster than the standby could replay them. Using pg_stat_replication, we found a significant gap between the primary LSN and replay LSN. We monitored disk I/O and WAL replay activity on the standby, and after the bulk load completed, replication caught up. We also recommended performing future loads during maintenance windows to avoid replication delays

In one production incident, users reported a query taking several minutes instead of a few seconds. I first checked pg stat_activity and found no blocking. Next, using EXPLAIN ANALYZE, I noticed the optimizer had switched from an Index Scan to a Sequential Scan because the table statistics were outdated after a large data load. We executed ANALYZE, refreshed statistics, and the query reverted to using the correct index. Execution time reduced from several minutes back to a few seconds.

In my current production environment, we use Physical Streaming Replication for High Availability. The primary server streams WAL records continuously to the standby server, and in case of failure we can perform a failover with minimal downtime.
If a business team requests only a few tables to be replicated to a reporting server, then I would recommend Logical Replication because it allows table-level replication rather than replicating the enfire database cluster.


In production, if an unexpected primary failure occurs, our priority is restoring service quickly while preventing split-brain. We first confirm which standby has the most current LSN and minimal lag. After promoting it, we validate application connectivity and business transactions. When the failed primary is recovered, we rebuild it as a standby and rejoin it to the cluster. This ensures a single authoritative primary at all times.

100 million rows in a single transaction is risky because it generates massive WAL, increases replication lag, consumes storage, and can lepact application performance.
I would perform the deletion in batches, for example 10,000 to 100,000 rows per batch, with commits after each batch.

SQL
DELETE FROM orders
WHERE created date < 2020-01-01
LIMIT 50000,
I would schedule the activity during a maintenance window or non-business hours.
During the operation, I would monitor
WAL generation rate
Replication lag (pg stat replication)
Disk utilization
CPU and 1/0 utilization
Long-running transactions If the table is partitioned by date, I would prefer dropping or detaching old partitions, which is much faster than deleting nows individually. After completion, I would evaluate whether VACUUM or VACUUM FULL is required depending on the amount of reclaimed space
In one environment, a business team requested purging historical data Instead of executing a single massive delete, we implemented batch deletions with controlled commit intervals. This prevented excessive HAL generation and replication lag Since the environment had streaming replication, we continuously monitored LSN novement and replay lag on the standby servers The cleanup completed successfully without Impacting business users.
