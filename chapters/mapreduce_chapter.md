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
    words = re.split("\W+",line)
    for word in words:
	word = word.lower()
	if word != '':
        	print("%s\t%d"%(word,1))
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
$ yarn jar /opt/hadoop/hadoop-streaming.jar \
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
    for word in words:
        word = word.lower()
        print('%s\t1'%word)
```

Do not forget to delete the outputs of the previous job with `-rm -r -f <hdfs_dir>`. The command to re-run the job will be exactly as before:

```console
$ hdfs dfs -rm -r -f wordcount
$ yarn jar /opt/hadoop/hadoop-streaming.jar \
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

### Adding Reducer
In order to add the reducer, its name has to appear in the `-files` and at `-reducer`. Besides, the number of reduce tasks `-numReduceTasks` has to be greater than zero. If this is the only MapReduce task, the number of the output files (excluding `_SUCCESS`) will match the number of the reducers. 

```console
$ hdfs dfs -rm -r -f wordcount
$ yarn jar /opt/hadoop/hadoop-streaming.jar \
-files mapper.py,reducer.py \
-mapper 'python mapper.py' \
-reducer 'python reducer.py' \
-numReduceTasks 1 \
-input texts/shakespeare.txt \
-output wordcount
```

After it finishes, the result is available in `wordcount/part-00000`:
```console
$ hdfs dfs -cat wordcount/part-00000 | head
```

## Debugging
By now you have likely tried a few MapReduce examples and perhaps not all of the successfully finished. How can one debug this code? It may get difficult for at least two reasons: (1) when a mapper or reducer fail, Hadoop will still try to re-run them and (2) the error messages are not too informative. One suggestion here is to first try the code using the format:
```console
$ cat <input_file> | python mapper.py | sort | python reducer.py
```

## Passing Files to the Nodes, a.k.a. Distributed Cache
While processing inputs, some MapReduce problems will need additional data. One example can be a word count task that skips the most common English words as non-informative. Suppose the list of such words is contained in a local file `stopwords.txt`[^stop_words_DFS]. Then the rest is a simple algorithmic task, where the mapper will have to pass through each word, check if the word is not in the stop words list and output the word. The portion of the code responsible for opening the file may look like this:
```python
...
def readfile(filepath):
	with open(filepath,'r') as f:
		words = f.read().split('\n')
		return words
...
	if word not in stop_words:
		# output <key,value>
...
```
However, the issue here is that both the mapper and reducer run at the `nodes` and not at the client system. Since `stopwords.txt` is missing at the nodes, the mappers will fail. To explicitly forward the file to the nodes, its name has to appear in the `-files` while starting the task:
```console
$ yarn jar /opt/hadoop/hadoop-streaming.jar \
-files mapper.py,reducer.py,stopwords.txt \
-mapper 'python mapper.py' \
-reducer 'python reducer.py' \
-numReduceTasks 1 \
-input texts/shakespeare.txt \
-output wordcount
```
Now, can mapper or reducer open files on the nodes for writing? Whether Hadoop allows this operation or not, it is not a good practice, since it makes mapper and/or reducer non-deterministic. In presence of failures, MapReduce can only guarantee repeatable results if both mapper and reducer return the same outputs for the same inputs, i.e. be deterministic.

In case MapReduce job needs multiple files, or the files get large, they can be sent as an archive. To illustrate this, let us analyze the first names that Sir William Shakespeare had used in his sonnets. Let us download the U.S. Census list of the first names[^first_names_url], save it locally with `curl <url> > first-names.txt`, and compress using `tar -czf <archive_name> <file(s)>`. 
```console
$ tar -czf first-names.tar first-names.txt
```

The file opening in the mapper will change to this:
```python
...
first_names = openfile('first-names.tar/first-names.txt')
...
```

... and the job will start with `-archives` parameter:
```console
$ yarn jar /opt/hadoop/hadoop-streaming.jar \
-files mapper.py,reducer.py \
-archives first-names.tar \
-mapper 'python mapper.py' \
-reducer 'python reducer.py' \
-numReduceTasks 1 \
-input texts/shakespeare.txt \
-output wordcount
```

### Unicode
Whenever the text data is in Unicode, make sure that you correctly process it. Below are a few operators to set default encoding to Unicode, to convert a string, and to split the string using `re`.
```python
...
import sys
reload(sys)
sys.setdefaultencoding('utf-8') 
...
line = unicode(line) # convert line to unicode
words = re.split("\W+",text, flags=re.UNICODE)
```

##Environment Variables and Counters
Hadoop creates a few useful environment variables available to the client code and it also allows the code to create their own variables. Boolean variable `mapred_task_is_map` allows checking if the current task is a mapper `"true"` or a reducer `"false"` and it can be used as follows:
```python
if os.environ['mapred_task_is_map'] == 'true':
	# mapper's code
else:
	# reducer's code
```
As the above example illustrates, it can be used to create a single code to work as both the mapper and reducer. Other environment variables available are: `mapreduce_map_input_file`, `mapreduce_map_input_start`,`mapreduce_map_input_length`, etc.

Hadoop will also create new environment variables if they are added to the yarn command. For instance, the command below will add `new_var` variable accessible via `os.environ["new_var"]`:
```console
$ yarn jar /opt/hadoop/hadoop-streaming.jar \
-D new_var = "new_value" \
...
```

### Calculating Pi with MapReduce

