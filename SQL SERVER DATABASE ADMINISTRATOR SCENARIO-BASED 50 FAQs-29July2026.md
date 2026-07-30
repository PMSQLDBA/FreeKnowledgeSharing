# SQL SERVER DATABASE ADMINISTRATOR SCENARIO-BASED FAQs

## 1. PARAMETER SNIFFING

### What is parameter sniffing and why does my stored procedure perform differently with different parameter values?

Parameter sniffing occurs when SQL Server compiles a query plan based on the first parameter value it receives, then reuses that plan for all subsequent executions with different parameters.

- SQL Server captures the parameter value at compile time and creates an execution plan optimized for that specific value
- When you execute the same procedure with a different parameter (for example, a value that returns 100 rows versus 100,000 rows), the plan remains the same and becomes inefficient
- You can detect this by running sp_executesql with different parameters and comparing execution times
- Enable trace flag 4136 to disable parameter sniffing, but this reduces optimization benefit for normally efficient parameters
- Use OPTIMIZE FOR hints like "OPTIMIZE FOR (@Parameter = 'value')" in the query to force compilation with a specific value
- Query Store helps identify parameter sniffing issues by showing multiple executions with different performance characteristics

### How do I fix a stored procedure where the execution plan is not optimal for all parameter values?

When a single execution plan doesn't perform well across different parameter ranges, you need to force recompilation or use adaptive plans.

- Add RECOMPILE hint to the stored procedure or specific query to force a new plan for each execution, ensuring the plan matches the parameter values
- Use OPTION (RECOMPILE) at the query level instead of at the procedure level to minimize recompilation overhead
- Implement OPTIMIZE FOR UNKNOWN hint to create a plan that works reasonably well for all parameter values rather than optimizing for a specific value
- Split the stored procedure into multiple procedures, each optimized for different parameter ranges
- Use Query Store to capture multiple plans and then enable plan forcing for the best performing plan
- Monitor CPU and compilation overhead when using RECOMPILE, as excessive recompilation increases server load

### Why does my procedure run slow on the first execution after deployment but fast on subsequent executions?

SQL Server compiles a new execution plan on the first execution of a stored procedure or query, which temporarily impacts performance.

- The first execution compiles the query into an execution plan; this compilation takes CPU and memory resources
- Subsequent executions use the cached plan from the first run, making them significantly faster
- If the first execution uses unusual parameter values, the cached plan may not be optimal for typical usage patterns
- Run sp_recompile on the procedure to clear the cached plan and force recompilation with new parameters
- Use DBCC FREEPROCCACHE to clear all cached plans, but this affects the entire server and should only be done during maintenance windows
- Deploy with warmup scripts that execute stored procedures with representative parameter values to build optimal plans

### How can I use Query Store to identify and resolve parameter sniffing issues?

Query Store captures multiple execution plans for the same query and tracks performance metrics for each plan variant.

- Enable Query Store by running ALTER DATABASE [DatabaseName] SET QUERY_STORE = ON
- Query sys.query_store_query_text and sys.query_store_plan to see all plan variants for a single query
- Look for queries with high variation in execution time or IO statistics to identify potential parameter sniffing problems
- Use dm_exec_query_stats to compare average duration and logical reads for different parameter values
- Force a specific execution plan that performs well across all parameter ranges using sp_query_store_force_plan
- Compare execution times before and after forcing a plan to confirm the fix resolves the sniffing issue

---

## 2. QUERY STORE

### What is Query Store and what problems does it solve?

Query Store is a built-in feature that captures and stores information about query execution plans, runtime statistics, and performance history.

- Query Store maintains a persistent history of execution plans for every query in your database, unlike plan cache which clears after server restart
- It captures metrics like CPU time, logical reads, execution count, and memory grant size for each execution
- You can view performance degradation over time and identify when a query execution plan changed for the worse
- Query Store enables plan forcing, allowing you to force SQL Server to use a specific execution plan instead of a different one
- It helps troubleshoot performance issues by correlating performance changes with plan changes or parameter variations
- Query Store reduces troubleshooting time for intermittent performance problems because historical data persists across server restarts

### How do I enable and configure Query Store for optimal monitoring?

Query Store requires proper configuration to capture relevant data without excessive overhead on your server.

- Enable Query Store by running ALTER DATABASE [DatabaseName] SET QUERY_STORE = ON
- Set QUERY_STORE operation mode to READ_WRITE for normal operations; use READ_ONLY mode if you want data collection without write access
- Configure the data retention period; 30 days is default, but increase to 90 days for better trend analysis on production databases
- Set INTERVAL_LENGTH_MINUTES to determine how often SQL Server flushes Query Store data to disk; default is 60 minutes
- Enable STATS_INTERVAL_MINUTES to define how often wait statistics are captured; this helps identify blocking and resource contention issues
- Monitor sys.dm_db_query_store_internal_config to verify current settings and adjust QUERY_CAPTURE_MODE to capture the right queries

### How do I use Query Store to detect and fix query regressions?

Query Store automatically tracks when query performance degrades compared to baseline performance.

- Query sys.query_store_runtime_stats to find queries with increasing CPU time or logical reads over multiple days
- Compare execution plans for the same query using sys.query_store_plan to identify when a plan change caused performance to degrade
- Use the Query Store UI in SQL Server Management Studio to see graphical comparisons of query performance over time
- When you identify a good performing plan, force that plan using sp_query_store_force_plan so SQL Server uses it consistently
- Remove query hints and RECOMPILE statements after forcing a better plan with Query Store
- Monitor query performance again for one to two weeks to confirm the fix resolves the regression

### What queries should I run against Query Store to monitor my top bottleneck queries?

Query Store system views provide detailed information about which queries consume the most resources.

- Query sys.query_store_query and sys.query_store_runtime_stats to find top 10 queries by CPU time using SUM(cpu_time)
- Join sys.query_store_plan and sys.query_store_runtime_stats to see the total logical reads consumed by each query
- Query sys.query_store_wait_stats to identify which wait types are affecting your worst performing queries
- Look for queries with high compilation count compared to execution count; this indicates excessive plan recompilation
- Create a view that ranks queries by total elapsed time to identify which queries consume the most total server resources
- Schedule a job to export Query Store data weekly to track trends and identify performance patterns over weeks or months

---

## 3. EXECUTION PLAN

### What is an execution plan and why is it important to understand it?

An execution plan is SQL Server's roadmap for executing a query; it shows every operation required to retrieve the data you requested.

- SQL Server creates an execution plan by analyzing your query syntax and table statistics, then determining the optimal sequence of operations
- The plan shows operations like Table Scan, Index Seek, Nested Loop Join, Hash Join, and Sort, along with estimated rows and actual rows
- Execution plans reveal inefficiencies like full table scans on large tables, key lookups on non-clustered indexes, or inefficient join algorithms
- Comparing estimated rows to actual rows shows whether SQL Server's statistics are accurate; large differences indicate statistics need updating
- The cost percentage shown for each operation is relative to the total query cost; operations with high percentages consume the most resources
- Understanding execution plans enables you to optimize queries and identify missing indexes more effectively than trying random tuning approaches

### How do I capture and analyze an execution plan in SQL Server?

Capturing accurate execution plans is essential for identifying performance bottlenecks in your queries.

- Use SET STATISTICS IO ON to see logical and physical reads generated by a query; high numbers indicate full table scans or missing indexes
- Enable actual execution plan in SQL Server Management Studio using Ctrl+Alt+A or Query Menu, then review the plan after query execution
- Use sp_helpindex to identify which indexes exist on a table, then compare against the indexes the execution plan recommends
- Look for operations with high CPU cost or high number of rows processed; these indicate where optimization will have the most impact
- Compare estimated rows to actual rows for each operation; significant differences mean your table statistics are out of date
- Export execution plans to XML format for archival or comparison using the "Save Execution Plan As" option in Management Studio

### What does it mean when my execution plan shows "high cost" and how do I fix it?

High cost in an execution plan indicates that an operation is consuming significant server resources, usually CPU, memory, or I/O.

- Cost is a relative measure calculated by SQL Server's cardinality estimator based on CPU and I/O expense of the operation
- Operations with cost above 1 percent of total query cost consume meaningful resources; prioritize optimizing these operations first
- High cost often results from Table Scans on large tables, Nested Loop Joins with many iterations, or Hash Match Aggregates on large datasets
- Add indexes to eliminate Table Scans and convert them to Index Seeks on the columns used in WHERE clauses
- For expensive Nested Loop Joins, add indexes on join columns or consider Merge Joins if data is already sorted
- Run UPDATE STATISTICS to refresh column statistics; inaccurate statistics cause SQL Server to choose expensive operations

### How do I compare two execution plans to understand why one query is faster than another?

Comparing execution plans reveals which operations differ and why one approach is more efficient than another.

- Capture both execution plans as actual plans, not estimated plans, to see real resource consumption and row counts
- Compare the number of operations; fewer operations usually indicate better efficiency, but focus on total cost instead of operation count
- Look at row counts for each operation; if one plan processes significantly more rows than another, it's doing unnecessary work
- Compare join types; if one plan uses Merge Join and another uses Nested Loop with millions of rows, the Merge Join is likely more efficient
- Check Index Seek versus Index Scan; Seek is more efficient when it uses an index to narrow down rows, while Scan must read all rows
- Use Query Store to compare performance metrics like CPU time and logical reads; the plan with lower actual metrics is the better plan

### Why does my execution plan show an estimated query cost of 0.000231 and what does this number mean?

Estimated query cost is a relative number representing the cost of a query compared to all queries in your database.

- The number itself has no absolute meaning; it is only valuable for comparing one query to another or the same query over time
- A cost of 0.000231 for a simple query is typical; costs above 0.1 indicate expensive queries that would benefit from optimization
- Estimated cost is based on estimated rows, not actual rows, so if your statistics are inaccurate, the cost estimate is also inaccurate
- Estimated cost alone is not useful for identifying slow queries; use Query Store to compare estimated cost to actual elapsed time
- Two queries with similar estimated costs may have very different actual performance depending on how accurate the cardinality estimates are
- Focus on optimizing queries that consume the most actual resources shown in Query Store or dm_exec_requests, not just estimated cost

---

## 4. KEY LOOKUPS

### What is a key lookup and why does it cause performance problems?

A key lookup occurs when a non-clustered index is used to find rows based on search criteria, but the index does not contain all columns needed in the SELECT list, so SQL Server must look up the clustered index to retrieve the missing columns.

- A key lookup adds extra I/O operations to your query because SQL Server must access the clustered index in addition to the non-clustered index
- Each key lookup costs approximately the same as one clustered index seek, so thousands of key lookups can cause significant performance degradation
- You see key lookups in execution plans as a distinct operation after an Index Seek, with a line connecting them showing the data flow
- Key lookups are most problematic when combined with expensive predicates on the non-clustered index, because you perform the overhead for many rows
- Add missing columns to the non-clustered index as INCLUDE columns to eliminate key lookups; INCLUDE columns are stored in the index leaf level
- Convert a non-clustered index to a covering index by including all columns needed for the query in the index

