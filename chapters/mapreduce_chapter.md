# MapReduce

##Introduction
MapReduce is the essential framework to process Big Data at least today. And, of course, the author will eventually add more text here... Just be patient...

## MapReduce without MapReduce or a cluster
It is actually possible to illustrate the work of MapReduce without having Hadoop or any other cluster with just the command line interface. We have earlier mentioned the Hello World problem in Big Data, which is Word Count. The task is to count the number of occurrences of each word in a potentially large text file. What is the solution to this problem in the MapReduce way? 

It is actually quite simple. The file contents is sent to a program called `mapper` that splits the text into words and emits strings like `"<word_1> 1"`,`"<word_2> 1"`, and so on. Occasionally, the words will repeat (the text is long), however the mapper still outputs `1` for each word no matter how many times it has appeared in the text.

At the next step, MapReduce framework rearranges the strings so that the similar words are put together. In CLI, this can be simulated by calling function `sort`. At this time, the list of strings may look like this: `"<word_1> 1"`,`"<word_1> 1"`,`"<word_2> 1"`,...,`"<word_n> 1"`.

The rearranged strings will get to the input of a `reducer` program that adds up all the `1` for each word and prints the word counts: ,`"<word_1> n_1"`, ,`"<word_2> n_2"`.

Below is an example that can run on Linux:
```console
$ cat shakespeare.txt | python mapper.py | sort | python reducer.py
```

Mapper.py
```python
import sys
import re

for line in sys.stdin:
    line = line.strip()
    words = re.split("\W+",line)
    for word in words:
	if word != '':
		word = word.lower()
        	print('%s\t1'%word)
```

Reducer.py
```python
import sys

cur_word = ""
cur_count = 0
for line in sys.stdin:
    word, count = line.strip().split('\t')
    count = int(count)
    if word == cur_word:
        cur_count += count
    else:
        if cur_word != "":
            print("%s\t%d"%(cur_word,cur_count))
        cur_word = word
        cur_count = count
            
print("%s\t%d"%(cur_word,cur_count))
```

What are the benefits of this code organization, i.e. splitting the processing into mapper and reducer? First, in case of large text files, the system can run multiple mappers simultaneously. The mappers can work on those nodes that contain file chunks and send the outputs to the common `data stream`. The reducers can also work simultaneously, as long as one word is not split between two or more reducers. However, Hadoop framework guarantees that this will not occur. Even if it does, another series of reducers will fix it. 

One last comment before running MapReduce on Hadoop. The word count is in fact a toy problem, which purpose is only to the general mechanism of the framework. For more complex problems, multiple mapper-reducers can be stacked so that the output of reducer n is the input of mapper n+1. 

##MapReduce on Hadoop
Now, finally, let us run our word count code on Hadoop. If HDFS in your system is still empty, go ahead and copy (`-copyFromLocal`) shakespeare.txt file, because it will be needed. Since Hadoop uses Java natively, running mapper and reducer in other languages is referred to as `streaming`. So, below is one example of streaming that does a half of the task, i.e. the mapping:

```console
yarn jar /opt/hadoop/hadoop-streaming.jar \
-files mapper.py \
-mapper 'python mapper.py' \
-numReduceTasks 0 \
-input texts/shakespeare.txt \
-output wordcount
```

The image below shows a portion of the verbose output by Hadoop. Among other things, it shows the progress of the mappers (and reducers, once we add them). Even though in this example Hadoop runs in pseudo distributed mode, the data is still split between the two mappers.
![Hadoop_streaming_output](/images/figures/hadoop_streaming_output_top.png)

Per our request, Hadoop used wordcount as the output directory. The directory contains three files. The first with the self-illustrating name indicates that the job has finished succesfully. The other two files are the outputs by the two mappers. The figure below also illustrates the first few lines in one of the files:
![Hadoop_streaming_output_dir](/images/figures/hadoop_streaming_output_dir.png)

### Unreliable components
Now that we see how Hadoop manages the properly operating components, let us simulate a node failure. Since when a node fails, all the jobs, which are running on the node will fail, this can be done by randomly failing a mapper. Let us randomly through an exception in the mapper, so that out of the two mappers may be one will fail. 

```python
import sys
import random
import re

if random.randint(0,10) > 3:
    raise Exception("Bang!!!")
    
for line in sys.stdin:
    line = line.strip()
    words = re.split("\W+",line)
    #words = line.strip().split(' ')
    for word in words:
        word = word.lower()
        print('%s\t1'%word)
```

Do not forget to delete the outputs of the previous job with `-rm -r -f <hdfs_dir>`. The command to re-run the job will be exactly as before:

```console
hdfs dfs -rm -r -f wordcount
yarn jar /opt/hadoop/hadoop-streaming.jar \
-files mapper.py \
-mapper 'python mapper.py' \
-numReduceTasks 0 \
-input texts/shakespeare.txt \
-output wordcount
```

Now we can see that during the run, one of the mappers had crashed. However, Hadoop started another mapper and recovered:
![Hadoop_streaming_output_failed_mapper](/images/figures/hadoop_output_failed_mapper.png)

So, at the end of the job, the total number of mappers launched was 3, where one mapper had failed.
![Hadoop_streaming_output_failed_mapper_counters](/images/figures/hadoop_output_failed_mapper_counters.png)

### Adding the Reducer
In order to add the reducer, its name has to appear in the `-files` and at `-reducer`. Besides, the number of reduce tasks `-numReduceTasks` has to be greater than zero. If this is the only MapReduce task, the number of the output files (excluding `_SUCCESS`) will match the number of the reducers. 

```console
hdfs dfs -rm -r -f wordcount

yarn jar /opt/hadoop/hadoop-streaming.jar \
-files mapper.py,reducer.py \
-mapper 'python mapper.py' \
-reducer 'python reducer.py' \
-numReduceTasks 1 \
-input texts/shakespeare.txt \
-output wordcount
```

## Exercises
1. Use MapReduce to count the number of words of each length in text. Example output:
```
1 1234
2 22100
3 2312
...
50 1
```
This means that the text contains 1234 one char long words, 22100 two character long words, etc.
2. You work for a social network and you deal with a file containing users with a list of friends. Suppose users are: A,B,C, and D. Then the file format can be as follows: 
```
A [B,C,D,E]
B [A,D,E]
C [A]
```
Your task is to create a MapReduce job that will return friends in common between two users:
```
A B [D]
B C []
```
3. Implement `SELECT * FROM <table> WHERE <condition>` with MapReduce.
4. Implement `SELECT MAX(<field>) FROM <table> GROUP BY <field>` with MapReduce.
5. One method for computing Pi (even though not the most efficient) generates a number of points in a square with side = 2. Suppose a circle with radius 1 is inscribed into the square and out of 100 points generated, 75 lay on the circle. Then, `4*75/10 ~= 3` approximates Pi. 
Write MapReduce code that implements the method. Hint: make mappers generate the points and reducer count the ratio. 
6. Write MapReduce code to implement matrix multiplication.