To illustrate the use of environment variables and parameters, let us calculate Pi with MapReduce. The method to calculate Pi will not be too efficient, however, it is easy to implement and fits the MapReduce model quite well. The idea behind the method is to find a ratio of points inside and outside of a circle insribed into a square. The mapper will generate a number of random points and output if the point is inside `1` or outside `0` of the circle. The reducer will find the ratio and output Pi. Below is the program that serves as both the mapper and reducer:
```python
import sys
import random
import re
import os

reload(sys)

num_points = 100 # default number of points
try:
	# attempt to get the number from parameters
	num_points = int(os.environ['num_points']) 
except:
	pass

if os.environ['mapred_task_is_map'] == 'true': # check if the job is a mapper
	for line in sys.stdin: #ignore whatever is in the input
		pass

	#generate points
	for i in range(num_points):
		x = random.uniform(0,1)
		y = random.uniform(0,1)
		if x*x+y*y < 1.0:
			sys.stderr.write("reporter:counter:Personal Counters,inside,1\n")
			print("1")
		else:
			sys.stderr.write("reporter:counter:Personal Counters,outside,1\n")
			print("0")
else: # The job is the reducer
	_sum = 0
	for line in sys.stdin:
        	_sum += int(line)
	pi = _sum*4.0/num_points
	print("Pi:%1.4f"%pi)
```

This MR job can run as normal, which will generate 100 points, or by sending the number of points as an environment variable on the nodes: `-D num_points=<>`. Note that in Hadoop streaming, `-input` is a mandatory parameter and cannot be omitted. This is a limitation of the example above for at least two reasons. First, in order to run the example, an empty file has to be created on HDFS (`hdfs dfs -touchz empty`); and second, there will be a limited amount of mappers, since the input data exists on only three HDFS nodes by default. Nethertheless, the example still works to illustrate environment variables and counters.
```console
yarn jar /opt/hadoop/hadoop-streaming.jar \
-D num_points=10000 \
-files mapper_params.py \
-mapper 'python mapper_params.py' \
-reducer 'python mapper_params.py' \
-numReduceTasks 1 \
-input empty \
-output pi
```

The counters can be found in the output, under personal counters:
![MR_counters_output](/images/figures/MR_counters_output.png)

### Data Aggregation
In Structured Query Language (SQL), data can be aggregated by a field or combination of fields. For instance, consider a data file below:
![Seattle_weather](/images/figures/road_weather_stations_seattle.png)

The file contains road and air temperature measurements at several locations around Seattle, WA. The task is to calculate average temperature across the locations, or aggregate by the date/time field. 

In MapReduce implementation, the mapper will scan through the file and use the date/time as the key, while leaving the combination of other fields as the value. The reducer will scan through the key-value pairs and aggregate the values pertaining to the same key, which is the date/time. Below is the sample code:

Mapper:
```python
import sys
import random
import re

for line in sys.stdin:
    l = line.split(',')
    if l[0] != 'StationName':
        loc, coord_lat, coord_lon, date_time, rec_id, road_temp,air_temp = l
        d = ','.join([loc,road_temp,air_temp.strip()])
        print("%s\t%s"%(date_time,d))
```

Reducer:
```python
import sys

cur_date_time = ""
avg_road_temp = 0
avg_air_temp = 0
count = 0

for line in sys.stdin:
    date_time, data = line.strip().split('\t')
    if date_time == cur_date_time:
        avg_road_temp += float(data.split(',')[1])
        avg_air_temp += float(data.split(',')[2])
        count += 1
    else:
        if cur_date_time != "":
            print("%s\t%2.2f\t%2.2f\t%d"%(cur_date_time,
		avg_road_temp/count,avg_air_temp/count,count))
        cur_date_time = date_time
        avg_road_temp = float(data.split(',')[1])
        avg_air_temp = float(data.split(',')[2])
        count = 1
            
print("%s\t%2.2f\t%2.2f\t%d"%(cur_date_time,
		avg_road_temp/count,avg_air_temp/count,count))
```

The output of this MR job should be similar to the image below, where the first column is the date/time, then the air and road temperature, and the last column contains the number of stations aggregated:
![Seattle_weather](/images/figures/road_weather_seattle_agg.png)

### Table Joins
Another SQL-like feature that is possible with MapReduce is a join of two (or potentially more) tables. SQL operates with several joins such as inner, left or right outer joins, and may be others. Below is an example of running an inner join between two tables containing county population. The first table has county id, and its name, while the second table has the history of its population. 
```python

```

## Review Questions
* Can a mapper (or a reducer) create a file on the node to store temporary computations?

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
2. You work for a social network where you deal with a file containing users with a list of friends. Suppose the users are: A,B,C, and D. Then the file format can be as follows: 
```
A [B,C,D,E]
B [A,D,E]
C [A]
```
Your task is to create a MapReduce job that will return friends in common between two users:
```
A B [D,E]
B C [A]
```
3. Implement `SELECT * FROM <table> WHERE <condition>` with MapReduce.
4. Implement `SELECT MAX(<field>) FROM <table> GROUP BY <field>` with MapReduce.
5. Implement inner join between two tables with MapReduce.
6. One method for computing Pi (even though not the most efficient) generates a number of points in a square with side = 2. Suppose a circle with radius 1 is inscribed into the square and out of 100 points generated, 75 lay on the circle. Then, `4*75/10 = 3` approximates Pi. 
Write MapReduce code that implements the method. Hint: make mappers generate the points and reducer count the ratio. 
7. Write MapReduce code to implement matrix multiplication.

[^first_names_url]:http://deron.meranda.us/data/census-derived-all-first.txt
[^stop_words_DFS]: For this application, there is no need to place it on DFS for two reasons: (1) the file is not large and (2) it will not help with data locality, since the file will reside on a single node. 