### How do I identify which columns are causing key lookups in my queries?

The execution plan and index properties clearly show which columns are missing from the index and causing key lookups.

- In the execution plan, locate the Key Lookup operator and review the "Output List" property to see which columns are retrieved during the lookup
- Compare the Output List columns to the index definition; columns not in the index are the ones causing the lookup
- Check the "Lookup Column" property to see which column from the non-clustered index is used to find rows in the clustered index
- Query sys.dm_db_missing_index_details to find recommendations for index improvements; missing columns are often listed here
- Run sp_helpindex on the table to review index definitions and identify which columns are missing from your non-clustered indexes
- Examine the SELECT list and WHERE clause of your query to understand which columns are actually needed and should be included in the index

### Should I add all columns to every index to eliminate key lookups?

Adding columns to indexes improves query performance but increases index maintenance overhead and storage requirements.

- Include columns only in indexes that are used frequently by slow queries; adding columns to rarely used indexes wastes resources
- Measure the performance improvement of adding INCLUDE columns before and after applying the change to confirm it resolves the key lookup issue
- Wide indexes consume more disk space and memory, increasing the cost of index maintenance operations like INSERT, UPDATE, and DELETE
- Consider consolidating multiple indexes on the same table into one covering index instead of maintaining separate indexes with different INCLUDE columns
- Use the execution plan to identify which queries generate the most key lookups; focus on optimizing the high-impact queries first
- Monitor index fragmentation after adding INCLUDE columns; fragmented indexes with many columns cause more I/O operations

### What is the difference between a key lookup on a non-clustered index and a clustered index scan?

Both operations retrieve data from the clustered index, but the path and efficiency differ significantly.

- A key lookup happens after an Index Seek on a non-clustered index finds qualifying rows, then looks up the clustered index to get remaining columns
- A clustered index scan must read all rows in the clustered index to find matching rows, then retrieves all needed columns immediately
- Key lookups scale better when the non-clustered index narrows down rows significantly, because you perform few lookups
- Clustered index scans scale worse as tables grow because they must read all data regardless of how many rows match your criteria
- Key lookups are more efficient than clustered index scans for queries that retrieve a small percentage of rows from large tables
- If your non-clustered index is not very selective, a clustered index scan may actually be faster because it avoids the double I/O

---

## 5. DEADLOCKS

### What causes deadlocks in SQL Server and how do I detect them?

A deadlock occurs when two or more processes hold locks that each other needs, causing all processes to wait indefinitely.

- Deadlocks happen when transactions acquire locks in different orders; process A locks resource 1 then requests resource 2, while process B locks resource 2 then requests resource 1
- SQL Server detects deadlocks by running a background detection routine; when found, it chooses one process as the deadlock victim and rolls back that transaction
- Enable trace flag 1222 to capture deadlock information in the SQL Server error log; this shows exactly which processes and tables are involved
- Query sys.dm_exec_requests to see blocking relationships between sessions in real time during deadlock investigations
- Set up an alert in SQL Server Agent to notify you immediately when deadlock error 1205 is raised
- Check application error logs for deadlock exceptions; the application usually receives the error before you see it in SQL Server

### What is the deadlock detection strategy in SQL Server and how often does detection run?

SQL Server has a background process that regularly searches for deadlock cycles in the lock wait graph.

- The deadlock detection algorithm runs periodically, not continuously, so a deadlock cycle may exist for a brief period before detection occurs
- The search algorithm identifies cycles in the lock wait graph; if process A waits for process B and process B waits for process A, a cycle exists
- When a deadlock is detected, SQL Server terminates one transaction and rolls it back, freeing locks and allowing other transactions to proceed
- SQL Server chooses the deadlock victim based on DEADLOCK_PRIORITY and the cost of rolling back; lower priority processes are more likely to be victims
- Very high frequency deadlock detection (trace flag 1204) can reduce throughput because the detection routine consumes CPU resources
- Deadlocks are not always bad; infrequent deadlocks that retry automatically may cause less total blocking than pessimistic locking strategies

### How do I troubleshoot a deadlock by analyzing the deadlock graph?

The deadlock graph captured in the error log shows exactly which processes, tables, and locks are involved.

- Enable trace flag 1222 and run SET STATISTICS IO ON to capture full deadlock information; review the error log for deadlock graph details
- Identify the two processes involved; the deadlock graph shows process-list with spid (session ID) and the query each session executed
- Locate the resource-list section showing which table, index, or resource is involved in the deadlock
- Find the deadlock-victim spid tag to see which process was chosen as the victim; this process is rolled back
- Examine lock mode for each process; deadlocks require each process to hold an incompatible lock type that the other needs
- Run the queries shown in the deadlock graph individually against your current database to understand their execution plans and lock behavior

### What strategies can I use to prevent deadlocks in SQL Server applications?

Preventing deadlocks requires consistent transaction design and appropriate lock strategies.

- Always access tables in the same order within transactions; if all transactions access Table A before Table B, deadlock cycles cannot form
- Keep transactions short and focused on a single logical unit of work; long transactions increase the likelihood of conflicts
- Use lower isolation levels like READ COMMITTED when possible; higher isolation levels like SERIALIZABLE acquire more locks and increase deadlock risk
- Add error handling in your application to catch deadlock exceptions (error 1205) and retry the transaction automatically
- Use NOLOCK hints on SELECT queries that do not require absolute currency, reducing lock contention and deadlock risk
- Avoid queries that scan large ranges of rows; use indexes to make queries more selective and reduce lock scope

### How do I use Extended Events to capture detailed deadlock information?

Extended Events provides more detailed deadlock information than trace flags alone.

- Create an Extended Event session to capture the sqlserver.xml_deadlock_report event, which provides the complete deadlock graph
- Run CREATE EVENT SESSION and specify ADD EVENT sqlserver.xml_deadlock_report to start capturing all deadlocks
- Review captured deadlock information in the Extended Events viewer to see the exact query, tables, lock modes, and processes involved
- Export Extended Event data to files for analysis in Excel or use T-SQL queries to parse the XML data
- Set up alerts in SQL Server Agent to notify your team when deadlocks are captured
- Use Extended Events data to identify patterns; if the same two queries deadlock repeatedly, it points to a specific application issue to fix

---

## 6. DATABASE BLOCKING

### What is blocking in SQL Server and how does it differ from deadlock?

Blocking occurs when one transaction holds a lock that prevents another transaction from acquiring a lock it needs; deadlock involves mutual blocking.

- Blocking is asymmetrical; Process A blocks Process B, but Process B is not necessarily blocking Process A
- Blocking is not automatic termination; it continues until the blocking process commits or rolls back its transaction
- Deadlock is mutual blocking; each process blocks the other, and SQL Server terminates one process to resolve it
- Light blocking is normal and expected; excessive blocking indicates lock contention from long transactions or inefficient queries
- Blocking sessions accumulate under the blocking session, forming a chain of blocked sessions waiting for one session to release locks
- Blocking causes application slowness because user requests queue behind blocking transactions instead of executing in parallel

### How do I identify which session is causing blocking?

Several system views and procedures help identify blocking sessions and the queries they are running.

- Query sp_who2 to see all active sessions; look for sessions where blk column shows a non-zero session ID; that is the blocking session
- Use sp_who2 to find the SPID of the blocking session, then find the batch or query that session is executing
- Query sys.dm_exec_requests to see blocking relationships; blocking_session_id column shows which session is blocking the current session
- Use sp_getapplock to see if application locks are involved; these may block sessions even after SQL Server locks are released
- Run sp_helptext on the blocking session SPID to view the query or stored procedure running in that session
- Check sys.dm_tran_locks to see what locks the blocking session holds and which resources are locked

### How do I clear blocking sessions without stopping SQL Server?

Blocking sessions can be terminated to restore performance, but you must first identify whether the data can be safely rolled back.

- Identify the blocking session SPID using sp_who2 or sys.dm_exec_requests
- Review the query or transaction being executed by the blocking session to understand what changes would be lost if you terminate it
- Kill the blocking session using KILL SPID command only if you are certain the data changes can be safely rolled back
- Set a wait timeout before killing; some applications automatically retry after timeout, which may resolve the conflict
- Consider using KILL SPID WITH STATUSONLY to check if the kill command is completing; this provides kill progress
- Implement timeouts in your application connection strings to prevent indefinite waiting on blocked queries

### What locking strategies reduce blocking in SQL Server?

Changing isolation levels and transaction scope helps minimize lock conflicts and blocking.

- Use READ COMMITTED isolation level (default) for most OLTP workloads; it balances consistency and concurrency
- Implement READ UNCOMMITTED isolation level for read-heavy queries that do not require absolute currency, eliminating read blocking
- Use SNAPSHOT isolation level for long-running reports that need point-in-time consistency without holding locks
- Minimize the time between acquiring a lock and releasing it by keeping transactions short and focused
- Break multi-table transactions into multiple single-table transactions if the sequence of updates does not require atomic consistency
- Use indexes on columns used in WHERE clauses to make queries more selective; selective queries hold fewer locks

### How do I monitor blocking over time to identify systemic lock contention?

Ongoing monitoring reveals patterns of lock contention that indicate systemic application issues.

- Create a job that runs every 5 minutes to capture current blocking state using sys.dm_exec_requests and insert into a monitoring table
- Query your monitoring table to find sessions that block other sessions repeatedly or for extended periods
- Generate reports showing peak blocking times and which queries are most frequently blocking
- Set up alerts to notify your team when blocking_session_id is not NULL for more than 10 consecutive samples
- Correlate blocking patterns with application activities; certain operations may consistently cause lock contention
- Review application code to find opportunities to reduce transaction scope or improve query efficiency based on blocking patterns

---

## 7. QUERY TUNING

### What is the systematic approach to tuning a slow query?

Effective query tuning follows a repeatable process that identifies the true bottleneck and tests solutions.

- Capture the actual execution plan for the query and compare estimated rows to actual rows to identify if statistics are accurate
- Check execution time and resource consumption using SET STATISTICS IO and SET STATISTICS TIME to quantify the problem
- Review the execution plan for expensive operations like Table Scans, Key Lookups, Nested Loop Joins on large rowsets, or expensive sorts
- Test potential solutions like adding indexes, updating statistics, or rewriting the query logic one change at a time
- Measure performance improvement after each change; discard changes that do not significantly improve performance
- Document the tuning solution and create a monitoring alert to ensure performance does not degrade if query patterns change

### Why does my query become slow after data volume increases, even though it was fast with small data?

Execution plan optimality depends on table size; plans efficient for small tables may not scale to large tables.

- SQL Server statistics show row count and column distributions; as data grows, the statistics change, and SQL Server may choose different execution plans
- Small data allows Table Scans to be efficient because reading all rows is fast; large data makes Table Scans inefficient
- Indexes that were not beneficial for small tables become beneficial for large tables, and SQL Server may now choose to use them
- Run UPDATE STATISTICS on all tables after major data loads; inaccurate statistics prevent SQL Server from choosing appropriate plans
- Review the execution plan with large data volume; a good plan for 1000 rows may be terrible for 100 million rows
- Add indexes on columns used in WHERE clauses and JOIN conditions if they do not exist; they significantly improve performance at scale

