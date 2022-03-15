---
comments: true
layout: post
title: Regression in taking LIMIT rows from a bucket table in Spark 3.1.1
---

It's been almost three years since my [last post on Spark Python UDF](https://manuzhang.github.io/posts/2019-04-18-spark-python-udf/index.html) and I've dug deeper and learned more about Spark. I've also kept a practice of writing down how I approached to issues at work. However, I usually find they are either too specific to our own environment or have been well documented or I was occupied with some reading. Eventually, there can be no more "excuse" and here we go again!

### Regression

During migration of Spark applications from 2.3.1 to 3.1.1, a regression was reported for the following query, whose running time increased from 33 seconds to over 3 minutes.

```sql
SELECT i, k
FROM t
WHERE j='z'
LIMIT 10;
```

`t` is a bucket table, with 10000 buckets, bucket column `i` and sort column `j`, and each bucket file is around 3 GB.

### Diagnostics 

Firstly, looking from the application UI, the query was executed quite differently. In 2.3.1, the result was returned early after 1 task reading 487.7 MB data, which is an expected behavior for `LIMIT` clause. In 3.1.1, however, more than 3 TB data need be read with over 200k tasks from 9 stages. It's almost a full table scan!

![2.3.1 stages](https://user-images.githubusercontent.com/1191767/158113221-64676211-a869-4949-97ef-498cd34ceb22.png)

![3.1.1 stages](https://user-images.githubusercontent.com/1191767/158113505-7aae47b0-13d5-42c7-a8bb-55872a15d5e1.png)

Meanwhile, this driver log in 2.3.1 indicted it's a bucketed scan which I didn't find in 3.1.1.

```
INFO FileSourceScanExec: Planning with 10000 buckets
```

Okay, I remember that in bucketed scan, when dividing source data into partitions, a bucket file is a partition regardless of its size. Otherwise, the partition size is decided by `spark.sql.files.maxPartitionBytes` which defaults to 128 MB. Hence, for 10000 bucket files, no more than 10000 tasks are needed to find 10 rows satisifying `j='z'` with bucketed scan. In contrast, there could be as many as 240,000(`3 * 1024 * 10000 / 128`) tasks with non-bucketed scan. Nonetheless, in this case Spark looked no further than the first file with bucketed scan in 2.3.1, which corresponded to at most 24 tasks with non-bucketed scan. 

#### Why were over 200k tasks launched in 3.1.1?
 
> the devil is in the details

After checking the codes, it turns out partitions are [sorted and read in reverse size order](https://github.com/apache/spark/blob/v3.1.1/sql/core/src/main/scala/org/apache/spark/sql/execution/DataSourceScanExec.scala#L609) in non-bucketed scan. The tail of the first file, whose size is smaller than 128 MB, is left until the non-tail parts of all files have been scanned. And when the answer happened to lie in the tail of the first file here, all files need be scanned! 

#### And why was the bucketed scan disabled in 3.1.1? 

The behavior change was brought in by [[SPARK-32859][SQL] Introduce physical rule to decide bucketing dynamically](https://github.com/apache/spark/pull/29804) with a new config   `spark.sql.sources.bucketing.autoBucketedScan.enabled`. The config was later [enabled by default except for cache query](https://github.com/apache/spark/pull/30138). After switching off the config and always enabling bucketed scan in 3.1.1, the query is executed as fast as 2.3.1.

The rational behind the change is to disable bucketed scan when there are no benefits, e.g. no join or group by on bucket column. It looks, however, not all cases have been considered when deciding whether bucketed scan is benefitting.

#### How were the tasks scheduled in 3.1.1?

One more puzzle for me is the special incremental way the tasks were scheduled in 3.1.1. Searching for keyword "limit" in configs has led me to `spark.sql.limit.scaleUpFactor` whose doc says

> Minimal increase rate in number of partitions between attempts when executing a take on a query. Higher values lead to more partitions read. Lower values might lead to longer execution times as more jobs will be run. (Default value is 4)

I tried increasing the config to 300k and the performance was a bit better with 239999 tasks immediately being scheduled in the second stage. I suppose it would much better if I could got more resources to run those tasks in parallel.

![3.1.1 stages with large scaleUpFactor](https://user-images.githubusercontent.com/1191767/158311306-26e955d5-4eef-45e6-b017-b7dd36c52f18.png)


### Summary

These configs are worth checking when taking `LIMIT` rows with filter from a bucket table.

Name | Default | Meaning | Since Version
---- | ------- | ------- | -------------
spark.sql.files.maxPartitionBytes | 128 MB | The maximum number of bytes to pack into a single partition when reading files | 2.0.0
spark.sql.sources.bucketing.enabled	| true | When false, we will treat bucketed table as normal table | 2.0.0
spark.sql.sources.bucketing.autoBucketedScan.enabled | true | Whe true, decide whether to do bucketed scan on input tables based on query plan automatically | 3.1.1
spark.sql.limit.scaleUpFactor | 4 | Minimal increase rate in number of partitions between attempts when executing a take on a query | 2.1.1

I also [raised to the community that such behavior change need be put in migration guide but they didn't agree](https://github.com/apache/spark/pull/35514).

An important lesson I've learned is to resolve all the doubts before turning to other suspicious points. In this case, I got distracted by the differences in execution plans and spent a lot of time in comparing how `LIMIT` clause was compiled into physical plans from 2.3.1 to 3.1.1.

### References

1. [Spark 3.1.1 Configuration](https://spark.apache.org/docs/3.1.1/configuration.html)