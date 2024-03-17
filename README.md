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






 


