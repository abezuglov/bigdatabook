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

## Actions: Take vs. Collect 
So far, we have experimented with a single action command `take(x)` that returns the first `x` rows of the dataset. This action is similar to invoking `head` in the command line interface. Take is a perfect action to make sure that the transformations produce expected outcomes. Action `collect` will return all rows in the dataset, so use it carefully.

## Transformations

### Map and flatMap
Map transformation is similar to the map function in MapReduce. In PySpark, map takes a function or (lambda function description) that performs the transformation. For example, consider ISIS religious texts file (available as a part of spark container), where each row needs to be transformed to a dictionary:

```python
def dict_like(row):
    keys = ['Magazine','Issue','Date','Type','Source','Quote','Purpose',_
		'Article Name']
    try:
        items = row.split(',',7)
        print(items)
        return dict(zip(keys,items))
    except:
        pass

dict_isis = isis.map(dict_like)
dict_isis.take(2)
```

The function specifies the transformation for each row of the dataset. The name of the function (dict_like) is passed as a parameter to map method of RDD. In case of simple single line transformations, a regular lambda function can be used:

```python
magazine_title = isis.map(lambda row:row.split(',',1)[0])
magazine_title.take(5)
```

When map generates more than one output per data element in RDD, the result will get a two (or more) dimensional list. For example, a map function that splits the rows in ISIS RDD into words:

![2D Map](images/figures/spark_2d_map.png)

Since each row may produce multiple words, the output is a two dimensional list. As its name suggests, flatMap will flatten the output and produce a regular list containing the words. A word count example demonstrating the usage of flatMap is below.

### Reduce by key
Another transformation from the MapReduce paradigm is `reduceByKey'. The function determines the operation to perform on the data sharing the same key. The function is both associative and commutative, since the order of the key-value pairs is not necessarily defined. Optionally, reduceByKey may specify the key partitioner. 

Consider the word count example implemented in Spark:
```python
import re

def word_split(row):
    words = re.split("\W+",row)
    words = [w.strip().lower() for w in words]
    return words
    
isis.flatMap(word_split).map(lambda x: (x,1)).reduceByKey(lambda x,y:x+y). \
	sortBy(lambda x:-x[1]).take(20)
```
The sequence of transformations here repeats the classic word count example. First flatMap splits the text into a list of words, then map transforms each word to a key-value pair `<word,1>`. Reduce sums the values pertaining to the same key and sort orders them in descending order. The final action take triggers the computation and produces the top twenty most frequent words in RDD. 

### Filter
Filter transformation allows selecting or skipping over the rows in the dataset. The (lambda) function here must return a boolean value to select (True) or filter out (False) the rows. The two examples below demonstrate the selection of all ISIS texts where the issue number is not equal to '1':

```python
issues = isis.filter(lambda row:row.split(',',2)[1] != '1')
issues.take(10)
```

and published in July:
```python
july_publications = isis.filter(\
	lambda row:row.split(',',3)[2].find('Jul') != -1 \
	).take(5)
```

### A few other operations
* sortBy -- One of the above examples used sortBy transformation that takes a lambda function to sort words by frequencies in descending order
* count -- This action returns the number of items in the dataset.
* foreach -- Applies transformation by the parameter function to the items in the dataset. 
* sample -- Transformation to select a portion of the dataset at random.
* distinct -- Returns distinct elements in the dataset

## Closures
Spark uses a rather complex method of scheduling and running operations on cluster, therefore using functions with side effects is never a good idea. The code below is an illustration of a bad Spark programming:
```python
counter = 0
def incCounter(row):
	global counter
	counter += 1

a = sc.parallelize(range(100))
a.foreach(incCounter)
print(counter)
```

Spark first breaks RDD operations into tasks that are run by executors. For each task, Spark provides the closure that contains the local variables, functions, etc. As a result, the executors will contain the `copies` of the counter variable and therefore the code output is undefined. Only in a situation, when the complete transformation gets to the single executor, the code above will produce the expected output (100). 

Using `foreach(print)` to output RDD elements should be avoided for a similar reason. It still prints the elements correctly, however the standard output is no longer the client terminal, but the standard output of the executor. 

## Review questions

## Exercises
1. Use ISIS religious texts. Make a dataset the top 10 quotes from Hud. 




