# Hadoop Optimizations

## Combiner
A brief analysis of the word count example in MapReduce can reveal at least one area where the computation can be optimized. For each word, the mapper issues tuples like `<word,1>' that are transmitted (potentially over the network) to the reducers. As a matter of preprocessing, each node can aggregate the data output by the local mapper. This intermediate step between the mapper and reducer is what the combiner is doing. 

Suppose the mapper output is a series of tuples: <the,1>, <lock,1>,<the,1>,<series,1>. Then the combiner puts together the counts for 'the' and issues <the,2>, <lock,1>,<series,1>. For the word count problem, the combiner can function identically to the reducer and so below is an illustration of its call:

```console
yarn jar /opt/hadoop/hadoop-streaming.jar \
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
yarn jar /opt/hadoop/hadoop-streaming.jar \
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

[^why_9999]: Why is it 9999 and not 10000? The first row in the dataset is the description containing the field names that is skipped. 

