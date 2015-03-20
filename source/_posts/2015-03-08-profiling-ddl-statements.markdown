---
layout: post
title: "Profiling DDL statements"
date: 2015-03-08 22:37:05 +0900
comments: true
categories: 
---

It has been a while since my last post. This is due to writing and publishing a technical paper for the Release Engineering conference (RELENG 2015), and traveling during a one month honeymoon.

Last September/November I visited the CHISEL group at UVIC in Victoria, Canada. With borrowed hardware, I profiled DDL statements in both PostgreSQL 9.3.6 and MySQL 5.5.41. DDL statements are statements which alter the structure of tables and relations in SQL databases. I did this with the intention to find out which DDL statements block other DML queries (SELECT, UPDATE, INSERT, and DELETE) from running, and in what kind of way.

# Method

To do this I created a small profiler tool named [Nemesis](http://github.com/quantumdb/nemesis). Nemesis works as followed:

1. Nemesis connects to either a MySQL or PostgreSQL database, and prepares an initial database containing a "users" table with 50 million records.
2. Nemesis then starts a user-defined amount of worker threads which continuously perform SELECT, UPDATE, INSERT, and DELETE statements on the "users" table to simulate a software application reading from and writing to the database. The start time, end time, and duration for each query is recorded.
3. After some time a migration scenario is executed which transforms the structure of the "users" table in some way. We also record the start time, end time and duration for these operations (typically one DDL statement).
4. We drop the database.
5. Repeat steps 1-4 for every defined migration scenario. But note that some scenarios cannot be performed in both MySQL and PostgreSQL, due to missing features in that database.

# Setup
For this series of experiments I gratefully used a server with a quad-core Intel Core i7-3770, a 1TB hard disk, and 16GB memory on loan from the CHISEL lab at UVIC. On this server I installed both PostgreSQL 9.3.6 and MySQL 5.5.41 on a Ubuntu 14.04 LTS installation. Both databases were slightly tweaked to use more of the available memory than the default settings. For those who are interested, MySQL used the InnoDB engine by default.

The results below have been taken from a series of experiments run both on MySQL as well as on PostgreSQL. In both cases, all scenarios were run with 6 worker threads in total: 

* 2 workers for SELECT statements.
* 2 workers for UPDATE statements.
* 1 worker for INSERT statements.
* 1 worker for DELETE statements.

# Results
The results are interesting as they differ significantly between MySQL and PostgreSQL. Below I've posted several migration scenarios one might perform on an existing table. For each scenario you'll find a image, which graphs the properties of the queries Nemesis performed on the database. The x-axis resembles time - each mark on this axis equals a second. The y-axis resembles the duration of a query - 1 pixel equals 1 millisecond (this is only to give a general feel for query performance). Each color represents a different type of query being performed.

* **Green**: INSERT statements
* **Red**: DELETE statements
* **Blue**: UPDATE statements
* **Yellow**: SELECT statements
* **Black**: DDL statements

The grey area resembles the period of time during which the DDL statement is being executed.


## Adding a nullable column
In this scenario we add a new column to the existing "users" table which may contain NULL values. 

### PostgreSQL

PostgreSQL seems to be doing this very well. We can see only a very small grey area, which means that this operation completed quickly. There seems to been no obvious impact on the performance of the other queries. This is good news if you need to perform this operation on a live production database.

![](/images/postgresql-add-nullable-column.png)

### MySQL

MySQL shows very different behavior for this scenario. Unlike PostgreSQL, MySQL takes considerably longer to perform this operation (in fact I terminated this operation after 60 seconds), as can be seen by the very wide grey area. Interestingly enough though, MySQL seems to allow SELECT statements to continue to work (since we can see the yellow lines in the grey area) but blocks all other types of DML queries. In essence, during this operation the table seems to be put in a sort of read-only state.

![](/images/mysql-add-nullable-column.png)

## Adding a non-nullable column
In this scenario we perform almost the exact same operation as in the previous scenario with the minor exception that the new column may not store any NULL values.

### PostgreSQL

PostgreSQL stumbles quite dramatically here. As opposed to the previous scenario where the DDL operation completed quickly, in this scenario it takes a lot longer as indicated by the grey area. Even worse is that during this period, none of the other queries are able to continue. It seems that this operation is not suitable to be executed on a live production database.

![](/images/postgresql-add-non-nullable-column.png)

### MySQL
Here we see that MySQL performs similar to PostgreSQL. This operation was also halted after 60 seconds, but throughout this period SELECT statements were still being executed while the other DML queries were blocked as indicated by the yellow lines inside of the grey area. It seems that during this period the table is again in some sort of read-only mode.

![](/images/mysql-add-non-nullable-column.png)

## Renaming a nullable column
This scenario is all about renaming an existing column in the "users" table. In this particular scenario the column is nullable - allowed to store NULL values.

### PostgreSQL

This is another operation which PostgreSQL seems to be quite comfortable with. There is only a very narrow grey area visible, which means that the DDL statement completes quickly. From the other plotted queries we can derive that there is no obvious performance impact once this operation has completed. It seems that this operation is therefore also suitable to be performed on a live production database.

![](/images/postgresql-rename-nullable-column.png)

### MySQL

Compared to PostgreSQL, MySQL seems to take considerably longer to perform this operation as indicated by the very wide grey area. In fact I terminated this operation after 60 seconds. Similar to previous scenarios, MySQL seems to operate in some sort of read-only state while this operation is performed.

![](/images/mysql-rename-nullable-column.png)

## Renaming a non-nullable column
Much like the previous scenario, this scenario renames an existing column. Only in this scenario the column is non-nullable - does not allow NULL values to be stored.

### PostgreSQL

Unlike the scenario where adding a new column had different impact depending on whether the column was nullable or non-nullable, in this scenario PostgreSQL seems to perform similar to the rename nullable column scenario. There are some very small disruptions in the performance of INSERT statements after completing the DDL statement, but this seems acceptable. It seems that this operation could also be performed on a live production database.

![](/images/postgresql-rename-non-nullable-column.png)

### MySQL

Again, MySQL seems to perform worse than PostgreSQL. Where PostgreSQL completes this operation quickly, MySQL took a lot longer. This operation was also terminated after 60 seconds. In addition we seen the same read-only mode we've seen before with other scenarios for MySQL.

![](/images/mysql-rename-non-nullable-column.png)

## Dropping a nullable column
In this scenario we're dropping an existing nullable column.

### PostgreSQL

PostgreSQL doesn't seem to have any problems dropping an existing nullable column from a table. There's a very thin grey area, meaning that the DDL operation completes quickly. There also doesn't seem to be any significant disturbances in the performance of the other queries. 

![](/images/postgresql-drop-nullable-column.png)

### MySQL

Again, MySQL seems to perform worse than PostgreSQL. I terminated the operation after 60 seconds, but again throughout this period, the table seems to be in some sort of read-only mode.

![](/images/mysql-drop-nullable-column.png)

## Dropping a non-nullable column
Similar to the previous scenario, this scenario involves dropping a column. In this case a non-nullable column is dropped.

### PostgreSQL

Although PostgreSQL again completes the DDL statement very quickly as can be seen by the short black line (the operation is so quick the grey area doesn't even show in the graph). However as can be seen, several seconds later the performance of the other DML queries is severely impacted. Some queries block for several seconds, which would result in severely degraded performance for a real application. Whether this is actually due to the DDL statement is hard to tell, but it might be due to the limiting factor of his server: IOPS on the slow hard disk. Perhaps PostgreSQL's VACUUM process is running there, or we've run into some kind of background task which decimates the server's performance.

![](/images/postgresql-drop-non-nullable-column.png)

### MySQL

Again we see the same performance we've seen from MySQL in previous scenarios. The operation was again terminated after 60 seconds, while the table remained in a read-only mode during this period.

![](/images/mysql-drop-non-nullable-column.png)

## Creating an index
In this scenario we create a new index on a column in the existing "users" table.

### PostgreSQL

PostgreSQL claims that it can create indices in a non-blocking fashion by using the "CONCURRENTLY" keyword in the DDL statement. As we can see in the corresponding graph the gray area is very large and extends beyond the end of the graph (in fact I terminated this operation after 60 seconds). However, we can clearly see that the other queries continue to perform without disruption or obvious performance issues. It seems that this can be safely used on live production databases.

![](/images/postgresql-create-index-on-column.png)

### MySQL

Unlike PostgreSQL, MySQL does not have any additional syntax to specify that an index should be created concurrently, or in a non-blocking fashion. As can be seen in the graph, MySQL performs very similar to previous scenarios. The DDL statement takes a while (and I terminated it again after 60 seconds), while during this period the table is put in a read-only mode.

![](/images/mysql-create-index-on-column.png)

## Renaming an existing index
In this scenario we rename an existing index on the "users" table.

### PostgreSQL

PostgreSQL also seems to be able to rename an index without problems during active use. No grey area is visible, and the black line for the DDL statement is very short indicating that this operation completed quickly. Since the graph doesn't reveal any performance issues or disruptions we can assume that this operation is also safe to use on a live production database.

![](/images/postgresql-rename-index.png)

### MySQL

MySQL doesn't seem to have any support for renaming an existing index. However, it seems that support for this [may be added in MySQL 5.7](http://dev.mysql.com/doc/refman/5.7/en/mysql-nutshell.html).

## Dropping an index
In this scenario we drop a previously created index on the "users" table.

### PostgreSQL

Like the "Create index" scenario PostgreSQL also offers a "CONCURRENTLY" keyword in the DDL statement for dropping an index. However as we can see from the graph, the performance of DML statements is severely reduced and disrupted while the index is being dropped. It would seem that this operation is not safe to use on live production databases.

![](/images/postgresql-drop-index-on-column.png)

### MySQL

Surprise! MySQL seems to be able to drop an existing index relatively quickly (roughly 1 second in our case). It's too hard to determine if during this period the table is put in a read-only mode, but since this operation completed so quickly, it may be perfectly safe to use on a live production database.

![](/images/mysql-drop-index-on-column.png)

## Making a column nullable
This scenario concerns making an already existing non-nullable column in the "users" table nullable. This effectively allows NULL values to be stored in this column when this operation has been performed.

### PostgreSQL

Again PostgreSQL seems to have no trouble performing this operation. The grey area is very narrow, meaning that the DDL operation completed quickly. In addition no performance issues or disruptions can be derived from the graph. It seems that this operation is also safe to use on a live production database.

![](/images/postgresql-make-column-nullable.png)

### MySQL

Unfortunately we see the same behavior from MySQL here as we've seen in most scenarios. The operation was terminated after 60 seconds, but throughout this period the table did not block SELECT queries. Other DML queries were blocked however.

![](/images/mysql-make-column-nullable.png)

## Making a column non-nullable
This scenario does the exact opposite of the previous scenario. It makes an already existing nullable column non-nullable. This effectively prohibits NULL values from being stored in this column when the operation has completed.

### PostgreSQL

Here PostgreSQL clearly stumbles again. The DDL operation which makes the column non-nullable takes just over 3 seconds, but as we can see in the graph during this time none of the other queries are able to complete. Depending on your requirements, this operation may not be safe enough to be used on a live production database.

![](/images/postgresql-make-column-non-nullable.png)

### MySQL

Again, MySQL performs worse compared to PostgreSQL. The operation was terminated after 60 seconds again, but during this period the table was again put in a read-only mode.

![](/images/mysql-make-column-non-nullable.png)

## Setting a default value on a nullable column
This scenario is about setting or changing the default value of an already existing nullable column.

### PostgreSQL

As we can see from the graph, PostgreSQL has no problems performing this operation. The very narrow grey area means that the DDL operation completes very quickly. The graph doesn't reveal any performance issues or disruptions for the other DML queries. It appears that this operation is also safe to use on a live production database.

![](/images/postgresql-set-default-expression-on-nullable-column.png)

### MySQL

Here it appears that MySQL performs similar to PostgreSQL for this scenario. We can see a very narrow grey area which means that the DDL operation is completed quickly. It would appear that this operation is safe to use on a live production database.

![](/images/mysql-set-default-expression-on-nullable-column.png)

## Setting a default value on a non-nullable column
This scenario is about setting or changing the default value of an already existing non-nullable column.

### PostgreSQL

This graph is very similar to the graph of the previous scenario for PostgreSQL. The operations seems to take slightly longer since the grey area is slightly wider, but the graph doesn't reveal any performance issues or disruptions for the other DML queries. So it appears that this operation (like the previous) is also safe to use on a live production database.

![](/images/postgresql-set-default-expression-on-non-nullable-column.png)

### MySQL

Compared to the previous scenario the performance is slightly worse. MySQL seems to take slightly longer to complete the DDL statement (still less than a second). It's too hard to determine if the other DML queries are blocked during this period. It would seem that this operation is also safe to use on a live production database.

![](/images/mysql-set-default-expression-on-non-nullable-column.png)

## Modifying the type of a nullable column
This scenario changes the type of an already existing nullable column in the "users" table from a "varchar(255)" type to a "text" type.

### PostgreSQL

PostgreSQL seems to handle this well. Like many other operations, the grey area is very narrow which means that the DDL statement completes quickly. In addition, the graph doesn't seem to show signs of any obvious performance issues or disruptions. It seems that this operation can be safely performed on a live production database.

![](/images/postgresql-modify-data-type-on-nullable-column.png)

### MySQL

Unfortunately MySQL yet again performs a lot worse than PostgreSQL. We can clearly see that this operation takes a long time to complete (and I again had to terminate it after 60 seconds). throughout this period the table was again put in a read-only mode.

![](/images/mysql-modify-data-type-on-nullable-column.png)

## Modifying the type of a non-nullable column
Like the previous scenario, this scenario attempts to modify the type of an already existing column from a "varchar(255)" type to a "text" type. In this instance the column is non-nullable, which means it doesn't allow for NULL values to be stored.

### PostgreSQL

Since the graph is essentially the same compared to the previous scenario, we can draw the same conclusion: This operation can safely be performed on a live production database.

![](/images/postgresql-modify-data-type-on-non-nullable-column.png)

### MySQL

This scenario could not be performed, [as it seems](http://stackoverflow.com/questions/3466872/why-cant-a-text-column-have-a-default-value-in-mysql) that the used version of MySQL doesn't support having default values for "TEXT" type columns, which was needed to perform this scenario.

## Modifying a column's type from integer to text
In this scenario we modify the type of an already existing non-nullable column from a "bigint" type to a "text" type. Assuming there's some more elaborate kind of conversion required than in the previous two scenarios, we should expect some sort of disruption or performance degradation for the DML queries.

### PostgreSQL

The graph reveals that PostgreSQL is indeed having trouble with this scenario. The DDL operation does not complete very quickly (in fact I terminated it after 60 seconds). throughout this period we can see that all other DML queries have been blocked from executing, resulting in an unusable database while this operation is in progress. This operation is obviously not safe to be executed on a live production database. In addition, it seems that the behavior of the DDL operation may be blocking or non blocking depending on from, and to which type the column is being transformed.

![](/images/postgresql-modify-data-type-from-int-to-text.png)

### MySQL

This scenario could not be performed, [as it seems](http://stackoverflow.com/questions/3466872/why-cant-a-text-column-have-a-default-value-in-mysql) that the used version of MySQL doesn't support having default values for "TEXT" type columns, which was needed to perform this scenario.

## Adding a nullable foreign key
This scenario adds a new foreign key to an already existing nullable column in the "users" table. This foreign key references records in a new "addresses" table which stores fictional addresses for our users. 

### PostgreSQL

This operation seems to take PostgreSQL just under 4 seconds to complete. During this period the other DML queries are blocked from executing. However after this brief period the disruption disappears, and a similar performance returns. Depending on the requirements of the application, this 4 second period may be acceptable, or it may not.

![](/images/postgresql-add-nullable-foreign-key.png)

### MySQL

Again MySQL takes a while to complete this operation. It was again terminated after 60 seconds, but throughout this period allowed SELECT queries to be processed, while blocking other DML queries.

![](/images/mysql-add-nullable-foreign-key.png)

## Adding a non-nullable foreign key
Finally we take a look at a scenario which closely resembles the previous scenario. The only difference in this scenario is that the foreign key constraint is applied to a non-nullable column. Since the column is pre-filled with references to the "addresses" table we suspect that the validation phase of creating this constraint will take longer, as it needs to verify that all the current values stored in this column really do reference a value as is defined in the referenced table.

### PostgreSQL

If we look more closely at the graph for PostgreSQL we can see that it indeed takes longer to perform this operation (just over 10 seconds). Similar to the previous scenario, during this period the other DML queries are clearly blocked, but performance of these queries is restored when the operation completes. This operation obviously blocks, and applying it on a live production database may be too costly.

![](/images/postgresql-add-non-nullable-foreign-key.png)

### MySQL

We see similar performance here compared to the previous scenario. The operation was again terminated after 60 seconds, and appears to put the table in a read-only mode for the duration of the operation.

![](/images/mysql-add-non-nullable-foreign-key.png)

# Conclusion
We've looked at various scenarios involving several "refactorings" one can apply to tables in a SQL databases. We've seen that some operations appear to be safe enough to be used on live production databases. There are however some caveats to these experiments:

* Tuning the configuration of a database may yield better performance which may lower the duration of some of these operations. For these experiments I only assigned more memory to be used by PostgreSQL and MySQL. I didn't tune any other aspect of the databases which may have yielded better performance.
* The duration of these operations also seems to take significantly longer when the database is starved of resources. Having a slow disk, or a almost fully utilizing the CPU seems to negatively affect the performance of these DDL statements.
* These operations may show different blocking or non-blocking behavior in different circumstances. Applying these operations to table with a different structure, constraints, or references may yield different results.

In other words: our measurements apply only to this particular setup in which we have tested these scenarios. If you wish to find out if a certain operation can be safely performed in your production environment, you would do well to test it on a staging environment which replicates the production environment (and it's workload) as best it can.

Most interesting I found to be the big differences between MySQL and PostgreSQL. MySQL seems to put tables in a read-only mode when performing DDL statements on tables. Unfortunately MySQL takes considerably longer to perform these operations when compared to PostgreSQL. 