### How do I decide whether to add an index or rewrite a query to fix performance issues?

Both index addition and query rewrites can improve performance; the choice depends on cost and maintainability.

- If the query plan shows a Table Scan on a column used in the WHERE clause, add an index first; this is usually the fastest solution
- If the query uses complex logic, correlated subqueries, or multiple aggregations, rewriting may be more effective than adding indexes
- Consider index maintenance cost; wide indexes or many indexes on a table slow down INSERT, UPDATE, and DELETE operations
- Estimate the space and maintenance cost of adding an index versus the expected performance improvement
- Test both solutions in a development environment and compare query performance and system impact
- Choose the solution that improves query performance with the least overall impact to the application and database maintenance

### What common query patterns indicate a need for tuning?

Certain query structures consistently cause performance problems and should be recognized as red flags.

- Correlated subqueries in the SELECT list force re-execution for every row; rewrite using JOIN or window functions
- Functions applied to columns in WHERE clauses prevent index usage; move functions out of WHERE clauses when possible
- DISTINCT on large result sets indicates the query is retrieving duplicate rows and could be rewritten more efficiently
- OR conditions in WHERE clauses reduce index selectivity; split into separate queries with UNION ALL if indexes exist for each condition
- LIKE patterns with leading wildcards like '%text' force Table Scans; rewrite or use full-text search for text searches
- NOT IN or NOT EXISTS clauses may cause Table Scans; test performance and consider rewriting with LEFT JOIN and IS NULL

### How do I use baseline performance data to measure tuning success?

Baseline data enables objective measurement of improvement, not just subjective perception of faster performance.

- Capture query execution time before any tuning changes using SET STATISTICS TIME
- Record the number of logical reads before tuning using SET STATISTICS IO
- Note CPU time and elapsed time from Query Store for the query before making changes
- After implementing tuning changes, run the same measurements and compare results
- Calculate the percentage improvement; 50 percent faster means 50 percent reduction in execution time or logical reads
- Monitor performance after one week to confirm the improvement persists; some tuning may improve initial response but not sustained performance

---

## 8. LONG RUNNING QUERY

### How do I find and identify which query is running long in production?

Several tools and views help identify slow queries currently executing or recently completed.

- Query sys.dm_exec_requests filtered by session_id where status = 'running' to see active queries; sort by total_elapsed_time to find slowest
- Use sp_who2 to see active sessions and their commands; sessions with high CPU or high elapsed time are candidates for investigation
- Check sys.dm_exec_query_stats for recently executed queries; longest_elapsed_time shows which queries took longest on recent executions
- Use Query Store to view all executed queries ranked by average duration or total CPU time
- Enable Extended Events to capture long running queries automatically; create a session that captures queries exceeding a threshold like 30 seconds
- Review SQL Server error log for query timeout messages if your application has query timeout settings

### What causes queries to run longer than expected?

Slow queries result from inefficient query plans, resource contention, or excessive work.

- Table Scans on large tables cause queries to read all data instead of using indexes to narrow down rows quickly
- Key Lookups on non-clustered indexes add extra I/O; the query must access the index, then the clustered index, for each row
- Expensive join algorithms like Nested Loop Joins with millions of rows cause exponential work; Hash or Merge Joins are more efficient
- Missing indexes force the query to scan tables that could be seeked with appropriate indexes
- High memory grant sizes indicate the query is sorting or hashing large amounts of data; optimizing the query to require less data reduces memory usage
- Blocking from other long-running transactions prevents the query from acquiring locks; the query waits instead of executing

### How do I capture the execution plan of a currently running long query?

You can capture the actual execution plan of a query that is still executing in real time.

- Identify the SPID of the long running query using sp_who2 or sys.dm_exec_requests
- Run SET STATISTICS IO ON and SET STATISTICS TIME ON in a separate connection to capture plan details
- Run sp_whoIsActive from Adam Machanic to get detailed information including the query text and plan
- Use extended events or profiler to capture the execution plan for the query; remember to stop tracing immediately after capturing
- Capture the plan multiple times if the query is still running; the plan may change as the query progresses through different phases
- Save the plan to XML for comparison with baseline plans or other slow queries

### What immediate actions can I take to speed up a long running query without modifying code?

Sometimes you can improve performance without changes to the application code.

- Kill the query if it is estimated to take hours and is not providing value; rewrite and redeploy the query
- Update statistics on relevant tables using UPDATE STATISTICS; inaccurate statistics cause inefficient plans
- Add indexes on columns used in WHERE clauses and JOIN conditions if indexes do not exist; this often provides immediate improvement
- Check for blocking; if the query is waiting for locks instead of executing, resolve the blocking issue
- Increase server resources temporarily like CPU or memory if resource contention is the bottleneck; this is a short-term fix
- Force a cached execution plan using Query Store if a better plan exists from a previous execution

### How do I establish a performance baseline and alert when queries exceed it?

Ongoing monitoring with alerts enables you to catch performance degradation early.

- Run expected-execution-time queries during normal business hours and record average elapsed time and CPU time
- Set a threshold at 150 percent of baseline; for example, if a query normally runs 1 second, alert when it exceeds 1.5 seconds
- Create a job that runs every 15 minutes to query sys.dm_exec_query_stats and compare recent execution times to baseline
- Send an alert via SQL Server Agent or email when a query exceeds the threshold more than 3 times in a row
- Include the execution plan in the alert so the on-call DBA can investigate immediately
- Review baseline expectations weekly; adjust if application behavior or data volume changes

---

## 9. INDEX SEEK

### What is an Index Seek and why is it more efficient than other data access methods?

An Index Seek uses the index structure to navigate directly to rows that match your criteria instead of reading all rows.

- Index Seek uses the B-tree structure of the index to find the first matching row, then scans forward only the matching rows
- Seek efficiency depends on how selective the index is; an index that narrows down from 1 million rows to 100 rows is more efficient than one that finds 900,000 rows
- Seek reads only the rows needed plus the index overhead, while other methods must read all or most rows from the table
- The query optimizer chooses Seek when it estimates that using the index will retrieve fewer rows than a Table Scan
- Seek efficiency decreases as the selectivity decreases; if an index returns 90 percent of rows, a Scan may be faster
- Multiple predicates in WHERE clause increase the number of predicates the index can use; more predicates mean more selective seeks

### What conditions enable SQL Server to use an Index Seek instead of an Index Scan?

Index Seek requires specific index structure and query conditions to be viable.

- Index on the column must exist; without an index, Table Scan is the only option
- The WHERE clause must reference the leftmost columns of the index in an order that allows the index to navigate efficiently
- Predicates must use comparison operators like equals, greater than, or BETWEEN; LIKE with leading wildcards or functions prevent Seek
- Column statistics must indicate the predicate is selective; if statistics show the predicate matches most rows, Scan may be more efficient
- The index must have sequential keys that allow the optimizer to estimate which rows match; non-sequential keys reduce efficiency
- Composite indexes must use predicates on the leading columns; predicates only on trailing columns may not enable Seek

### How do I encourage SQL Server to use an Index Seek on specific columns?

Creating appropriate indexes on the correct columns is the primary way to enable Index Seek.

- Create an index on columns used in WHERE clauses; the index should have leading columns that match your WHERE conditions
- For composite indexes, order columns from most selective to least selective; the most selective columns should be leading columns
- Include columns in the SELECT list as INCLUDE columns if they are not part of the search key; this eliminates Key Lookups
- Create filtered indexes if a subset of rows is queried frequently; filtered indexes are smaller and more selective than full table indexes
- Remove or disable indexes that are not used; extra indexes slow down INSERT and UPDATE operations without benefiting queries
- Use Index Hints cautiously; instead of forcing Seek with INDEX (IndexName), create a better index and let the optimizer choose naturally

### What does Seek Predicate versus Residual Predicate mean in the execution plan?

Seek Predicate and Residual Predicate show how much work is done by the index versus after the index.

- Seek Predicate shows the predicates that the index uses to navigate and find matching rows; this work is fast
- Residual Predicate shows predicates applied after rows are retrieved from the index; these predicates cannot use the index
- Ideally, all predicates should be Seek Predicates; if important predicates are Residual, the index is not optimally structured
- If a predicate is Residual, SQL Server applies it to every row retrieved by the Seek; this adds CPU work for each row
- Move important predicates to Seek by creating indexes with those columns as leading columns or making the WHERE clause match the index structure
- Residual Predicates are not a failure; they are normal when predicates do not match the index key columns

---

## 10. INDEX SCAN

### What is an Index Scan and when does SQL Server choose to Scan instead of Seek?

An Index Scan reads all rows in the index or table to find matching rows, similar to a Table Scan but using the index.

- Index Scan occurs when the WHERE clause predicates cannot be used to navigate the index, or when the index is not selective enough to justify Seek
- Scan reads all leaf pages of the index sequentially, making it slower than Seek for selective predicates but faster than Table Scan
- SQL Server chooses Scan when statistics indicate that the predicate matches most rows; Seeking through most rows plus navigating the B-tree is slower than Scan
- Scan is sometimes more efficient than Seek for queries that need to retrieve large percentages of rows from a table
- Scan requires fewer logical reads than Seek for large result sets because it does not navigate the index B-tree multiple times
- Scan efficiency depends on index fragmentation; highly fragmented indexes cause more physical I/O than defragmented indexes

### Why does my WHERE clause cause an Index Scan instead of Index Seek?

The query structure determines whether an index can be used for Seek; certain patterns force Scan.

- Functions applied to index columns like WHERE UPPER(ColumnName) = 'VALUE' prevent Seek because the function must be evaluated for every row
- LIKE predicates with leading wildcards like WHERE ColumnName LIKE '%Text' cannot use index navigation; Scan is required
- OR conditions that reference different columns may require Scan if no single index covers all conditions efficiently
- Predicates only on non-leading columns of a composite index do not enable Seek; leading columns must be in the predicate
- NOT conditions like WHERE ColumnName NOT IN (list) usually cause Scan because the index cannot efficiently find non-matching rows
- Complex predicates that cannot be simplified may cause Scan; rewrite predicates or add indexes to enable Seek

### How do I optimize a query to use Index Seek instead of Index Scan?

Changing the query or adding indexes shifts the query from Scan to more efficient Seek operations.

- Remove functions from column predicates; rewrite WHERE UPPER(ColumnName) = 'VALUE' as WHERE ColumnName = 'VALUE' if the column values can be adjusted
- Use LIKE patterns without leading wildcards; WHERE ColumnName LIKE 'Text%' enables Seek while WHERE ColumnName LIKE '%Text' requires Scan
- Create indexes with leading columns that match your WHERE predicates; the index structure determines whether Seek is possible
- For composite predicates, create composite indexes with columns in order of selectivity; most selective columns first
- Simplify complex predicates; break them into multiple simpler queries that each use a Seek operation
- Monitor the percentage of rows returned; if you are retrieving most rows, Scan may be more efficient than many Seeks

