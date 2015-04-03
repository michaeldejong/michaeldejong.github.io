---
layout: post
title: "Revisiting profiling DDL statements - MySQL's return"
date: 2015-04-03 16:02:59 +0200
comments: true
categories: 
---

In the previous blogpost I covered the results of a series of experiments I ran to profile DDL statements for PostgreSQL 9.3.6 and MySQL 5.5.41. But people on Twitter rightfully pointed out that I profiled databases which were around 2 years old, and newer versions have been released since. People claimed that in both PostgreSQL 9.4 and MySQL 5.6 there have been some improvements with regard to the blocking nature of DDL statements. So I figured this warrants a re-run of the experiments I ran earlier.

# Setup
In this post I will be comparing the performance and locking properties of DDL statements of PostgreSQL 9.3.6, PostgreSQL 9.4.1, MySQL 5.5.41, and MySQL 5.6.19. The reason I'm re-running the experiments for the older version of PostgreSQL and MySQL is because I no longer have the same server, and am now running these experiments on a Digital Ocean droplet with 8 cores, 16 GB memory, and a 160 GB SSD drive. The biggest difference compared to the previous setup is the SSD drive, so I'm expecting better performance from both databases.

In addition Nemesis has been modified to execute slightly different DDL statements for MySQL, which allows you to specify a specific algorithm and/or type of lock.

# Results

Like in the previous blogpost I'm running through several scenarios where the schema of a database is being modified while it is in use. For each scenario you'll find a image, which graphs the properties of the queries Nemesis performed on the database. The x-axis resembles time - each mark on this axis equals a second. The y-axis resembles the duration of a query - 1 pixel equals 1 millisecond (this is only to give a general feel for query performance). Each color represents a different type of query being performed.

* **Green**: INSERT statements
* **Red**: DELETE statements
* **Blue**: UPDATE statements
* **Yellow**: SELECT statements
* **Black**: DDL statements

The grey area resembles the period of time during which the DDL statement is being executed.

## Adding a nullable column

### PostgreSQL 9.3
![](/images/ddl-profiling/postgresql-9.3/add-nullable-column.png)

### PostgreSQL 9.4
![](/images/ddl-profiling/postgresql-9.4/add-nullable-column.png)

### MySQL 5.5
![](/images/ddl-profiling/mysql-5.5/add-nullable-column.png)

### MySQL 5.6
![](/images/ddl-profiling/mysql-5.6/add-nullable-column.png)

We can see that PostgreSQL 9.4 and 9.3 both perform this operation in a non-blocking fashion. The DDL statement is quick to complete, and does not seem to show any negative impact on performance afterwards. For MySQL 5.6 we can see that there is a big improvement over MySQL 5.5. Although the operation takes a long time to complete, all DML queries continue to work while the DDL statement is running.

## Adding a non-nullable column

### PostgreSQL 9.3
![](/images/ddl-profiling/postgresql-9.3/add-non-nullable-column.png)

### PostgreSQL 9.4
![](/images/ddl-profiling/postgresql-9.4/add-non-nullable-column.png)

### MySQL 5.5
![](/images/ddl-profiling/mysql-5.5/add-non-nullable-column.png)

### MySQL 5.6
![](/images/ddl-profiling/mysql-5.6/add-non-nullable-column.png)

Here we see that PostgreSQL 9.4 and 9.3 both perform this operation in a blocking fashion. The DDL statement takes a long time to complete and doesn't allow for other DML queries to complete while this operation is in progress. For MySQL 5.6 we can see that there is another big improvement over MySQL 5.5 (and PostgreSQL). Although the DDL operation still takes a long time to complete, all DML queries continue to work while the DDL statement is running.

## Renaming a nullable column

### PostgreSQL 9.3
![](/images/ddl-profiling/postgresql-9.3/rename-nullable-column.png)

### PostgreSQL 9.4
![](/images/ddl-profiling/postgresql-9.4/rename-nullable-column.png)

### MySQL 5.5
![](/images/ddl-profiling/mysql-5.5/rename-nullable-column.png)

### MySQL 5.6
![](/images/ddl-profiling/mysql-5.6/rename-nullable-column.png)

Here we see that PostgreSQL 9.4 and 9.3 both perform this operation in a non-blocking fashion. The DDL statement is quick to complete, and does not seem to show any negative impact on performance afterwards. For MySQL 5.6 we can see that the table is put in a read-only mode while this operation is in progress. MySQL 5.6 performs a lot better and similar to PostgreSQL. The operation is quick to complete with no obvious performance issues afterwards.

