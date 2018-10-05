# Hadoop Optimizations

## Combiner
A brief analysis of the word count example in MapReduce can reveal at least one area where the computation can be optimized. For each word, the mapper issues tuples like `<word,1>' that are transmitted (potentially over the network) to the reducers. As a matter of preprocessing, each node can aggregate the data output by the local mapper. This intermediate step between the mapper and reducer is what the combiner is doing. 

Suppose the mapper output is a series of tuples: <the,1>, <lock,1>,<the,1>,<series,1>. Then the combiner puts together the counts for 'the' and issues <the,2>, <lock,1>,<series,1>. For the word count problem, the combiner can function identically to the reducer and so below is an illustration of its call:

```console
$ yarn jar /opt/hadoop/hadoop-streaming.jar \
-files mapper.py,reducer.py \
-mapper 'python mapper.py' \
-combiner 'python reducer.py' \
-reducer 'python reducer.py' \
-numReduceTasks 1 \
-input /texts/shakespeare.txt \
-output wordcount
```

The figure below summarizes word count MapReduce with the combiner. The mapper outputs 927,614 key-value pairs that would normally be sent to the reducer(s). In this example, however, these pairs are preprocessed by the combiner that now outputs only 35,061 pairs, which is a substantial reduce in data volume (~1/30). Finally the reducer processes them and outputs 23,723 pairs, which indicates that most processing has been done by the combiners in parallel. Even though there was only a single reducer, it did not have to run through the heavy processing. 

![word_count_combiner](/images/figures/word_count_combiner.png)

### Calculating Averages
Unfortunately, the combiner is not always identical to the reducer. Consider the case of calculating arithmetic averages of a series of numbers. Suppose that the mapper outputs the numbers, the reducer adds them up, and calculates the average. If the combiner jumps in and calculates the averages similar to the reducer, the final average might not be accurate, when the amount of data processed by the mappers vary in size. 

One way to circumvent the issue is to make the combiner output the sum and count of processed key-values. The reducer then keeps adding up the numbers and counts and finally calculates the average correctly. Below is the code for mapper, combiner, and reducer working on calculating average of road and air temperature in Seattle:

Mapper:
```python
import sys

for line in sys.stdin:
    try:
        line = line.split(',')
        print('%s\t%s'%(line[0],float(line[5])))
    except:
        pass
```

Combiner:
```python
import sys

cur_station = ""
station_count = 0
sum_temps = 0

for line in sys.stdin:
    try:
        line = line.split('\t')
        station, temp = line[0], float(line[1])

        if station != cur_station:
            if cur_station != "":
                print("%s\t%s\t%s"%(cur_station, sum_temps, station_count))
            cur_station = station
            station_count = 1
            sum_temps = temp
        else:
            station_count += 1
            sum_temps += temp
    except:
        pass
        
print("%s\t%s\t%s"%(cur_station, sum_temps, station_count))
```

Reducer:
```python
import sys

cur_station = ""
station_count = 0
sum_temps = 0

for line in sys.stdin:
    line = line.split('\t')
    station, _sum_temps, _station_count = line[0], float(line[1]), float(line[2])
        
    if station != cur_station:
        if cur_station != "":
            print("%s\t%2.2f"%(cur_station, sum_temps/station_count))
        cur_station = station
        station_count = _station_count
        sum_temps = _sum_temps
    else:
        station_count += _station_count
        sum_temps += _sum_temps
        
print("%s\t%2.2f"%(cur_station, sum_temps/station_count))
```

```console
$ yarn jar /opt/hadoop/hadoop-streaming.jar \
-files mapper.py,reducer.py,combiner.py \
-mapper 'python mapper.py' \
-combiner 'python combiner.py' \
-reducer 'python reducer.py' \
-numReduceTasks 1 \
-input  road_weather_10000.csv \
-output avg_temp
```

Similar to the previous case, the figure below demonstrates that the major chunk of work has been done by the combiner, that reduced the number of key-value pairs from close to 10,000[^why_9999] to 15:
![word_count_combiner](/images/figures/temp_avg_combiner.png)

## Partitioner
The MapReduce examples above use the tab symbol to separate the key from value in the key-value pairs. In majority of those cases, the key is atomic, being a string or a number. Sometimes, however, the key is compound, containing two or more fields. For instance, when analyzing IPv4 addresses containing four octets, each octet may represent a separate field and the MapReduce task may need to process them separately. 

![partitioner_fail](/images/figures/partitioner_fail.png)

Another example was a table join from the previous chapter, where the components of the key were the county ID and file name. Besides, this MR join implementation would only work in case of a single reducer that severely limited its application for big datasets. Let us consider the limitation in more detail. In the original problem, there are two CSV files, representing DB tables that need to be joined by a common key `county_id`. The mapper issues pairs like `county_id file_name "other fields"`, the reducer scans through them and joins all records with identical ID, but different file names. In case of multiple reducers, there is always a chance that the data is not partitioned correctly. Two or more reducers can get pairs pertaining to the same county and the join will not produce correct results. To circumvent the issue, the key can get partitioned so that all county data gets to the same reducer. The code below illustrates the concept.

```console
$ yarn jar /opt/hadoop/hadoop-streaming.jar \
-D mapreduce.map.output.key.field.separator=. \
-D stream.num.map.output.key.fields=2 \
-D mapreduce.partition.keypartitioner.options=-k1,1 \
-files mapper.py,identity_mr.py \
-mapper 'python mapper.py' \
-reducer 'python identity_mr.py' \
-numReduceTasks 3 \
-input IA_counties.csv,IA_counties_population.csv \
-output table_join \
-partitioner org.apache.hadoop.mapred.lib.KeyFieldBasedPartitioner
```

![partitioner_fail](/images/figures/partitioner_success.png)

## Comparator
Comparator specifies a custom method to sorting keys at the shuffle and sort stage. In the example below, the comparator will sort on the second field in descending order, treating the field as a number:

```console
yarn jar /opt/hadoop/hadoop-streaming.jar \
-D mapreduce.job.output.key.comparator.class=org.apache.hadoop.mapreduce.lib.partition.KeyFieldBasedComparator \
-D mapreduce.map.output.key.field.separator=. \
-D mapreduce.partition.keycomparator.options=-k2,2nr \
...
```

## Review Questions
* Come up with a combiner for a MapReduce job that calculates the median of numbers

## Exercises
1. Create a MapReduce job (mapper, reducer, and combiner) to calculate bigram frequencies in English.
2. Create a MapReduce job (mapper, reducer, and combiner) to calculate word collocations (use two words) in English.
3. Use MapReduce to split ISIS tweets file into two files. The first containing the original tweets and the second -- the retweets. The retweets will have `RT` at the beginning of the message. Use counters to output the number of original tweets, retweets, and the total number of tweets. Hint: use partitioner and set two reduce jobs.
4. Pass through the tweets and generate the list of users. Do your best to clean the words and numbers that are not user names. The output should be a file where user names separated by commas. You have to treat the file as Big Data, i.e. use MapReduce. 
5. Determine which user was mentioned the most. Use only the original users, i.e. those from the file. Sort by the number of retweets using key comparator. 

[^why_9999]: Why is it 9999 and not 10000? The first row in the dataset is the description containing the field names that is skipped. 

