---
title: The "Spark writes too many output files" conundrum
date: 2021-05-03T19:00:00+01:00
draft: false
tags: 
    - spark
    - glue
    - bigdata
    - s3
category: spark
keywords: 
    - spark
    - glue
    - bigdata
    - s3
---

As a Big Data engineer, one of the most common problems I see people facing is "why is Spark writing 300 files". Many engineers (specially ones in BI environments) are used to having datasets written as a single file, typically in a plain text format like CSV or JSON. Not only that, but many tools like Databases or BI dashboards expect data to be part of a single file - which obviously clases with how Spark works.

## Why does this happen?

*Warning: the text below is a simplified, high-level explanation of Spark internals. If you want deeper understanding, please refer to [the bible](https://www.oreilly.com/library/view/spark-the-definitive/9781491912201/).*

Spark is a Big Data framework designed to process VERY large amounts of data. The way it achieves this is by distributing your data over a cluster of nodes, and having each node process it independently rather than a single machine do all the work. This distribution (or partitioning) of data is the culprit here.

Whenever you execute a Spark application, Spark will check the contents of the source path you specified. Take an AWS S3 path for example: 

1. Spark will list how many files are in the path along with their size, then create in-memory partitions out of those files. 
2. Each partition will be mapped to a Spark task, and then Spark will launch as many Spark executors as necessary to process these tasks concurrently. 
3. Each executor will execute your job's transformations on a different node in a cluster
4. Once all operations in your job are completed, each executor will then write each in-memory partition as an output file to your destination path. 

This is why many output files are generated: there will be one output file for each Spark partition your dataset is partitioned into at the moment of writing. 

## How do I fix this?

Unfortunately, there's no easy solution here. Following the aforementioned logic, the only solution would be to have only one Spark partition before writing - but let me explain why that's a terrible solution.

### Repartitioning to 1 partition

As explained above, Spark partitions your data in order to have many nodes process it instead of a single one. This lets you do things like [processing 100 TB of data in-memory](https://opensource.com/business/15/1/apache-spark-new-world-record) by having many small machines instead of having to somehow build a single machine with more than 100 TB of RAM.

Even though Spark and its connectors have their own logic to handle partitioning for you, you can still manually alter this with either [repartition](https://spark.apache.org/docs/latest/api/java/org/apache/spark/sql/Dataset.html#repartition-org.apache.spark.sql.Column...-) or [coalesce](https://spark.apache.org/docs/latest/api/java/org/apache/spark/sql/Dataset.html#coalesce-int-). So you could use coalesce ([and only coalesce](https://stackoverflow.com/questions/31610971/spark-repartition-vs-coalesce)) to reduce the number of partitions to 1, then write.

Now I think you'll probably know where I'm heading with this. If you really are using Spark to process large amounts of data, changing the number of partitions to 1 will:

* Send all data to one node, which means a massive network shuffle. Bad performance.
* Have only one thread of one executor process all your data. Again, bad performance, but also you are most likely going to crash the executor out of memory.

The truth of the matter is Spark was never designed to do this, and even if it gave you the option, many distributed systems like S3 or HDFS favor distributed reads rather than having thousands of API calls or read operations towards the same file.

**TL;DR: only do this when strictly necessary**, and be aware that it is a terrible practice.

### Compacting after writing

If writing to a single file is not possible, the only solution is to compact files after writing them. How this happens will greatly depend on:

* The filesystem the files are in. Many systems like HDFS or S3 don't support append operations, so the only solution is to read each file, concatenate it in-memory and write the output. Obviously if this is done in-memory it might not be possible in Big Data contexts.

* The file format you are using. Many file formats do not support simply appending to the end of a file, so again only in-memory processing with a library that understands the format can be done.

* The compression format being used. Same as before: you can't simply `cat` two zip files expecting them to magically merge.

So again, there's no easy, works-for-all solution here.

### The S3 workaround

If you are dealing with plain-text, S3-stored datsets, there's some light at the end of the tunnel. S3 allows for [multipart upload operations](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html), which let users upload/copy large files at faster speeds by (like Spark does) dividing them into smaller chunks and submitting them individually. 

Multipart uploads can be used to copy, and in this case one could do one-to-many or many-to-one operations - the latter being exactly what we want here. AWS demonstrates such a scenario with the Ruby SDK [here](https://aws.amazon.com/blogs/developer/efficient-amazon-s3-object-concatenation-using-the-aws-sdk-for-ruby/), but it could be implemented in any language, and it could be sped up with threading since [multipart uploads can be concurrent](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html#distributedmpupload). A simple test on my end reveals ~10 seconds to compact 1GB worth of JSON files - which is actually pretty good.

## Conclusion

This is a complex problem that has more to do with industry standards than with any application in particular - data processing and visualization tools have to start accepting *a path* rather than *a file* as the input.

If you are using S3, writing your own custom process to compact uncompressed, plain-text files is possible. If you are running your data processing workloads using AWS Glue for instance, writing a simple Python Shell job that runs after your main Spark job and compacts all files should be quite simple and effective. Even more so, now that [Glue supports custom connectors](https://aws.amazon.com/about-aws/whats-new/2020/12/aws-glue-launches-aws-glue-custom-connectors/) one could create a custom connector that writes all data to a single S3 object - although I imagine there's several limitations here such as the S3 transaction limit, and the fact that S3 doesn't really favor submitting large volumes of requests towards the same key. Also there's a 5MB minimum size for multipart parts - which means jobs will fail if partitions are smaller than that.

In any case this is not something easy to solve, and there's definitely room for a cool project here.