---

## 11. TABLE SCAN

### What is a Table Scan and why does SQL Server perform them?

A Table Scan reads every row from the table or clustered index to find rows matching your criteria.

- Table Scan is performed when no appropriate index exists for the query predicates
- SQL Server performs Scan when it estimates that reading all rows is faster than navigating an index to find matching rows
- Scan efficiency depends on table size; Scan on a 1000-row table is acceptable, but on a 100-million-row table, it consumes significant resources
- Parallel Table Scans split the table into multiple segments and scan them in parallel, improving performance on large tables and multi-processor systems
- Table Scan is the fallback operation when no index is available; adding appropriate indexes typically converts Scans to Seeks
- Scan is not always bad; for small tables or when retrieving most rows, Scan is actually more efficient than Seek

### Why does adding an index not eliminate a Table Scan from my query?

Index addition does not always result in Seek if the index structure or query predicates do not match.

- If the WHERE clause uses a function like UPPER or SUBSTRING, the index cannot be used even if it exists on that column
- If the WHERE clause references non-leading columns of a composite index, Seek may not be possible; the leading columns must be in the predicate
- If the index is on a different table than the query references, it does not affect that table's Scans
- If the statistics on the new index are stale, SQL Server may estimate that Scan is faster than Seek
- If the column has low selectivity, SQL Server may correctly estimate that Scan is faster than Seek for most rows
- Run DBCC FREEPROCCACHE to clear cached plans after adding an index; the cached plan may still use the old Table Scan strategy

### How do I identify which tables are being scanned in my workload?

Finding frequently scanned tables helps identify high-impact optimization opportunities.

- Query sys.dm_db_index_usage_stats filtered by user_seeks = 0 and user_lookups = 0 to find indexes with only scans
- Sort by user_scans descending to identify which tables have the most scans
- Review the query that performs the scan using sys.dm_exec_query_stats to understand the scan predicate
- Look for tables where user_scans is much higher than user_seeks; this indicates opportunities for better indexes
- Run DBCC DROPCLEANBUFFERS and then review sys.dm_db_index_usage_stats after a typical workload to see the most impactful scans
- Create alerts for tables scanned more than 1000 times per hour; these tables should have indexes to support scan queries

### What strategies eliminate Table Scans from high-volume queries?

Eliminating unnecessary Scans on large tables can dramatically reduce CPU and I/O consumption.

- Add indexes on columns used in WHERE clauses; appropriate indexes enable Seek instead of Scan
- Partition large tables by a column like date or customer ID; partitioned tables can sometimes scan only the relevant partition
- Use filtered indexes to create smaller indexes on subsets of data; queries filtering on the filter predicate use the smaller index
- Rewrite queries to use different logic that is more index-friendly; for example, avoid NOT conditions that may require Scan
- Consider full-text search indexes if querying text columns; full-text indexes support LIKE and text predicates without Table Scan
- Implement columnstore indexes for analytical queries; columnstore indexes support efficient scans of large datasets

---

## 12. DATABASE CORRUPTION

### What is database corruption and what are the common causes?

Database corruption occurs when the data or structure stored in the database files becomes inconsistent or damaged.

- Hardware failures like failing disk drives write corrupted data to database files; data is lost or overwritten incorrectly
- Sudden power loss during write operations leaves the database in an inconsistent state; SQL Server recovery cannot repair all damage
- SQL Server crashes or forceful termination while writing data can leave data structures inconsistent
- Memory errors cause SQL Server to write incorrect data to disk or read incorrect data from disk
- Malware or unauthorized access to database files can damage data structures intentionally or unintentionally
- Storage array failures or network issues during database restore operations can introduce corruption

### How do I use DBCC CHECKDB to detect corruption?

DBCC CHECKDB is the primary tool for detecting logical and physical corruption in SQL Server databases.

- Run DBCC CHECKDB (DatabaseName) to scan all database structures and identify inconsistencies
- DBCC CHECKDB performs checks on allocation pages, index structures, table structures, and cross-page references
- Review the output for messages mentioning "corrupt", "inconsistent", or "error"; these indicate detected corruption
- Severity of corruption is indicated by the number of errors found; a few errors may indicate minor corruption while hundreds indicate major damage
- Run DBCC CHECKDB with REPAIR_REBUILD to repair errors automatically; this option rebuilds corrupted indexes
- Run DBCC CHECKDB with REPAIR_ALLOW_DATA_LOSS if corruption is severe; this option may delete corrupted pages to restore consistency

### What do I do when DBCC CHECKDB reports corruption?

Corruption requires immediate response to restore the database to a consistent state.

- Stop all applications from accessing the database to prevent additional corruption or data loss
- Create a backup of the corrupted database before running repairs; you may need the backup if repair attempts cause further issues
- Run DBCC CHECKDB with SINGLE_USER mode to prevent other connections interfering with the repair
- Run DBCC CHECKDB with REPAIR_REBUILD first to attempt non-destructive repairs; this fixes most issues
- If REPAIR_REBUILD reports errors that cannot be fixed, run DBCC CHECKDB with REPAIR_ALLOW_DATA_LOSS to remove corrupted data
- Restore the database from a clean backup if corruption is severe; this is often faster and safer than attempting repairs

### How do I identify which table or index is corrupted?

DBCC CHECKDB output shows the specific object that corruption affects.

- Review DBCC CHECKDB output messages to identify the table name and index name mentioned in error messages
- Look for patterns in error messages; if all errors mention a specific table, that table is most likely corrupted
- Run DBCC CHECKTABLE (TableName) to focus on a specific table if you suspect it is corrupted
- Query sys.objects to find the object ID from the error message and map it to a table or index name
- Run DBCC CHECKDB (DatabaseName, NOINDEX) to check tables only, skipping indexes; this identifies whether indexes or tables are corrupted
- Review the repair log files if available; SQL Server writes detailed repair information to recovery logs

### How do I prevent database corruption?

Preventive measures reduce corruption risk significantly.

- Maintain hardware correctly; replace failing drives before they fail completely
- Install battery-backed write cache on storage systems to ensure data is safely written even during power loss
- Keep SQL Server and the operating system updated with the latest patches; many corruption-related bugs are fixed in patches
- Monitor disk health using storage array monitoring; watch for SMART errors that indicate imminent drive failure
- Enable checksum validation for pages using CHECKSUM or TORN_PAGE_DETECTION; this detects corruption when pages are read
- Test restore procedures regularly to ensure you can recover from corruption; backups are useless if you cannot restore them

---

## 13. TABLE CORRUPTION

### What is table corruption and how does it differ from database corruption?

Table corruption is a subset of database corruption affecting only the table data structures.

- Table corruption occurs when row storage within the table is damaged, making some rows unreadable or inaccessible
- Table corruption may affect all rows or only specific rows depending on which data pages are damaged
- Database corruption can affect index structures, allocation structures, or system tables; table corruption is limited to user table data
- A single corrupted table can often be isolated and repaired without affecting other tables
- Queries that reference corrupted rows fail with error messages about incorrect page structure
- Corrupted tables with REPAIR_ALLOW_DATA_LOSS lose affected rows permanently; they cannot be recovered

### How do I detect which rows in a table are corrupted?

Identifying corrupted rows helps you determine the scope of data loss and prioritize recovery efforts.

- Run DBCC CHECKTABLE (TableName) to report corruption specific to that table
- Try to select from the table; SQL Server usually reports an error with the page and row that is corrupted
- Run SELECT queries that scan the entire table; queries that access corrupted rows fail while others succeed
- Review SQL Server error messages mentioning "8939", "823", or "824" which indicate page corruption
- Export unaffected rows to a new table; then restore or rebuild the table from backup
- Use UNION ALL to query multiple tables or versions of the same table; rows that query from one table but not another are likely corrupted

### What is the recovery process for a corrupted table?

Recovery depends on whether a good backup exists and what corruption level is acceptable.

- Restore the table from a recent clean backup using RESTORE DATABASE if a backup exists; this is the fastest and safest recovery
- If no good backup exists, use DBCC CHECKDB with REPAIR_ALLOW_DATA_LOSS to remove corrupted rows and restore table consistency
- Expect data loss if repair is necessary; any rows on corrupted pages are lost permanently
- After repair, verify the table contains expected data by counting rows and checking for missing data compared to your audit logs
- Update statistics on the repaired table using UPDATE STATISTICS to ensure correct query optimization
- Implement monitoring and checksum validation to detect corruption early before major data loss occurs

### How do I prevent table corruption?

Preventive measures protect table data from damage.

- Maintain disk health by monitoring storage array status and replacing failing components
- Enable checksum page verification using ALTER DATABASE SET PAGE_VERIFY = CHECKSUM
- Run DBCC CHECKDB regularly on schedules; weekly for critical databases and monthly for less critical ones
- Maintain backups and test restore procedures regularly; backups are the safety net if corruption occurs
- Monitor for hardware errors in storage subsystem; replace failed components before they cause database corruption
- Keep transaction logs to enable point-in-time recovery if corruption is detected after backup but before your last clean backup

### Can corrupted tables be recovered without data loss?

Data loss during corruption recovery depends on corruption severity and backup availability.

- If a clean backup exists from before corruption occurred, restore the table from that backup with zero data loss
- If backup exists but is older, you can restore the backup and then apply transaction logs from the backup point to the corruption time for nearly zero data loss
- If corruption affects only the index and not the table itself, rebuild the index from the table data with zero data loss
- If corruption affects table row data, you can recover non-corrupted rows by copying them to a new table; corrupted rows are lost
- Enable mirroring or availability groups to maintain a synchronized copy of the database; if primary corrupts, switch to the mirror
- Use REPAIR_REBUILD instead of REPAIR_ALLOW_DATA_LOSS when possible; REBUILD repairs structures while preserving data

---

## 14. CLUSTERED INDEX MAINTENANCE

### Why do clustered indexes require maintenance and what problems does fragmentation cause?

Fragmented clustered indexes degrade query performance because they require additional I/O operations.

- Fragmentation occurs when pages are not stored sequentially; when SQL Server reads the index, it must jump between physical locations on disk
- Severely fragmented indexes cause additional disk I/O; a sequential scan of 1000 pages may jump to hundreds of different disk locations
- Page splits happen during INSERT and UPDATE operations; when a page is full, new rows go to new pages, fragmenting the index
- High fragmentation reduces cache effectiveness; fewer relevant pages fit in memory, increasing disk reads
- Leaf page order is disrupted by fragmentation; index scans become inefficient when next leaf pages are not located sequentially
- Fragment removal through index rebuild or reorganize restores sequential page layout and improves scan performance

### How do I measure index fragmentation and decide when to rebuild or reorganize?

Fragmentation percentage determines whether reorganize or rebuild is most appropriate.

