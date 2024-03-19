# SQL Server DB monitoring with the Query Store
An introduction to the query store
- [Introduction](#introduction)
- [Regressed Queries](#regressed-queries)
- [Overall Resource Consumption](#overall-resource-consumption)
- [Top Resource Consuming Queries](#top-resource-consuming-queries)
- [Queries With Forced Plans](#queries-with-forced-plans)
- [Queries With High Variation](#queries-with-high-variation)
- [Query Wait Statistics](#query-wait-statistics)
- [Conclusion](#conclusion)

### Introduction
Despite the hype of NoSQL, relational databases are still dominant in their niche in the industry and have a significant share of the store/retrieve engines. For instance, in retail systems, healthcare, and financial systems such as financial ETL platforms and core banking tools, a large part of the business logic is managed with relational databases such as SQL Server, Oracle, MySQL, PostgreSQL, etc.

One of the main reasons for the popularity of these databases is the ACID guarantee and, in general, providing a transaction-based system that can be relied on at critical operational points (especially mission-critical operations that have a transactional nature).

These systems (or databases) are typically committed to the fundamental concepts of relational algebra, data integrity, atomic execution, and several other criteria with strong theoretical/mathematical support.

These relational databases remain important for specific applications where factors like atomicity and consistency are paramount, even over high write operation performance (OPS) in write/read-heavy applications. However, this comes with its own maintenance and tuning complexities. Like any system, they require monitoring to discover incidents and, more importantly, achieve system observability to identify the root cause of these issues.

Before we dive into the details and structure of this tool, it is better to define the problem and use cases in more detail.

The following features generally initiate a requirement in monitoring storage systems, for which we seek an approach.

- Identifying trends in data growth and operations (trendy read/writes)
- Monitoring CPU usage in specific periods with different workloads
- Monitoring main memory access and usage in different workloads
- Identifying short and transient CPU-burst and IO-burst events (generally with a dynamic/probabilistic pattern)
- Monitoring index performance
- Monitoring transaction execution time
- Finding simple and multi-level operational bottlenecks
- Monitoring the number of locks occurring on records, pages, and indices
- Graphical monitoring to identify bursts and extract meaningful patterns
- Monitoring execution plans created for queries
- Extracting the resource access pattern of a specific database
- Identifying deadlocks, keylocks, and locks in general
- Extracting query execution statistics
- Identifying queries and plans with parameterization problems
- etc.

These issues are not only a "problem" in database monitoring, but also in monitoring other software systems. In SQL Server, there are several tools including SQL Profiler, Activity Monitor and Standard Reports which can extract good information and somehow carry the load of a large part of the monitoring process. However, in the recent versions of this database (SQL Server 2016 onwards), a new tool called Query Store has been introduced. This tool tries to collect a history of query execution and execution plans and then generate a series of reports, which is why it was initially called "Flight data recorder". The basis of this tool seems to be this relatively old page [+link](https://learn.microsoft.com/en-us/archive/msdn-magazine/2008/january/sql-server-uncover-hidden-data-to-optimize-application-performance) with useful queries.

 The diagram below illustrates the architecture of the Profiling process in Query Store:

 ![image](https://github.com/MaysamPx/SQL-server-monitoring-with-query-store/assets/13215181/207d9dac-de0c-4d61-ae76-f01c686e7670)

> In this design, the "monitoring" process is implicitly and invisibly present wherever a query needs to be executed, meaning that the process of "monitoring" itself was a cross-cutting requirement in the design of this tool.

One of the strengths of SQL Server is its query optimizer and its capabilities, which often, but not necessarily try to produce the best execution plan for queries [+link](https://learn.microsoft.com/en-us/sql/relational-databases/query-processing-architecture-guide?view=sql-server-ver16). In other words, before executing any query in the form of a Function or SP, the database produces an execution plan for it. Query store also listens for the generation of these plans (Query plan in the diagram) and after production, keeps it along with the query text (Query text) in main memory under the title "Query plan and text".

As another parallel and separate task, it also writes and buffers the details of query execution under the title "Query runtime statistics" in memory. This information buffered in main memory is stored asynchronously in a relational database and eventually stored on disk.

The Query Store tool is available through new versions of SSMS and can be configured with various settings, including the periods for Profiling, the size of the Log files, the duration of Log retention, etc. The process of initial configuration is explained straightforwardly [here](https://www.sqlshack.com/sql-server-query-store-overview/) and suggested settings and some important tips are also provided [here](https://learn.microsoft.com/en-us/sql/relational-databases/performance/best-practice-with-the-query-store?view=sql-server-ver16).

Once you have configured this tool, you will have access to the following categories of reports:

<img width="304" alt="image" src="https://github.com/MaysamPx/SQL-server-monitoring-with-query-store/assets/13215181/872fbe28-7d0d-4195-93ea-7f504485da88">

### Regressed Queries
Queries that have recently become slow; in other words, queries whose execution plans have become worse than previously generated plans are categorized here. Usually, plans are cached in main memory, so the Profiler updates the Plan cache in a predetermined or default period and reports it as Regressed plans.

![image](https://github.com/MaysamPx/SQL-server-monitoring-with-query-store/assets/13215181/8b79d9bd-8213-4d58-b408-622f241b286a)

If the system runs slow under high load then check this section first! Especially during peak times:

- End-user-oriented systems (like web applications in finance, healthcare, education, governance, etc) have busy periods with bursts of activity.
- These bursts can overload the system and hide slow queries during low usage.
- This section helps pinpoint those slow queries during peak loads.

### Overall Resource Consumption
Indicates the consumption of various logical and physical resources in a specified period (by default, the last month). This section aggregates the "Query runtime statistics" data, which is visible in the Profiling architecture.

This report is shown in the form of a bar chart in the following 4 general criteria:

![image](https://github.com/MaysamPx/SQL-server-monitoring-with-query-store/assets/13215181/d2fa389d-e536-420f-a4a6-db6c9b3c0f9b)

- **Duration:**	The duration of the query execution, in milliseconds.
- **Execution count:**	The number of times the query has been executed.
- **CPU time:**	The amount of CPU time that was used by the query, in milliseconds.
- **Logical reads:**	The number of logical reads that were performed by the query.

For each period (each day), the details shown in the image below are provided for each of the 4 criteria above.

![image](https://github.com/MaysamPx/SQL-server-monitoring-with-query-store/assets/13215181/1ac57a3b-5668-4e52-8dd5-92f5a3725488)

Each of these fields can be useful in an interesting way. For instance, by comparing Logical reads and Logical writes, you can get a ratio of the amount of data that is written to and read (Write intensive vs. Read intensive queries), or as another example, you can measure the disk access ratio compared to the number of Logical reads.

Another interesting thing is the amount reported for "Temp DB Memory Used in KB"; empirically, it can be a solution. For instance, consider a database that works with a large number of Table-valued functions and SPs, and there are also various Temp table variables defined in each of them. This value provides a view and reports the amount of Temp DB used by the current database in the specified range.

> These reports are generally a good option for **periodic reviews**. For example, you can set the time window to the size of the intervals between releases and measure the impact of improvements and changes.

### Top Resource Consuming Queries

This report is also one of the most useful reports of Query Store. For all the metrics that you saw in the previous report, it provides a detailed and comparative report that includes the following:

- Execution plans were created along with a performance chart for each (Plan summary)
- A descending bar chart of queries based on the selected metric (e.g. Duration)
- Details of each execution plan (including the execution cost of each part of the query)

> The interesting point about this report is that it essentially sees the metrics themselves as a resource and identifies queries with the highest amount of that metric as expensive queries.

For example, in the Execution Count metric, which reports the number of times each query is executed, the following can be inferred:

- Comparison of the number of executions of queries and obtaining a proportion between APP requests and database executions (to verify the number of requests on the APP side and DB side).
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

This section lists the plans that have been selected as Forced plans for different queries. A Forced plan is a query plan that is forced to be used by the database engine, even if it is not the most efficient plan.

![image](https://github.com/MaysamPx/SQL-server-monitoring-with-query-store/assets/13215181/5dd24c27-a8a5-4d77-a9b7-74990d85525e)

In the image above, I have provided an example of the Forced plans I have previously forced. You can see that for a query, SQL Server has detected 9 execution plans, and based on the average duration metric, I have selected the execution plan with the ID "15809" as the better plan and forced it.

> One of the interesting features that this section offers is the ability to compare different plans with the forced plan. In other words, you can measure whether the forced plan is performing as expected over time.

In addition to these, it also reports several characteristics for each forced plan that can give the database designer and optimizer a precise view, including the following:

- Number of plan force failures:
> You may have forced a plan before making changes, such as deleting or adding an index. Then, you delete the index that affected the plan, so the plan will no longer compile as before, and as a result, forcing it will also fail.

- The last time the selected plan was compiled.
- The last time the query was executed.
- The last time the query was executed with the forced plan.

### Queries With High Variation
One of the interesting sections of Query Store is this type of report. In this section, queries that do not have a stable execution pattern are reported and categorized. For example, a query may perform slowly and sometimes better based on different performance metrics, and this trend continues in a fluctuating manner without any specific pattern, which will be reported in this section with a high probability.

In general, in the SQL Server database, queries that have parameterization problems (replacing literal values with placeholder variables to prevent creating multiple duplicate plans for the same query) can be seen in this section.

![image](https://github.com/MaysamPx/SQL-server-monitoring-with-query-store/assets/13215181/38e705a8-20f8-4cb6-84b3-4d273f2722e5)

As you can see in the above chart, by selecting the CPU Time metric and the Standard Deviation sub-metric, a query that has different behaviour in terms of execution time and processing power in a given time range is well distinguished. For better analysis, during the system peak load, the Profiling time range can be limited to a specific value to measure the performance of queries in different time windows.

The issue of Parametrization in SQL Server is well discussed [here](https://learn.microsoft.com/en-us/sql/relational-databases/performance/specify-query-parameterization-behavior-by-using-plan-guides?view=sql-server-ver16).


### Query Wait Statistics
In this section, you can see queries that are waiting for resources from different aspects, including memory, CPU, network IO, locks, and so on.
![image](https://github.com/MaysamPx/SQL-server-monitoring-with-query-store/assets/13215181/e6f2cec9-84c9-4cd2-8b4a-10396cbc665e)

One of the interesting features of this section is the deadlock report. In my experience, by taking the following report, queries that have the highest amount of Wait in the Lock category are likely to have deadlocks of a particular type (e.g., Keylock, Page lock, or Row lock).

**Query Wait Statistics** ---> **Lock** ---> **Based on "max wait time"**

For instance, in the image below, I have provided a report of a database with relatively complex business logic in the financial field.

![image](https://github.com/MaysamPx/SQL-server-monitoring-with-query-store/assets/13215181/92a6bd93-0be5-4908-ba6d-42b42ab5facf)

In this chart, the three queries that have the same Wait time are exactly the three SPs and Functions that have encountered a keylock issue on a primary clustered index and a non-clustered covered index.

![image](https://github.com/MaysamPx/SQL-server-monitoring-with-query-store/assets/13215181/221a8fa9-3b1a-48f9-afbb-bba2c82ea797)

Of course, it is better to use Extended Events, which are completely tailored to this issue, to find deadlocks. However, the reports in this section can also be useful.

## Conclusion
In general, when you have transferred a relatively complex part of business logic to the database, or even handled it on the APP side. Still, ultimately the database is under fire from various requests with a high execution rate. On the other hand, performance and slowdown problems arise without knowing exactly which part is causing them, this tool will come in handy amazingly.

Query Store helped us to find clues to several query performance, index, and other related problems in a financial system with 40+ M transactions.