## Renaming a non-nullable column

### PostgreSQL 9.3
![](/images/ddl-profiling/postgresql-9.3/rename-non-nullable-column.png)

### PostgreSQL 9.4
![](/images/ddl-profiling/postgresql-9.4/rename-non-nullable-column.png)

### MySQL 5.5
![](/images/ddl-profiling/mysql-5.5/rename-non-nullable-column.png)

### MySQL 5.6
![](/images/ddl-profiling/mysql-5.6/rename-non-nullable-column.png)

Here we see that PostgreSQL 9.4 and 9.3 both perform this operation in a non-blocking fashion. The DDL statement is quick to complete, and does not seem to show any negative impact on performance afterwards. For MySQL 5.6 we can see that the table is again put in a read-only mode while this operation is in progress. MySQL 5.6 performs a lot better and more similar to PostgreSQL. The operation is quick to complete with no obvious performance issues afterwards.

## Dropping a nullable column

### PostgreSQL 9.3
![](/images/ddl-profiling/postgresql-9.3/drop-nullable-column.png)

### PostgreSQL 9.4
![](/images/ddl-profiling/postgresql-9.4/drop-nullable-column.png)

### MySQL 5.5
![](/images/ddl-profiling/mysql-5.5/drop-nullable-column.png)

### MySQL 5.6
![](/images/ddl-profiling/mysql-5.6/drop-nullable-column.png)

Here we see that PostgreSQL 9.4 and 9.3 again both perform this operation in a non-blocking fashion. The DDL statement is quick to complete, and does not seem to show any negative impact on performance afterwards. For MySQL 5.6 we can see that the table is put in a read-only mode while this operation is in progress. MySQL 5.6 performs a lot better. The operation takes a long time to complete, but doesn't stop other DML queries from being executed.

## Dropping a non-nullable column

### PostgreSQL 9.3
![](/images/ddl-profiling/postgresql-9.3/drop-non-nullable-column.png)

### PostgreSQL 9.4
![](/images/ddl-profiling/postgresql-9.4/drop-non-nullable-column.png)

### MySQL 5.5
![](/images/ddl-profiling/mysql-5.5/drop-non-nullable-column.png)

### MySQL 5.6
![](/images/ddl-profiling/mysql-5.6/drop-non-nullable-column.png)

Here we see that PostgreSQL 9.4 and 9.3 again both perform this operation in a non-blocking fashion. The DDL statement is quick to complete, and does not seem to show any negative impact on performance afterwards. For MySQL 5.6 we can see that the table is put in read-only mode while this operation is in progress. MySQL 5.6 performs a lot better compared to MySQL 5.5. The operation takes a long time to complete, but doesn't stop other DML queries from being executed.

## Creating an index

### PostgreSQL 9.3
![](/images/ddl-profiling/postgresql-9.3/create-index-on-column.png)

### PostgreSQL 9.4
![](/images/ddl-profiling/postgresql-9.4/create-index-on-column.png)

### MySQL 5.5
![](/images/ddl-profiling/mysql-5.5/create-index-on-column.png)

### MySQL 5.6
![](/images/ddl-profiling/mysql-5.6/create-index-on-column.png)

Here we see that PostgreSQL 9.4 and 9.3 again both perform this operation in a non-blocking fashion. The DDL statement takes a long time to complete, but doesn't block other DML queries from being executed. For MySQL 5.6 we can see that the table is put in read-only mode while this operation is in progress. MySQL 5.6 performs a lot better. Similar to PostgreSQL, the operation takes a long time to complete, but during this period DML queries can still be executed.

## Renaming an existing index

### PostgreSQL 9.3
![](/images/ddl-profiling/postgresql-9.3/rename-index.png)

### PostgreSQL 9.4
![](/images/ddl-profiling/postgresql-9.4/rename-index.png)

This is strange. In PostgreSQL 9.3 this operation quickly completed without any obvious performance issues. In PostgreSQL 9.4, this operation took around 1 second to complete, and blocked other DML queries from executing. Depending on your requirements, I would label this operation as "Use at own risk" for production environments. MySQL 5.5 doesn't have support for renaming indices. MySQL 5.6 seems to have support, but I couldn't get this to work for the scenario as defined in Nemesis.

## Dropping an index

### PostgreSQL 9.3
![](/images/ddl-profiling/postgresql-9.3/drop-index-on-column.png)

### PostgreSQL 9.4
![](/images/ddl-profiling/postgresql-9.4/drop-index-on-column.png)