- Query sys.dm_db_index_physical_stats to get avg_fragmentation_in_percent for each index
- Fragmentation below 10 percent requires no action; normal use maintains acceptable fragmentation
- Fragmentation between 10 and 30 percent should be resolved by index reorganize; this online operation removes fragmentation
- Fragmentation above 30 percent requires index rebuild; rebuild completely reconstructs the index with zero fragmentation
- Reorganize is fast and online; rebuild can be offline and slow on very large indexes
- Schedule maintenance during off-peak hours; defragmentation consumes CPU and I/O, impacting user queries

### What is the difference between index rebuild and index reorganize?

Both operations remove fragmentation, but the methods and impacts are very different.

- Index rebuild drops the entire index and rebuilds it from scratch; this takes more time but completely removes fragmentation
- Index reorganize scans the leaf level and reorders pages to be sequential; this is faster but may not remove all fragmentation
- Rebuild statistics are automatically updated as a result of the operation; reorganize does not automatically update statistics
- Rebuild can be done online without blocking queries; offline rebuild may be required on enterprise editions for certain index types
- Reorganize must complete sequentially; large index reorganizes take significant time and consume log file space
- Rebuild is more effective for severe fragmentation; reorganize is adequate for moderate fragmentation

### How do I automate index maintenance to keep performance consistent?

Scheduled maintenance ensures indexes remain defragmented without manual intervention.

- Create a SQL Server Agent job that runs DBCC SHOWCONTIG or sys.dm_db_index_physical_stats every night
- Add logic to rebuild indexes with fragmentation above 30 percent and reorganize indexes with fragmentation above 10 percent
- Schedule rebuild operations during maintenance windows; rebuilds are resource-intensive and should not run during peak hours
- Use maintenance plans in SQL Server Management Studio to automate index maintenance for a group of databases
- Monitor log file growth during maintenance; index operations can cause significant transaction log growth
- Set alerts to notify your team if index maintenance fails or if fragmentation exceeds thresholds

---

## 15. STATISTICS AND CARDINALITY

### What are statistics and how do they affect query optimization?

Statistics enable SQL Server to estimate row counts and distributions, allowing it to choose optimal execution plans.

- Statistics store information about data distribution in columns; they include row count, null count, and distribution histograms
- The cardinality estimator uses statistics to predict how many rows will match query predicates
- Accurate statistics lead to accurate row count estimates and optimal query plan selection
- Inaccurate statistics mislead the optimizer; it may choose inefficient plans for queries with wrong row count estimates
- Missing statistics force SQL Server to use default estimates; these defaults are often inaccurate for selective predicates
- Statistics become stale as data changes; old statistics may not represent current data distribution

### How do I know when statistics are outdated and need updating?

Outdated statistics show inaccurate row count estimates in execution plans.

- Compare estimated rows to actual rows in the execution plan; large differences indicate statistics are outdated
- Query sys.dm_db_stats_properties to see the date statistics were last updated; compare to the last significant data load
- Look for recent INSERT, UPDATE, or DELETE operations that may have changed data distribution significantly
- Run DBCC SHOW_STATISTICS to view the specific statistics histogram; compare to the actual current data distribution
- Query sys.stats_columns joined to sys.tables to find which statistics are available for a table
- Enable automatic update statistics; this enables automatic refresh as data changes, but may delay query execution

### How do I update statistics to improve query performance?

Refreshing statistics helps the optimizer choose better execution plans for queries.

- Run UPDATE STATISTICS (TableName) to update all statistics on a table
- Use FULLSCAN option to analyze all rows; UPDATE STATISTICS (TableName) WITH FULLSCAN ensures accurate statistics
- Run RESAMPLE option to sample rows proportionally to table size; UPDATE STATISTICS (TableName) WITH RESAMPLE is faster than FULLSCAN
- Create a job to run UPDATE STATISTICS nightly on all tables; this ensures statistics are current
- Specifically target high-impact statistics; run UPDATE STATISTICS on columns used in joins and WHERE clauses
- Monitor query plan changes after updating statistics; better plans should result in improved performance

### Why do I see "estimated rows = 1" in the execution plan even though the actual result is 1000 rows?

Row count estimates of 1 indicate SQL Server has very little confidence in the estimate, usually due to complex predicates or missing statistics.

- Complex predicates combining multiple columns reduce statistics usefulness; SQL Server assumes independence between columns
- Missing statistics on columns involved in the predicate prevent accurate estimation; creating statistics on the column may help
- Predicate operations like functions prevent statistics usage; UPPER(ColumnName) = 'VALUE' cannot use statistics on ColumnName
- Outdated statistics combined with recent data changes cause estimates to be wildly inaccurate
- Recompile the query to get a fresh estimate; sometimes the plan cache contains incorrect estimates
- Run DBCC UPDATEUSAGE to update row count metadata; inaccurate row count metadata misleads the estimator

---

## 16. MISSING INDEXES

### How does SQL Server recommend missing indexes?

SQL Server tracks index usage and recommends indexes that would improve query performance.

- Query sys.dm_db_missing_index_details to see all index recommendations made by SQL Server
- sys.dm_db_missing_index_stats shows which queries would benefit from each recommended index and how much improvement is expected
- Improvement_measure column shows the estimated percentage improvement; higher values indicate more impactful indexes
- Join sys.dm_db_missing_index_details and sys.dm_db_missing_index_stats to see the recommended index columns and expected impact
- Filter by equality_columns and inequality_columns to understand which columns should be leading columns of the index
- Review included_columns to see which columns SQL Server recommends including to eliminate key lookups

### What criteria should I use to decide whether to create a recommended missing index?

Not all recommended indexes should be created; evaluate impact before creating.

- Create indexes on frequently executed queries; queries executed thousands of times per day benefit more from index improvement than rarely-run queries
- Focus on indexes with high improvement_measure; indexes with estimated 50 percent improvement are worth creating before those with 5 percent improvement
- Consider the cost of index maintenance; indexes on columns with high INSERT and UPDATE frequency increase write overhead
- Evaluate table size; indexes on small tables with few rows may not provide significant benefit
- Check if similar indexes already exist; consolidate recommendations into fewer indexes instead of creating many single-column indexes
- Test the index in a development environment before creating on production; confirm the improvement matches the estimate

### How do I create missing indexes based on SQL Server recommendations?

Once you have identified a beneficial index, create it carefully and monitor the results.

- Create the index using the CREATE INDEX statement recommended in sys.dm_db_missing_index_details
- Use ONLINE = ON option for large indexes if you are using SQL Server Enterprise Edition; this allows queries to run during index creation
- Monitor index creation progress; large indexes may take minutes or hours to create
- Verify the index improved query performance by comparing execution times before and after creation
- Run UPDATE STATISTICS after creating new indexes; statistics on new indexes are empty initially
- Monitor CPU and I/O after index creation; ensure index maintenance overhead on INSERT and UPDATE does not outweigh query benefits

---

## 17. INDEX FRAGMENTATION

### How does index fragmentation develop over time?

Page splits during data modifications cause logical fragmentation; physical fragmentation develops when new pages are not contiguous on disk.

- When a page is full and a new row must be inserted, SQL Server allocates a new page; the new page is not necessarily adjacent to the current page
- This logical fragmentation causes the index scan to jump between non-sequential pages, increasing I/O operations
- Physical fragmentation occurs when the new pages are not stored sequentially on disk; this increases disk head movement
- Fragmentation increases gradually as data is inserted and updated; it is normal and expected
- Reads of sequential index pages become inefficient; cache misses increase and disk reads increase
- Defragmentation restores sequential page layout and reduces I/O for index scans

### What is the impact of high index fragmentation on query performance?

Heavily fragmented indexes significantly degrade performance for queries that scan the index.

- Fragmented index scans require more disk I/O than sequential scans because the disk head jumps between locations
- Physically fragmented indexes have lower cache hit ratio; the CPU cache becomes less effective when pages are not sequential
- Query response time increases for full index scans; the overhead of non-sequential page access is noticeable for large indexes
- CPU usage may increase because the storage controller and driver spend more resources seeking to non-sequential pages
- Overall system performance degrades when many heavily fragmented indexes are accessed simultaneously
- Defragmentation often resolves performance issues without query optimization or index addition

### What is the recommended fragmentation threshold for defragmentation operations?

Different fragmentation levels require different maintenance strategies.

- Fragmentation below 10 percent is considered good; no maintenance is needed
- Fragmentation between 10 and 30 percent should be addressed by index reorganize; reorganize is fast for this range
- Fragmentation above 30 percent requires index rebuild; reorganize is ineffective for severe fragmentation
- Very large indexes with fragmentation above 30 percent should be rebuilt during off-peak hours; rebuild is resource-intensive
- Monitor fragmentation trends; if fragmentation consistently climbs above 20 percent, increase maintenance frequency
- Use the fragmentation threshold to prioritize maintenance; focus on indexes with fragmentation above 30 percent first

---

## 18. TRANSACTION LOG

### How does the transaction log store data and how does it prevent data loss?

The transaction log records all database modifications, enabling recovery to a consistent point in time.

- Every INSERT, UPDATE, and DELETE operation is recorded in the transaction log before being applied to data pages
- The log record includes the operation type, the affected page, the change details, and a sequence number
- When a transaction commits, the log record is written to disk before control returns to the application; this ensures durability
- If SQL Server crashes after a transaction commits but before the change is written to data pages, the transaction log enables recovery
- During recovery, SQL Server replays committed transactions from the log; uncommitted transactions are rolled back
- The log guarantees ACID properties; transactions are durable even if data pages have not been written to disk

### Why does my transaction log grow very large and how do I control its size?

Transaction log growth is controlled by backup frequency and recovery model selection.

- The log grows as new transactions occur; it can only be reused after transactions are backed up
- In FULL recovery model, the log must be backed up frequently; log backups truncate the log, allowing space to be reused
- Long-running transactions prevent log truncation; the log must retain records for all active transactions
- Bulk operations like INSERT SELECT without logging can cause rapid log growth; use BULK_LOGGED recovery model for large imports
- Monitor log space usage with sp_spaceused; if free space is low, increase log file size or increase backup frequency
- Enable automatic log file growth; even with backups, temporary growth occurs between backup cycles

### How do I recover a database to a specific point in time using the transaction log?

Point-in-time recovery restores the database to a specific time using backup and log restore.

- Restore the most recent full backup using RESTORE DATABASE with NORECOVERY
- Restore all differential backups taken after the full backup with NORECOVERY
- Restore all transaction log backups taken after the differential backup up to the desired time point using RESTORE LOG with STOPAT
- Use STOPAT in the final RESTORE LOG command to stop recovery at the exact time desired
- Verify the recovered database contains data at the desired point in time; check timestamps and row counts
- After successful recovery, run RESTORE with RECOVERY to bring the database online

---

## 19. BLOCKING CHAINS

### What is a blocking chain and how do I trace the blocking hierarchy?

A blocking chain occurs when Session A blocks Session B, and Session B blocks Session C, creating a wait hierarchy.

