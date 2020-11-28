---
layout: blog
title: "Notes on the “Apache Hive” paper" 
date: 2020-11-27
tags: [tech]

---
Hive is a data warehouse solution that allows users to execute SQL-like queries on Hadoop. It was initialized by Facebook in 2010.

# What problem does Hive try to solve?
The tech Giants like Facebook then were experiencing rapid data growth. Facebook data grew from 15TB in 2007 to 700TB in 2010, a scale the traditional DB could not handle well.

Before Hive, there is the distributed computing and storage system Hadoop. However the map-reduce interface is not user friendly. It usually took several hours to just write a word count program. Hive aims to address the problem of lacking an efficient and user-friend interface to process data in distributed systems.

# What is the proposed solution?
The Hive abstracts the data processing on distributed systems as the interface called HiveQL. It is similar to SQL with some limitations and extensions.

## Data Model & Query Language
What data model does Hive take? Hive models the data as tables. The column can have primitive types `(bigint, int, smallint, tinyint, float, double, and string)` and complex type `map<key, value>, list<element>, struct<field-name: field-type, ...>`.

Here is an example of create a table t1.
```
CREATE TABLE t1 (st string, fl float, li list<map<string, struct<p1: int, p2:int>>);
```
and access the field of one column
```
SELECT li[0][‘key’].p2 FROM t1;
```

The HiveQL supports common SQL syntax such as SELECT, WHERE, GROUP BY, JOIN, UNION. One limitation of HiveQL is not supporting inserting into an existing table or data partition (INSERT INTO, UPDATE, DELETE). all inserts will overwrite the existing data. On ther other hand, HiveQL extends SQL with analysis expressed as map-reduce programs.
```
FROM (MAP doctext USING ‘python wc_mapper.py’ AS (word, cnt)
FROM docs
CLUSTER BY word
) a
REDUCE word, cnt USING ‘python wc_reduce.py’;
```

## Data Storage
With the data model and query language defined, how does Hive store the data in HDFS?
There are three layers in the file system to store tables.
* Tables — A table is stored in a directory in hdfs.
* Partitions — A partition of the table is stored in a subdirectory within a table’s directory.
* Buckets — A buck is stored in a file within the partition’s or table’s directory.
![](../../assets/images/hive_post/systems.png)

The partition greatly improves the query efficiency. Usually the HiveQL query filters the table by certain partitions. By adding the partition subdirectory, the hive system only needs to scan the files under the matched partition directories.
```
CREATE TABLE test_part(c1 string, c2 int)
PARTITIONED BY (ds string, hr int);
SELECT * FROM test_part WHERE ds = ‘2020–10–20’
```

The bucket shards the data under one partition and improves the efficiency of sampling. If the data under one partition is still too big to fit into one file, the bucket is the solution. A bucket can be created with the clause CLUSTERED BY
```
CREATE TABLE user_info_bucketed(user_id BIGINT, firstname STRING, lastname STRING)
COMMENT ‘A bucketed copy of user_info’
PARTITIONED BY(ds STRING)
CLUSTERED BY(user_id) INTO 256 BUCKETS;
```
The bucket number is computed by hash(bucketing_column) MOD num_buckets.
After creating these paths, Hive needs to write the table to the files on INSERT and read the table from the files on SELECT. Hive provides two file formats: TextInputFormat/TextOutputFormat for text files and
SequenceFileInputFormat/SequenceFileOutputFormat’ for binary files. Hive also provide built-in SerDe to serialize/deserialize the data. Customized SerDe is supported.
# System Architecture
Given the data model, query language and data storage, how does Hive actually execute the query? To answer this we need to dive into the Hive system. The hive architecture consists of several components:
* Metastore — stores the metadata about tables, columns, partitions etc.
* Driver — manages the lifecycle of a HiveQL statement.
* Query Compiler — translates the HiveQL to DAG map-reduce tasks.
* Execution Engine: executes the tasks
* HiveServer — thrift server, JDBC/ODBC server
* Clients component — CLI, web UI
* Extensibility interfaces: SerDe, UDF etc
![](../../assets/images/hive_post/systems-Hive architecture.png)
## Metastore
The metastore has two main use cases. 1. User query and modify the table information. 2. Hive Compiler accesses the table information to compile HiveQL. The metastore is built on a traditional RDBMS as the compiler needs low latency serving. Also availability and scalability are necessary and the metastore is replicated in servers.
## Query Compiler
The compiler is the core part of the Hive system. It compiles the HiveQL into an execution plan with the following steps:
1. Parsing. convert HiveQL into AST.
1. Type checking and Semantic Analysis. convert the AST to Query block(QB) tree, and intermediate representation of DAG.
1. Optimization. It performs a chain of transformations such as column pruning, predicate puhdown, partition pruning, and etc. User-defined transformations can be added to the chain. The original paper has a detailed description of the step.
1. Generation of the Physical Plan. The logical plan generated at the end of the optimization phase is then split into multiple map/reduce and hdfs tasks.
Here is an example provided by the original paper. Given two tables: status_updates (userid int, status string, ds string) and profiles (userid int, school string, gender int),
The query
```
FROM (SELECT a.status, b.school, b.gender
FROM status_updates a JOIN profiles b
ON (a.userid = b.userid
AND a.ds=’2009–03–20' )) subq1
INSERT OVERWRITE TABLE gender_summary
PARTITION(ds=’2009–03–20')
SELECT subq1.gender, COUNT(1)
GROUP BY subq1.gender
INSERT OVERWRITE TABLE school_summary
PARTITION(ds=’2009–03–20')
SELECT subq1.school, COUNT(1)
GROUP BY subq1.school
join the two tables, aggregate the students by gender and by school and dump the aggregated data into two tables.
```
![](../../assets/images/hive_post/systems-HiveQL - MR DAG.png)
HiveQL to Hadoop DAG of tasks
## Execution Engine
The engine schedules the map/reduce tasks to Hadoop. It serializes the DAG into plan.xml, which is deserialized by ExecMapper and ExecReducers instances in Hadoop. The final result is stored in the desired location.
# Miscs
It has been 10 years since the Hive paper was published. The Hive has been transferred to the Apache Software Foundation and Hive has gone through many iterations, but the fundamental idea and system design are still valid.

# References
[Hive — A Petabyte Scale Data Warehouse Using Hadoop[2010]](http://infolab.stanford.edu/~ragho/hive-icde2010.pdf)
https://cwiki.apache.org/confluence/display/Hive/SerDe
https://cwiki.apache.org/confluence/display/Hive/LanguageManual+Select
https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL+BucketedTables