### MySQL 5.5
![](/images/ddl-profiling/mysql-5.5/drop-index-on-column.png)

### MySQL 5.6
![](/images/ddl-profiling/mysql-5.6/drop-index-on-column.png)

Here we see that PostgreSQL 9.4 and 9.3 again both perform this operation in a non-blocking fashion. The DDL statement takes less than a second to complete, and doesn't block other DML queries from being executed. For MySQL 5.5 we can clearly see that other DML queries are being blocked from executing for just over 2 seconds. MySQL 5.6 performs a lot better. The operation completes quickly, and doesn't show any signs of performance issues.

## Making a column nullable

### PostgreSQL 9.3
![](/images/ddl-profiling/postgresql-9.3/make-column-nullable.png)

### PostgreSQL 9.4
![](/images/ddl-profiling/postgresql-9.4/make-column-nullable.png)

### MySQL 5.5
![](/images/ddl-profiling/mysql-5.5/make-column-nullable.png)

### MySQL 5.6
![](/images/ddl-profiling/mysql-5.6/make-column-nullable.png)

Here we see that PostgreSQL 9.4 and 9.3 again both perform this operation in a non-blocking fashion. The DDL statement is quick to complete, and doesn't show any performance issues. For MySQL 5.5 and MySQL 5.6 we can clearly see that the table is put in a read-only mode for a prolonged period of time.

## Making a column non-nullable

### PostgreSQL 9.3
![](/images/ddl-profiling/postgresql-9.3/make-column-non-nullable.png)

### PostgreSQL 9.4
![](/images/ddl-profiling/postgresql-9.4/make-column-non-nullable.png)

### MySQL 5.5
![](/images/ddl-profiling/mysql-5.5/make-column-non-nullable.png)

### MySQL 5.6
![](/images/ddl-profiling/mysql-5.6/make-column-non-nullable.png)

Here we see that PostgreSQL 9.3 and 9.4 both perform this operation in a blocking fashion. The DDL statement takes 13 and 25 seconds respectively, and clearly blocks other DML queries from being executed. For MySQL 5.5 and MySQL 5.6 we can clearly see that the table is put in a read-only mode for a prolonged period of time.

## Setting a default value on a nullable column

### PostgreSQL 9.3
![](/images/ddl-profiling/postgresql-9.3/set-default-expression-on-nullable-column.png)

### PostgreSQL 9.4
![](/images/ddl-profiling/postgresql-9.4/set-default-expression-on-nullable-column.png)

### MySQL 5.5
![](/images/ddl-profiling/mysql-5.5/set-default-expression-on-nullable-column.png)

### MySQL 5.6
![](/images/ddl-profiling/mysql-5.6/set-default-expression-on-nullable-column.png)

Hurray! Both versions of both MySQL and PostgreSQL seem to be doing this in a non-blocking fashion. The operation is quick to complete in all cases, with no obvious performance issues.

## Setting a default value on a non-nullable column

### PostgreSQL 9.3
![](/images/ddl-profiling/postgresql-9.3/set-default-expression-on-non-nullable-column.png)

### PostgreSQL 9.4
![](/images/ddl-profiling/postgresql-9.4/set-default-expression-on-non-nullable-column.png)

### MySQL 5.5
![](/images/ddl-profiling/mysql-5.5/set-default-expression-on-non-nullable-column.png)

### MySQL 5.6
![](/images/ddl-profiling/mysql-5.6/set-default-expression-on-non-nullable-column.png)

Another hurray! Both versions of both MySQL and PostgreSQL seem to be doing this in a non-blocking fashion as well. The operation is quick to complete in all cases, with again no obvious performance issues.

## Modifying the type of a nullable column

### PostgreSQL 9.3
![](/images/ddl-profiling/postgresql-9.3/modify-data-type-on-nullable-column.png)

### PostgreSQL 9.4
![](/images/ddl-profiling/postgresql-9.4/modify-data-type-on-nullable-column.png)

### MySQL 5.5
![](/images/ddl-profiling/mysql-5.5/modify-data-type-on-nullable-column.png)

### MySQL 5.6
![](/images/ddl-profiling/mysql-5.6/modify-data-type-on-nullable-column.png)

Here we see that PostgreSQL 9.3 and 9.4 both perform this operation in a non-blocking fashion. The DDL statement is quick to complete, and doesn't show any obvious performance issues. For MySQL 5.5 we can clearly see that the table is put in a read-only mode, blocking DML queries other than SELECT. For MySQL 5.6 we can clearly see that the operation is non-blocking. It's quick to complete without any performance issues.