- Use sp_who2 to see the blocking hierarchy; the blk column shows which session is blocked by which
- Start with the session at the top of the chain; this is usually the session holding locks that block all others
- The top session typically has blk = 0; all other sessions in the chain show a non-zero blk value
- Kill the top-level blocking session to unblock all sessions below it; killing a session in the middle does not unblock the chain
- Trace each session downward; Session A blocks Session B which blocks Session C, forming a chain
- Identify the query running in the top-level blocking session and understand why it is taking so long

### How do I find long blocking chains and prioritize which sessions to kill?

Blocking chains vary in length and impact; prioritize killing the top-level session to unblock many other sessions.

- Use sys.dm_exec_requests to find sessions where blocking_session_id is not NULL; these are blocked sessions
- Count how many sessions are blocked by each blocking session; sessions blocking many others should be addressed first
- Look for blocking chains longer than 5 sessions; these indicate serious lock contention affecting many users
- Query the blocking session to understand its purpose; killing critical jobs may not be appropriate even if they block others
- Check if the blocking session is making progress using sp_whoIsActive; sessions stuck in infinite loops should be killed
- Implement timeouts in your application to prevent indefinite waiting on blocked queries; clients timeout and retry instead of waiting forever

---

## 20. MAINTENANCE WINDOW OPERATIONS

### What maintenance operations should run during off-peak hours to minimize impact?

Certain maintenance operations consume significant resources and should be scheduled carefully.

- Index rebuild requires exclusive locks and heavy CPU and I/O; schedule during off-peak hours
- Database consistency checks with DBCC CHECKDB consume CPU and I/O; run during maintenance windows
- Full backups read all data pages and consume I/O; consider running during off-peak hours or on secondary replicas
- Statistics updates scan table data and consume CPU; schedule when query load is low
- Transaction log backup frequency should remain high even during peak hours to prevent log runaway
- Use maintenance plans to schedule operations automatically; SQL Server Agent can run scheduled jobs

### How long do maintenance operations typically take and how do I estimate completion time?

Maintenance duration depends on database size and the specific operation.

- Index rebuild time is proportional to index size; expect 5 to 60 minutes for large indexes
- DBCC CHECKDB time depends on database size; expect 30 seconds for small databases to hours for large databases
- Full backup time depends on data volume; expect 30 seconds to 30 minutes depending on backup device and data compression
- Statistics update time depends on table size and statistics count; full scan statistics take longer than sampled statistics
- Use information_schema and sys.dm_db_index_physical_stats to estimate maintenance time before running operations
- Test maintenance operations in a development environment on a copy of production data to estimate completion time

---

## 21. WAIT STATISTICS

### What are SQL Server wait statistics and what information do they provide?

Wait statistics show what resources sessions are waiting for, revealing the true bottleneck.

- SQL Server tracks time spent waiting for CPU, disk I/O, locks, latches, and network resources
- Waiting on CPU indicates insufficient CPU resources; queries are ready to run but CPU is busy
- Waiting on disk I/O (PAGEIOLATCH) indicates slow disk performance or missing indexes causing excessive I/O
- Waiting on locks (LCK_M_*) indicates lock contention; other sessions are blocking the current session
- Waiting on latches indicates internal SQL Server contention for internal structures
- Wait statistics reveal the actual problem; high CPU wait indicates CPU tuning is needed, not I/O optimization

### How do I use wait statistics to identify performance bottlenecks?

Wait statistics provide objective data about which resources are constrained.

- Query sys.dm_db_wait_stats to see all wait types and the total time sessions spent waiting for each wait type
- Sort by wait_time_ms descending to find which wait types consume the most time
- Calculate wait_time_ms divided by total_waits to find average wait time per occurrence
- High average wait time indicates that individual waits are long; fix the resource causing the wait
- High total wait time indicates that many waits are occurring; fix the underlying cause of the waits
- Compare wait statistics before and after applying optimization; confirm wait time decreases

### What are the most common wait types and what do they indicate?

Common wait types point to specific resource issues.

- PAGEIOLATCH_SH and PAGEIOLATCH_EX indicate disk I/O bottlenecks; improve I/O performance or optimize queries
- LCK_M_* waits indicate lock contention; review blocking chains and transaction isolation levels
- PREEMPTIVE_OS_* waits indicate the query is waiting for the operating system; these are often I/O related
- CXPACKET waits indicate parallelism waits; the query is waiting for parallel task execution to complete
- WRITELOG waits indicate the query is waiting for transaction log writes; improve log I/O performance
- SLEEP waits indicate intentional waits; queries using WAITFOR or scheduled jobs produce these waits

---

## 22. TEMPDB MANAGEMENT

### What is tempdb and what workloads use it most heavily?

Tempdb is a system database that stores temporary tables, work tables for sorting and hashing, and version stores.

- Every SQL Server instance has one tempdb database; it is shared by all user databases and system operations
- Temporary tables created with CREATE TABLE #TableName are stored in tempdb
- Table variables declared with DECLARE @TableName TABLE are stored in tempdb
- Sorting and hashing operations that spill to disk use tempdb; these operations use tempdb when the sort or hash data exceeds memory
- SNAPSHOT isolation level stores row versions in tempdb; long-running queries with SNAPSHOT isolation prevent tempdb cleanup
- Tempdb contention often occurs when many sessions create temporary tables or perform sort operations simultaneously

### How do I identify tempdb contention and reduce it?

Tempdb contention causes reduced concurrency and slower query execution.

- Monitor tempdb file usage; if multiple sessions are creating temporary tables, tempdb I/O becomes a bottleneck
- Query sys.dm_tran_version_store to see how much version store space is used; long-running transactions prevent cleanup
- Look for many sessions waiting on tempdb I/O; high average waits on PAGEIOLATCH indicate tempdb contention
- Add multiple tempdb data files (one per CPU core) to distribute I/O across disks
- Increase tempdb file size; pre-allocate space to prevent contention from auto-growth
- Use table variables instead of temporary tables for small data sets; table variables use different storage
- End long-running transactions quickly; they prevent tempdb cleanup and consume version store space

---

## 23. LOG WAIT STATISTICS

### Why do I see high WRITELOG wait times and what does it indicate?

High WRITELOG waits mean queries are waiting for transaction log writes to complete.

- WRITELOG waits indicate that the log disk is slow or that too many transactions are writing to the log
- During heavy transactional load, queries must wait for their log records to be written and hardened to disk
- If log disk I/O is the bottleneck, move the log to a faster disk or add more disks for the log
- Reduce log writes by consolidating small transactions into batch operations; fewer transactions means fewer log writes
- Use asynchronous I/O for log writes if available on your storage system; this reduces wait time
- Monitor log write performance using perfmon; if disk queue length is high, log I/O is the bottleneck

---

## 24. MEMORY MANAGEMENT

### How does SQL Server allocate and use memory?

SQL Server uses memory for multiple purposes; understanding allocation helps identify memory pressures.

- Buffer pool is the primary memory consumer; it caches data pages to reduce disk I/O
- Plan cache stores compiled query plans; large numbers of unique queries can consume significant plan cache
- Sorts and hashes use memory grants; large operations request memory and spill to tempdb if memory is insufficient
- Lock structures use memory; very high lock count from heavy transactional workloads consumes lock memory
- Log cache stores log records before they are written to disk; high transaction rates increase log cache usage
- Connection memory stores connection-specific data like stored procedure parameters and local variables

### How do I detect memory pressure and respond to it?

Memory pressure indicates SQL Server does not have enough memory for optimal operation.

- Query sys.dm_os_memory_clerks to find which memory types consume the most memory
- Monitor available_physical_memory_mb using sys.dm_os_sys_memory; high pressure occurs when this value is low
- Look for excessive disk I/O on tempdb; this indicates sort and hash operations are spilling to disk instead of using memory
- Monitor query execution time; if query performance degrades under memory pressure, add memory to the server
- Reduce plan cache size if many ad-hoc queries are clogging the cache; implement plan guides or parameterized queries
- Lower MAXDOP setting to reduce parallel query memory consumption; smaller parallel degree uses less memory

---

## 25. PERFORMANCE MONITORING

### What metrics should I monitor continuously to track SQL Server health?

Ongoing monitoring reveals trends and enables early detection of problems before they impact users.

- Monitor CPU utilization; sustained high CPU (above 80 percent) indicates CPU is the bottleneck
- Monitor memory available; low available memory (below 20 percent) indicates memory pressure
- Monitor disk I/O rate (IOPS) and latency; high I/O and high latency indicate disk is the bottleneck
- Monitor query response time; increasing trend indicates performance degradation
- Monitor blocking sessions; any session blocking others for more than a minute requires investigation
- Monitor exception rates; increasing rate of timeout or deadlock exceptions indicates system stress

### How do I set up automated alerts for performance issues?

Automated alerts enable proactive response to problems before they become critical.

- Configure SQL Server Agent alerts for error 1205 (deadlock); notify when deadlocks occur
- Create a job to capture blocking chains when any session blocks others for more than 5 minutes
- Set alerts for wait times; alert when PAGEIOLATCH average wait exceeds 10ms
- Create alerts for large numbers of queries in the query store; alerts when query volume exceeds 1000 queries per minute
- Set up perfmon alerts for CPU above 80 percent or memory below 20 percent
- Create email notifications for all alerts; include relevant data for the on-call DBA to investigate

---

## 26. QUERY PLAN CACHE

### What is the query plan cache and how does SQL Server manage it?

The query plan cache stores compiled execution plans to avoid recompilation overhead on subsequent executions.

- When you execute a query for the first time, SQL Server compiles it into an execution plan; this compilation takes CPU time
- The compiled plan is stored in the plan cache; subsequent executions use the cached plan instead of recompiling
- The plan cache is memory-resident and is cleared when SQL Server restarts; compilation starts from scratch on restart
- SQL Server automatically ages out infrequently used plans when memory pressure occurs; this frees memory for other uses
- Stored procedures are cached as single plans; each stored procedure is compiled once and the plan is reused for all executions
- Ad-hoc queries are cached but consume more memory because each unique query text creates a separate plan cache entry

### Why does clearing the plan cache sometimes improve performance?

Clearing the cache forces recompilation of queries with fresh statistics, eliminating stale plans.

- Inaccurate statistics in cached plans cause suboptimal query execution; recompilation uses current statistics
- Plan choice based on obsolete parameter values (parameter sniffing) can be eliminated by clearing the cache and recompiling
- Clearing the cache temporarily reduces performance because queries must recompile; overall improvement comes later with better plans
- Use DBCC FREEPROCCACHE cautiously; clearing the entire cache affects all queries and temporarily impacts performance
- Clear specific plans using DBCC FREEPROCCACHE (plan_handle) if you suspect only specific queries have stale plans
- Prevent excessive clearing; each clearing session causes recompilation overhead for all subsequent executions

### What is adhoc query waste and how do I reduce it?

Adhoc query waste occurs when unique query texts create separate cache entries even for logically similar queries.

- Each unique query text creates a new plan cache entry; minor variations like different parameter values or extra spaces create different entries
- Adhoc queries that are never repeated waste memory; they consume cache space but are never reused
- Enable forced parameterization using ALTER DATABASE SET PARAMETERIZATION FORCED to force parameter reuse
- Use prepared statements or stored procedures instead of ad-hoc queries; these reuse cached plans
- Monitor plan cache size; if cache size is stable or growing despite stable query volume, adhoc waste may be the problem
- Query sys.dm_exec_cached_plans to find plans with execution_count = 1; these are never-reused adhoc plans

