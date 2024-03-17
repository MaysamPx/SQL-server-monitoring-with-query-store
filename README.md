# SQL-server-monitoring-with-query-store
An introduction to the query store

Despite the hype of NoSQL, relational databases are still dominant in their own niche in the industry and have a significant share of the technical market. For example, in financial systems, such as financial ETL platforms and core banking tools, a large part of the business logic is managed with relational databases such as SQL Server, Oracle, MySQL, etc. in a data-centric way (even incorrectly :) ).

One of the main reasons for the popularity of these databases is probably the "guarantee of data consistency" and, in general, providing a transaction-based system that can be relied on at critical operational points (especially financial operations that have a transactional nature).

These systems (or databases) are typically loyal and committed to the fundamental concepts of relational algebra, data integrity, atomic execution, and several other criteria, while (sometimes) using low resources.

In essence, these relational databases remain important for specific applications where factors like reliability and consistency are paramount, even over high write operation performance (OPS) in streaming applications. However, this comes with its own maintenance and tuning complexities. Like any system, they require monitoring to pinpoint incidents and, more importantly, achieve system observability to identify the root cause of these issues.

Before we dive into the details and structure of this tool, it is better to define the "importance of the issue" in more detail.

The following generally make "database performance monitoring" a separate "requirement" in itself, for which we seek an approach.

- Identifying trends in data growth and operations (trendy events)
- Monitoring CPU usage in specific time periods with different workloads
- Monitoring main memory access and usage in different workloads
- Identifying short and transient CPU-burst and IO-burst events (generally without a fixed pattern)
- Monitoring index performance
- Monitoring transaction execution time
- Finding simple and multi-level operational and data bottlenecks
- Monitoring the number of locks occurring on records, pages, and indexes
- Graphical monitoring to identify spikes and extract meaningful patterns
- Monitoring execution plans created for queries
- Extracting the resource access pattern of a specific database
- Identifying deadlocks, keylocks, and locks in general
- Extracting query execution statistics
- Identifying queries and plans with parameterization problems
- etc.

