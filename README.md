# SQL-server-monitoring-with-query-store
An introduction to the query store

Despite the hype of NoSQL, relational databases are still dominant in their own niche in the industry and have a significant share of the technical market. For example, in financial systems, such as financial ETL platforms and core banking tools, a large part of the business logic is managed with relational databases such as SQL Server, Oracle, MySQL, etc. in a data-centric way (even incorrectly :) ).

One of the main reasons for the popularity of these databases is probably the "guarantee of data consistency" and, in general, providing a transaction-based system that can be relied on at critical operational points (especially financial operations that have a transactional nature).

These systems (or databases) are typically loyal and committed to the fundamental concepts of relational algebra, data integrity, atomic execution, and several other criteria, while (sometimes) using low resources.

In essence, these relational databases remain important for specific applications where factors like reliability and consistency are paramount, even over high write operation performance (OPS) in streaming applications. However, this comes with its own maintenance and tuning complexities. Like any system, they require monitoring to pinpoint incidents and, more importantly, achieve system observability to identify the root cause of these issues.