---

## 27. QUERY TIMEOUT

### How do I set and troubleshoot query timeout issues?

Query timeouts prevent indefinite query hangs but must be balanced against legitimate long-running queries.

- Query timeout is usually set in application connection strings; common values are 30 or 60 seconds
- COMMAND_TIMEOUT in SQL Server Management Studio defaults to 30 seconds; increase for long-running queries
- Query timeout at SQL Server level is set differently than application timeout; application timeout is the client-side limit
- When a query reaches the timeout, the client cancels the request; SQL Server aborts the query and rolls back changes
- If queries timeout after recent optimization, check whether the optimization actually worked; use Query Store to verify performance
- Increase timeout value for legitimate long-running queries like nightly reports; do not set the same timeout for all query types

### What do I do when legitimate queries are timing out?

Long-running queries need either optimization or adjusted timeout settings.

- Optimize the query first; faster execution is better than longer timeouts
- If the query cannot be optimized further, increase the timeout value to allow the query to complete
- Use connection pooling with dedicated connections for long-running queries; this prevents connection timeout from affecting other queries
- Monitor query completion time; if it is very close to the timeout, it may fail due to load variation
- Schedule long-running queries during off-peak hours when system load is low; this improves reliability
- Break long-running queries into multiple smaller queries if the result set can be produced incrementally

---

## 28. SNAPSHOT ISOLATION

### What is SNAPSHOT isolation and when should I use it?

SNAPSHOT isolation provides point-in-time consistent reads without blocking other writers.

- SNAPSHOT queries see data as it existed at the start of the transaction; modifications made by other transactions are not visible
- SNAPSHOT isolation eliminates dirty reads, non-repeatable reads, and phantom reads without locking
- SNAPSHOT isolation uses row versioning stored in tempdb; every row version is retained in tempdb until all transactions using that version complete
- Use SNAPSHOT isolation for long-running read transactions that do not need the latest data; this prevents blocking of short write transactions
- SNAPSHOT isolation can cause update conflicts; if two transactions update the same row, the second update fails with error 3960
- Monitor tempdb usage when using SNAPSHOT isolation; long-running transactions with SNAPSHOT can consume significant tempdb space

### How do I implement SNAPSHOT isolation in my database?

Enabling and using SNAPSHOT isolation requires database setting and query configuration.

- Enable SNAPSHOT isolation by running ALTER DATABASE [DatabaseName] SET ALLOW_SNAPSHOT_ISOLATION ON
- Enable read_committed_snapshot for optimistic concurrency control on READ COMMITTED queries; this enables all queries to use row versioning
- Change query isolation level using SET TRANSACTION ISOLATION LEVEL SNAPSHOT before starting the transaction
- Use explicit transactions to control the scope of SNAPSHOT isolation; SNAPSHOT applies to the entire transaction
- Test SNAPSHOT isolation in development; ensure update conflicts do not cause application failures
- Monitor tempdb growth; heavy SNAPSHOT usage can cause tempdb to grow rapidly

---

## 29. COMPRESSION

### What compression types does SQL Server support and when should I use them?

SQL Server supports row and page compression to reduce storage space and improve I/O performance.

- Row compression stores fixed-length columns as variable-length; this saves space for NULL or sparse values
- Page compression applies row compression then looks for repeating patterns on the page; it achieves higher compression than row compression
- Compression reduces storage space and I/O bandwidth; smaller data pages means fewer pages to read from disk
- Compression consumes CPU time for compression and decompression; high CPU utilization from compression may offset I/O benefits
- Enable compression on large tables with high I/O usage; compression benefits are most noticeable on I/O bound workloads
- Test compression on a copy of your data first; some workloads see performance improve while others degrade

### How do I enable compression and measure its effectiveness?

Implementing compression requires careful measurement to confirm benefits.

- Enable row compression using ALTER TABLE [TableName] REBUILD WITH (DATA_COMPRESSION = ROW)
- Enable page compression using ALTER TABLE [TableName] REBUILD WITH (DATA_COMPRESSION = PAGE)
- Measure space savings by comparing table size before and after compression using sp_spaceused
- Measure performance impact by comparing query execution time before and after compression
- Enable compression on indexes using ALTER INDEX [IndexName] REBUILD WITH (DATA_COMPRESSION = PAGE)
- Monitor CPU usage after enabling compression; if CPU increases significantly, adjust compression strategy

---

## 30. PARTITIONING

### What is table partitioning and what problems does it solve?

Partitioning divides large tables into smaller segments that can be managed independently.

- Partitions divide a table by a column value; commonly partitioned by date ranges like yearly or monthly
- Partitioning enables parallel execution; queries can scan multiple partitions in parallel
- Partitioning enables partition elimination; queries scanning specific partitions skip unused partitions
- Partitioning enables independent maintenance; one partition can be rebuilt while others remain online
- Partitioning supports sliding window loads; old partitions can be dropped and new partitions added without full table maintenance
- Partitioning has overhead; partition key columns appear in indexes, and query plans must consider partitions

### How do I implement partitioning on an existing table?

Partitioning an existing table involves creating a partition scheme and moving data.

- Create a partition function using CREATE PARTITION FUNCTION to define partition ranges
- Create a partition scheme using CREATE PARTITION SCHEME to assign partitions to filegroups
- Create a new table using the partition scheme; then insert data from the old table
- Drop the old table and rename the new partitioned table to the original name
- Update indexes to use the partition scheme
- Test queries to ensure partition elimination is working; queries should skip unused partitions

---

## 31. REPLICATION AND DISTRIBUTION

### What are SQL Server replication types and what problems does each solve?

SQL Server replication enables copies of the database on other servers, supporting high availability and reporting workloads.

- Snapshot replication copies a full copy of data at defined intervals; this is used for small databases or full refreshes
- Transactional replication copies committed transactions from the publisher to subscribers; this keeps copies nearly synchronized
- Merge replication allows changes on both publisher and subscribers; changes are merged and conflicts are resolved automatically
- Peer-to-peer replication allows changes on any server in a peer network; changes propagate to all servers
- Use replication for reporting; the subscriber copy can be queried without impacting the publisher workload
- Use replication for high availability; if the primary server fails, users can connect to the subscriber

### How do I monitor replication health and latency?

Replication health monitoring ensures data is synchronized across servers.

- Query distribution database to see replication status; check for failed transactions
- Use replication monitor in SQL Server Management Studio to view current replication latency
- Monitor for replication agent failures; replication agents stop if errors occur
- Check for skipped transactions; these indicate data inconsistency between publisher and subscriber
- Verify data consistency by comparing row counts between publisher and subscriber
- Monitor replication queue depth; high queue depth indicates the subscriber is lagging behind the publisher

---

## 32. MIRRORING AND ALWAYSON

### What is database mirroring and how does it provide high availability?

Database mirroring maintains a synchronized copy of the database on a mirror server.

- The principal server is the primary database; the mirror server maintains an exact copy in real time
- Every transaction on the principal is immediately copied to the mirror; transaction commit waits until the mirror acknowledges receipt
- If the principal fails, you can failover to the mirror with no data loss
- Mirroring can run synchronously (waiting for mirror confirmation) or asynchronously (not waiting)
- Synchronous mirroring guarantees no data loss but introduces transaction latency; asynchronous mirroring is faster but may lose recent transactions
- Use mirroring for mission-critical databases where data loss is not acceptable and brief downtime is acceptable

### What is AlwaysOn and how does it differ from mirroring?

AlwaysOn is a more modern high availability feature supporting multiple replicas and more flexible failover.

- AlwaysOn can maintain up to 3 synchronous replicas and 2 asynchronous replicas; mirroring supports only one mirror
- AlwaysOn supports readable secondaries; read-only queries can run on replica copies without impacting the primary
- AlwaysOn supports read-only routing; read queries are automatically routed to replica copies instead of the primary
- AlwaysOn uses Availability Groups to manage multiple databases as a unit; related databases failover together
- AlwaysOn listener provides a virtual network name; applications connect to the listener instead of a specific server name
- Failover is automatic with AlwaysOn; if the primary fails, a replica is promoted with no manual intervention required

---

## 33. DATABASE RESTORE SCENARIOS

### How do I restore a database to the same server with a different name?

Restoring with a different name creates a copy of the database instead of replacing the original.

- Use RESTORE DATABASE [NewDatabaseName] FROM DISK = 'BackupFilePath' to restore with a new name
- Specify MOVE clauses to move data and log files to new locations; MOVE prevents disk space conflicts with the original database
- RESTORE DATABASE [NewDatabaseName] FROM DISK = 'BackupFilePath' WITH MOVE 'LogicalName' TO 'NewPhysicalPath'
- After restore completes, the new database is independent of the original; changes to one do not affect the other
- Restore permissions; the restored database uses the backup permissions; update logins and permissions as needed
- Test the restore to ensure the backup is valid before relying on it for recovery

### How do I perform a test restore to verify backup validity?

Test restores confirm that backups are valid and can be recovered when needed.

- Run a test restore to a development environment or temporary location
- Create a schedule for weekly test restores; this catches backup corruption early
- Restore to a different server if possible; this tests the full recovery path including different hardware
- Verify data integrity after restore by checking row counts and sampling data
- Test restore with recovery to verify databases can be brought online after restore
- Run DBCC CHECKDB on the restored database to ensure integrity

---

## 34. CAPACITY PLANNING

### How do I forecast storage needs and plan for database growth?

Storage planning prevents running out of space and ensures adequate performance.

- Monitor current database size and growth rate; trending over weeks shows patterns
- Project future size based on current growth; if database grows 10 percent per month, calculate future size at 6 and 12 months
- Account for database backups; backup space is often larger than the database size
- Include transaction log space; high transaction rates require more log space
- Plan for index space; indexes typically consume 20 to 50 percent additional space depending on selectivity
- Add headroom; do not plan exactly to capacity; add 50 percent extra space for unexpected growth

### How do I assess whether current hardware can support future workload?

Hardware assessment identifies whether upgrades are needed.

- Identify peak transaction rate; measure transactions per second during peak hours
- Monitor CPU, disk I/O, and memory utilization at peak load; identify the limiting resource
- Estimate peak load increase; if user count will double, transaction rate will approximately double
- Calculate future resource requirements; if CPU is currently 60 percent of capacity and load doubles, CPU will exceed capacity
- Identify upgrade options; increase CPU, disk I/O, or memory depending on which resource is limiting
- Test upgrades in a lab or production replica before implementing to confirm improvement

---

## 35. SECURITY BEST PRACTICES

### How do I implement column-level encryption for sensitive data?

Transparent Data Encryption protects data at the column level without changing application code.