These issues are not only a "problem" in database monitoring, but also in monitoring other systems. In SQL Server database, there are several tools including SQL Profiler, Activity Monitor and Standard Reports which can extract good information and somehow carry the load of a large part of the monitoring process. However, in the new versions of this database (SQL Server 2016 onwards), a new tool called Query Store has been introduced. This tool tries to collect a history of query execution and execution plans and then generate a series of reports, which is why it was initially called "Flight data recorder". The basis of this tool seems to be this relatively old page [+link](https://learn.microsoft.com/en-us/archive/msdn-magazine/2008/january/sql-server-uncover-hidden-data-to-optimize-application-performance) with useful queries.

 The diagram below illustrates the architecture of the Profiling process in Query Store:

 ![image](https://github.com/MaysamPx/SQL-server-monitoring-with-query-store/assets/13215181/207d9dac-de0c-4d61-ae76-f01c686e7670)

> In this design, the "monitoring" process is implicitly and invisibly present wherever a query needs to be executed, meaning that the act of "monitoring" itself was a cross-cutting requirement in the design of this tool.

One of the strengths of SQL Server is its query optimizer and its capabilities, which often, but not necessarily try to produce the best execution plan for queries [+link](https://learn.microsoft.com/en-us/sql/relational-databases/query-processing-architecture-guide?view=sql-server-ver16). In other words, before executing any query in the form of a Function or SP, the database produces an execution plan for it. Query store also listens for the production of these plans (Query plan in the diagram) and after production, keeps it along with the query text (Query text) in main memory under the title "Query plan and text".

As another parallel and separate task, it also writes and buffers the details of query execution under the title "Query runtime statistics" in memory. This information buffered in main memory is stored asynchronously in a relational database and eventually stored on disk.

The Query Store tool is available through new versions of SSMS and can be configured with various settings, including the periods for Profiling, the size of the Log files, the duration of Log retention, etc. The process of setting up and initial configuration is explained straightforwardly [here](https://www.sqlshack.com/sql-server-query-store-overview/) and suggested settings and some important tips are also provided [here](https://learn.microsoft.com/en-us/sql/relational-databases/performance/best-practice-with-the-query-store?view=sql-server-ver16).

Once you have configured this tool, you will have access to the following categories of reports:

<img width="304" alt="image" src="https://github.com/MaysamPx/SQL-server-monitoring-with-query-store/assets/13215181/872fbe28-7d0d-4195-93ea-7f504485da88">

### Regressed Queries
Queries that have recently become slow; in other words, queries whose execution plans have become worse than previously generated plans are categorized here. Usually, plans are cached in main memory, so the Profiler updates the Plan cache in a predetermined or default period and reports it as Regressed plans.

![image](https://github.com/MaysamPx/SQL-server-monitoring-with-query-store/assets/13215181/8b79d9bd-8213-4d58-b408-622f241b286a)

When the system workload increases and there is a noticeable slowdown, we empirically check this section first [at the time of the slowdown and shortly after it is resolved]. Especially when the system is at its peak workload. For example, in financial systems in the capital market [such as the stock order management system (OMS)] that have a limited and defined activity period [with various peaks] and events such as market opening, traders' attempts to headshot stocks, buy and sell queues, etc. that typically happen in batches and competitively and are correlated with other operations, this section has a useful application and helps to identify slow queries, something that was not seen when the query was executed individually and separately, in a low-load situation.

### Overall Resource Consumption
Indicates the consumption of various logical and physical resources in a specified period (by default, the last month). This section aggregates the "Query runtime statistics" data, which is visible in the Profiling architecture in the first image.

This report is shown in the form of a bar chart in the following 4 general criteria:

![image](https://github.com/MaysamPx/SQL-server-monitoring-with-query-store/assets/13215181/d2fa389d-e536-420f-a4a6-db6c9b3c0f9b)

- **Duration:**	The duration of the query execution, in milliseconds.
- **Execution count:**	The number of times the query has been executed.
- **CPU time:**	The amount of CPU time that was used by the query, in milliseconds.
- **Logical reads:**	The number of logical reads that were performed by the query.

For each period (each day), the details shown in the image below are provided for each of the 4 criteria above.

![image](https://github.com/MaysamPx/SQL-server-monitoring-with-query-store/assets/13215181/1ac57a3b-5668-4e52-8dd5-92f5a3725488)

Each of these fields can be useful in an interesting way. For example, by comparing Logical reads and Logical writes, you can get a ratio of the amount of data that is written to and read (Write intensive vs. Read intensive queries), or as another example, you can measure the disk access ratio compared to the number of Logical reads.

Another interesting thing is the amount reported for "Temp DB Memory Used in KB"; empirically, it can be a solution. For example, consider a database that works with a large number of Table-valued functions and SPs, and there are also various Temp table variables defined in each of them. This value provides a view and reports the amount of Temp DB used by the current database in the specified range.

> These reports are generally a good option for periodic reviews. For example, you can set the time window to the size of the intervals between releases and measure the impact of improvements and changes.

### Top Resource Consuming Queries

This report is also one of the most useful reports of Query Store. For all the fields [or metrics] that you saw in the previous report [the same image above], it provides a detailed and comparative report that includes the following:

- Execution plans were created along with a performance chart for each (Plan summary)
- A descending bar chart of queries based on the selected metric (e.g. Duration)
- Details of each execution plan (including the execution cost of each part of the query)

> The interesting point about this report is that it essentially sees the metrics themselves as a resource and identifies queries with the highest amount of that metric as expensive queries.

For example, in the Execution Count metric, which reports the number of times each query is executed, the following can be inferred:

- Comparison of the number of executions of queries and obtaining a proportion between APP requests and database executions [to verify the number of requests on the APP side and DB side].
- Detection of abnormal behaviour in the APP when a corresponding query is executed with a higher deviation than the rest.
- Detection of a high number of executions of a query and performing an optimization process with a specific goal, including appropriate indexing and use of Memory-optimized tables and others.

The CPU time metric can be used to easily identify CPU-bound queries or any query with the highest processing power demand. This metric can also provide insight to the technical and commercial product designer in terms of service design and commercial discussions. For example, in API design, it can be useful to know the amount of resources consumed and to make predictions about SLA discussions. However, it is important to note that this metric is not a perfect measure of efficiency and should not be used as the sole criterion for evaluating the performance of a query.

The Physical reads metric can be used to identify the following:

- The performance of indexes at the disk level (e.g., the cost of updating clustered indexes)
- The efficiency of indexes in the generated plans
- Comparison of generated plans and the ability to force a plan that has performed better
- The performance of pages and partitions
- The different approaches to index and table traversal (including index seek, index and table scan, key lookup, and other related matters)

The Wait time metric can be used to easily identify queries with "low execution count and high latency" or "high execution count and noticeable latency".

> Similar to the previous sections, in this section, you can also prefer one execution plan over the others and add it to the Forced plans list.

### Queries With Forced Plans

### Queries With High Variation

### Query Wait Statistics

## Conclusion
In general, when you have transferred a relatively complex part of business logic to the database, or even handled it on the APP side. Still, ultimately the database is under fire from various requests with a high execution rate. On the other hand, performance and slowdown problems arise without knowing exactly which part is causing them, this tool will come in handy amazingly.

We turned to this tool when other tools such as Activity Monitor, SQL Profiler, etc. either reported fewer details or did not have a good UI for Visualization details. Therefore, finding clues to several performance, index, and other related problems was difficult without Query Store.










 


