---
published: false
---
## Optimizing Functions 2

In this post, I'm going to tell you about optimizing functions (hence the title). I am going to give you some tips and tricks to keep the performance at the maximum. 

My first point will be to explain some options from the CREATE FUNCTION and my second will be to complete this with  some other approaches to get the best out of your functions (this applies to procedures as well).

https://www.postgresql.org/docs/current/xfunc-optimization.html

From this very short entry within the official documentation, there are some takeaways already. The first thing to know is the function type or volatility category you are using. 

### Volatile is stealing your speed


It might not seem obvious to all the users, but the default option for type is VOLATILE. The reason for this is very practical. It is because it is the most versatile form of function. With it you can do anything in a database, even change the data within the tables.

* You can find this in the CREATE FUNCTION page: *A query using a volatile function will re-evaluate the function at every row where its value is needed.* In other words, this is bad for performance because there will be no caching the results. (ref https://www.postgresql.org/docs/current/xfunc-volatility.html)
* Indexes on VOLATILE functions are prohibited. No way to index the result to get faster query. 
* As a result of being the default behavior, any SQL developing software (such as PgAdmin or others) do not display the type and just assumes the type is VOLATILE.

Tip #1:
*Use explicit type.*

Tip #2:
*Use the strictest type to the behavior you expect from your function. If the type is not permissive enough the database will tell you by throwing an error.*
From the doc: a function containing only SELECT commands can safely be marked STABLE. Same for a function that relies on timestamps with timezone.

Tip #3:
*Index whenever you can using the right kind of index (more about that in another post). Only IMMUTABLE and STABLE functions allow for this.*

Tip #4:
*Take note that tip #3 implies that the index will be replicated on the secondaries. Use this to your advantage. The index can lower the cost of the calculations made by the functions but could also just be used to balance CPU and memory usage to the standby servers (if no index is used).*

What not to do:
#1: 

### Parallel whenever possible

Don't needlessly restrict your functions. Postgres handles parallel query through the planner and balances the resources according to max_worker_processes (which is a hard limit), max_parallel_workers, max_parallel_workers_per_gather (ref + check).

Tip #5: Make use of parallel when it's possible and restrict it when needed.

Tip #6: Make your code readable. You never know when someone will need to update your work.

Tip #7: You can parallel a lot of things nowadays (database, functions, tables(!), you could also physically parallelize = partition or subpartition a table)

### The cost of living

### Pretty maids all in a row/Row your boat

### Level up the cursor

### C > SQL

Let's face it C is a lot harder to code than a SQL function. But C is faster than SQL. If you have explored every option here, and you think it's too slow, you might want to turn to C. 

### Make use of everything you have 

Don't stop at optimizing the function. SQL and PLpgSQL can be optimized to. 

Theses are just a few ways to improve your functions (and your SQL queries in a broader way)
* CTEs
* work_mem and TEMP TABLE
* UNLOGGED TABLES
* JOINS vs NESTED QUERIES vs cartesian product
* NOT IN vs NOT EXISTS 
* IN vs VALUES vs ARRAY
* INDEXES
* EXPLAIN (ANALYZE) is your friend


There is a lot more parameters to better the performance (geqo, jit...)

EXPLAIN is your friend





Enter text in [Markdown](http://daringfireball.net/projects/markdown/). Use the toolbar above, or click the **?** button for formatting help.