- Transparent Data Encryption (TDE) encrypts all data in the database file; data is encrypted at rest and decrypted when accessed
- Enable TDE using CREATE DATABASE ENCRYPTION KEY and ALTER DATABASE SET ENCRYPTION ON
- TDE requires a database master key and certificate for key management
- Application queries do not require changes; TDE is transparent to applications
- TDE has a performance cost; CPU usage increases due to encryption and decryption
- Back up certificates and keys used for TDE; losing these makes the encrypted database unrecoverable

### How do I audit database access and modifications?

Auditing records who accessed the database and what they changed.

- Use SQL Server Audit feature to capture login attempts, SELECT queries, and modifications
- CREATE SERVER AUDIT to define the audit scope and output destination
- CREATE DATABASE AUDIT SPECIFICATION to define which events to audit
- Query sys.dm_audit_actions to see available audit events
- Archive audit logs regularly; audit logs consume disk space and impact performance
- Test audit configuration before enabling in production; auditing has a performance cost

---

## 36. INSTANCE CONFIGURATION

### How do I configure SQL Server instance settings for optimal performance?

Instance-level configuration affects all databases on the server.

- Set Max Degree of Parallelism (MAXDOP) to the number of CPU cores; parallelism improves performance for large queries
- Set Cost Threshold for Parallelism to 25; queries with cost above 25 use parallelism
- Enable optimize for ad hoc workloads if many ad-hoc queries create plan cache bloat
- Set server memory based on system memory and other applications; dedicate sufficient memory to SQL Server
- Enable instant file initialization for faster data file growth; this skips zeroing new space in data files
- Disable expensive features you do not use; features like full-text search consume resources when enabled

---

## 37. LOG BACKUP STRATEGY

### How frequently should I backup the transaction log?

Transaction log backup frequency determines how much data loss is acceptable after a failure.

- Backup the log every 15 minutes for mission-critical systems; this limits data loss to 15 minutes
- Backup the log every hour for normal systems; data loss is at most 1 hour
- Backup the log every day or every few days for development databases; data loss is acceptable
- Log backups truncate the log and free space for reuse; without frequent log backups, the log fills the disk
- Very frequent log backups increase total backup time and storage; balance frequency against acceptable data loss
- Use log backup retention to automatically delete old backups; old backups can be safely deleted after being copied to long-term storage

---

## 38. BACKUP RETENTION

### How long should I retain database backups?

Retention duration depends on recovery requirements and storage costs.

- Retain full backups for at least 2 weeks; this provides recovery point for recent problems
- Retain full backups for 30 days for production databases; this supports longer-term recovery
- Retain full backups for 90 days or one year for compliance; some regulations require longer retention
- Retain transaction log backups for the same duration as full backups; these enable point-in-time recovery
- Delete old backups to save storage costs; calculate the cost of storage versus the risk of needing an old backup
- Test restores from retained backups periodically; this confirms backups are still valid and recovery procedures work

---

## 39. MONITORING STORED PROCEDURES

### How do I identify and optimize slow stored procedures?

Slow procedures often have the same root causes as slow ad-hoc queries.

- Run the procedure and capture execution time using SET STATISTICS TIME
- Capture the execution plan for the procedure; procedures often use cached plans that are suboptimal
- Check for parameter sniffing; procedures with variable performance at different parameters may have sniffing issues
- Update statistics on tables referenced in the procedure
- Review the procedure logic; unnecessary loops or multiple queries may indicate the procedure can be simplified
- Use Query Store to track procedure performance over time; compare baseline to recent performance

### How do I handle recompilation of stored procedures?

Excessive procedure recompilation can increase CPU and reduce concurrency.

- Identify procedures with high compilation rates using sys.dm_exec_procedure_stats
- Add RECOMPILE hint only if recompilation is truly beneficial; recompile causes every execution to recompile
- Use OPTIMIZE FOR hint to force compilation with specific parameter values
- Avoid changing procedure code frequently; each change invalidates the cached plan
- Use parameterized queries within procedures to avoid parameter sniffing issues
- Monitor procedure performance after addressing recompilation issues

---

## 40. WORKLOAD CLASSIFICATION

### How do I classify different workloads and optimize for each?

Different workloads have different performance requirements and optimization strategies.

- OLTP workloads emphasize transaction throughput and response time; small queries with high concurrency
- OLAP workloads emphasize full data scans and aggregations; large queries with low concurrency
- Reporting workloads read data without modification; optimize for query speed on read-only copies
- Mixed workloads have both OLTP and OLAP characteristics; balance competing requirements
- Identify your primary workload type; optimization strategies differ significantly between OLTP and OLAP
- Use resource governor to allocate resources to workloads; critical workloads get higher priority

---

## 41. PLAN FORCING

### What is plan forcing and when should I use it?

Plan forcing makes SQL Server use a specific execution plan instead of choosing a different plan.

- Query Store enables plan forcing; use sp_query_store_force_plan to force a good plan
- Use plan forcing when Query Store identifies multiple plans for the same query and one plan is significantly better
- Use plan forcing as a temporary fix; the underlying cause (parameter sniffing, statistics) should still be fixed
- Plan forcing prevents plan changes; if data patterns change, a forced plan may become suboptimal
- Review forced plans periodically; rebase the forced plan if a better plan is now available
- Use plan hints sparingly; hints prevent the optimizer from choosing better plans as the system evolves

---

## 42. DYNAMIC MANAGEMENT VIEWS

### What are the most useful DMVs for performance troubleshooting?

DMVs provide real-time visibility into SQL Server performance.

- sys.dm_exec_requests shows currently executing queries and their resource consumption
- sys.dm_exec_sessions shows all active sessions and their login name and database
- sys.dm_db_index_usage_stats shows index usage; identify unused indexes and heavily used indexes
- sys.dm_os_performance_counters shows system-wide performance metrics including I/O and CPU
- sys.dm_exec_query_stats shows query execution statistics for recently executed queries
- sys.dm_os_waiting_tasks shows sessions waiting on resources; identify blocking and contention

### How do I write complex DMV queries to extract meaningful insights?

DMV queries can be combined to answer complex performance questions.

- Join sys.dm_exec_requests to sys.dm_exec_sessions to see who is running what query
- Use sys.dm_exec_query_plan to see the execution plan associated with a specific query
- Combine sys.dm_db_index_usage_stats with sys.objects to find unused indexes by name
- Correlate sys.dm_exec_query_stats with sys.dm_db_wait_stats to see wait types for specific queries
- Use sys.dm_tran_locks with sys.dm_exec_requests to identify what locks are held by running queries
- Test DMV queries in a development environment first; some queries have high overhead on production systems

---

## 43. QUERY HINTS AND FORCING

### What query hints are available and when should I use them?

Query hints override the optimizer's decisions; use them only when necessary.

- NOLOCK hint allows reading uncommitted data; eliminates read blocking for dirty reads
- INDEX hint forces the optimizer to use a specific index; useful when the optimizer chooses wrong index
- RECOMPILE hint forces query recompilation; useful for parameter sniffing but increases CPU
- MAXDOP hint limits parallelism; useful for preventing expensive parallel queries from consuming all CPU
- LOOP JOIN, HASH JOIN, MERGE JOIN hints force specific join algorithms
- Use hints sparingly; let the optimizer choose and only force plans when necessary through plan forcing instead of hints

---

## 44. MISSING STATISTICS ANALYSIS

### How do I identify missing column statistics?

Some columns lack statistics; this prevents accurate row count estimates.

- Query sys.stats to see all statistics objects; compare to column count
- Look for columns without statistics; these may be missing statistics
- Run DBCC SHOW_STATISTICS to view histogram and distribution data for statistics
- Check if auto-generated statistics exist; these are created automatically by SQL Server
- Create statistics for important columns that lack them; use CREATE STATISTICS
- Update statistics regularly; automated statistics update ensures they stay current

---

## 45. FILTERED STATISTICS

### How do I use filtered statistics to improve query optimization?

Filtered statistics provide information about subsets of data, improving estimates for filtered queries.

- Create filtered statistics on commonly used predicates; for example, WHERE Status = 'Active'
- Use CREATE STATISTICS with WHERE clause to define the filtered condition
- Filtered statistics are more selective than full-table statistics; row count estimates are more accurate
- Filtered statistics take up less space than separate full-table statistics
- Use filtered statistics for large tables with skewed data distribution; this improves accuracy for common queries
- Update filtered statistics on the same schedule as regular statistics

---

## 46. EXTENDED EVENTS FOR PERFORMANCE

### How do I use Extended Events to capture performance data?

Extended Events is a lightweight replacement for Profiler; it has lower performance impact.

- CREATE EVENT SESSION to start capturing events
- ADD EVENT to specify which events to capture
- Filter events using ADD FILTER to reduce captured data volume
- Capture slow queries using event_file output to store data in files
- Query captured events using Extended Events UI in SQL Server Management Studio
- Export event data to CSV or other formats for analysis

---

## 47. RESOURCE GOVERNOR

### How do I use Resource Governor to manage workload resources?

Resource Governor allocates CPU, memory, and I/O resources to different workloads.

- Create resource pools with limits on CPU percentage and memory
- Create workload groups within resource pools
- Assign login names or host names to workload groups
- Limit resource usage for low-priority workloads; this protects high-priority workloads
- Monitor resource governor effectiveness using sys.dm_resource_governor_workload_groups
- Test resource governor configuration before enabling in production

---

## 48. COST ANALYSIS

### How do I perform cost analysis on database operations?

Understanding operation cost helps identify expensive operations.

- Review execution plan estimated costs; higher cost operations consume more resources
- Compare estimated cost to actual resources consumed; verify cost estimates are accurate
- Identify highest-cost operations in complex queries; focus optimization efforts on high-cost operations
- Use STATISTICS IO and STATISTICS TIME to measure actual cost compared to estimated cost
- Analyze cost for different input parameters; parameter sniffing often shows as different costs for same query with different parameters
- Use cost analysis to prioritize optimization; optimize highest-cost operations first

---

## 49. TROUBLESHOOTING INTERMITTENT ISSUES

### How do I troubleshoot intermittent performance problems?

Intermittent issues are difficult to troubleshoot because problems disappear before investigation.

- Establish baseline performance metrics before problems occur
- Set up continuous monitoring to capture data during problem episodes
- Use Query Store to see historical performance; query Store captures all queries even between incidents
- Collect extended events during problem windows; this provides detailed trace of activity
- Compare performance during good times to problem times; identify what changed
- Set up alerts to capture problem activity automatically instead of waiting for user reports

---

## 50. DISASTER RECOVERY PLANNING

### How do I develop and test a disaster recovery plan?

A disaster recovery plan ensures your business can survive data loss or system failure.

- Document recovery time objective (RTO); how long the database can be unavailable
- Document recovery point objective (RPO); how much data loss is acceptable
- Design backup and replication strategy to meet RTO and RPO requirements
- Schedule regular disaster recovery drills; test actual recovery from backups
- Document all recovery procedures step-by-step; include contact information for key personnel
- Maintain recovery documentation in an off-site location; make it accessible during disaster
- Review and update disaster recovery plan annually or when infrastructure changes

---

END OF FAQ DOCUMENT