## Modifying the type of a non-nullable column

### PostgreSQL 9.3
![](/images/ddl-profiling/postgresql-9.3/modify-data-type-on-non-nullable-column.png)

### PostgreSQL 9.4
![](/images/ddl-profiling/postgresql-9.4/modify-data-type-on-non-nullable-column.png)

Here we see that PostgreSQL 9.3 and 9.4 both perform this operation in a non-blocking fashion. The DDL statement is quick to complete, and doesn't show any obvious performance issues. For both versions of MySQL this scenario could not be performed, [as it seems](http://stackoverflow.com/questions/3466872/why-cant-a-text-column-have-a-default-value-in-mysql) that these versions of MySQL don't support having default values for "TEXT" type columns, which was needed to perform this scenario.

## Modifying a column's type from integer to text

### PostgreSQL 9.3
![](/images/ddl-profiling/postgresql-9.3/modify-data-type-on-nullable-column.png)

### PostgreSQL 9.4
![](/images/ddl-profiling/postgresql-9.4/modify-data-type-on-nullable-column.png)

Here we see that PostgreSQL 9.3 and 9.4 both perform this operation in a non-blocking fashion. The DDL statement is quick to complete, and doesn't show any obvious performance issues. For both versions of MySQL this scenario could not be performed, [as it seems](http://stackoverflow.com/questions/3466872/why-cant-a-text-column-have-a-default-value-in-mysql) that these versions of MySQL don't support having default values for "TEXT" type columns, which was needed to perform this scenario.

## Adding a nullable foreign key

### PostgreSQL 9.3
![](/images/ddl-profiling/postgresql-9.3/add-nullable-foreign-key.png)

### PostgreSQL 9.4
![](/images/ddl-profiling/postgresql-9.4/add-nullable-foreign-key.png)

### MySQL 5.5
![](/images/ddl-profiling/mysql-5.5/add-nullable-foreign-key.png)

### MySQL 5.6
![](/images/ddl-profiling/mysql-5.6/add-nullable-foreign-key.png)

Ah this is terrible. For both versions of PostgreSQL and MySQL this operation is performed in a blocking fashion. This operation takes a while and blocks other DML queries while it is being executed. Both versions of MySQL do allow for SELECT statements to be executed while this operation is in progress.

## Adding a non-nullable foreign key

### PostgreSQL 9.3
![](/images/ddl-profiling/postgresql-9.3/add-non-nullable-foreign-key.png)

### PostgreSQL 9.4
![](/images/ddl-profiling/postgresql-9.4/add-non-nullable-foreign-key.png)

### MySQL 5.5
![](/images/ddl-profiling/mysql-5.5/add-non-nullable-foreign-key.png)

### MySQL 5.6
![](/images/ddl-profiling/mysql-5.6/add-non-nullable-foreign-key.png)

This scenario is terrible as well. For both versions of PostgreSQL and MySQL this operation is performed in a blocking fashion. This operation takes a while and blocks other DML queries while it is being executed. Both versions of MySQL do allow for SELECT statements to be executed while this operation is in progress.

## Conclusion.

We've looked at various scenarios involving several "refactorings" one can apply to tables in a SQL databases. We've seen that some operations appear to be safe enough to be used on live production databases whereas other should be avoided. We have seen that there have been significant improvements for MySQL 5.6 compared to MySQL 5.5. I have not observed any improvements for PostgreSQL 9.4 over 9.3. Same as in the previous blogpost, there are some caveats to these experiments:

* Tuning the configuration of a database may yield better performance which may lower the duration of some of these operations. For these experiments I only assigned more memory to be used by PostgreSQL and MySQL (around ~4GB), and to make better use of the SSD by lowering disk lookup penalties for PostgreSQL. I didn't tune any other aspect of the databases which may have yielded better performance.
* The duration of these operations also seems to take significantly longer when the database is starved of resources. Having a slow disk, or a almost fully utilizing the CPU seems to negatively affect the performance of these DDL statements. In this series I used an SSD drive, which significantly sped up all queries done by Nemesis on both MySQL and PostgreSQL.
* These operations may show different blocking or non-blocking behavior in different circumstances. Applying these operations to table with a different structure, constraints, or references may yield different results.

In other words: our measurements apply only to this particular setup in which we have tested these scenarios. If you wish to find out if a certain operation can be safely performed in your production environment, you would do well to test it on a staging environment which replicates the production environment (and it's workload) as best it can.