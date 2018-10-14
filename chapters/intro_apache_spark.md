# Apache Spark

## Introduction
Apache Hadoop permits is a powerful framework for processing Big Data. It allows parallel processing of huge amounts of data by breaking it into chunks and processing them on the nodes. It automatically gathers the output from the nodes, shuffles, sorts it, and saves on HDFS. Besides, Hadoop framework is also an economical solution for non-time critical results. 

On the other side, some Hadoop ecosystem features limit its use to all analytics problems in Big Data. Below are a few:

* While Hadoop is good for a small number of huge files, its performance on a large number of small files deteriorates;
* Slow processing speed to dumping intermediate results to disk;
* Batch processing only, no interactive ad-hoc processing;
* No real-time data processing;
* Mapping-reducing is not always intuitive (like SQL);
* Limited abstraction -- it is hard to deal with complex code without abstraction;
* Cluster RAM is not utilized well;

![Searches for 'Hadoop' on Google \label{Hadoop_searches}](images/figures/google_hadoop.png)

Figure \ref{Hadoop_searches} shows the amount of Google searches for Hadoop since 2004. The trend reached its maximum around year 2014 and it has been declining since then. Other tools have come to existence that eliminate or mitigate the limitations of Hadoop ecosystem.

The primary focus of this chapter is Spark, another product by Apache Software Foundation. Spark started as a project by UC Berkeley in 2009 and it came to its first release in 2012. One of the major features of Spark at this time was its in-memory data processing implemented with a `Resilient Distributed Dataset (RDD)`. Later versions of Spark (under Apache) have implemented interactive queries and stream processing, fast ad-hoc analysis and fast decision-making, GraphX and MLib. The latter two are the libraries for graph processing and machine learning. 

## Resilient Distributed Dataset (RDD)
The central concepts of RDD are that it is (1) resilient and (2) distributed. The data in RDD is residing on multiple nodes in a cluster (distributed) and whenever a node is unavailable, the dataset is able to recompute the missing or damaged partitions (resilient). Another important feature of an RDD is that it is `immutable`. The output of all data transformation operations is a _new_ dataset. 

There are two types of operations on a dataset: `transformations` and `actions`. The transformations change the data and return a new dataset. The sequence of transformations is recorded, but the transformations themselves are not invoked immediately, the are _lazy_. Actions are the operations that trigger the computations and they return values. An example of an action can be to return top five records from the dataset or similar. 

## Hands-on RDD
To try Apache Spark, download and start image: `abezuglov/data201_spark`. The figure below shows the start of Apache Spark console (pyspark) from the terminal window in the container and a few commands with their outputs.

![Apache Spark](images/figures/cmd_pyspark.png)

Pyspark is a Python interface to Apache Spark and it runs as `pyspark`. It creates a globally accessible object SparkContext (`sc`) that is further used to invoke other commands. 

To create an RDD from fixed data, use `sc.parallelize()`. Note that printing RDD will not return its data and to access it an `action` operation has to be used. The example below uses action `take`:

```console
$pyspark
>>>a = sc.parallelize([1,2,3])
>>>print(a)
>>>print(a.take(2))
```

## Map Transformation

## Filter Transformation




